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

