# C.2 Managing threads

内容提要：（这里就不翻译了，感觉怎么翻译都没有原文来的简洁，下同，没翻译的是觉得原文表意更佳）

+ starting threads, and various ways of specifying code to run on a new thread
+ waiting for a thread to finish vs leaving it to run
+ uniquely identifying threads



本章的操作有：

launching a thread -> waiting for it to finish / running it in the background（2.1）

passing additional parameters to the thread function **when it's launched**

how to **transfer ownership** of a thread from one `std::thread`object to another

choosing the number of threads to use 

identifying particular threads 



[toc]

## 2.1 Basic thread management

每个C++程序至少有一个线程，那就是我们的`main()`。

我们可以启动更多的线程，并且用另一个函数作为它的入口点(entry point)。

这些我们启动的线程，以及我们最开始的`main()`线程就并行地运行啦。

同我们的程序在`main()`中return就会exit一样，当我们启动的其他线程return，这个线程就exit了。


## 2.4 Choosing the number of  threads at runtimes

一个比较有用的库函数是`std::thread::hardware_concurrency()`，这个函数能够返回一个程序能真正并行的线程数，它可能是服务器的CPU的逻辑核数。

下面代码展示了一个并行版本的`std::accumulate`，它将任务分成了几个线程，每个线程都设定了最小的处理元素数目，以防止开启太多线程造成的开销。注意，下面的代码我们假设不会抛出异常，尽管有可能会抛出线程（比如说，一个`std::thread`的构造函数不能开始一个新的执行线程），代码如下：

```c++
// A naive parallel version of std::accumulate
template<typename Iterator, typename T>
struct accumulate_block
{
    void operator()(Iterator first, Iterator last, T& result)
    {
        result = std::accumulate(first, last, result);
    }
}

template<typename Iteratorr, typename T>
T parallel_accumulate(Iterator first, Iterator last, T init)
{
    unsigned long const length = std::distance(first, last);
    
    if(!length)																		  // 1
        return init;
    
    unsigned long const min_per_thread = 25;
    unsigned long const max_threads = (length + min_per_thread -1)/min_per_thread;			// 2
    
    unsigned long const hardware_threads = 
        std::thread::hardware_concurrency();
    
    unsigned long const num_threads =												// 3
        std::min(hardware_threads!=0 ? hardware_treads : 2, max_threads );
    
    unsigned long const block_size = lenght/num_threads;								  // 4
    
    std::vector<T> results(num_threads);
    std::vector<std::thread> threads(num_threads-1);									// 5
    
    Iterator block_start = first;
    for(unsigned long i = 0; i<(num_threads-1); ++i)
    {
        Iterator block_end=block_start;
        std::advance(block_end, block_size);											// 6
        threads[i] = std::thread(accumulate_block<Iterator,T>(), block_start, block_end, std::ref(results[i]));		// 7
        block_start = block_end;														// 8
    }
    accumulate_block<Iterator,T>()(
    	block_start, last, results[num_threads-1]);										// 9
    std::for_each(threads.begin(), threads.end(), std::men_fn(&std::thread::join));			// 10
    
    return std::accumulate(results.begin(), results.end(), init);						// 11
}
```

代码1 处，如果输入的为空，那么就返回初始值`init`。否则，就至少有一个元素。

在代码2处，可以将元素分给不同的线程去处理，设定了每个线程要处理的最少元素个数，那我们给每个线程**最少**分这些个数的元素处理，就一共**最多**需要(length + min_per_thread -1)/min_per_thread个线程（PS:有一个线程可能处理的元素数不足min_per_thread，其他的线程都处理min_per_thread个元素，所以计算公式是这样）。这样可以避免处理很少的数据的时候却开启了逻辑CPU数量个线程（这导致了额外的非必要开销，根本不需要那么多线程）。

代码3处，我们选择我们需要的线程数量，它是我们机器可以支持的最大真并行线程数（逻辑CPU数）和我们所需的最大线程数之间的**最小者**。我们不希望在面对计算型任务的时候，使用超过逻辑CPU数量个数的线程，因为那会导致频繁的线程切换，从而导致不必要的**上下文切换**开销。

在代码4处，我们计算了每个线程要处理的元素的个数。需要注意，这个地方使用了int间的除法，会有舍入误差，因此在后面我们用一个单独的线程去处理那些**元素被均匀地派发给其他每个线程后还剩余的元素**。（这里需要重新计算的原因是，我们上一步才确定了真正要用的线程的个数）。

在代码5处，需要注意，我们要开启num_threads-1个线程，因为我们已经有一个线程了（也就是当前这个在执行的线程，后面可以看出，当前这个线程用来处理剩余元素）。

只需要一个简单的for loop去开启这些线程，在代码6处，`std::advance(block_end, block_size)`的作用是，将迭代器`block_end`向后移动`block_size`个位置，也就是将迭代器`block_end`移动到了当前要处理的块的**尾后**（同时这个位置也是块要处理的开始）。

然后代码7处，新开启了一个线程对这个块进行了计算（PS：给这个线程对象传的第一个参数是`accumulate_block<Iterator,T>()`，是一个`accumulate_block`临时对象，是个右值，然后这个对象有个`operator()`，可以call）。另外还用了我们前面说过的引用 `std::ref(results[i])`，直接在`results[]`上更新结果。

代码8，下一个线程的开始的迭代器位置是当前线程的尾后迭代器。

在代码9处，我们**在当前线程**处理剩下的所有的没有被计算的元素。我们知道结束的地方是`last`。

当算完最后这个块的时候，我们可以通过`std::for_each`来等待所有线程完成（代码10）。

最后将每个线程计算的结果再通过`std::accumulate`累加起来。



**需要注意的是，我们没有办法直接从一个线程里直接返回一个值**，所以，我们需要传递一个引用，来进行数据的交互。

另外，第四章将要讲到的**futures**可以提供一种从线程返回值的方法。



在上面我们讲到这个例子中，在线程开始的时候，我们就把线程所需要的所有的东西都传进去了，包括它用来返回结果的那个`result`。

但是事实并不总是如此，有时候，我们需要在执行的过程中识别线程。我们可以像代码2.7那样传一个`i`作为标识符来标识线程，**但是**，如果一个函数在很多层的调用栈里需要一个标识符，并且任何线程都能call到它，那么用刚刚说的这种方式就不是很方便了，所以接下来要介绍的东西就是C++线程库里为我们提供的一种对给个线程的独一无二的标识符。

## 2.5 Indentifying threads

线程标识符的类型是`std::thread::id`，并且可以通过两种方式得到：

1. 一个线程的标识符，可以通过call这个线程相关联的那个`std::thread`对象的成员函数`get_id()`来得到。如果一个`std::thread`对象没有相关联的执行线程，那么在这个对象上call `get_id()`会返回一个默认构造的`std::thread::id`对象，这代表“没有任何线程”。
2. 当前线程的id可以通过call `std::this_thread::get_id()`来获得。

它们都包含在在\<thread\>头文件里。

`std::thread::id`类型的对象可以拷贝和比较，否则它们作为标识符的意义就不大了。

如果两个线程标识符相等，那么意味着它们代表同一个线程，或者都表示，没有任何执行线程。

如果两个标识符不同，那么意味着它们不是同一个线程，或者其中一个没有任何执行线程，而另一个有一个执行线程。

线程库不限制我们对于线程标识符是否相同的检查，并且线程标识符对象支持比较操作运算符，这就意味着，我们可以用线程标识符作为key来与容器关联，或者进行排序，或者比较等等，来方便我们编程。

`std::thread::id`实例常用来检查线程是否需要执行一些操作。

比如，在前面的代码2.8中，开启其他线程的原始线程的行为可能会有变化，原始线程在启动其他线程之前需要通过`std::this_thread::get_id()`先保存那个要启动的线程的id，然后在运行的时候通过检查线程id来区分线程是做哪个工作，如下所示

```c++
std::thread::id master_thread;
void some_core_patr_of_algorithm()
{
	if(std::this_thread::get_id()==master_thread)
    {
        do_master_thread_work();
    }
    do_common_work();
}
```



## 2.6 Summary

本章介绍了C++ 线程库 对线程提供的基本操作的支持：启动线程，等待线程运行完毕，不等线程运行完（因为将其放到后台去了）。

同时我们也介绍了在线程启动的时候，如何将对执行线程启动的函数进行传参，如何改变执行线程的拥有权，如何创建一堆线程并给他们分配任务以加速程序，最后还介绍了标识线程的方法，标识线程可以使我们有方法让特定的线程执行特定的功能或者处理特殊的数据。

虽然在代码2.8中我们介绍了让不同线程独立处理不同数据的方法，但是有时我们需要在线程运行时**共享数据**。

第三章主要围绕对线程间直接共享数据的讨论展开。

第四章则讨论了不通过共享数据 来实现同步的操作的一般性问题。



那么我们该如何启动一个`std::thread`线程呢？



### 2.1.1 Launching a thread

如同第一章中介绍的那个最简单的例子，通过构建一个`std::thread`对象就可以启动一个线程。

最简单的，我们启动的这个线程可以是void-returning并且不接受参数，线程在启动后，运行至return停止。

复杂的，可以对这个新的线程传参，并且在它运行的时候，可以通过系统的一些通信机制进行一些操作，然后由这些信号来决定这个线程是否结束。

不管怎么说，启动一个新的线程，总要从构建`std::thread`对象开始：

```C++
void do_some_work();
std::thread my_thread(do_some_work);
```



只要是callable的类型，都可以传给`std::thread`用来构造一个线程对象。

如果一个类的实例实现了`operator ()`，也就是实现了function call operator，那么就可以把这个对象传给`std::thread`用来构造一个线程对象：

```c++
class background_task
{
    public:
    void operator()() const
    {
        do_something();
        do_something_else();
    }
};

background_task f;
std::thread my_thread(f);
```

这种情况下，这个提供function call的对象会被**拷贝**到新的线程的存储空间内，并且在运行的时候会用这个拷贝过来的对象，而不是原本的那个对象（这个过程中应该会执行原来那个对象的拷贝构造吧？）。

**值得注意的是**，在传递这样一个function object（就是指实现了callable 的class 的实例）的时候，需要传递一个正确的参数，也就是需要传进去callable的function 或者实现了operator function call的object。如果不注意的话，C++的语法解释就不能为我们正常的创建一个线程对象。



举个例子：

```c++
std::thread my_thread(background_task());
```

注意上面这行代码和前面的粒子中的最后一行的区别。上面这行代码**声明了一个名为`my_thread`的函数**，这个函数有一个参数（这个参数是一个类型指针，这个指针指向一个函数，这个函数没有参数并返回一个background_task 对象，PS：好吧，这有点绕口，我们一点一点来分析这里，首先我们考虑一个情况，用一个类名来call，结果就是会调用这个类的default constructor，构造一个对象，那么这里也一样，用类名来call，就会返回一个对象，而在进行语句声明的时候，可以不带参数名，但是一定要有参数的类型声明，那么这里的 background_task()就是这个参数的类型名，这个类型名说明我们需要一个函数指针，指向的函数不接受参数，并且调用它之后可以返回一个background_task对象，emmm差不多能明白了吧），并且返回一个`std::thread`对象。**而不是启动一个新线程**。

这当然不是我们想要的。

那么避免这种**我们不想要的“错误的 声明 形式“**，可以怎么做呢？三种方法：

1. 像前面的那样，使用实例化的对象传参，防止错误解析，如下：

   ```c++
   background_task f;
   std::thread my_thread(f);
   ```

2. 用一个括号括起来，防止错误的解析为函数声明。如下：

   ```c++
   std::thread my_thread((background_task()));
   ```

3. 用花括号，使用初始化列表（C++11特性，用初始化列表的初始化都是传对象）：

   ```c++
   std::thread my_thread{background_task()};
   ```



一种可以避免这种问题的 callable object使用 lambda表达式(C++11新特性）。前面的粒子用lambda表达式可以写成下面这个样子：

```c++
std::thread my_thread([](
	do_something();
    do_something_else();
));
```



前面介绍了如何启动一个线程，很简单:)

那么启动之后，我们需要显式决定是想等它完成(用后面介绍的 `join`)，或让它自己跑它自己的不管它(用后面介绍的`detach`)。

如果在`std::thread`对象运行结束之前还没决定如何处理它，那么，我们的程序就结束了。因为`std::thread`的析构函数会call `std::terminate()`。

因此我们需要确保我们的线程是 join 还是detach。

注意，我们只需要在这个线程对象destroyed之前确定是join 还是detach就好，这个线程可能会跑很久。如果我们detach它，那么这个线程对象destroyed之后**线程还可能会跑很久（取决于它本来需要跑多久）。**

如果我们选择detach这个线程对象，需要分析这个线程在运行完之前是否能够获取它原本该取得的数据。

针对上面这个问题，我们可能遇到的一个问题是，这个线程对象有一个对局部变量的引用或指向它的指针，然后这个局部变量所在的function exit了，那么这个局部变量就没了，但是线程还没有完成，那么这个线程就访问不了这个变量了。举个例子：

```c++
// Listing 2.1
struct func
{
    int& i;
    
    func(int& i_):i(i_){}
    
    void operator()()
    {
        for(unsigned j=0; j<10000; ++j)
        {
            do_something(i);		// 1                潜在的访问 dangling reference的风险（dangling reference 的意思是，一个对已经destroyed的对象的引用
        }
    }
};

void oops()
{
    int some_local_state=0;
    func my_func(some_local_state);
    std::thread my_thread(my_func);
    my_thread.detach();				//2    不等待线程my_thread完成
}									//3    oops函数已经exit了，但是my_thread线程可能还在运行
```

上面这个例子，在 语句2中，我们明确了 oops函数不等待my_thread的线程运行结束（这也是detach的含义，将从oops所在的线程所派生出来的新线程与这个oops所在的线程分离）。当oops函数exit时，my_thread线程可能还没结束，这时my_thread线程里对i的引用就会出现问题，因为oops的exit导致它所包含的local variable都没了（因为退栈）。

一个**解决**这种情况的常用**方法**是，让这个线程函数 **自己包含并且复制数据到线程里**，而不是共享数据。

通常来说，用一个**访问局部变量的函数**创建线程不是好的行为，除非，能够**保证**在函数exit之前能完成这个线程的任务。

当然，上面说的都是针对线程使用detach的情形。使用join就可以防止上面的问题了。





### 2.1.2 Waiting for a thread to complete

如果想等一个线程完成，可以在这个对象上call `join()`。	

`join()`是简单而且强力的。

当然如果想完成更多的事情，比如说想检查thread是否完成，或是只等线程一小段时间，那么需要用到像条件变量或者是future这样的机制，这在第四章中介绍。

另外，调用`join()`将会清除跟这个线程有关的存储(cleans up any storage associated with the thread)。并且这个线程对象就不再跟这个线程有关了(so the `std::thread` object is no longer associated with the now-finished thread)。（这里不太明白）。

it isn't associated with any thread

所以只能对线程对象call一次`join()`，一旦call了`join()`，这个`std::thread`对象就不再joinable了，而且`joinable()`会返回false。

（上面两行不太明白啥意思，）





### 2.1.3 Waiting in exceptional circumstances

如果想detach一个线程，通常可以在这个线程开始之后立即call `detach()`。

但如果想等待一个线程，需要**非常小心地**选择一个地方call`join()`。

这意味着，需要注意可能抛出的异常，这个异常需要在call `join()`前，在线程启动后抛出。（why？）

如果想在不发生异常的的情况下call `join()`，那么在遇到一个异常时，需要在抛出之前call `join()`，以避免accidental lifetime problems（这是个啥问题？）

举个例子：

```c++
// Listing 2.2
struct func;				// 2.1中定义了

void f()
{
    int some_local_state=0;
    func my_func(some_local_state);
    std::thread t(my_func);
    try
    {
        do_something_in_current_thread();
    }
    catch(...)
    {
        t.join();			// 1
        throw;
    }
    t.join();				// 2
}
```

上面的代码中用 try/catch 来确保在exit当前的function之前，无论是正常结束，还是因为有异常而结束，我们都让访问local variable的线程运行完了。

上面的代码有些麻烦。我们可以用RAII策略来实现相同的效果，如下所示：

```c++
// Listing 2.3
// using RAII to wait for a thread to complete
class thread_guard
{
    std::thread& t;
    
public:
    explicit thread_guard(std::thread& t_):			// 构造函数，用explicit修饰，禁止隐式类型转换
    	t(t_) {}								// 	成员初始化列表 初始化方式
    ~thread_guard()									// 析构函数
    {
        if(t.joinable())							// 1
        {
            t.join();								// 2
        } 
    }
    thread_guard(thread_guard const&)=delete;		// 3                 禁止复制构造					
    thread_guard& operator=(thread_guard const&)=delete	// 禁止 operator assignment
};

struct func;										// 见 2.1 定义

void f()
{
    int some_local_state = 0;
    func my_func(some_local_state);
    std::thread t(my_func);
    thread_guard g(t);
    
    do_something_in_current_thread();
}													// 4
```

当当前线程执行到位置4 时，局部对象会发生析构，顺序与构造相反，也就是说 `thread_guard`实例会先析构，在上面的代码中，我们可以看到，在`thread_guard` class的析构函数里，我们会调用`join()`(位置2所表示的那样)。**即使函数f的exit是因为`do_something_in_current_thread()`抛出一个异常，这个析构函数总会调用，这就保证了总会call `join()`。

需要注意的是在位置1的代码，我们需要判断这个`std::thread`对象是否能call`joinable()`，因为一个线程对象只能`join()`一次。

然后位置3处向编译器声明，不要为我们生成拷贝构造函数和operator assignment，原因是对这样一个对象copy或者assigning很危险，it might then outlive the scope of the thread it was joining。

**如果我们不需要等待这个线程完成，那么就可以通过`detach()`来避免这个 异常安全 类型的问题**。

调用`detach()`可以让当前线程和我们新产生的`std::thread`对象无关，并能保证当`std::thread`对象destroyed的时候不会call `std::terminate()`。

（有个地方不太理解，虽然前面有讲`std::thread`对象析构会call `std::terminate()`，但是为什么要这么做呢？）



### 2.1.4 Running threads in the background

`std::thread`对象 call `detach()`可以让这个对象与当前所在线程无关，并在后台运行(run in the background)。

一旦这么做了也就意味着我们不能再等待这个线程完成了。

一旦一个线程 detached 了，就不可能再获得一个与这个线程有关的线程对象了，所以也就不可能将它join了。

run 在background 的线程（也就是detach过的线程）的控制权和拥有权就交给了 C++ Runtime Library（也即是不在我们手里了，我们控制不了它了），当然这就保证了当这个线程exit的时候其所拥有的资源可以正常的收回。

detached threads 通常被称为**deamon threads**(守护线程？)，这来自于UNIX的deamon process的概念，这个process运行在后台，而且没有任何显式的用户接口。

这种线程通常运行很久。声明周期通常可能跟整个程序运行时间相同，经常用来执行一些后台任务，比如说监视文件系统，清理没用的缓存，优化数据结构等等。。。

像2.1.2节的例子那样，在一个`std::thread`对象上call `detach()`，当这个call完成，这个对象就与当前执行的线程无关了，并且就不再`joinable`了，也即是如果再对这个线程对象call `joinable()`就会return false。

想要对一个`std::thread`call`detach()`，首先得确定这个对象是可`detach()`的，**只有在call`joinable()`return true的线程对象上才能call `detach()`。


## 2.2 Passing arguments to a thread function

给callable object或者 function传参非常简单，就像给`std::thread`构造函数传参一样简单。如：

```c++
void edit_document(std::string const& filename);
// ... 其他操作
std::thread t(edit_document, new_name);
```

上面代码中，传到`std::thread`构造函数里的两个参数，第一个用来像之前一样作为 callable function构建线程对象，而第二个参数`new_name`实际上是传给这个callable function的参数。

但需要记住的是，默认情况下，参数是**复制**到**新创建的线程所拥有的的内存空间内的**。**即使，我们传入的这个函数的形参是一个引用也会这样**。

比如：

```c++
void f(int i, std::string const& s);
std::thread t(f, 3, "hello");
```

上面代码创建了一个与t关联的新的线程，这个新线程调用 `f(3, "hello")`。

**需要注意的是**，即使f需要一个`std::string`作为第二个参数，但是，实参`"helllo"`是作为char const\* 一个一个字符的传到新线程的内存中，并且**在新线程中完成从char const\*到`std::string`的转换过程的**。

这就导致，当参数是指向automatic variable的指针时，需要非常注意，如下代码：

```c++
void f(int i, std::string const&s);

void oops(int some_param)
{
	char buffer[1024];						// 1
    std::thread t(f, 3, buffer);			// 2
    t.detach();								
}
```

上面的代码中，1处的`buffer`是局部变量指针（数组名可以看成是指针），用来给新线程传参。

由于线程对象调用了`detach()`，而且前面分析了，参数是将内存中的数据一个一个拷贝到新线程内存空间中的，那么在oops exit的时候，这个参数还没复制完，就会导致 dangling pointer。

解决方案是，在传参之前，先转换成`std::string`对象：

```c++
void f(int i, std::string const&s);

void oops(int some_param)
{
	char buffer[1024];						
    std::thread t(f, 3, std::string(buffer));			// 使用 std::string 将char const*显式转换成str对象来避免dangling pointer
    t.detach();								
}
```



**所以上述问题在于，`std::thread`的构造函数在将参数拷贝到新线程内存空间中，是按照传进来的那个东西的样子，而非根据声明的形式进行拷贝的**。就像前面的例子，我们声明的是，函数f接受的是`std::string`对象，但是实际给到线程用来传参的是个C-style 字符串，它实际的类型是 char数组，所以在进行内存拷贝的时候，是按照char数组，将这些char一个一个进行拷贝的。



相反的情况也可能发生：对象已经拷贝完了，但是我们想要的是reference。

举个例子：

```c++
void update_data_for_widget(widget_id w, widget_data& data);			// 1

void oops_again(widget_id w)
{
    widget_data data;
    std::thread t(update_data_for_widget, w, data);						// 2
    display_status();
    t.join();
    process_widget_data(data);											// 3
}
```

函数`updata_data_for_widget`的第二个参数类型期望是个对`widget_data`对象的引用，因为我们想利用这个函数对这个对象进行修改，并在后续操作中继续使用这个对象。但是，`std::thread`的构造函数却不知道这回事。

当我们在运行2的时候，创建了一个新的线程对象，并且将data拷贝了一份给到这个线程所拥有的内存，并且引用与这个在新线程内存里的data关联在一起了（而不是与我们所设想的那样与`oops_again`函数中的那个data关联在一起。这个时候，就产生了临时对象。

所以，当这个新线程运行完后， 它所占有的内存的东西都destroy了，包括前面拷贝的那个data，这当然不会使`oops_again`函数中的data有任何变化。

那么语句3中的处理所传入的data就不是我们所预期的那个样子。

**解决方案**：我们需要通过`std::ref`将所需作为引用的参数进行打包。如下：

```c++
std::thread t(updata_data_for_widget, w, std::ref(data));		// 用std::ref()打包了data，这样就可以正确的传递一个reference
```

用了上面的这个`std::ref`，我们就可以正确的传递一个对所预期的data的引用，而不是对拷贝到新线程内存空间中的那个临时的data的引用。

`std::thread`的构造函数和`std::bind` 的操作使用相同的机制。

也就是说，我们可以传递一个成员函数指针作为传给`std::thread`构造函数的第一个参数，然后再传递一个合适的**对象指针**作为**前面传的那个参数的第一个参数**（这里的意思是，我们需要给那个member function传递一个用来调用它的对象，所以需要传递一个对象指针）。举例如下：

```c++
class X
{
public:
    void do_lengthy_work();
};

X my_x;
std::thread t(&X::do_lengthy_work, &my_x);	//1 成员函数指针 &X::do_lengthy_work， 对象指针 &my_x，用来调用成员函数
```

上面的代码1处会在新线程调用 `my_x.do_lengthy_work()`。

前面的分析可以看出，如果想调用一个成员函数，那么我们至少得传两个参数给这个`std::thread`构造函数：成员函数指针，预期调用这个成员函数的对象的指针。那么，从第三个参数开始，就可以作为这个成员函数的参数了（也就是线程构造函数的第三个参数是成员函数的第一个参数，以此类推）。



另外一个有趣的情况是，提供的参数是只能**移动(move)**但不能**复制(copy)**，也就是数据只能由一个对象传递给另一个对象，传递完后，原先的那个对象就变成空的了（也就是不再拥有传递出去的那份资源）。

可以使用移动构造函数和移动赋值语句来实现拥有权的转换。

这样的移动特性，使得这样的一个对象可以作为参数传递给新线程的函数，也可以作为返回。如果 源对象是临时对象（也就是不具名的），那么这个移动的语言是自动发生的，但是如果源对象是**具名**对象（也就是不为临时对象），那么转换必须显式调用`std::move()`。下面是一个使用`std::move`来转换动态创建的对象的拥有权给新创建的线程对象的例子：

```c++
void process_big_object(std::unique_ptr<big_object>);

std::unique_ptr<big_object> p(new big_object);	
p->prepare_data(42);
std::thread t(process_big_object, std::move(p));
```

通过在`std::thread`的构造函数里显式调用`std::move(p)`，`big_object`的拥有权首先被转移到新线程所创建的内存空间，进而转移到`process_big_object`。

虽然`std::thread`对象拥有一个动态分配的对象的行为不像unique_ptr（也就是说这个动态分配的对象是可复制的，而非仅能移动的），但是这些线程对象却实拥有一份有unique_ptr这样行为的资源，因为每个线程对象都要对一个线程的执行负责。

所以，线程对象的拥有权也是可以转变的，因为**线程对象是movable的，但是 非copyable**。这保证了，任何时候都**只有一个对象与特定线程的执行有关**。同时也使得程序员可以在对象间改变拥有权。



## 2.3 Transferring ownership of a thread

我们可能有这样的需求：写了一个函数，这个函数创建了一个线程并在后台执行，我们想把这个线程的控制权交给这个函数，而不是等待这个线程自己执行完；或者，我们想创建一个线程，然后把这个线程的控制权交给某个函数，让这个函数等待这个线程完成。

上面的两种情况都需要我们转移控制权。

`std::thread`是一类可以 移动(movable)，但是不能复制(copyable)的。这就意味着一个在执行的特定线程的控制权可以在`std::thread`实例之间移动(move)。

下面的例子展示了控制权移动的情况，创建了两个在执行的线程，并且在三个线程实例之间转移(move)控制权:

```c++
void some_function();
void some_other_function();
std::thread t1(some_function);			// 1
std::thread t2=std::move(t1);			// 2
t1=std::thread(some_other_function;)	// 3
std::thread t3;							// 4
t3 = std::move(t2);						// 5
t1 = std::move(t3);						// 6
```

上述代码分析：

首先，在代码1执行完后，一个与线程对象`t1`关联的新线程开始执行。

代码2：在`t2`构建完成后，通过调用`std::move()`显式地将控制权交给t2，此时，`t1`就不再拥有相关的在执行的线程了（因为线程是一种资源，move可以将右值进行转移，也就是将资源转交，原本的t1在资源被移走后，就不再拥有这份资源），同时，运行`some_function`的线程现在跟`t2`关联了，也就是`t2`现在拥有了这个线程资源。

代码3：首先，在`=`右边，先产生并运行了一个由临时对象持有的新线程(复制语句右端是线程构造函数构造了一个临时对象)，然后，**线程资源被move到了t1，这行代码执行完后，最终是t1接管这个线程资源**。因为，对于临时对象，移动(move)是**自动而且隐式的**。而不需要去call `std::move()`进行显式的移动(move)。

代码4：通过默认构造函数构造了一个`std::thread`对象t3，但是，**并没有任何与t3相关的在执行的线程(因为没有传进去callable的东西)**。

代码5：与t2关联的那个执行线程现在被move到了t3（也即是说现在t2没有相关的执行线程了），而且是通过显式的调用`std::move()`，因为t2是具名对象（也就是不是一个临时对象，它有名字，是个左值，不是右值），

代码6：在执行到代码6之前，t1现在持有运行着`some_ohter_function`函数的线程，t2没有关联的线程，t3持有运行着`some_function`的线程。代码6试图将这个运行着`some_function`的线程交由t1持有，但此时，t1已经持有一个线程了，这就会导致，**调用`std::terminate()`终止程序**。因为需要保证`std::thread`的析构函数的一致性（？）。这里的情况跟2.1.1讲的相似，在那里，我们必须**在这个线程运行结束销毁之前**显式的指出，我们是想阻塞**等待**一个线程执行完(调用`join()`)，还是将这个线程扔到后台让它自己执行完（调用`detach()`），这里在使用**移动复制语句**时也是一样的，我们**不能将一个线程“丢弃”**（这里 的意思是，一个线程对象只能持有一份线程资源，而t1本来有一份资源，如果再将t3的给它，它势必要丢弃一份）。

`std::thread`对于move的支持使得执行线程的拥有权可以从函数里传出来（也就是，可以通过调用一个写好的函数，传出一个有执行线程关联的线程对象）。如下面的例子：

```c++
std::thread f()
{
    void some_function();
    return std::thread(some_function);
}

std::thread g()
{
    void some_other_function(int);
    std::thread t(some_other_function, 42);
    return t;
}
```

（对代码前的那句话的更多的解释：这里我们定义了两个函数`f`和`g`，在它们俩的定义里，都有构造了一个线程对象并有一个与之关联的执行线程资源，这两个函数都返回了这个执行线程资源（通过移动语义support by move））

与前面同样的道理，如果想把一个执行线程的控制权**交到（或者说，传递）一个函数里**（前面的例子是从函数里交出执行线程资源的控制权），我们可以按值传递一个`std::thread`作为参数给函数，如下：

```c++
void f(thread::thread t);
void g()
{
	void some_function();
    f(std::thread(some_function));		// 1
    std::thread t(some_function);
    f(std::move(t));					// 2
}
```

（PS：上面的代码是原书中的，但个人觉得可能有问题，在代码1处，给f传了一个执行线程对象，在代码2又传了一个？一个函数执行的动作在运行期间已经确定了，PPS：可能传进去的这个执行线程对象并没有move给另一个线程对象，这样就不会有前面说的因为move而导致的drop问题。



`std::thread`对move的支持带来一个好处，我们可以重新建立2.3代码给出的`thread_guard`，并且让它真正获得线程的拥有权，而不是像之前一样只是执行线程对象的reference。这样就避免了因为原来的对执行线程对象的reference所可能带来的不好的结果（比如说dangling reference，也就是执行控制权已经离开一个scope，导致这个scope里的临时对象资源被回收，或者说被销毁，而还存在对这个临时资源的reference）。

同时，转移控制权也意味着，没有其他的将这个执行线程join或者detach的可能。

于是，用move，我们将本来的`thread_guard`重新写成了`scoped_thread`。实现如下：

```c++
class scope_thread
{
	std::thread t;				// 注意，这里不再是一个reference，我们通过move可以将执行线程的控制权转交到这个对象中
public:
	explicit scoped_thread(std::thread t_):				// 1
    	t(std::move(t_))
        {
            if(!t.joinable())							// 2
                throw std::logic_error("No thread");
        }
    ~scoped_thread()
    {
        t.join();										// 3
    }
    scoped_thread(scoped_thread const&) = delete;
    scoped_thread& operator=(scoped_thread const&) = delete;
};

struct func;					// 见2.1定义

void f()
{
    int some_local_state;
    scoped_thread t(std::thread(func(some_local_state)));	// 4  
    
    do_something_in_current_thread();
}														// 5
```

这里的实现跟2.3很像，但是新线程直接传递给`scoped_thread`，而不是先创建一个具名函数（如前面说的，这里的`std::thread(func(some_local_state))`是一个临时对象，而对临时对象的move是隐式而且自动的，不需要像2.3那样先创建具名对象，如果这里创建具名对象，那效率会低一些）。

如果最初始的线程（也就是函数`f`所在的线程）执行到了代码5处，也就是函数执行完了，控制权要从这个函数的scope里交出去了，那么这个scope里的所有对象都要destroy，对于非内置类型，会调用析构函数，那么就会执行代码3处的`join()`，来阻塞线程，直到执行完这个调用join的线程。在这里，我们不需要在析构函数里检测`joinable`，因为我们是通过move获得的这个执行线程的控制权，在构造函数的时候已经检测过`joinable`了，所以只有这个执行线程能调用`join`或`detach`，而我们能确定，我们没有调用过这两个方法，所以，不用判断了（PS：但是我有个问题，前面说过了，需要在这个执行线程析构前决定它是join还是detach，那么，在这个析构函数里才join，万一之前这个执行线程已经执行完了，那么不就terminate了？）



`std::thread`对于move的支持，使得我们可以构建线程对象的容器（如果这些容器是move aware的，比如std::vector<>）。这就使我们可以写出下面这样的代码：

```c++
void do_work(unsigned id);

void f()
{
	std::vector<std::thread> threads;
    for(unsigned i=0;i<20;++i)
    {
        threads.push_back(std::thread(do_work,i));		// spawn threads
    }
    std::for_each(threads.begin(), threads.end(),
                 std::mem_fn(&std::thread::join));		// call join() on each thread in turn
}
```


