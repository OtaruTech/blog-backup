---
title: FastDDS进程间内存共享
date: 2022-04-26 23:50:23
tags:
- FastDDS
- DDS
- 自动驾驶

categories:
- FastDDS

cover: /img/fastdds-03-shm-arch.png
---

## 概述

> 本文描述了一个通过共享内存实现进程间通信模型，通过这个模型，不同的软件组件可以进行数据交换。这个模型的目标是提供了一种基于共享内存的数据传输层，来适配一个实时订阅发布的DDS（Data Distribution Service）框架。
> 

## 上下文

eProsima是接收到了一个商业上的方案通过共享内存传输来改进FastRPTS产品。下面是这一块列出的一些目标。

### **通过标准的网络传输的改进**

- 减少操作系统内核的系统调用：这对于UDP/TCP是不可避免的（甚至在loopback的设备上）
- 大数据传输支持：网络传输往往需要把大数据切片传输
- 避免数据的序列化和反序列化过程：在异构网络中是不可能的，但是在进程间共享内存中是可行的
- 减少内存拷贝：共享内存可以实现零拷贝，客户端可以直接拿到共享buffer的指针进行访问

### **目标**

- 提高进程间通信的性能
- 创建一套可移植的共享内存库（Windows/Linux/MacOS）
- 文档
- 测试
- 示例代码

## 架构

### **设计理念**

- **Segment：**一块固定大小的共享内存，不同的进程可以访问。共享内存快有一个全局的名称，所以任何一个知道这个名称的进程都可以打开这块内存，然后映射到自己的地址空间
- **SegmentId：**`SegmentId`是共享内存快的唯一标识ID，是一个16字符的UUIDs
- **Shared memory buffer:** 是一块共享内存快内分配的buffer
- **Buffer descriptor：**共享内存的缓存可以被buffer描述符引用，这些描述符就像是这块buffer的一个指针，它们可以在不同进程之间拷贝，作为一个最小的消耗。一个描述符包含了`SegmentId`和对应buffer在内存块中的偏移
- **Shared memory port：**是一个通信的通道，通过一个`port_id`(`uint32_t`数字)来标识。通过这个通道，buffer描述符可以发送到另外一个进程。它是一个ring-buffer，也存储在共享内存中（与buffer描述符一样）。不同的进程可以通过同一个端口进行数据的读写操作。多个listener可以被注册到一个端口，当buffer描述符被推送到ring-buffer的时候可以被通知到。多个数据产生者可以推送到同一个端口。端口中包含了一个被注册的listener数量的原子计数器，缓冲区的每个位置也包含了一个listeners数量的计数器，当listener读取buffer描述符时，计数会减小，所以当计数值为0时，ring-buffer的位置可以被释放。Port还有一个进程间的条件变量，listener可以通过它来等待到来的buffer描述符
- **Listener：**Listeners监听被推送到端口的buffer描述符。listener给应用程序提供了等待和访问buffer中数据的方法。当一个消费者从port listener中取出一个buffer描述符，查看描述符的`SegmentId`，来打开原始共享内存块，当原始内存块在本地映射好之后，消费者就可以通过偏移量来访问到描述符中的数据了
- **SharedMemoryManager：**应用程序通过`SharedMemoryManager`可以访问到上述共享内存相关的资源。一个进程至少需要一个`SharedMemoryManager`。这个manager为应用程序提供了创建共享内存块，在内存块中分配数据，推送buffer描述符到共享内存端口，创建关联到端口的listener

### 一个**示例场景**

![interprocess_shared_mem1.png](fastdds-03-shm-arch/interprocess_shared_mem1.png)

我们来看一下上面的例子。有三个进程，每个进程都有一个`SharedMemManager`对象，每个进程都创建了自己的共享内存快，意图存储数据然后分享给其他进程。

P1、P2和P3进程同事参与了RTPS-DDS环境（DOMAIN）。使用multicast可以完成对参与者的发现，所以我们选用共享内存port0作为“multicast”端口来用作发现功能，因此，所有参与者第一件要做的事情就是打开共享内存port0。对于每一个参与者，通过创建一个绑定到port0的listener，来读取到来的buffer描述符。

每一个参与者打开一个端口接收unicast消息，端口1、2和3分别和创建listener的端口关联。

参与者发送的第一个消息是multicast发现消息：“我在这里，我正在监听端口N”。所以参与者们在本地的内存块中分配一个buffer，通过port0把信息推送到buffer描述符。观察ring-buffer是如何存储buffer描述符到Data 1a(P1), Data2a(P2), Data3a(P3)，到这里所有的进程就已经把发现的消息描述符推送出去了。

发送阶段之后，参与者就知道了别的参与者和他们的“unicast”端口，所以他们可以通过unicast端口发送消息给指定的参与者。

最后，我们来观察一下P1是如何分享下Data1c给P2和P3的，这是通过推送buffer描述符到P2和P3的unicast端口来完成的。这样就可以通过零拷贝的方式把数据分享给别的进程了（只拷贝buffer描述符）。这对于UDP和TCP这种通信方式而言是非常重要的一个提升。

### **设计注意事项**

- **最小化全局进程间的锁：**进程间的锁是非常危险的，加入参与者的进程在获得全局锁之后崩溃了，所有的协作进程都会被影响到。在这个设计中，读写数据是无锁操作。为了性能考虑等待数据端口，当端口是空的时候，需要进程间锁的机制：互斥锁和条件变量。在multicast端口中是非常危险的，因为当一个listener在等待的时候崩溃了，会block这个端口中别的listener
- **可伸缩的进程数量：**每个进程创建自己的本地的共享内存块，这比申请全局的共享内存更有好处
- **每个应用/进程客制化内存使用**

### 对应到FstRTPS的设计

**传输层**

- **Locators：**LOCATOR_KIND_SHM被定义来表示当前endpoint使用共享内存。Locator_t：
    - kind：16
    - port：Locator的端口号包含一个共享的port_id
    - address：出了第一个byte用来标识unicast(address[0]=”U”)或者multicast(adress[0]=”M”)，其他地址都被设置为0
- **SharedMemTransportDescriptor**
    - segment_size：共享内存块的大小
    - port_queue_capacity：共享内存端口消息队列的大小
    - port_overflow_policy：DISCARD或者FAIL
    - segment_overflow_policy：DISCARD或者FAIL
    - max_message_size
- **Default metatraffic multicast locator**
- **Default metatraffic unicast locator**
- **Default output locator**
- **OpenInputChannel**
- **OpenOutputChannel**

**Class设计**

![interprocess_shared_mem2.png](fastdds-03-shm-arch/interprocess_shared_mem2.png)

**RTPS层**

FastRTPS传输是关联到participant的，所以新的SHM传输可以通过SharedMEMTransportDescriptor的一个类绑定到参一系列参与者的传输。