# chapter 3.  Moving to Modern C++

​	C++11 和 C++14 有很多重要的特性，auto、智能指针、移动语义、Lamdba、并发---每个都很重要，这一章我们专门来简述这些新特性。本章将讲解到什么时候应该使用大括号而不是括号来创建对象？为什么别名声明比typdef更好？constexpr 与const 有何不同？const 成员函数和线程安全有什么关系？。。。

#### *item 7. 创建对象时区分 () 和 {}*

​	根据我们对 C++11 的认识，初始化语法选择会存在多种。作为一般规则，初始化值可以使用括号、等号或者大括号指定：

```c++
int x(0);		// initializer is in parentheses
int y = 0;		// initializer follow "="
int z{0};		// initializer is in braces
```

​	许多情况下，也可以同时使用等号和大括号：

```c++
int z = {0};
```

​	这里我们忽略同时使用等号和大括号的语法，因为 C++ 通常将其视为仅用大括号的版本。初始化的标志经常误导新手认为赋值正在发生，但其实并不是。对于像 int 这类内置类型，区别是学术性的（？？？？不太懂这个翻译），但是对于用户自定义的类型，区别初始化和赋值很重要，因为涉及不同的函数调用：

```c++
Widget w1;		// call default constructor
Widget w2 = w1;	// not an assigment; calls copy ctor
w1 = w2;		// an assigment; calls copy operator=
```

​	即使有几种初始化语法，但是某些情况下 C++98 也无法表达所需的初始化。例如，无法指示应该创建一个包含一组特定值（eg:1,2,3,5）的 STL 容器。为了解决多个初始化语法的问题，以及不能覆盖所有初始化场景，C++11 引入了统一初始化：一个单一的初始化语法，至少在概念上，它可以在任何地方使用并表达一切。因为它基于大括号，因此我更喜欢称它"大括号初始化"。 "统一初始化"是一个想法，"花括号初始化"是一种语法结构。

​	花括号初始化可以让我们表达以前无法表达的东西。使用花括号，指定容器的初始内容：

```c++
std::vector<int> v{1,3,5};		// v's initial content is 1,3,5
```

​	花括号还可以被用于非静态数据成员指定默认初始值。此功能（C++11新特性）与 "=" 初始化类似，但括号初始化可能不好使：

```c++
class Widget{
  ...
  private:
    int x{0};		// fine, x's default value is 0
    int y = 0;		// also fine
    int z(0);		// error!
};
```

​	另一方面，不可复制的对象（例如，std::atomic --- 见 item40）可以使用花括号或圆括号初始化，但是不能使用 "=":

```c++
std::atomic<int> ai1{0};		// fine
std::atomic<int> ai2(0);		// fine
std::atomic<int> ai3 = 0;		// error!
```

​	因此很容易理解为什么花括号初始化被称为"统一"。在 C++ 的三种初始化表达式中，只有花括号是可以用于所有地儿的。花括号初始化的一个新特性是它禁止内置类型之间的隐式转换。如果花括号中的表达式的值不能保证与初始化对象的类型相同，则代码编译不过：

```c++
double x,y,z;
...
int sum1{x + y + z};	// error! sum of doubles may not be expressible int
```

​	使用括号和"="则不会进行这种检查，而是进行了强转：

```c++
int sum2(x+y+z);		// okay (value of expression truncated to an int)
int sum3 = x + y + z;	// ditto
```

​	另一个值得注意的特性是它可以规避 C++ 烦人的解析。这种解析体现为，任何可以被解析为声明的东西都会被理解为一个声明，如下：

```c++
Widget w1(10);		// call Widget ctor with argument 10
```

​	上面的初始化没有问题，但是如果你想尝试使用类似的语法调用一个带有零参数的Widget构造函数，那你其实是声明了一个函数而不是一个对象：

```c++
Widget w2();		// monst vexing parse! declares a function named w2 that returns a 						// Widget
```

​	函数列表不能使用花括号声明函数，所以使用花括号构造对象不存在这个问题：

```c++
Widget w3{};		// calss Widget ctor no args
```

​	上面个讲述了花括号在上下文中应用最广泛，可以防止隐式类型转换，还可以避免C++的解析影响。这么多好处为啥我们直接都是用花括号呐？花括号的缺点是它有时一些令人惊讶的行为。这种行为源于花括号、std::initializer_list 和构造函数重载解析之间异常纠结的关系。他们的交互可能导致代码看起来应该做一件事，但实际上做另外一件事。例如，Item2 解释了当自动变量具有花括号初始化时，推导类型是 std::initializer_list，即使使用相同初始化器声明变量的其他方式会产生更直观的类型。因此，你越喜欢使用 auto，你可能不太可能使用花括号初始化。

​	在构造函数调用中，只要不涉及 std::initializer_list 参数，括号和花括号就具有相同的含义:

```c++
class Widget {
  public:
    Widget(int i, bool b);		// ctors not declaring
    Widget(int i, double d);	// std::initializer_list params
    ...
};
Widget w1(10, true);		// calls first ctor
Widget w2{10, true};		// als calls firest ctor
Widget w3(10, 5.0);			// calls second ctor
Widget w4{10, 5.0};			// also calls second ctor
```

​	但是如果有一个或者多个构造函数声明为 std::initializer_list 类型的参数，则花括号初始化语法会直接调用 std::initializer_list 的重载。如果编译器有任何方法可以将使用花括号初始化的调用转换为 std::initializer_list 的构造函数调用时，则编译器优先使用 std::initializer_list 的构造函数。如下添加 Widget std::initializer_list 参数的构造函数：

```c++
class Widget{
  public:
    Widget(int i, bool b);			// as before
    Widget(int i, double d);		// as before
    Widget(std::initializer_list<long, double> il);	// added
    ...
};
```

​	Widget w2 和 w4 将会调用新加的构造函数，即使参数是 std::initializer_list<long, double>  与非 std：：initializer_list 构造函数相比匹配更差！再看：

```c++
Widget w1(10, true);		// uses parens and, as before, calls firest ctor

Widget w2{10, true};		// user braces, but now calls std::initializer_list ctor
							// (10 and true convert to long double)

Widget w3(10, 5.0);			// user parens and, as before calls second ctor

Widget w4{10, 50};			// uses braces, but now calls std::initializer_list ctor
							// (10 and 5.0 convert to long double)
```

​	甚至复制和移动构造也会收到 std::initializer_list 构造的影响：

```c++
class Widget{
public:    
  Widget(int i, bool b);		// as before
  Widget(int i, double d);		// as before
  Widget(std:;initializer_list<long, double> il);	// as before
  operator float() const;		// convert to float
  ...
};
Widget w5(w4);		// uses params, calls copy ctor

Widget w6{w4};		// uses braces, calls std::initializer_list ctor
					// (w4 convert to float, and float convert to long double)

Widget w7(std::move(w4));	// uses params, calls move ctor

Widget w8{std::move(w4)};	// uses braces, calls std::initializer_list ctor
							// (for same reason as w6)
```

​	编译器使用花括号初始化是会最高优先级选择 std::initializer_list 的构造函数，即使无法匹配最佳形式。在看个例子：

```c++
class Widget{
  public:
    Widget(int i, bool b);		// as before
    Widget(int i, double d);	// as before
    Widget(std::initializer_list<bool> il);	// element type is now bool
    ...							// no implicit conversion funcs
};

Widget w{10, 5.0};		// error! requires narrowing conversions
```

​	在这个例子中使用花括号前两个构造函数回避忽略，即使第二个构造函数完全匹配，但是编译器还是会调用 std::initializer_list 参数的构造函数。之后会尝试将 int(10) 和 double(5.0) 转换为 bool，但是这个无法实现，所以代码会编译不过。

​	只有当无法将花括号初始化表达式中的参数类型转换为 std::initializer_list 中的类型时，编译器才会回调用正常的重载。例如，如果我们将 std::initializer_list<bool> 构造函数替换为采用 std::initializer_list<std:string> 的构造函数，则非 std::initializer_list 构造函数才会被调用，因为无法将 int 和 bool 转换为 std::string ：

```c++
class Widget{
  public:
    Widget(int i, bool b);		// as before
    Widget(int i, double d);	// as before
    //std::initializer_list element type is now std::string
    Widget(std::initializer_list<std::string> il);	
    ...							// no implicit conversion funcs
};

Widget w1(10, true);	// uses parens, still calls first ctor
Widget w2{10, true};	// uses braces, now calls first ctor
Widget w3(10, 5.0);		// uses parens, still calls second ctor
Widget w4{10, 5.0};		// uses braces, now calls second ctor
```

​	到这儿基本上有关花括号初始化的内容基本告一段落，但是有个特殊情况就是空的花括号代表啥？空的花括号其实意味着没有参数，所以体会调用默认构造而不是空的 std::initializer_list :

```c++
class Widget{
  public:
    Widget();								// default ctor
    Widget(std::initializer_list<int> il);	// std::initializer_list ctor
    ...		// no implicit conversion funcs
};

Widget w1;		// calls default ctor
Widget w2{};	// also calls defalt ctor
Widget w3();	// most vexing parse! declares a function!
```

​	如果你想用一个空的 std::initializer_list 调用一个 std::initializer_list 构造函数，你可以通过将花括号作为构造函数来实现--通过将空的花括号放在括号或大括号内来界定你传递的内容：

```c++
Widget w4({});		// calls std::initializer_list ctor with empty list 
Widget w5{{}};		// ditto
```

​	讲了这么多你可能会想这玩意有啥用呀？哎！你别说还真有点用，而且这些细节比想象中要重要。受影响的其中之一就是 std::vector，std::vector 有一个非 std::initializer_list 构造函数，它允许指定容器的初始大小和每个元素应该具有的值，但还有一个构造函数它允许使用 std::initialzier_list 指定容器中的初始值。假设我们创建一个std::vector<int> 并将两个参数传递给构造函数，那么将两个参数括在括号中与花括号中会产生巨大的差异：

```c++
std::vector<int> v1(10, 20);	// use non-std::initializer_list ctor: create 
								// 10-element std::vector, all element have value of 20
std::vector<int>  v2{10, 20};	// use std::initializer_list ctor: create 2-element
								// std::vector, element values are 10 and 20
```

​	总而言之，第一点，在添加 std::initializer_list 重载接口时需要谨慎。第二点，使用这类重载是需要了解其原理。再来看另外一个例子，假如你想从任意数量参数来创建任意类型的对象，可变参数模板可以方便实现：

```c++
template<typename T, typename... Ts>
void doSomeWork(Ts&&... params)		// type of object to create types of arguments to use
{
    create local T object from params...
	...
}
```

​	有两个方法可以将伪代码行转换为真正的代码（有关 std::forward 的信息，item25 详解）：

```c++
T localObject(std::forward<Ts>(params)...);	// using parens
T localObject{std::forward<Ts>(params)...};	// using braces
```

​	所以你考虑这个调用代码：

```c++
std::vector<int> v;
...
doSomeWork<std::vector<int>>(10, 20);
```

​	如果 doSomeWork 在创建 localObject 时使用括号，则结果是具有10个元素的 std::vector。如果 doSomeWork 使用大括号，则结果是带有2个元素的 std::vector。哪个正确呐？doSomeWork 的作者不可能知道，只有调用者才知道。这正是标准库函数 std::make_unique  和 std::make_shared 面临的问题（参看 item21）。这些函数通过在内部使用括号并将此决定记录为其接口的一部分来解决问题。

   **小结**

* 花括号初始化是使用最广泛的初始化语法，它可以避免隐式类型转换，并且可以避免C++的犯人解析问题
* 在构造函数重载解析期间，花括号初始化的表达式会优先匹配 std::initializer_list ，即使不是完全匹配
* 括号和花括号之间的选择会导致很大的差别，如 std::vector<numerc type>
* 括号和花括号的使用会导致模板创建对象出现不确定性

---

#### *item 8. 尽量使用 nullptr 而不是 0 和 NULL*

​	从字面看：0是一个int，而不是一个指针。如果C++发现自己在只能使用指针的上下文使用0，它会勉强将0理解为空指针，但这只是备用选择。C++的主要策率规则是将0理解为int，而非指针。NULL是整形也非int，所以这里重点强调0和NULL非指针类型。在 C++98 中，这意味着重载指针和证书可能会导致意外，将0 或 NULL 传递给此类重载从未称为指针重载：

```c++
void f(int);	// there overload f
void f(bool);
void f(void*);
f(0);			// calls f(int), not f(void*)
f(NULL);		// might not complie, but typically calls f(int). Never calls f(void*)
```

​	从上面的现象不难看出为啥 C++98 会强调避免重载指针和整数类型的指南（C++11 也一样）。nullptr 是非整数类型，但它也非指针类型，你可以视它为所有类型的指针。 nullptr 实际是 std::nullptr_t ，而且 std::nullptr_t  可以隐式转换为所有原始指针类型，这就使得 nullptr 表现得好像它是所有类型的指针。

​	使用 nullptr 调用重载函数 f 会调用 void* 重载（即指针重载），因为 nullptr 不能被视为任何整形：

```c++
f(nullptr);		// calls f(void*) overload
```

​	除了避免重载解析意外，nullptr 还会使得代码变得清晰，尤其是涉及到自动变量时。如下：

```c++
auto result = findRecord(/* arguments */);
if (result == 0)
{
    ...
}

auto result = findRecord(/* arguments */);
if (result == nullptr)
{
    ...
}
```

​	这两部分代码对比我们会发现使用 nullptr 会更加直观的阅读代码含义是返回指针类型数据。这个情况在函数模板使用中会体现的更加明显，假设你有一些函数只有获取锁后才会被调用，每个函数的指针类型不同：

```c++
int f1(std::shared_ptr<Widget> spw);	// call these only when the appropriate mutex is 
double f2(std::unique_ptr<Widget> upw);	// locked
bool f3(Widget* pw);
```

​	想要传递空指针的调用如下：

```c++
std::mutex f1m, f2m, f3m;		// mutexs for f1, f2, f3
using MuxGuard = 				// 
    std::lock_guard<std::mutex>;
...
{
    MuxGuard g(f1m);		// lock mutex for f1
    auto result = f1(0);	// pass 0 as null ptr to f1
}							// unlock mutex
...
{
    MuxGuard g(f2m);		// lock mutex for f2 
    auto result = f2(NULL);	// pass NULL as null ptr to f2
}							// unlock mutex
...
{
    MuxGuard g(f3m);			// lock mutex for f3 unlock 
    auto result = f3(nullptr);	// pass nullptr as null ptr to f3
}								// unlock mutex
```

​	前两次调用未能使用 nullptr 但是代码依旧有效。这导致无效的加锁解锁的流程，这个情况可以通过模板设计避免，来看下面例子：

```c++
// C++11 version
template<typename FuncType, typename MuxType, typename PtrType>
auto lockAndCall(FuncType func, MuxType& mutex, PtrType ptr) -> decltype(func(ptr))
{
    MuxGuard g(mutex);
    return func(ptr);
}

//C++14 version
template<typename FuncType, typename MuxType, typename PtrType>
decltype(auto) lockAndCall(FuncType func, MuxType& mutex, PtrType ptr)
{
    MuxGurad g(mutex);
    return func(ptr);
}

// callers call 
auto result1 = lockAndCall(f1, f1m, 0);		// error!
...
auto result2 = lockAndCall(f2, f2m, NULL);	// erorr!
...
auto result3 = lockAndCall(f3, f3m, nullptr);	// fine
```

​	第一个失败的原因是：0 是 int 类型，模板特化 PtrType 被推导为 int，然后调用 f1(shared_ptr<Widget>) 时入参 int 类型会出现不配匹的情况，所以编译不通过。

​	第二个失败的原因是：跟第一个类似，因为 NULL 是整形类型，PtrType 推导的类型于 f2(unique_ptr<Widget>)参数不匹配，所以编译不通过。

​	第三个成功的原因是：nullptr 入参 PtrType 推导类型是 std::nullptr_t 。当传递给 f3(Widget*) 时，因为存在从std::nullptr_t 隐式转换为所有指针类型，所以编译通过。

**小结**

* 优先使用 nullptr 而非 0 和 NULL 来表示空指针。
* 避免重载整形和指针类型。

-----

#### *Item 9. 优先使用别名而非 typedef*

​	因为STL使用起来非常丝滑所以很多程序员爱不释手，但是有个恶心的地方就是名称冗长的情况，如下 std::unique_ptr<std::unordered_map<std::string,  std::string>> ，为了避免这个情况引入 typedef：

```c++
typedef std::unique_ptr<std::unordered_map<std::string,  std::string>> UPtrMapSS;
```

typedef 是 C++98 引入的，当然 C++11 也支持，但是 C++11 还引入了一个别名功能：

```c++
using UPtrMapss = std::unique_ptr<std::unordered_map<std::string,  std::string>>;
```

而且在声明函数指针时使用别名会比 typedef 刚容易理解：

```c++
// FP is synonym for a pointer to a function taking an int and
// a const std::string* and returning nothing
typedef void (*FP) (int, const std::string);		// typedef
using FP = void(*)(int, const std::string);			// alias decaration
```

​	这时你会疑问既然他们的功能相同，那我该如何选择呐？当然是有选择规则的，在使用模板时，别名可以被模板化（这种情况下被称为别名模板），而 typedef 不能。所以 C++11 可以直接使用别名表示模板，typedef 则必须嵌套在模板内。例如，使用自定义分配器 MyAlloc 的链表：

```c++
/// using alias
template<typename T>							// MyAllocList<T> is synonym for 
using MyAllocList = std::list<T, MyAlloc<T>>; 	// std::list<T, MyAlloc<T>>
MyAllocList<Widget> lw;							// client code

// using typedef
template<typename T>							// MyAllocList<T>::type
struct MyAllocList{								// is synonym for
  typedef std::list<T, MyAlloc<T>> type;	  	// for std::list<T, MyAlloc<T>>
};
MyAllocList<Widget>::type lw;					// client code
```

​	如果在模板内使用 typedef 定义的 list 类型并这个 list 类型依赖模板参数，这个时候情况会变得更糟，因为必须在 typedef 定义的类型前添加  typename:

```c++
template<typename T>
class Widget{							// Widget<T> contains a MyAlloc<T>
  private:								// as a data member
    typename MyAllocList<T>::type list;
    ...
};
```

​	这里，MyAllocList<T>::type 的特化类型依赖模板参数（T）。因此 MyAllocList<T>::type 是一个依赖类型，C++ 规定依赖类型的名称必须在类型名称之前。但是如果使用 using 别名就不用这么麻烦：

```C++
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;		// as before
template<typename T>
class Widget{
  private:
    MyAllocList<T> list;			// no "typename", no "::type"
    ...
};
```

​	在你看来，使用 using 定义的别名 MyAllocList<T> 看起来和 typedef 定义的 MyAllocList<T>::type 一样依赖模板参数 T。但是编译器在处理 Widget 模板时，遇到 using 定义的别名模板 MyAllocList<T> 时知道 MyAllocList<T> 是一个类型名称，因为 MyAllocList 是一个别名模板：它必须命名一个类型。MyAllocList<T> 因此是一个非以来类型，并且既不需要也不允许使用类型说明符。当编译器遇到 typedef 定义的 MyAllocList<T> 时，它无法确定这是命名了一个类型，因为这可能是 MyAllocList 的特化（如果没使用 typename 的话）。再观察如下例子：

```C++
class Wine{...};
template<>						// MyAllocList specialzation for when T is Wine
class MyAllocList<Wine>{
  private:
    enum class WineType			// see Item 10 fro info on "enum class"
    {White, Red, Rose};
    
    WineType type;				// in this class, type is a data member!
    ...
};
```

​	*如你所见，MyAllocList<Wine>::type 不是一种类型。如果使用 Wine 实例化 Widget，则 Widget 模板内的 MyAllocList<T>::type 是指一个数据成员，而不是类型。那么，在 Widget 模板内部，MyAllocList<T>::type 是否指的是一种类型实际上取决于 T 是什么，这就是为什么编译器坚持要通过在它前面加上 typename 来断言它是一种类型。*

​	*如果你接触过模板元编程（TMP），你肯定遇到过需要采用模板类型参数并从中创建修改后的类型需求。例如，给定某种类型 T，你可能希望去除 T 包含的任何常量或引用限定符。又如，你可能希望将 const std::string& 转换为 std::string 。或者，你可能希望向类型添加 const 或将其转换为左值引用，例如，将Widget 转换为  const Widget 或 Widget&*（item23 和 item27将会介绍有关模板元编程的相关知识）。---说实话这段没看懂

​	C++11 提供了工具来完成这类工作，表现得形式是 type traits，它是 <type_traits> 头文件中的各种模板。这个头文件中有数十种类型特征，但是并不是都可以提供类型转换，不提供转换的也提供了方便使用的接口。给定对应需要进行转换的类型 T，结果类型为 std::transformation<T>::type。例如：

```C++
// C++11 version
std::remove_const<T>::type			// yields T from const T
std::remove_reference<T>::type		// yields T from T& and T&&
std::add_lvalue_reference<T>::type	// yields T& from T
    
//C++14 version
std::remove_const_t<T>
std::remove_reference_t<T>
std::add_lvalue_reference_t<T>
    
    
// C++14 其实是对 C++11 添加了别名
template<class T>
using remove_const_t = typename remove_const<T>::type;
template<class T>
using remove_reference_t = typename remove_reference<T>::type;
template<class T>
using add_lvalue_reference_t =
typename add_lvalue_reference<T>::type;
```

**小结**

* typedef 不支持模板化，但是 using 别名支持
* 别名模板避免使用 ""::type" 后缀，并且在模板中，通常还需要使用 "typename" 前缀
* C++14 为所有 C++11 类型特征转换提供了别名模板

----

#### *Item 10. 优先使用作用域限制的 enum 而不是无作用域的 enum*

​	一般而言，在花括号声明的变量名作用域是括号内。但是对于 C++98 风格的 enum 中的枚举元素并不是这样，枚举元素和包含它的枚举类型同属于一个作用域空间，这就意味着这个作用域中不能再有同名定义：

```c++
enum Color{black, white, red};		// black, white, red 和 Color 同属于一个定义域
auto white = false;					// error!white already declared in this scope
```

​	C++11 针对枚举这种无作用域的情况，提供了作用域枚举：

```C++
enum class Color{black, white, red};		// black, white, red are scoped to Color
auto white = false;							// fine, no other "white" is scope
Color c = white;							// error! no enumerator named "white" 
											// is in this scope
Color c = Color::white;						// fine
auto c = Color::white;						// alse fine (and in accord with
											// Item5's advice)
```

​	因为作用域枚举是通过 "enum class" 声明的，所以有时也称它为枚举类。枚举类除了***作用域限制外***还有个特点是***类型强度强***，传统的枚举是可以隐式转换为整型类型（并进一步转换为浮点型）。来看例子：

```c++
// 传统枚举
enum Color{black, white, red};		// unscoped enum
std::vector<std::size> 				// func. returning
    primeFactors(std::size_t x);	// prime factors of x
Color c = red;
...
if (c < 14.5)						// compare Color to double(!)
{									
    auto factor = primeFactors(c);	// compute prime factors of a Color(!)
    ...
}

// 枚举类
enum class Color{black, white,red};		// enum is now scoped
Color c = Color::red;					// as before, but with scope qualifier
...
if (c < 14.5)							// error! can't compate Color and double
{
    auto factors = primeFactors(c);		// error! can't pass Color to function
    ...									// expecting std::size_t
}
```

​	但是如果你确实需要从Color类型转换为不同的类型，可以使用强制类型转换：

```c++
if (static_cast<double>(c) < 14.5)	// odd code, but it's valid
{
    auto factors = 
        primeFactors(static_cast<std::size_t>(C));	//suspect,but it compile
    ...
}
```

​	相比，传统枚举枚举类还有一个特点就是***前置声明，即可以在不定义枚举内容的情况下声明他们的名称***：

```c++
enum Color;			// error!
enum class Color;	//fine
```

​	有一些误导性的现象。在 C++11 中，传统枚举也是可以被前置的，但是必须经过一些额外的操作，C++ 中每个枚举都有一个由编译器确定的完整底层类型。拿 Color 传统枚举举例：

```c++
enum Color{black, white, red};
```

​	编译器可能会选择 char 作为底层类型，因为只有三个值要表示。但是，某些枚举的值范围要打的多，例如：

```c++
enum Status{
    good = 0,
    failed = 1,
    incomplete = 100,
    corrupt = 200,
    indeterminate = 0xFFFFFFFF
};
```

​	一般来说为了充分利用内存，编译器会选择一个足以表示枚举的最小类型。但是有时为了运行速度进行优化会牺牲内存大小，这种情况下编译器可能不会选择占用内存最小的可允许的潜在类型，但是依旧可以优化内存存储。为了实现这一点，C++98 仅支持枚举的定义（所有枚举元素被列举出来）而不允许使用枚举声明。这样可以保证在枚举类型被使用前，编译器已经给每个枚举类型选择潜在存储规则。不能事先声明枚举类型有几个不足。最引人注意的就是***会增加编译依赖性***。再来看上面Status枚举，这个枚举会在整个系统中都会被用到，因此引用的地方都依赖一个头文件。当引入一个新枚举成员时：

```c++
enum Status{
    good = 0,
    failed = 1,
    incomplete = 100,
    corrupt = 200,
    audited = 500,
    indeterminate = 0xFFFFFFFF
};
```

​	如果任何一个子系统，即使一个函数用到这个新元素，有可能导致整个系统的代码需要被重新编译。在 C++11 中，消除了这种情况：

```c++
enum class Status;		// 前置声明
void continueProcessing(Status s);	// 使用前置声明枚举
```

​	如果 Status 的定义被修改（如上添加 audited 枚举元素），包含这个声明的头文件不需要重新编译。因为 continueProcessing 没有使用 audited，cintunueProcessing 的实现也不需要重新编译。

​	但是如果编译器需要在枚举之前知道它的大小，C++11 的枚举体怎么做到可以前置声明，而 C++98 的枚举体无法实现？原因很简单，对于枚举类的潜在类型是已知的，对于枚举类，默认潜在的类型是 int：

```c++
enum class Status;		// 潜在类型是 int
```

如果默认类型不适用你的使用情况，你可以重载它：

```c++
enum class Status: std::uint32_t;	// Status 潜在类型是std::unint32_t
```

为了给没有作用域的枚举体指定潜在类型，你需要做相同的事情，结果可能是前置声明：

```c++
enum Color: std::uint_8;		// 没用定义体的枚举的前置声明，潜在类型是 std::uint8_t
```

潜在类型的指定也可以放在枚举体的定义处：

```c++
enum class Status: std::uint32_t{
    good = 0,
    failed = 1,
    incomplete = 100,
    corrupt = 200,
    audited = 500,
    indeterminate = 0xFFFFFFFF
};
```

​	虽然枚举类有避免命名冲突、避免隐式类型转换等好处，但是传统的枚举类型在应用 C++11 std::tuple 中某个域时还是挺有的。例如，假设我们有一个元组，元组中保存着名称、电子邮件、影响力值：

```c++
using UserInfo = 				// 别名
    std::tuple<std::string,		// 名称
				std::string,	// 电子邮件
				std::size_t>;	// 影响力
```

尽管注释已经说明元组的每个部分代表什么意思，但是当你遇到像下面这样的源代码时，可能注释没什么用：

```c++
UserInfo uInfo;						// 元组类型对象
...
auto val = std::get<1>(uInfo);		// 得到第一个域的值

// 使用枚举类进行优化
enum UserInfoFields {uiName, uiEmail, uiReputation};
UserInfo uInfo;
...
auto val = std::get<uiEmail>(uInfo);	// 获取电子邮件的值
```

上面代码正常工作的原因是 UserInfoFields 到 std::get() 要求的 std::size_t 的隐式类型转换。如果使用枚举类去实现就会显的冗长：

```c++
enum class UserInfoFields{uiName, uiEmail, uiReputation};
UserInfo uInfo;
...
auo val =
    std::get<static_cast<std::size_t>(UserInfoFields::uiEmail)>(uInfo);
```

​	写一个以枚举元素为参数返回对应的 std::size_t 的类型值可以减少这种冗余性。 std::get 是一个模板，你提供的值是一个模板参数（注意用的是尖括号，不是圆括号），因此负责将枚举元素转化为 std::size_t 的函数必须在必须在编译阶段就确定它的结果。就像 item15 解释的，必须是一个 constexpr 函数。

​	实际上，这个函数应该是一个 constexpr 函数模板，因为它应该对任何类型的枚举体有效。如果我们要进行泛化，我们也应该泛化返回类型。我们应该返回枚举的底层类型而不是返回 std::size_t。它可以通过 std::underlying_type 类型获得（有关类型特征的信息，参看 item9）。最后，我们将它声明为 noexcept（参看item14)，因为我们知道他永远不会产生异常。结果是一个函数模板 toUType，接受任意枚举并可以将其值作为编译时常量返回：

```c++
template<typename E>
constexpr typename std::underlying_type<E>::type
    toUType(E enumerator) noexcept
{
    return
        static_cast<typename std::underlying_type<E>::type>(enumerator);
}
```

在 C++14 中，可以通过用更简洁的 std::undeyling_type_t 替换 typename std::underlying_type<E> ::type 来简化 toUType（参看 item9)：

```c++
template<typename  E>			
constexpr std::underlying_type_t<E>		// c++14
	toUType(E enumerator) noexcept
{
    return static_cast<std::underlying_type_t<E>>(enumerator);
}

// 使用 auto 进一步优化(item13)
template<typename E>
constexpr auto
	toUType(E enumerator) noexcept
{
    return static_cast<std::underlying_type_t<E>>(enumerator);
}
```

​	不管实现如何，toUtype 允许我们访问元素字段，如：

```c++
auto val std::get<toUType(UserInfoFields::uiEmail)>(uInfo);
```

**小结**

* C++98 的传统枚举是无作用域枚举
* 枚举类仅在枚举类内可见，枚举成员只能通过强制转换为其他类型
* 传统枚举和枚举类都支持支持指定底层类型，枚举类的默认基础类型是 int，传统枚举没有默认的基础类型
* 枚举类可以前置声明，传统枚举只有当它们的声明被指定了底层类型时，传统枚举才可以前置声明

----

#### *Item 11. 优先使用 delete 关键字删除函数而不是使用 private 修饰但不实现函数*

​	如果向其他人员提供代码但是不希望他们调用部分代码，通常不提供相关声明即可。但 C++ 会自动声明部分函数，如果阻止客户端调用这些函数就很麻烦了。这种情况只出现在"特殊成员函数"中，即C++在需要时自动生成的成员函数。item17 会详细讨论这些函数，这里我们只介绍拷贝构造和赋值构造。这一章主要介绍C++11对C++98进行优化的部分，在C++98中一般就是仅用拷贝构造或者赋值构造（或者两个都仅用）。

​	C++98仅用的方法是将这些函数声明为私有成员并不进行定义。例如，C++标准库中 iostream 的继承关系中基类是 basic_ios。所有 istream 和 ostream 类都是直接或间接继承它。复制 istream 和 ostream 是不可取的，因为不清楚这些操作该做什么。例如，一个 istream 对象表示一个输入流，其中一些可能已经被读取，而其中一些可能会在以后被读取。如果要复制 istream，是否需要复制所有以读取的值以及将要读取的所有值？处理此类问题的最简单方法是将他们定义为不存在。禁止复制流就是这么做的。为了使 istream 和 ostream 类不可复制，在C++98中指定了 basic_ios 如下：

```c++
template<class charT, class traits = char_traits<charT> >
class basic_ios: public ios_base{
	public:
    	...
	private:
    	basic_ios(const basic_ios&);			// not defined
    	basic_ios& operator=(const basic_ios&);	// not defined
};
```

​	将这些函数声明为私有成员可以防止客户端调用他们，故意不定义意味着如果仍然可以访问他们的代码（成员函数或类的友元）使用它们时，链接将会由于缺少函数定义而失败。C++11中有一个更好的方法来实现这个需求：使用 "=delete" 将拷贝构造和赋值构造标记为已删除的函数。这是在C++11中指定的 basic_ios 的相同部分：

```c++
template<class charT, class traits = char_traits<charT> >
class basic_ios: public ios_base{
    public:
    	...
        basic_ios(const basic_ios&) = delete;
    	basic_ios& operator(const basic_ios&) = delete;
		...
};
```

​	delete函数不能以任何方式使用，因此成员函数和友元函数代码中，如果尝试调用这些函数将无法编译。这是对 C++98 的改进，C++98 这些不当使用直到链接时才会被诊断出来。

​	通常 delete 的函数被声明为共有属性。这是因为当客户端代码尝试使用成员函数时，C++ 会先价差删除状态。当客户端代码尝试使用已 delete 的私有成员函数时，编译器会报错函数为私有成员，即使该函数的可访问性并不会真正影响她是否可以使用（也就是错误转移了）。

​	delete 函数的一个重要优点是可以 delete 任何函数，例如，假设我们有一个非成员函数，它接受一个整数并返回它是否是幸运数字：

```c++
bool isLucky(int number);
```

​	C++ 继承了 C 中类型隐形转换特性，但是有些可编译的类型转换调用毫无意义：

```c++
if (isLucky('a')) ...		// is 'a' a lucky number?
if (isLucky(true)) ...		// is 'true'?
if (isLucky(3.5)) ...		// should we truncate to 3 before checking for luckiness?
```

​	如果幸运数字确实必须是整数，我们希望组织编译此类调用。实现这一点的方法是为我们要过滤掉的类型创建 delete 函数重载：

```c++
bool isLucky(int number);		// original function
bool isLucky(char) = delete;	// reject chars
bool isLucky(bool) = delete;	// reject bools
bool isLucky(double) = delete;	// reject doubles and floats

// caller
if (isLucky('a')) ...			// error! call to deleted function
if (isLucky(true)) ...			// error!
if (isLucky(3.5f)) ...			// error!
```

​	（关于 double 重载的评论说双精度和浮点数都被拒绝，你可能会迟疑，但仔细想一下，如果将浮点数转换为 int 或 double 之间选择，C++ 跟倾向于 double。因此，使用浮点数调用 isLucky 将调用 double 重载，而不是 int 重载）

​	delete 函数可以执行的另外一个特性是（相比函数私有化方式不能）防止使用应该仅用的模板实例化。例如，假设需要一个使用内置指针的模板：

```c++
template<typename T>
void processPointer(T* ptr);
```

​	之阵中有两个特殊情况。一个是 void*，因为没办法解引用、自增或自减等。另一个是 char* 指针，因为他们通常表示指向 C 样式字符串，而不是只想单个字符的指针。这些特殊情况通常需要特殊处理，在 processPointer 模板的情况下，我们假设正确的处理是拒绝使用这些类型的调用。那么直接 delete 这些类型的实例防止调用：

```C++
template<>
void processPointer<void>(void*) = delete;

template<>
void processPointer<char>(char*) = delete;

template<>
void processPointer<const void>(const void*) = delete;

teplate<>
void processPointer<const char>(const char*) = delete;

// delete other so on 禁用其他类型也是一样
...
```

​	对比私有化的方法，如果类内有个模板函数将其声明为私有成员，这并不能更改其访问级别（因为不能提供成员函数模板特化与主模板不同的访问级别）。例如，如果 processPointer 是 Wideget 内的成员函数模板，并且您想禁用对 void* 指针的调用，使用 C++98 的方法（编译不过）：

```c++
class Widget{
  public:
    ...
	template<typename T>
	void processPointer(T* ptr){...}
  private:
    template<>
    void processPointer<void>(void*);		// error!
};
```

​	这里的问题是，模板的特化必须写在命名空间作用域内，而不是类的作用域内。这个问题对于 delete 方式是不存在的，因为它们不受限于访问权限。他们可以在类外声明：

```c++
class Widget{
  public:
    ...
	template<typename T>
	void processPointer(T* ptr) {...}
	...
};

template<>
void Widget::processPointer<void>(void*) = delete;
```

**小结**

* 优先使用 delete 而不是设置 private 属性不定义
* delete 可以设置任何函数，包括非成员函数和模板实例

-----

#### ***Item 12. 使用 override 关键字声明覆盖的函数***

​	C++ 面向对象编程围绕类、继承和虚函数展开。这里需要明白区分几个概念"覆盖"（虚函数），"重写"，"重载"。

​	"覆盖"：基于虚函数，派生类重新定义积累虚函数（函数名，参数，返回值相同、加 virtual 修饰）

​	"重写"：派生类与基类存在同名函数（非虚函数）

​	"重载"：同名参数不同，这里不赘述了

虚函数的存在实现了多态---基类接口调用派生类函数：

```c++
class Base{
  public:
    virtual void doWork();			// base class virtual function
    ...
};

class Derived: public Base{
  public:
    virtual void doWork();			// override Base::doWork ("virtual" is optional here)
    ...
};

std::unique_ptr<Base> upb =			// create base class pointer to derived class object
    std::make_unique<Derived>();	// see item21 for info on std::make_unique
...
upb->doWork();						// call doWork through base class ptr; derived
									// class function is invoked
```

发生"覆盖"满足的条件：

* 基类函数必须是虚函数
* 基类函数名和派生类函数名必须相同（析构函数除外）
* 基函数和派生类函数的参数必须相同
* 基函数和派生类的常量必须相同（常函数修饰符）
* 基函数和派生类函数的返回类型和异常规范必须兼容

这些都是C++98的规范，C++11又新增了一个：

* 函数的引用限定符必须相同。它可以将成员函数的使用限制为仅为左值或仅为右值。成员函数不需要是虚拟的才使用：

```c++
class Widget{
  public:
    ...
	void doWork()&;		// this version of doWork applies only when *this is an lvalue
    void doWork()&&;	// this version of doWork applies only when *this is an rvalue
};
...
Widget makeWidget();	// factory function (return rvalue)
Widget w;				// normal object (an lvalue)
...
w.doWork();				// calls Widget::doWork for lvalues (i.e.,Widget::doWork&)
makeWidget().doWorkd();	// calss Widget::doWork for rvalues (i.e.,Widget::doWork&&)
```

​	稍后详细介绍带引用限定符的成函数，这里主要说明，如果基类的虚函数带引用限定符，则该派生类必须也带引用限定符。如果没有，声明的函数只存在于派生类，不会覆盖基类。

```c++
class Base{
  public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3()&;
    void mf4() const;
};
class Derived: public Base{
  public:
    virtual void mf1();					// 基类const限定，但是派生类没有
    virtual void mf2(unsigned int x);	// 基类int参数，派生类unsinged int
    virtual void mf3()&&;				// 基类左值限定，派生类右值限定
    void mf4() const;					// 基类未声明 virtual
};
```

​	这些失误却能正常编译但是功能事与愿违，C++11 提供了一种方法来显式派生类函数应该覆盖积累版本：声明使用你 override：

```c++
// 编译不过
class Derived: public Base{
  public:
    virtual void mf1() override;
    virtual void mf2(unsigned int x) override;
    virtual void mf3()&& override;
    virtual void mf4() const override;
};
```

​	C++11 在之前已有的关键字基础上添加了两个上下文关键字，override 和 final。这些关键具有保留的特征，但是仅限于某些上下文。在"覆盖"的情况下，只有当它出现在成员函数声明的末尾时，它才具有意义。如果有已经使用名称 override 名称的代码可以保留，也没问题：

```c++
class Waring{				// potential legacy class from c++98
  public:
    ...
	void override();		// legal in both c++98 and c++11 (with the same meaning)
	...
};
```

​	这就是"覆盖"的全部内容，但不是关于成员函数引用限定符的全部内容（稍后再讲）。如果我们想编写一个只接受左值参数的函数，我们声明一个非常量左值引用参数：

```c++
void doSomething(Widget& w);			// accepts only lvalue Widgets
```

​	如果我们想编写一个只接受右值参数的函数，我们声明一个右值引用参数：

```c++
void doSomethin(Widget&& w);
```

​	成员函数引用限定符的作用类似于 const 修饰符，对于引用限定成员函数的需求并不常见，但是也有。例如，假设我们的 Widget 类有一个 std::vector 数据成员，并且我们提供一个访问它的函数：

```c++
class Widget{
  public:
    using DataType = std::vector<double>;	// see Item9 for info on "using"
    ...
	DataType& data(){return values;}
	...
  private:
    DataType values;
};

Widget w;
...
auto vals1 = w.data();				// copy w.values into vals1

Widget makeWidget();
auto vals2 = makeWidget().data();	// copyt values inside the Widget into vals2

// 以上都是会存在调用拷贝构造的情况，如何避免，看如下
class Widget{
  public:
    using DataType = std::vector<double>;
    ...
	DataType& data() &		// for lvalue Widgets, return lvalue
    {return values;}
    
    DataType data() &&
    {return std::move(values);}	// for rvalue Widgets, return rvalue
	...
  private:
    DataType values;
};

...
auto vals1 = w.data();				// calls lvalue overload for Widget::data, 
									// copy-constructs vals1

auto vals2 = makeWidget().data();	// calls rvalue overload for Widget::data,
									// constructs vals2
```

**小结**

* 声明"覆盖函数"使用 override
* 成员函数引用限定符可以区分对待左值和右值对象（*this）

----

#### ***Item 13. 使用 const_iterator 而不是 iterator***

​	const_iterator 是指针常量的 STL 等价物（只想不可修改的值）。在无需修改迭代器指向内容时尽可能使用 const_iterator，对于 C++98 和 C++11 都是如此。但在 C++98 中，对 const_iterator 的支持并不是太好。创建也不容易，而且使用它的方式也受到限制。例如，假设我们要在 std::vector<int> 中搜索第一次出现的1983，然后在这个地方插入1998。如果 vector 中没有1983，则在末尾插入。C++98 中使用迭代器很容易实现 ：

```c++
std::vector<int> values;
...
std::vector<int>::iterator it = std::find(values.begin(), values.end(), 1983);
values.insert(it, 1998);
```

​	这里的 iterator 并非最佳选择，因为这段代码永远不会更改迭代器指向的内容。正常来说，改为 const_iterator 应该很简单，但是在 C++98 中，却不是：

```c++
typedef std::vector<int>::iterator IterT;
typedef std::vector<int>const_iterator ConstIterT;	// typedef
std::vector<int> values;
...
ConstIterT ci =
    std::find(static_cast<ConstIterT>(values.begin()), 		// cast
              static_cast<ConstIterT>(values.end()),		// cast
              1983);
values.insert(static_cast<IterT>(ci), 1998);		// may not compile see below
```

​	这个例子演示的是 C++98 所以使用了 typedef 。std::find 中强制类型转换是因为 values 是在 C++98 中非 const 容器中，也没有好办法从一个非 const 容器中得到一个 const_iterator。强制类型转换并非必要的，因为可以从其他方法得到 const_iterator （比如，可以绑定 values 到一个 const 的引用变量，使用这个变量替代代码中的 values），但是不管使用哪种方式，从一个非 const 容器中得到一个 const_iterator 牵扯太多东西。

​	上面代码编译不过，这个是因为不存在 const_iterator 到 iterator 的转换，甚至是使用 static_cast 也不行。甚至最暴力的 reinterpret_cast 也不行。（这并不是 C++98 的限制，C++11 也只如此）还有一些方法可以生成类似 const_iterator 行为的 iterator，但是他们都不是很明显，也不通用，这里就不介绍了。这里想说的是 C++98 中使用 const_iterator 很费劲，非必要可以不使用。

​	但是在 C++11 中，const_iterator 既容易获取也容易使用。容器中成员函数 cbegin 和 cend 可以产生 const_iterator，甚至非 const 的容器可以这样做，STL 成员函数通常使用 const_iterator 来进行定位（也就是说，插入和删除 insert 和 erase）。优化 C++98 代码：

```C++
std::vector<int> values;
...
auto it = std::find(values.cbegin(), values.cend(), 1983);
values.insert(it, 1998);
```

​	C++11 中只有一种使用 const_iterator 的短处就是在编写最大化泛型库的代码时候。代码需要考虑率一些容器或者类似于容器的数据结构提供 bgein 和 end（外加cbegin，cend，rbegin等等）作为非成员函数。例如，对于内置数组就是这种情况，和一些第三方库中提供一些的函数接口。最大化泛型代码使用非成员函数而不是成员函数的版本。

​	例如，我们可以将我们一直是在使用的代码应用到 find 和 insert 模板：

```c++
template<typename C, typename V>
void findAndInsert(C& container,			// in container, find first occurence
                   const V& targetVal,		// of targetVal, then insertVal
                   const V& insertVal);		// there
{
    using std::cbgein;
    using std::cend;
    
    auto it = std::find(cbegin(container),	// non-member cbegin
                       cend(container),		// non-member cend
                       targetVal);
    
    container.insert(it, insertVal);
}
```

​	这在 C++14 中工作正常，但在 C++11 中不行。由于 C++ 增加了非成员函数 begin 和 end，但是没有增加 cbegin、cend、rbegin、rend、crbegin 和 crend。如果使用 C++11 编写通用代码并且使用的库都没有提供非成员的 cbegin 和 友元模板，你可以很容易自己来实现。如下，cbegin 的实现：

```c++
template<class C>
auto = cbegin(const C& container)-> decltype(std::begin(container))
{
    return std::begin(container);		// see explanation below
}
```

​	这个 cbegin 模板接受任何类型的表示类似容器的数组结构 C 的参数，并且它通过常量引用参数访问这个容器。如果 C 是传统容器类型（例如 std::vector<int>），则容器将是对该容器的 const 版本的引用（const std::vector<int>&）。在 const 容器上调用非成员 begin 函数（C++11提供）会产生一个 const_iterator，而这个迭代器就是这个模板返回的。如果 C 是内置数组类型，则此模板也适用。在这种情况下，容器成为对 const 数组的引用。C++11 为数组提供一个特殊版本的非成员 begin，它返回一个指向数组第一个元素的指针。const 数组的元素是 const，所以 const 数组的份成员 begin 返回的指针是 const 指针，const 指针实际上是数组的 const_iterator。（要深入了解模板如何专门用于内置数组，请参阅item1关于将引用参数用于数组的模板中的类型推导）。这里重点是鼓励尽可能使用 const_iterator，基本原则是只要有意义就使用 const -- C++98 中使用常量迭代器不适用，在 C++11 中非常实用，C++14 有对 C++11 进行了查漏补缺。

**小结**

* 尽量使用 const_iterator
* 编写通用代码时倾向使用非成员函数 begin、end、rbgein等等。

----

#### ***Item 14. 如果函数无异常则生命 noexcept***

​	**C++98中的异常规范**：throw 关键字除了可以用在函数体中抛出异常，还可以用在函数头和函数体之间，指明当前函数能够抛出的异常类型，这成为异常规范，有些教程也称为异常指示符或一场列表。如下例子：

```c++11
double func1(char param) throw(int);
```

​	函数 func1 只能抛出 int 类型的异常。如果抛出其他类型的异常，try 将无法捕获，并直接调用 std::unexpected。如果函数会抛出多种类型的异常，那么可以用逗号隔开：

```c++
double func2(char param) throow(int, char, exception);
```

​	如果函数不会抛出任何异常，那么只需要写一个空括号即可：

```c++
double func3(char param) throw();
```

​	同样的，如果函数 func3 还是会抛出异常，try 也会检测不到，并且也会直接调用 std::unexpected。

​	C++ 规定，派生类虚函数的异常规范必须与基类虚函数的规范一样严格，或者更严格。只有这样，当通过基类指针（或者引用）调用派生类虚函数时，才能保证不违背及类成员函数的异常规范。看如下：

```c++
class Base{
  public:
    virtual int fun1(int) throw();
    virtual int fun2(int) throw(int);
    virtual int fun3() throw(int, string);
};
class Derived: public Base{
  public:
    int fun1(int) throw(int);		// 错！一场规范不如throw()严格
    int fun2(int) throw(int);		// 对！有相同的一场规范
    string func3 throw(string);		// 对！异常规范比throw(int,string)更严格
};
```

​	**异常规范与函数定义和函数声明**：C++ 规定，异常规范在函数声明和定义中必须同时指明，并且要严格保持一致，不能更加严格或者更加宽松，如：

```c++
// 错！定义中有异常规范，声明中没有
void func1();
void func1() throw(int){}

// 错！定义和声明中的异常规范不一致
void func2() throw(int);
void func2() throw(int, bool){}

// 对！定义和声明中的异常规范严格一致
void func3() throw(float, char*);
void func3() throw(float, char*){}
```

​	**异常规范在C++11中被摒弃**：异常规范的初衷是好的，它希望让我们看到函数的定义或者声明后，立马知道该函数会抛出什么类型的异常，好让我们可以使用 try-catch 来捕获。如果没有异常规范，必须阅读函数源码才能知道函数会抛出什么异常。不过这有时候也不容易做到。例如，func_outer() 函数可能不会引发异常，但是它调用了另外一个函数 func_inner()，这个函数可能会引发异常。再如，编写一个函数调用了老式的一个函数库，此时不会引发异常，但是老式库更新后却引发了异常。

​	1、一场规范的检查是在运行期而不是编译期，因此程序员不能保证所有异常都能得到 catch 处理

​	2、由于第一点的存在，编译器需要生成额外的代码，在一定程度上妨碍优化

​	3、模板函数中无法使用

​		template<class T>

​		void func(T k)

​		{

​			T x(k);

​			x.do_something();

​		}

​			赋值函数、拷贝构造函数和do_something() 都有可能抛出异常，这取决于类型T的实现，所以无法给函数 func 指定异常类型。

​	4、实际中，我们只需要两种异常类型说明：抛异常和不抛异常，也就是throw(...)和throw()。

​	所以 C++11 摒弃了throw异常规范，而引入了新的异常说明符 noexcept。

**C++11 noexcept**:

​	noexcept 紧跟在函数的参数列表后面，它只用来表示两种状态："不抛异常"和"抛异常"。

```c++
void func_not_throw() noexcept;			// 保证不抛出异常
void func_not_throw() noexcept(true);	// 和上式一个意思
void func_throw() noexcept(false);		// 可能会抛出异常
void func_throw();						// 和上式一个意思，如不显示说明，默认是会抛出
										// 异常（除了析构函数，详见下面）
```

对一个函数而言：

​	1、noexcept 说明符要么出现在该函数的所有声明语句和定义语句，要么一次也不出现

​	2、函数指针及该指针指的函数必须具有一致的异常说明

​	3、在typedef 或 类型别名中则不能出现 noexcept

​	4、在成员函数中，noexcept 说明符需要跟在 const 及引用限定符之后，而在 final、override 或 虚函数的 = 0 之前

​	5、如果一个虚函数承诺了它不会抛出异常，则后续派生的虚函数也必须做出同样的承诺；相反，如果基类的虚函数允许抛出异常，则派生类的虚函数既可以抛出异常，也可以不允许抛出异常

需要注意的是，**编译器不会检查带有 noexcept 说明符的函数是否有 throw**。

```c++
#include <iostream>
using namespace std;
void func_not_throw() noexcept
{
    throw 1;
}
int main()
{
    try{
        func_not_throw();	//直接terminate,不会被catch
    } catch (int){
        cout << "catch int" << endl;
    }
    
    return 0;
}
```

所以程序员在 noexcept 的使用上要格外小心。

noexcept 除了可以用作用说明符（Specifier），也可以用作运算符（operator）。noexcept 运算符是一个一元运算符，它的返回值是一个 bool 类型的右值常量表达式，用于表示给定的表达式是否会抛出异常。如：

```c++
void f() noexcept{}
void g() noexcept(noexcept(f)) { // g() 是否是 noexcept 取决于 f()}
    f();
}
```

其中 noexcept(f) 返回 true，则上式就相当于 void g() noexcept(true)。

析构函数默认都是 noexcept 的。C++11 标准规定，类的析构函数都是 noexcept 的，除非显示指定为 noexcept(false)。

```c++
class A{
  public:
    A(){}
    ~A(){}	// 默认不抛出异常
};

class B{
  public:
    B(){}
    ~B() noexcept(false){}   // 可能会抛出异常
};
```

在为某个异常进行栈展开的时候，会依次调用当前作用域下每个局部对象的析构函数，如果这个时候析构函数有抛出自己的未经处理的另一个异常，将会导致 std::terminate。所以析构函数应该从不抛出异常。

**显式指定异常说明符的益处**：

1、语义

​	从语义上，noexcept 对于程序员之间的交流有利，就像 const 限定符一样

2、显示指定 noexcept 的函数，编译器会进行优化

​	因为在调用 noexcept 函数时不需要记录 exception handler，所以编译器可以生成更高效的二机制码（编译器是否优化不一定，但理论上 noexcept 给了编译器更多的优化机会）。另外编译器在编译一个 noexcept(false) 的函数时可能会生成很多冗余的代码，这些代码虽然只在出错的时候执行，但还是会对 Instruction Cache 造成影响，进而影响程序整体的性能。

**容器操作针对 std::move 的优化**：举个例子，一个 std::vector<T>，若要进行 reserve 操作，一个可能的情况是，需要重新分配内存，并把之前原有的数据拷贝（copy）过去，但是如果 T 的移动构造函数是 noexcept 的，则可以移动（move）过去，大大提高效率。

```c++
#include <iostream>
#include <vector>
using namespace std;
class A{
  public:
    A(int value){}
    A(const A& other)
    {
        std::cout << "copy constructor";
    }
    A(A &&other) noexcept
    {
        std::cout << "move constructor";
    }
};

int main()
{
    std::vector<A> a;
    a.emplace_back(1);
    a.emplace_back(2);
    
    return 0;
}
```

上述代码可能输出：

```c++
move constructor
```

但是如果把移动构造函数的 noexcept 说明符去掉，则会输出：

```c++
copy constructor
```

你可能会问，为什么在移动构造函数是 noexcept 时才能使用？这是因为如果它执行的是 Strong Exception Guarantee，发生异常时需要还原，也就是说，你调用它之前是什么样，抛出异常后，你就得恢复成啥样。但是对于移动构造函数发生异常，是很难恢复回去的，如果在恢复移动（move）的时候发生异常了呐？但复制构造函数就不同了，它发生异常直接调用它的析构函数就行了。

***使用建议***：

我们所编写的函数默认都不适用，只有遇到以下的情况你在思考是否需要使用，

1、析构函数

​	这个不多说必须应该是noexcept

2、构造函数（普通、复制、移动），赋值运算符重载函数

​	尽量让上面的函数都是 noexcept，这可能会给你的代码带来一定的运行执行效率

3、还有那些你可以 100% 保证不会 throw 的函数

​	比如像是 int，pointer 这类的 getter，setter 都可以用 noexcept。因为不可能出错。但请一定要注意，不能保证的地方请不要用，否则会害人害己！切记！如果你还是不知道该在哪里用，可以看标准 Boost 的源码，全局搜索 BOOST_NOEXCEPT，你就大概明白了。

-----

#### ***Item 15. 尽可能使用 constexpr***

​	constexpr 是一个很让人疑惑的关键字。当应用于对象时，他本质上是 const 的增强形式，但是应用于函数时，它具有完全不同的含义。从概念上讲，constexpr 表示的值不仅是常量，而且在编译期间是已知的。但是 constexpr 修饰函数就不是这样了。

​	先从 constexpr 修饰对象开始，这些对象实际上是 const，并且他们确实具有在编译时已知的值。编译器期间已知的值有一定特殊用处。例如它们可能被放置在只读存储器中，尤其对于嵌入式系统的开发人员来说，这可能是一个相当重要的特性。另外它可以用于常量表达式上下文（包括 std::array 对象的长度，枚举值，对齐说明符等）。如果你想使用常量可以使用 constexpr：

```c++
int sz;							// non-constexpr variable
...
constexpr auto arraySize1 = sz;	// error! sz's value not known at compilation
std::array<int, int> data1;		// error! same problem
constexpr auto arraySize2 = 10;	// fine, 10 is a compile-time  constant
std::array<int, arraySize2> data2;	// fine, arraySize2 is constexpr
```

请注意，const 与 constexpr 略有不同，因为不需要使用编译期间已知的值初始化 const 对象：

```c++
int sz;							// as before
...
const auto arraySize = sz;		// fine, arraySize is const copy of sz
std::array<int, arraySize> data;// error, arraySize's value not known at compilation
```

​	简单地说，所有的 constexpr 对象都是 const，但是不是所有 const 对象都是 constexpr。如果您希望编译器保证变量具有可在需要需要编译时常量的上下文中使用的值，则需要使用的是 constexpr，而不是 const。

​	接下来观察 constexpr 修饰函数的情况：

* constexpr 函数应用在编译时常量的上下文。如果传递给 constexpr 函数的参数已知，则在编译期间会计算结果。如果在编译期间不知道入参是啥，则编译不过
* 如果使用一个或多个编译器间未知的值调用 constexpr 函数时，它和普通函数一样，在运行时计算结果。

​	假设我们需要一个数据结构来保存可以以多种方式运行的实验结果。例如，在实验过程中，照明级别可以是高、低或关闭，风扇速度和温度等也是如此。如果存在与试验相关的环境条件，每个环境条件都有三种可能的状态，组合为 3^n 。因此，存储所有条件组合的实验结果需要一个具有容纳 3 个值的数据结构。

​	假设每个结果都是一个 int 并且在编译期间 n 是已知的（或可以计算），那么可以使用 std::array 存储。但是我们需要一种在编译期间计算 3^n 的方法。C++ 标准库提供了 std::pow ，这个功能满足，但是存在两个问题。首先，std::pow 适用于浮点类型，我们需要一个整数结果。其次，std::pow 不是 constexpr（即，在使用编译时值调用时不能保证返回编译结果），因此我们没法使用它来指定 std::array 大小。

​	因此我们需要自己实现这个功能：

```c++
constexpr
int pow(int base, int exp) noexcept		// pow's a constexpr func that never throws
{
    ...									// iml is below
}

// 入参 constexpr,编译时执行
constexpr auto numConds = 5;
std::array<int, pow(3, numConds)> results;	// results has 3^numConds elements

// 入参非 constexpr 变量，运行时也可以调用
auto base = readFromDB("base");			// get these values at runtime
auto exp = readFromDB("exponent");
auto baseToExp = pow(base, exp);		// call pow function at runtime
```

​	由于 constexpr 函数在编译期间调用时必须能有返回结果，因此对其实现会有一些限制。C++11 和 C++14 的限制有所不同。在 C++11 中，constexpr 函数只能包含一个执行语句：一个 return。在 constexpr 中有两个使用技巧，一个是 "?:" 条件表达式，一个是递归。看如下 pow 的实现：

```c++
constexpr int pow(int base, int exp) noexcept
{
    return (exp==0 ? 1 : base*pow(base, exp-1));
}		// 增加函数调用栈
```

​	在 C++14 中这个限制宽松一些：

```c++
constexpr int pow(int base, int exp) noexcept
{
    auto result = 1;
    for (int i=0; i<exp; ++i) result *= result;
	return result;
}	// 多条语句
```

​	constexpr 函数仅限于获取和返回 "字面量"(constexpr)，这就意味着需要在编译期间确定值的类型。在 C++11 中，除了 void 之外的所有内置类型都符合，但是用户自定义的类型可能也可能是"字面量"，因为构造函数和其他成员函数可能是 consetexpr：

```c++
class Point{
  public:
    constexpr Point(double xVal = 0, double yValue = 0) noexcept
	: x(xVal),y(xVal){}
    
    constexpr double xValue() const noexcept {return x;}
    constexpr double yValue() const noexcept {return y;}
    
    void setX(double newX) noexcept {x = newX;}
    void setY(double newY) noexcept {y = newY;}
  private:
    double x,y;
};
```

​	这个例子里，将 Point 构造函数声明为 constexpr，因为如果在编译期间传递给它的参数是已知的，则在编译期间也可以知道构造的 Point 的数据成员的值。所以初始化的 Point 是 constexpr：

```c++
constexpr Point p1(9.4, 27.7);		// fine, "runs" constexpr ctor during compilation
const Point p2(28.8, 5.3);			// also fine
```

​	类似的，xValue 和 yValue 也可以是 constexpr，因此如果在编译期间已知值的 Point 对象（如上）调用它们，则可以在编译期间知道数据成员 x,y 的值。所以也可以使用 Point 的这两个成员函数初始化 constexpr 对象：

```c++
constexpr
Point midpoint(const Point& p1, const Point& p2) noexcept
{
    return {(p1.xValue() + p2.xValue())/2 ,
           (p2.yValue() + p2.yValue())/2 };		// call constexpr member funcs
}

constexpr auto mid = midpoint(p1, p2);		// init constexpr object w/result of 													// constexpr function
```

​	***这里有一个启发一点：编译期间完成的工作和运行时完成的工作之间传统上相当严格的界限开始变得模糊，一些传统上在运行时完成的计算可以迁移到编译时。参与迁移的代码越多，您的软件运行速度就越快。 （不过，编译可能需要更长的时间。）***

​	在 C++11 中，setX 和 setY 不能声明为 constexpr ，原因有两个：首先，它们修改修改操作对象，在 C++11 中，constexpr 成员函数默认是 const。其次，他们有 void 返回类型。但是两个限制在 C++14 中被取消：

```c++
class Point{
  public:
    ...
	constexpr void setX(double newX) noexcept	// c++14
    { x = newX; }
    constexpr void setY(double newY) noexcept	// c++14
    { y = newY; }
    ...
};

// return reflection of p with respect to the origin(C++14)
constexpr Point reflection(const Point& p) noexcept
{
    Point result;				// create non-const Point
    result.setX(-p.xValue());	// set its x and y values
    result.setY(-p.yValue());
    
    return result;				// return copy of it
}

// Client code 
constexpr Point p1(9.4, 27.7);			// as above
constexpr Point p2(28.8, 5.3);
constexpr auto mid = midpoint(p1, p2);
constexpr auto reflectedMid = reflection(mid);		// reflectedMid's value 
													// is (-19.1, 16.5) and known during
													// complication
```

**小结**

* constexpr 对象是常量，并使用编译期间已知的值进行初始化
* constexpr 函数可以在使用编译期间已知值的参数调用时产生编译时结果
* constexpr 对象和函数可以在比 constexpr 对象和函数更广泛的上下文中使用
* constexpr 是对象或函数接口的一部分

----

#### ***Item 16. 使 const 成员函数线程安全***

​	在解决有关数学方面相关的问题时，如声明一个表示多项式的类。在这个类中，有一个函数来计算多项式的根，即多项式计算为零的值。这个函数不会修改多项式，因此将其声明为 const ：

```c++
class polynomial{
  public:
    using RootsType = 			// data structure holding values where polynomial
        std::vector<double>;	// evals to zero (see Item9 for info on "using")
    ...
	RootsType roots() const;
    ...
};
```





**小结**



-----

#### Item 17. 了解特殊成员函数生成

**小结**



