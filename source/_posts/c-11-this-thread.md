---
title: 命名空间-this_thread
date: 2022-03-16 20:18:00
tags:
- C++11
- thread

categories:
- C++
- C++11

cover: /img/c++11-this_thread.png
---

在C++11中不仅添加了线程类，还添加了一个关于线程的命名空间`std::this_thread`，在这个命名空间中提供了四个公共的成员函数，通过这些成员函数就可以对当前线程进行相关的操作了。

## 1. get_id

调用命名空间`std::this_thread`中的`get_id()`方法可以得到当前线程的线程ID，函数原型如下：

```cpp
thread::id get_id() noexcept;
```

关于函数使用对应的示例的代码如下：

```cpp
#include <iostream>
#include <thread>
using namespace std;

void func()
{
    cout << "子线程: " << this_thread::get_id() << endl;
}

int main()
{
    cout << "主线程: " << this_thread::get_id() << endl;
    thread t(func);
    t.join();
}
```

程序启动，开始执行`main()`函数，此时只有一个线程也就是主线程，当创建了子线程对象`t`之后，指定的函数`func()`会在子线程中执行，这时通过调用`this_thread::get_id()`就可以得到当前线程的线程ID了。

## 1. sleep_for()

线程被创建之后有五种状态：`创建态，就绪态，运行态，阻塞态（挂起态），退出态（终止态）`，关于状态之间的转换和进程是一样的。

线程和进程的执行有很多项次之处，在计算器中启动的多个线程都需要占用CPU资源，但是CPU的个数是有限的并且每个CPU在同一时间点不能同事处理多个任务，`为了能够实现并发处理，多个线程都是分时复用CPU时间片，快速的交替处理哥哥线程中的任务，因此多个线程之间需要争抢CPU时间片，抢到了就执行，抢不到则无法执行`（因为默认所有的线程优先级都相同，内核也会从中调度，不会出现某个线程永远抢不到CPU时间片的情况）。

命名空间`this_thread`中提供了一个休眠函数`sleep_for`，调用这个函数的线程会从`运行态`变成`阻塞态`并在这种状态下休眠一定的时长，因为阻塞态的线程已经让出了CPU资源，代码也不会执行，所以线程休眠过程中对CPU来说没有任何负担。这个函数就原型如下，参数是需要指定休眠的时长，是一个时间段：

```cpp
template <class Rep, class Period>
  void sleep_for (const chrono::duration<Rep,Period>& rel_time);
```

示例代码如下：

```cpp
#include <iostream>
#include <thread>
#include <chrono>
using namespace std;

void func()
{
    for (int i = 0; i < 10; ++i)
    {
        this_thread::sleep_for(chrono::seconds(1));
        cout << "子线程: " << this_thread::get_id() << ", i = " << i << endl;
    }
}

int main()
{
    thread t(func);
    t.join();
}
```

在`func()`函数的`for`循环中使用了`this_thread::sleep_for(chrono::seconds(1));`之后，没循环一次程序都会阻塞1秒钟，也就是说每隔1秒才会进行一次输出。需要注意的是：程序休眠完成之后，会从阻塞态重新变成就绪态，就绪态的线程需要再次争抢CPU时间片，抢到之后才会变成运行态，这时候程序才会继续向下运行

## 3. sleep_util()

命名空间`this_thread`中提供了另一个休眠函数`sleep_util()`，和`sleep_for()`不同的是它的参数类型不一样

`**sleep_util()**`: 指定线程阻塞到某一个指定的时间点`time_point`类型，之后解除阻塞

`**sleep_for()**`: 指定线程阻塞一定的时间长度`duration`类型，之后解除阻塞

函数原型如下：

```cpp
template <class Clock, class Duration>
  void sleep_until (const chrono::time_point<Clock,Duration>& abs_time);
```

示例程序如下：

```cpp
#include <iostream>
#include <thread>
#include <chrono>
using namespace std;

void func()
{
    for (int i = 0; i < 10; ++i)
    {
        // 获取当前系统时间点
        auto now = chrono::system_clock::now();
        // 时间间隔为2s
        chrono::seconds sec(2);
        // 当前时间点之后休眠两秒
        this_thread::sleep_until(now + sec);
        cout << "子线程: " << this_thread::get_id() << ", i = " << i << endl;
    }
}

int main()
{
    thread t(func);
    t.join();
}
```

`sleep_util()`和`sleep_for()`函数的功能是一样的，只不过前者是基于时间点去阻塞线程，后者是基于时间段去阻塞线程，项目开发过程中根据实际情况选择最优的解决方案即可。

## 4. yield()

命名空间`this_thread`中提供了一个非常绅士的函数`yield()`，在线程中调用这个函数之后，处于运行态的线程会主动让出自己已经抢到的CPU时间片，最终变为就绪态，这样其他的线程就有更大的概率能够抢到CPU时间片了。

使用这个函数的时候需要注意一点，线程调用了`yield()`之后会主动放弃CPU资源，但是这个变为就绪态的线程会马上参与到下一轮CPU的抢夺战中，不排除它能继续抢到CPU时间片的情况，这时概率问题。

```cpp
void yield() noexcept;
```

函数对应的示例程序如下：

```cpp
#include <iostream>
#include <thread>
using namespace std;

void func()
{
    for (int i = 0; i < 100000000000; ++i)
    {
        cout << "子线程: " << this_thread::get_id() << ", i = " << i << endl;
        this_thread::yield();
    }
}

int main()
{
    thread t(func);
    thread t1(func);
    t.join();
    t1.join();
}
```

在上面的程序中，执行`func()`中的`for`循环会占用大量的时间，在极端情况下，如果当前线程占用CPU资源不释放就会导致其他线程中的任务无法被处理，或者该线程每次都能抢到CPU时间片，导致其他线程中的任务没有机会被执行。解决方案就是每次执行一次循环，让该线程主动放弃CPU资源，重新和其他线程再次抢夺CPU时间片，如果其他线程抢到了CPU时间片就可以执行相应的任务了。

**结论：**

1. `std::this_thread::yield()`的目的是避免一个线程长时间占用CPU资源，从而导致多线程处理性能下降
2. `std::this_thread::yield()`是让当前线程主动放弃了当前自己抢到的CPU资源，但是在下一轮还会继续抢