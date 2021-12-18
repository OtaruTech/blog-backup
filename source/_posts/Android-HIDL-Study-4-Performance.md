---
title: 'Android HIDL学习（4） ---- 高性能比较（HIDL, FMQ, MMAP）'
date: 2019-12-15 21:12:23
categories:
- Android
- HIDL
tags:
---
#### 写在前面
公司一些方案，在Andoird P上架构必须要修改成HIDL，不然会遇到一系列的Selinux的问题，所以决定还是按照标准的Android HIDL的架构重新写了方案（因为比较机密，所以不透露具体方案代码）。但是我们的这个模块对性能的要求非常高，不然咱们的设备怎么能打败竞争对手呢，怎么屹立在世界500强呢，对吧。^_^
因为我们做的工业设备，对实时性要求比较高，但是HIDL的设计毕竟是需要进程间通信来调用到比较low level的接口的，肯定会有一些性能损失，但是好在还有一些别的机制来挽回损失，本文就来探讨一下这个问题，以及对不同的方式做一下性能的比对。

<!--more-->

#### 几种不同的方式对比
我们的需求是，应用层需要调用一些接口到kernel中的驱动，有一个HAL层封装了对驱动的操作，应用层去调用HAL层接口，其实就跟标准的AOSP的模块类似。那么问题就来了，以前是直接调用HAL接口，然后通过open/read/write/ioctl来跟驱动通信就好了，比较关心性能损耗就是系统调用到kernel中的时间损耗，其他都还好。但是一旦改成HIDL接口的写法，应用层就变成了HIDL的client端，调用到HAL的server端是用过binder/hwbinder进程间通信完成的，会有一些性能的损耗，我们就来测一下这个损耗是多少，因为我们的应用场景对这个时间比较关心，所以需要做这些分析，而且应用层会频繁的调用HAL接口。
如果比较难理解的话，我们来举个例子：
这个例子是在Android Camera开发过程中可能遇到的，在camera预览过程中，上层app需要实时的获取camera采集的图像数据，这个数据量是很大的，而且这个过程也是很复杂的，因为在预览过程中还需要调整ISP进行不同效果参数的调整，在这个过程中对camera模块进行控制很频繁（通过I2C把数据写到camera模组中）。我们假设要写进去的数据在应用层，需要通过HAL层接口调用到驱动中进行真正的I2C通信。

![](https://s2.ax1x.com/2020/03/10/8C4WPx.jpg)

左边黄色的，是使用以前的HAL层架构，直接app进程直接function call调用hall层的函数，通过ioctl的system call把数据传送到kernel层。
右边蓝色的，是使用HIDL来实现app调用底层I2C操作，分为两部分，app作为HIDL的client端通过binder进程间通信来调用server端的接口。
对我们而言，比较关心的就是proxy client端进程间调用到server端的时间延迟，毕竟是进程间通信，肯定没有直接调用来的快。而且两个不同进程，就涉及到数据的拷贝，当i2c需要写入的数据很大而且调用次数很多的时候这个拷贝和传输的延迟就会显得比较突出了。
我们这里就使用几种不同的调用方式来看时间延迟和效率问题：

*   直接调用
*   HIDL接口传输
*   Oneway HIDL interface
参考代码：

[https://github.com/JayZhang0708/HIDL-4](https://github.com/JayZhang0708/HIDL-4)

##### 最终调用干活的方法

```
#define LOG_TAG     "Sample#Lib"
#include <log/log.h>
#include <string.h>
#include "sample.h"

int writeMessage(uint8_t *data, int32_t size)
{
    int i;
    uint8_t tmp = 0;
    for(i = 0; i < size; i++) {
        tmp += data[i];
        tmp &= 0xff;
    }

    return tmp;
}
```
很简单，其实就是遍历传过来的数组。

##### 直接调用

```
void test_function_call(void)
{
    android::StopWatch stopWatch("test_function_call");
    writeMessage(buffer, BUFFER_SIZE);
}
```

##### HIDL接口传递

```
void test_hidl_interface(void)
{
    SampleMessage message;
    message.size = BUFFER_SIZE;
    message.data.resize(BUFFER_SIZE);
    ::memcpy(&message.data[0], buffer, BUFFER_SIZE);
    android::StopWatch stopWatch("test_hidl_interface");
    benchmark->writeMessage(message);
}
```

##### Oneyway HIDL接口

```
void test_oneway_hidl_interface(void)
{
    SampleMessage message;
    message.size = BUFFER_SIZE;
    message.data.resize(BUFFER_SIZE);
    ::memcpy(&message.data[0], buffer, BUFFER_SIZE);
    android::StopWatch stopWatch("test_oneway_hidl_interface");
    benchmark->writeMessageOneway(message);
}
```

##### 结果

04-30 04:14:09.425 29520 29520 D StopWatch: StopWatch test_function_call (us): **1** 
04-30 04:14:09.426 29520 29520 D StopWatch: StopWatch test_hidl_interface (us): **325** 
04-30 04:14:09.426 29520 29520 D StopWatch: StopWatch test_oneway_hidl_interface (us): **98** 
代码详细就不介绍了，之前的几篇文章都有些。
结果显而易见，直接调用是很快的，基本没有啥lentency，使用HIDL接口传输数据会比较费时间，我们这里传输了1000个byte所以比较明显。
然后就是第三个，使用了oneway
```
package vendor.sample.benchmark@1.0;

interface IBenchmark {
    writeMessage(SampleMessage message);
    oneway writeMessageOneway(SampleMessage message);
};
```
一般来说binder进程间通信时同步的，也就是当client调用接口到server的时候，需要server处理结束，然后返回给client。
当我们把接口设置成oneway，就可以把接口的调用变成异步的，当client调用接口的时候，系统会重新起一个线程来处理client的函数，调用接口的线程会直接返回。
所以当我们有一些接口不需要得到返回值而且不需要block当前线程的时候需要把接口定义为oneway来提升调用的效率，当然了，如果你压根就不考虑性能的话，那就无所谓啦。
##### 谈谈内存共享
其实在用到比较大数据传输的时候，最好的选择是使用内存共享来实现不同进程间的通信，因为使用内存共享涉及到的代码比较多，这里就不具体讲了，详细的话后面有机会再单独写一篇，在Android中如何使用内存共享实现不同进程间通信。
比较常见的例子就是Camera的HIDL实现了，把framebuffer在kernel和应用层不同进程间进行共享，实现buffer填充的时候零拷贝动作。最常用的就是使用Android的ION框架来实现。
##### 最后是FMQ
FMQ的出现就是为了解决HIDL接口通信性能差的，我们后面单独来讲解Android的FMQ，其实在浏览AOSP原生的代码中使用FMQ的场景不多，在我自己的使用过程中，我一般用作事件的通知，但是原来的写法就是使用HIDL 的callback机制来实现，我尝试改为FMQ来实现，实测下来效果不是很明显，但是会减小系统开销，不需要频繁的进程间通信了。
具体可以参考下面链接：
[https://source.android.com/devices/architecture/hidl/fmq](https://source.android.com/devices/architecture/hidl/fmq)
