# chapter 5.  Rvalue References, Move Semantics, and Perfect Forwarding

当第一次了解它们时，移动语义和完美转发看起来非常简单：

* **移动语义**使编译器可以用成本较低的移动替换昂贵的复制操作。与复制构造和赋值赋值运算符能赋予你控制拷贝对象的能力一样，move 构造函数以及 move 赋值运算符提供给你对 move 语义的控制。move 语义使得 move-only 类型的创建成为可能，比如说 std::unique_ptr，std::future 以及 std::thread。
* **完美转发**让我们可以写出接受任意参数的函数模板，并且将之转发到其他的函数，使得目标函数接受的参数和转发函数接受的参数相同

​	右值引用是将两个看起来毫不相关的两个概念联系起来的粘合剂。他们是使移动语义和完美转发成为可能的底层支持。

​	对这些功能的体验越多，越能意识到最初的认知只是基于众所周知的冰山一角。移动语义、完美转发和右值引用的世界比看起来更微妙。例如，std::move 不会移动任何东西，完美转发是不完美的。移动操作并不是总是比复制操作更低耗；当它们是时，它们并不总是向您期望的那样低耗；并且他们并不总是在移动有效的上下文中调用。构造 "type&&" 并不总是代表一个右值引用。

​	无论你深入研究这些功能有多远，似乎总有更多的东西有待发现。幸运的是，他们的深度是有限度的。本章将带你了解这些特征。例如，了解 std::move 和 std::forward 的常见用法，带着迷惑性的 type&& 用法对你来说变得很平常。你会理解移动操作各种让人感到奇怪的表现的原因。

​	在本章中的 Items 中，特别重要的是要记住参数始终是左值，即使它的类型是右值引用。也就是说，给定

```c++
void f(Widget&& w);
```

参数 w 是一个左值，即使它的类型是 rvalue-reference-to-Widget。（如果疑惑，请查看第二页左值和右值的概述）。

#### *item 23. 理解 std::move 和 std::forward*

​	通过理解 std::move 和 std::forward 不做什么来认识它们是非常有用的。std::move 不移动任何东西。std::forward 也不转发任何东西。在运行时，它们什么都不做。不产生可执行代码，一个比特的代码也不产生。

​	std::move 和 std::forward 只是执行转换的函数（确切的说应该是函数模板）。std::move 无条件的将它的参数转换成一个右值，而 std::forward 当特定的条件满足时，才会执行它的转换。这就是他们本来的样子。这样的理解产生了一些新的问题，但是，基本上就是这么回事。

​	下面是 C++11 的 std::move 的一种实现样例，虽然不能完全符合标准的细节，但是也非常相近了：

```c++
template<typename T>
typename remove_reference<T>::type&&
    move(T&& param)
{
    using ReturnType = 						// alias declaration;
	typename remove_reference<T>::type&&;	// see Item9
    return static_cast<ReturnType>(param);
}
```

​	以上代码中需要留意两个地方，一个是函数 move 返回类型，它非常具有迷惑性，千万别看蒙了。另一个是最后的转换，包含了 move 函数的本质。如你所见，std::move 接受一个对象的引用做参数（准确的说，应该是一个 通用引用）。请看 Item24。这个参数的格式是 T&& param，但是请不要误解为 move 接受的参数类型就是右值引用，请继续往下看），并且返回指向同一个对象的引用。

​	函数返回值的 "&&" 部分表明 std::move 返回的是一个右值引用。但是，真如 Item28 条解释的那样，如果 T 的类型恰好是一个左值引用，T&& 的类型就会是左值引用。为了阻止这种事情的发生，我们用到了类型特征（请看 Item9），在 T 上面应用 std::remove_reference，它的效果就是"去除" T 身上的引用，因此保证了 "&&" 应用到一个非引用的类型上面。这就确保了 std::move 真正返回的是一个右值引用，这很重要，因为从函数返回的右值引用是右值。因此 std::move 就做了一件事：将它的参数转换成了右值。

​	顺便说一句，std::move 可以在 C++14 中轻松实现。多亏了函数返回类型推导（参看Item3）和标准库的别名模板 std::remove_reference_t（参看Item9），std::move 可以这样写：

```c++
template<typename T>					// C++14; still in namespace std
decltype(auto) move(T&& param)
{
    using ReturnType = remove_reference_t<T>&&;
    return static_cast<ReturnType>(param);
}
```

看起来舒服多了，不是吗？

​	因为 std::move 只是将其参数强制转换为右值，所以有人建议它的名称应该类似于 rvalue_cast 的名称。尽管如此，我们的名字是 std::move，所以记住 std::move 做什么和不做什么很重要。它做转换但是不做移动。

​	当然，右值是移动的候选对象，因此将 std::move 应用于对象告诉编译器该对象有资源从中移动。这就是 std::move 具有名称的原因：使指定可以从中移动的对象变得容易。

​	事实上，右值通常只是移动的候选者。假设您正在编写一个表示注释的类。类的构造函数采用包含注释的 std::string 参数，并将参数复制到数据成员。受 Item41 影响，声明一个值传递参数：

```c++
class Annotation
{
  public:
    explicit Annotation(std::string text);		// param to be copied, so per Iterm41,
    											// pass by value
    ...  
};
```

​	但是 Annotation 的构造函数只需要读取文本的值。它不需要修改入参，根据一个历史悠久的传统：能使用 const 的时候尽量使用。你修改了构造函数的声明，text 改为 const：

```c++
class Annotation
{
  public:
    explicit Annotation(const std::string text);	// param to be copied
    ...												// so per Item41, pass by value
};
```

​	为了避免将文本复制给数据成员而产生复制消耗，仍然使用 Item41 的建议并将 std::move 应用于文本，从而产生一个右值：

```c++
class Annotation
{
  public:
    explicit Annotation(const std::string text)
	: value(std::move(text))					// "move" text into value; this
	...											// code doesn't do what it seems to!
  private:
    std::string value;
};
```

​	此代码编译，链接，运行。代码将数据成员的值设置为 text。这里唯一与你想象中不一样的点是 text 不是 move 给 value 数据成员，而是 copy 过去的。虽然 text 被 std::move 转换为右值，但 text 被声明为 const std::string，因此在转换之前，文本是左值 const std::string，并且转换的结果是右值 const std::string，但在整个过程中，常量性仍然存在。

考虑当编译器必须确定要调用哪个 std::string 构造函数时所产生的影响。有两个可能：

```c++
class string					// std::string is actually a typedef for
{								// std::basic_string<char>
  public:
    ...
	string(const string& rhs);	// copy ctor
    string(string&& rhs);		// move ctor
    ...
};
```

​	在 Annotation 构造函数的成员初始化列表中，std::moving(text) 的结果是一个 const std::string 类型的右值。该右值不能传递给 std::string 的传递给 std::string 的移动构造函数，因为移动构造函数采用对非常量 std::string 的右值引用。然而，因为 lvalue-reference-to-const 的参数类型可以被 const rvalue 匹配上，所以 rvalue 可以被传递给拷贝构造函数。因此，即使 text 被转换成了右值，上文中的成员初始化仍调用 std::string 的拷贝构造函数！这样的行为对于保持  const 的正确性是必须的。从一个对象里 move 出一个值通常会改变这个对象，所以语言不允许将 const 对象传递给可以修改他们的函数（例如移动构造函数）。

​	从本例中我们可以学到两点。首先，如果你想对这些对象执行 move 操作，就不要把它们声明为 const。对 const 对象的 move 请求通常会悄悄的执行到 copy 操作上。

​	std::forward 的情况和 std::move 类似，但是和 std::move 无条件地将它的参数转化为右值不同，std::forward 在特定的条件下才会执行转化。std::forward 是一个有条件的转化。为了理解它何时转化合适不转化，我们来回想一下 std::forward 的典型使用场景。最常见的场景是：一个函数模板接受一个通用引用参数，将它传递给另外一个函数（作参数）：

```c++
void process(const Widget& lvalArg);			// process lvalues
void process(Widget&& rvalArg);					// process rvalues

template<typename T>							// template that passes param to process
void logAndProcess(T&& param)
{
    auto now = std::chrono::system_clock::now();
    makeLogEntry("Calling 'process'", now);
    process(std::forward<T>(param));
}
```

如下一个左值调用，一个右值调用：

```c++
Widget w;

logAndProcess(w);				// call with lvalue
logAndprocess(std::move(w));	// call with rvalu
```

​	在 logAndProcess 的实现中，参数 param 被传递给了函数 process。process 对左值和右值进行了重载。当我们使用左值调用 logAndprocess 时，我们期望：forward 给 process 的也是一个左值，当我们用右值来调用 logAndProcess 时，我们希望 process 的右值重载被调用。

​	但是 param 和其他所有函数参数一样，是一个左值。因此，每次调用 logAndProcess 中的 process 都调用的是 process 的左值重载。为了防止这种情况，我们需要一种机制，当且仅当初始化 param 的参数（传递给 logAndProcess 的参数是右值）时，才能将 param 强制转换为右值。这正是 std::forward 所要做的。这就是 std::forward 是一个条件转换的原因：只要当它的参数用右值初始化时，它才转换为右值。

​	你可能想知道 std::forward 怎么知道它的参数是否被一个右值初始化。比如说，在以上的代码中，std::forward 怎么知道 param 被一个左值或者右值初始化？答案很简单，这个信息蕴藏在 logAndProcess 的模板参数 T 中。这个参数传递给 std::forward，然后 std::forward 来从中解码出此信息。欲知详情，参看 Item28。

​	std::move 和 std::forward 都可以归之为转换。唯一不同的一点是，std::move 总是执行转换，而 std::forward 是在某些条件满足时才做。你可能觉得我们不用 std::move，只使用 std::forward 会不会还一些。从一个纯技术角度来说，答案是肯定的：std::forward 是可以都做了，std::move 不是必须的。当然，可以说这两个函数都不是必须的，因为我们可以在任何地方都直接写转换代码，但这显然太恶心！

​	std::move 的吸引力在于方便、减少出错的可能性和更清晰。考虑我们想要跟踪移动构造函数被调用次数的类。我们只需要一个在移动构造期间递增的静态计数器。假设类中唯一的非静态数据是 std::string，这里是实现移动构造函数的传统方式（即使用 std::move）:

```c++
class Widget
{
  public:
    Widget(Widget&& rhs):s(std::move(rhs.s))
    {
        ++moveCtorCalls;
    }
    ...
  private:
    static std::size_t moveCtorCalls;
    std::string s;
};
```

要使用 std::forward 实现相同的行为，代码将如下所示：

```c++
class Widget
{
  public:
    Widget(Widget&& rhs): s(std::forward<std::string>(rhs.s))	// unconventional,
    {															// undesirable
        ++moveCtorCalls;										// implementation
    }
    ...
};
```

​	首先请注意，std::move 只需要一个函数参数（rhs.s），而 std::forward 需要一个函数参数（rhs.s）和一个模板类型参数（std::string）。然后请注意，我们传递给 std::forward 的类型应该是非引用，因为这是编码传递的参数是右值的约定（参数看Item28）。综上，这意味着 std::move 比 std::forward 使用起来更方便（至少少敲了不少字），免去了让我们传递一个表示函数参数是否是一个右值的参数类型。消除了我们传递错误类型（比如说，传一个 std::string& ，会导致数据成员 s 被拷贝构造，而不是想要的移动构造）的可能性。

​	更重要的是，std::move 的使用表明了对右值的无条件转换，而 std::forward 只对绑定了右值的引用进行转换。这是两个非常不同的行为。std::move 就是为了移动操作而生的，而 std::forward，就是将一个对象转发（或者说传递）给另外一个函数，同时保留此对象的左值性或右值性（lvalueness or rvalueness）。所以我们需要两个不同的函数（并且是不同函数名字）来区分这两个操作。

   **小结**

* std::move 表现为无条件转换为右值。就其本身而言，它不会移动任何东西
* 当且仅当参数绑定到右值时，std::forward 才将其参数强制转换为右值
* std::move 和 std::forward 在运行时都不做任何事情

---

#### *item 24. 区分通用引用和右值引用*

​	为了声明一个类型T的右值，你写下了 T&&。下面的假设看起来合理：你在代码中看到一个 "T&&" 时，你看到的就是一个右值引用。但是，它可没想象中那么简单：

```c++
void f(Widget&& param);					// rvalue reference
Widget&& var1 = Widget();				// rvalue reference
auto&& var2 = var1;						// not rvalue reference

template<typename T>
void f(std::vector<T>&& param);			// rvalue reference

template<typename T>
void f(T&& param);						// not rvalue reference
```

​	事实上，"T&&" 有两个不同的意思。首先，当然是作为右值引用，这样的引用表现起来和预期一致：只和右值做绑定，它们存在的意义就是表示出可以从中移动的对象。

​	"T&&" 的另外一个含义是：既可以是右值引用也可以是左值引用。这样的引用在代码里看起来像是右值引用（即T&&)，但是他们也可以表现得就像它们是左值引用（即T&）那样。它们的双重性质允许他们绑定到右值（如右值引用）和左值（如左值引用）。此外，它们可以绑定到 const 或者 non-const，volatile 或 non-vaolatile，甚至是 const + volatile 对象上面。它们几乎可以绑定到任何东西上面。为了对得起它的全能，我们称之为通用引用。

通用医用出现在两种情况下。最常见的是函数模板参数，例如上面示例代码中的这个例子：

```c++
tem   plate<typename T>
void f(T&& param);				// param is a universal reference
```

第二个上下文是自动声明，包括上面示例代码中的这个：

```c++
auto&& var2 = var1;				// var2 is a universal reference
```

这些上下文的共同点是存在类型推导。在模板 f 中，正在推导 param 的类型，而在 var2 的声明中，正在推导 var2 的类型。将其与一下示例（也来自上面的示例代码）进行比较，其中缺少类型推导。如果你看到没有类型推导的"T&&"，则你正在查看右值引用：

```c++
void f(Widget&& param);					//no type deduction; param is an rvalue reference
Widget&& var1 = Widget();				//no type deduction; var1 is an rvalue reference
```

因为通用引用是引用，所以它们必须被初始化。通用引用的初始值设定项确定它是表示右值引用还是左值引用。如果初始值设定项是右值，则通用引用对应于右值引用。如果初始值设定项是左值，则通用引用对应于左值引用。对于作为函数参数的通用引用，调用处提供了初始化程序：

```c++
template<typename T>
void f(T&& param);					// param is a universal reference

Widget w;							// lvalue passed to f; param's type is Widget&
f(w)								// (i.e., an lvalue reference)

f(std::move(w));                    // rvalue passed to f; param's type is Widget&&
									// (i.e., an rvalue reference)
```

要使引用具有通用性，类型推导是必要的，但这还不够。引用声明的形式也必须是正确的，它要求的格式也很严格，必须是"T&&"。再看之前的例子：

```c++
template<typename T>
void f(std::vector<T>&& param);			// param is an rvalue reference
```

当 f 被调用时，类型 T 会被推导出来(除非调用者明确指定它，一个我们一般不会关心的比较极端的情况)。但是 param 的类型声明形式不是"T&&"，而是"std::vector<T>&&"。这排除了 param 是通用引用的可能性。因此，param 是一个右值引用，如果尝试将左值传递给 f，编译器会直接报错：

```c++
std::vector<int> v;
f(v);									// error!can't bind lvalue to rvalue reference
```

即使是 const 限定符的简单存在也足以取消引用的通用性：

```c++
template<typename T>
void f(const T&& param);				// param is an rvalue reference
```

如果在模板中看到一个类型为 "T&&" 的函数参数，你可能认为它是一个通用引用。但是并不能单纯的这么认为。那是因为在模板中并能保证类型推导的存在。考虑 std::vector 中 push_back 成员函数：

```c++
template<class T, class Allocator = allocator<T>>				// from c++ Standards
class vector{
public:
  void push_back(T&& x);
  ...
};
```

push_back 的参数具有通用引用的正确形式，但是这种情况下没有类型推导。这是因为如果没有 vector 的实例化，push_back 就无法存在，并且该实例化的类型完全决定了 push_back 的声明。如下情况:

```c++
std::vector<Widget> v;
// 导致std:;vector模板被实例化如下:
class vector<Widget, allocator<Widget>>{
public:
  void push_back(Wigdet&& x);			// rvalue reference
  ...
};
```

如此来看 push_back 没有使用类型推导。vector<T> 的 push_back（有两个--函数重载）总是声明一个类型为 rvalue-reference-to-T 的参数。

相比之下，std::vector 中类似的 emplace_back 成员函数确实使用了类型推导：

```c++
template<class T, class Allocator = allocator<T>> 	// still from C++ Standards
class vector<{
  public:
    template<class... Args>
	void emplace_back(Args&&... args);
    ...
};
```

​	这里，类型参数 Args 与 vector 的类型参数 T 无关，因此每次调用 emplace_back 时都必须推导出 Args。(好吧，Args 确实是一个参数包，而不是一个类型参数，但出于本次讨论的目的，我们可以将其视为类型参数。)

​	之前说通用引用格式必须是 "T&&"，emplace_back 的类型参数被命名为 Args，但这不影响 args 是一个通用引用，管它叫做 T 还是叫做 Args，没啥区别。举个例子，下面的 template 接受的参数就是通用引用。一是因为格式是"type&&"，而是因为 param 的类型会被推导（再提一下，除非调用者显式指明类型这种极端的情况）。

```c++
template<typename MyTemplateType>				// param os universal reference
void someFunc(MyTemplateType&& param);
```

​	之前说过 auto 变量也可以是通用引用。确切的说，使用 auto&& 类型声明的变量是通用引用，因为发生了类型推导而且他们具有正确的形式（T&&）。auto 通用引用不像用于函数模板参数的通用引用常见，但是它们在c++11中不时出现。他们在 C++14 中出现的更多，因为 C++14 lambda 表达式可能会声明 auto&& 参数。例如，如果您想编写一个 C++14 lambda 来记录任意函数调用所花费的时间，可以这样：

```c++
auto timeFuncInvocation = 
    [](auto&& func, auto&&... params)				// C++14
	{
    	start timer;
    	std::forward<decltype(func)>(func)(
        	std::forward<decltype(params)>(params)...	// invoke func of params
            );
    	stop timer and record elapsed times;
	};
```

​	如果对上例中 "std::forward<decltype(blah blah blah)>" 代码感到疑惑可以从参看 Item33。但是不用太在意，这里是重点讲解 lambda 声明 auto&& 参数。func 是一个通用引用，可以绑定到任何可以调用对象，左值或者右值。args 是零个或多个通用引用（即通用引用参数包），可以绑定到任意数量的任意类型的对象。由于 auto 通用引用，timeFuncInvocation 可以记录几乎所有函数的运行时长。（有关"any"和"pretty much any",可以参数看Item30）。----事实就是--------》**引用折叠**

**小结**

* 如果函数模板参数对于推到的类型 T 具有类型 T&&，或者如果使用 auto&& 声明对象，则该参数或者对象是通用引用
* 如果类型声明的形式不是精确的 type&&，或者如果没有发生类型推导，type&& 表示一个右值引用
* 通用引用对于右值引用，如果它们是用有值初始化的。如果它们用左值初始化，它们对应于左值引用。

----

#### *item 25. 对右值引用使用 std::move,对通用引用使用 std::forward*

右值引用仅绑定到可移动的对象，如果你有一个右值引用参数，你就知道它所绑定的对象可能会被移动：

```c++
class Widget{
	Widget(Widget&& rhs);		// rhs definitely refers to an
    ...							// object eligible for moving
};
```

在这种情况下，需要以允许这些函数利用对象的右值的方式将此类对象传递给其他函数。这种做法是将绑定到此类对象的参数转换为右值。正如 Item23 所描述，这不仅是 std::move 的作用，也是它的创建目的：

```c++
class Widget{
  public:
    Widget(Widget&& rhs)					// rhs is rvalue reference
	: name(std::move(rhs.name)),p(std::move(rhs.p))
    {...}
    ...
  private:
    std::string name;
    std::shared_ptr<SomeDataStructure> p;
};
```

另一方面，通用引用（参看 Item24）可能绑定到一个可以移动的对象。仅当使用右值初始化通用引用时，才应将其强制转为右值。Item 23 解释了这正是 std::forward 所做的：

```c++
class Widget{
  public:
	template<typename T>
    void setName(T&& newName)				// newName is 
    {name = std::forward<T>(newName);}		// univalsal reference
    ...
};
```

​		简而言之，当将右值引用转发到其他函数时，它们应该无条件地转换为右值（通过 std::move），因为它们总是绑定到右值，而通用引用应该有条件地转发为右值（通过 std::forward）转发它们时，因为它们只是有时绑定到右值。

​		Item23 解释了在右值引用上使用 std::forward 可以表现出正确的行为，但是代码冗长、容易出错且单调，因此应该避免将 std::forward 与右值引用一起使用。更糟糕的是将 std::move 与通用引用一起使用的想法，可能会导致意外修改左值（例如，局部变量）：

```c++
class Widget{
  public:
    template<typename T>
    void setName(T&& newName)		// univarsal reference compiles,but is bad,bad,bad!
    {name = std::move(newName);}
    ...
  private:
    std::string name;
    std::shared_ptr<SomeDataStructure> p;
};

std::string getWidgetName();		// factory function
Widget w;
auto n = getWidegetName();			// n is local variable
w.setName(n);						// moves n intot w!
...									// n's value now unknow
```

​		你可能会说 setName 肯定不应该将其参数声明为通用引用。这样的引用不能是 const（见 Item 24），但 setName 肯定不应该修改它的参数。你可能会指出，如果 setName 存在 const 左值和右值的重载，则可以避免整个问题。像这样：

```c++
class Widget{
  public:
    void setName(const std::string& newName)	// set from const lvalue
    { name = newName; }
    void setName(std::string&& newName)
    { name = std::moving(newName); }
    ...
};
```

​		在这种情况下肯定是没问题的，但是还是有缺陷。首先，需要编写和维护更多的源代码（两个函数而不是一个模板）。其次，可能效率较低。例如，考虑 setName 的这种用法：

```c++
w.setName("Adela Novak");
```

​		在 setName 通用引用版本中，字符串 "Adela Novak"  传递给 setName 后， w 内部的name 数据成员通过 std::string 的赋值运算符接口直接创建，不会创建临时变量；然而，在 setName 的重载版本中，会为 setName 的参数创建一个临时的 std::string 对象来绑定，然后这个临时的 std::string 将被移动到 w 的数据成员中。因此，对 setName 的调用将需要执行一个 std::string 构造函数（一创建临时对象）、一个 std::string 移动赋值运算符（将 newName 移入 w.name ）和一个 std::string 析构函数（销毁临时对象）。这明显比尽调用带有 const char* 指针的 std::string 赋值运算符消耗要高。具体的性能消耗可能因实现而异，但事实是，用一对重载的函数替换采用通用引用的模板在某些情况下，左值引用和右值引用可能会产生运行时成本。如果我们实例中 Widget 数据成员是任意类型（而非已知的 std::string）,则性能差距会显著扩大，因为并非所有类型都像 std::string 那样容易移动（看 Item29）。

​		此外，重载左值和右值最严重的问题不是源码数量或惯用性，也不是代码运行时性能，而是**可扩展性差**。Widget::setName 只接受一个参数，所以只需要两次重载，但是对于接受多个参数的函数，每个参数都可以是左值或右值，重载的数量呈几何级数增长：n 个参数需要能够 2^n 个重载。这还不是最糟糕的，一些函数--实际上是函数模板--接受无限数量的参数，每个可以是左值或右值。此类函数的典型例子是 std::make_shared，以及从 C++14 开始的 std::make_unique（参见 Item21）。来看看他们最常用的重载声明：

```c++
template<class T, class... Args>			// from C++11 Standard
shared_ptr<T> make_shared(Args&&... args);

template<class T, class... Args>			// from C++14 Standard
unique_ptr<T> make_unique(Args&&... args);
```

​		对于这样的函数，重载左值和右值不可行，必须使用通用引用。在这些函数中，如果通用引用参数传递给其他函数时，我们应该使用 std::forward 进行转发。

​		在某些情况下，你可能需要在单个函数中多次使用绑定到右值引用或通用引用的对象，并且你想在完成操作前不移动它。这种情况下，你会在最后才使用 std::move（右值引用）或 std::forward（通用引用）。

```c++
template<typename T>
void setStringText(T&& text)		// textt is univ. reference
{
    sign.setText(text);				// use text, but don't modify it
    
    auto now = std::chrono::system_clock::now();	// get current time
    
    signHistory.add(now, std::forward<T>(text));	// conditionally cast text tp rvalue
}
```

​		例子中，我们要确保 text 的值不会被 sign.setText 更改，因为我们想在调用 signHistory.add 时使用该值。因此，仅在最终使用通用引用时才使用 std::forward。

​		对于 std::move 也是这样（将 std::move 应用于最后一次使用的右值引用），但需要注意的是，在极少情况下，你需要调用 std::move_if_noexcept 来替代 std::move。是何原因请参看 Item14。

​		如果有一个按值返回的函数，并且返回值对象绑定到了右值引用或通用引用，那么需要在返回引用时应该使用 std::move 或 std::forward。通过一个例子了解原因，假如使用 operator+ 函数将两个矩阵相加，其中已知左侧矩阵是右值（因此可以重用它来保存矩阵的和）：

```c++
Matrix operator+(Matrix&& lhs, const Matrix& rhs)		// by-value return
{
    lhs += rhs;
    return std::move(lhs);								// move lhs into return value
}
// 通过使用 std::move 将 lhs 转换为右值移动到函数返回值
// 如果不使用 std::move 如下
Matrix operator+(Matrix&& lhs, const Matrix& rhs)		// as above
{
    lhs += rhs;
    return lhs;											// copy lhs into return value
}
```

​		上例中不使用 std::move 时，lhs 是一个左值，编译器会将其赋值给返回值。如果 Matrix 类型支持移动构造，那么在 retrurn 语句中使用 std::move 会产生更高效的代码。如果 Matrix 不支持移动，将其转换为右值也不会有什么坏处，因为右值将触发 Maxtrix 的复制构造函数（参看 Item23）。对 Matix 稍加修改也会支持移动，下次编译也会自动生效。

​		通用引用跟和 std::forward 的情况类似。假设一个函数 reduceAndCopy，接受一个 unredused Fraction 对象，将其 reduce ，然后返回处理后的对象的副本。如果原始对象是一个右值，则应将其值移动到返回值（避免创建副本的消耗），但是如果原始对象是左值，则必须创建实际的副本。如下：

```c++
template<typename T>
Fraction reduceAndCopy(T&& frac)			// by-value return universal reference param
{
    frac.reduce();
    return std::forward<T>(frac);			// move rvalue into return value, copy lvalue
}
```

​		如果不使用 std::forward，frac 会无条件的复制到 reduceAndCopy 的返回值中。有人可能会说是否可对返回的局部变量进行相同的优化？如下：

```c++
Widget makeWidget()			// "Copying" version of make Widget 
{
    Widget w;				// local varable
    ...						// configure w
	return w;				// "copy" w into return value
}
// "优化"
Widget makeWidget()			// Moving version of makeWidget
{
    Widget w;
    ...
	return std::move(w);	// move w into return value (don't do this!!)
}
```

​		以上的"优化"明显是有缺陷的。这是因为标准委员会通过为函数返回值分配的内存中构建它来避免复制局部变量 w。这就是RVO（return value optimization）。 所以每个像样的 C++ 编译器都会使用 RVO 来避免复制 w。也就是说 makeWidget 的 "Copying" 版本实际上不会复制任何内容。

​		makeWidget 的"移动"版本（假设Widget提供了移动构造函数）：它将 w 的内容移动到 makeWidget 的返回值。但是为什么编译器不适用RVO来消除移动，再次在分配给函数返回值的内存中构建 w 呐？答案很简单：它们做不到，因为只有当返回的是局部对象时才会执行 RVO，这不会在 makeWidget 的移动版本中生效。而在移动版本中返回的不是局部变量w，而是对 w 的引用--std::move(w) 结果。返回对局部变量的引用不满足 RVO 所需条件，因此编译器必须将 w 移动到函数的返回值。试图通过 std::move 来避免复制局部变量的做法，实际上限制了编译器的优化选项。

​		另外，针对满足 RVO 条件的环境，编译器要么执行复制省略，要么将 std::move 隐式应用于正在返回的局部变量（也就类似我们所写的"移动版本"）。

​		值传递的函数参数情况也是类似。对于函数返回值，它们不符合复制省略的条件，但是如果它们返回，编译器会将它们视为右值。如下代码：

```c++
Widget makeWidget(Widget w)		// by-value parameter of same type as function's return
{
    ...
	return w;
}

/// 编译器优化
Widget makeWidget(Widget w)
{
    ...
	return std::move(w);		// treat w as rvalue
}
```

​		但是在某些情况下，将 std::move 应用于局部变量可能是一件合理的事情（即，当将它传递个一个函数并且你知道将不再使用该变量时）。

**小结**

* 在上下文最后，将 std::move 应用于右值引用，std::forward 应用于通用引用
* 对从按值返回的函数返回的右值引用和通用引用执行相同的操作
* 切勿将 std::move 或 std::forward 应用于局部变量，如果它们满足 RVO 优化

---

#### *item 26. 避免重载通用引用*

​		假设你需要编写一个函数，将函数名作为参数，记录当前日期和时间，然后将该名称添加到全局变量机构中。你可能会想出这样一个函数：

```c++
std::multiset<std::string> names;			// global data structure
void logAndAdd(const std::string& name)
{
    auto now =								// get current time
        std::chrono::system_clock::now();
    log(now, "logAndAdd");					// make log entry
    names.emplace(name);					// add name to global data
    										// structure; see Item 42 for info on emplace
}
////这不是不合理的代码，但是它没有达到应用的效率。考虑三个潜在的调用
std::string petName("Darla");
logAndAdd(petName);						// pass lvalue std::string
logAndAdd(std::string("Persephone"));	// pass rvalue std::string
logAndAdd("Patty Dag");					// pass string literal
```

​		第一个调用中，logAndAdd 的参数名绑定到变量 perName。在 logAndAdd 中，name 最终会传递给 names.emplace。因为 name 是一个左值，所以他被复制到 names 中。没有办法避免这个副本，因为左值 petName 被传递到了 logAndAdd。  

​		第二个调用中，参数 name 绑定到一个右值（从 "Persephone" 显示的临时 std::string）。name  本身是一个左值，因此它被复制到 names 中。这次调用中，我们产生了拷贝，但是可以通过只移动完成。

​		第三个调用中，参数 name 也是绑定到一个右值，但这次绑定到一个从 "Patty Dag"  隐式创建的临时 std::string。在第二次调用中，name 被复制到 names 中，但在这种情况下，最初传递给 logAndAdd 的参数是字符串文本。如果将该字符串文本直接传递给 emplace，则根本不需要创建临时 std::string。相反，emplace 会使用字符串文本直接在 std::multiset 内部创建 std::string 对象。所以，第三个调用中也存在复制的消耗，事实上我们都不用有移动的消耗，更何况复制。

​		我们可以通过重写 logAndAdd 入参通用引用（参看Item24），根据 Item25 使用 std::forward 转发引用给 emplace 来消除第二次和第三次调用中低效率问题：

```c++
template<typename T>
void logAndAdd(T&& name)
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}
std::string petName("Darla");		// as before
logAndAdd(petName);					// as before, copy lvalue into multiset
logAndAdd(std::string("Persephone"));	// move rvalue instead of copying it
logAndAdd("Patty Dog");				// create std::string in multiset instead of copying
									// a temporary std::string
```

​		现实中 logAndAdd 的入参可能并不会直接访问 name，例如通过库表查找对应的 name，如下 logAndAdd 重载：

```c++
std::string nameFromIdx(int x);		// return name corresponding to idx
void logAndAdd(int idx)
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(nameFromIdx(idx));
}

//调用
std::string petName("Darla");		// as before
logAndAdd(petName);						// as before, these calls
logAndAdd(std::string("Persephone"));	// all invoke the
logAndAdd("Patty Dog");					// T&& overload

logAndAdd(22);							// calls int overload

// 一些特殊的情况
short nameIdx;
...									// give nameIdx a value
logAndAdd(nameIdx);					// error!
```

​		上面代码最后一行报错为啥？我们来看一下，在两个 logAndAdd 的重载中，使用通用引用的重载可以推出 T 是 short，产生精确匹配。int 版本的重载必须将 short 转换为 int 才能匹配。根据正常的重载解析规则，精确匹配胜过通过转换的匹配，因此调用了通用引用的重载。

​		在该重载中，参数 name 绑定到传入的 short。然后将 name 转发到 names 的 emplace 成员函数（std::multiset\<std::string>），然后将其转发到 std::string 构造函数。但是 std::string 的没有入参 short 的构造函数，所以会失败。

​		采用通用引用的函数是 C++ 中最贪婪的函数。它们实例化可以实例化为几乎任何类型的参数创建精确匹配。（在 Item30 中描述了几种特殊的情况）因为通用引用匹配的类型超多，所以将通用引用应用于重载不是一个好的设计方案。

​		要避免这一点也很简单，只需要对 logAndAdd 进行 forward 转发这样小小的改动。这里假设存在相同问题的一个类Person如下：

```c++
class Person{
  public:
    template<typename T>									// perfect forwarding ctor;
    explicit Person(T&& n): name(std::forward<T>(n)) {}		// initializes data member
    
    explicit Person(int index): name(nameFromIdx(idx)) {}	// int ctor
    ...
  private:
    std::string name;
};
```

​		与 logAndAdd 的情况一样，传递 int 以外的整数类型（例如，std::size_t、short、long等）将会调用通用引用构造函数重载而不是 int 重载，这将导致编译失败 。然而，这里的问题更糟糕，因为 Person 中存在自动生成的函数。Item17 中阐述了在适当的条件下，C++ 将生成 copy 和 move 构造函数，即使类包含一个模板化的构造函数，被实例化后也会生成复制或移动构造函数。如果 Person 存在复制和移动构造函数，Person 可能看起来如下：

```c++
class Person{
public:
    template<typename T>				// perfect forwarding ctor
    explicit Person(T&& n): name(std::forward<T>(n)) {}
    
    explicit Person(int idx);			// int ctor
    
    Person(const Person& rhs);			// copy ctor(compiler-generated)
    
    Person(Person&& rhs);				// move ctor(compiler-genetated)
    ...
};

// 如下使用案例
Person p("Nancy");
auto cloneOfP(p);						// create new Person from p; this won't compile!
```

​		在这里，我们试图从另一个 Person 创建一个 Person，这是最明显的复制构造案例。（p 是一个左值，所以我们完全不用顾虑它会调用移动构造）但是这段代码并不会调用复制构造函数，而是调用完美转发构造函数。然后，该函数将尝试使用 Person 对象来初始化内部成员 std::string，这时编译器会抛出一大堆难以理解的错误信息。原因如下，（编译器的匹配规则）因位 p 是一个非常量左值，将模板化构造函数实例化为入参 Person 类型的非常量左值可以精确匹配调用。实例化后的 Person 看起来如下：

```c++
class Person{
public:
    explicit Person(Person& n)			// instantiated from perfect-forwarding template
	: name(std::forward<Person>(n)){}
    
    explicit Person(int idx);			// as before
    
    explicit Person(const Person& rhs);	// copy ctor(compiler-generated)
    explicit Person(Person&& rhs);		// move ctor(compiler-generated)
    ...
};

// 知道原理后，稍加改动就会变为我们期待的调用方式
const Person cp("Nancy");		// object is now const
auto cloneOfP(cp);				// calls copy constructor!
```

虽然模板化构造函数可以实例化为如下:

```c++
class Person{
public:
    explicit Person(const Person& n);		// instantiated for template
    Person(const Person& rhs);				// copy ctor(compiler-generated)
};
```

 		但是并不调用它，因为 C++ 中重载解析规则之一是，当存在模板实例化函数和非模板函数（即，"正常"函数）都匹配调用时，"正常"函数的优先级高。

(如果想了解为什么编译器会在实例化一个模板化的构造函数时生成复制构造，可以参看 Item17)

```c++
class SpecialPerson:public Person{
public:
    SpecialPerson(const SpecialPerson& rhs)	// copy ctor; calls base class
	: Person(rhs)							// forwarding ctor!
    {...}
    
    SpecialPerson(SpecialPerson&& rhs)		// move ctor; calls base class
	: Person(std::move(rhs))				// forwarding ctor
    {...}
};
```

​		正如注释中阐述的那样，拷贝和移动构造在基类中都是调用的完美转发构造函数（原理跟上述调用完美转的情况类似，只是入参变为 SpecialPerson）。

​		讲这么多其实就是想让你避免重载通用引用参数。但是，如果我们确实需要一个转发大多数参数类型的函数那我们该怎么办呐？Item 27将会继续讲解。

 **小结**

* 通用引用的重载几乎总会导致通用引用重载被调用频率高于预期
* 完美转发构造函数问题更为糟糕，因为他们通常比非常量左值的复制构造函数更匹配，并且它们可能会在派生类调用基类复制构造和移动构造时"偷偷"被执行

---

#### *item 27. 熟悉重载通用引用的替代方案*

​	Item26 阐述了重载通用引用会导致各种问题，无论是独立函数还是成员函数（尤其是构造函数）。但是也给出了通用引用有用的例子，如果他能像我们想象中那样运行就完美了。本章通过避免重载通用引用的设计，或者限制匹配的参数类型等方法来实现我们所需的行为。下面的讨论基于在 Item26 中介绍的示例。如果最近没有阅读过该章，那请再复习一下吧。

*<u>**放弃重载**</u>*

Item26 中第一个示例 logAndAdd 就是一个典型的代表，像这样的函数通过简单的修改函数名可以规避重载的使用。如，两个 logAndAdd 重载可以拆解为 logAndAddName 和 logAndAddNameIdx。但是这个方法不适用第二个例子，即 Person 构造函数，因为构造函数名称是固定的。

***<u>入参 const T&</u>***

另一种方法是恢复到 C++98 使用 入参左值常量引用 替代 入参通用引用。缺点是没有我们预期的那样高效，放弃效率来保证准确性。

***<u>按值传递</u>***

通过 值传递 替换 传递引用 不会增加程序复杂性。这种设计遵循 Item41 中的建议，在知道将复制对象时考虑按值传递对象，这里详细讨论其工作原理及其效率。接下来，展示如何在 Person 示例中使用该技术：

```c++
class Person{
  public:
    explicit Person(std::string n)	// replaces T&& ctor; see
    : name(std::move(n)){}          // Item41 for use of std::move
    
    explicit Person(int idx)		// as before
    : name(nameFromIdx(idx)){}
    ...

  private:
    std::string name;
};
```

因为除了入参 std::string 的构造函数外，就只剩入参 int 。所以 Person 构造函数的所有 int 和 类似 int 的参数（如，std::size_t、short、long）都被传给 int 重载。同理，所有 std::string 类型的参数（以及可以创建 std::string 的参数，如"Ruth"常量字符串）都被传递给入参 std::string 的构造函数。针对 0 和 NULL 的使用参照 Iterm 8 的使用规范使用来规避歧义。

***<u>使用标签调度</u>***

​		入参左值常量引用 和 值传递 都不支持完美转发。如果使用通用引用的动机是完美转发，我们就必须使用通用引用没有其他选择。那么如果不放弃重载也不放弃通用引用，我们如何避免对通用引用进行重载呐 ？

​		其实也没有那么难。对重载函数的调用是通过查看所有重载的所有参数以及调用点的所有参数来决定的，然后选择具有最佳整体匹配的函数-考虑所有参数、参数组合。通用引用参数通常会为任何入参提供精确匹配，但如果通用引用是*参数列表*的一部分，并且参数列表包含其他非通用引用参数，因为非通用引用的存在会导致通用引用的匹配不会被执行。这就是标签调度的实现基础，来看 Item26 中 logAndAdd 的一个示例：

```c++
std::multiset<std::string> names;		// global data structure
template<typename T>					// make log entry and add name to data structure
void logAndAdd(T&& name)
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}
```

​		就其本身而言，这个函数工作正常，但是如果我们引入一个入参 int 的重载函数，就会出现 Item26 中 short 入参匹配通用引用的情况。本章我们将重新实现 logAndAdd 以委托给两个函数，一个用于处理 int，另一个用于处理所有类型入参，而不是添加重载。logAndAdd 本身将接受所有参数类型，包括整数和非整数。

​		实际工作的两个函数将被命名为 logAndAddImpl，即我们使用重载。其中一个函数将采用通用引用。这里我们即使用了重载有使用了通用引用。但是每个函数也必将包含第二个参数，该参数指示传递的参数是否为整数，这个参数将决定我们到底是用哪个重载。

```c++
template<typename T>
void logAndAdd(T&& name)
{
    logAndAddImpl(std::forward<T>(name), std::is_integral<T>()); 	// not quite correct
}
```

​		这个函数将其参数转发给 logAndAddImpl，而且还传递一个参数，用来指示该参数的类型（T）是否为整形。但是，Item28 中阐述，如果将左值参数传递给通用引用 name ，则 T 会被推导为左值引用。因此，如果将 int 类型的左值传递给 logAndAdd，T 将被推导为 int&。这就意味着 std::is_integral<T> 对于任何左值参数都是 false，即使该参数确实表示一个整数。

​		意识到问题的出处那就好解决了，因为标准 C++ 库中有一个类型特征 std::remove_reference，它可以从一个类型中移除引用。再来看看优化后的代码：

```c++
template<typename T>
void logAndAdd(T&& name)
{
	logAndAddImpl(
        std::forward<T>(name),
        is_integral<typename std::remove_reference<T>::type>()
    );
}
// C++14 中可以使用 std::remove_reference_t<T>() 来替代以上代码，来减少代码冗余
```

接下来将目光移至 logAndAddImpl 的实现：******

```c++
// 匹配非整型类型 std::is_integral<typename std::remove_reference<T>::type 为 false
template<typename T>
void logAndAddImpl(T&& name, std::false_type)
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}

// 匹配整形类型
std:: string nameFromIdx(int int);			// as in Item26
void logAndAddImpl(int idx, std::true_type)
{
	logAndAdd(nameFromIdx(idx));
}
```

​		以上设计中，类型 std::true_type 和 std::false_type 就是"标签"，唯一的目的就是强制重载解析按照我们想要的方式进行。请注意，我们甚至没有命名这些参数。这个参数在运行时不会有任何作用，事实上我们希望编译器能认识到这些标签参数是无用的并且希望编译器可以优化。（有些编译器会这样做，至少有时会）通过创建正确的标签对象调用 logAndAdd 中正确的重载分支。因此，这种设计称之为：标签调度。它是模板元编程的标准构建块，在 C++ 库中可以频繁看到它的身影。

​		我们的目的不在于强调标签调度的原理，而是它允许结合使用通用引用和重载但不会出现 Item26 中描述的问题。分支函数 logAndAdd 接受一个不受约束的通用引用参数，但是这个函数没有重载。logAndAddImpl 函数是重载的，并且带有一个通用引用参数，函数的具体调用不仅取决于通用引用参数，还取决于标签参数，并且标签值被设计为不产生多个可行匹配。因此，标签决定了调用哪个重载。通用引用参数始终会精确匹配其参数类型的调用。

***<u>约束采用通用引用的模板</u>***

​		标签调度的一个关键是只需存在单个（未重载）函数作为客户端API。这个单一的函数将完成的工作分配给指定的函数。创建非重载的'"分支"函数很容易，但是 Item26 中第二个案例，即 Person 类的完美转发构造函数是一个例外。编译器可能会自己生成复制和移动构造函数，因此即使你只编写一个构造函数并且在其中使用了标签调度，一些构造也可能是调用了编译器生成的构造函数，绕过标签调度（自定义的构造）。

​		其实真正的问题不是因为没有调用标签构造函数，恰恰相反，而是调用了它。对于一个类的左值请求，我们几乎总是希望调用类的复制构造函数来处理，但是，如 Item26 所示，提供一个接受通用引用的构造函数会导致入参非常量左值（non-const lvalues）时调用通用引用构造函数（而不是复制构造函数）。Item26 还解释了当基类声明完美转发构造函数并且派生类以传统方式实现复制和移动构造函数，则调用派生类复制或移动构造函数时会调用基类的完美转发构造函数（正常应该调用基类的复制或移动构造）。

​		对于像这样的情况，如果重载函数接受一个通用引用比想象中更贪婪，但是针对单分支调度的函数有没有那么贪婪。这里了解一种不同的技术，这种技术可以让你记下包含通用引用的函数模板所允许使用的条件。这时我们就需要使用 std::enable_if。

​		std::enable_if 可以强制编译器表现的好像某个模板不存在。这样的模板我们称之为禁用。默认情况下所有的模板都是启用的。我们这里提到的例子希望仅在传递的类型不是 Person 的情况下才调用 Person 完美转发构造函数。如果传递的类型是 Person，我们希望禁用完美转发构造函数（如，编译器忽略它），这样就会去调用复制或者移动构造函数被调用，这是我们在使用 Person 对象初始化另外一个 Person 时所希望的。因为 std::enable_if 不会影响实现，所以使用它实现起来不会很复杂：

```c++
class Person{
  public:
    template<typename T, typename = typename std::enable_if<condition>::type>
        explicit Person(T&& n);
    ...
};
```

​		这里我们重点讨论控制是否启用该构造函数的条件表达式，想了解更多 std::enable_if 的原理和细节可以查看相关资料。

​		我们想要指定的条件是 T 为非 Person，也就是说，只有当 T 是 Person 以外的类型时，才应该启用模板化构造函数。由于有一个类型是否相同(std::is_same)，所以我们想要的条件应该是 !std::is_same<person, T>::value。这接近我们的需求，但并不完全正确，因为，Item28 阐述用左值初始化的通用引用推导出的类型总是左值引用。这意味着对于这样的代码，

```c++
Person p("Nancy")'
auto cloneOfP(p);		// initialuze from lvalue
```

通用构造函数中的类型 T 将被推导为 Person&。类型 Person 和 Person& 不相同，std::is_same<Person, Person&>::value 为 false。

所以这里我们说 T 不是 Person 类型，还需要忽略如下情况：

 * 包括 Person 的引用。为了确定是否应该启用通用引用构造函数，类型 Person、Person& 和 Person&& 都与 Person 相同。

 * 它是 const 还是 volatile。就我们而言，const Person 和 volatile Person 以及 const volatile Person 和 Person 是一样的。

   这就意味着我们需要一种方法来剥离 T 中的 引用、const 和 volatile，然后在检查该类型是否与 Person 相同。标准库给我们提供了我们所需的所有东西（std::decay）。std::decay<T>::type 与 T 相同，这时引用和 cv 限定符（即 const 或 volatile 限定符）被删除了。（std::decay 还可以将数组和函数类型转换为指针，参看 Item1）那么，我们想要的控制构造函数是否启用的条件是：

   ```c++
   !std::is_same<Person, typename std::decay<T>::type>::value
   ```

   即，Person 与 T 不是同一类型，忽略任何引用或cv限定符。（正如 Item9 所阐述的，std::decay 前面的 "typename" 是必须的，因为类型 std::decay<T>::type 取决于模板参数 T。)

   ​		将这个判断条件插入上面的 std::enable_if 模板中，并对结果进行格式化，以便更容易地看到各个部分是如何组合在一起的，就会为 Person 的完美转发构造函数生成如下声明：

   ```c++
   class Person{
     public:
       template<typename T, 
       	typename = typename std::enable_if<
               !std::is_same<Person, typename std::decay<T>::type>::value>::type
                   >
           explicit Person(T&& n);
       ...
   };
   ```

   ​		这似乎看起来已经万事大吉，让我们再来看看 Item26 中派生自 Person 的类以传统方式实现复制和移动操作：

   ```c++
   class SpecialPerson:public Person{
     public:
       SpecialPerson(const SpecialPerson& rhs)		// copy ctor;calls base class
       : Person(rhs)								// forwarding ctor!
       {...}
       
       SpecialPerson(SpecialPerson&& rhs)			// move ctor;calls base class
   	: Person(std::move(rhs))					// forwarding cotr!
       {...}
       ...
   };
   ```

   ​		上面的例子和 Item26 中展示的相同。当我们复制或移动一个 SpecialPerson 对象，我们期望复制或移动它的基类部分使用基类的复制和移动构造函数，但是传给基类构造函数的类型是 SpecialPerson，因为 SpecialPerson 跟 Person 是不同的类型（使用 std::decay 也是不同的），这就导致基类的通用引用构造函数被调用。

   ​		派生类部分遵循复制和移动构造函数的常规规则，因此解决这个问题的方法在于基类，特别是控制 Person 的通用引用构造函数是否启用的条件中。我们现在意识到，我们需要针对 Person 类和 Person 的派生类禁用其通用引用构造函数。

   ​		标准模板库中有一个 std::is_base_of 用来判断继承关系。std::is_base_of<T1, T2>::value 如果是 true，则表示 T2 是从 T1 派生来的。有了它我们想修改控制 Person 的完美转发构造函数的条件就简单了。使用 std::is_base_of 替代 std::is_same 实现如下：

   ```c++
   class Person{
     public:
       template<
       	typename T, typename = 
       		typename std::enable_if<
   	    	!std::is_base_of<Person, typename std::decay<T>::type>::value>::value
   									>::type
   			>
   		explicit Person(T&& n);
       ...
   };
   ```

   ​		这时我们已经使用 C++11 的语法完成了我们所需的功能。如果使用 C++14，这段代码依旧 OK，但是我们可以使用 std::enable_if 和 std::decay 的别名来简化代码，如下 ：

   ```c++
   class Person{							// C++14
     public:
       template<
       	typename T, 
       	typename = std::enable_if_t<	// less code here
               		!std::is_base_of<Person, std::decay_t<T>>::value
   					>
   		>
   		explicit Person(T&& n);
       ...
   };
   ```

   ​		到此为止，我们已经知道如何使用 std::enable_if 对实参类型进行选择性禁用 Person 的通用引用构造函数，但是仍然无法区分整形和非整形实参。那我需要做的是 (1) 添加一个 Person 构造函数重载来处理整形参数，(2) 进一步约束模板化的构造函数，以便禁用这类参数。

   ```c++
   class Person{
     public:
       template<
       	typename T,
       	typename = std::enable_if_t<
               !std::is_base_of<Person, std::decay_t<T>>::value
   			&&
   			!std::is_integral<std::remove_reference_t<T>>::value
   			>
   		>
   		explicit Person(T&& n)			// ctor for std::strings and
   		: Person(std::forward<T>(n))	// args convertible to std::strings
           {...}
       
       	explicit Person(int idx)		// ctor for integral args
   		: name(nameFromIdx(idx))
           {...}
       	
       	...								// copy and move ctors, etc
     private:
       std::string name;
   };
   ```

***<u>权衡</u>***

​		本章前三种技术----放弃重载、使用 const T& 和值传递，主要是为要调用的函数形参指定一个类型。最后两个技术----标签调度和约束模板，因为使用完美转发所以不指定参数类型，基础是指定类型然后做指定类型的工作。

​		作为一种规则，完美转发的效率更高，因为它避免了创建符合参数类型的临时变量。Person 的完美转发构造函数，直接将字符串 "Nancy" 转发内部成员 name，然后直接调用 std::string 的构造函数。而不使用完美转发会创建一个临时的 std::string 对象，然后满足 Person 的构造函数。

​		但完美转发也有缺点。一个是某些类型的参数不能被完美地转发，即使它们可以被传递给具有特定类型的函数。Item30 会讲述这些完美转发的失败案例。

​		第二个问题是客户端传递无效参数时错误消息的可理解性。例如，假设一个创建 Person 对象的客户端传递一个由 char16_t（一种在 C++11 中 引入的表示16位字符的类型）组成的字符串常量，而不是由 char （std::string 是由 char 组成的）：

```c++
Person p(u"Konrad Zuse");	// "Konrad Zuse" consists of characters 
							// of type const char16_t
```

​		通过本章中前三种方法的检查，编译器会提示没有从 const char_16t[12] 到 int 或 std::string 的转换。然而，使用基于完美转发的方法，const char16_t 数组可以绑定到构造函数的形参上。然后它被转发到 Person 的 std::string 数据成员的构造函数，只有执行到这里，调用者传入的（const char16_t数组）和所需的（std::string 构造函数可以接受的类型）之间的不匹配才会被发现。这时提示的错误信息会非常的难以理解可能会有上百行提示信息。

​		本例中，通用引用只转发一次（从 Person 构造函数转发到 std::string 构造函数）。但是在复杂的系统中，通用引用转发的次数越多，出错的信息可能越令人困惑。保留通用引用参数接口的原因可能就是起初考虑性能而保留的。

​		在 Person的例子中，我们知道转发函数的通用引用参数应该是 std::string 的初始化数据，因此我们可以使用 static_assert 来验证它是否能够满足。std::is_construtible 类型特征执行编译时测试，以确定是否可以从不同类型（或类型集）的对象构造一种类型的对象，因此实现断言很容易：

```c++
class Person{
  public:
    template<					// as before
    	typename T,
    	typename = std::enable_if_t<
            	!std::is_base_of<Person, std::decay<T>>::value
				&&
				!std::is_intergral<std::remove_reference_t<T>>::value
				>
			>
		explicit Person(T&& n)
		: name(std::forward<T>(n))
        {
            // assert that a std::string can be created from a T object
            static_assert(
            	std::is_constructible<std::string, T>::value, 
                "Parameter n can't be used to construct a std::string"
            );
            
            ...
        }
    
    ...				// remainder of Person class (as before)
};
```

​		如果客户端代码试图从一个不能用于构造 std::string 的类型创建 Person，则会产生指定的错误消息。遗憾的是，在这个示例中，static_assert 在构造函数的主体中，但是转发代码作为成员初始化列表的一部分位于它之前。所以编译的结果通常的错误消息出现后（费解的错误提示信息），才会出现由 static_assert 产生产生的容易理解的消息。

   **小结**

* 通用引用和重载组合的替代方法包括使用不同的函数名，通过给左值添加 const 限定符传参，通过值传递传参以及使用标签调度。
* 通过 std::enable_if 约束模板允许同时使用通用引用和重载，但它控制编译器使用通用引用重载的条件
* 通用引用参数通常具有效率优势，但它们通常具有可用性缺陷

---

#### *item 28. 了解引用折叠*

​		Item23 中阐述当实参被传递给模板函数时，模板形参推导出的类型将编码实参是左值还是右值。Item23 没有提到，只有当实参被用来初始化一个通用引用的形参时才会发生这种转换，因为通用引用是在 Item24 才引入的。总之，像下面的例子关于通用引用的左、右值编码的结果，无论传给 param 的左值还是右值，都可以推导出 T 的参数编码。

```c++
template<typename T>
void func(T&& param)
```

​		编码机制很简单。当将左值作为参数传递时，T 被推导为左值引用。当传递右值时，T 被推导为非引用。（注意不对称性：左值被编码为左值引用，而右值被编码为非引用）因此：

```c++
Widget widgetFactory();			// function returning rvalue
Widget w;						// a variable (an lvalue)
func(w);						// call func with lvalue; T deduced to be Widget&
func(widgetFactory());			// call func with rvalue; T deduced to be Widget
```

​		在对 func 的两次调用中，都传递了一个 Widget，但由于一个 Widget 是左值，一个是右值，因此模板参数 T 的类型就不同了。这决定了通用引用是变成右值引用还是左值引用，这也是 std::forward 工作的基本机制。

​		在进一步研究 std::forward 和通用引用之前，我们必须留意，在 C++ 中对引用的引用是非法的。如果你声明一个这样的案例，那么编译器就会报错：

```c++
int x;
...
auto&& rx = x;			// error! can't declare reference to reference
```

​		但是考虑一下当一个左值被传递给一个接受通用引用的函数模板时会发生什么：

```c++
template<typename T>
void func(T&& param)		// as before
func(w);					// invoke func with lvalue; T deduced as Widget&
```

​		如果我们使用为 T 推导的类型（即，Widget&）并使用它来实例化模板，我们会得到：

```c++
void func(Widget& && param);
```

​		一个引用的引用！但是编译器却没报错！在 Item24，因为我们知道用一个左值引用参数初始化通用引用，参数的类型应该是一个左值引用，但编译器是如何从 T 的推到类型中得到结果，并将其替换到模板中，这是最终的函数签名？

```c++
void func(Widget& param);
```

​		答案是引用折叠。是的，禁止引用的引用，但是编译器可能会在特定的上下文中生成它们，其中就有模板实例化。当编译器生成对引用的引用时，引用折叠决定接下来会发生什么。

​		有两种引用（左值和右值），因此有四种可能的引用-引用组合（左值到左值，左值到右值，右值到左值和右值到右值）。如果一个引用出现在一个允许这样做的上下文（例如，在模板实例化期间），那么引用将遵循如下规则折叠为一个引用：

```c++
如果其中一个引用是左值引用，则结果是左值引用。否则（例如，如果两者都是右值引用），结果就是右值引用。
```

​		在上面的例子中，将推导出的类型 Widget& 替换到模板函数中会产生一个左值引用，引用折叠最后的结果是一个左值引用。

​		引用折叠是使用 std::forward 工作的关键部分。正如 Item25 中锁阐述的，std::forward 被应用于通用引用参数，来看一个使用案例：

```c++
template<typename T>
void f(T&& fParm)
{
    ...									// do some work
	someFunc(std::forward<T>(fParam));	// forward fParam to someFunc
}
```

​		因为 fParam 是一个通用引用，所以我们知道类型参数 T 将编码传递给 f 的参数（即用于初始化 fParam 的表达式）是左值还是右值。std::forward 的任务是将  fParam（一个左值）强制转换为右值，当且仅当 T 编码传递给 f 的参数是右值时，即 T 是非引用类型。

​		下面是如何实现 std::forward 的方法：

```c++
template<typename T>
T&& forward(typename remove_reference<T>::type& param)		// in namespace std
{
    return static_cast<T&&>(param);
}
```

​		这并不完全符合标准（省略了一些接口细节），但是对于理解 std::forward 的行为来说，这些差异是无关紧要的。假设传递给 f 的参数是一个 Widget 类型的左值。T 将被推导为 Widget&，对 std::forward 的调用将实例化为 std::forward<Widget&>。将 Widget& 插入 std::forward 实现会产生这样的结果：

```c++
Widget& && forward(typename remove_reference<Widget&>::type& param)
{
    return static_cast<Widget& &&>(param);
}
```

​		类型特征 std::remove_reference<Widget&>::type 产生 Widget（参看 Item9），所以 std::forward 变成：

```c++
Widget& && forward(Widget& param)
{
    return static_cast<Widget&>(param);		// still in namespace std
}
```

​		如你所见，当将左值参数传递给函数模板 f 时，std::forward 被实例化以接受并返回一个左值引用。在 std::forward 里面的转换没有任何作用，因为 param 的类型已经时 Widget&，所以将他转换为 Widget& 没有效果。传递给 std::forward 的左值参数将返回一个左值引用。根据定义，左值引用就是左值，因此将左值传递给 std::forward 会返回一个左值，正如我们所期待的那样。

​		现在假设传递给 f 的参数是一个 Widget 类型的右值。在本例中，f 的类型参数 T 的推断类型将是 Widget。因此 f 内部对 forward 的调用将是 std::forward<Widget>。在 std::forward 实现中用 Widget 替换 T 会得到这样的结果：

```c++
Widget&& forward(typename remove_reference<Widget>::type& param)
{
    return static_cast<Widget&&>(param);
}
```

​		对非引用类型 Widget 应用 std::remove_reference 会得到与它开始时（Widget）相同的类型，所以 std::forward 变成如下：

```c++
Widget&& forward(Widget& param)
{
    return static_cast<Widget&&>(param);
}
```

​		这里没有对引用的引用，所以没有引用折叠，这是调用 std::forward 的在最终实例化版本。从函数返回会的右值引用被定义为右值，因此在本例中，std::forward 将把 f 的参数 fParam（一个左值）转换为右值。最终的结果是，传递给 f 的右值参数将被转发给 someFunc 作为右值，这正是应该发生的。

​		在 C++14 中，因为 std::remove_reference_t 的存在使得实现会更加简洁：

```c++
template<typebane T>
T&& forward(remove_reference_t<T>& param)	// C++14; still in namespace std
{
    return static_cast<T&&>(param);
}
```

​		引用折叠发生在 4 个上下文中。第一个也是最常见的是模板实例化。第二个是自动变量的类型生成。细节基本上和模板是一样的，因为自动变量的类型推导基本上和模板的类型推导是一样的（参看Item2）。再来看看本章之前提到的例子：

```c++
template<typename T>
void func(T&& param)
Widget widgetFactory();			// function returning rvalue
Widget w;						// a variable(an lvalue)
func(w);						// call func with lvalue; T deduced to be Widget&
func(widgetFactory());			// call func with rvalue; T deduced to be Widget


//这些转换可以使用 auto 模式实现，如下声明：
auto&& w1 = w;
//用左值初始化w1，从而推导出 auto 的类型是 Widget&。在w1的生命中 auto 插入 Widget& 将产生
//以下引用到引用的代码：
Widget& && w1 = w;
//进行引用折叠后
Widget& w1 = w;
//结果 w1 是一个左值引用


//另外，看这个声明
auto&& w2 = widgetFactory();
//用右值初始化 w2 ，auto 将推导出非引用的 Widget,用 Widget 替代 auto 会得到如下结果：
Widget&& w2 = widgetFactory();
//这里没有出现引用到引用的情况，w2 是一个右值引用
```

​		我们现在可以理解 Item24 中引入的通用引用。通用医用并不是一种新类型的引用，它实际上是在满足两个条件的上下文中的右值引用：

* 类型推导区分左值和右值。推导 T 类型的左值为 T&，T 类型的右值为 T。
* 引用折叠

 		通用引用的使用很有用，因为它使你不必在意存在引用折叠的上下文，不必在意左值和右值不同类型的推导不必在意类型推导后出现引用折叠。

​		除了上边**模板实例化**和**自动类型生成**外，第三种是**使用 typedef 和声明别名**（参看Item9）。使用 typedef 创建或求值过程中出现引用的引用时，引用折叠会介入消除它们。例如，假设我们有一个 Widget 类模板，其中包含一个用于右值引用类型的类型定义：

```c++
template<typename T>
class Widget{
  public:
    typedef T&& RvalueRefToT;
    ...
};

//假设我们使用一个Widget的一个左值引用类型实例化它：
Widget<int&> w;
//替换完 T 后 typedef 如下：
typedef int& && RvalueRefToT;
//进行引用折叠后
typedef int& RvalueRefToT;
```

很明显，我们为 typedef 定义的名称可能并不像我们期望的那样应该是一个右值引用：当 Widget 用左值引用类型实例化时，RvalueRefToT 是一个 typedef 为左值引用的别名。

​		发生引用折叠的最后一个上下文是**使用 decltype**。如果在分析涉及 decltype 的类型时，出现引用的引用，那么引用折叠会被触发。（有关 decltype 的相关信息，参看Item3）

   **小结**

* 引用折叠发生在如下四个上下文：模板实例化、自动类型生成、创建，声明 typedef 和别名声明、decltype
* 当编译器在一个引用折叠的上下文中生成引用时，结果会变成一个引用。如果其中一个原始引用是左值引用，则结果是左值引用。否则他是一个右值引用
* 在类型推导区分左值和右值上下文以及引用折叠的上下文中，通用引用一般视为右值引用

---

#### *item 29 假设移动操作不存在，消耗也不低，也不可用*

​		移动语义可以说是 C++11 的主要特性。"移动容器现在就跟复制指针一样高效！"，你可能会听到，"复制临时对象开销很高，编码来避免它相当于提前优化！"这种情绪很容易理解。移动语义确实是一个重要的特性。它不仅允许编译器将高消耗的复制操作替换为移动操作，还要求编译器在满足适当条件时自动去完成这样的操作。拿着 C++98 代码，用符合 C++11 的编译器和标准库重新编译，你的软件会运行的更快。

​		但是移动语义并非完美！让我们从观察不支持移动语义的几种类型来开始说起。C++11 对整个 C++98 标准库进行了彻底的修改，为移动比复制更快的类型添加了移动操作，并且修改了库组件的实现来利用这些操作。对于没有进行 C++11 标准优化的应用程序（或者使用的库）中的类型，编译器中存在的 move 操作可能不会产生优化效果。当然，C++11 很愿意为缺少移动操作的类生成移动操作，但这只会发生在声明中没有赋值操作、移动操作或析构函数的类（参看 Item17）。禁用移动操作的数据成员或基类（例如，禁用引动操作--参看Item11）也将禁用编译器生成移动操作。对于没有明确支持移动的类型，并且这些类型不符合编译器生成移动操作的条件，没有理由期望 C++11 提供比 C++98 更高的性能改进。

​		即使是具有明确移动支持的类型可能也不想你希望的那样收益。例如，C++11 标准库中的所有容器都支持移动，但是如果认为所有容器移动操作性能都很高那就错了。对于一些容器来说，不存在真正高效的方法来移动其成员。另外，一些容器提供的真正高效的移动操作可能无法满足容器元素。

​		考虑 C++11 中的一个新容器 std::array。std::array 是一个内置的 STL 接口数组。它与其他容器存在本质上的不同，每个标准容器都将其成员存储在堆上。这类容器类型的对象（作为数据成员），从概念上将，只有一个指向对内存的指针，用于存储容器的内容。（实际情况更为复杂，但对于本文的分析而言，这些差异并不重要）该指针的存在使得在常量时间内移动这个容器的内容成为可能：只需要将指向容器内容的指针从原容器复制到目标容器，最后将源指针置为空：

```c++
std::vector<Widget> vw1;
// put data into vm1						[vw1]  ------> Widgets( Pointer )
...
// move vw1 into vw2.Runs in
// constant time.Only ptrs in vw1 and vw2	[vw1]==null
// are modified								[vw2]  ------> Widgets( Pointer )
auto vw2 = std::move(vw1);
```

std::array 对象缺少这样的指针，因为 std::array 的内容直接存储在 std::array 对象中：

```c++
std::array<Widget, 1000> aw1;
//put data into aw1									aw1
...												 [Widgets]
//move aw1 into aw2. Runs
//in linear time.All element						aw1
//in aw1 are moved into aw2						[Widgets(moved from)]
auto aw2 = std::move(aw1);							aw2
    											[Widgets(moved to)]
```

​		注意，aw1 中的元素被移动到 aw2 中。假设 Widget 是移动比复制快的类型，移动 Widget 的std::array 将比复制相同的 std::array 快。所以 std::array 当然提供了移动支持。然而，移动和复制 std::array 都具有线性时间的复杂度，因为每个元素都必须被复制或移动。这与听到的"移动一个容器现在就像两个指针一样高效"的说法有所差异。

​		另外，std::string 提供了常量时间移动操作和线性时间拷贝。这听起来好像移动比复制根块，但是事实并非如此。许多字符串实现使用小字符串优化（SSO）。使用 SSO，"小"字符串（例如，常量不超过15字节的字符串）会被存储在 std::string 对象的缓冲区中，而不是用堆空间存储。使用基于 SSO 实现的小字符串移动并不比复制快，因为它不满足"只复制一个指针"的性能优势实现原理。

​		使用 SSO 的动机是大量证据表明短字符串是许多程序使用的标准。使用内部缓冲区来存储这些字符串的内容，不需要动态分配内存，这通常效率更高。这样就导致复制操作并不比移动操作慢的情况出现。

​		即使对于支持快速移动操作类型，看上去应该是应用移动操作但是最终还是会导致复制操作被调用。Item14 中阐述，一些标准库的容器操作提供了强异常安全来确保升级到 C++11 后，遗留依赖 C++98 标准的代码不会被破坏，底层的复制操作可能会取代移动操作，除非确定移动操作不会抛出异常。结果就是，即使一个类型提供了比相应赋值操作更高效的移动操作，甚至代码特征上移动操作更合适（例如，源对象是右值），编译器可能仍然强制调用复制操作，因为移动操作没有声明 noexcept。

因此，在以下几种情况，C++11 的移动语义不会体现其价值：

* 无移动操作：要移动的对象无法提供移动操作。因此，移动请求变为复制请求
* 移动不快：要移动的对象的移动操作并不比复制操作快
* 移动操作不可用：将要进行移动的上下文需要一个不触发异常的移动操作，但移动操作没有声明 noexcept

需要提一下，另一种情况下移动语义也没有效率增益：

* 源对象是左值：除了极少数例外（参看Item25），只有右值可以被用作移动操作的源

​		本章的标题是假设移动操作不存在、不高效、不可使用。这在泛型代码中是典型的情况，例如，在编写模板时，因为不知道正在使用的所有类型。在这种情况下，必须像 C++98  中那样保守地使用复制对象---在移动语义未出现前。对于"不稳定"地代码也是如此，例如，代码中所使用地类型需要相对频繁地修改。

​		但是，通常我们一般比较了解我们代码中使用到的类型（例如，他们是否支持高效地移动操作）。在这种情况下，就不需要做任何假设。你可以简单地查找正在使用的类型的移动支持细节。如果这些类型提供高效的移动操作，并且你在调用移动操作的上下文中使用到了这些对象，那么你可以放心使用移动语义，将赋值操作替换为移动操作。

   **小结**

* 考虑移动操作不存在、性能不高、不可使用的情况
* 在已知的类型或支持移动语义的代码中，不需要顾虑太多

---

#### *item 30. 熟悉完美转发失败案例*

​		C++11 最突出的特这之一是完美转发。完美转发，其实并不"完美"！当你愿意忽略一些东西时，C++11 的完美转发才能达到真正的完美。本章将详细介绍这些情况。

​		我们先回顾一下"完美转发"，"转发"是指一个函数向另一个函数传递---转发---它的参数。目标是让第二个函数（被转发的函数）接受与第一个函数（执行转发的函数）所接收的相同的对象。这就排除了按值参数，因为它们是原始调用者传入的内容副本。我们希望被转发的函数可以直接操作原始参数。指针形参也排除，因为我们不想强制调用者传递指针。当涉及到通用转发时，我们将处理是引用的参数。

​		完美转发意味着我们不只是转发对象，我们还转发它们显著的特征：它们的类型，他们是左值还是右值，他们是 const 还是 volatile。结合我们见要处理的引用形参的观察，可以看出必须使用通用引用（参看Item24）,因为只有通用引用形参会编码传递给它的参数的左值和右值信息。

​		现在假设有一个函数 f，我们想要编写一个函数（实际上是一个函数模板）来给它转发数据。我们实现大概是这样：

```c++
template<typename T>
void fwd(T&& param)						// accept any argument
{
    f(std::forward<T>(param));			// forward it to f
}
```

​		转发功能本质上是通用的。例如， fwd 模板接受任意类型的参数，并且转发这些参数。这种泛型逻辑的进一步扩展是不仅使用模板，还可以接收可变参数，因此可以接受任意数量的参数。fwd 的实现如下：

```c++
template<typename... Ts>
void fwd(Ts&&... params)				// accept any arguments
{
    f(std::forward<Ts>(params)...);		// forward them to f
}
```

​		这种形式在标准库容器的 emplace 函数和智能指针函数 std::make_shared 、std::make_unique（参看Item21）中都可以看到。

​		现在给定的目标函数 f 和转发函数 fwd，如果调用带有特定参数的 f 做了一件事，但是调用带有相同参数的 fwd 做了不同的事，那么完美转发就会失败。

```c++
f(expression);			// if this does one this,
fwd(expression);		// but this does something else, fwd fails
						// to perfectly forward expression to f
```

​		有几种导致出现这个失败的情况。了解它们是啥以及是如何导致的很重要，接下来一起来看看不能完美转发的情况。

***<u>大括号初始化</u>***

假设 f 的定义如下：

```c++
void f(const std::vector<int>& v);
```

在这种情况下，使用大括号初始化调用 f 可以编译通过，

```c++
f({1,2,3});			// fine, "{1,2,3}" implicitly
					// converted to std::vector<int>
```

但是相同的大括号初始化传递个 fwd 就会编译不通过：

```c++
fwd({1,2,3});		// error! doesn't compile
```

这是因为使用大括号初始化是完美转发的一个失败案例。

​		所有这些失败案例都是相同的原因。直接调用 f（如，f({1,2,3})），编译器会对比传递的参数和函数 f 声明的参数类型是否兼容，而且如果有必要，会进行隐式转换使调用成功。在上面的例子中，会从 {1,2,3} 生成一个临时的 std::vector<int> 对象，这样 f 的参数 v 就会绑定一个 std::vector<int> 对象。

​		当通过转发函数模板 fwd 间接调用时，编译器不再对 fwd 调用点传递的实参和 f 中的形参声明进行对比。相反，它推导传递给 fwd 的实参类型，并将推导出的类型与 f 的形参声明进行比较。发生以下任何一种情况，完美转发将会失败：

* 编译器无法推导 fwd 的一个或多个参数类型。这种情况下，编译会失败。
* 编译器推导出 fwd 的一个或多个参数是"错误"类型。这里，"错误"可能意味着使用 fwd 推导出的类型调用 f 的行为不同于直接调用传递给 fwd 的参数。如果 f 是一个重载的函数，并且由于"错误"的类型推导，在 fwd 内部调用的 f 的重载与直接调用 f 时将调用的重载不同，这可能是导致这个情况出现的一个原因。

​		在上面的 "fwd({1,2,3})" 调用中，问题是将一个带大括号的初始化表达式传递给一个没有声明 std::initializer_list 的函数模板形参，就像标准版说的那样，这是一个"非推断上下文"。简单的说就是编译器不能在调用 fwd 时推导出表达式 {1,2,3} 的类型，因为 fwd 的参数没有声明为 std::initializer_list。由于不能推导 fwd 参数的类型，编译器必须可以拒绝这个调用。

​		有意思的，Item2 阐述了使用大括号初始化表达式地自动类型推导（auto）是成功的。这样地变量被认为是 std::initializer_list 对象，这为以下情况提供了一个简单地解决方案：转发函数应该推导地类型是 std::initializer_list --- 使用 auto 声明一个局部变量，然后将局部变量传递给转发函数：

```c++
auto il = {1,2,3};		// il's type deduced to be std::initalizer_list<int>
fwd(il);				// fine, perfect-forward il of f
```

***<u>使用0或NULL表示空指针</u>***

​		Item8 说过，当你尝试传递 0 或 NULL 作为空指针传递给模板时，类型推导出错了，推导出一个整形（通常是 int）而不是你传递的参数的指针类型。结果就是 0 和 NULL 都不能作为k空指针被完美转发。但是，秀姑起来也很简单：像 Item8 所述，使用 nullptr 替代 0 和 NULL。

***<u>仅声明 static const 整形数据成员</u>***

​		通常不需要在类中定义static const 整形数据成员；不声明限定符就可以了。这是因为编译器对这些成员的值执行了*常量传播*，因此不需要为它们预留内存。考虑如下代码：

```c++
class Widget{
  public:
    static const std::size_t MinVals = 28;	// MinVals' declaration
    ...
};
...											// no defn. for MinVals
std::vector<int> widgetData;
widgetData.reserver(Widget::MinVals);		// use of MinVals
```

​		这里，我使用 Widget::MinVals （之后简称为 MinVals）来指定 widgetData 的初始容量，尽管 MinVals 没有定义。编译器通过在所有提到 MinVals 的地方填充值 28 来处理缺失的定义（这是必须的）。事实上，没有为 MinVals 的值预留存储是没问题。如果 MinVals 的地址被获取（例如，如果有人创建了一个指向 MinVals 的指针），那么 MinVals 将需要存储（这样指针就有指向的东西），而上面的代码，尽管可以编译，但是在链接时将会失败，直到提供了 MinVals 的定义。

考虑到这一点，假设 f（函数 fwd 转发给的函数）是这样声明：

```c++
void f(std::size_t val);
```

使用 MinVals 调用 f 是可以的，因为编译器只会用它的值替换 MinVals:

```c++
f(Widget::MinVals);			// fine, treadted as "f(28)"
```

但是使用 fwd 间接调用 f，情况可能就不太一样了：

```c++
fwd(Wodget::MinVals);		// error! shouldn's link
```

这段代码可以编译通过但是应该不能成功链接。如果代码是取 MinVals 的地址你就不会疑惑了，因为潜在的问题是相同的，没有定义 MinVals。

​		尽管源代码中没有任何参数接受 MinVals 的地址，但 fwd 的参数是一个通用引用，而编译器生成的代码中，引用通常被视为指针。在程序的底层二进制代码中（以及硬件上），指针和引用本质上是一样的。有句话说的好呀：引用是优化后的指针。在这种情况下，通用引用传递 MinVals 值和通过指针传递 MinVals 值是一样的，因此必须有一些内存供指针指向。因此，通过引用传递整形 static const 数据成员通常要求先定义这些成员，这种要求导致使用完美转发可能会出现失败的情况。

​		上面提到的"应该"不会链接成功？根据标准，通过引用传递最小值需要定义它。但是并不是所有的实现都执行这个要求。因此，根据编译器和连接器的不同，你可能会发现未定义的整形 static const 数据成员可以进行完美转发。但是这就造成了可移植性问题，要使其可移植只需要提供整形 static const 数据成员的定义即可。对于 MinVals 应该这样：

```c++
const std::size_t Widget::MinVals;			// in Widget's .cpp file
```

​		注意不要进行重复初始化（这个例子中 MinVals 初始化为 28）。不过，不要对这个细节感到太过紧张。如果忘记在两处进行初始化，编译器报错提示的。

***<u>重载函数名和模板名</u>***

​		假设我们的函数 f（通过 fwd 将参数转发到的函数）可以通过传递一个函数来定制它的行为，这个函数可以完成一些自定义功能。假设这个函数接受并返回整形数，f 可以这样声明：

```c++
void f(int(*pf)(int));			// pf = "processing function"
```

当然 f 也是可以使用更简单的非指针语法声明。这样的声明看起来是这样，并且它的含义与上面的声明相同：

```c++
void f(int pf(int));			// declares same f as above
```

然后假设我们有一个重载函数 processVal:

```c++
int processVal(int val);
int processVal(int val, int priority);
```

然后调用：

```c++
f(processVal);					// fine
```







   **小结**



---

#### 

