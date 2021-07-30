# chapter 1. 类型转换

​	C++98拥有一套类型推导规则:函数模板的规则。C++11稍微修改了该规则并添加了另外两个,一个是auto，一个是decltype。C++14扩展了auto和decltype可以使用上下文。类型推导的广泛应用可以让给你从显式或赘余的类型声明中解脱出来。这样可以让C++软件更具适应性，因为在源码中某一处改变类型，通过类型推导会自动同步到其他地方。不过使用类型，会导致代码难以理解，因为编译器进行的类型推导并非你所想象的那样。

​	如果对类型推导的运作方式没有深入的了解，对如今的C++进行有效的编程是很难的。发生类型推导的上下文太多了：调用模板函数、大多数出现auto的情况、在decltype表达式中、以及从C++14开始使用神秘的decltype(auto)构造(函数??)。

​	本章提供了每个C++开发人员都需要的关于类型推导的信息。解释模板类型推导的工作原理、auto构建的方式以及decltype如何实现推导。甚至解释如何强制编译器使类型推导结果可预见，从而使编译器推导出是你所需的类型。

#### *item1 模板类型推导*

​	一个复杂系统的用户不知道它是如何运作的，但是依然对它的作用感到满意，这诠释了一些涉及理念。C++模板类型推导通过这种设计理念获得了巨大成功。数以百万的程序员即使很难给出更多关于如何推导出这些函数使用的类型的推导，通过给模板函数传参也能获得满意的结果。(不懂但是会用)

​	如果你是其中一员的话，我这里有一个好消息一个坏消息。好消息是模板的类型推导是现代C++最引人注目的基础功能之一，特性：auto.如果你感觉C++98推到模板类型的方式还不错，那么你对C++如何推导auto类型也很满意(推导规则一样)。坏消息是当盲板类型推导应用到auto的上下文时，有时看起来不如将他们应用于模板直观。因此，真正理解由auto创建的模板类型推导尤为重要。本项包含了你需要了解的内容。

​	我们忽略一部分伪代码看看下面的例子，我们可以将函数模板视为如下：

   ```c++
   template<typename T>
   void f(ParamType param);
   ```

​	调用：

​	`f(expr)` //通过表达式调用   

​	在编译期间编译器会使用 expr 推导出两种类型：一个是T，一个是 ParamType。这两个的类型经常会不同，因为 ParamType 通常会包含修饰，如:const或者引用限定符。像下面的模板声明：

```c++
template<typename T>
void f(const T& param);  //ParamType is const T&
int x=0;
f(x);	
```

​	T被推导为int，但是 ParamType 被推导为 const int &。

​	很容易理解T和函数参数类型被推导为相同类型。即T是 expr 的类型。上面的例子就是这个情况：x是int，然后T被推导为int。但不总是这样。推导出T的不仅取决于 expr 的类型，还取决于 ParamType 的形式。有以下三个情况：

* ParamType 是一个指针或者引用类型，但不是一个通用引用。(通用引用将会在 item24 中介绍。这里你只需要 知道的是它们存在并且它们不同于左值引用和右值引用)

* ParamType 是一个通用指针

* ParamType 既不是一个指针也不是一个引用

  因此，我们有三种类型的推导场景需要检查。每个例子将基于下面的模板和调用：

  ```c++
  template<typename T>
  void f(PramType param);
  f(expr);  //deduce T and ParamType from expr
  ```

  ---

  ***Case1:ParamType是引用类型或者指针类型***

  ​	最简单的情况是当ParamType是引用类型或者指针类型但又不是通用引用时，这种情况下，类型推导过程如下：

  ​	1、如果expr的类型是引用，则忽略引用部分。

  ​	2、然后将expr的类型与ParamType进行模式匹配以确定T

  例如，以下模板：

  ```c++
  template<typename T>
  void f(T& param);  // param is a reference	
  ```
  
  然后我们声明如下变量:
  
  ```c++
  int x = 27;         // x is an int
  const int cx = c;   // cx is a const int
  const int& rx = x;  // rx is a reference to x as a const int
  ```
  
  看下面的调用以及param和T的推导：
  
  ```c++
  f(x);	//T is int, param's type is int&
  f(cx);	//T is const int, param's type is const int&
  f(rx);	//T is const int, param's type is const int&
  ```
  
  ​	在第二和第三个调用中，我注意到因为使用了 const 修饰了值，T被推导为 const int，因此参数的类型被推导为 const int&。当将一个 const 对象传递给一个引用参数时，这是为了保持该对象不可变，这对接口调用者很重要，即参数是常量引用。这就是将 const 对象传递给采用T& 参数的模板是安全的原因：对象的 const 成为T推导类型的一部分。
  
  ​	第三个例子中，发现即使rx的类型是一个引用，T也被推导为非引用。这是因为在类型推导过程中忽略了rx的引用。
  
  ​	这些例子中都展示了左值引用，但是跟右值引用的推导完全相同。当然，只有右值参数可以传递给右值引用参数，但是这个限制跟类型推导无关。
  
  ​	如果我们将f的参数类型从T& 更改为 const T&，不会有大的变化。cx 和rx 的常量属性还是以旧，但是因为我们现在设定 param 是对常量的引用，所以不再需要将 const 推导为T的一部分：
  
  ```c++
  template<typename T>
  void f(const T& param);	// param is now a ref-to-const
  
  int x = 27;				//as before
  const int cx = x;	  //as before
  const int &rx = x;    //as before
  
  f(x); 		// T is int, param's tyoe is const int
  f(cx);		// T is int, param's type is const int &
  f(rx);		// T is int, param's type is const int &
  ```

  ​	跟以前一样，在类型推导过程中会忽略 rx 的引用。

  ​	如果 param 是一个指针（或者一直指向 cons t的指针）而不是引用，推到方式是一样的：

  ```c++
  template<typename T>
  void f(T* pararm);		//param is now a pointer
  int x = 27;
  
  const int *px = &x;
  f(&x);		// T is int, param's type is int*
  f(px);		// T is const int, param's type is const int *
  ```

  ----

  ***Case2:ParamType是通用引用***

  ​    对于采用通用引用参数的模板，事情就没那么明显。这类参数的声明类似于右值引用（即，在采用类型参数T的函数模板中，通用引用的声明类型为T&&），但是当传参左值时表现不通。在item24中会详细介绍，这里先简单提一下：

  ​	- 如果 expr 是左值，则T和 ParamType 都被推导为左值引用

  ​	这就很特殊。首先，这是模板中唯一一种T被推导为引用的情况。其次，即使ParamType被声明为右值引用，ParamType 还是会被推导为左值引用。

  ​	-如果 expr 是右值，则"正常"（即case1）应用推到规则

  例如：

  ```c++
  template<typename T>
  void f(T&& param);		// param is now a unvarsal reference
  int x = 27;				// as before
  const int cx = x;		// as before
  const int& rx = x;		// as before
  
  f(x);	 // x is lvalue, so T is int&  .     param's type is also int &
  f(cx);	 // cx is lvalue, so T is const int&  .  param's type is alse const int &
  f(rx);	 // rx is lvalue, so T is const int & .  param's type is alse const int &
  f(27);	 // 27 is rvalue, so T is int . param's type is therefore int&&	
  ```
  
  ​	item24 会详细阐述为什么这些示例会以这种方式运行。这里重点是想说明通用引用参数的类型推导规则不同于左值引用或右值引用。特别是当使用通用引用时，左值参数和右值参数的区别。非通用引用类型不会有这种推导规则。
  
  ---
  
  ##### ***Case3:ParamType 既不是一个指针也不是一个引用***
  
  ​	当ParamType既不是引用也不是指针时，推导将会通过值传递方式处理：
  
  ```c++
  template<typename T>
  void f(T param);					// param is now passed by value
  ```

  ​	这就意味之 param 将是入参的一个副本---一个全新的对象。新对象 param 侧面反映了 expr 是如何推导出T的：

   	1. 如果expr的类型是引用，则忽略引用部分
   	2. 如果在忽略expr的饮用后，expr是const，也会忽略const。如果是volatile，同样会被忽略。（volatile 对象并不常见，他们通常仅用于实现设备驱动程序。item40 将会详细介绍）
  
  下面来看示例：
  
  ```c++
  int x = 7;         // as before
  const int cx = x;  // as before
  const int &rx = x; //as before
  
  f(x);     // T's and param's type are both int
  f(cx);    // T's and param's type are again both int 
  f(rx);    // T's and param's type are still both int 
  ```
  
  ​	注意即使cx和rx是const值，但是param不是const。这也说明param是一个独立于cx和rx的对象----cx和rx的副本。cx和rx不能修改的属性并不会传递给param。推导param类型时会忽略expr的常量属性是因为：expr不能修改并不意味着它的副本不能被修改。
  
  ​	认识到值传递时const（和valatile）属性会被忽略很重要。正如我们所见，对于引用或指针形参入参const属性的实参时param会携带const属性。接下来考虑一下另一种情况，expr是一个常量指针指向常量对象，并且expr被传递给一个值传递的形参：
  
  ```c++
  template<typename T>
  void f(T param);                   // param is still passed by value
  const char* const str = "good";    //ptr is const pointer to const object
  f(ptr);							   // pass arg of type const char* const
  ```

  ​	星号右边的const声明ptr是常量：ptr的指向不可变，也不能设置为NULL。（星号左边的const表示ptr指向的--字符串--是常量不可修改）。当ptr传递给函数 f 时，构成指针的比特位被复制到param中。因此，指针本身（ptr）将按值传递。根据值传递的类型推到规则，ptr 的const 属性被忽略，然后param的类型被推导为const char* ，即指向const 字符串的可修改指针值的指针。ptr 所指向的常量类型属性在推导过程中会保留，但是会忽略指针本身的const属性。

  ----

  #####   ***数组参数***

  ​	上面讲述的内容几乎涵盖主流模板推导，但是还有部分少数案例需要了解。虽然数组和指针有时似乎可以选关乎转换，但是数组和指针类型不同。造成这种错觉的一个主要原因是在许多情况下，数组会退化为指向其第一个元素的指针，如下可通过编译的代码：

  ```c++
  const char name[] = "good boy"; // name's type is const char[9]
  const char* ptrToName = name;	// array decays to pointer
  ```
  
  ​	这里const char* 类型指针ptrToName被初始化为const char[13] 类型的name。这两种类型不同，但是由于数组退化规则，代码可以编译通过。但是如果我们见数组传递一个值传递的模板会怎样呐？又会发生什么？如下代码：
  
  ```c++
  template <typename T>
  void f(T param);	// template with by-value parameter
  f(name);			// what types are deduced for T and param?
  ```

  ​	我们先来观察一下数组参数的函数：

  ```c++
  void myFunc(int param[]);
  ```

  ​	但是数组声明被视为指针声明，这就意味着myFunc等效于如下声明：

  ```c++
  void myFunc(int * param);
  ```

  ​	数组参数和指针参数的这种等价性可能会导致我们 认为数组和指针类型相同。因为数组参数生命是被视为指针类型声明，所以按值传递给模板的数组类型被推导为指针类型。这就意味着在对模板f的调用中，参数类型T被推导为const char*。

  ​	虽然函数不能声明真正的数组参数，但是可以声明数组引用的参数。我们可以修改模板函数f 通过引用获取其参数：

  ```c++
  template <typename T>
  f(T& param);			// template with by-reference parameter
  f(name);					// pass array to f
  ```

  ​	这时推导出T为数组的实际类型并且包括数组的大小，因此上例中T被推导为const char[13]，而f 的参数（对数组的引用）类型为const char &[13]。有趣的是，声明对数组的引用的能力可以创建一个模板来推导数组包含的元素数量：

  ```c++
  // return size of an array as a compile-time constant.
  // (the array parameter has no name, because we care 
  // only about the number of elements it contains)
  template <typename T, std::size_t N>
  constexpr std::size_t arraySize(T (&)[N]) noexcept
  {
  	return N;
  }
  ```
  
  ​	在 item15 中会讲到, 函数声明带 constexpr 时期结果可用在编译过程中。这使得可以声明一个和第二个数组具有相同数量的数组，第二个数组的大小是花括号初始化算来的：
  
  ```c++
  int keyValues[] = {1, 3, 7, 9, 11, 22, 35};	// keyValues has 7 elements
  int mappendVals[araySize(keyVals)];			// so does mappedVals
  ```

  ​	当然与内置数组相比我们更喜欢 std::array :

  ```
  std::array <int, arraySize(keyVals)> mappedVals;		// mappedVal's size is 7
  ```

  ​		关于 arraySize 被声明为 noexcept ，是为了帮助编译器生成更好的代码。有关像信息请参看item14。

  ---

  #####   ***函数参数***

  ​	数组并不是C++中唯一会退化为指针的类型。函数类型会退化为函数指针，关于数组类型推导的所有内容都适用于函数的类型推导。看如下例子：

  ```c++
  void someFunc(int, double);	// someFunc is a function; type is void(int, double)
  template<typename T>
  void f1(T param);		// in f1, param passed by value
  
  template<typename T>
  void f2(T& param);		// in f2, param passed by ref
  
  f1(someFunc);		// param deduced as ptr-to-func; type is void(*)(int, double)
  f2(someFunc);		// param deduced as ref-to-func; type is void(&)(int, double)
  ```

  ​	小结：本节主要讲解模板类型推导的规则，如果想了解编译器推导出啥类型，请看 item4。

  ​	**注意事项**：

  	*	在模板类型推导期间，作为引用的参数被视为非引用，即它们的引用被忽略
  	*	在推到通用引用参数时，佐治参数得特殊处理
  	*	在推导值传递的参数时，const ，volatile 参数被视为非 const 和 非 volatile
  	*	在模板类型推导期间，数组或函数名称的参数会退化为指针，除非他们用于初始化引用
  
  ---
  
  ### *item2 ：了解 auto 类型推导*
  
  ​	如果已经阅读了关于模板类型推导的 item1 ，那么你已经了解了关于 auto 类型推导所需几乎所有内容。除了一个奇怪的例外，自动类型推导是模板类型推导。但是这怎么可能(你可能会怀疑)？模板类型推导涉及模板和函数和参数，但是 auto 不涉及这些东西呀！
  
  ​	确实，但是这不重要。模板类型推导和 auto 类型推导之间存在直接映射。这是一种字面上的一对一算法转换。
  
  ​	在 item1 中，使用通用函数模板来解释模板类型推导：
  
  ```c++
  template <typename T>
  void f(ParamType param);
  f(expr);
  ```
  
  ​	在这个例子中，编译器使用 expr 去推导 ParamType 和 T 的类型。
  
  ​	当使用 auto 声明变量时，**auto 相当于模板中的 T，并且变量的类型说明符为 ParamType**。话不多说来看例子：
  
  ```c++
  auto x = 27;
  const auto cx = x;
  const auto& rx = x;
  ```
  
  ​	这些事例中为了推导x、cx、rx的类型，编译器的行为就好像每个声明都有一个模板，并使用相应的初始化表达式调用该模板：
  
  ```c++
  template <typename T>
  void func_for_x(T param);    // conceptual template for deducing x's type
  func_for_x(27);              // conceptual call: param's deduced type is x's type
  
  template <typename T>
  void func_for_cx(const T param); // conceptual template for deducing cx's type
  func_for_cx(x);					 // conceptual call: param's deduced type is cx's type
  
  template <typename T>
  vpid func_for_rx(const T& param);// conceptual template for deducing rx's type
  func_for_rx(x);					 // conceptual call: param's deduced type is rx's type
  ```
  
  ​	auto 的类型推导与模板推导类型一样，只有一个例外(后续我们会讲到)。item1 基于ParamType 的类型调用函数模板给 param 入参，将模板类型推导分为三种情况。在使用 auto 的变量声明中，类型说明符替换 ParamType，因此也有三种情况：
  
  * Case 1: 类型说明符是指针或引用，但不是通用引用
  * Case 2: 类型说明符是一个通用引用
  * Case 3: 类型说明符既不是指针也不是引用
  
  Case1和Case3 我们上边如上已举过例子：
  
  ```cc
  auto x = 27;		// case 3 (x is neither ptr nor reference)
  const auto cx = x;  // case 3 (cx isn't either)
  const auto& rx = x; // case 1 (rx is a non-univalsal ref)
  ```
  
  Case2 ：
  
  ```c++
  auto&& uref1 = x;	// x is int and lvalue,so uref1's type is int&
  auto&& uref2 = cx;	// cx is const int and lvalue,so uref2's type is const int&
  auto&& uref3 = 27;	// 27 is int and rvalue,so uref3's type is int&&
  ```
  
  在 item1 中讲到的数组和函数名称在非引用类型调用时会退化为指针的情况，同样也会在 auto 推导中发生：
  
  ```c++
  const char name[] = "good boy"; // name's type is const char[13]
  auto arr1 = name;		// arr1's type is const char*
  auto& arr2 = name;		// arr2's type is const char(&)[13]
  
  void someFunc(int, double);		// someFunc is a function; type is void(int, double)
  auto func1 = someFunc;	// func1's type is void(*)(int, double)
  auto& func2 = someFunc;	// func2's type is void(&)(int, double)
  ```
  
  如你所见，auto 类型推导规则类似于模板类型推导。
  
  如果我们想声明一个初始值为 27 的 int，C++98 提供两种语法选择：
  
  ```c++
  int x1 = 27;
  int x2(27);
  ```
  
  C++11，还增加了下面语法支持：
  
  ```c++
  int x3 = {27};
  int x4{27};
  ```
  
  ​	总共四种语法，但是都是一个目的：初始化一个值为 27 的 int。但是正如 item5 所解释的，使用 auto 代替固定类型声明变量是有好处的，优化上面代码：
  
  ```c++
  auto x1 = 27;	// type is int, value is 27
  auto x2(27);	// ditto
  auto x3 = {27};	// type is std::initializer_list<int>, value is {27}
  auto x4{27};	// ditto
  ```
  
  ​	这些声明都可以编译通过，但是经过替换 auto 后他们的含义有所不同。前两个语句确实声明了一个值为27的 int 类型变量。然而，后面两条语句声明了一个类型为 std::initializer_list<int> 的变量，其中包含一个值为 27 的单个元素！这是由于 auto 的特殊类型推到规则。当 auto 声明变量的初始值的初始值使用大括号括起来时，推导出来的类型是 std::initializer_list。如果不能推导出这样的类型（例如，因为花括号初始化器中的值是不同类型），代码将会编译不通过：
  
  ```c++
  auto x5 = {1, 2, 3.0};	// error! can't deduce T for std::initializer_list<T>
  ```
  
  ​	上例中因为 x5 使用 auto 初始化所以是使用 std::initializer_list<T> 来推导类型，但是大括号内的类型有两种无法推导出统一的类型，最终导致推到失败。***auto 的这种花括号初始化方式是唯一不同与模板推导的地方***。但是用 auto 声明变量使用了花括号时推导类型是 std::initializer_list<T> 的实例。但是如果给对应模板传入相同的初始化器（花括号数据），推到将失败，代码会报错：
  
  ```c++
  auto x = {11, 23, 9};  //x's type is std::initializer_list<int>
  
  template <typename T>
  void f(T param);	   // template with parameter declaration equivalent to x's 						   // declaration
  f({11, 23, 9});		   // error! can't deduce type for T
  ```
  
  ​	但是如果在模板中指定 param 是某个未知 T 的 std::initializer_list<T>，则模板类型推导将推导出 T 是什么：
  
  ```c++
  template <typename T>
  void f(std::initializer_list<T> initList);
  f({11,23,9});  // T deduced as in, and initList's type is std::initializer_list<int>
  ```
  
  ​	以上也是 C++11 编程中的一个陷阱，当声明其他类型时意外声明了 std::initializer_list 变量。C++14 允许 auto 指示应该推导出函数的返回类型（见 item3），并且 C++14 lambdas 可以在参数中使用 auto。但是 auto 的这些用途使用模板类型推导，而不是 auto 类型推导。因此，返回类型为 auto，且返回花括号初始化器的函数将无法编译：
  
  ```c++
  auto createInitiList()
  {
      return {1,2,3,4}; // error: can't deduce type for {1,2,3}
  }
  ```
  
  ​	在 C++14 lambda 中使用的参数类型规范中使用 auto 时也是如此：
  
  ```c++
  std::vector<int> v;
  ...
  auto resetV = 
      [&v](const auto& newValue){v = newValue;}; // c++14
  ...
  resetV({1,2,3});	// error! can't deduce type for {1,2,3}
  ```
  
  ***小结：***
  
  * auto 类型推导通常跟模板类型推到相同，但是自动类型赋值花括号初始化时表示 std::initializer_list，但是模板推导不通。
  * 函数返回类型或 lambda 参数中的 auto 意味着模板类型推导，而不是 auto 类型推导。
  
  -----
  
  ### *item3：了解 decltype*
  
  ​	给定名称或者表达式，decltype 会告诉你名称或表达式的类型。但是有时它提供的结果有些让人捉摸不透，我们将从典型的案例开始--那些正常情况，与模板和 auto 类型推导的过程相反，decltype 通常会回溯传给它的名称或表达式的确切类型：
  
  ```c++
  const int i = 0;	// decltype(i) is const int
  bool f(const Widget& w);	// decltype(w) is const Widget&
  							// decltype(f) is bool(const Widget&)
  struct Point{
    	int x,y; 				// decltype(Point::x) is int
  };							// decltype(Point::y) is int
  
  Widget w;					// decltype(w) is Widget
  if (f(w)) ...				// decltype(f(w)) is bool
  
  template <typename T>		// simplified version of std::vector
  class vector{
      public:
      ...
  	T& operator[](std::size_t index);
  	...
  };
  vector<int> v;				// decltype(v) is vector<int>
  ...
  if (V[0] == 0) ...			// devltype(v[0]) is int&
  
  ```
  
  ​	在 C++11 中，decltype 的主要用途可能是声明函数模板，其中函数的返回类型取决于其参数类型。例如，假设我们想编写一个函数，它接受一个支持通过 [] 进行索引容器的成员，然后再返回索引的结果之前对用户进行身份验证。函数的返回类型应与索引操作返回的类型相同，类型为 T 的对象容器上 operator[] 通常返回T&。例如，std::deque 就是这种情况，而 std::vector 几乎总是如此。但是，对于 std::vector<bool>，operator[] 不返回 bool &，相反，它返回一个全新的对象。在 item6 中讨论这种情况的原因和如何产生的，这里主要说明容器的 operator[] 返回类型取决于容器。
  
  ​	decltype很容易实现这一点，这是我们编写模板的第一部分，展示使用 decltype 来计算返回类型。模板需要一些改进，但是我们暂时推迟：
  
  ```c++
  template <typename Container, typename Index>	// works, but requires refinement
  auto authAndAccess(Container& c, Index i)
  	-> decltype(c[i])
  {
      authenticateUser();
      return c[i];
  }
  ```
  
  ​	在函数名前使用 auto 与类型推导无关。相反，它表明正在使用 C++11 的尾随返回类型语法，即函数的返回类型将在参数列表之后（在"->"之后）声明。***尾随返回类型的优点是函数的参数可以在返回类型的规范中使用。例如，在 authAndAccess 中，我们指定使用 c 和 i 返回类型。如果我们以传统方式将返回类型放在函数名之前，c 和 i 将不可用，因为他们还没有被声明***。有了这样的声明，authAndAccess 返回类型即 operator[] 作用于传入容器时返回的任何类型，正是我们所期望的类型。
  
  ​	C++11 允许推到单个语句 lambda 表达式的返回类型，C++14 将此扩展到所有 lambda 和所有函数，包括具有多个语句的函数。在 authAndAccess 的例子中，这就意味着 C++14 中我们可以忽略尾随返回类型，只留下前导 auto。使用这种声明形式，auto 确实意味着将进行类型推导。特别是，这意味着编译器将从函数的实现中推导出函数的返回类型：
  
  ```c++
  template <typename Container, typename Index> //C++14 not quite correct
  auto authAndAccess(Container& c, Index i)
  {
      authenticateUser();
      return c[i];			// return type deduced from c[i]
  }
  ```
  
  ​	item2  解释了 auto 返回类型规范相关的函数，编译器使用模板类型推导。这个情况下就会出现问题，正如我们所讨论的，对于大多数 T 的容器 operator[] 返回 T&，但是 item1 阐述了在模板类型推导期间，初始化表达式的引用会被忽略。考虑这个对客户端代码意味着什么：
  
  ```c++
  std::deque<int> d;
  ...
  authAndAccess(d, 5) = 10; // authendticate user, return d[5], then assign 10 to 							  // it,thiswon't complie!
  ```
  
  ​	这里，d[5] 返回一个 int&，但是 authAndAccess 的 auto 返回类型推导将会剥离引用，从而产生一个 int 返回类型。作为函数返回值的 int 是一个右值，因此上面的代码尝试将 10 赋值给一个右值 int，这在 C++11 中是被禁止的，所以编译不同。
  
  ​	为了让 authAndAccess 像我们想要的那样工作，我们需要使用 decltype 类型推导，即指定 authAndAccess 应该返回与表达式 c[i] 返回完全相同的类型。C++14 中允许使用 decltype(auto) 来推导 auto 的类型。最初看起来矛盾的 decltype 和 auto，实际上很有意义：auto 指定需要推导的类型，而 decltype 表示应该在推导过程中使用 decltype 规则。因此，我们可以这样编写 authAndAccess:
  
  ```c++
  template <typename Container, typename Index> // c++14; works, 
  											  // but still requires refinement
  decltype(auto)
  authAndAccess(Container& c, Index i)
  {
      authenticateUser();
      return c[i];
  }
  ```
  
  ​	现在 authAndAccess 将真正返回 c[i] 返回的任何内容。特别的，对于 c[i] 返回的任何内容。特别是，对 c[i]  返回 T& 的常见情况，authAndAccess 也将返回 T&，而在 c[i] 返回对象的罕见情况下，authAndAccess 也将返回一个对象。
  
  ​	decltype(auto) 的使用不仅限于函数返回类型。它也可以使用在变量声明，decltype 类型推导规则将应用于初始化表达式：
  
  ```c++
  Widget w;
  const Widget& cw = w;
  auto myWidget1 = cw;			// auto type deduction
  								// myWidget1's type is Widget
  decltype(auto) myWidget2 = cw;	// decltype type deduction:
  								// myWidget2's type is const Widget&
  ```
  
  ​	接下来讲述 authAndAccess 的改进，再看一下 C++14 版本的 authAndAccess 声明：
  
  ```c++
  template<typename Container, typename Index>
  decltype(auto) authAndAccess(Container& c, Index i);
  ```
  
  ​	容器的入参是非常量引用，因为返回容器元素的引用允许客户端修改该容器。这就意味着无法将右值容器传递给此函数。右值不能绑定到左值引用（除非是常量引用）。
  
  ​	将右值容器传递给 authAndAccess 是一个边缘情况。作为临时对象的右值容器通常会在调用 authAndAccess 的语句结束时被销毁，这就意味着对该容器中元素的引用（即 authAndAccess 返回的元素）将终止在创建它的语句结尾。尽管如此，将临时对象传递给 authAndAccess 还是有意义的，就比如客户端可能只想赋值临时容器中的元素：
  
  ```c++
  std::deque<std::string> makeStringDeque();	// factory function
  // make copy of 5th element of deque returned
  // from makeStringDeque
  auto s = authAndAccess(makeStringDeque(), 5);
  ```
  
  ​	支持以上这些功能意味着我们需要修改 authAndAccess 的声明以接受左值和右值。声明一个左值引用参数，另一个声明右值引用参数可以完美解决，但是我们就得维护两个函数。为了避免这种情况我们可以让 authAndAccess 绑定一个左值和右值的引用参数，就是 item24 中讲解的通用引用。因此可以这样声明authAndAccess:
  
  ```c++
  template <typename Container, typename Index> 			// c is now a 
  decltype(auto) authAndAccess(Container&& c, Index i);   // universal reference
  ```
  
  ​	在这个模板中，我们不知道操作的容器是什么类型，也不知道使用到的 Index 是啥类型。对未知类型的对象使用值传递通常会造成不必要的拷贝，还有对象的切片问题（参考 item41），但是我们现在使用的标准模板是可以通过索引来获取元素的（如，operator[] 作用于 std::string，std::vector，std::deque）。所以在这里我们使用按值传递。(*就是说这里的代码有缺陷，只是为了演示，点明注意一下！！！*)
  
  ​	下面我们会用到 item25 中讲到的 std::forward 实现 C++14 的终极版本模板:
  
  ```c++
  template<typename Container, typename Index>	// final C++14 version
  decltype(auto) authAndAccess(Container&& c, Index i)
  {
      authenticateUser();
      return std::forward<Container>(c)[i];
  }
  ```
  
  ​	以上的模板声明应该满足我们所有的要求了，但是必须使用 C++14 编译器。如果非 C++14 编译器版本则需要使用 C++11 版本模板，功能相同只是 C++11 版本需要指定返回类型：
  
  ```c++
  template <typename Container, typename Index>	// final C++11 version
  auto authAndAccess(Container&& c, Index i)
  -> decltype(std::forward<Container>(c)[i])
  {
      authenticateUser();
      return std::forward<Container>(c)[i];
  }
  ```
  
  ​	要完全理解 decltype 的行为，还必须熟悉部分特殊情况。其中大部分的内容都太晦涩难懂，无法在书中去描述讨论，为此我们观察其中一个来深入了解 decltype 及其用法。
  
  ​	将 decltype 应用于名称会产生该名称的声明类型。名称是左值表达式，但是不会影响 decltype 的行为。左值表达式比名称要复杂，decltype 推导的类型始终是左值引用。也就是，如果除名称之外的左值表达式具有类型 T，则 decltype 将该类型推导为 T&。这很少有影响，因为大多数左值表达式的类型固有的包含左值引用限定符。例如，返回左值的函数总是返回左值引用。
  
  ​	这里有个值得关注的点，如下：
  
  ```c++
  int x = 0;
  ```
  
  ​	x 是变量的名称，所以 decltype(x) 是 int。但是将名称 x 阔在括号里--"(x)"--会产生一个比名称更复杂的表达式。x 和 (x) 都是左值，因此 decltype((x)) 推导出的结果是 int&。***将变量名称使用()包裹起来会更改 decltype 推导的类型!!!***。这是 C++11 其中一个新奇的地方，再结合 C++14 对 decltype(auto) 的支持，在函数的 return 语句中如果发生如上微不足道的更改将会影响函数返回类型的推导：
  
  ```c++
  decltype(auto) f1()
  {
  	int x = 0;
      ...
  	return x;		// decltype(x) is int, so f1 return int
  }
  
  decltype(auto) f2()
  {
      int x = 0;
      ...
  	return (x);		// decltype((x)) is int&, so f2 return int&
  }
  ```
  
  ​	这不仅会更改函数返回的类型，还会导致访问一个局部变量的引用！这将会导致出现未知的行为，需要多加留意。
  
  **小结：**
  
  * decltype 几乎总是能正确推导除变量或表达式的类型
  * 对于除变量名意外的 T 类型左值表达式，decltype 推导类型为 T&
  * C++14 支持 decltype(auto) 同时使用，它同 auto 一样，从初始值中推导出类型，但是它使用 decltype的推导规则
  
  ----
  
  ### *item4：知道如何查看被推导类型*
  
  ​	查看类型推导结果的工具选择取决于你需要的信息处于软件开发过程的那个阶段。我们将在三个阶段阐述：编码阶段，编译期间以及运行阶段。
  
  ##### **IDE 编辑器**
  
  ​	当你的光标悬停在实体上时，IDE中的代码编辑器通常会显示程序实体的类型（例如，变量、参数、函数等）。例如，给定下面代码：
  
  ```c++
  const int theAnswer = 42;
  auto x = theAnswer;
  auto y = &theAnswer;
  ```
  
  IDE编辑器将会展示 x 的推导类型是 int，y 的类型是 const int*。这些功能需要编译器支持，通常内置类型IDE很容易识别，但是涉及复杂的类型，IDE显示的信息可能不是特别有用。
  
  ##### **编译器诊断**
  
  ​	让编译器显示它推导出的类型的一种有效办法是以导致编译出错的方式使用该类型。报告的错误信息几乎肯定提到导致问题的类型。例如，如果我们想查看前面例子 x 和 y 推导的类型，我首先声明一我们没有定义的类模板：
  
  ```c++
  template <typename T>	// declaration only for TD; TD == "Type Displayer"
  class TD;
  ```
  
  ​	尝试实例化此模板将会引发错误信息，因为没有要实例化的模板定义。要查看 x 和 y 的类型，只需要尝试使用他们的类型实例化 TD:
  
  ```c++
  TD<decltype(x)> xType;	// elicit error containing
  TD<decltype(y)> yType;	// x's and y's types
  ```
  
  使用变量名形式的变量是为了方便查看错误日志，对于上面的代码，编译器发出的诊断信息部分如下：
  
  ```c++
  error: aggregate 'TD<int> xType' has incomplete type and cannot be defined
  error: aggregate 'TD<const int *> yType' has incomplete type and cannot be defined
  ```
  
  不同编译器提供相同的信息，但是形式不同，如下：
  
  ```c++
  error: 'xType' uses undefined class 'TD<int>'
  error: 'yType' uses undefined class 'TD<const int *>'
  ```
  
  撇开格式的差异不同，所有的编译器都会提示带类型信息的错误消息。
  
  ##### 运行输出
  
  ​	显示类型信息的 printf 方法（不是我建议使用printf）直到运行时才能使用，但是它提供了对格式的完全控制。重点在于创建显示自己关心的类型的文本表示。你可能会想 typeid 和std::type_info::name 简直就是救星！在我们继续探索为 x 和 y 推到的类型时，你可能认为我们可以这样写：
  
  ```c++
  std::cout << typeid(x).name() << '\n';	// display types for
  std::cout << typeif(y).name() << '\n';	// x and y
  ```
  
  ​	这个方法的实现原理是这样的，在诸如 x 和 y 之类的对象上调用 typeid 会产生一个 std::type_info 对象，而 std::type_info 有一个成员函数 name，它会产生一个 C 风格的字符串（即，一个const char*）类型名称。
  
  ​	对 std::type_info::name 的调用不会保证返回任何合理的信息，但是会尽力提供帮助。人性化的程度各不相同。例如，GUN 和 Clang 编译器会提示 x 的类型为 "i"，y 的类型为 "PKi"。一旦了解了这些编译器的输出信息，"i" 表示 "int"，"PK" 表示"指向 kconst const 的指针"，这信息就会变得有意义。（两个编译器都支持c++filt，它可以对这种"错位"类型进行解码。）微软的编译器输出的信息会好一些：x 是 "int"，y 是 "const int *"。
  
  ​	从结果看 x 和 y 的类型是正确的，所以你能会认为类型推导问题已经解决，但是这太草率了，让我们来看一个更复杂的例子：
  
  ```c++
  template <typename T>				// template function to be called
  void f(const T& param);
  
  std::vector<Widget> createVec();	// factory function
  const auto vw = createVec();		// init vw w/factory return
  
  if (!vw.empty())
  {
      f(&vw[0]);						// call f
      ...
  }
  ```
  
  ​	这段代码包含一个用户定义的类型 Widget 、一个STL容器 std::vector 和一个自动变量 vw ，更方便我们观察编译器的类型推导。如：模板类型参数 T 和 f 中函数参数 param 推导出了哪些类型。
  
  ​	在这个问题上使用 typeid 会更简单点。只需要在 f 中添加一些代码即可显示想要观察的类型：
  
  ```c++
  template <typename T>
  void f(const T& param)
  {
      using std::cout;
      cout << "T = " << typeid(T).name() << '\n';			// show T
      cout << "param = " << typeid(param).name() << '\n';	// show param's type
      ...
  }
  ```
  
  由 GUN 和 Clang 编译器生成的可执行程序输出如下：
  
  ```c++
  T = PK6Widget
  param = PK6Widget
  ```
  
  我们已经知道，对于这些编译器，PK 的意思是"pointer to const "，所以唯一的疑惑是这个 6 。这只是类名（Widget）包含的字符数。所以这些编译器告诉我们 T 和 param 都是 const Widget* 类型。
  
  微软编译器也是如此：
  
  ```c++
  T = class Widget const *
  param = class Widget const *
  ```
  
  ​	三个对立的编译器产生相同的信息，表明信息是准确的。但是仔细看，在模板 f 中，param 的声明类型是 const T&。既然如此，T 和 param 的类型相同是不是很奇怪？例如，如果 T 是 int，则 param 的类型应该是const int & --- 根本不是同一个类型。侧面反映 std::type_info::name 的结果并不可靠。在这种情况下，举个例子，所有的三种编译器报告的 param  的类型都是不正确的。更深入的话，它们本来就是不正确的，因为 std::type_info::name  的特化指定了类型会被当做它们被传给模板函数的时候的按值传递的参数。正如 item1所述，这就意味着如果类型是一个引用，他的引用特性会被忽略，如果在忽略引用之后存在 const（或者 volatile  ），它的 const  特性（或者 volatile  特性）会被忽略。这就是为什么 param  的类型—— const Widget * const &  ——被报告成了 const Widget*  。首先类型的引用特性被去掉了，然后结果参数指针的 const  特性也被消除了。
  
  ​	同样的，由IDE编辑器显示的类型信息也并不准确——或者说至少并不可信。对之前的相
  同的例子，一个我知道的IDE的编辑器报告出 T  的类型：
  
  ```c++
  const
  std::_Simple_types<std::_Wrap_alloc<std::_Vec_base_types<Widget,
  std::allocator<Widget> >::_Alloc>::value_type>::value_type *
  ```
  
  还是这个相同的IDE编辑器， param  的类型是：
  
  ```c++
  const std::_Simple_types<...>::value_type *const &
  ```
  
  ​	这个没有 T  的类型那么吓人，但是中间的“...”会让你感到困惑，直到你发现这是IDE编辑器的一种说辞“我们省略所有 T  类型的部分”。带上一点运气，你的开发环境也许会对这样的代码有着更好的表现。
  ​	如果你更加倾向于库而不是运气，你就应该知道 std::type_info::name  可能在IDE中会显示类型失败，但是Boost TypeIndex库（经常写做Boost.TypeIndex）是被设计成可以成功显示的。这个库并不是C++标准的一部分，也不是IDE和模板的一部分。更深层的是，事实上Boost库（在boost.com）是一个跨平台的，开源的，并且基于一个偏执的团队都比较喜欢的协议。这就意味着基于标准库之上使用Boost库的代码接近于一个跨平台的体验。
  ​	这里展示了一段我们使用Boost.TypeIndex的函数 f  精准的输出类型信息：
  
  ```c++
  #include <boost/type_index.hpp>
  template<typename T>
  void f(const T& param)
  {
  using std::cout;
  using boost::typeindex::type_id_with_cvr;
  // show T
  cout << "T = "
  << type_id_with_cvr<T>().pretty_name()
  << '\n';
  // show param's type
  cout << "param = "
  << type_id_with_cvr<decltype(param)>().pretty_name()
  << '\n';
  …
  }
  ```
  
  ​	这个模板函数 boost::typeindex::type_id_with_cvr  接受一个类型参数（我们想知道的类型信
  息）来正常工作，它不会去除 const  ， volatile  或者引用特性（这也就是模板中的“ cvr  ”的
  意思）。返回的结果是个 boost::typeindex::type_index  对象，其中的 pretty_name  成员函数
  产出一个 std::string  包含一个对人比较友好的类型展示的字符串。
  ​	通过这个 f  的实现，再次考虑之前使用 typeid  导致推导出现错误的 param  类型信息：
  
  ```c++
  std::vector<Widget> createVec(); // 工厂方法
  const auto vw = createVec(); // init vw w/factory return
  if (!vw.empty()) {
  f(&vw[0]); // 调用f
  …
  }
  ```
  
  在GNU和Clang的编译器下面，Boost.TypeIndex输出（准确）的结果：
  
  ```c++
  T = Widget const*
  param = Widget const* const&
  ```
  
  微软的编译器实际上输出的结果是一样的：
  
  ```c++
  T = class Widget const *
  param = class Widget const * const &
  ```
  
  这种接近相同的结果很漂亮，但是需要注意IDE编辑器，编译器错误信息，和类似于Boost.TypeIndex的库仅仅是一个对你编译类型推导的一种工具而已。所有的都是有帮助意义的，但是到目前为止，没有什么关于类型推导法则1-3的替代品。
  
  **小结：**
  
  * 类型推导的结果常常可以通过IDE的编辑器，编译器错误输出信息和Boost TypeIndex库的结果中得到
  * 一些工具的结果不一定有帮助性也不一定准确，所以对C++标准的类型推导法则加以理解是很有必要的
  
  
  
  
  
  
  
  
  
  
  
  
  
  ​	
  
  ​	
  
  ​	
  
  ​	
  
  





​	

​	























