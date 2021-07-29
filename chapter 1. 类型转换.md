# chapter 1. 类型转换

​	C++98拥有一套类型推导规则:函数模板的规则。C++11稍微修改了该规则并添加了另外两个,一个是auto，一个是decltype。C++14扩展了auto和decltype可以使用上下文。类型推导的广泛应用可以让给你从显式或赘余的类型声明中解脱出来。这样可以让C++软件更具适应性，因为在源码中某一处改变类型，通过类型推导会自动同步到其他地方。不过使用类型，会导致代码难以理解，因为编译器进行的类型推导并非你所想象的那样。

​	如果对类型推导的运作方式没有深入的了解，对如今的C++进行有效的编程是很难的。发生类型推导的上下文太多了：调用模板函数、大多数出现auto的情况、在decltype表达式中、以及从C++14开始使用神秘的decltype(auto)构造(函数??)。

​	本章提供了每个C++开发人员都需要的关于类型推导的信息。解释模板类型推导的工作原理、auto构建的方式以及decltype如何实现推导。甚至解释如何强制编译器使类型推导结果可预见，从而使编译器推导出是你所需的类型。

#### *item1 模板类型推导*

​	一个复杂系统的用户不知道它是如何运作的，但是依然对它的作用感到满意，这诠释了一些涉及理念。C++模板类型推导通过这种设计理念获得了巨大成功。数以百万的程序员即使很难给出更多关于如何推导出这些函数使用的类型的推导，通过给模板函数传参也能获得满意的结果。(不懂但是会用)

​	如果你是其中一员的话，我这里有一个好消息一个坏消息。好消息是模板的类型推导是现代C++最引人注目的基础功能之一，特性：auto.如果你感觉C++98推到模板类型的方式还不错，那么你对C++如何推导auto类型也很满意(推导规则一样)。坏消息是当盲板类型推导应用到auto的上下文时，有时看起来不如将他们应用于模板直观。因此，真正理解由auto创建的模板类型推导尤为重要。本项包含了你需要了解的内容。

​	我们忽略一部分伪代码看看下面的例子，我们可以将函数模板视为如下：

'''

​	template<typename T>

​	void f(ParamType param);

'''

​	调用：

​	`f(expr)` //通过表达式调用   

​	在编译期间编译器会使用expr推导出两种类型：一个是T，一个是ParamType。这两个的类型经常会不同，因为ParamType通常会包含修饰，如:const或者引用限定符。像下面的模板声明：

'''

​	template<typename T>

​	void f(const T& param);  //ParamType is const T&

'''

  	然后这样调用：

'''

​	int x=0;

​	f(x);

'''

​	T被推导为int，但是 ParamType被推导为const int &。

​	很容易理解T和函数参数类型被推导为相同类型。即T是expr的类型。上面的例子就是这个情况：x是int，然后T被推导为int。但不总是这样。推导出T的不仅取决于expr的类型，还取决于ParamType的形式。有以下三个情况：

* ParamType是一个指针或者引用类型，但不是一个通用引用。(通用引用将会在item24中介绍。这里你只需要 知道的是它们存在并且它们不同于左值引用和右值引用)

* ParamType是一个通用指针

* ParamType既不是一个指针也不是一个引用

  因此，我们有三种类型的推导场景需要检查。每个例子将基于下面的模板和调用：

  '''

  template<typename T>

  void f(PramType param);

  f(expr);  //deduce T and ParamType from expr

  '''

  ---

  ***Case1:ParamType是引用类型或者指针类型***

  ​	最简单的情况是当ParamType是引用类型或者指针类型但又不是通用引用时，这种情况下，类型推导过程如下：

  ​	1、如果expr的类型是引用，则忽略引用部分。

  ​	2、然后将expr的类型与ParamType进行模式匹配以确定T

  例如，以下模板：

  '''

  ​	template<typename T>

  ​	void f(T& param);  // param is areference

  '''

  然后我们声明如下变量:

  '''

  ​	int x = 27;      //x is an int

  ​	const int cx = c;  //cx is a const int

  ​	const int& rx = x;  //rx is a reference to x as a const int

  '''

  看下面的调用以及param和T的推导：

  '''

  ​	f(x);	//T is int, param's type is int&

  ​	f(cx);	//T is const int, param's type is const int

  ​	f(rx);	//T is const int, param's type is const int&

  '''

  ​	在第二和第三个调用中，我注意到因为使用了const修饰了值，T被推导为const int，因此参数的类型被推导为const int&。当将一个const对象传递给一个引用参数时，这是为了保持该对象不可变，这对接口调用者很重要，即参数是常量引用。这就是将const 对象传递给采用T& 参数的模板是安全的原因：对象的const成为T推导类型的一部分。

  ​	第三个例子中，发现即使rx的类型是一个引用，T也被推导为非引用。这是因为在类型推导过程中忽略了rx的引用。

  ​	这些例子中都展示了左值引用，但是跟右值引用的推导完全相同。当然，只有右值参数可以传递给右值引用参数，但是这个限制跟类型推导无关。

  ​	如果我们将f的参数类型从T& 更改为const T&，不会有大的变化。cx和rx的常量属性还是以旧，但是因为我们现在设定param是对常量的引用，所以不再需要将const 推导为T的一部分：

  '''

  ​	template<typename T>

  ​	void f(const T& param);	// param is now a ref-to-const

  ​	int x = 27;				//as before

  ​	const int cx = x;	  //as before

  ​	const int &rx = x;    //as before

  ​	f(x); 		// T is int, param's tyoe is const int

  ​	f(cx);		// T is int, param's type is const int &

  ​	f(rx);		// T is int, param's type is const int &

  '''

  ​	跟以前一样，在类型推导过程中会忽略rx的引用。

  ​	如果param是一个指针（或者一直指向const的指针）而不是引用，推到方式是一样的：

  '''

  ​	template<typename T>

  ​	void f(T* pararm);		//param is now a pointer

  ​	int x = 27;

  ​	const int *px = &x;

  ​	f(&x);								// T is int, param's type is int*

  ​	f(px);								// T is const int, param's type is const int *

  '''

  ----

  ***Case2:ParamType是通用引用***

  ​    对于采用通用引用参数的模板，事情就没那么明显。这类参数的声明类似于右值引用（即，在采用类型参数T的函数模板中，通用引用的声明类型为T&&），但是当传参左值时表现不通。在item24中会详细介绍，这里先简单提一下：

  ​	- 如果expr是左值，则T和ParamType都被推导为左值引用

  ​	这就很特殊。首先，这是模板中唯一一种T被推导为引用的情况。其次，即使ParamType被声明为右值引用，ParamType还是会被推导为左值引用。

  ​	-如果expr是右值，则"正常"（即case1）应用推到规则

  例如：

  '''

  ​	template<typename T>

  ​	void f(T&& param);					// param is now a unvarsal reference

  ​	int x = 27;									// as before

  ​	const int cx = x;						  // as before

  ​	const int& rx = x;						// as before

  ​	f(x);												// x is lvalue, so T is int&  .     param's type is also int &

  ​	f(cx);											  // cx is lvalue, so T is const int&  .  param's type is alse const int &

  ​	f(rx);											   // rx is lvalue, so T is const int & .  param's type is alse const int &

  ​	f(27);											  // 27 is rvalue, so T is int .  param's type is therefore int&&	

  '''

  ​	item24 会详细阐述为什么这些示例会以这种方式运行。这里重点是想说明通用引用参数的类型推导规则不同于左值引用或右值引用。特别是当使用通用引用时，左值参数和右值参数的区别。非通用引用类型不会有这种推导规则。

  ---

  ##### ***Case3:ParamType 既不是一个指针也不是一个引用***

  ​	当ParamType既不是引用也不是指针时，推导将会通过值传递方式处理：

  '''

  ​	template<typename T>

  ​	void f(T param);					//param is now passed by value

  '''

  ​	这就意味之param将是入参的一个副本---一个全新的对象。新对象param侧面反映了exor是如何推导出T的：

   	1. 如果expr的类型是引用，则忽略引用部分
   	2. 如果在忽略expr的饮用后，expr是const，也会忽略const。如果是volatile，同样会被忽略。（volatile 对象并不常见，他们通常仅用于实现设备驱动程序。item40 将会详细介绍）

  下面来看示例：

  '''

  ​	int x = 7;               // as before

  ​	const int cx = x;   // as before

  ​	const int &rx = x; //as before

  ​	f(x);                // T's and param's type are both int

  ​	f(cx);              // T's and param's type are again both int 

  ​	f(rx);              // T's and param's type are still both int 

  '''

  ​	注意即使cx和rx是const值，但是param不是const。这也说明param是一个独立于cx和rx的对象----cx和rx的副本。cx和rx不能修改的属性并不会传递给param。推导param类型时会忽略expr的常量属性是因为：expr不能修改并不意味着它的副本不能被修改。

  ​	认识到值传递时const（和valatile）属性会被忽略很重要。正如我们所见，对于引用或指针形参入参const属性的实参时param会携带const属性。接下来考虑一下另一种情况，expr是一个常量指针指向常量对象，并且expr被传递给一个值传递的形参：

  '''

  ​	template<typename T>

  ​	void f(T param);         // param is still passed by value

  ​	const char* const str = "good";    //ptr is const pointer to const object

  ​	f(ptr);							// pass arg of type const char* const

  '''

  ​	星号右边的const声明ptr是常量：ptr的指向不可变，也不能设置为NULL。（星号左边的const表示ptr指向的--字符串--是常量不可修改）。当ptr传递给函数 f 时，构成指针的比特位被复制到param中。因此，指针本身（ptr）将按值传递。根据值传递的类型推到规则，ptr 的const 属性被忽略，然后param的类型被推导为const char* ，即指向const 字符串的可修改指针值的指针。ptr 所指向的常量类型属性在推导过程中会保留，但是会忽略指针本身的const属性。

  ----

  #####   ***数组参数***

  ​	上面讲述的内容几乎涵盖主流模板推导，但是还有部分少数案例需要了解。虽然数组和指针有时似乎可以选关乎转换，但是数组和指针类型不同。造成这种错觉的一个主要原因是在许多情况下，数组会退化为指向其第一个元素的指针，如下可通过编译的代码：

  '''

  ​	const char name[] = "good boy";   // name's type is const char[9]

  ​	const char* ptrToName = name;	// array decays to pointer

  '''

  ​	这里const char* 类型指针ptrToName被初始化为const char[13] 类型的name。这两种类型不同，但是由于数组退化规则，代码可以编译通过。但是如果我们见数组传递一个值传递的模板会怎样呐？又会发生什么？如下代码：

  '''

  ​	template <typename T>

  ​	void f(T param);				// template with by-value parameter

  ​	f(name);							 // what types are deduced for T and param?

  '''

  ​	我们先来观察一下数组参数的函数：

  '''

  ​	void myFunc(int param[]);

  '''

  ​	但是数组声明被视为指针声明，这就意味着myFunc等效于如下声明：

  '''

  ​	void myFunc(int * param);

  '''

  ​	数组参数和指针参数的这种等价性可能会导致我们 认为数组和指针类型相同。因为数组参数生命是被视为指针类型声明，所以按值传递给模板的数组类型被推导为指针类型。这就意味着在对模板f的调用中，参数类型T被推导为const char*。

  ​	虽然函数不能声明真正的数组参数，但是可以声明数组引用的参数。我们可以修改模板函数f 通过引用获取其参数：

  '''

  ​	template <typename T>

  ​	f(T& param);			// template with by-reference parameter

  ​	f(name);					// pass array to f
  
  '''
  
  ​	这时推导出T为数组的实际类型并且包括数组的大小，因此上例中T被推导为const char[13]，而f 的参数（对数组的引用）类型为const char &[13]。有趣的是，声明对数组的引用的能力可以创建一个模板来推导数组包含的元素数量：
  
  '''
  
  ​	// return size of an array as a compile-time constant.
  
  ​    // (the array parameter has no name, because we care 
  
  ​	// only about the number of elements it contains)
  
  ​	template <typename T, std::size_t N>
  
  ​	constexpr std::size_t arraySize(T (&)[N]) noexcept
  
  ​	{
  
  ​		return N;
  
  ​	}
  
  '''
  
  ​	在 item15 中会讲到, 函数声明带 constexpr 时期结果可用在编译过程中。这使得可以声明一个和第二个数组具有相同数量的数组，第二个数组的大小是花括号初始化算来的：
  
  ""
  
  ​	int keyValues[] = {1, 3, 7, 9, 11, 22, 35};		// keyValues has 7 elements
  
  ​	int mappendVals[araySize(keyVals)];			// so does mappedVals
  
  '''
  
  ​	当然与内置数组相比我们更喜欢 std::array :
  
  '''
  
  ​	std::array <int, arraySize(keyVals)> mappedVals;		// mappedVal's size is 7
  
  '''
  
  ​	关于 arraySize 被声明为 noexcept ，是为了帮助编译器生成更好的代码。有关像信息请参看item14。
  
  ---
  
  #####   ***函数参数***
  
  ​	数组并不是C++中唯一会退化为指针的类型。函数类型会退化为函数指针，关于数组类型推导的所有内容都适用于函数的类型推导。看如下例子：
  
  '''
  
  ​	void someFunc(int, double);		// someFunc is a function; type is void(int, double)
  
  ​	template<typename T>
  
  ​	void f1(T param);							// in f1, param passed by value
  
  ​	template<typename T>
  
  ​	void f2(T& param);						 // in f2, param passed by ref
  
  ​	f1(someFunc);								 // param deduced as ptr-to-func; type is void(*)(int, double)
  
  ​	f2(someFunc);								 // param deduced as ref-to-func; type is void(&)(int, double)
  
  '''
  
  ​	小结：本节主要讲解模板类型推导的规则，如果想了解编译器推导出啥类型，请看 item4。
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  ​	
  
  ​	
  
  ​	
  
  ​	
  
  





​	

​	























