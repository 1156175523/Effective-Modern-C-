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
  
  ​	当使用 auto 声明变量时，auto 相当于模板中的 T，并且变量的类型说明符为 ParamType。话不多说来看例子：
  
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
  
  ### *item2 ：了解 decltype*
  
  ​	给定名称或者表达式，decltype 会告诉你名称或表达式的类型。但是有时它提供的结果有些让人捉摸不透，我们将从典型的案例开始--那些正常情况，与模板和 auto 类型推导的过程相反，decltype 通常会会回溯传给它的名称或表达式的确切类型：
  
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
  
  ​	在函数名前使用 auto 与类型推导无关。相反，它表明正在使用 C++11 的未遂返回类型语法，即函数的返回类型将在参数列表之后（在"->"之后）声明。尾随返回类型的优点是函数的参数可以在返回类型的规范中使用。例如，在 authAndAccess 中，我们指定使用 c 和 i 返回类型。如果我们以传统方式将返回类型放在函数名之前，c 和 i 将不可用，因为他们还没有被声明。有了这样的声明，authAndAccess 返回类型即 operator[] 作用于传入容器时返回的任何类型，正是我们所期望的类型。
  
  ​	C++11 允许推到单个语句 lambda 表达式的返回类型，C++14 将此扩展到所有 lambda 和所有函数，包括具有多个语句的函数。在 authAndAccess 的例子中，这就意味着 C++14 中我们可以忽略尾随返回类型，只留下前导 auto。使用这种声明形式，auto 确实意味着将进行类型推导。特别是，这意味着编译器将从函数的实现中推导出函数的返回类型：
  
  ```c++
  template <typename Container, typename Index> //C++14 not quite correct
  auto authAndAccess(Container& c, Index i)
  {
      authenticateUser();
      return c[i];			// return type deduced from c[i]
  }
  ```
  
  ​	item2 
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  ​	
  
  ​	
  
  ​	
  
  ​	
  
  





​	

​	























