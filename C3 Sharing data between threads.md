# C3 Sharing data between threads

This chapter covers:

+ problems whit sharing data between threads
+ protecting data with mutexes
+ alternative facilities for protecting shared data 



使用线程来支持并发的一个好处是可以容易并且直观地在线程间进行**数据共享**。接下来将围绕数据共享介绍存在的一些问题。

在线程间不正确的共享数据时导致并发相关的bug的最大的原因。

本章将介绍在C++中如何在线程间安全地共享数据，并避免潜在的问题。

目录：

[toc]

## 3.1 Problems with sharing data between threads

归根结底，线程间共享数据所带来的问题都来自于 修改数据的结果。

**如果所有的数据都是只读的，那么就不会有任何问题**，因为某个线程对数据的读不会对其他线程读这份数据有任何影响。

如果多个线程共享一份数据，而且会修改这份数据，那么就会出现一些潜在的问题，就需要好好注意。



多个线程对于共享数据的修改会导致**竞争**行为，也就是这些线程的执行顺序会导致结果不同。



### 3.1.1 Race conditions

做了一些介绍性的描述。

有些race conditions是比较温和可接受的，不会对结果产生什么影响，比如说往一个队列里添加要处理的事件，如果事件先后执行顺序无关紧要，那么这种竞争就几乎不会有危害。我们主要讨论的是会造成程序出问题的race conditions。

race conditions在debugger模式下通常不可再现。



### 3.1.2 Avoiding problematic race conditions

有很多方法可以解决这种有危害的竞争：

+ 最简单的方法是将我们的数据结构**通过一种保护机制打包起来**，确保只有进行修改的线程可以访问这些数据，当修改完成后其他线程才能访问这部分数据。C++提供了许多支持这种行为的机制，本章将会进行介绍。
+ 使用**无锁编程**技术，使得不同线程之间的修改是成系列的。无锁编程在第七章介绍。

+ 使用**transaction**来处理数据的更新。这要求对数据的修改和读都保存在 transaction log内，并且进行提交。这通常被称为STM(software transactional memory)，但本书并不介绍这部分，因为C++并没有提供对STM的直接支持。

C++提供的最基础的共享数据保护机制是**`mutex`**。



## 3.2 Protecting shared data with mutexes

我们可以通过**互斥量mutex** (**mut**ual **ex**clusion)来实现我们的同步机制，即，在一个线程对共享数据进行修改时，其他线程只能等待。

当我们要处理一个共享数据的时候，我们**lock跟这个数据相关的mutex**，等处理完之后，再**unlock这个mutex**。

线程库保证，一旦一个线程lock了一个mutex，那么其他线程想再lock这个mutex，只能等待前面成功lock这个mutex的线程unlock这个mutex。

这样就保证所有的线程看到的共享数据是相同的。

但是mutex不是万能的，我们需要精心设计我们的代码的结构(3.2.2)，避免接口的潜在竞争条件(3.2.3)。

mutex也带来了新的问题，比如说死锁(3.2.4)，或者是一次保护了太多或太少的数据(3.2.8)。



### 3.2.1 Using mutexes in C++

在C++中，可以通过构建一个`std::mutex`的实例来获得对mutex的支持，成员函数`lock()`和`unlock()`用来lock和unlock。但是并不推荐直接对`std::mutex`实例进行`lock()`，因为如果这样做了，我们得记得在所有的用到的代码中进行`unlock()`，包括在异常处理中。

C++库提供了模板类`std::lock_guard`，它通过RAII的机制来保管一个mutex实例，在它的构造函数中进行lock，析构函数保证unlock。

下面的代码是一个保护一个list的例子，`std::mutex`和`std::lock_guard`都在`<mutex>`头文件里。

```c++
// listing 3.1 Protecting a list with a mutex
#include<list>
#include<mutex>
#include<algorithm>

std::list<int> some_list;							// 1
std::mutex some_mutex;								// 2

void add_to_list(int new_value)
{
    std::lock_guard<std::mutex> guard(some_mutex);	// 3
    some_list.push_back(new_value);
}
bool list_contains(int value_to_find)
{
    std::lock_guard<std::mutex> guard(some_mutex);			// 4
    return std::find(some_list.begin(), some_list.end(), value_to_find) != some_list.end();
}

```

上面的代码中，在代码1 处，我们有一个全局变量，然后用一个`std::mutex`实例来对其提供保护。

在函数`add_to_list`和`list_contains`中，都通过`std::lock_guard`提供了对线程持有数据的互斥保护。（有个问题，这里怎么知道这个函数有什么数据资源或者怎么保护数据的）

但通常代码不这么写，在面向对象设计中，通常把数据和函数都放在一个类里，那么这两个函数都将变成一个类的成员函数，数据都会变成private 数据成员，这样一方面对数据进行了封装，另一方面，我们更加明确这个锁和数据是相关的（也就是这个锁是用来保护这个数据的，解答了前面的那个问题），增强了保护性。

如果所有成员函数都使用了`std::lock_guard`，也就是在进入成员函数的时候，进行lock，离开成员函数的时候，进行unlock，那么就可以保证数据被很好的保护了。

但是，这并不是**完全**正确的，如果一个成员函数返回了对保护数据的引用或者指针呢？这就导致，我们前面精心设计的lock-unlock机制就没用了，因为，获得了这个reference 或pointer的代码对数据进行修改的时候并没有lock mutex（前面我们对mutex的lock和unlock操作都是因为我们进入了member function，而member function中都有lock_guard，这是一种RAII策略，里面保证了，在这个lock_guard构造好的时候，它所保管的那个mutex是执行了lock的，然后我们离开这个member function的时候，自动调用这个lock_guard实例的析构函数，这里面包含了对mutex的unlock操作，我们也就释放了这个mutex，但是这个通过reference 或pointer对数据的操作却跟前面不同，这个地方直接把数据暴露了出来，我们不再能提供对mutex 的 lock 和unlock，也就不能保证数据不会有race conditions了）

所以说，通过mutex保护数据，需要我们**谨慎地设计接口**，要保证任何方式获取数据都会lock mutex。



### 3.2.2 Structuring code for protecting shared data

如前面所说的，保护数据并不是用一个`lock_guard`就行了，如果成员函数传出一个指针  或者引用，那么这会使我们的努力白费。如果成员函数不返回指针或引用，或者传出的参数不包含指针引用，那么就安全了吗？也不。

我们还需要注意，**不能向成员函数调用的函数**传递受保护数据的指针或引用，如下代码：

```c++
// Listing 3.2
class some_data
{
    int a;
    std::string b;
public:
	void do_something();    
};

class data_wrapper
{
private:
	some_data data;
    std::mutex m;
public:
    template<typename Function>
    void process_data(Function func)
    {
        std::lock_guard<std::mutex> l(m);
        func(data);							//1 向用户函数传递了 受保护的数据
    }
}

some_data* unprotected;

void malicious_function(some_data& protected_data)
{
    unprotected = &protected_data;
}

data_wrapper x;

void foo()
{
    x.process_data(malicious_function);	//2 passing a malicious function
    unprotected->do_something();		//3 unprotected access to protected data
}
```

上面的代码中，我们对成员函数`process_data`做了保护，而且它也没有传出指针或引用。但这个成员函数向它调用的函数传递了保护数据的指针，就导致后面对受保护数据进行了超出我们预期的修改行为（具体来说，是成员函数向非我们可控的函数传递了保护数据的指针或引用，导致在那个非可控的函数中存在修改保护数据的潜在风险）。

guideline:**不要将保护数据的指针或引用传出lock所在的scope，不管是从函数中返回，存储在外部可见的内存中，还是将它们作为参数传入用户支持的函数里**。



上面所介绍的并不是唯一可能出现的问题，还有更多的问题，仍然可能出现race condition，即使数据使用了互斥量去保护。



### 3.2.3 Spotting race conditions inherent in interfaces