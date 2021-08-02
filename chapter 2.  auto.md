# chapter 2.  auto

#### *item 5. 首选auto而非显示类型声明*

看如下例子，迭代器解引用初始化初始化局部变量的声明：

```c++
template <typename It>
void dwin(It b, It e)
{
    while(b != e)
    {
		typename std::iterator_traits<it>::value_type;
    	currValue = *b;
    	...
    }
}
```

​	typename std::iterator_traits<It>::value_type 用来表示迭代器指向的值的类型？真的是这样？声明一个局部变量，他的类型是闭包类型，只有编译器知道无法被写出。但是从 C++11 开始，由 auto 的引入而解决。auto 变量的类型是从初始化表达式推导出来的，所以使用 auto 必须初始化。

```c++
int x1;				// potentially uninitialized
auto x2;			// error! inintializer required
auto x3 = 0;		// fine, x's value is well-defined
```

​	由于这个特性，声明局部变量解引用迭代器的问题得以解决：

```c++
template <typename It>
void dwim(It b, It e)
{
    while (b != e)   
    {
        auto currValue = *b;
        ...
    }
}
```

​	因为 auto 使用类型推导（参见 item2）,它可以表示只有编译器知道的类型。

```c++
// comparison func. for Widgets pointed to by std::unque_pts
auto dereFURLess = 
    [](const std::unique_ptr<Widget>& p1, const std::unique_ptr<Widget>& p2)
	{ return *p1 < *p2; };
```

​	C++14 中 lambda 表达式的参数可以包含 auto ，所以使用起来会更加丝滑：

```c++
// C++14 comparison function for values pointed to by anything to 
//   by anything pointer-like
auto derefLess = 
    [](const auto& p1, const auto& p2) {return *p1 < *p2;};
```

​	你或许会想 std::function 也可以声明闭包变量。当然这个可以，但并非我们所想。什么是 std::function ？让我们来了解一下。std::function 是 C++11 标准库中的一个模板，它包含了函数指针的概念。函数指针只能指向函数，而 std::function 对象可以引用所有可以调用的对象，即可以向函数一样调用任何对象。正如在创建函数指针时必须函数类型一样，在创建 std::function 对象时也必须指定要引用函数的类型，我们可以通过函数模板参数是声明函数的类型。例如：要声明一个名为 func 的 std::function 对象，该对象可以引用任何形如此类型可调用的对象，

```c++
// c++11 signature for std::unique_ptr<Widget> comparison function
bool (const std::unique_ptr<Widget>&, const std::unique_ptr<Widget>&)
```

像上面个的函数我们可以这样：

```c++
std::function<bool(const std::unique_ptr<Widget>&,
                  const std::unique_ptr<Widget>&)> func;
```

​	因为 lambda 表达式可以生成可调用对象，所以闭包可以存储在 std::function 对象中。也就是说我们可以不使用 auto 声明如下 derefUPLess：

```c++
std::function<bool(const std::unique_ptr<Widget>&, 
                  const std::unique_ptr<Widget>&) 
    derefUPLess = 
    		[](const std::unique_ptr<Widget>&p1, const std::unique<Widget>& p2)
			{reutn *p1 < *p2}
```

​	抛开语法冗长和重复参数外，使用 std::function 和 auto 还有不同。包含闭包的自动变量和闭包具有相同的类型，并且仅使用闭包所需的内存。持有闭包的 std::function 声明变量的类型是 std::function 模板的实例，并且对于任何给定的类型具有固定大小。但是这个大小可能不足以满足它要求存储的闭包，这种情况下 std::function 构造函数将分配堆内存来存储闭包。这就导致通常 std::function 对象比自动变量对象使用的内存更多，并且由于限制内联和产生间接函数调用的实现细节，通过 std::function 对象调用闭包几乎肯定比通过 auto 声明的对象调用慢。简而言之，std::function 的使用通常要比 auto 消耗更大，运行更慢。这么一比较，auto 不仅是用简洁而且性能更好，所以推荐使用 auto（item34 中我们还会讨论优先使用 lambda 表达式而不是使用 std::bind）。

​	auto 的优势不仅仅在于避免未初始化的变量、冗长的变量声明和直接持有闭包的能力。另外一点是它能够避免我们所说变量内存占用问题（32位，64位变量类型占用内存不同问题）：

```c++
std::vector<int> v;
...
unsigned sz = v.size();
```

​	v.size() 的官方返回类型是 std::vector<int> size_type，但很少有开发者知道 std::vector<int>::size_type 是无符号长整形（unsigned long），所以很多时候写 unsigned 就完事儿了，如上代码。但是这就会导致存在越界隐患（在 32位unsigned long 四字节和64位 unsigned long 八字节）。但是用 auto 可以完美规避这个情况：

```c++
auto sz = v.size();	//sz's type is std::vector<int>::size_type
```

再来观察下面代码：

```c++
std::unordered_map<std::string, int> m;
...
for (const std::pair<std::string, int>& p:m)
{
    ...				// do something with p
}
```

这个例子看上去好像没啥问题，但是有个问题 std::unordered_map 的 key 部分是 const，因此哈希表中的 std::pair 类型不是 std::pair<std::string, int>，而是 std::pair<const std::string, int>。这明显不是上面例子中 p 声明的类型。因此，编译器将努力寻找一种方法将 std::pair<const std::string, int> 对象哈希表中类型转换为 std::pair<std::string, int> 对象。结果就是通过复制 m 中的每个对象，然后将引用 p 绑定到这个临时对象。这个因为类型不匹配导致的复制可以 auto 消除：

```c++
for (const auto p&:m)
{
    ...	
}
```

   **小结**

* auto 变量必须初始化、可以避免移植不同平台导致的隐患、可以提高程序效率、重构时一定程度上减少改动

---

#### *item 6. 当自动推导出不需要的类型时，使用显式类型化的初始化习惯用法*

​	item5 阐述了相比显示声明使用 auto 的一些优势，但是有时使用 auto 也不出现非我们所想的情况。例如，假设我们有一个函数，它接受一个 Widget 并返回一个 std::vector<bool>，其中的每个 bool 表示Widget 是否提供特定功能：

```c++
std::vector<bool> features(const Widget& w);
```

​	进一步假设第五个元素具有高优先级：

```c++
Widget w;
...
bool highPriority = features(w)[5];		// is w high priority?
...
processWidget(w, highPriority);			// process w in accord with its priority?
```

使用 auto 替换：

```c++
auto highPriority = features(w)[5];		// is w high priority?
```

代码编译通过但是代码的结果发生改变：

```c++
processWidget(w, highPriority);			// undefined behavior!
```

出现未定义的问题是因为 highPriority 的类型不再是 bool。虽然 std::vector<bool> 包含的是 bool 元素但 std::vector<bool> 的operatorp[] 不返回对容器元素的引用。相反，它返回的是一个 std::vector<bool>::reference 类型的对象（一个嵌套在 std::vector<bool>中的类）。std::vector<bool> 存在是因为 std::vector<bool> 被指定为以压缩形式表示每一位 bool 值。这就给 std::vector<bool> 的 operator[] d带来了问题，因为 std::vector<T> 应该返回 T& ,但是 C++ 禁止对位的引用。无法返回 bool&，因此std::vector<bool> 的 operatorp[] 返回一个行为类似 bool& 的对象。所以 std::vector<bool>::reference 对象必须具备 bool& s 可以使用的上下文。其中一个功能就是 std::vector<bool>::reference 可以隐式转换为 bool（不是 bool& 而是 bool ）。了解了这些细节后我们再来看代码：

```c++
bool highPriority = features(w)[5];		// declare highPriority's type explicitly
```

​	这里 feature 返回一个 std::vector<bool> 对象，在该对象上调用 operator[]。operator[] 返回一个 std::vector<bool>::reference 对象，然后隐式转换为初始化 highPriority 所需要的 bool。

相比 auto 变量情况：

```c++
auto highPriority = features(w)[5];		// deduce highPriority's type
```

​	此时 highPriority 的类型为 std::vector<bool>::reference 的一个副本，但是其中包含了 features(w) 返回的临时变量的内存信息，所以之后调用将会出现未定义的行为：

```c++
processWidget(w, highPriority);		// undefined behavior! highPriority contains
									// dangling pointer!
```

​	std::vector<bool>::reference 是代理类的一个例子：一个为了模拟和增强其他类型的行为而存在的类。代理类的用处很多。例如， std::vector<bool>reference 的存在提供了一种错觉，即 std::vector<bool> operator[] 返回对应位的引用，而标准库中的智能指针类型（参看 Chapter4）是将资源管理移植到原始指针上的代理类。代理类的效用是公认的，"代理"的思想也被应用于设计模式。

​	一些代理类被设计得一目了然，像 std::shared_ptr 和 std::unique_ptr 。另外一些则显得有些晦涩（会屏蔽一些复杂的实现），像 std::vector<bool>::reference ，跟类似的还有 std::bitset、std::bitset::reference。

​	此外 C++ 库中的采用了一种称为表达式模板的技术。此类库最初是为了提高数字代码的效率而开发的。给定一个类 Matrix 和 Matrix 的对象 m1，m2，m3 和 m4：

```c++
Matrix sum = m1 + m2 + m3 + m4;
```

​	如果 Matrix 对象的 operator+ 返回一个代表结果代理而不是结果本身。也就是说，两个 Matrix 对象的operator+ 将返回一个代理类的对象，例如 Sum<Matrix, Matrix> 而不是 Matrix 对象。与 std::vector<bool>::reference 和 bool 的情况一样，存在从代理到 Matrix 的隐式转换，这将允许从右侧表达式生成的代理对象初始化 sum（该类传统上会对整个表达式进行编码，即 Sum<Sum<Sum<Matrix,Matric>,Matrix>,Matrix>。这个肯定应该对用户屏蔽）。一般来说，这些存在"隐形"代理类不适用 auto。此类类的对象通常设计的寿命不会超过单个语句，因此创建这些类型的变量往往违反类设计的初衷。

因此你想避免这种形式的代码：

```c++
auto someVar = expression of "invisible" proxy class type;
```

但是如何识别代理对象是何时被使用呐？使用代码时很难发现他们的存在。他们应该是不可见的，至少概念上是这样的！一旦找到了他们，你真的要放弃 auto。让我们来先看看如何找到它们的问题，尽管"隐式"代理类被设计在日常使用中不受程序员关注，但在使用库中通常会记录他们是这样做的。这个时候除了文档外我们还可以查看对应的头文件，以 std::vector<bool>::reference 为例：

```c++
namespace std{
    template <class Allocator>{
        public:
        ...
        class reference{...}
        reference opreator[] (size_type n);
    };
}
```

​	假设你你就是要使用 auto 变量赋值又存在代理类咋办，看如下：

```c++
auto highPriority = static_cast<bool>(features(w)[5]);
auto sum = static_cast<Matrix>(m1+m2+m3+m4);
```

​	这种强转类型的语法不仅限于生成代理类类型的初始化程序。意创建与初始化表达式生成的变量类型不同的变量也有用。例如，计算公差值：

```c++
double calcEsilon();		// return tolerance value
```

​	calcEsilon 返回双精度值，但是假设对于你的应用程序，浮点数的精度是足够的，并且你关心浮点数和双精度数之间的大小差异。你可以声明一个浮点变量来存储结果：

```c++
float ep = calcEpsilon();	// implicitly convert double -> float
```

​	这并不能说明"我故意降低函数的精度"。但是，使用显式类型强制初始化语句的声明确实是：

```c++
auto ep = static_cast<float>(calcEpsilon());
```

再观察一个例子，通过计算一个 0.0 到 1.0 之间的双精度值（0.5表示容器中间）来获取随即访问迭代器（如，std::vector、std::deque、std::arrary）的元素。观察如下代码：

```
int index = d * c.size();				// 掩盖 double 转 int
auto index = static_cast<int>(d * c.size());	// 直观看出做了转换
```

**小结**

* "不可见"代理类会导致 auto 推导出初始化表达式的"错误"类型
* 显示类型转换的初始化用法可以强制 auto 推导出你想要的类型