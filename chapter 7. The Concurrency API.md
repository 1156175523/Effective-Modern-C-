* # chapter 7.  The Concurrency API

  ​		C++11 最大的成就之一就是将并发特性整合到语言和库中。熟悉其他线程 api（例如，pthead 或 Windows 线程）的程序员可能会发现 C++ 只是提供了相对简单的并发特性，这是因为 C++ 对并发的支持是基于编译器-编写器而存在。这就意味着，程序员可以编写跨平台的多线程程序。因此标准库的并发元素（task、future、thread、mutex、condition_variable、atomic等等）为开发并发软件提供了各种工具。

  ​		之后章节中，请注意标准库中有两种 future：std::future 和 std::shared_future。在很多情况下，区分并不重要，所以只会简单提到 future，意思是指这两种 future。

  #### *item 35. 优先使用任务模式而不是线程模式*

  ​		如果你想一部运行一个函数 doAsyncWork，你有两个选择。你可以创建 std::thread 并在线程中运行 doAsyncWork，从而使用基于线程的方式实现：

  ```c++
  int doAysncWork();
  std::thread t(doAsyncWork);
  ```

  或者你可以将 doAsyncWork 传递给 std::async，使用基于任务的策略实现：

  ```c++
  auto fut = std::async(doAsyncWork);		// "fut" for "future"
  ```

  在这个调用中，传递给 std::async 的函数对象（如，doAysncWork）被认为是一个任务。

  ​		基于任务的实现通常优于基于线程的实现，直观的就是代码量少。这里，doAsyncWork 会产生一个返回值。假设我们对调用 doAsyncWork 的返回值感兴趣，对于基于线程的调用，没有直接的方法来访问它。对于基于任务的方法，就很简单，因为从 std::async 返回的 future 提供了 get 方法。如果 doAsyncWork 抛出异常，get 函数的存在更重要，因为 get 也提供了对异常的访问。对于基于线程的实现，如果 doAsyncWork 抛出异常，程序就会直接死掉（通过调用 std::terminate）。

  ​		基于线程和基于任务的编程之间更根本的区别在于基于任务提供更高层抽象。它让你从线程管理的细节中解脱出来，这里需要总结一下并发 C++ 软件中"线程"的三个含义：

  * ***硬件线程***是实际执行计算的线程。现在的机器架构为每个 CPU 内核提供一个或多个硬件线程。
  * ***软件线程***（也称为 OS 线程或系统线程）是操作系统跨所有进程管理的线程并在硬件线程上调度执行。通常可以创建比硬件线程更多的软件线程，因为当软件线程被阻塞时（例如，在 I/O 或等待互斥量或条件变量时），可以通过执行其他未阻塞的线程提高吞吐量。
  * ***std::thread***是 C++ 进程中的对象，充当底层软件线程的句柄。一些 std::thread 对象表示"空"句柄，即，没有对应于软件线程，因为它们处于默认构造状态（因此没有函数可以执行），已经移动过的线程（移动 std::thread 后充当底层软件线程句柄），已经加入了-jioned （函数已经运行完成）或已经分离- detached（底层线程之间的连接已经被切断）。

  ​		软件线程是一种有限的资源。如果你尝试创建的线程数超出系统所能提供的范围，则会引发 std::system_error 异常。即使你要运行的函数不能抛出异常，也会是一样的情况。例如，即使 doAsyncWork 是 noexcept，

  ```c++
  int doAsyncWork() noexcept;				// see Item14 for noexcept
  ```

  如下语句可能会导致异常：

  ```c++
  std::thread t(doAsyncWork);				// throws if no more threads are avaliable
  ```

  ​		编写良好的软件必须规避这种情况，但是如何处理呐？一种方法是在当前线程上运行 doAsyncWork，但是这可能导致负载不平衡，如果当前线程是 GUI 线程，则会导致响应问题。另一个选择是等待一些现有的线程完成，然后再尝试创建一个新的 std::thread ，但是现有的线程可能正在等到 doAsyncWork 应该执行的操作（例如，产生一个结果或通知一个条件变量）。

  ​		即使没有用完线程，也可能会遇到过载的情况。就是当就绪准备运行（即为阻塞）的软件线程比硬件线程多时。当出现这种情况时，线程调度（通常是操作系统的一部分）会对这些软件线程进行时间片轮询处理。当一个线程的时间片完成而开始另一个线程时，会执行上下文切换。这样的上下文切换会系统的整体线程管理开销，并且当软件线程被调度到的硬件线程与之前软件线程时间片调度不同时，会有更高的开销。这种情况下，（1）CPU缓存对于该软件线程通常是陌生的（即，它包含很少对其有用的数据和指令）和（2）在该内核上运行"新"软件线程会"污染"已在该核心上运行并可能被安排再次在该内核上运行的"旧"线程的 CPU 缓存。

  ​		避免过载是很困难的，因为软件线程与硬件线程的最佳比率取决于软件线程的可运行频率，并且可能会动态变化，例如，当程序从 I/O 密集型转到计算密集型时。软件线程与硬件线程的最佳比率还取决于上下文切换的成本以及软件线程使用 CPU 缓存效率。此外，硬件线程的数量和 CPU 缓存的详细信息（例如，它们的大小和它们的相对速度）取决于机器架构，因此即使你在一个平台已经调整应用程序避免过载（同时仍然保持硬件资源紧张），不能保证你的解决方案在其他类型的机器上也能正常运行。

  ​		使用 std::async 可以完美规避这个问题，将这些处理交给标准库：

  ```c++
  auto fut = std::async(doAsyncWork);	// onus of thread mgmt is on implement of 
  									// the Standard Library
  ```

  ​		这个调用将线程管理责任转移到 C++ 标准库的是实现者。例如，收到超限线程异常的可能性显著降低，因为这个调用永远不会产生异常。你可能会想"怎么可能？"。"如果我要创建的线程比系统所提供的多，那我通过调用 std::thread 和 std::async 又有啥关系呐？"。当然有关系，因为以上方式调用 std::async 时（

  即，使用默认启动策略--参看 Item36），并不能保证它会创建一个新的软件线程。相反，它允许调度程序安排指定的函数（本例中为 doAsyncWork）在请求 doAsyncWork 结果的线程上运行（即，在调用 get 或 wait future 的线程上），并且如果系统过载或者线程不足时，通过配置启动策略，调度程序可以合理地自动利用其中一种运行方式。

  ​		如果你使用"在需要结果地线程上运行它"的方式编程时，我之前提过可能会遇到负载平衡的问题，并且这些问题不会因为你使用了 std::async 和运行时调度器而消失。当涉及到负载平衡时，运行时调度器会比你更全面地了解机器上发生地事情，因为它管理来自所有进程地线程，而不仅仅是运行你的代码进程。

  ​		使用 std::async 时，GUI 线程上响应问题依然存在，因为调度器无法知道哪个线程有严格的响应要求。在这种情况下，你需要将 std::launch::async 启动策略传递给 std::async。以此确保你要运行的函数运行在不同的线程上（参看Item36）。

  ​		最先进的线程调度器使用系统范围的线程池来避免过载，并且它们通过工作窃取算法改善了硬件内核之间的负载平衡。C++ 标准不要求使用线程池或工作窃取算法，而且老实说，在 C++11 并发规范方面的一些技术使用它们比想象中要困难。尽管如此，一些供应商在他们的标准库实现中还是利用了这项技术。如果你采用基于任务的方法进行并发编程，那么随着这种技术的广泛应用，你会不断从中获取其带来的好处。另一方面，如果你直接使用 std::thread 编写程序，你就承担了处理线程消耗、过载和负载平衡的负担，更不用说你的解决方案如何保持与同一台机器上其他进程中运行的程序相吻合。

  ​		与基于线程的编程相比，基于任务的设计使你避免手动管理线程的麻烦，并且它提供了一种自然的方式来检查异步执行函数的结果（即，返回值或异常）。尽管如此，某些情况下，直接使用线程可能还是合适的。它们包括：

  * 你需要访问底层线程实现的 API。C++ 并发 API 通常使用底层级别的特定于平台的 API 实现，一般是 pthread 或 Window的 Thread。这些 API 目前比 C++ 提供的更丰富。（例如，C++ 没有线程优先级或关联性的概念）为了提供对底层线程实现的 API 的访问，std:thread 对象通常提供 native_handle 成员函数。对于 std::future（即 std::async 返回的内容），没有与此对用的功能。
  * 你需要应用程序进行线程优化。例如，你正在开发一个执行已知配置文件的服务器软件，该软件作为唯一重要的进程部署在固定硬件特征的机器上。
  * 你需要在 C++ 并发 API 之外实现线程技术，例如，实现线程池。

  但是这些都是比较罕见的情况。大多数时候，你应该选择基于任务的设计，而不是使用基于线程的编程。

     **小结**

  * std::thread API 没有提供从异步运行函数中获取返回值的直接方法，如果这些运行的函数抛出异常，程序就会终止。
  * 基于线程的编程需要手动管理线程消耗、过载、负载平衡和适应新平台。
  * 基于任务的编程通过 std::async 和默认启动策略可以帮你成功规避这些问题。

  ---

  #### *item 36 如果异步必不可少，则指定 std::launch::async*

  ​		当你调用 std::async 来执行一个函数（或其他可调用对象）时，你通常打算异步运行该函数。std::async 有两个标准启动策略，每个策略都由 std::launch 枚举器的范围枚举表示。（有关范围枚举，请参看 Item10）假设函数 f 被传递给 std::async 执行，

  * std::launch::async 启动策略指 f 必须异步运行，即在不同的线程上运行
  * std::launch::deferred 启动策略指 f 只能在由 std::async 返回 future 调用 get 或 wait 时运行。也就是说，f 的执行被推迟到这些调用之后。当 get 或 wait 被调用时，f 将同步执行，也就是说，调用者将阻塞直到 f 完成运行。如果 get 和 wait 都没有被调用，f 将永远不会运行。

  ​		有一点需要注意，std::async ***默认的启动方式***既不是单独使用启动策略中的其中一个，而是***两个启动策略的组合***：

  ```c++
  auto fut1 = std::async(f);			// run f using default launch policy
  
  auto fut2 = std::async(std::launch::async|std::launch::deferred, 	// run f either
                        f);											// deferred or 																		// async
  ```

  ​		因此，默认启动策略允许 f 以异步或同步的方式运行。正如 Item35 所指出的，这种灵活性使得标准库的 std::async 和线程管理组件承担起了线程创建和销毁，避免过载以及负载平衡的工作。这就是使用 std::async 进行并发编程如此方便的原因之一。

  ​		但是使用默认启动策略的 std::async 有一些有趣的含义。给定一个执行此语句的线程 t，

  ```c++
  auto fut = std::async(f);			// run f using default launch policy
  ```

  * 无法预测 f 是否会与 t 并发运行，因为 f 可能被安排为 deferred 方式运行（延迟运行）。
  * 无法预测 f 是否运行在调用 get 或 wait fut 的线程上。如果这个线程是 t，这就意味着不能预测 f 是否运行在一个不同于 t 的线程上
  * 可能无法预测 f 是否运行，因为不能保证 fut 在程序的每条路径上都会调用 get 或 在 fut 上 wait。

  

  ​		默认启动策略的灵活性通常会使得 thread_local 变量的访问变得模糊，因为如果 f 访问 thread_local storage（TLS）时，无法预测哪个线程的变量将被访问：

  ```c++
  auto fut = std::async(f);		// TLS for f possibly for independent thread,but 
  								// possibly for thread invoking get or wait on fut
  ```

  ​		这还会影响使用超时的基于等待的循环，因为对于 std::launch::deferred 任务（参看 Item35）调用 wait_for 或 wait_until 会延迟等待结果。这就意味着下面的循环，看起来应该最终结束，实际上，可能会永远运行：

  ```c++
  using namespace std::literals;			// for C++14 duration suffixes; see Item34
  void f()
  {
      std::this_thread::sleep_for(1s);	// f sleep for 1 second, then returns
  }
  
  auto fut = std::async(f);				// run f asynchronously (conceptually)
  
  while(fut.wait_for(100ms) != std::future_status::ready)		// loop until f has
  {															// finished running...
      ...														// which may never happen!
  }
  ```

  ​		如果 f 与调用 std::async 的线程并发运行（即，如果为 f 选择的启动策略是 std::launch::async），则这里没有问题（假设 f 最终完成），但是如果 f 使用 deferred 延迟处理，则 fut.wait_for 将始终返回 std::future::timeout(如果设置时延)，永远不会出现 std::future_status::ready，所以出现死循环。

  ​		这种 bug 在开发和单元测试过程中很容易被忽略，因为它们只有在高负载下才会被表现出来。就是当系统负载过高或线程耗尽时，任务最终有可能被指定为延迟策略。毕竟，如果硬件没有过载或线程耗尽的危险，系统没有理由不将任务调度为并发执行。

  ​		修复的方法很简单：就只需要检查 std::async 调用对应的 future 来查看任务是否被延迟，如果是的话避免进入基于超时的循环。尴尬的是没有直接的方法来查看 future 的任务是否被延迟。相反，你必须调用一个基于超时的函数--一个函数像 wait_for。在本例中，你并不想等待任何东西，你只想看看返回值是否为 std::future_status::deferred，因此，不用怀疑直接将 wait_for 的超时设置为 0 （***查阅STL使用手册发现如果是 deferred 任务时，wait_for会立即返回 std::future_status::deferred***【Docker linux 测试还是返回timeout，VS2019 设置时间>=0 都是deferred】，【docker linux】***而且使用 async 默认创建的 future 是 deferred， 所以建议显示指定***）：

  ```c++
  auto fut = std::async(f);				// as above
  if (fut.wait_for(0s) == std::future_status::deferred)		// if task is deferred...
  {										// ... use wait or get on fut
      ...									// to call f synchronously
  }
  else									// task isn't deferred
  {
      while(fut.wait_for(100ms) != std::future_status::ready)		// infinite loop not
      {															// possible (assuming)
   																// f finishes
          ...		// task is neither deferred nor ready,
              	// so do concurrent work until it's ready
      }
      ...			// fut is ready
  }
  ```

  ​		根据上面的结果，满足以下条件，使用 std::async 和 默认的任务启动策略是可以的：

  * 该任务不需要与调用 get 或 wait 的线程并发运行
  * 不管哪个线程的 thread_local 变量是被读还是被写
  * 要么保证在 std::async 返回的 future 中会调用 get 或 wait，要么可以接受这个任务可能不会永远不会被执行。
  * 使用 wait_for 或 wait_until 代码判断是延迟任务的可能性d

  如果这些条件任何一条都不满足，你希望 std::async 真正异步执行安排的任务。就必须显示指定启动策略为 std::launch::async：

  ```c++
  auto fut = std::async(std::launch::async, f);	// launch f asynchronously
  ```

  ​		事实上，有一个功能类似 std::async，但是自动使用 std::launch::async 作为启动策略的函数，是一个很方便的工具，所以编写很方便。如下是 C++11 版本：

  ```c++
  template<typename F, typename... Ts>
  inline
  std::future<typename std::result_of<F(Ts...)>::type>
  reallyAsync(F&& f, Ts&&... params)				// return future
  {												// for asynchronous
      return std::async(std::launch::async,		// call to f(params...)
                       std::forward<F>(f),
                       std::forward<Ts>(params)...);
  }
  ```

  ​		这个函数接受一个可调用对象 f 和 0 个或多个参数 params，并将它们完美转发给（参看Item25） std::async，传递 std::launch::async 作为启动策略。像 std::async 一样，它返回一个 std::future 作为入参 params 调用 f 的结果。确定结果类型很简单，因为特征类型（type trait）std::result_of 可以提供。（关于类型特征的一般信息参看 Item9），reallyAsync 的使用就像 std::saync：

  ```c++
  auto fut = reallyAsync(f);			// run f asynchronously; throw if std::async
  									// would throw
  ```

  C++14 中，推导 async 的返回类型的能力简化了函数声明：

  ```c++
  template<typename F, typename... Ts>
  inline
  auto
  reallyAsync(F&& f, Ts&&... params)
  {
      return std::async(std::launch::async,
                       std::forward<F>(f),
                       std::forward<Ts>(params)...
                        );
  }
  ```

     **小结**

  * std::async 的默认启动策略是允许自动选择同步或异步执行任务
  * 这种灵活性导致访问 thread_local 时的不准确性，也就是说这个任务可能永远不会执行，并且影响了基于超时的等待调用的程序逻辑
  * 如果需要执行异步任务，请指定 std::launch::async

  ---

  #### *item 37. 使 std::threads 在所有路径上都是 unjoinable*

  ​		每一个 std::thread 对象都处于两种状态之一：joinable 或 unjoinable。一个 joinable 线程对应于可运行或正在运行的地城异步执行线程。例如，与被阻塞或等待调度的底层线程相对应的 std::thread 是 joinable。与底层线程相对应的 std::thread 对象也被认为是 joinable。

  ​		一个 unjoinable 的 std::thread 就是预期的那样：一个不可以 joinable 的std::thread。非 joinable 的 std::thread 对象包括：

  * 默认构造函数 std::thread。这样的 std::thread 没有要执行的函数，因此不对应底层的执行线程
  * std::thread 对象已经被移动。移动的结果就是，用于执行（如果有的话）的 std::thread 的底层线程现在对应于一个不同的 std::thread
  * std::thread 已经 joined 的线程。在 joined 之后，std::thread 对象不再对应于已经完成运行的底层执行线程
  * std::thread 已经被 detached。分离会断开一个 std::thread 对象和它所对应的底层执行线程之间的连接

  ​		std::thread 的可连接性很重要的一个原因是，如果调用可连接线程的析构函数，则程序的执行将终止。例如，假设我们有一个函数 doWork，它接受一个过滤函数 filter 和一个最大值 maxVal 作为参数。doWork 检查 0 ~ maxVal 之间的所有值满足计算条件。如果这种过滤很费时，判断 doWork 的条件是否满足也很费时，那么这两件事并发处理是合理的。

  ​		我们之前说设计倾向于基于任务模式（参看Item35）,但是假设我们想设置执行过滤线程的优先级。Item35 解释了着需要使用线程本地句柄，并且只能通过 std::thread API 访问；基于任务的 API（即 future）不提供。因此我们的实现是基于线程，而不是基于任务。

  实现大概如下：

  ```c++
  constexpr auto tenMillion = 10000000;		// see Item 15 for constexpr
  
  bool doWork(std::function<bool(int)> filter,	// returns whether computation was
             int maxVal = tenMillion)				// performed; see Item2 for
  {												// std::function
      
      std::vector<int> goodVals;					// values that satisfy filter
      
      std::thread t([&filter, maxVal, &goodVals]	// populate goodVals
                    {
                        for (auto i=0; i<=maxVal; ++i)
                        {
                            if (filter(i)) goodVals.push_back(i);
                        }
                    }
                   );
      
      auto nh = t.native_handle();		// use t's handle to set it's priority
      ...
          
  	if (conditionsAreSatisfied())
      {
          t.join();						// let t finish
          performComputation(goodVals);
          return true;
      }
      return false;						// computation was not performed
  }
  ```

  在解释为什么这段代码有问题之前，我要说明一下，利用 C++14 的标号作为数字的分隔符的能力，可以使 tenMilion 的初始化值更具可读性：

  ```c++
  constexpr auto tenMillion = 10'000'000;			//C++14
  ```

  ​		需要提一嘴，在线程 t 开始运行后设置它的优先级有点像马逃跑后关闭马厩门。更好的设计是让 t 处于挂机状态（这样就可以在它进行任何计算之前调整它的优先级），但是这里为了不分散读者注意力，缺失的有关代码可以参看 Item39，其中讲述了如何启动挂起的线程。

  ​		回到 doWork。如果 conditionsAreSatisfied() 返回 true，一切正常，但是它返回 false 或抛出异常，当 doWork 结束时调用其析构函数时，std::thread 对象 t 将会是 joinable 。这会导致程序的执行终止。

  ​		你可能想知道为什么 std::thread 析构函数会以这种方式运行。只是因为另外两个明显的选择可以说更糟。它们是：

  * **隐式连接（join）**。这种情况下，std::thread 的析构函数将等待其底层异步执行线程完成。这听起来很合理，但是可能导致难以追踪的性能异常。例如，如果 conditionsAreSatisfied() 已经返回 false，那么 doWork 将等待其过滤器应用于所有值（就是还会进行计算处理），这违背了我们的直观理解。
  * **隐式分离（detach）**。这种情况下，一个 std::thread 的异构函数将切断 std::thread 对象与其底层执行线程之间的连接。底层线程继续运行。这听起来和 join 方法一样合理，但是它可能导致更严重的调试问题。例如，在 doWork 中，goodVals 是一个通过引用捕获的局部变量。它在 lambda 内部被修改（通常通过 push_back）。假设，当 lambda 异步运行时，conditionsAreSatisfied() 返回 false。这种情况下，doWork 将返回，并且它的局部变量（包括 goodVals）将被销毁。它的堆栈帧将会被弹出，其线程将在 doWork 的调用点继续执行。该调用点之后的语句在某些时候会进行其他函数调用，并且可能会有这样的调用，它会使用曾经被 doWork 堆栈帧占用过的部分或全部内存。我假设称这个函数为 f。当 f 运行时，doWork 启动的 lambda 仍然会异步运行。该 lambda 可以在堆栈内存上调用 push_back，该堆栈内存曾经是 goodVals 但是现在位于 f 的堆栈帧内的某个位置。这样的调用会修改曾经是 goodVals 的内存，也就意味着从 f 的角度来看，其堆栈帧中的内容可能会自己改变！

  ​		标准化委员会认为，破坏可连接线程的后果非常严重，以至于他们基本上禁止了它（析构一个可连接线程会导致线程终止）。

  ​		也就是说你得确保，你使用的 std::thread 对象，在定义它的作用域之外每条路径上都是不可连接的。但是覆盖每条路径可能很富在。它包括从作用域末尾结束，以及通过 return、continue、break、goto 或 exception 跳出。可以有很多路径。

  ​		每条路径代码块结束后要执行一些操作时，一般的做法是将该操作放在局部对象的析构函数中。这类对象称为 RAII 对象，它们来自的类成为 RAII 类（RAII 本身代表 "Resource Acquisition Is Initialization"，尽管该技术的关键是销毁而不是初始化）。RAII 类在标准库中很常见，例如包括 STL 容器（每个容器的析构函数会销毁容器的内容并释放其内存）、标准智能指针（Item18-20 阐述了 std::unique_ptr 的析构函数对指向的对象调用其删除器，以及 std::shared_ptr 中的析构函数和 std::weak_ptr 递减引用计数），std::fstream 对象（它们的析构函数关闭它们对应的文件）等等。然而对于 std::thread 对象并没有标准的 RAII 类，这可能是因为标准委员会拒绝了 join 和 detach 作为默认选项，根本不知道这样的类应该做什么。

  ​		好在我们自己编写一个并不困难。例如，下面的类允许调用者指定当 ThreadRAII 对象（std::thread 的 RAII 对象）被销毁时应该调用 join 还是 detach:

  ```c++
  class ThreadRAII{
    public:
      enum class DtorAction{join, detach};	// see Item 10 for enum class info
      
      ThreadRAII(std::thread&& t, DtorAction a) // in dtor, take action a on t
      :aciton(a),t(std::move(t)){}
      
      ~ThreadRAII()
      {										// see below for joinability test
          if (t.joinable())
          {
              if(action == DtorAction::join)
              {
                  t.join();
              }
              else
          	{
              	t.detach();
          	}
          }
          
      }
      
      std::thread& get() {return t;}			// see below
      
    private:
      DtorAction action;
      std::thread t;
  };
  ```
  
  上面的代码已经很清晰明了，但是还是需要描述几点：
  
  * 构造函数只接受 std::thread 右值，因为我们想将传入的 std::thread 移动到 ThreadRAII 对象中（回想一下 std::thread 对象是不可复制的）。
  * 构造函数中的参数顺序被设计为对调用者来说直观的方式（首先指定 std::thread，然后指定析构函数动作，这样比反过来更容易理解），但是成员初始化列表被设计成先匹配数据成员的声明顺序。这个顺序将 std::thread 对象放在最后。在这个类中，顺序没有区别，但通常，一个数据成员的初始化可能依赖于另外一个数据成员，而且由于 std::thread 对象可能会在初始化后立即开始运行函数，因此在类中最后声明它是一个好习惯。这保证了在构造它时，在其之前的所有数据成员都已经初始化，因此保证被异步运行线程 std::thread 所需的数据成员安全访问。
  * ThreadRAII 提供了一个 get 函数来访问底层的 std::thread 对象。这类似于标准智能指针提供的 get 函数，这些函数允许访问其底层的原始指针。提供 get 可以避免 ThreadRAII 复制完整的 std::thread 接口，这意味着 ThreadRAII 对象可以在需要 std::thread 对象的上下文中使用。
  * 在 ThreadRAII 析构函数调用 std::thread 对象 t 上的成员函数之前，它会检查以确保 t 是可连接的。这是必要的，因为***在不可链接的线程上调用 join 或 detach 会产生未定义的行为***。有可能客户端构建了一个 std::thread ，从中创建了一个 ThreadRAII 对象，使用 get 获取对 t 的访问，然后对 t 进行移动操作或调用 join 或 detach 操作。这些操作都会导致 t 成为不可链接状态。
  
  你可能会担心这段代码，
  
  ```c++
  if (t.joinable())
  {
      if (action == DtorAction::join)
      {
          t.join();
      }
      else
      {
          t.detach();
      }
  }
  ```
  
  可能存在竞争，因为在 t.joinable() 和 join 或 detach 的调用之间，可能存在另外一个线程使得 t 变为不可链接状态（unjoinable），有这个想法很好，但是这种担忧是无依据的。std::thread 对象只有通过调用成员函数 join 、detach 或 对其进行了移动操作才会从 joinable 状态变为 unjoinable 状态。在调用 ThreadRAII 对象的析构函数时，不应该由其他线程对该对象进行成员函数调用。如果同时调用，则肯定存在竞争，但它不存在析构函数内部，而是在客户端代码中，试图同时调用一个对象的两个数据成员函数（析构函数和其他函数）。一般来说，只有当所有成员函数都是 const 成员函数时，对单个对象的同时成员函数调用才是安全的（参看Item16）。使用 ThreadRAII 应用到我们的 doWork 例子中是这样的：
  
  ```c++
  bool doWork(std::function<boo(int)> filter,		// as before
              int maxVal = temMillion)
  {
      std::vector<int> goodVals;					// as before
      
      ThreadRAII t(std::thread([&filter, maxVal, &goodVals]
                               {
                                  for (auto i=0; i<=maxVal; ++i)
                                  {
                                      if (filter(i)) goodVals.push_back(i);
                                  }
                               }),
                   			ThreadRAII::DtorAction::join	// RAII action
                  );
      
      auto nh = t.get().native_handle();
      ...
  	if (conditionsAreSatisfied())
      {
          t.get().join();
          performComputation(goodVals);
          return true;
      }
      return false;
  }
  ```
  
  ​		例子中我们选择在 ThreadRAII 析构函数中对异步运行的线程执行 join，因为之前解释过进行 detach 可能会导致一些一些难以调试的现象。之前也提到了如果执行 join 会导致出现性能异常（坦率地说，这可能也会导致调试困难），但是相比较未定义的行为（detach 导致的）、程序终止（使用原始 std::thread）或性能异常，性能异常似乎也是最好的选择了。
  
  ​		Item39 阐述了使用 ThreadRAII 析构函数中执行 std::thread 的 join 操作不仅会导致性能问题，还会导致程序挂起。这类问题的 "正确" 解决方案是与异步运行的 lambda 通信，我们不再需要它工作时，它应该提前返回，但是 C++11 中不支持中断线程。但是可以通过我们自己实现，这里就不赘述了读者参看 Item39。
  
  ​		Item17 提到，因为 ThreadRAII 声明了一个析构函数，所以编译器不会生成移动操作，但是 ThreadRAII 对象没有理由不能移动。如果编译器要生成这些函数，这些函数可以正常运行，所有需要我们自己创建：
  
  ```c++
  class ThreadRAII
  {
    public:
      enum class DtorAction{join, detach};		// as before
      
      ThreadRAII(std::thread&& t, DtorAction a)	// as before
      :action(a),t(std::move(t)){}
      
      ~ThreadRAII(){...}
      
      ThreadRAII(ThreadRAII&&) = default;					// support moving
      ThreadRAII& operator=(ThreadRAII&&) = default;
      
      std::thread& get() {return t;}				// as before
      
    private:										// as before
      DtorAction action;
      std::thread t;
  };
  ```
  
     **小结**
  
  * 使 std::thread 在所有路径上都是 unjoinable
  * std::thread 析构函数执行 join 会导致难以调试的性能异常
  * std::thread 析构函数执行 detach 可能导致难以调试的未定义行为
  * 在数据成员列表中最后声明 std::thread 对象
  
  ---
  
  #### *item 38. 注意不同线程句柄析构函数行为*
  
  ​		Item37 阐述了一个 joinable std::thread 对应一个底层运行的系统线程。非延迟任务 future（见 Item36）与系统线程也是类似关系。因此，std::thread 对象和 future 对象都可以认为是系统线程的句柄。
  
  ​		但是有趣的是，std::thread 和 future 的析构函数行为却不相同。正如 Item37 所指出的，销毁一个 joinable 状态的线程会导致你的程序终止，因为两个明显的替代方法----隐式join和隐式detach----会产生更糟糕的结果。但是，future 的析构函数有时表现得好像它做了隐式join，有时又好像它做了隐式detach，有时又好像两者都没做。它永远不会导致程序终止。这这种处理行为值得我们仔细研究。
  
  ​		我们可以观察到 future 是通信通道的一端，通过它，被调用方将结果传递给调用方。被调用方（通常是异步运行的）将计算结果写入通信通道（通常是通过 std::promise 对象），调用方使用 future 读取结果。你可以认为，虚线箭头表示从被调用方到调用方的信息流：
  
  **【Caller】future <---------------------------------------std::promise(typically)【Callee】**
  
  ​		但是被调用者的结果存储在哪里呐？被调用者可能会在调用者调用相应的 future 之前完成，所以结果不能存储在被调用者的 std::promise 中。该对象是被调用方的本地对象，当被调用方完成时将被销毁。
  
  ​		结果也不能存储在调用者的 future 中，因为（除其他原因外）一个 std::future 可用于创建一个 std::shared_future（从而将被调用者结果的所有权从 std::future 转移到 std::shared_future），然后可以在源 std::future 销毁后多次复制。考虑到并非所有结果类型都可以复制（如，仅移动类型）并且结果的生命周期必须至少与引用它的最后一个 future 一样长，那么与被调用者对应的潜在的多个 future 中，哪个一个应该包含它的结果？
  
  ​		因为被调用者关联的对象和与调用者关联的对象都不是存储结果的合适位置。这块区域称为共享状态。共享状态通常由基于堆的对象表示，但是没有标准的类型、接口和实现。标准库作者可以以他们喜欢的任何方式自由的实现共享状态。
  
  ​		我们可以设想被调用者、调用者和共享状态之间的关系如下，使用虚线箭头表示：
  
  **【Caller】future<---------[Callee's Result] Shared State<--------std::promise(typically)【Callee】**
  
  ​		共享状态的存储很重要，因为 future 的析构函数（本 Item 的主题）的行为是由与 future 关联的共享状态决定的。特别是，
  
  * 非延迟类型最后一个 future 的析构函数指向一个通过 std::async 生成的共享状态，直到该任务完成。本质上，这种 future 的析构函数对正在运行的异步执行任务的线程执行了隐式 join
  * 所有其他类型 future 对象的析构函数只会销毁 future 对象。对于运行异步运行的任务，这类似于底层线程的隐式 detach。对于延迟任务不是最后一个 future ，其任务永远不会运行。
  
  
  
  ​		这些规则听起来比实际复杂。我们真正处理的是一种简单的"正常"行为和一个例外。正常的行为是 future 的析构函数会销毁 future 对象，仅此而已。不会加入任何东西，不 detach 任何东西，不运行任何东西。它只会销毁 future 的数据成员。（实际上，它还做了其他事。它递减共享状态中的引用计数，这个状态被引用它的 future 和被调用者的 std::promise 操作。这个引用计数使库能够知道何时可以销毁共享状态。有关引用计数相关信息可以参看 Item19。）
  
  ​		这种正常行为的例外情况只出现在满足以下所有条件的future ：
  
  * 通过调用 std::async 而创建的共享状态 future
  * 任务的启动策略是 std::launch::async（参看Item36），要么因为它是由运行时系统选择的，要么因为它是在调用 std::async 时指定的
  * future 是指向共享状态的最后一个 future。对于 std::future ，永远是这个情况。对于 std::shared_future ，如果其他关联相同共享状态的 std::shared_future 只会"正常销毁"（即，简单的销毁它的数据成员）
  
  ​		只有当这些条件都满足时，future 的析构函数才表现出特殊的行为，**这种行为是阻塞的**，**直到异步任务运行的任务完成**。实际上，这相当于一个与运行 std::async 创建的任务线程的隐式 join。
  
  ​		你可能会问"为什么对于由 std::async 启动的非延迟任务有这个特殊的共享状态规则？"。据我所知，标准委员会希望避免隐式 detach 相关的问题（参看Item37），但它们不想采取激进方式强制程序终止（像可连接的 std::thread ，参看Item37），所以他们对隐式 join 进行了妥协。这个决定并非没有争议，过于 C++14 中放弃这种行为的讨论更严肃。最后，没有做任何更改，所以 C++11 和 C++14 中 future 析构函数的行为一致。
  
  ​		future 的 API  无法确定由 std::async 产生的 future 是否指向共享状态，因此给定一个任意 future 对象，无法确定它是否会在析构函数中阻塞，等待一个异步任务的完成。这里有一些有趣的暗示：
  
  ```c++
  // this container might block in its dtor,because one or more contained futures 
  // could refer to a shared state for a non-deferred task launched via std::async
  std::vector<std::future<void>> futs;	// see Item 39 for info on std::future<void>
  
  class Widget{
    public:
      ...
    private:
      std::shared_future<double> fut;
  };
  ```
  
  ​		当然，你有办法知道给定的 future 是否满足触发特殊析构函数（例如，有程序逻辑实现），你可以保证 future 不会阻在其析构函数中阻塞。例如，只有调用 std::async 产生的共享状态才有资格获得这种特殊行为，但有其他方式可以创建共享状态。一种是使用 std::packaged_task。std::packaged_task 对象为异步执行准备一个函数（或其他可调用对象），方法是包装它，使其结果放入共享状态。然后可以通过 std::packaged_task 的 get_future 函数获取引用该共享状态的 future：
  
  ```c++
  int calcValue();				// func to run
  std::packaged_task<int()> pt(calcValue);		// wrap calcValue so it can run 
  												// asynchronously
  auto fut = pt.get_future();		// get future for pt
  ```
  
  ​		这里我们知道 fut 不是通过调用 std::async 创建的共享状态，因此它的析构函数是正常行为。一旦被创建，std::packaged_task 就可以在线程上运行。（也可以通过调用 std::async 来运行，但是如果你想使用 std::async 运行一个任务，就没有什么理由创建一个 std::packaged_task，因为在被调度任务运行之前 std::async 可以做任何 std::packaged_task 所做的事）。
  
  ​		std::packaged_task 不可以复制，因此当 pt 传递给 std::thread 构造函数时，必须将其强制转换为右值（通过 std::move - 参看Item23）:
  
  ```c++
  std::thread t(std::move(pt));		// run pt on t
  ```
  
  ​		这个例子可以让我们深入了解 future 析构函数的正常行为，但更容易看出这些语句是否放在一个块中：
  
  ```c++
  {				// begin block
      std::packaged_task<int()> pt(calcValue);
      
      auto fut = pt.get_future();
      std::thread t(std::move(pt));
      ...									// see below
  }										// end block
  ```
  
  ​		有意思的是在创建 std::thread t 对象后 "..." 代码块结束之前这个区域内 t 会发生什么。存在三种可能：
  
  * t 什么也没发生。这种情况下，t 将在代码块结尾变为 joinable。这将导致程序终止（参看Item37）
  * 在 t 上执行 join。这种情况下，不需要 fut 在其析构函数中阻塞，因为调用代码中已经存在 join
  * detach t。这种情况下，fut 不需要在其析构函数中 detach ，因为调用代码中已经这样做了
  
  ​		换句话说，当你有一个 future 对应于由于 std::packaged::task 产生的共享状态时，通常不需要采用特殊的销毁策略，因为终止、join 或 detach 的操作将通过运行它的 std::thead 代码决定。
  
     **小结**
  
  * future 析构函数通常只是销毁 future 的数据成员
  * 由 std::async 由 std::lauch::async 创建的共享状态 future 会被阻塞到任务完成为止
  
  ---
  
  #### *item 39. 考虑一次性事件通信的无效性*
  
  ​		有时，一个任务通知第二个异步运行的任务发生了特定的事情是很有用的，因为第二个任务在事件发生之前无法继续运行。可能是数据结构已经初始化，计算阶段已经完成，或者与检测到重复的操作。这种情况下，进行线程间通信最佳方式是什么？
  
  ​		最直观的实现方法是使用条件变量（condvar）。我们将检测条件的任务称为检查任务，将对条件做出反应的任务称为响应任务，那么思路很简单：响应的任务等待一个条件变量，而检测线程在事件发生后通知该条件变量。检测任务实现很简单：
  
  ```c++
  std::condition_variable cv;			// condvar for event
  std::mutex;							// mutex for use with cv
  ...								// detect event
  cv.notify_one();					// tell reacting task
  ```
  
  ​		如果有多个响应任务需要通知，那么将 notify_one 替换为 notify_all ，但是这里我们假设只有一个响应任务。
  
  ​		响应任务的代码有点复杂，因为在调用条件变量上 wait 之前，它必须通过 std::unique_lock 对象锁定一个互斥对象。（在等待条件变量之前锁定互斥对象是线程库的典型做法。通过 std::unique_lock 对象锁定互对象的需求只是 C++11 API 的一部分）以下是伪代码：
  
  ```c++
  ...					// prepare to react
      
  {					// open critical section
      std::unique_lock<std::mutex> lc(m);		// lock mutex
      cv.wait(ll);	// wait for notify ;this isn't correct~
      
      ...				// react to event (m is locked)
          
  }					// close crit. section; unlock m via lk's dtor
  
  ...					// continue reactiong (m now unlocked)
  ```
  
  ​		这种实现总让人产生一种感觉就是两个任务之间存在共享数据。但是有时检查和响应任务之间只是顺序依赖，不会涉及到共享数据。例如，检测任务可能负责初始化全局数据结构，然后将其转交给相应任务完成后续任务操作。检测任务初始完数据结构之后就不会访问该数据结构，那么这两个任务之间是逻辑分离的。不需要互斥锁。
  
  ​		另外还有两个问题需要注意：
  
  * 如果检测任务在响应任务等待之前通知条件变量，响应任务将会挂起。为了通知一个条件变量唤醒另外一个任务，另一个任务必须等待该条件变量。如果检测任务恰好在响应任务执行等待之前执行通知，响应任务将错过通知，并且永远处于等待状态。
  * wait 语句无法识别虚假唤醒。线程 api （许多语言中---不仅仅 C++） 中的一个事实是，即使没有通知条件变量，等待条件变量的代码也可能被唤醒。这种唤醒被称为虚假唤醒。正确的做法是通过确认正在等待的条件确实发生来处理它们，并将此作为唤醒后第一个动作。C++ 条件变量 API是处理这中异常情况很简单，因为它允许将lambda（或其他函数对象）测试等待条件传递给 wait 。响应任务中的 wait 调用可以这样实现，利用这个功能需要响应任务能够确定它等待的条件是否为真。但在我们考虑的场景中，它等待的条件是检测线程负责的事件是否发生。响应线程可能无法确定它等待的事件是否已经发生。这也是为啥它要等待一个条件变量的原因。
  
  ```c++
  cv.wait(lk,
         []{return whether the event has occurred;});
  ```
  
  ​		有很多情况下，使用条件变量进行任务通信很合适，但是这里的情况并不是其中之一。接下来很多人就会想到另一个技巧---共享布尔标志。该标志最初是 false，当检测任务完成他要标识的任务后，设置它为 true:
  
  ```c++
  std::atomic<bool> flags(false);	// shared flag; see Item40 for std::atomic
  
  ...								// detect event
      
  flag = ture;					// tell reacting task
  ```
  
  ​		而响应线程只轮询该标志。当它看到标志被设置，它就知道所等待的事件已经发生：
  
  ```c++
  ...								// prepare to react
      while(!flag);				// wait for event
  ...								// react to event
  ```
  
  ​		这种实现方法没有基于条件变量的设计缺陷。没有必要使用互斥锁。如果检测线程在响应线程之前设置了标志，那也没问题，也不会存在类似虚假互换的东西。
  
  ​		不好的一点是响应任务中的轮询成本。在任务等待标志设置期间，任务本质上是被阻塞的，但是它仍在运行。因此，它可能会占用另外一个任务正在使用的硬件线程，每次启动或完成其时间片时都会产生上下文切换的成本，并且它可能会一直占用内核运行（CPU100%），但是如果没有轮询的话就会节省这些开销。一个真正得到受阻任务不会做这些事情。这也是基于条件变量的优势，它确实使响应任务处于阻塞状态。一般的设计中会将条件变量和标记相结合。标记用于标识事件是否发生，但是对标记的访问是互斥锁同步的。
  
  ​		因为互斥锁组织了对该标记的并发访问，所以，正如 Item40 所阐述的，不需要将该标记设置为 std::atomic；简单的 bool 就可以了。然后检测任务看起来像这样：
  
  ```c++
  std::condition_variable cv;			// as before
  std::mutex m;
  
  bool flag(false);					// not std::atomic
  ...									// detect event
  {
      std::lock_guard<std::mutex> g(m);	// lock m via g'ctor
      flag = true;					// tell reacting task (part 1)
  }									// unlock m via g's dtor
  
  cv.notify_one();					// tell reacting task (part 2)
  ```
  
  响应任务：
  
  ```c++
  ...									// prepare to react
      
  {										// as before
      std::unique_lock<std::mutex> lk(m);	// as before
      cv.wait(lk,
              []{return flag;}			// use lambda to avoid spurious wakeups
             );
      
      ...									// react to event(m is locked)
  }
  ...										// continue reacting (m now unlocked)
  ```
  
  ​		这种方法避免了我们讨论过的问题。不管响应任务是否在检查任务通知之前等等，它都可以工作，它可以在虚假唤醒的情况下工作，而且不存在轮询。但是还是看着不舒服，因为检测任务和响应任务以一种奇怪的方式通信。通知条件变量告诉响应任务，它等待的事件可能已经发生，但响应任务必须检查该标记以确认。设置标志告诉响应人物事件已经确定发生，但是检测任务仍然必须通知条件变量，以便响应任务将唤醒并检查该标记。这种方法有效但似乎并没有完全解决问题。
  
  ​		另一种方法是通过让响应任务等待检测任务的 future 来避免条件变量、互斥锁和标记。这似乎是个很奇怪的想法。毕竟，Item38 解释了 future 表示的是从被调用者到（通常是异步的）调用者的通信通道，这里的检测任务和响应任务之间没有被调用者-调用者关系。但是，Item38 也指出了，发送端为 std::promise 接收端为 future 的通信通道不仅仅用于被调用者的通信。这种通信通道可以用于你需要将信息从程序中一个位置传输到另一个位置的任何情况。在本例中，我们将使用它将信息从检测任务传递到响应任务，我们将传递的信息是关注的事件是否发生。
  
  ​		设计很简单。检测任务持有一个 std::promise 对象（即，通信通道的写入端），响应任务有一个对应的 future 。当检测任务发现事件发生时，它设置 std::promise （即写入通信通道）。同时，响应任务等待它的 future。等待会阻塞响应任务，直到 std::promise 被设置。
  
  ​		现在，std::promise 和 future（即 std::future 和 std::shared_future）都是需要类型参数的模板。该参数表示需要通过通信通道传输的数据类型。但是，在我们的例子中，没有要传递的数据。这个响应任务唯一感兴趣的是它的 future 是否被设置。std::promise 和 future 模板需要的是一个表示没有数据的类型。这个类型就是 void 。因此，检测任务将使用 std::promise<void>，而响应任务将使用 std::future<void> 或 std::shared_future<void>。当检测任务发现事件发生时会设置它的 std::promise ，并且响应任务将会在它的 future 上 wait。即使响应任务不会从检测任务收到数据，通信通道也会允许响应任务知道检测任务什么时候通过 std::promise 的 set_value "写入"了它的 void 数据。
  
  ```c++
  std::promise<void> p;
  // detect task
  ...						// detect event
  	p.set_value();		// tell reacting task
  ...
      
  // react task
  ...							// prepare to react
      p.get_future().wait();	// wait on future correcsponding to p
  ...							// react to event
  ```
  
  ​		与使用标记的方法一样，这种设计不需要互斥锁，无论检测任务在响应任务等待之前是否设置了它的 std::promise 都可以工作，并且不会存在虚假唤醒的问题（只有条件变量容易受到这个问题的影响）。与基于条件变量的方法一样，响应任务在进行等待调用后确实被阻塞，因此它等待是不会消耗系统资源。
  
  ​		当然，基于 future 的方法可以规避一些问题，但是还有其他的担忧。例如，Item38 阐述了 std::promise 和 future 之间是共享状态，共享状态通常是动态分配的。因此，这种设计会导致基于堆的内存分配和销毁消耗。
  
  ​		或许更重要的是， std::promise 只会被使用一次。std::promise 和 future 之间的通信通道是一次性的：它不能被重复使用。这与基于条件变量和基于标记的设计由明显的不同，两者都可以用于多次通信。（条件变量可以被反复通知，并且标记可以被清除和再次设置）。
  
  ​		一次性限制并不像你想象中那样严格。假设你想创建一个处于挂起的系统线程。也就是说在线程准备运行之前你想获取线程创建相关的信息，正常的线程创建已经无法满足。或者你可能像创建一个挂起的线程，以便你可以让它在运行之前对其进行配置。这些配置可能包括设置优先级或者核心关联性等内容。C++ 并发API 没有提供操作这些的方法，但 **std::thread 对象提供了 native_handle 成员函数，其结果旨在让你访问平台的底层线程 API（通常是 POSIX 线程或Windows 线程）**。较低层级别的 API 通常可以配置线程特征，例如优先级和关联性。
  
  ​		假设你只想挂起一个线程一次（创建之后，但在它运行它的线程函数之前），使用 void future 的设计是很合理的选择。这也是该技术的本质：
  
  ```c++
  std::promise<void> p;
  void react();				// funct for reacting task
  void detect()				// func for detecting task
  {
      std::thread t([]		// create thread
                    {
                        p.get_future().wait();	// suspend t untill future is set
                        react();
                    }
      			);
      ...						// here,t is suspend prior to call to react
          
  	p.set_value();			// unsuspend t (and thus call react)
      ...						// do additional work
  	t.join();				// make t unjoinable (see Item37)
  }
  ```
  
  ​		因为在 detect 结束时保证 t 是不可链接的状态很重要，所以使用像 Item37 中 ThreadRAII 这样的 RAII 类很可取。想象中的代码应该如此：
  
  ```c++
  void detect()
  {
      ThreadRAII tr(								// use RAII object
          	std::thread([]
              	        {
                          	p.get_future().wait();
                              react();
                  	    }),
          	ThreadRAII::DtorAction::join		// risky!(see below)
  		);
      
      ...						//  thread inside tr is suspended here
          
  	p.set_value();			// unsuspend thread inside tr
      ...
  }
  ```
  
  ​		代码看上去很安全。问题是，如果在第一个 "..." 区域（带有 "线程在 tr 内部是被挂起的" 注释的区域），一个异常被触发，set_value 将永远不会被 p 调用。这就意味运行 lambda 的线程永远不会结束，这就是问题，因为 RAII 对象 tr 被设置为在 tr 的析构函数中对该线程执行 join。也就是说，如果代码第一个 "..." 区域出现异常，这个函数将会挂起，因为 tr 的析构函数有缘不会完成。
  
  ​		有很多方法可以解决这个问题，这里主要是向读者展示开发中可能遇到的问题。接下来展示如何将源代码（即，不适用 ThreadRAII）扩展为挂起和取消挂机，而且是多个响应任务时。简单说一下，响应代码中会使用到 std::shared_future 而不是 std::future。一旦你知道了 std::future 的共享成员函数将其共享状态的所有权转移到共享函数生成的 std::future 对象，那编程将变得非常简单。唯一微妙之处在于，每个响应线程都需要有自己的 std::shared_future 副本来引用共享状态，因此响应线程中运行的 lambda 是按值获取 std::shared_future 的：
  
  ```c++
  std::promise<void> p;			// as before
  void detect()					// now for multiple reacting tasks
  {
      auto sf = p.get_future().share();	// sf's type is std::shared_future<void>
      
      std::vector<std::thread> vt;		// container for reacting threads
      
      for(int i=0; i< threadToRun; ++i)
      {
          vt.emplace_back(				// wait on local copy of sf;see Item42
              [sf]{sf.wait(); react();}	// for info on emplace_back
          		);
      }
      
      ...							// detect hangs if this "..." code throw!
  	p.set_value();
      ...
  	for(auto& t:vt)				// make all threads unjoinable;
      {							// see Item2 for info on "auto&"
          t.join();
      }
  }
  ```
  
  使用 future 的设计可达到这种效果，所以这就是为什么应该考虑将其用于一次性事件通信。
  
     **小结**
  
  * 对于简单的时间通信，基于条件变量的设计需要使用多余的互斥锁，对检测和响应任务的相对进度施加限制，并要求响应任务验证事件已经发生了。
  * 基于使用标记的设计避免了这个问题，但是它是基于轮询不会阻塞，导致消耗硬件资源
  * 使用条件变量和标记结合的设计会使得通信机制变得呆板
  * 使用 std::promise 和 std::future 可以避免这些问题，但是这方法会使用堆内存来共享状态，并且仅限于一次性通信
  
  ---
  
  #### *item 40. 并发使用 std::atomic, 特殊内存使用 volatile*
  
  ​		可怜的 volatile。它本不该出现在本章节中，因为它跟并发无关。但是在其他编程语言中（例如，java 和 C#），volatile 对于这类编程很有用，甚至在 C++ 中，一些编译器已经为 volatile 注入了语义，使其适用于并发软件（但仅当使用这些编译器时）。如果是为了消除围绕着它的困惑，那么有必要在本章中讨论 volatile。
  
  ​		我们有时会混淆 volatile 与 C++ 特性 std::atomic 的区别。这个模板的实例化（例如，std::atomic<int>，std::atomic<bool>，std::atomic<Widget>等等）提供了保证被其他线程的原子操作。一旦构建了一个 std::atomic 对象，对它的操作就像在临界区添加了互斥锁一样，但是这些操作通常是使用特殊的机器指令来实现的，比使用会吃所更有效。
  
  ​		考虑以下使用 std::atomic 的代码：
  
  ```c++
  std::atomic<int> ai(0);		// initialize ai to o
  ai = 10;					// atomiclly set ai to 10
  std::cout << ai;			// atomically read ai's value
  ++ai;						// atomically increment ai to 11
  --ai;						// atomically decrement ai to 10
  ```
  
  ​		在执行这些语句时，其他线程读取 ai 的值只可能是0、10或11，不存在其他值（当然，假设这个线程是唯一修改 ai 的线程）。
  
  ​		这个例子需要注意两点。首先，在 "std::cout << ai" 语句中，ai 是一个 std::atomic 确保了 ai 读取时是原子性的。但是不能保证这个语句是原子性操作。在读取 ai 的值和调用操作符 << 将其写入标准输出之间，另一个线程可能已经修改了 ai 的值。这对语句的行为没有影响，因为操作符 << 对 int 类型时使用按值传递（输出值是从 ai 读取的值），但更重要的是要明白该语句中**对 ai 的读取是原子操作**。
  
  ​		该示例的第二个需要注意的地方是最后两个语句的行为----ai 的递增和递减。这些都是读-改写（RMW）操作，但都是以原子方式执行。这是 std::atomic 类型最好的特征之一：**一旦构造了 std::atomic 对象，其上的所有成员函数，包括那些包含 RMW 操作的函数，都保证被其他线程视为原子函数**。
  
  ​		相反，在对线程上下文中，使用相应 volatile 的代码实际上什么都不能保证：
  
  ```c++
  volatile in vi(0);				// initialize vi to 0
  vi = 10;						// set vi to 10
  std::cout << vi;				// read vi's value
  ++vi;							// increment vi to 11
  --vi;							// decrement vi to 10
  ```
  
  ​		这段代码的执行过程中，如果其他线程正在读取 vi 的值，它们可能会读取到任何内容，例如-12，68，4990727----任何东西！这样的代码会有未定义的行为，因为这些语句修改了 vi，所以如果相同时间其他线程读取 vi，就会存在对既不是 std::atomic 也没有互斥锁保护的内存被同时读写，这就是数据竞争。
  
  ​		参考一个具体示例观察 std::atomic 和 volatile 在多线程中行为有何不同。假设有一个简单计数器，该计数器由多线程递增。我们初始化计数器为0：
  
  ```c++
  std::atomic<int> ac(0);			// "atomic counter"
  volatile int vc(0);				// "volatile counter"
  ```
  
  然后计数器将在两个同时运行的线程中各递增一次：
  
  ######### Thread1 ########              ######### THread2 #########
  
  ​					++ac;																++ac;
  
  ​					++vc;																++vc;
  
  ​		当两个线程都完成时，ac 的值（即 std::atomic 的值）必定是2，因为每个增量都是不可分割的原子操作。另一个方面，vc 的值不能保证是 2，因为它的增量不能以原子的方式发生。每次增量操作包括：读取 vc 的值，增加被读取的值，然后将结果写回 vc 中。但是这三个操作不能保证对易失性对象进行原子性操作，所以 vc 中两个增量的组成部分可能会交错，如下所示：
  
  1、线程1 读取vc的值，也就是0
  
  2、线程2 读取vc的值，仍然是0
  
  3、线程1 将读取的0增量为1，然后将该值写入vc
  
  4、线程2 将读取的0增量为1，然后将该值写入vc
  
  因此vc的值最终为1，尽管它增加了两次。
  
  ​		这不是唯一可能的结果。vc 的最终值通常不可预测的，因为 vc 存在数据竞争，而标准规定数据竞争会导致未定义的行为，这意味着编译器可能会生成代码去做一些事情。当然，编译器不会利用这个漏洞进行恶意操作。相反，它们执行的优化在没有数据竞争的程序中是有效的，而这些优化在存在竞争的程序中产生意想不到或不可预测的行为。
  
  ​		使用 RMW 操作的情形，不是 std::atomic 会成功，volatile 会失败的唯一情况。假设一个任务计算了另一个任务需要的重要值。当第一个任务计算出该值时，它必须将该值通知给第二个任务。Item39 阐述了第一个任务将所需值得可用性传递给第二个任务得一种方法是使用 std::atomic<bool> 标记。计算该值得任务代码如下：
  
  ```c++
  std::atomic<bool> valAvailable(false);
  auto imptValue = computeImportantValue();		// compute value
  valAvailable = ture;							// te;; other task it's available
  ```
  
  ​		当我们阅读这段代码时，我们知道对 imptValue 的赋值发生在对 valAvailable 的赋值之前，但编译器看到的都是独立的变量的赋值。一般来说，编译器可以对这些不相关的赋值进行重新排序。也就是说，给定这个赋值序列（其中a、b、x和y 对应于自变量），
  
  ```c++
  a=b;
  x=y;
  ```
  
  编译器可能会重新排序：
  
  ```c++
  x=y;
  a=b;
  ```
  
  ​		即使编译器不重新排序，底层硬件也可能会这样做（或可能让其他内核觉得他已经这么做了），因为这样有时会让代码运行得更快。
  
  ​		然而，使用 std::atomic 对代码的重新排序施加了限制，其中一个限制是源代码中的 std::atomic 变量之前的代码不能在之后发生（或在其他内核中出现）。这就意味着在我们的代码中，
  
  ```c++
  auto imptValue = computeImportantValue();		// compute value
  valAvailable = true;							// tell other task it's available
  ```
  
  编译器不仅必须保留对 imptValue 和 valAvailable 的赋值顺序，还必须生成代码以确保底层硬件也是如此。因此，将 valAvailable 声明为 std::atomic 可以确保关键排序要求----imptValue 必须被所有线程看到，以便在 valAvailable 做更改之前被更改。
  
  ​		将 valAvailable 声明为 volatile 并不会带来相同强制排序限制：
  
  ```c++
  volatile bool valAvailable(false);
  auto imptValue = computeImportantValue();
  valAvailable = true;	// other threads might see this assigment before
  						// the one to imptrValue!
  ```
  
  在这里，编译器可能会翻转对 imptValue 和 valAvailable 的复制顺序，即使它们没有这么做，它们也可能无法生成机器码，从而阻止底层硬件使其他内核上的代码在 imptValue 之前看到 valAvailable 的变化。
  
  ​		这两个问题---不能保证操作原子性和对代码重新排序的情况---解释了为什么 volatile 对于并发编程没有用处，但没有解释它的用处。简而言之，它是告诉编译器，它们正在处理"非普通"的内存。
  
  ​		"普通"内存的特点是，如果你写入一个值到内存位置，这个值会一致保留在那里，直到有东西覆盖它。如果我们有一个普通的 int 变量，
  
  ```c++
  int x;
  ```
  
  编译器观察到下面一系列的操作，
  
  ```c++
  auto y = x;		// read x
  y = x;			// read x again
  ```
  
  编译器可以通过消除对 y 的赋值来优化生成代码，因为这与 y 的初始化是多余的。
  
  普通内存还有一个特点，就是如果你把一个值写到某个内存位置，但从来没有动过它，然后再向那个内存位置写入，第一次写入会被消除掉，因为它从来没有被使用过。如下两个语句，
  
  ```c++
  x=10;			// write x
  x=20;			// write x again
  ```
  
  编译器会消除第一个，也就是说如果我们的源码中存在类似这样的代码，
  
  ```c++
  auto y = x;		// read x
  y = x;			// read x again
  x = 10;			// write x
  x = 20;			// write x again
  ```
  
  编译器会这样处理:
  
  ```c++
  auto y = x;		// read x
  x = 20;			// write x
  ```
  
  ​		上面这种冗余读和冗余写的代码，在技术上称为冗余负载和死存储。编译器遇到这些看似合理的源码和执行模板实例、内联和各种常见的需要重新排序的情况时，编译器通常会去除冗余的负载和死存储。
  
  ​		此类优化仅"普通"内存运行时才有效，"特殊"内存则不会。可能最常见的"特殊"内存就是用于内存映射 I/O 的内存。此类内存实际上是与外围设备通信，例如外部传递器或显示器、打印机、网络端口等，而不是进行读写的"普通"内存（即RAM）。这种情况下，再观察一下冗余读取的代码：
  
  ```c++
  auto y = x;			// read x
  y = x;				// read x again
  ```
  
  ​		例如，如果 x 对应于温度传感器报告的值，那么 x 的第二次读取就不是冗余的，因为第一次和第二次读取之间，温度可能发生了变化。
  
  ​		冗余写操作的情况类似，例如如下代码：
  
  ```c++
  x = 10;				// write x
  x = 20;				// write x again
  ```
  
  ​		如果 x 对应于无线电发射机的控制端口，则代码可能是正在向无线电发出命令，而值 10 和 20 分别对应不同的命令。优化第一次分配将改变发送到无线电的命令顺序。
  
  ​		volatile 是告诉编译器我们正在处理特殊内存。告诉编译器"不要对这个内存上的操作执行任何优化"。因此，如果 x 对应于特殊的内存，它应该声明为 volatile:
  
  ```c++
  volatile int x;
  ```
  
  ​		考虑对源代码序列的影响：
  
  ```c++
  auto y = x;		// read x
  y = x;			// read x again(can't be optimized away)
  x = 10;			// write x(can't be optimized away)
  x = 20;			// write x again
  ```
  
  ​		这正是我们想要的，如果 x 是内存映射的（或者已经映射到跨进程共享的内存位置等）。有个问题就是代码中，y 的类型是什么：int 还是 volatile int？
  
  ​		在处理特殊内存时必须保留看似冗余的负载和死存储，这是为啥不使用 std::atomic 的原因。如果使用 volatile 情况大不相同，如果暂时忽略这一点而专注编译器执行的操作，理论上讲，编译器可能会如下操作：
  
  ```c++
  std::atomic<int> x;
  auto y = x;			// conceptually read x(see below)
  y = x;				// conceptually read x again(see below)
  
  x = 10;				// write x
  x = 20;				// write x again
  
  // after optimise like this
  auto y = x;			// conceptually read x(see below)
  x = 20;				// write x
  ```
  
  ​		对于特殊内存，这显然是不能接受的行为。巧在是，当 x 是 std::atomic 时，这两个语句都不会编译：
  
  ```c++
  auto y = x;	// error!
  y = x;		// error!
  ```
  
  ​		这是因为 std::atomic 的复制操作被删除了（参看Item11）。也是有原因的，考虑如果用 x 初始化 y 成功编译后会发生什么？因为 x 是 std::atomic ，所以 y 的类型也被推导为 std::atomic（参看Item2）。之前说过 std::atomic 最好的一点是它们的所有造作都是原子的，但是为了将 x 复制给 y，编译器必须在原子操作中生成读取 x 和 写入 y 的代码。硬件通常不能做到这一点，所以 std::atomic 类型不支持复制构造。由于同样的原因，复制赋值也被删除，这就是为什么从 x 到 y 的赋值不能编译通过的原因。（std::atomic 中没有显示声明 move 操作，因此，根据 Item17 中描述的编译器生成的特殊函数规则，std::atomic 既不提供 move 构造也不提供 move 赋值）。
  
  ​		可以将 x 的值放入 y 中，但是需要使用 std::atomic 的成员函数来加载和存储。load 成员函数以原子方式读取 std::atomic 的值，而 store 成员函数以原子方式写入 std::atomic 的值。用 x 初始化 y，然后把 x 的值放在 y 中，代码必须这样写：
  
  ```c++
  std::atomic<int> y(x.load());		// read x
  y.store(x.load());					// read x again
  ```
  
  ​		这样可以编译，但读取 x（通过 x.load()）初始化或存储到 y 都是单独的调用，也没有理由让一条语句作为一个整体成为一个原子操作执行。
  
  ​		鉴于这个代码，编译器可以通过将 x 的值存储在寄存器中而不是读取它两次来"优化"它：
  
  ```c++
  register = x.load();			// read x into register
  std::atomic<int> y(register);	// init y with register value
  y.store(register);				// store register value into y
  ```
  
  ​		如你所见，只从 x 读取了一次，这是在处理特殊内存时必须避免的优化。（不允许对 volatile 变量进行优化）。
  
  ​		因此，情况应该很清楚：
  
  * std::atomic 可用于并发编程，但不适用于访问特殊内存。
  * volatile 可用于访问特殊内存，但不适用于并发编程。
  
  因为 std::atomic 和 volatile 有不同的用途，它们甚至可以一起使用：
  
  ```c++
  volatile std::atomic<int> vai;		// operations on vai are atomic and can't be
  									// optimized away
  ```
  
  如果 vai 对应于多线程并发访问的内存映射 I/O 会非常有用。
  
  ​		最后需要注意的是，一些开发人员更喜欢使用 std::atomic 的 load  和 store 成员函数，即使是必要的情况下，因为它在源码中明确表示所涉及的变量不是"正常的"。这么做其实是有道理的。访问 std::atomic 通常比访问非 std::atomic 慢，因为之前提到使用 std::atomic 会阻止编译器进行默写代码的重排等优化，反之则会优化。因此，调用 std::atomic 的 load 和 store 可以帮助识别潜在的可扩展性阻塞点。从正确性角度来看，没看到对存在变量上的调用可能意味着它在向其他线程传递消息（例如，指示数据可用性的标志），可能意味着该变量应该被声明为 std::atomic 的时候没有被声明为 std::atomic。
  
  ​		这或许是一个代码风格的问题，但是选择 std::atomic 和 volatile 会有截然不同的结果。
  
     **小结**
  
  ---
  
  * std::atomic 用于在不使用互斥锁的情况下从多线程访问数据。它是编写并发软件的工具。
  * volatile 是针对读和写不应该被优化的内存，它是用来处理特殊内存的。
