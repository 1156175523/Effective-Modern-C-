# chapter 4.  Moving to Modern C++

列举几个原始指针（raw pointer）不那么讨人喜欢的理由：

1、从它的声明看不出它指向的是一个单个的对象还是一个数组

2、当使用完它时，从它的声明看不出是否应该把它销毁，即指针是否拥有它指向的东西

3、确定应该销毁指针指向的内容时无法确定如何销毁。你应该使用 delete ，还是有不同的销毁机制（例如，应该将指针传递给一个专用的销毁函数）

4、如果发现需要使用 delete 进行销毁，由于 1 中阐述的原因可能无法确定使用单个对象形式 "delete" 还是数组形式 "delete[]"。如果使用错误的形式将会导致未定义的行为

5、假设确定指针拥有它所指向的东西，并且你发现了如何销毁它，但是也很难确保你在代码中的每条路径（包括由于异常引起的路径）执行一次销毁。缺少销毁会导致内存泄漏，多执行会导致未定义的行为

6、一般没办法判断指针是否是悬挂指针，即确定一个指针不再拥有它只想的对象。当一个指针指向的对象被销毁了，该指针就变成了悬挂指针。

​	C++ 11 中引入了智能指针，它是原生指针的一个升级版，可以规避很多原生指针的陷阱。C++11 中规定了四个智能指针：std::auto_ptr，std::unique_ptr，std::shared_ptr，std::weak_ptr。它们都是协助管理已分配对象的生命周期，即通过确保在适当的时间（包括发生异常时）以适当的方式销毁此类对象来避免资源泄漏。

​	C++98 中 std::auto_ptr 已被弃用。C++98 尝试将 std::auto_ptr 标准化后来实现 C++11 中的 std::unique_ptr 的功能，为了达到这个目的， move 语法不可或缺，但是 C++98 当时还没有 move 语法，所以做了一个妥协方案：利用拷贝操作来模拟 move。这导致一些奇怪的代码（如拷贝一个 std::auto_ptr 会将它置为null！）和一些让人费解的使用限制（不能在容器中使用 std::auto_ptr）。std::unique_ptr 做到了 std::auto_ptr 所能做的所有事情，而且它的实现更高效。

#### *item 18. 使用 std::unique_ptr 获取独占所有权资源管理*

​	当使用智能指针时，最先想到的是 std::unique_ptr，默认情况下 std::unique_ptr 与原始指针的大小相同，并且对于大多数操作（包括解引用），它们执行相同的指令。这意味着在内存和周期紧张的情况下也可以使用它们，std::unique_ptr 的性能跟原始指针几乎相同。

​	std::unique_ptr 体现了独占语义，一个非空的 std::unique_ptr 永远拥有它指向的对象，move 一个 std::unique_ptr 会将所有权从源指针转向目的指针（源指针指向null）。拷贝一个 std::unique_ptr 是不允许的，假如说真的可以允许拷贝 std::unique_ptr，那么将会有两个 std::unique_ptr 指向同一个资源，每个都认为它自己拥有且可以销毁那块资源。因此，std::unique_ptr 是一个 move-only 类型。当它面临析构时，一个非空的 std::unique_ptr 会销毁它所拥有的资源。默认情况下，std::unique_ptr 会使用 delete 来释放他所包裹的原生指针指向的空间。

​	std::unique_ptr 的一个常见用法是作为一个工厂函数返回一个继承层级中的一个特定类型的对象。假设我们有一个基类 Investment 的投资类型（例如股票、债券、房地产等）的层次结构。

class Investment{ ... };

class Stock : public Investment { ... };                                               Investment

class Bond : public Investment { ... };                                         |-----------|----------|

class RealEstate : public Investment { ... };                            Stock      Bond    RealEstate    

​	这种层次结构的工厂函数通常会在堆上分配一个对象并返回一个指向它的指针，调用者负责在不需要时删除该对象。这跟 std::unique_ptr 完美匹配，因为调用者负责管理工厂返回的资源（即对它的独占所有权），并且当 std::unique_ptr 销毁时会自动删除它所管理的资源。投资层次结构的工厂函数可以这样声明：

```c++
// creater
template<typename... Ts>									// return std::unique_ptr
std::unique_ptr<Investment> makeInvestment(Ts&&... params);	// to an object created from
															// the given args

// caller
{
    ...
	auto pInvestment =						// pInvestment is of type
        makeInvestment( arguments );		// std::unique_ptr<Investment>
	...
}											// destroy *pInvestment
```

​	同样也可以使用在所有权转移的场景中，例如，工厂函数返回 std::unique_ptr 可以移动到一个容器中，容器元素随即被移动到一个对象中使用，并且该对象随后被销毁。当这种情况发生时，对象的 std::unique_ptr 数据成员也将被销毁，并且它的销毁将导致从工厂返回的资源被销毁。如果这个过程中由于异常或者其他非典型控制流（例如，函数异常提前return或循环异常break结束）而中断，则拥有托管资源的 std::unique_ptr 最终将调用其析构函数，进而释放它所管理的资源。

​	默认情况下，是通过 delete 来销毁资源的，但是构造构成中，std::unique_ptr 也可以自定义要使用的删除器：任意函数（或函数对象，包括由 lambda 表达式生成的对象），然后再需要时调用自定义的删除器来销毁资源。如果 makeInvestment 创建的对象不应该直接删除，而是应该先写一条日志，那么 makeInvestment 可以如下实现:

```c++
auto delInvmt = [](Investment* pInvestment){   // custom deleter (a lambda expression)
    				makeLogEntry(pInvestment);
    				delete pInvestment;
				}

template<typename... Ts>
std::unique_ptr<Investment, decltype(delInvmt)>		// revised return type
	makeInvestment(Ts&&... params)
{
    std::unique_ptr<Investment, decltype(declInvmt)>	// ptr to be returned
        pInv(nullptr, delInvmet);
    
    if ( /* a Stock object should be created */ )
	{
		pInv.reset(new Stock(std::forward<Ts>(params)...));
	}
	else if ( /* a Bond object should be created */ )
	{
		pInv.reset(new Bond(std::forward<Ts>(params)...));
	}
	else if ( /* a RealEstate object should be created */ )
	{
		pInv.reset(new RealEstate(std::forward<Ts>(params)...));
	}
	return pInv;
}
```

了解以下内容会让你的编程更丝滑：

* delInvmt 是 makeInvestment 返回对象的自定义删除器。**所有自定义删除函数参数为指向要销毁对象的原始指针，然后执行必要的删除操作。** 上例中自定义删除函数是先调用 makeLogEntry 然后使用 delete 销毁对象。使用 lambda 表达式创建 delInvmt 相比传统的函数编程要方便。
* **当使用自定义的删除器时，必须将其类型指定为 std::unique_ptr 的第二个类型参数**。在这种情况下，这就是 delInvst 的类型，这也是为啥 makeInvestment 的返回类型是 std::unique_ptr<Investment, decltype(delInvmt)> 的原因（decltype 的相关内容参看 item3）。
* makeInvestment 的基本逻辑是创建一个空的 std::unique_ptr，使其指向一个适当类型的对象，然后放回它。要将自定义删除函数 delInvmt 与 oInv 关联，我们将其作为构造函数的第二个参数入参。
* **尝试将原始指针，例如从 new 分配的 std::unique_ptr 将无法编译，因为它会构成从原始指针到智能指针的隐式转换**。这种隐式转换可能会存在问题，因此 C++11 的智能指针禁止这种操作。这也是为啥使用 reset 来让 pInv 获取对象所有权的原因。
* 每次使用 new 时，我们使用 **std::forward  完美转发**传递给 makeInvestment 的参数（参看 item25）。这使得调用者提供的所有信息直接用于创建的对象。
* 自定义删除器采用 Investment* 类型的参数。无论在 makeInvestment 中创建的实际对象类型是什么（即Stock，Bond，RealEstate），它最终都会在 lambda 表达式中作为 Investment* 对象删除。这就意味着我们通过基类指针删除派生类对象。因此，基类---Investment---必须有一个虚析构函数：

```c++
class Investment{
  public:
    ...
	virtual ~Investment();		// essential design component!
	...
};
```

在 C++14 中，函数返回类型推导的存在（参看item3）促使 makeInvestment 可以使用如下更简单、更具封装的方式表示：

```c++
template<typename... Ts>
auto makeInvestment(Ts&&... params)			// C++14
{
    auto delInvmt = [](Investment* pInvestment)			// this is now inside make
				    {									// Investment
        				makeLogEntry(pInvestment);
        				delete pInvestment;
				    }
    std::unique_ptr<Investment, decltype(delInvmt)>		// as before
        pInv(nullptr, delInvmt);
    
    if ( … ) // as before
	{
		pInv.reset(new Stock(std::forward<Ts>(params)...));
	}
	else if ( … ) // as before
	{
		pInv.reset(new Bond(std::forward<Ts>(params)...));
	}
	else if ( … ) // as before
	{
		pInv.reset(new RealEstate(std::forward<Ts>(params)...));
	}
	return pInv;  
}
```

​	之前提过，当使用默认的删除器时（即 delete），可以认为 std::unique_ptr 对象与原始指针的大小相同。当std::unique_ptr 用到自定义的 删除器时，就不太一样了。作为函数指针的删除器通常会导致 std::unique_ptr 的大小翻倍。对于函数对象的删除器，大小的变化取决于函数对象中存储了多少状态。无状态函数对象（例如，无捕获异常的 lambda 表达式）不会导致大小的增加，也就是说当自定义删除器可以实现为函数或无捕获异常的lambda 表达式时，优选 lambda ：

```c++
auto delInvmt1 = [](Investment* pInvestment)		// custom deleter
				{									// as stateless
    				makeLogEntry(pInvestment);		// lambda
    				delete pInvestment;
				}

template<typename... Ts>							// return type
std::unique_ptr<Investment, decltype(delInvmt1)>	// has size of
makeInvestment(Ts&&... args);						// Investment*

void delInvmt2(Investment* pInvestment)				// custome deleter as function
{
    makeLogEntry(pInvestment);
    delete pInvestment;
}

template<typename... Ts>							// return type has size of 
std::unique_ptr<Investment, void(*)(Investment)>	// Investment* plus at least size
makeInvestment(Ts&&... params);						// of function pointer!
```

​	拥有多种状态的函数对象删除器会导致 std::unique_ptr 对象体量变大。如果发现自定义的删除器导致 std::unique_ptr 容量变得非常大以至于无法接受，那就需要优化设计了。

​	std::unique_ptr 有两种形式，一种用于单对象（std::unique_ptr<T>），另一种用于数组（std::unique_ptr<T[]>）。因此，std::unique_ptr 指向的内容从来不会产生任何歧义。std::unique API 会识别对象形式。例如，单个对象形式没有索引运算符（operator[]），而数组形式缺少解引用运算符（operator* 和 operator->）。

​	C++11 使用 std::unique_ptr 表达独占所有权。但是它最引人注目的特性是它可以轻而易举且有效的转化为 std::shared_ptr：

```c++
std::shared_ptr<Investment> sp =			// converts std::unique_ptr
    makeInvestment( arguments );			// to std::shared_ptr
```

​	这就是 std::unique_ptr 很适合作为工厂函数返回值类型的原因。工厂函数不清楚调用者想使用独占所有权语义还是共享拥有权语义（即 std::shared_ptr）。通过返回 std::unique_ptr ，工厂函数将选择权交给调用者，调用者在需要时可以将 std::unique_ptr 转化为其他可以转换的类型（了解更多 std::shared_ptr 信息，查看item19）。

   **小结**

* std::unique_ptr 是一个小巧、快速、只可移动的智能指针，用于管理具有独占所有权语义的资源
* 默认情况下，资源销毁通过 delete 完成，但是可以指定自定义删除器。有状态删除器和函数指针作为删除器会增加 std::unique_ptr 对象的大小
* 将 std::unique_ptr 转化为 std::shared_ptr 很容易

---

#### *item 19. 使用 std::shared_ptr 进行共享所有权资源管理*

​	C++11 中 std::shared_ptr 实现了共享资源的自动释放功能。通过 std::shared_ptr 访问的对象，它们的生命周期由这些指针通过共享所有权管理。没有特定的 std::shared_ptr 拥有该对象，而是当所有 std::shared_ptr 相互协作以确保不再需要时销毁它。当最后一个指向对象的 std::shared_ptr 停止指向该对象时（例如，因为 std::shared_ptr 被销毁或使其指向不同的对象），该 std::shared_ptr 销毁指向的对象。与垃圾回收机制一样，客户端不需要关心管理指向对象的生命周期，但与析构函数一样，对象销毁的时间是确定的。

​	std::shared_ptr 可以通过查询资源的引用计数来判断它是否是指向资源的最后一个，引用计数是与资源关联的值，用于跟踪有多少 std::shared_ptr 指向它。std::shared_ptr 构造函数增加这个计数，析构函数减少这个计数。（例如，sp1 和 sp2 是指向不同对象的 std::shared_ptr，赋值操作 "sp1 = sp2;" 修改 sp1 指向 sp2 指向的对象。这样操作后的结果就是 sp1 之前指向对象的引用计数递减，sp2 指向对象的引用计数递增。）如果执行递减后 std::shared_ptr 发现引用计数为 0，则不再有 std::shared_ptr 指向该资源，所以 std::shared_ptr 会销毁它。

由引用计数而引发的性能问题：

* std::shared_ptr 的大小是原始指针的两倍，因为它们内部包含指向资源的原始指针以及指向资源的引用计数
* 用于引用计数的内存必须动态分配。从概念上讲，引用计数与被指向的对象相关联，但被指向的对象对此一无所知。所以它们没有存储引用计数的内存。（std::shared_ptr 可以管理包括内置对象在内的所有对象）Item21  阐述了适用 std::make_shared 情况和不适用情况。但是无论何种方式创建，引用计数都是动态分配的数据
* 引用计数的递增和递减必须是原子的，因为不同线程中可以同时存在读取者和写入者。例如，在一个线程中指向资源的 std::shared_ptr 可能正在执行某个析构函数（因此递减它指向的资源的引用计数），而在另外一个线程中，指向同一个对象的 std::shared_ptr 在被复制（因此增加引用计数）。原子操作通常比非原子操作慢，因此即使引用计数通常只有一个单元，你应该也要考虑到它的执行成本更高

但是也不是所有情况都会存在引用计数的递增，移动构造就是一个特殊的情况。从一个 std::shared_ptr 移动构造 另一个 std::shared_ptr 将会导致源 std::shared_ptr 置为 null，这就意味着旧的 std::shared_ptr 在新的 std::shared_ptr 开始时停止指向资源。因此，不需要对引用计数进行操作。所以移动操作会比复制操作更快一些（包括移动构造和移动赋值）。

​	跟 std::unique_ptr 一样，std::shared_ptr 也是使用 delete 作为默认资源销毁机制，但是它也支持自定义删除器。这个支持的设计不同于 std::unique_ptr，对于 std::unique_ptr 自定义删除器是智能指针类型的一部分，对于 std::shared_ptr 则不是：

```c++
auto loggingDel = [](Widget* pw)			// custom deleter
				{							// (as in Item 18)
    				makeLogEntry(pw);
    				delete pw;
				}

// deleter type is part of ptr type
std::unique_ptr<Widget, decltype(loggingDel)> upw(new Widget, loggingDel);

// deleter type is not part of ptr type
std::shared_ptr<Widget> spw(new Widget, loggingDel);
```

​	std::shared_ptr 设计更加灵活。假设存在两个 std::shared_ptr<Widget>，每个都有一个不同类型的自定义删除器（例如，自定义删除器是通过 lambda 表达式指定的）：

```c++
auto customDeleter1 = [](Widget *pw){...}		// custom deleters, each with a
auto customDeleter2 = [](Widget *pw){...}		// different type

std::shared_ptr<Widget> spw1(new Widget, customDeleter1);
std::shared_ptr<Widget> spw2(new Widget, customDeleter2);
```

​	因为 pw1 和 pw2 具有相同的类型，所以它们可以放置在该类型对象的容器中：

```c++
std::vector<std::shared_ptr<Widget>> vpw{sp1, sp2};
```

​	它们也可以相互分配，并且它们每个都可以传递给参数为 std::shared_ptr<Widget> 类型的函数。这些功能 std::unique_ptr 却不能完成，因为它们的自定义删除器的类型不同会影响 std::unique_ptr 的类型。

​	与 std::unique_ptr 的另一个区别是，指定自定义删除器不会改变 std::shared_ptr 对象的大小。无论删除器如何，std::shared_ptr 对象的大小都是两个指针。但是有个问题就是这个自定义删除器可以是函数对象，而函数对象可以包含任意数量的数据，这就意味着它们可以任意大。std::shared_ptr 如何在不使用更多内存的情况下引用任意大小的删除器呐？

​	他肯定是做不到的。他可能需要使用更多内存。但是，该内存不是 std::shared_ptr 对象的一部分。它在堆上，或者，如果 std::shared_ptr 的创建者使用自定义的 allocator 对 std::shared_ptr 进行了支持，那么它就在 allocator 管理的内存当中。之前说过 std::shared_ptr 对象包含一个引用计数指针。这个说的不够确切，这个指针

其实是指向一个称为**控制块**的更大数据结构。每个由 std::shared_ptr 管理的对象都有一个控制块。除了引用计数之外，控制块还包含自定义删除器的副本（如果指定的话）。如果指定了自定义 allocator，则控制块也包含该 allocator 的副本。控制块可能还包括其他的数据，如 Item21 所述，称为弱计数的辅助引用计数，这里我们就不进行细说了。我们可以设想与 std::shared_ptr<T> 对象关联的内存看起来像这样：

![image-20210823113034266](C:\Users\1\AppData\Roaming\Typora\typora-user-images\image-20210823113034266.png)

​	对象的控制块由创建对象的第一个 std::shared_ptr 的函数设置。一般而言，为对象创建 std::shared_ptr 的函数不知道其他 std::shared_ptr 是否已经指向该对象，因此使用以下控制块创建规则：

* std::make_shared （参看Item21）会创建一个控制块。它会创建一个新对象来指向，所以在调用 std::make_shared 时肯定没有该对象的控制块。
* 当从唯一所有权指针（即 std::unique_ptr 或者 std::auto_ptr）构造 std::shared_ptr 时创建控制块。唯一所有权指针不使用控制块，因此指向的对象应该没有控制块。（作为其构造的一部分，std::shared_ptr 承担指向对象的所有权，因此唯一所有权指针设置为 null）
* 当原始指针调用 std::shared_ptr 构造函数时，会创建一个控制块。如果你想从一个已经有控制块的对象创建一个 std::shared_ptr，你应该传递一个 std::shared_ptr 或 一个 std::weak_ptr（参看Item20）作为构造函数的参数，而不是原始指针。std::shared_ptr 的构造函数入参 std::shared_ptr 或 std::weak_ptr 时不会创建新的控制块，因为它们可以依赖传递给它们的智能指针来指向相应的控制块

​	当使用了一个原生的指针构造多个 std::shared_ptr 时，这些规则的存在会使得被指向的对象包含多个控制块，这样会产生未定义行为。多个控制块意味着多个引用计数，多个引用计数就意味着对象会被摧毁多次。如下代码：

```c++
auto pw = new Widget;							// pw is raw ptr
...
std::shared_ptr<Widget> spw1(pw, loggingDel);	// create control block for *pw
...
std::shared_ptr<Widget> spw2(pw, loggingDel);	// create 2nd control block for *pw
```

​	spw1 和 spw2 使用原始指针 pw 构造共享指针，它们都会创建控制块，也就是说会存在两个引用计数。当两个引用计数器都为 0 时，它们都会释放 *pw，这就会出现 double free 的情况。

​	这里得到两个关于 std::shared_ptr 使用建议。首先，尽量避免使用原始指针传递给 std::shared_ptr 构造函数。通常替换为使用 std::make_shared（参看Item21），但在上面的示例中，我们使用的是自定义删除函数器，而 std::make_shared 无法实现。其次，如果必须将原始指针传递给 std::shared_ptr 构造函数，直接传递 new 的结果，而不是通过原始指针变量。从通过一原始指针创建第二个 std::shared_ptr 就可以直接使用 std::shared_ptr 的复制构造，这样不会有任何负面影响：如果上面代码可以如下重写：

```c++
std::share_ptr<Widget> spw1(new Widget, loggingDel);	// direct use of new
std::share_ptr<Widget> spw2(spw1);			// spw2 uses same control block as sp1
```

​	this 指针也会导致的使用也会导致 std::shared_ptr 构造出多个控制块。假设我们的程序使用多个 std::shared_ptr 来管理 Widget，并且有一个结构来跟踪已处理的 Widget:

```c++
std::vector<std::shared_ptr<Widget>> processedWidgets;
```

​	假设 Widget 有一个执行处理的成员函数：

```c++
class Widget{
  public:
    ...
	void process();
    ...
};

void Widget::process()
{								
    ...									// process the Widget
	processedWidgets.emplace_back(this);// add it to list of processed Widgets;
    									// this is wrong!
}
```

​	注释里说明了这样做是错的，指的是传递 this 指针，并不是因为使用 emplace_back（如果想了解 emplace_back，请参看 item42）。给 std::shared_ptr 传递 this 就相当于传递一个原始指针，也就是说 std::shared_ptr 每次创建都会创建控制块，这就会导致出现 double free 未定义的行为。

​	std::shared_ptr 的API包含了修复这一问题的机制。这可能是C++标准库里最诡异的方法名字了：std::enable_shared_from_this。如果希望由 std::shared_ptr 管理的类能够从 this 指针安全的创建std::shared_ptr，那么需要继承这个模板基类。如下继承 std::enable_shared_from_this 基类的示例：

```c++
class Widget: public std::enable_shared_from_this<Widget>
{
  public:
    ...
	void process();
	...
};
```

​	std::enable_shared_from_this 是一个基类模板。它的类型参数是被派生的类的名称，所以 Widget 继承自 std::enable_shared_from_this<Widget>。这种派生类继承自以派生类类型为模板参数的基类是完全合法的，并且会涉及到一个设计模式，名字是 The Curiously Recurring Template Pattern(CRTP) 重复模板模式。std::enable_shared_from_this 定义了一个成员函数，该函数为当前对象创建一个 std::shared_ptr，但它不会复制控制块。这个成员函数是 shared_from_this，如果你想要一个 std::shared_ptr 指向与 this 指针相同的对象可以使用这个成员函数。对 Widget::process 进行优化：

```c++
void Widget::process()
{
    //as before, process the Widget
    ...
	// add std::shared_ptr to current object to processedWidgets
	processedWidgets.emplace_back(shared_from_this());
}
```

​	在内部，**shared_from_this** 查找当前对象的控制块，并创建一个新的 std::shared_ptr 引用该控制块。这个设计依赖于具有关联控制块的当前对象。对此，必须有一个指向当前对象的现有 std::shared_ptr （例如，在调用 shared_from_this 成员函数之外的一个）。如果没有这样的 std::shared_ptr 存在（即，**如果当前对象想没有关联的控制块**），**尽管 shared_from_this 通常抛出异常，仍然会出现未定义行为**。

​	为了阻止客户端在 std::shared_ptr 指向对象之前调用 shared_from_this 成员函数，从 std::enable_shared_from_this 继承的类通常将其构造函数声明为私用，并让客户端通过调用返回 std::shared_ptr 的工厂函数来创建对象。例如，Widget 的实现：

```c++
class Widget: public std::enable_shared_from_this<Widget>
{
  public:
    // factory function that perfect-forwards args to a private ctor
    template<typename... Ts>
    static std::shared_ptr<Widget> create(Ts&&... params);
    
    ...
	void process();					// as before
	...
  private:
    ...								// ctors
};
```

​	一个控制块可能只有几个模块大小，尽管自定义的 deleters 和 allocators 可能会使它更大。通常控制块的实现会比你想象中更复杂。它利用了继承甚至还用到了虚函数（确保指向的对象能正确销毁）这就意味着使用 std::shared_ptr 会应用控制块使用虚函数而导致一定的机器开销。

​	  std::shared_ptr 并不是所有资源管理问题的最佳解决方案。但是对于它提供的功能，std::shared_ptr 的成本非常合理。在使用默认 deleter 和 默认 allocator 以及 std::make_shared 创建 std::shared_ptr 的典型条件下，控制块的大小只用约三个模块。它的分配基本上是不耗费内存空间的（它被合并到被指向对象的内存分配中，详细信息，参看Item21）。解引用一个 std::shared_ptr 花费的代价不会比解引用一个原始指针多多少。执行需要引用计数的操作（例如，复制构造或赋值构造，析构）需要一两个原子操作，但是这些操作通常映射到单独的机器指令，因此尽管与非原子指令相比它们可能成本高一些，但是它们仍然只是单个指令。控制块中的虚函数机制在被std::shared_ptr 管理的对象生命周期中一般只使用一次：当析构销毁对象时。

​	花费了相对较少的代价，你就获得了对动态资源生命周期的自动管理。大多数时间，想要以共享方式来管理对象，使用 std::shared_ptr 是一个大多数情况下都比较好的选择。如果你发现自己开始怀疑是否承受得起使用 std::shared_ptr 的代价时，首先请重新考虑是否真的需要使用共享式的管理方法。如果独占式的管理方式可以实现，则优选 std::unique_ptr ，因为它的性能开销与原始指针大致相同，并且从 std::unique_ptr 转换为 std::shared_ptr 很简单，因为 std::shared_ptr 可以从一个 std::unique_ptr 创建。反过来就不行，一旦将资源的声明周期管理变为 std::shared_ptr，则它的所有权形式不能更改。即使引用计数为1，也无法回收资源的所有权，以便让 std::unique_ptr 管理它。资源和指向它的 std::shared_ptr 之间属于"至死方休，不允许取消和变卦"。
​	**std::shared_ptr 不能做的另外一件事是不能处理数组**。与 std::unique_ptr 的另外一个不同之处在于，std::shared_ptr 有一个 API，该 API 专为指向单个对象的指针而设计。没有 std::shared_ptr<T[]> 这样的用法。如果使用 std::shared_ptr<T> 来指向一个数组，指定一个自定义的 deleter 来做数组的删除操作（即delete []）。这样做可以通过编译，但是却是一个坏主意，原因有二，首先，std::shared_ptr 没有重载操作符 [] ，所以如果通过数组访问需要通过基于指针的运算来进行，第二，std::shared_ptr 支持派生到积累的指针转换，这对单个对象有意义，但是应用于数组时类型系统会出现漏洞。（出于这个原因，std::unique_ptr<T[]> API 禁止这种转换）最重要的是， C++11 提供了内置数组的替代方案（std::arrary,std::vector,std::string）。给数组声明一个指向智能指针通常是不太合理的设计。

**小结**

* std::shared_ptr 为任意资源的共享生命周期管理提供了接近垃圾回收机制
* 与 std::unique_ptr 相比，std::shared_ptr 对象通常会大两倍，会产生控制块的开销，并且需要原子引用计数操作
* 默认资源销毁是通过 delete，但是也支持自定义的 deleter。deleter 的类型对 std::shared_ptr 的类型没有影响
* 避免从原始指针类型变量创建 std::shared_ptr

-----

#### *Item 20. 将 std::weak_ptr 用于创建类似 std::shared_ptr 但是可能悬挂的指针

​	std::weak_ptr 拥有 std::shared_ptr 一样的智能指针，但是它不参与指向资源的共享所有权。换句话说，像 std::shared_ptr 这样的指针但不会影响对象的引用计数。这种智能指针必须可以处理 std::shared_ptr 不会遇到的问题：它所指向的对象可能已被销毁。一个真正的智能指针通过持续跟踪判断它是否已经悬挂来处理这个问题，悬挂意味着它指向的对象已经不复存在。这就是 std::weak_ptr 的功能所在，std::weak_ptr 不能被单独使用，它是std::shared_ptr 作为参数的产物，std::weak_ptr 通常由一个 std::shared_ptr 来创建。

```c++
auto spw = std::make_shared<Widget>();	// after spw is constructed the pointed-to
										// Widget's ref count(RC) is 1. 
										// (See Item21 for info on std::make_shared)
...
std::weak_ptr<Widget> wpw(spw);			// wpw points to same Widget as spw.RC remains 1
...

spw = nullptr;							// RC goes to 0,and the Widget is destroy.
										// wpw now dangles

if (wpw.expired()) ...					// if wpw doesn't point to an object...
```

​	悬挂的 std::weak_ptr 可以称作是过期了（expired），可以直接通过 expired 成员函数检查。通常我们最常用的逻辑是：查询 std::weak_ptr 是否已经过期，如果没有过期的话，访问它所指向的对象。但是实现起来没这么简单，因为 std::weak_ptr 缺少解引用操作，也就是没办法完成这样操作的代码。即使有，将检查和取消引用分开也会引入竞争：在对 expired 的调用和取消引用操作之间，另一个线程可能会重新分配或销毁指向该对象的最后一个 std::shared_ptr，从而导致该对象被销毁。这种情况下，您的解引用会产生未定义的行为。

​	解决这个问题需要提供原子操作，用于检查 std::weak_ptr 是否过期，如果没有则允许访问它指向的对象。这是通过 std::weak_ptr 创建一个 std::shared_ptr 来完成的。该操作有两种形式，这取决于尝试从中创建std::shared_ptr 时，如果 std::weak_ptr 已过期，希望发生什么。一种形式是 std::weak_ptr::lock，它返回一个 std::shared_ptr。如果 std::weak_ptr 已过期，则 std::shared_ptr 为 null：

```c++
std::shared_ptr<Widget> spw1 = wpw.lock();		// if wpw's expired,spw is null
auto spw2 = wpw.lock();							// same as above,but uses auto
```

​	另一种形式是 std::shared_ptr 构造函数以 std::weak_ptr 作为参数。在这种情况下，如果 std::weak_ptr 已过期，则会抛出异常：

```c++
std::shared_ptr<Widget> spw2(wpw);	// if wpw's expired, throw std::bad_weak_ptr
```

​	你可能仍然想知道 std::weak_ptr 有什么用处。考虑一个工厂函数，它根据唯一 ID 生成指向只读对象的智能指针。根据 Item18 关于工厂函数返回类型的建议，它返回一个 std::unique_ptr:

```c++
std::unique_ptr<const Widget> loadWidget(WidgetID id);
```

​	如果 loadWidget 是一个开销很大的调用（例如，因为他执行文件或数据库I/O）并且 ID 被重复使用是很常见的，一个合理的优化是编写一个函数执行 loadWidget 所作的事情，但也缓存它的结果。然而，用每个请求过的 Widget 阻塞缓存可能会导致其自身的性能问题，因此另一个合理的优化是不再使用时销毁缓存的 Widget。

​	对于这个缓存的工厂函数，std::unique_ptr 返回类型并不适合。调用者当让应该收到指向缓存对象的智能指针，调用者当然应该确定这些对象的生命周期，但缓存需要一个指向对象的指针。缓存的指针需要能够检测到他们何时悬空，因为当工厂客户端使用完工厂返回的对象时，该对象将被销毁，相应的缓存条目将被悬空。因此缓存的指针应该是 std::weak_ptr ---- 可以检测它们何时悬空的指针。这意味着工厂的返回类型应该是 std::shared_ptr，因为只有当对象的生命周期由 std::shared_ptr 管理时，std::weak_ptr 才能检测它们何时悬挂。这是 loadWidget 缓存版本的实现：

```c++
std::shared_ptr<const Widget> fastLoadWidget(WidgetID id)
{
    static std::unordered_map<WidgetID, std::weak_ptr<const Widget>> cache;
    auto objPtr = cache[id].lock();	// objPtr is std::shared_ptr to cached object
    								// (or null if object's not in cache)
    if (!objPtr)
    {
        objPtr = loadWidget(id);	// if not in cache,
        cache[id] = objPtr;			// load it cache it
    }
    
    return objPtr;
}
```

​	示例中使用 C++11 的哈希表容器（std::unordered_map），尽管它没有提供所需的WidgetID算法以及相等的比较函数。

​	fastLoadWidget 的实现不是很完美，因为他忽略了一个事实，缓存可能把一些已经过期的 std::weak_ptr（对应的Widget不会使用了，已经被销毁了）。所以他的实现还可以在改善一下，但是我们还是不用深究了，以为深究对我们继续深入了解 std::weak_ptr 没有用处。我们不如探究第二种使用 std::weak_ptr 的场景：观察者模式。这种模式的主要组成部分是主要侧组成部分是 objects【主体】（状态可能发生变化的对象）和 obversers【观察者】（状态发生变化时要通知的对象）。大多数实现中，每个主体包含一个指向其观察者的指针数据成员。这使得主体可以轻松发布状态更改通知。主体不用控制观察者的生命周期（即，观察者啥时候销毁），但是必须确保当观察者被销毁后不能访问它。一个合理的设计是让每个主体持有一个关联观察者的 std::weak_ptr，从而使主体可以在使用指针前确定指针是否悬垂。

​	考虑一个包含对象A，B 和 C 的数据结构，其中A和C共享B的所有权，因此持有B的 std::shared_ptr：

​	![image-20210903153241294](C:\Users\1\AppData\Roaming\Typora\typora-user-images\image-20210903153241294.png)

​	假设有一个从 B 返回到 A 的指针，这应该是什么类型的指针呐？这里有三个选择：

* 原始指针。使用这种方法，如果 A 被销毁，但 C 继续指向 B，B 将包含一个指向 A 的指针，该指针将悬垂。B 将无法检测到这个变化，因此如果 B 对该指针进行解引用会造成未定义的行为。
* std::shared_ptr。在此设计中，A 和 B 彼此包含 std::shared_ptr。由此产生的 std::shared_ptr 循环（A指向B 和B指向A）将阻止 A 和 B 被销毁。即使 A 和 B 无法从其他数据结构访问（例如，因为C不再指向B），它们的引用计数都是1。如果发生这种情况，A和B将被泄露，出于实际目的：程序将无法访问它们，资源永远不会被回收。
* std::weak_ptr。这中方式避免了上述两个问题。如果 A 被销毁，B 指向它的指针将悬空，但是 B 将能够检查到这一点。此外，虽然 A 和 B 会相互指向，但是 B 的指针不会影响 A 的引用计数，因此当 std::shared_ptr 不再指向它时，不会影响 A 销毁。

​	使用 std::weak_ptr 显然是最好的选择。std::shared_ptr 循环中使用 std::weak_ptr 打破循环并不常见。在定义比较严格的数据结构，如树，子节点一般被父节点拥有。当父节点被析构时，子节点也应该被析构。从父结点指向子节点的链接因此最好使用 std::unique_ptr。因为子节点不应该比父结点存在的时间长，从子结点指向父节点的链接可以安全的使用原生指针来实现，解引用也不会存在危险。

​	当然，并不是所有的以指针为基础的数据结构都是严格的层级关系。如果不是的话，就像刚才所说的缓存以及观察者列表的情形，使用 std::weak_ptr 是最棒的选择了。

​	从效率的观点来看，std::weak_ptr 和 std::shared_ptr 的情况基本相同。std::weak_ptr 和 std::shared_ptr 对象大小相同，它们使用与 std::shared_ptr 相同的控制块（参看 Item19），并且诸如构造、销毁和赋值之类的操作涉及原子引用的计数操作。这可能让你吃了一惊，因为本章开始的时候说 std:::weak_ptr 不参与引用计数的操作。那里的意思是 std::weak_ptr 不参与对象的共享所有权，因此不影响被指向对象的引用计数。但是，实际上在控制块中存在第二个引用计数，std::waek_ptr 操作这个引用计数（参看Item 21）。

**小结**

* 将 std::weak_ptr 用于可能悬挂的 std::shared_ptr 指针
* 潜在使用 std::weak_ptr 的场景包括缓存，观察者列表，以及阻止 std::shared_ptr 循环

----

#### *Item 21. 优先使用 std::make_unique 和 std::make_shared 而不是 new*

​	std::make_shared 是 C++11 的一部分，但是 std::make_unique 不是，它是从 C++14 开始加入标准库。如果你使用的是 C++11，也不用担心，因为 std::make_unique 的基本版本很容易自己实现。如下：

```c++
template<typename T, typename... Ts>
std::unique_ptr<T> make_unique(Ts&&... params)
{
    return std::unique_ptr<T>(new T(std::forwaord<Ts>(params)...));
}
```

​	如你所见，make_unique 只是完美转发了它的参数列表到它要创建对象的构造函数中，由 new 出来的原生指针构造一个 std::unique_ptr，并将它返回。这种方式的构造不支持数组以及自定义 deleter（参看Item18），但是它说明只需稍加努力，便可自己创建出所需要的 make_unique。不要把自己实现的版本放在命名空间 std 下面，因为假如说日后你升级到 C++14 的标准库实现，你可不想自己实现的版本和便准库提供的版本产生冲突。

​	std::make_unique 以及 std::make_shared 是三个 make 函数中两个：make 函数接受任意数量的参数，然后将它们完美转发给动态创建的对象的构造函数，并且返回指向那个对象的智能指针。第三个 make 函数是 std::allocate_shared，除了第一个参数是一个用来动态分配内存的 alloctor 对象，他表现起来就像 std::make_shared。

​	即使对使用和不使用 make 函数创建智能指针进行比较，也揭示了为什么使用此类函数更可取的第一个理由，考虑：

```c++
auto upw1(std::make_unique<Widget>());			// with make func
std::unique_ptr<Widget> upw2(new Widget);		// without make func
auto spw1(std::make_shared<Widget>());			// with make func
std::shared_ptr<Widget> spw2(new Widget);		// without make func
```

​	上例中不使用 make 函数的例子会重复使用 Widget：使用 new 需要重复写一遍类型，而使用 make 函数不需要。重复敲 type 违背了软件工程的一个基本原则：代码重应该避免。源代码里面的重复会使得编译次数增加，导致对象的代码变得臃肿，由此产生的 code base 变得难以改动以及维护。他经常会导致产生不一致的代码。一个 code base 中的不一致代码会导致 bug，并且敲某段代码两遍会比敲一遍更费事，谁不想写程序时少敲点呐。

​	第二个偏向 make 函数的原因是为了保证产生异常后程序的安全。假设我们有一个函数根据某个优先级来处理Widget:

```c++
void processWidget(std::shared_ptr<Widget> spw, int priority);
```

​	通过值传递 std::shared_ptr 可能看起来可疑，但 Item41 解释了如果 processWidget 总是制作 std::shared_ptr 的副本（例如，通过将其存储在跟踪已处理的 Widget 的数据结构中），这可能是合理的设计选择。现在假设我们有一个函数来计算相关的优先级，

```c++
int computePriority();
```

我们在调用 processWidget 时使用 new 方式而不是  std::make_shared：

```c++
// potential resource leak!
processWidget(std::shared_ptr<Widget>(new Widget), computePriority());
```

正如注释所指出的，这段代码可能会泄漏由 new 变出的 Widget。但是怎么可能呐？调用代码和被调用函数均使用了防止内存泄漏的 std::shared_ptr。这个跟编译器将源代码翻译成目标代码有关。在运行时，必须在调用函数之前，函数的参数必须被推算出来，因此在对 processWidget 的调用中，必须在 processWidget 开始执行之前发生以下事情：

* "new Widget"表达式必须被执行，即必须在堆上创建一个 Widget
* 必须执行负责管理 new 产生的指针的 std::shared_ptr<Widget> 的构造函数
* computePriority 必须执行

但是编译器并不会对这些操作按如上顺序进行。"new Widget" 必须在 std::shared_ptr 的构造函数被调用之前执行，因为 new 的结果作为该构造函数的一个参数，因为 computePriority 可能在这些调用之前执行，或者之后，更关键的是，或者在它们之间。这样的话，编译器可能按如下操作顺序产生代码：

1、执行 "new Widget"

2、执行 computePriority

3、执行 std::shared_ptr 的构造函数

​	如果生成这样的代码，并且在运行时，computerPriority 产生了异常，那么步骤1中动态分配的 Widget 将被泄漏，因为它永远不会存储在应该在步骤3中开始管理它的 std::shared_ptr 中。使用 std::make_shared 可以避免这个问题，调用代码如下：

```c++
// no potential resource leak
processWidget(std::make_shared<Widget>(), computePriority());
```

​	在运行时，std::shared_ptr 和 computerPriotity 不管先调用谁。如果是 std::make_shared，则在调用 computePriority 之前，指向动态分配的 Widget 的原始指针安全地存储在返回的 std::shared_ptr 中。如果随后 computePriority 产生异常，则 std::shared_ptr 析构函数将确保它拥有的 Widget 被销毁。如果首先调用 computePriority 并产生异常，则不会调用 std::make_shared，因此无需担心动态分配的Widget。

​	如果我们用 std::unique_prt 和 std::make_unique 替换 std::shared_ptr 和 std::make_shared，完全相同的同理适用。因此，在编写异常安全代码时使用 std::make_unique 替代 new 与使用 std::make_shared 一样重要。

​	std::make_shared 的一个特殊功能（与直接使用 new 相比）是提高了效率。使用 std::make_shared 允许编译器利用简洁的数据结构产出更简洁，更快的代码。考虑下面直接使用 new 的效果：

```c++
std::shared_ptr<Widget> spw(new Widget);
```

​	很明显的情况是代码只需一次内存分配，但实际上它执行了两次。Item19 解释了每个 std::sahred_ptr 都指向了一个包含被指向对象的引用计数的控制块，控制块的分配工作在 std::shared_ptr 的构造函数内完成。直接使用 new，就需要一次为 Widget 分配内存，第二次需要为控制块分配内存。如果使用的是 std::make_shared：

```c++
auto spw = std::make_shared<Widget>();
```

​	一个分配就够了。那是因为 std::make_shared 分配了一块内存用来保存 Widget 对象和控制块。这个优化减少了程序的静态大小，因为代码中只包含了一次内存分配调用，并且加快了代码的执行速度，因为内存只被分配一次。此外，使用 std::make_shared 消除了对控制块中某些薄记信息的需要，可能会减少程序的总内存占用。

​	std::make_shared 的效率分许同样适用于 std::allocate_shared，因此 std::make_sahred 的性能优势也扩展到该函数。

​	优先使用 make 函数而不是直接使用 new 的论据是强有力的。尽管它们具有软件工程、异常安全和效率优势，但是，本 Item 的指导是更喜欢 make 函数，而不是完全依赖它们。那是因为在某些情况下不能或不应该使用它们。

​	例如，没有一个 make 函数允许自定义删除器的规范（参看 Item18 和 Item19），但 std::unique_ptr 和 std::shared_ptr 都有构造函数可以这样做。给定一个 Widget 的自定义删除器，

```c++
auto widgetDeleter = [](Widget *pw){...};
std::unique_ptr<Widget, decltype(widgetDeleter)>
    upw(new Widget, widgetDeleter);
std::shared_ptr<Widget> spw(new Widget, widgetDeleter);		// make 函数做不到
```

​	make 函数的第二个限制源于其实现的语法细节。第7条解释了当创建一个对象的类型重载构造函数时，无论是否有 std::initializer_list 参数，使用大括号创建对象首选 std::initializer_list 构造函数，而使用括号创建对象调用非 std::initializer_list 构造函数。make 函数将它们的参数完美地传递给对象的构造函数，但它使用括号还是大括号来实现的呐？对于某些类型，这个问题的答案有很大的不同。例如，这些调用中：

```c++
auto upv = std::make_unique<std::vector<int> >(10,20);
auto spv = std::make_shared<std::vector<int> >(10,20);
```

生成的智能指针是指向具有10个元素的 std::vector，每个元素的值为20，还是指向具有两个元素的 std::vector，一个是10，一个是20？还是结果不确定？好消息是它不是不确定的：两个调用都会创建一个大小为10的 std::vector，所有值都设置为20。这就意味着在 make 函数中，完美的转发代码使用括号，而不是大括号。坏消息是，如果想使用花括号初始化构造指向对象，则必须直接使用 new。使用 make 函数需要能够完美转发花括号初始化器，但是，正如 Item30 所解释的，花括号初始化器不能完美转发。但是，Item30 还描述了一种解决方法：使用自动推导从花括号初始化器（参看Item2）创建一个 std::initializer_list 对象，然后通过 make 函数传递自动创建的对象：

```c++
// create std::initializer_list
auto initList = {10, 20};
// create std::vector using std::initializer_list ctor
auto spv = std::make_shared<std::vector<int> >(initList);
```

​	对于 std::unique_ptr，这两个场景（自定义删除器和花括号初始化器）是其 make 函数有问题的唯一场景。对于 std::shared_ptr 及其 make 函数，还有两个。两者都是边缘情况，但是一些开发者生活在边缘，而你可能就是其中之一。

​	一些类定义了自定义的 operator new 和 operator delete 。这些函数的存在意味着全局的分配和回收方法不再适用。通常情况下，这种自定义的 new 和 delete 都被设计为只分配或者销毁恰好是一个属于该类的对象大小的内存，例如，Widget 的 new 和 delete 操作符经常被设计为：只是处理大小是 sizeof(Widget) 的内存块的分配和回收。而 std::shared_ptr 支持的自定义的分配（通过 std::allocate_shared）以及回收（通过自定义的edleter）的特性，上文描述的过程就支持的不好了，因为 std::allocate_shared 所分配的内存大小不仅仅是动态分配对象的大小，它所分配的大小等于对象的大小加上一个控制块的大小。所以，使用 make 函数创建的对象类型如果包含了此类版本的 new 和 delete 操作符重载，此时使用 make 确实不合适。

​	使用 std::make_shared 相对于直接使用 new 的大小及性能优点源自于：std::shared_ptr 的控制块是和被管理的对象放在同一个内存块中的。当该对象的引用计数变成了 0，该对象被销毁（析构函数被调用）。但是，它所占用的内存直到控制块被销毁才能被释放，因为被动态分配的内存块同时包含两者。

​	控制块除了它自己的引用计数，还记录了一些其他的信息。引用计数记录了多少个 std::shared_ptr 引用了当前控制块，但是控制块还包含了第二个引用计数，记录了多少个 std::weak_ptr 引用了当前的控制块。第二个引用计数被称之为 weak_count（备注：在实际情况中，weak count 不总是和引用控制块的 std::weak_ptr 的个数相等，库的实现往 weak count 添加了额外的信息来生成更好的代码（facilitate better code generation）。但为了本 Item 的目的，我们忽略这个事实，假设他们是相等的)。当 std::weak_ptr 检查它是否过期（请看 Item19）时，它看它所引用的控制块中引用计数（不是weak count）是否是0（即是否还有 std::shared_ptr 指向被引用的对象，该对象是否因为引用 0 被析构），如果是0，std::weak_ptr 就过期了，否则反之。

​	只要一个 std::weak_ptr 还引用着控制块（即，weak_count 大于 0），控制块就会继续存在，包含控制块的内存就不会被回收。被 std::shared_ptr 的 make 函数分配的内存，直至指向它的最后一个 std::shared_ptr 和 最后一个 std::weak_ptr 都被销毁时，才会得到回收。

​	如果对象类型非常大，并且销毁最后一个 std::shared_ptr 和 最后一个 std:weak_ptr 之间的时间很长，那么在对象被销毁和它占用的内存被释放之间可能会出现延迟：

```c++
class ReallyBigType{...};
auto pBigObj = std::make_shared<ReallyBigType>(); 	// create very large object via 
													// std::make_shared

...				// create std::shared_ptrs and std::weak_ptrs to large object,
    			// use them to work with it
    
...				// final std::shared_ptr to object destroyed here, but
    			// std::weak_ptrs to it remain
    
...				// during this period,memory formerly occupied by large
    			// object remains allocated
    
...				// final std::weak_ptr to object destroyed here;
    			// memory for control block and object is released
    
```

​	如果直接使用 new，一旦指向 ReallyBigType 的最后一个 std::shared_ptr 被销毁，对象所占用的内存马上得到回收。（本质上使用了 new，控制块和动态分配的对象所处的内存不在一起，可以单独回收）

```c++
class ReallyBigType {...};			// as before
// create very large object via new
std::make_shared<ReallyBigType> pBigObj(new ReallyBigType);

... 		// as before,create std::shared_ptrs and std::weak_ptrs to object
    		// use them with it
    
...			// final std::shared_ptr to object destroy here,but std::weak_ptrs to
    		// it remain; memory for object is deallocated
    
...			// during this period, only memory for the control block remains allocated
    
...			// final std::weak_ptr to object destroyed here;
    		// memory for control block is released
```

​	你发现自己处于一个使用 std::make_shared 不是很可行甚至不可能的境地，你想到了之前我们提到的异常安全的问题。实际上直接使用 new 时，只能保证你在一句代码中，只做了将 new 的结果传递给一个智能指针的构造函数，没有做其他事情。这也阻止编译器在使用 new 和调用来管理新对象智能指针的构造函数之间，插入可能会抛出异常的代码。

​	举个例子，对于之前检查的异常不安全的 processWidget 函数，我们在之上做个微小的修改。这次我们指定自定义的删除器：

```c++
void processWidget(std::shared_ptr<Wdiget> spw, int priority);		// as before
void cusDel(Widget* ptr);			// custom deleter
/// 这里有个异常不安全的调用方式,如下可能导致内存泄漏
processWidget(std::shared_ptr<Widget>(new Widget, cusDel), computePriority());
/// std::shared_ptr 构造函数调用前调用 computePriority 产生异常，则动态分配的内存被泄漏
```

​	在此我们使用了自定义的 deleter，所以就不能使用std::make_shared 了，想要避免这个问题，需要将 Widget 的分配和 std::shared_ptr 的构造放到自己的语句中，然后用得到的结果 std::shared_ptr 入参 processWidget 来调用。这是该技术的本质，我们可以对其进行调整以提高其性能：

```c++
std::shared_ptr<Widget> spw(new Widget, cusDel);
processWidget(spw, computePriority());		// correct,but not optimal;see below
```

​	确实可行，因为即使构造函数抛出异常，std::shared_ptr 也已经接收了传递给它的构造函数的原生指针的所有权。在本例中，如果 spw 的构造函数抛出异常（例如，假如因为无力去给控制块动态分配内存），它依然可以保证 cusDel 可以在 "new Widget" 产生的执政上调用。

在异常非安全的调用中，我们传递一个右值给 processWidget:

```c++
/// arg 是一个右值
processWidget(std::shared_ptr<Widget>(new Widget, cusDel), computePriority());
```

在异常安全的调用中，我们传递了一个左值：

```c++
/// arg 是一个左值
processWidget(spw, computePriority());
```

这就是造成性能问题的原因。因为 processWidget 的 std::shared_ptr 参数是按值传递的，从右值构造只需要一个移动，而从左值构造需要一个副本。对于 std::shared_ptr，差异可能很明显，因为复制 std::shared_ptr 需要对其引用计数进行原子增量，而移动 std::shared_ptr 则根本不需要引用计数的操作。为了使异常安全代码达到异常不安全代码的性能水平，我们需要将 std::move 应用到 spw 以将其转换为右值（参看 Item23）:

```c++
processWidget(std::move(spw), computePriority());		// both efficient and exception 
														// safe
```

**小结**

* 与直接使用 new 相比，make 函数消除了源代码重复，提高了异常安全性，并且对于 std::make_shared 和 std::allocate_shared，生成的代码更小、更快
* 不适合使用 make 函数的情况包括需要指定自定义删除器和希望传递带括号的初始化程序
* 对于 std::shared_ptr，make 函数可能不适用的情况包括：(1)具有自定义内存管理的类和(2)内存非常紧张的系统，非常大的对象以及对应的 std::shared_ptr 和 std::weak_ptr 生命周期很长 

----

#### *Item 22. 使用 Piml Idiom 时，在实现文件中定义特殊的成员函数*

​	如果你曾将因为程序过多的build次数头疼过，你肯定对于 Pimpl（Pointer to implementation）做法很熟悉。它的做法是：把对象的成员变量替换为一个指向已经实现类（或者是结构体）的指针。将曾经在主类中的数据成员迁移到该实现类中，通过指针来间接的访问这些数据成员。举个例子，假设 Widget 看起来像是这样：

```c++
class Widget{
  public:
    Widget();
    ...
  private:
    std::string name;
    std::vector<double> data;
    Gadget g1,g2,g3;
    // Gaget is some user-defined type
};
```

​	因为 Widget 的数据成员是 std::string、std::vector 和 Gadget 类型，所以这些类型的头文件必须存在以供 Widget 编译，这意味着 Widget 客户端必须 #include<string>、#include<vector>、#include<gadget.h>。这些头文件增加了 Widget 客户端的编译时间，而且它们使这些客户端依赖于头文件的内容。如果头文件内容改变，Widget 客户端必须重新编译。标准头文件<string>和<vector>不会经常更改，但是 gadget.h 可能会经常修改。

​	在C++98中应用 Pimpl Idion 可以让 Widget 将其数据成员替换为指向已声明但未定义的结构的原始指针：

```c++
class Widget{					// still in header "widget.h"
  public:
    Widget();
    ~Widget();					// dctor is needed-see below
    ...
        
  private:
    struct Impl;				// declare implementation struct
    Impl *pImpl;				// and pointer to it
};
```

​	由于 Widget 不再提及 std::string，std::vector 和 Gadget 类型，因此 Widget 客户端不再需要 #include 这些类型的头文件。这加快了编译速度，而且如果这些头文件的内容发生变化，Widget 客户端不会受到影响。

​	已声明但是未定义的类型称为不完整类型。Widget::Impl 就是这样的类型。对于不完整的类型，很少能操作它，但声明一个指向它的指针是可以的。Pimpl Idiom 利用的就是这一点。

​	Pimpl Idiom 的第一部分是数据成员的声明，该成员是指向不完整类型的指针。第二部分是对象的动态分配和销毁，该对象包含曾经在原始类中的数据成员。分配和释放代码在实现文件中，例如，对于 Widget，在 widget.cpp中：

```c++
#include "widget.h"					// in impl. file "widget.cpp"
#include "gadget.h"
#include <string>
#include <vector>

struct Widget::Impl{				// definition of Widget::Impl
	std::string name;				// with data members formerly
    std::vector<double> data;		// in Widget
    Gadget g1,g2,g3;
};

Widget::Widget()					// allocatte data members for
:pImpl(new Impl)					// this Widget object
{}

Widget::~Widget()					// destroy data memebers for this object
{delete pImpl;}
```

​	在这里，展示了 #include 指令，以明确对 std::string、std::vector 和 gadget.h 头文件的依赖。但是这些头文件已从 widget.h （对 Widget 客户端可见并由其使用）移至 widget.cpp（仅对 Widget 实现者可见并使用）。例子中还强调了动态分配和释放 Impl 对象的代码。当 Widget 被销毁时析构函数需要释放这个对象。这种代码风格延续了 C++98 风格（使用原始指针），本章建立在智能指针优于原始指针的思想之上，如果我们想要的是在Widget 构造函数中动态分配一个 Widget::Impl 对象并且在 Widget 销毁时同时销毁它，则 std::unique_ptr （参看Item18）正好满足我们的需求。如下代码：

```c++
class Widget{							// in "widget.h"
  public:
    Widget();
    ...
  private:
    stuct Impl;
    std::unique_ptr<Impl> pImpl;		// use smart pointer instead of raw pointer
};

/// implementition file
#include "widget.h"					// in "widget.cpp"
#include "gadget.h"
#include <string>
#include <vector>

struct Widget::Impl{				// as before
	std::string name;
    std::vector<double> data;
    Gadget g1,g2,g3;
};

Widget::Widget()					// per Item 21, create
:pImpl(std::make_unique<Impl>())	// std::unique_ptr
{}									// via std::make_unique
```

​	你会发现 Widget 析构函数不再存在，无显示释放资源。std::unique 会在它被销毁时自动删除它所指向的东西，所以我们不需要自己删除任何东西。这是智能指针的吸引力之一：它们消除了我们通过手动释放资源。

下面这段代码可以编译，但是客户端使用可能会有问题：

```c++
#include "widget.h"
Widget w;					// error!
```

​	收到的错误消息取决于使用的编译器，但是通常提示将 sizeof 或 delete 应用于不完整类型的内容，这些操作不是被禁止的。

​	Piml 做法结合 std::unique_ptr 竟然会产生错误，这很让人震惊。因为(1) std::unique_ptr 自身标榜支持不完整类型。(2)Piml 做法是 std::unique_ptr 众多的使用场景之一。幸运的是，让代码工作起来也是很简单的。我们首先需要理解为啥会出错。

​	在执行 w 被析构（如当出作用域时）的代码时，报错了。在此时，Widget 的析构函数被调用。在定义使用 std::unique_ptr 的 Widget 的时候，我们并没有声明析构函数，因为我们不需要在 Widget 的析构函数内写任何代码。依据编译器自动生成特殊成员函数（请参看Item17）的普通规则，编译器问问哦我们生成了一个析构函数。在哪个自动生成的析构函数中，编译器插入代码，调用 Widget 的数据成员 pImpl 的析构函数。pImpl 是一个 std::unique_ptr< Widget::Impl >，即一个使用默认 deleter 的 std::unique_ptr。默认 deleter 是一个函数，对 std::unique 里面的原生指针调用 delete。然而，在调用 delete 之前，编译器通常会默认 deleter 先使用 C++11 的 static_assert 来确保原生指针指向的类型不是不完整类型（staticassert 编译时候检查，assert 运行时候检查）。当编译器生成 Widget 的析构函数时，调用的 static_assert 检查就会失败，导致出现错误信息。在 w 被销毁时，这些错误信息才会出现，但是因为与其他的编译器生成的特殊成员函数相同，Widget 的析构函数也是 inline的。出错指向 w 被创建的那一行，因为改行创建了 w，导致后来（w 出作用域时）w 被隐性销毁。

​	为了修复这个问题，需要确保，在生成销毁 std::unique_ptr< Widget::Impl > 的代码时，WIdget::Impl 是完整的类型。当它的定义被编译器看到时，它就是完整类型了。而 Widget::Impl 在 widget.cpp 中被定义。所以编译成功的关键在于，让编译器只在 widget.cpp 内，在 widget::Impl 被定义之后，看到 Widget 的析构函数体（该函数体就是放置编译器自动生成销毁 std::unique_ptr 数据成员的代码的地方）。

要实现这个比较简单，在 widget.h 中声明 Widget 的析构函数，但是不定义：

```c++
class Widget{					// as before
  public:
    Widget();
    ~Widget();					// declaration only
    ...
  private:						// as before
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};

// 在widget.cpp里面的Widget::Impl定义之后再定义析构函数
#include "widget.h"			// as before, in "widget.h"
#include "gadget.h"
#include <string>
#include <vector>

struct Widget::Impl{			// Widget::Impl
	std::string name;
    std::vector<double> data;
    Gadget g1,g2,g3;
};

Widget::Widget()				// as before
:pImpl(std::make_unique<Impl>()) {}

Widget::~Widget(){}			// ~Widget definition
```

​	这样做就没问题了，增加的代码量也很少。但是如果你只是想将析构移动到实现文件中，并不想自定义（因为编译器默认生成的功能已满足）。那么可以使用 "=default" 定义析构函数：

```c++
Widget::~Widget()=default;			// same effect as above
```

​	现在使用 Pimpl Idiom 的类是支持移动操作的，因为编译器生成的移动操作完全符合：在底层 std::unique 上执行移动。正如 Item17 所解释的，Widget 中析构函数的声明会阻止编译器生成移动操作，因此如果想要支持移动操作，必须自定义这些函数。鉴于编译器生成的移动操作已经满足我们的需求，可以尝试如下实现：

``` c++
class Widget{				// still in "widget.h"
  public:
    Widget();
    ~Widget();
    
    Widget(Widget&& rhs) = default;					// right idea
    Widget& operator=(Widget&& rhs) = default;		// wrong code!
    
  private:											// as before
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};
```

​	跟之前没有析构函数的类出错原因类似，编译器生成的移动赋值运算符需要在重新赋值之前销毁 pImpl 指向的对象，但在 Widget 头文件中 pImpl 指向一个不完整的类型。移动构造函数的情况有所不同。不同在于编译器通常只会在移动构造函数中出现异常时才生成用于销毁 pImpl 的代码，并且要求销毁的 pImpl 是完整类型。

因为问题跟之前一样，所以修复-将移动操作的定义移至实现文件中：

```c++
class Widget{						// still in "widget.h"
  public:
    Widget();
    ~Widget();
    
    Widget(Widget&& rhs);				// declaration only
    Widget& operator=(Widget&& rhs);
    
    ...
  private:							// as before
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};

#include <string>					// as before, in widget.cpp
...
struct Widget::Impl{...};			// as before

Widget::Widget()
:pImpl(std::make_unique<Impl>()){}	// as before

Widget::~Widget() = default;		// as before

Widget::Widget(Widget&& rhs) = default;		// definitions
Widget& Widget::operator=(Widget&& rhs) = default;
```

​		Pimpl 做法是一种减少 class 的实现和 class 的使用之间编译依赖的一种方式，但是，从概念上来讲，这种做法并不改变类的表现方式。原来的 Widget 类包含了 std::string、std::vector 以及 Gadget 数据成员，并且假设 Gadget 像 std::string 和 std::vector 那样，可以拷贝。所以按理说 Widget 也要支持拷贝操作。我们必须自定义这些拷贝函数，因为（1）对于带有 move-only 类型（像 std::unique_ptr）的类，编译器不会生成拷贝操作的代码。（2）即使生成了，生成的代码也只会拷贝 std::unique_ptr（即，执行浅拷贝），而我们想要的是拷贝指针所指向的资源（即，深拷贝）。

现在的做法我们已经熟悉了，在头文件中声明这些函数，然后在实现文件中实现这些函数：

```c++
class Widget{								// still in "widget.h"
  public:
    ...										// other funcs, as before

	Widget(const Widget& rhs);				// declarations only
    Widget& operator=(const Widget& rhs);

  private:									// as before
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};

#include "widget.h"							// as before,in "widget.cpp"
...

struct Widget::Impl{...};					// as before
Widget::~Widget() = default;				// other funcs, as before

Widget::Widget(const Widget& rhs)			// copy ctor
: pImpl(std::make_unique_ptr<Impl>(*rhs.pImpl))
{}

Widget& Widget::operator=(const Widget& rhs)// copy operator
{
    *pImpl = *rhs.pImpl;
    return *this;
}
```

​	两种功能实现都很常规。在每种情况下，我们只是将 Impl 结构的字段从源对象 rhs 复制到目标对象 *this。我们不是一个一个地复制字段，而是利用这样一个事实：编译器会为 Impl 申城拷贝操作地代码，这些操作会自动将 struct 的内容逐项拷贝，就不需要我们手动来做了。我们因此通常调用 Widget::Impl 的编译器生成的拷贝操作符来实现了 Widget 拷贝操作符。在拷贝构造函数中，注意到我们遵循了 Item21 的建议，不直接使用 new，而是优先使用 std::make_unique。

​	上面的例子中，为了实现 Pimpl 做法，std::unique_ptr 是我们使用的智能指针类型，因为对象内 pImpl 对应的事项对象（Widget::Impl 对象）拥有独占所有权。但是，很有意思的是，当我们对 pImpl 使用 shared_ptr 来替代 std::unique_ptr ，发现本章的建议不再适用了。没必要在 Widget.h 中声明析构函数，没有了用户自定义的析构函数，编译器会自动生成 move 操作代码，而且生成的代码表现行为正好满足我们的需求，widget.h 实现如下：

```c++
class Widget{							// in "widget.h"
  public:
    Widget();
    ...									// no declarations for dtor or move operations

  private:
    struct Impl;
    std::shared_ptr<Impl> pImpl;		// std::shared_ptr instead of std::unique_ptr
};

// client code 
Widget w1;
auto w2(std::move(w1));					// move-construct w2
w1 = std::move(w2);						// move-assign w1
```

所有代码都会如我们所有通过编译，w1 会被默认构造，它的值会被 move 到 w2，之后 w2 的值又被 move 会 w1。最后 w1 和 w2 都得到析构（这使得指向的 Widget::Impl 对象被析构）。

​	对 Pimpl 应用 std::unique_ptr 和 std::shared_ptr 的表现行为不同的原因是：它们之间支持自定义的 deleter 的方式不同。对于 std::unique_ptr，deleter 的类型是智能指针的一部分，这就使得编译器生成更小的运行时数据结构以及更快的运行时代码成为可能。更好效率的结果是要求当编译器生成特殊函数（如析构函数以及移动操作）被使用时，std::unique_ptr 所指向的类型必须是完整的。对于 std::shared_ptr 来说，deleter 的类型不是智能指针的一部分。虽然会造成较大的运行时数据结构和慢一些的代码。但是在调用编译器生成的特殊函数时，指向的类型不需要是完整的。

​	对于 Pimpl 做法，在 std::unique_ptr 和 std::shared_ptr 的特点之间，其实并没有一个真正的权衡。因为 Widget 和 Widget::pImpl 之间是独占的拥有权，std::unique_ptr 在此项工作中很合适。然而，在一些其他的场景中，共享使用有权存在，std::shared_ptr 才是一个合适的选择，就没有必要像 std::unique_ptr 这样的函数定义的做法了。

**小结**

* Pimpl Idiom 通过减少类客户端和类实现之间的编译依赖性来减少构建时间
* 对于 std::unique_ptr pImpl 指针，在类头文件中声明特殊成员函数，但在实现文件中实现它们。即使默认函数实现是可以接受的，也要这样做
* 上述建议适用于 std::unique_ptr，但不适用于 std::shared_ptr
