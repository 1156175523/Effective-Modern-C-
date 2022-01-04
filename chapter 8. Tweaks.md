# chapter 8.  Tweaks

​		对于 C++ 中每一个通用技术或特性，都有合理的使用情况，也有不合理的使用情况。描述什么时候使用通用技术或特性是有意义的，因为这样做通常更简单，但本章涵盖两个例外。一般技术是使用按值入参，一般特性是使用进位（emplace）。关于何时适应它们决定因素有很多，但提倡尽量使用 emplace。尽管如此，两者都是有效的现代 C++ 编程的重要成员，后面的Item提供了在你的软件中哪里合适使用它们。

#### *item 41. 考虑传递可复制参数的值，这些参数移动操作一般比总复制更高效*

​		一些函数参数旨在被复制。例如，成员函数 addName 可能会将其参数赋值到私有容器中。为了效率，这样的函数赋值左值参数，或移动右值参数：

```c++
class Widget{
  public:
    void addName(const std::string& newName)	// take lvalue;
    { names.push_back(newName); }				// copy it
    
    void addName(std::string&& newName);		// take rvalue;
    { names.push_back(std::move(newName)); }	// move it; see Item25 for 
    											// use of std::move
    
  private:
    std::vector<std::string> names;
};
```

​		这样是可行的，但它需要写两个本质上相同的函数。这一点有点麻烦：两个函数声明，两个函数实现，两个函数需要记录，两个函数要维护。

​		此外，目标代码中有两个函数--当两个函数被内联后，可能会出现膨胀问题导致程序的空间占用变高，如果没有内联，那么会在你的目标代码中出现两个函数。

​		另外一种方法是使 addName 成为一个具有通用引用的函数模板（参看Item24）：

```c++
class Widget{
  public:
    template<typename T>							// take lvalues and
    void addName(T&&)								// rvalues; copy lavlues,
    {												// move rvalues; see
        names.push_back(std::forward<T>(newName));	// Item 25 for use of
    }												// std::forward
};
```

​		这方式可以减少源代码，但是通用引用的使用会导致其他复杂的问题。作为模板，addName 的实现通常必须在头文件中。它可以在对象中生成多个函数，因为它不仅可以对左值和右值进行实例化，还可以对 std::string 和 可转换为 std::string 的类型进行实例化（参看Item25）。与此同时，有些参数类型不能通过通用引用传递（参看Item30），如果客户端传递不正确的参数类型，编译器会提示一些很难理解的错误信息（参看Item27）。

​		是否有一种方法可以编写像 addName 这样的函数，使得左值复制，右值被移动，不管在源码还是目标代码中都只有一个函数要处理，并且还能避免使用通用引用，岂不是很好？确实有。你所要做的就是放弃作为 C++ 程序员学到的第一条规则。这条规则是避免按值传递用户定义的类型对象。对于像 addName 函数中 newName 这样的参数，按值传递可能是一个完全合理的策略。

​		对于 addName 中 newName 为什么适合按值传递之前，我们先看一下它是如何实现的：

```c++
class Widget{
  public:
    void addName(std::string newName)
    { names.push_back(std::move(newName)); }	// take lvalue or rvalue; move it
    ...
};
```

​		这段代码唯一不明显的部分是 std::move 对参数 newName 的应用。通常，std::move 与右值引用一起使用，但在这种情况下，我们直到（1）newName 是与调用者传入的任何内容完全独立的对象，因此更改 newName 不会影响调用者，并且（2）这也是 newName 的最终用途，因此移动它不会对函数的其他部分产生任何影响。

​		以上代码中只有一个 addName 函数实现了不管是源码还是目标代码中可以避免代码重复。因为没有使用到通用引用所以也不会导致头文件膨胀、奇怪的故障情况或混淆的错误信息。但是这种设计的效率如何呐？我们通过值传递，岂不是消耗更大。

​		在 C++98 中，这种理解是合理的。无论调用者传参什么，参数 newName 都将通过复制构造创建。但是，在 C++11 中，addName 只对左值进行复制构造。对于右值，它将会进行移动构造。如下：

```c++
Widget w;
...
std::string name("Bart");
w.addName(name);				// call addName with lvalue
...
w.addName(name + "Jenne");		// call addName with rvalue(see below)
```

​		在第一次调用 addName 时（当name被传递时），参数 newName 被初始化为左值。因此，newName 是复制构造的，就像在 C++98 中一样。在第二个调用中，newName 是调用 std::string 的 operator+ （即追加操作）而得到的结果。这个对象是个右值，这时 newName 将会进行移动构造。

​		这样一来我们的代码就看起来简洁了许多。但是还需要记住一些注意事项。如果我们回顾一下我们考虑过的 addName 的三个版本，这样做会更简单：

```c++
class Widget{
  public:
    void addName(const std::string& newName)	// Aproach 1: overload for lvalues and
    { names.push_bak(newName); }				// rvalues
    
    void addName(std::string&& newName)
    { names.push_back(std::move(newName)); }
    ...
};

class Widget{
  public:										// Aproach 2: use universal reference
    template<typename T>
    void addName(T&& newName)
    { names.push_back(std::forward<T>(newName)); }
    ...
};

class Widget{
  public:
    void addName(std::string newName)			// Aproach 3: pass by value
    { names.push_back(std::move(newName)); }
    ...
};
```

​		我们称前两个版本称为"按引用方法"，因为它们都基于通过引用传递参数。以下是我们可能使用到这些调用的场景：

```c++
Widget w;
...
std::string name("Bart");
w.addName(name);					// pass lvalue
...
w.addName(name + "Jenne");			// pass rvalue
```

​		在这两个场景中使用我们实现的三个 Widget版本，从复制和移动操作的角度考虑一下性能消耗。这种性能消耗计算会忽略编译器对复制和移动操作的优化（如果有的话），因为这种优化依赖于上下文和编译器，在实践中不会改变分析的本质。

* 重载：无论传递的是左值还是右值，调用者的实参都被绑定到一个名为 newName 的引用。就复制和移动操作来说，这没有任何成本消耗。在左值重载中，newName 被复制到 Widget::names 中。在右值重载中，它被移动到 Widget::names 中。成本消耗：左值一步拷贝，右值一步移动。

* 通用引用指针：与重载一样，调用者的实参被绑定到引用 newName。这是一个无消耗的操作。由于使用了 std::forward，左值 std::string 参数被复制到 Widget::names 中，而右值 std::string 参数被移动。std::string  参数成本汇总与重载相同：左值一步复制，右值一步移动。

  Item25 阐述了如果调用者传递了一个不是 std::string 类型的参数，它会被转发给 std::string 构造函数，这可能会出现零复制或移动操作。因此，采用通用引用的函数是唯一有效的。但是，这并不会影响这个 Item 中的分析，所以我们这里假设调用者总是传递 std::string 参数，这样会更容易分析。

* 按值传递：无论传递的左值还是右值，都必须构造参数 newName。如果传入左值，则需要进行复制构造。如果传递了右值，则需要进行移动构造。在本函数体中，newName 被无条件地移动到 Widget::names 中。因此，成本汇总是左值一个副本复制加一个移动，右值两个移动操作。与引用方法相比，左值和右值都多移动了一步。

这跟本章强调的优先使用按值传递可复制地参数是不是有点违背呀？这么说有4个原因：

***1、******你应该只考虑使用按值传递***。是的，这样可以只编写一个函数。是的，它只在目标代码中生成一个函数。是的，它避免了与通用引用相关地问题。但是它的成本比其他选择要高，而且有些能耗还没有提到的，会在下面介绍。

***2、考虑只对可复制的参数按值传递***。不能通过此测试的参数肯定是 move-only 类型，因为如果它们不可复制，但函数总是进行复制，则必须通过 move 构造函数创建。回想一下，按值传递比重载传递的优势在于，在按值传递时，只需要编写一个函数。但是对于只能移动类型，不需要提供左值参数重载，因为复制左值需要调用复制构造函数，而仅移动类型的复制构造函数是禁用的。这就意味着只需要支持右值参数，在这种情况下，"重载"解决方案只需要一个重载：接受右值引用的那个。

​		考虑 Widget 类数据成员是 std::unique_ptr 。std::unique_ptr 是 move-only 的，所以"重载"方法的 setter 由单个函数组成：

```c++
class Widget{
  public:
    ...
	void setPtr(std::unique_ptr<std::string>&& ptr)
    { p = std::move(ptr); }
  private:
    std::unique_ptr<std::string> p;
};

// 调用
Widget w;
...
w.setPtr(std::make_unique<std::string>("Modern C++"));
```

​		这里，std::unique_ptr\<std::string> 是 冲 std::make_unique（参看Iterm21） 返回的右值然后通过右值引用传递给 setPtr，最后它被移动到数据成员 p 中。总的消耗就是移动了一次。

如果 setPtr 是按值获取其参数，

```c++
class Widget{
  public:
    ...
	void setPtr(std::unique_ptr<std::string> ptr)
    { p = std::move(ptr); }
    ...
};
```

同样的调用将会移动构造参数 ptr，然后 ptr 将被移动赋值给数据成员 p。因此，总的消耗是两次移动---是"重载"方法的两倍。

***3、仅对于移动成本低的参数，值传递值得考虑***。当移动成本低时，额外的移动操作成本通常是可以接受的，但当移动成本不低时，执行不必要的移动类似于执行不必要的复制，这也是 C++98 中倡导避免按值传递的原因。

***4、应该只对总是被复制的参数考虑按值传递***。要了解为什么这么重要，假设在复制参数到 names 容器之前，addName 会检查 newName 是不是太短或太长然后忽略掉。按值传递的实现可以这样：

```c++
class Widget{
  public:
    void addName(std::string newName)
    {
        if ((newName.length() >= minLen) && (newName.length() <= maxLen))
        {
            names.push_back(std::move(newName));
        }
    }
    ...
  private:
    std::vector<std::string> names;
};
```

即使没有向 names 添加任何内容，此函数也会产生构造和销毁 newName 的成本。在按应用传递的实现中不会出现这种情况。

​		即使你在处理一个函数，该函数可复制类型进行无条件的复制，且这个类型的移动成本较低，但有时按值传递可能并不合适。这是因为函数可以通过两种方式复制形参：通过构造（即复制构造或移动构造）和通过赋值（即复制赋值或移动赋值）。addName 使用构造函数：它的形参 newName 被传递给 vector::push_back，在该函数中，newName 是通过复制构造给 std::vector 末尾创建的新元素中。对于使用构造函数来复制其形参的函数，我们前面分析已经完成：使用按值传递会导致左值和右值实参都要进行额外的移动操作。

​		当使用赋值复制参数时，情况更为复杂。例如，加入我们有一个表示密码的类。因为可以更改密码，所以我们提供了一个设置接口 changeTo。使用值传递策略，我们可以这样实现 Password：

```c++
class Password{
  public:
    explicit Password(std::string pwd)			// pass by value construct text
	:text(std::move(pwd)){}
    
    void changeTo(std::string newPwd)			// pass by value assign text
    { text = std::move(newPwd); }
    
    ...
  private:
    std::string text;							// text of password
};
```

将密码保存为纯文本会让你的软件存在安全隐患，但是请在这里忽略，考虑以下代码：

```c++
std::string initPwd("Supercalifragilisticexpialidocious");
Password p(initPwd);
```

​		这里毋庸置疑，p.text 使用给定的密码构造的，并且在构造函数中使用按值传递会产生 std::string 移动构造的成本，如果采用重载或完美转发，则不需要。

​		这个程序的用户可能对密码不那么乐观，因为在许多词典中都能可以找到"Supercalifragilisticexpialidocious"。因此她或他可能会采用导致执行以下代码的操作：

```c++
std::string newPassword = "Beware the Jabberwork";
p.changeTo(newPassword);
```

​		新密码是否优于旧密码暂且不说，这时用户的问题。我们的问题是，changeTo 使用赋值来复制参数 newPwd ，可能会导致该函数的值传递策略开销激增。

​		传递给 changeTo 的实参是一个左值（newPassword），因此当构造 newPwd 形参时，调用的是 std::string 复制构造函数。该函数分配内存来保存新密码。然后将 newPwd 移动赋值给 text，这将导致已经被 text 占用的内存被释放。因此，在 changeTo 中有两个动态内存管理操作：一个为新密码分配内存，一个为旧密码释放内存。

​		但在这种情况下，旧密码（"Supercalifragilisticexpialidocious"）比新密码（"Beware the Jabberwork"）长，所以不需要分配或释放任何东西。如果使用重载方法，很有可能什么都不会发生：

```c++
class Password{
  public:
    ...
	void changeTo(const std::string& newPwd)	// the overload for lvalues
    { text = newPwd; }			// can reuse text;s memory if 
    							// text.capacity() >= newPwd.size()
    
    ...
  private:
    std::string text;			// as above
};
```

​		在这种场景中，按值传递的成本包括额外的内存分配和回收成本，这些成本可能比 std::string 移动操作的成本高几个数量级。

​		有趣的是，如果旧密码比新密码短，通常不可能在分配过程中避免分配-回收，在这种情况下，按值传递将以按值引用传递相同的速度运行。因此，基于赋值的参数复制的代价取决于参与赋值的对象的值！这种分析适用于在动态分配内存中保存值得任何参数类型。不所有类型都符合条件，但很多类型都符合条件--包括 stdd::string 和 std::vector。

​		这种潜在得成本增加通常只出现在传递左值参数时，因为执行内存分配和回收得需要通常只发生在执行真正的复制操作（即，无移动）时。对于右值参数，移动几乎总是足够的。

​		结果就是，对于使用赋值复制形参的函数来说，按值传递的额外成本取决于所传递的类型、左值与右值参数的比值、该类型是否使用动态分配的内存，如果是，则该类型的赋值操作符以及与赋值目标关联的内存至少与赋值源关联的内存一样大的可能性。对于 std::string ，它取决于实现是否使用小字符串优化（SSO----参看Item29），如果是，则分配的值是否适合 SSO 缓存。

​		因此，正如我所说，当参数通过赋值复制时，分析按值传递的代价跟复杂。通常，最实用的方法是采用"已知无害"策略，即使用重载或通用引用，而不是按值传递，除非已经证明按值传递可以生成对所需参数类型有效的代码。

​		对于要求运行速度尽可能快的软件，按值传递不是一个可行的策略，因为即使是一个简单的移动操作优化也很重要。此外，我们并不总是清楚会有多少次的移动操作。在 Widget::addName 的例子中，按值传递只会引起一个额外的移动操作，但假设 Widget::addName 被 Widget::validateName 调用，这个函数也是按值传递。（大概它有一个总是复制它的参数的原因，例如，把它存储在一个包含所有它验证的值的数据结构中）并且假设 validateName 又被第三个函数调用了，它也是通过值传递。。。

​		这么一来你大概就明白了。当存在函数调用链时，每个函数调用链都采用按值传递，因为"它需要一个廉价的移动操作"，真个调用调用链的成本可能是无法容忍的。通过使用引用参数传递，调用链不会产生这种累积的开销。

​		一个与性能无关但仍然值得记住的问题是，按值传递与按引用传递不同，它容易受到切片问题的影响。这是我们熟悉的 C++98 基础，所以这里不多提，但如果你设计的一个函数，它存在与基类或者其他会存在派生情况的类型，你不能将其声明为按值传递，因为在派生过程中派生类对象会产生""切片现象"：

```c++
class Widget{...};								// base class
class SpecialWidget: public Widget{...};		// derived class

void processWidget(Widget w);		// func for any kind of Widget, including
									// derived types, suffers from slicing problem
...
SpecialWidget sw;
...
processWidget(sw);					// processWidget sees a Widget, now a 
									// SpecialWidget!
```

​		如果你不熟悉切片问题，可以在网上了解更多相关介绍，了解更多相关指示。你会发现切片问题的存在是C++98 不提倡值传递的另外一个原因（除了效率问题）。C++11 并没有从根本上改变 C++98 关于值传递的方法。通常，按值传递仍然会导致性能损失，并且仍然会导致切片问题。C++11 中新增的是针对左值和右值参数的优化。对于具有移动语义函数的可复制类型的右值使用重载或通用引用，都会存在缺点。对于可复制、移动成本低的类型的特殊情况，并且不考虑切片的情况下，值传递方式可以替代重载或通用引用方式，而且可一个避免它们的劣势。

   **小结**

* 对于可复制、移动成本低的参数，通常都是复制的，按值传递可能和按应用传递一样高效，更容易实现，生成的目标代码更少。
* 通过构造复制参数可能比通过赋值复制参数更消耗成本。
* 按值传递受到切片问题的影响，因此它通常不适用于基类参数类型。

#### *item 42. 考虑使用 emplace 而不是 insert*

​		如果你有一个容器，例如 ，std::stirng ，那么当你通过插入函数（即 insert、push_front、push_back或对于 std::forward_list、insert_after）添加新元素时，传递给函数的参数是 std::string 类型似乎是合乎逻辑的。毕竟这就是容器的内容。

​		尽管合理但是并不总是代表是对的。考虑如下代码：

```c++
std::vector<std::string> vs;			// container of std::string
vs.push_back("xyzzy");					// add string literal
```

这里容器包含的是 std::string，而你实际入参的是字符串常量。字符串常量并不是 std::string ，也就是说传递给 push_back 的参数不是容器所持有的类型。std::vector 的 push_back 对左值和右值的重载如下：

```c++
template<class T, 									// from the C++11 Stamdard
			class Allocator = allocator<T>>
class vector{
  public:
    ...
	void push_back(const T& x);						// insert lvalue
    void push_back(T&& x);							// insert rvalue
    ...
};
```

​		在调用 vs.push_back("xyzzy") 中，编译器会发现实参的类型（const char[6]）与 push_back 接受的形参类型（对 std::string 的引用）不匹配。它们通过将字符串常量创建一个临时 std::string 对象来解决不匹配的问题，并将该临时对象传递给 push_back。换句话说，它们对待这个调用好像是这样处理的：

```c++
vs.push_back(std::string("xyzzy"));	// create temp. std::string and pass it to push_back
```

​		这样代码编译通过并能正常运行，但是有效率问题。要在一个 std::string 容器中创建一个新元素，必须调用一个 std::string 构造函数，但上面的代码并不只调用一个构造函数。它会用到两个，同样也会调用 std::string 的析构函数。下面是在运行时调用 push_back 会发生的事情：

1、一个临时的 std::string 对象由字符串常量 "xyzzy" 而创建。这个对象没有名字；我没暂且叫它 temp。temp的构造是第一个 std::string。因为它是一个临时对象，所以 temp 是一个右值。

2、temp 被传递给 push_back 的右值重载，在那里它被绑定到右值引用参数 x。然后在内存中为 std::vector 构造一个 x 的副本。这个构造-----第二个构造----实际上在 std::vector 中创建了一个新对象。（用于将 x 复制到 std::vector 中的构造函数是 move 构造函数，因为 x 是一个右值引用，在它被复制之前被转换为右值。有关右值引用参数转换为右值的信息，参看 Item25）

3、在 push_back 返回后，立即销毁 temp，从而调用 std::string 析构函数。

​		考虑到性能方面的问题，如果有一个方法可以将字符串常量直接传递给步骤 2 中在 std::vector 中构造的 std::string 对象中，就可以成功避免临时变量的构造和析构。这个方法就是使用 emplace_back。

​		emplace_back 正是我们想要的：它使用传递给它的任何参数来直接在 std::vector 内部构造一个 std::string。不涉及临时变量：

```c++
vs.emplace_back("xyzzy");	// construct std::string inside vs directly from "xyzzy"
```

​		emplace_back 使用完美转发，因此，只要你触发完美转发短板（参看 Item30），你可以通过 emplace_back 传递任何数量的任意类型组合的函数。例如，如果你想通过 std::string 构造函数在 vs 中创建一个 std::string 并接受一个字符和一个重复计数，可以这样实现：

```c++
vs.emplace_back(50, 'x');	// insert std::string consisiting of 50 'x' characters
```

​		emplace_back 对于每个支持 push_back 的标准容器都是可用的。类似的，每个支持 push_front 的标准容器也支持 emplace_front。每个支持 insert 的标准容器（除了 std::forward_list 和 std::array 之外）都支持 emplace。关联容器提供 emplace_hint 来补充 insert 函数，它接受一个 "hint" 迭代器，而 std::forward_list 有 emplace_after 来匹配 insert_after。

​		使植入函数优于插入函数的原因是它有灵活的接口。插入函数需要接受要插入的对象，而植入函数可以入参对象构造函数所需的参数。这种差异使得植入函数避免创建和销毁临时对象。

​		因为容器持有的类型的实参可以传递给植入函数（实参因此导致函数执行复制或移动构造），所以即使插入函数不需要临时变量，也可以使用植入函数。这种情况下，植入函数和插入函数本质上一样。例如，给定，

```c++
std::string queenIfDisco("Donna Summer");
```

两种调用作用于容器的效率是一样的：

```c++
vs.push_back(queenOfDisco);		// copy-construct queenOfDisco at end of vs
vs.emplace_back(queenOfDisco);	// ditto
```

​		植入函数可以做插入函数几乎所有的事。而且效率可能会更高，为啥不一直使用插入函数呐？理论和实践是有区别的。在标准库的当前实现中，有些情况下，如预期的那样，植入函数性能由于插入函数，但是遗憾的是，还有一些情况插入函数的效率更高。这种情况不太好描述，因为这取决于所传递参数的类型、所使用的容器、容器中被要求插入或放置的位置、所包含类型构造函数的异常安全性，以及对于禁止重复值的容器（如，std::set，std::map，std::unorder_set，std::unorder_map），要添加的值是否已经在容器中。以此，通常性能优化的建议是：对它们进行测试。

但是当满足下面所有情况时，植入函数几乎肯定比插入效率更高：

* ***被添加的值被构造到容器中，而不是赋值***。像之前的例子（将一个值为"xyzzy"的 std::string 添加到一个 std::vector vs 中）显示了这个值被添加到 vs 的末尾，一个还不存在对象的地方。因此，新值必须构造到 std::vetor 中。如果我们修改这个例子，使新的 std::string 进入一个已经被对象占用的位置，那就是另一回事了。考虑：

```c++
std::vector<std::string> vs;			// as before
...										// add element to vs
vs.emplace_back(vs.begin(), "xyzzy");	// add "xyzzy" to beginning of vs
```

​		对于这段代码，很少有实现会将添加的 std::string 构造到 vs[0] 占用的内存中。相反，它们将 移动-赋值 对象到适当的位置。但是移动赋值需要一个对象移动，这就意味着需要创建一个临时对象来作为移动的源。由于植入相对于插入的主要优势在于，既不会创建也不会销毁临时对象，当通过赋值将添加的值放入容器时，植入的优势就不再体现。

​		 向容器添加值是通过构造函数还是赋值来完成 ，这通常取决于实现者。但是，基于节点的容器几乎总是使用构造来添加新值，而且大多数标准容器都是基于节点的。只有 std::vector，std::deque 和 std::string 不是。（std::array 也不是，但它不支持插入或植入，所以这里不相关）在非基于节点的容器中，你可以依赖 emplace_back 来使用构造而不应是赋值来获得新值，对于 std::deque，emplace_front 也是如此。

* ***传递的实参类型与容器持有的类型不同***。同样，植入优于插入同样源于这样一个事实：当传入的参数类型不是容器所持有的类型时，它的接口不需要创建和销毁临时对象。当要将 T 类型的对象添加到容器时，没有理由植入比插入运行更快，因为不需要创建临时对象来满足插入接口。
* ***容器不会拒绝重复的值***。这意味着容器要么允许重复，要么添加的对象要保持唯一，这个问题的原因是，为了检查容器中是否已经存在一个值，植入实现通常会创建一个带有新值的节点，以便将该节点的值与现有的容器节点进行对比。如果要添加的值不在容器中，则将节点链接到其中。但是，如果这个值已经存在，那么部署就会种植，节点就会被破坏，这就意味着它会存在创建和销毁消耗。这样种情况植入不会比插入高效。

之前的调用满足上述所有条件，它们比相应的 push_back 调用运行更快。

```c++
vs.emplace_back("xyzzy");	// construct new value at end if container;
							// don't pass the type in container;
							// rejecting duplicates

vs.emplace_back(50,'x');	// ditto
```

​		在决定是否使用植入功能时，还有两个问题需要留意。**首先是资源管理**。假设你有一个 std::shared_ptr<Widget> 的容器,

```c++
std::list<std::shared_ptr<Widget>> ptrs
```

并且你想添加一个 std::shared_ptr，它应该通过自定义删除器释放（参看Item19）。Item21 解释了你应该使用 std::make_shared 来创建 std::shared_ptr，但是它也承认了有些情况下你不能这样做。其中一种情况适当你想要指定自定义删除器时。这种情况下，你必须直接使用 new 来获得由 std::shared_ptr 管理的原始指针。

```c++
// 如果自定删除器是这个函数
void killWidget(Widget* pWidget);
// 使用插入函数调用如下
ptrs.push_back(std::shared_ptr<Widget>(new Widget, killWidget));
// 也可以这样实现
ptrs.push_back( {new Widget, killWidget} );
```

​		上面无论使用哪个方式，在调用 push_back 之前都会构造一个临时的 std::shared_ptr。push_back 的参数是对 std::shared_ptr 的引用，所以必须有一个 std::shared_ptr 来引用这个参数。

​		emplace_back 可以避免创建临时 std::shared_ptr，但在本例中，这个临时对象存在的价值超它能耗。考虑发生以下事件：

1、在上面的任意一个调用中，都构造了一个临时的 std::shared_prt 对象来保存 "new Widget" 生成的原始指针。叫这个对象 temp。

2、push_back 通过引用获取临时变量。在分配 list 节点保存  temp 副本期间，抛出内存不足异常。

3、当异常从 push_back 传播出去时，会销毁 temp。被 std::shared_ptr 所管理的 Widget，会自动调用 killWidget 释放。

​		也就是说即使发生异常，也不会有任何内存泄漏：通过 "new Widget" 在 push_back 调用中创建的 Widget 在 std::shared_ptr 的析构函数中释放，该析构函数是为了管理它（temp）而创建的。

现在再来看一下如果使用 emplace_back 会发生什么：

```c++
ptrs.emplace_back(new Widget, killWidget);
```

1、从 "new Widget" 得到的原始指针被完美转发到 emplace_back 内部，然后到分配 list 节点时，报错并抛出内存不足异常。

2、当异常从emplace_back 传播出去时，原始指针（在堆上获取 Widget 的唯一方法）丢失了。Widget （以及它所拥有的资源）会泄露。

​		在这种情况下，错不再 std::shared_ptr。通过使用带有自定义删除器的 std::unique_ptr 可能会出现同样的问题。从根本上说，像 std::shared_ptr 和 std::unique_ptr 这样的资源管理类的有效性取决于被立即传递给资源管理对象的构造函数的资源（例如来自 new 的原始指针）。这也是 std::make_shared 和 std::make_unique 函数自动执行的重要原因。

​		在调用保存资源管理对象容器的插入函数（如，std::list\<std::shared_ptr\<Widget\>\>）时，函数的参数类型通常确保在获取资源（例如，使用 new）和构造管理资源的对象之间没有任何东西。在植入函数中，完美转发憨痴了资源管理对象的创建，直到它们可以在容器中构造，这就导致了出现资源泄露的机会。所以标准容器都容易受到这个问题的影响。在使用资源管理对象的容器时，如果选择植入函数而不是插入函数，必须注意确保降级异常安全性而提高代码效率。

​		无论如何，你不应该将 "new Widget" 这样的表达式传递给 emplace_back 或 push_back 或大多数其他函数，因为正如 Item21 所述那样，这可能会导致我们刚才检查过的那种异常安全问题。不使用 "new Widget" 获取指针，而是使用单独的语句将其转换为一个资源管理对象，然后将该对象作为右值传递给最初希望将 "new Widget" 传递给的函数。（Item21 有详细介绍）因此，使用 push_back 的代码应该这样写:

```c++
// push_back
std::shared_ptr<Widget> spw(new Widget, killWidget);	// create Widget and have
														// spw manage it

ptrs.push_back(std::move(spw));							// add spw as rvalue

// emplace_back
std::shared_ptr<Widget> spw(new Widget, killWidget);
ptrs.emplace_back(std::move(spw));
```

​		无论哪种方式，都会有创建和销毁 spw 的成本。选择植入函数而不是插入函数的动机是避免创建临时变量，但是这就是 spw 的概念，当将资源管理器对象读入到容器中时，植入函数的性能不太可能优于插入函数，并且你遵循了确保在获取资源和将其转交给资源管理对象之间没有任何干扰的正确实践。
​		植入函数第二个值得注意的方面是与显式构造函数的交互。假设创建一个正则表达式对象的容器：

```c++
std::vector<std::regex> regexes;
//假如有如下调用
regexes.emplace_back(nullptr);		// add nullptr to container of regexes?
```

​		当出现如上失误时，但编译正常，会浪费时间去调试定位问题。但是为啥可以插入一个空针？指针不是正则表达式呀，如果你尝试这样做，

```c++
std::regex r = nullptr;		// error! won't compile
```

编译时不通过的。有趣的是，如果你调用push_back，也会提示报错：

```c++
regexes.push_back(nullptr);	// error! won't compile
```

导致这个奇怪行为的原因是：std::regex 对象可以由字符串构造。这就是为什么会有如下合法有用的调用方式：

```c++
std::regex upperCaseWord("[A-Z]+");
```

从一个字符串创建一个 std::regex 可能会产生一个相对较大的运行时开销，因此，为了即按照这种无意中产生的开销，带有 const char* 指针的 std::regex 构造函数是 explicit（禁止隐式类型转换）。这就是为什么这些语句为啥编译不过的：

```c++
std::regex r = nullptr;		// error! won't compile
regexes.push_back(nullptr);	// error! won't compile
```

这两种情况下，我们都请求从指针到 std::regex 的隐式转换，而该构造函数的 explicit 声明防止这种转换。然而，在对 emplace_back 的调用中，我们并没有声明传递一个 std::regex 对象。相反，我们将传递 std::regex 的一个构造函数。这不是隐式转换请求，而是被看作如下：

```c++
std::regex r(nullptr);			// compiles
```

虽然可以编译，但是会有未定义的行为。接受 const char* 指针的 std::regex 构造函数要求所指向的字符串包含一个有效的正则表达式，而空指针并不符合。使用 push_back 和 emplace_back 相似的初始化语法却会产生不同的结果：

```c++
std::regex r1 = nullptr;
std::regex r2(nullptr);
```

​		 在标准的官方术语中，用于初始化 r1 的语法（使用等号）对应于所谓的复制初始化。相反，用于初始化 r2 的语法（使用括号，但也可以使用大括号）产生了所谓的直接初始化。复制初始化不允许使用显式构造函数。直接初始化是没问题。这就是导致以上 r1 不能编译，r2 可以编译的原因。

​		回到 push_back 和 emplace_back，emplace_back 使用直接初始化，这就意味它可以在使用显式构造函数。push_back 使用复制初始化，所以它不能。因此：

```c++
regexes.emplace_back(nullptr);		// compile.Direct init permits use of explicit
									// std::regex ctor taking a pointer

regexes.push_back(nullptr);			// error! copy init forbids use of that ctor
```

​		当你使用植入函数时，要特别小心确保你传递的参数是正确的，因为即使显示构造函数也可以被编译器考虑，因为它们会尝试找到一个方法来保证你的代码编译合法。

   **小结**

* 原则上，植入函数有时应该比插入函数更有效，而且绝不能降低效率
* 在实践中，当 (1) 添加的值被构造到容器而不是赋值时; (2)传递的实参类型不同于容器持有的类型时；(3) 容器不会拒绝添加的值没因为它是一个副本时；使用植入函数效率更高
* 植入函数可以执行"类型转换"，而插入函数会拒绝这些转换
