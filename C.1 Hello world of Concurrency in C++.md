# C.1 Hello world of Concurrency in C++

本章内容概括：

+ 并发和多线程的含义
+ 为什么在程序中要用到并发和多线程
+ C++并发史
+ C++程序中的多线程是什么样子的



## 1.1 并发是什么？

### 1.1.1 计算机系统里的并发

操作系统相关的书里都有，还更详细，这里大概浏览了一下，懒得写了 :)





## 1.2 Why use concurrency?

两个主要的原因：

1. separation of concerns 模块划分
2. performance 性能



### 1.2.1 Using Concurrency for separation of concerns

使用并发使任务分离，使得不同任务逻辑之间的划分更清晰。

这种情况下线程的数量和CPU核数无关。



### 1.2.2 Using Concurrency for performance

两种方式可以利用并发提高性能：

1. 将任务分成多个部分然后并行执行，称之为 task parallelism。虽然这听上去比较直观，但是实现起来比较复杂，因为任务不同的部分之间可能会有依赖关系。可能使将不同的数据用不同的线程处理，称之为data parallelism，或者将算法中不同环节交给不同的线程去处理。
2. 更大的数据并行，比如说同时处理几个文件。

### 1.2.3 什么时候不用并发？

当代价大于收益时。

使用并发时，代码写起来比较费脑子、更复杂，而且难看懂（相对来说），所以很容易有bug。

另外，多线程带来的提升可能没有预想中的大。因为启动线程有开销，

如果任务执行的特别快，可能执行任务的时间比开启线程的时间还要小很多，就导性能还不如之前。

另外，线程是有限的资源，开启太多线程会耗费大量内存和地址空间，因为每个线程都要分配一个栈空间。

虽然对于64位的机器不会有耗尽内存的危险，但它仍然是资源有限的。

还有就是，越多的线程，代表着OS要进行越多的上下文切换。如果一个线程的任务的解决时间非常小，那么这个切换的时间就会很突出，那么用多线程就会导致整体性能下降。

使用并发性来提升性能跟其他的优化策略一样：它有可能会提升性能，但也会导致代码变复杂，更难懂，更容易出错。





## 1.3 Concurrency and multithreading in C++

C++11 标准使我们可以不用再写 依赖于特定平台 的多线程代码了。

### 1.3.1 History of multithreading in C++





## 1.4 Getting Started

### 1.4.1 hello, concurrent world

```c++
#include<iostream>
#include<thread>

void hello()
{
	std::cout << "Hello Concurrent World\n" << std::endl;
}

int main()
{
	std::thread t(hello);			// 3
	t.join();						// 4
}
```

\<threads>内包含了处理线程的类和函数，而保护数据的在其他头文件里。

每个线程都有一个初始化函数（initial function），它是新线程执行的开始。

程序中的初始线程是 `main()`，其他的线程都通过 `std::thread`的构造函数创建线程对象，在这个例子中，创建了一个名为`t`的`std::thread`对象，并利用函数`hello()`将其初始化。

上面的代码创建了一个新的线程。

语句3启动新的线程，这个时候，初始线程（也就是`main()`）仍然在执行。

如果没有语句4，那么在3执行完后，`main()`就结束了，但是新的线程`t`可能还没有开始执行。

因此我们加入语句4，让calling thread也就是`main()`去等待与它有关的线程（也就是在这个calling thread`main()`里产生的线程）。