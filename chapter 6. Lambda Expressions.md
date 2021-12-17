# chapter 6.  Lambda Expressions

​		lambda 表达式并未给语言带来新的表达能力。lambda 所能做的一切你都可以通过多一点输出代码来完成。但是 lambda 是创建函数对象的一中非常方便的方法，它对日常 c++ 软件开发的影响是巨大的。如果没有 lambda，STL 的 "_if" 算法（例如，std::find_if、std::remove_if、std::count_if等）往往只用于最普通的谓词，但当 lambda 可用时，使用这些算法时就可以变得灵活。同样，使用比较函数（例如，std::sort、std::nth_element、td::lower_bound等）的自定算法也是如此。除了这些以外，lambda 还可以为 std::unique _ptr 和 std::shared_ptr（参看Item18 和 19）快速创建自定义删除器，而且 lambda 还使得线程 API 中条件变量的谓词的说明同样简单（参看 Item39）。除了标准库之外，lambda 还促进了回调函数、接口适应函数和针对一次性调用的上下文特定函数的动态规范。lambda 确实使 C++ 成为一种令人使用舒坦的编程语言。

​		与 lambda 相关的词汇可能会令人困惑。这里有一个简短的回顾：

* lambda 表达式就是一个表达式。它是源码的一部分，如下：

```c++
std::find_if(container.begin(), container.end(),
             [](int val){return 0 < val && val < 10;}	// lambda 表达式
            );
```

* 闭包是由 lambda 创建的运行时对象。根据捕获模式，闭包保存捕获数据的副本或引用。在上面对 std::find_if 的调用中，闭包是在运行时作为 std::find_if 的第三个参数传递的对象
* 闭包类是一个实例化的闭包的类。每个 lmbda 都会是编译器生成一个唯一的闭包类。lambda 中的语句成为其闭包类的成员函数中的可执行指令

​		lambda 通常用于创建仅用作函数参数的闭包。上面对 std::find_if 的调用就是这种情况。然而，闭包通常可以被复制，所以通常可能有多个对应于单个 lambda 的闭包类型。例如，下面代码中 c1、c2 和 c3 都是 lambda 生成的闭包副本：

```c++
{
    int x;						// x is local variable
    ...
	auto c1 =
        [x](int y){return x*y > 55;};	// c1 is copy of the closure produced 
    									// by thi lambda
    
    auto c2 = c1;						// c2 is copy of c1
    auto c3 = c2;						// c3 is copy of c2
    ...
}
```

​		非正式地，模糊 lambda、闭包和闭包类之间的边界是完全可以接受的。但是接下来地章节中，区别编译期间存在的内容（lambda和闭包类）、运行时存在的内容（闭包）以及它们之间的关系通常很重要。

#### *item 31. 避免默认捕获模式*

​		C++11 中有两种默认的捕获模式：按引用和按值。默认的按引用捕获可能导致悬空引用。默认值捕获诱使你认为你可以规避这个问题（实际上不是），并且它使你认为你的闭包是对立的（实际上可能不是）。本小节将对此进行阐述，了解默认的按引用捕获带来的危险。

​		按引用捕获会导致闭包包含对局部变量或在定义 lambda 范围内可用参数的引用。如果该 lambda 创建的闭包生存周期超过局部变量或参数的生存周期，闭包中的引用将会被悬挂。例如，假设我们有一个成员是过滤函数的容器，每个函数都接受一个 int 值，并返回一个 bool 值，指示传入的值是否满足过滤器：

```c++
using FilterContainer =							// see Item 9 for "using",
    std::vector<std::function<bool(int)>>;		// Item2 for std::function

FilterContainer filters;						// filtering funcs
```

我们可以像这样为 5 的倍数添加一个过滤器：

```c++
filters.emplace_back(							// see Item 42 for info
    [](int value){return value%5 == 0;}			// on emplace_back
);
```

然而，我们可能需要在运行时计算除数，也就是说，我们不能仅仅将 5 硬编码到 lambda 中。所以添加过滤器可能会像这样：

```c++
void addDivisorFilter()
{
    auto calc1 = computeSomeValue1();
    auto calc2 = computesomeValue2();
    
    auto divisor = computeDivisor(calc1, calc2);
    
    fiters.emplace_back(							// danger! ref to divisor
        [&](int value){return value%divisor == 0;}	// will dangle!
    );
}
```

​		这段代码是即将出现问题，lambda 使用局部变量 divisor，但当 addDivisorFilter 返回时，该变量不再存在。即，在运行 fiters.emplace_bak 之后失效，所以添加到 filters 的函数本质上是是失败的。使用该 filter 创建的函数会出现未定义的行为。

​		现在，如果 divisor 的按引用捕获是显式的，也会存在同样的问题：

```c++
filters.emplace_back(
    [&divisor](int value)					// danger! ref to
    { return value%divisor == 0; }			// divisor will still danger
);
```

​		但是通过显式捕获，更容易看出 lambda 的可行性取决于 divisor 的生命周期。此外，写出名称"divisor"提醒我们确保 divisor 至少与 lambda 的闭包生命周期一样长。这比"[&]"传达的一般意义"确保没有任何东西悬挂"更加醒目。

​		如果你知道一个闭包会立即被使用（例如，通过传递给 STL 算法）并且不会被复制，那么他所持有的引用就不会比创建它的 lambda 的环境中的局部变量和参数存在更长的时间。这种情况下，你可能会说，不存在悬空引用的风险，因此没有理由避免默认的按引用捕获模式。例如，我们的过滤 lambda 表达式只用做 C++11 的 std::all_of 的参数，它返回一个范围内所有满足某一条件的元素：

```c++
template<typename C>
void workWithContainer(const C& container)
{
    auto calc1 = computeSomeValue1();				// as above
    auto calc2 = computeSomeValue2();				// as above
    auto divisor = computeDivisor(calc1, calc2);	// as above
    using ContElemT = typename C::value_type;		// type of element in container
    
    using std::begin;								// for genericity
    using std::end;									// see Item 13
    
    if (std::all_of(											// if all values
        begin(container), end(container),						// in container
        [&](const ConElemT& value){return value%divisor == 0;})	// are multioles
    	)														// of divisor...
	{...}
	else
	{...}
}
```

​		的确，这是安全的，但是它的安全性有点不稳定。如果发现 lambda 在其他上下文很有用（例如，作为要添加到过滤器容器的函数），并且被复制粘贴一个上下文闭包生命周期超过 divisor，那么就又会出现悬挂的情况，捕获子句中没有任何内容特别提醒你对 divisor 执行生命周期分析。长远来看，明确列出 lambda 依赖的局部变量和参数是更好的软件工程。

​		顺便说一下，在 C++14 lambda 参数规范中使用 auto 的能力意味着以上代码可以在 C++14 中简化。ContElemT 的类型定义也可以取消，然后按如下实现：

```c++
if (std::all_of(begin(container), end(container),
               [&](const auto& value) {return value%divisor==0;}	// C++14
               )
   )
```

​		解决 divisor 问题的一种方法是默认按值捕获模式。也就是说，我们可以将 lambda 添加到 filters 中，如下所示：

```c++
filters.emplace_back(								// now divisor
    [=](int value){return value%divisor == 0;}		// can't dangle
);
```

​		对于本例来说，这已经够用了，但是通常情况下，默认按值捕获并不会像想象中那样万能。假如按值捕获指针，则将指针赋值到由 lambda 产生的闭包中，但是在 lambda 之外的代码删除指针还是会导致副本悬空。

​		根据第4章中了解到的智能指针，你可能会说不会的。只有C++98才会使用原始指针和删除它，但是不排除这个情况。

假设 Widget 可以做的一件事是在过滤器中添加实例：

```c++
class Widget{
  public:
    ...
	void addFilter() const;	// ctors,etc. add an entry to filters
  private:
    int divisor;				// used in Widget's filter
};
```

Widget::addFilter 可以这样定义：

```c++
void Widget::addFilter() const
{
    filters.emplace_back(
        [=](int value){return value%divisor == 0;}
    );
}
```

以上代码存在严重问题，捕获仅适用于创建 lambda 的范围内可见的非静态局部变量（包括参数）。在 Widget::addFilter 的主体中，divisor 不是局部变量，它是 Widget 类的数据成员。它无法被捕获，然而，如果取消默认捕获模式，代码将无法编译：

```c++
void Widget::addFilter() const
{
    filters.emplace_back(
        [](int value){return value%divisor == 0;}	// error! divisor not available
    );
}
```

此外，如果尝试显式捕获除数（通过值或通过引用），捕获将不会编译，因为 divisor 不是局部变量或参数：

```c++
void Widget::addFilter() const
{
    filters.emplace_back(
        [divisor](int value){return value%divisor == 0;}	// error!no local divisor to
        													// capture
    );
}
```

那这事咋回事儿呐？这是因为原始指针 this 的存在。每个非静态成员函数都有一个  this 指针，每次提到类的数据成员时都会使用该指针。所以，在任何 Widget 成员函数中，编译器在内部将 divisor 的使用替换为 this->divisor。在带有默认按值捕获的 Widget::addFilter 版本中：

```c++
void Widget::addFilter() const
{
    filters.emplace_back(
        [=](int value){return value%divisor == 0;}
    );
}
```

被编译器捕获的是 Widget 的 this  指针，而不是 divisor。编译器处理后的代码如下：

```c++
void Widget::addFilter() const
{
    auto currentObjectPtr = this;
    filters.emplace_back(
        [currentObjectPtr](int value)
        {return value%currentObjectPtr->divisor ==0;}
    );
}
```

理解这一点等同于理解由这个 lambda 产生的闭包的可行性与 Widget 的生命周期相关联，该 Widget 的 this 指针被拷贝到其中。考虑一下这段代码，根据第4章，使用智能指针变量：

```c++
using FilterContainer =							// as before
    std::vector<std::function<bool(int)>>;

FIlterContainer filters;						// as before

void doSomeWork()
{
    auto pw =						// create Widtget;see
        std::make_unique<Widget>();	// Item 21 for std::make_unique
    
    pw->addFilter();				// add filter that use Widget::divisor
    ...
}									// destory Widget; filters now hold dangling pointer!
```

​		当调用 doSomeWork 时，会创建一个依赖于由 std::make_unique 生成的 Widget 对象的过滤器，也就是说，一个包含该 Widget 指针副本的过滤器-------Widget 的 this 指针。这个过滤器会被添加到过滤器中，但是当 doSomeWork 完成时，这个 Widget 被管理其生命周期的 std::unique_ptr 销毁（参看 Iterm 18）。从这一点看，filters 包含一个带有悬挂指针的实例。

这个特殊的问题可以通过创建一个你想要捕获的数据成员的局部变量来解决，然后捕获这个副本：

```c++
void Widget::addFilter() const
{
    auto divisorCopy = divisor;			// copy data member
    filters.emplace_back(
        [divisorCopy](int value)				// capture the copy use the copy
        { return value % divisorCopy == 0; }
    );
}
```

老实说，按照这种实现方式，默认按值捕获可以正常工作，

```c++
void Widget::addFilter() const
{
    auto divisorCopy = divisor;			// copy data member
    
    filter.emplace_back(
        [=](int value)
        { return value % divisorCopy == 0; }	// capture the copy use the copy
    );
}
```

C++14 中存在广义 lambda 捕获，可以用来直接捕获数据成员：

```c++
void Widget::addFilter() const
{
    filters.emplace_back(
        [divisor = divisor](int value)		// C++14;
        { return value % divisor ==0; }		// copy divisor to closure use copy
    );
}
```

​		然而，对于广义 lambda 捕获没有默认捕获模式这样的东西，所以即使在 C++14 中，本章的建议--- 避免默认不获模式。

​		默认按值捕获的一个缺点是它们表示相应的闭包是自包含的，跟闭包之外的数据是隔离的。一般来说，事实并非如此，因为 lambda 表达式可能不仅依赖于局部变量和参数（可能会被捕获），还依赖于具有静态存储持续时间的对象。此类对象在全局或命名空间范围内定义，或在类、函数或文件中声明为静态。这些对象可以在 lambda 表达式中使用，但不能被捕获。考虑一下我们前面看到的添加除数过滤器函数的修订版本：

```c++
void addDivisorFileter()
{
    static auto calc1 = computeSomeValue1();			// now static
    static auto calc2 = computeSomeValue2();			// now static
    static auto divisor =
        computeDivisor(calc1, calc2);
    
    filters.emplace_back(
        [=](int value)
        {return value%divisor == 0;}	//captures nothing!refers to above static
    );
    
    ++divisor;							// modify divisor
}
```

​		这段代码一般人看到"[=]"后认为"lambda 复制了它使用的所有对象，因此是自包含的"。但是它并非独立的，这个 lambda 没有使用任何非静态局部变量，因此没有补货任何内容。相反，lambda 的代码引用了静态变量 divisor 。在每次调用 addDivisorFilter 之后，当 divisor 被增加时，通过这个函数添加到 filters 的所有 lambda 都将使用最新 divisor 的值。实际上，这些 lambda 是通过引用捕获 divisor 的，跟按值捕获的子句相矛盾。如果不使用默认按值捕获，可以消除代码这种误解的风险。

   **小结**

* 默认按值引用捕获可能会导致悬空引用
* 默认按值捕获容易收到悬空指针的影响（尤其是这个），并且它会误导性暗示 lambda 是自包含的

---

#### *item 32. 使用 init capture 将对象移动到闭包中*

​		有时，按值捕获和引用捕获都不是你想要的。如果你有一个 move-onlu 的对象（例如，std::unique_ptr 或 std::future)，你想把它放到一个闭包中，C++11 没办法做到这一点。如果你有一个对象，它的复制成本很高，但移动成本低（例如，标准库中的大多数容器），并且你想把这个对象放到一个闭包中，你更倾向于移动它而不是复制它。然而，C++11 还是做不到这一点。

​		但是 C++14 就另提别论了，它支持将对象移动到闭包中。但是 C++11 中有一些方法可以近似地进行移动捕获。

​		即使采用 C++11，移动捕获地缺失也被认为是一个缺点。简单地补救方法就是将这个缺失在添加到 C++14 中，但是标准化委员会选择了不同的方法。他们引入了一种新的捕获机制，这种机制非常灵活，移动捕获只是它能实现的技能之一。这个功能被称为 init capture。它几乎可以完成 C++11 捕获表单所能做的所有事情，并且更多。有一件事你不能用 init capture 来表达，那就是默认捕获模式，但是 Item31 解释了无论如何你应该规避这种模式。（对于 C++11 capture 所涵盖的情况，init capture 的语法有点啰嗦，所以在 C+11 capture 完成任务的情况瞎，使用它事完全合理的）。

使用 init capture 你可以指定：

1、*由 lambda 生成的闭包类中的数据成员的名称* 和 2、*初始化该数据成员的表达式*

以下是如何使用 init capture 将 std::unique_ptr 移动到闭包中的方法：

```c++
class Widget{						// some useful type
  public:
    ...
	bool isValidated() const;
    bool isProcessed() const;
    bool isArchived() const;
  private:
    ...
};

auto pw = std::make_unique<Widget>();		// create Widget; see Iterm21 for info to
											// std::make_unique

...											// configure *pw
    
auto func = [pw = std::move(pw)]				// init data mbr
			{									// in closure w/
    			return pw->isValidated() &&		// std::move(pw)
                    pw->isArchived();
			};
```

​		上面的 lambda 捕获表中使用了 init capture。"=" 左边是你要指定的闭包类中的数据成员的名称，右边是初始化表达式。有趣的是，"=" 左边的作用域范围与右边的范围不同。左边的范围是闭包类范围。右边的作用域预定义 lambda 的范围相同。在上面的例子中，"=" 左边的名字 pw 指的是闭包类中的一个数据成员，而右边的名字 pw 指的是 lambda 上面声明的对象。即调用 std::make_unique 初始化的变量。因此 "pw = std::move(pw)" 意味着 "在闭包中创建一个数据成员 pw，并使用将 std::move 应用到局部变量 pw 的结果来初始化该数据成员"。

​		通常，lambda 中的代码位于闭包类作用域，因此在这里使用 pw 指的是闭包类的数据成员。

​		这个例子中的注释 "configure *pw" 表明，在 std::make_unique 创建 Widget 之后，在 lambda 捕获 Widget 的 std::unique_ptr 之前，Widget 以某种方式进行了修改。如果不需要这样的配置，例如，如果 std::make_unique 创建的 Widget 适合被 lambda 直接捕获，那么局部变量 pw 就非必要了，因为闭包类的数据成员可以由 std::make_unique 直接初始化：

```c++
auto func = [pw = std::make_unique<Widget>()]		// init data mbr in closure w/
			{										// result of call to make_unique
    			return pw->isValidated()
                    && pw->isArchivaed();
			};
```

​		需要明白的一点是 C++14 的 "捕获" 概念是从 C++11 中延申推广的，因为在 C++11 中，不能捕获表达式的结果。因此，init capture 的另一个名称是---推广lambda捕获。

​		但是，如果你使用的是一个或多个编译器不支持 C++14 的 init capture 怎么办？如何用一个不支持移动捕获的语言来完成移动捕获？

​		请记住，lambda 表达式只是一种生成类和创建该类型对象的方法。你可以用 lambda 做任何事，手工也可以实现。例如，刚才看到的 C++14 示例代码可以用 C++11 写成如下：

```c++
class IsValAndArch{								// "is validated and archived"
  public:
    using DataType = std::unique_ptr<Widget>;
    
    explicit IsValAndArch(DataType&& ptr)		// Item25 explains use of std::move
        :pw(std::move(ptr)) {}
    
    bool operator()() const
    { return pw->isValidated() && pw->isArchived(); }
    
  private:
    DataType pw;
};

auto func = IsValAndArch(std::make_unique<Widget>());
```

​		这比编写 lambda 需要跟多的工作量，但是并不能改变这样一个事实：如果你想要 C++11 中的一个类支持其数据成员的移动初始化，唯一的差别就是你需要多敲一些代码。如果你想继续使用 lambda（考虑到它的便利），可以在 C++11 中模拟移动捕获：1、将要捕获的对象通过 std::bind 生成一个 function 对象  2、然后，给 lambda 一个对 "捕获" 对象的引用。

​		如果你熟悉 std::bind ，那么代码对你来说很简单。如果你不熟悉 std::bind，那么花点时间熟悉这些代码是值得的。假设你想创建一个局部 std::vector，放入一组适当的值，然后将它移动到一个闭包中。在 C++14 中实现很简单：

```c++
//####C++14 实现
std::vector<double> data;			// object to be moved int closure
...							// populate data
auto func = [data = std::move(data)]	// C++14  init capture
			{ /* uses of data */ };

//####C++11 实现
std::vector<double> data;			// as above
...
auto func =
    std::bind(									// C++11 emulation of init capture
    	[](const std::vector<double>& data)
    	{ /* uses of data */ },
    	std::move(data)
	);
```

​		像 lambda 表达式一样，std::bind 产生函数对象（std::function）。例子中调用 std::bind 绑定对象返回的函数对象。std::bind 的第一个参数是一个可调用对象。后续参数表示要传递给该对象的值。

​		绑定对象包含传递给 std::bind 的所有参数的副本。对于每个左值参数，绑定对象中的相应对象是复制构造的。对于每个右值，他都是移动构造的。在这个例子中，第二个参数是一个右值（std::move 的结果---参看Item23），所以数据被移动构造到绑定对象中。这种移动构造是移动捕获模拟的关键，因为将右值移动到绑定对象是我们解决无法右值移动到 C++11 闭包的方法。

​		当 bing 对象被"调用"时（即，它的函数调用作用符被调用），它存储的参数是最初被传递给 std::bind 的可调用对象的参数。在本例中，当调用 func （bind对象）时，func 中移动构造的数据副本是 std::bind 中传递给 lambda 的参数。

​		这个 lambda 与我们在 C++14 中使用的 lambda 相同，只是添加了一个参数 data，以对应我们的伪移动捕获对象。该参数是对绑定对象中数据副本的左值引用。（它不是一个右值引用，因为虽然用于初始化数据副本表达式（"std::move(data)"）是一个右值），但数据副本本身是一个左值。）因此，在 lambda 中使用数据将操作绑定对象中移动构造的数据副本。

​		默认情况下，由 lambda 生成的闭包类中的 operator() 成员函数是 const。它的效果是在 lambda 体中所有的数据成员都是不可修改的。但是，再绑定对象中移动构造的数据副本不是 const，因此为了防止数据副本在 lambda 中被修改，lambda 的形参被声明为 reference-to-const。如果 lambda 被声明为 mutable，则其闭包类中的 operator() 将不会被声明为 const，并且在 lambda 的形参声明中省略 const 是合适的：

```c++
auto func = std::bind(
    		[](std::vector<double>& data) mutable		// C++11 emulation
    		{ /* uses of data */ },						// of init  capture
    		std::move(data)								// for mutable lambda
			);
```

​		因为 bind 对象存储了传递给 std::bind 的所有参数的副本，所以我们示例中的 bind 对象包含了作为其第一个参数的 lambda 所产生的闭包的副本。因此闭包的生存期与绑定对象的生存周期相同。这很重要，因为这意味着 只要闭包存在，绑定对象包含的伪移动捕获对象也就存在。针对上面接受的内容你需要注意以下几点：

* 无法将对象移动构造到 C++11 闭包中，但可以将对象移动构造到 C++ 绑定对象中
* 在 C++11 中模拟移动捕获包括将对象移动构造为绑定对象，然后通过引用将移动构造的对象传递给 lambda
* 因为绑定对象的生命周期和闭包的生命周期相同，所以可以将绑定对象中的对象视为闭包中成员

第二个示例使用 std::bind 模拟移动捕获，下面是之前描述过的在闭包中创建 std::unique_ptr 的 C++14 代码：

```c++
auto func = [pw = std::unique_ptr<Widget>()]					// as before, create
			{ return pw->isValidated() && pw->isArchived(); };	// pw in closure
```

C++11 模拟版本：

```c++
auto func = std::bind(
    			[](const std::unique_ptr<Widget>& pw)
    			{ return pw->isValidated() && pw->isArchived(); },
    			std::make_unique<Widget>()
			);
```

在 Item34 中，提到提倡使用 lambda 而不是 std::bind。但是也介绍了需要使用 std::bind 的情况。（在 C++14 中， init capture 和 auto 参数等特性规避了这些情况）

   **小结**

* init capture ---- 自我理解就是初始化捕获列表

* 使用 C++14 的 init capture 可以将对象移动到闭包中  
* 在 C++11 中可以通过手写类或 std::bind 模拟 init capture

---

#### *item 33. 在 auto&& 参数上使用 decltype 来 std::forward 转发*

​		C++14 最令人兴奋的特性之一是泛型 lambda - 它在参数规范中使用 auto。这个特性的实现很简单：lambda 的闭包类中的 operator() 是一个模板。例如，给定这个：

```c++
auto f = [](auto x) { return func(nomalize(x)); };
```

闭包类的函数调用操作符实现可以理解为如下：

```c++
class SomeComplierGeneratedClassName{
  public:
    template<typename T>					// see Item 3 for
    auto operator()(T x) const				// auto return type
    { return func(nomalize(x)); }
    ...										// other closure class
};											// functionality
```

​		在这个例子中，lambda 对它的参数 x 所作的唯一一件事就是转发它来实现标准化。如果这种标准化对左值和右值的处理不一样，那么这个 lambda 就没办法实现，因为入参可能事左值也可能是右值。

​		正确的处理应该是完美转发 x 来实现。这需要修改代码两处，首先，x 必须成为一个通用引用（参看 Item24）,其次，它必须通过 std::forward 进行转发（参看Item25）。整体来说这些改动并不大：

```c++
auto f = [](auto&& x)
		{ return func(nomalize(std::forward<???>(x))); };
```

​		这时你会发现该给 std::forward 传递啥类型呐？通常完美转发时，通常模板函数中会接受一个 T 类型，这是只需要给 std::forward 传递 T 就可以了。在 lambda 生成的闭包类中模板 operator() 有一个 T，但是不能在从 lambda 引用它，所以它也没啥用。

​		Item28 讲述过，如果将左值参数传递给通用引用形参，则该形参的类型将成为左值引用。如果传递一个右值，该参数将成为一个右值引用。这意味着在 lambda 中我们需要判断参数 x 的类型，decltype 就提供了这个方法（参看Item3）。如果一个左值，通过 decltype(x) 将产生一个左值引用类型。如果传递了一个右值，decltype(x) 将产生一个右值引用类型。

​		Item28 还说明了在调用 std::forward 时，惯例规定参数类型用一个左值引用来表示一个左值，一个非引用来表示一个右值。在我们的 lambda 中，如果 x 被绑定到左值，decltype(x) 将产生一个左值引用，这符合惯例。但是，如果 x 被绑定到一个右值，decltype(x) 将产生一个右值引用，而不是非引用。

​		但是来看一下 Item28 中 C++14 版本实现的 std::forward：

```c++
template<typename>
T&& forward(remove_reference_t<T>& param)		// in namespace std
{
    return static_cast<T&&>(param);
}
```

​		如果客户端代码想要完美转发一个 Widget 类型的右值，通常用 Widget 类型（非引用类型）实例化 std::forward，并且 std::forward 模板产生这样的函数：

```c++
Widget&& forward(Widget& param)				// instantiation of 
{											// std::forward when T is Widget
    return static_cast<Widget&&>(param);
}
```

​		但是考虑一下，如果客户端代码想要完美转发相同的 Widget 类型的右值，但是没有按照约定将 T 指定为非引用类型，而是将它指定为右值引用，然后会发生发生什么情况。也就是说，考虑将 T 指定为 Widget&& 会发生什么。在 std::forward 被初始实例化， std::remove_reference_t 应用之后，但是在引用折叠之前（参看Item28），std::forward 看起来会是这样：

```c++
Widget&& && forward(Widget& param)				// instantiation of std::forward
{												// wher T is Widget&& (before
    return static_cast<Widget&& &&>(param);		// reference-collapsing)
}
```

​		应用引用折叠规则后，对右值引用的右值引用变为单个右值引用，实现如下：

```c++
Widget&& forward(Widget& param)				// instantion of std::forward
{											// when T is Widget&&
    return static_cast<Widget&&>(param);	// (after reference-collaosing)
}
```

​		如果将这个实例化与调用 std::forward 并将 T 设置为 Widget 时的结果进行对比，你会发现它们是相同的。这就意味着用右值引用类型实例化 std::forward 的结果与用非引用类型实例化它的结果相同。

​		这或许是个好消息，因为当一个右值作为参数传递给我们的 lambda 参数 x 时， decltype(x) 产生一个右值引用类型。依据上面讲述的，对于左值和右值，将 decltype(x) 传递给 std::forward 会得到我们预期的效果。因此，我们的完美转发 lambda 可以这样：

```c++
auto f = 
    [] (auto&& param)
	{
    	return 
            func(normalize(std::forward<decltype(param)>(param)));
	};
```

根据 C++14 lambda 表达式可以接受变参，然后进行进一步优化：

```c++
auto f = 
    [](auto&&... params)
	{
    	return 
            func(normalize(std::forward<decltype(params)>(params)...));
	};
```

   **小结**

* 在 auto&& 参数上使用 decltype 然后传递给 std::forward 进行转发它们

---

#### *item 34. 优先选择 lambda 而不是 std::bind*

​		std::bind 是 C++11 继 C++98 的 std::bind1st 和 std::bind2nd 之后的版本，但是，从 2005 年开始，它非正式的成为标准库的一部分。这时，标准化委员会通过了一份名为 TR1 的文件，其中包括 bind 的规范。（在 TR1 中，bind 在不同的命名空间中，所以它是 std::TR1::bind，而不是 std::bind，而且一些接口细节是不同的。）可能你就是使用过的一员，而且你还挺喜欢使用这些工具的。但是本章中，想告诉你在 C++11 中，lambda 几乎总是比 std::bind 更好使，在 C++14 中就更不用多说了。本章需要你对 std::bind 有一定的了解。如果不是，那么在继续之前，你需要先对它有一定的了解。

​		在 Item32 中，我们将从 std::bind 返回的函数对象称之为绑定对象。倾向于使用 lambda 的重要原因是 lambda 更具可读性。例如，假设我们有一个设置声音报警的函数：

```c++
// typedef for a point in time(see Item 9 for syntax)
using Time = std::chrono::steady_clock::time_point;

// see Item 10 for "enum class"
enum class Sound {Beep, Siren, Whistle};

// typedef for a length of time
using Duration = std::chrono::steady_clock::duration;

// at time t, make sound s for duration d
void setAlarm(Time t, Sound s, Duration d);
```

​		再假设在程序的某个时刻，我们决定要一个闹钟，一小时后响，并且持续30秒。但是，警报的声音还没有确定。我们可以编写一个 lambda 来修改 setAlarm 接口指定特定的报警声音：

```c++
// setSoundL("L" for "lambda") is a function object allowing a
// sound to be specified for a 30-sec alarm to go  off an hour after it's set
auto setSoundL = 
    [](Sound L)
	{
    	// make std::chrono components avialable w/o qualifiacation
    	using namespace std::chrono;
    	// alarm to go off in an hour for 30 seconds
    	setAlarm(steady_clock::now() + hours(1), s, seconds(30));
	};
```

​		在上面的 lambda 中 setAlarm 的调用，看起来就像很普通的函数调用，即使是熟悉 lambda 的读者也可以看到传递给 lambda 的参数 s 是作为参数传递给 setAlarm 的。

​		我们可以通过使用 秒（s）、毫秒（ms）、小时（h）等标准后缀简化 C++14 代码，这些标准后缀建立在 C++11 对用户定义文字的支持之上。这些后缀是在 std::literals 命名空间中实现的，因此上面代码可以如下改动：

```c++
auto setSoundL =
    [](Sound s)
	{
    	using namespace std::chrono;
    	using namespace std::literals;			// for C++14 suffixes
    	setAlarm(steady_clock::now() + 1h, s, 30s);		// C++14, but same meaning as 
	};													// above
```

​		下面是我们第一次尝试编写相应 std::bind 的调用。它有一个错误，我们一会儿修复一下，但是正确的代码更复杂，即使是一个简单的示例版本也带来了很多麻烦：

```c++
using namespace std::chrono;			// as above
using namespace std::literals;
using namespace std::placeholders;		// needed for use of "_1"

auto setSoundB =						// "B" for "bind"
    std::bind(setAlarm, steady_clock::now() + 1h,	//incorrect! see below
             _1, 30s);
```

​		不像 lambda 那样需要突出注意哪一点，这段代买的读者只需要知道调用 setSoundB 会以调用 std::bind 中指定的 time 和 duration 调用 setAlarm。对于不熟悉的人来说，占位符 "_1" 简直就是个鬼，如果你了解 std::bind 的话，你会清楚这些参数跟 setAlarm 的参数一一对应。这个占位符参数的类型对 std::bind 的调用中没有被识别，所以读者必须参看 setAlarm 声明来确定传递给 setSoundB 的参数类型。

​		但是，正如我所说的，代码并不完全正确。在 lambda 中，表达式 "steady_clock::now() +1h" 显然是 setAlarm 的一个参数，它将在 setAlarm 被调用时计算。这个场景就是：我们希望在调用 setAlarm 一个小时后响起。然而，在 std::bind 调用中，"steady_click::now() + 1h" 作为参数传递给 std::bind，而不是 setAlarm。这就导致当调用 std::bind 时，表达式将被求值，由该表达式得到的时间将存储在最终产生的绑定对象中。因此，报警将被设置为在调用 std::bind 后一个小时响起，而不是在调用 setAlarm 后一个一小时响起。

​		要解决这个问题，需要告诉 std::bind 将表达式的求值延迟到 setAlarm 被调用时，方法是在第一个调用中嵌套对 std::bind 的调用：

```c++
auto setSoundB =
    std::bind(setAlarm,
             std::bing(std::plus<>(), steady_clock::now(), 1h),
             _1,
             30s );
```

​		如果你熟悉 C++98 的 std::plus 模板，你会惊讶的发现，在这段代码中，尖括号之间没有指定类型，即，代码包含的是 "std::plus<>" 而不是 "std::plus<type>"。在 C++14 中，标准操作符模板的模板类型实参通常可以省略，所以这里不是需要提供它。C++11 没有提供这样的特性，所以 C++11 的 std::bind 不等价与 lambda：

```c++
using namespace std::chrono;				// as above
using namespace std::placeholders;

auto setSoundB =
    std::bind(setAlarm,
              std::bind(std::plus<steady_clock::time_point>(), 
                        steady_clock::now(), hours(1)),
              _1,
              seconds(30)
             );
```

当 setAlarm 有重载，会出现一个新的问题。假设有一个重载接受第四个参数来指定报警音量：

```c++
enum c;ass Volume{ Normal, Loud, LoudPlusOlus };
void setAlarm(Time t, Sound s, Duration d, Volume v);
```

lambda 会像之前一样正常运行，因为重载解析会选择 setAlarm 的三个参数版本：

```c++
auto setSoundL =						// same as before
    [](Sound s)
	{
    	using namespace std::chrono;				// fine,calls 3-arg version of 
    	setAlarm(steady_clock::now() + 1h, s, 30s);	// setAlarm
	};
```

std::bind 的调用就会出现编译失败的情况：

```c++
auto setSoundB =
    std::bind(setAlarm,
              std::bind(std::plus<>(),
                        steady_clock::now(),
                        1h),
              _1,
              30s);
```

​		问题是编译器无法确认应该将两个 setAlarm 函数中的哪个函数传递给 std::bind。它们只有一个函数名，而函数名本身就是具有二义性的。

​		为了让 std::bind 的调用编译通过，setAlarm 必须强制转换为正确的函数指针类型：

```c++
using SetAlarm3ParamType = void(*)(Time t, Sound s, Duration d);
auto setSoundB =											// now okay
    std::bind(static_cast<SetAlarm3ParamType>(setAlarm),
              std::bind(std::plus<>(),
                        steady_clock()::now,
                        1h
                       ),
              _1,
              30s
			);
```

​		上边改动会成功，但是带来 lambda 和 std::bind 之间的另一个区别。在 setSoundL 的函数调用操作符（即 lambda 闭包类的函数调用操作符）内部，对 setAlarm 的调用是一个普通的函数调用，通常编译器会使用内联：

```c++
setSondL(Sound::Siren);		// body of setAlarm may well be inlined here
```

​		然而，对 std::bind 的调用是将指向 setAlarm 的函数指针传递给它，这就意味着在 setSoundB 的函数调用操作符（即，bind 对象的函数调用操作符）内部，对 setAlarm 的调用是通过一个函数指针进行的。编译器不太可能通过函数指针内联函数调用，这意味着通过 setSound 调用 setAlarm 比通过 setSoundL 调用更不可能完全内联：

```c++
setSoundB(Sound::Siren);	// body of setAlarm is less likely to be inlined here
```

因此，与 std::bind 相比使用 lambda 可能会生成更快的代码。setAlarm 示例仅涉及一个简单的函数调用。如果想做更复杂的事情，使用会更倾向于 lambda。例如，考虑 C++14 中这个 lambda，它返回入参是否在最小值（lowVal）和最大值（highVal）之间，其中 lowVal 和 highVal 是局部变量：

```c++
auto betweenL =
    [lowVal, highVal](const auto& val)				// C++14
	{ return lowVal <= val && val <= highVal; };
```

std::bind 也可以实现，但是结构会非常的晦涩：

```c++
using namespace std::placeholders;				// as above
auto betweenB =
    std::bind(std::logical_add<>(),								//C++14
              std::bind(std::less_equal<>(), lowVal, _1),
              std::bind(std::less_equal<>(), _1, highVal)
              );
```

C++11 中，我们必须指定要比较的类型，然后 std::bind 调用将如下所示：

```c++
auto betweenB =
    std::bind( std::logical_and<bool>(),						//C++11 version
              std::bind(std::less_equal<int>(), lowVal, _1),
              std::bind(std::less_equal<int>(), _1, highVal)
             );
```

当然，在 C++11 中，lambda 不能接受一个 auto 参数，所以必须指定一个类型：

```c++
auto betweenL =
    [lowVal, HighVal](int val)									// C++11 version
	{ return lowVal <= val && val <= highVal; };
```

综上所述，我们可以发现 lambda 不仅短小，而且更加易懂和维护。

早些时候，我说过，对于那些不怎么使用 std::bind 的人，对它的占位符会一头雾水。但是费解的不止占位符。假设我们有一个函数创建 Widget 的压缩副本，并且我们想创建一个函数对象，它允许我们指定一个特定的 Widget w 应该被压缩多少，然后使用 std::bind 将创建对象：

```c++
enum class CompLevel{ Low, Normal, High };			// compression level
Widget compress(const Widget& w,
               CompLevel lev);						// make compressing copy of w

Widget w;
using namespace std::placeholders;
auto compressRateB = std::bind(compress, w, _1);
```

​		现在，当我们将 w 传递给 std::bind 时，它必须被存储起来，以便稍后调用 compess。它被存储在对象 compressRateB 中，但是它是如何存储的-------通过值还是引用？这是有区别的，因为如果 w 在调用 std::bind 和 调用 compressRateB 之间被修改，则按引用存储的 w 也会相应改变，而按值存储不会。

​		答案是它是按值存储的，但知道这一点的唯一方法是记住 std::bind 是如何工作的；在 std::bind 的调用中没有它的直观迹象。与 lambda 方法相比，其中 w 是通过值还是引用获取是很直观的：

```c++
auto compressRateL =
    [w](CompLevel lev)					// w is captured by value;lev is passed by value
	{ return compress(w, lev) };

compressRateL(CompLevel::High);			// arg is passed by value
```

​		但是在对由 std::bind 产生的对象的调用中，参数是如何传递的呐？

```c++
compressRateB(CompLevel::High);			// hiw is arg passed?
```

​		同样，只有了解了 std::bind 的工作原理后才知道是如何传参的。（答案是传递给绑定对象的所有参数都是通过引用传递的，因为此类对象的函数调用操作符使用了完美转发）

​		与 lambda 相比，使用 std::bind 的代码可读性较差，而且可能效率较低。在 C++14 中，std::bind 没有合理的用例。但是在 C++11 中，std::bind 在两种受约束的情况下展现其存在价值：

* 移动捕获。C++11 lambda 不提供移动捕获，但是可以通过 lambda 和 std::bind 的组合来模拟。有关详细信息请参看 Item32，其中还提到了 C++14 提供初始化捕获特性，可以避免这种模拟。
* 多态函数对象。因为绑定对象上的函数调用操作符使用完美转发，所以它可以接受任何类型的参数（Item30 中描述了完美转发的限制）。当你希望将一个有模板化函数调用操作符的对象进行绑定时，std::bind 会很有用。如，给如下类，

```c++
class PolyWidget{
  public:
    template<typename T>
    void operator()(const T& param);
    ...
};
```

std::bind 可以如下绑定 PolyWidget:

```c++
PolyWidget pw;
auto boundPW = std::bind(pw, _1);
```

boundPW 接下来可以入参各种类型的参数来调用：

```c++
boundPW(1930);			// pass int to PolyWidget::operator()
boundPW(nullptr);		// pass nullptr to PolyWidget::operator()
boundOW("Rosebud");		// pass string literal to PolyWidget::operator()
```

对于 C++111 lambda 是没办法做到这一点的。但是，在 C++14 中，通过带 auto 的参数 lambda 轻松实现：

```c++
auto boundPW = [](const auto& param)			// C++14
				{ pw(param); };
```

​	当然，这些是边缘情况，而且是暂时的，因为支持 C++14 lambda 的编译器会越来越普遍。当 bind 在 2005 年被非正式添加到 C++ 时，它比 1998 年的前辈有了很大的进步。但是，在 C++11 中增加了对 lambda 的支持，使得 std::bind 过时，而 C++14 中，基本找不到使用 std::bind 的用例。

   **小结**

* lambda 可读性强，表达能力强，可能比使用 std::bind 更高效
* 在 C++11 中，std::bind 可能用于实现 **移动捕获** 或 绑定**模板化函数调用符的对象**

---

