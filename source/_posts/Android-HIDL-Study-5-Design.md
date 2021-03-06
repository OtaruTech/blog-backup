---
title: 'Android HIDL学习（5） ---- 设计要素'
date: 2019-12-15 21:13:18
categories:
- Android
- HIDL
tags:
cover: /img/android-hidl.png
---
前面我们学习了如何使用HIDL来设计或者重构之前在HAL层的代码，而且也对比了一些高性能的编程方式，这里我们在来一下Android的HIDL在设计上的一些考虑。

<!--more-->

HIDL指定了数据结构和方法的命名，这些命名类似于JAVA中的类，所以HIDL的语法对于C++和JAVA程序员来说是非常熟悉的，尽管有些关键字不怎么相同，HIDL还使用JAVA的注释方式。

#### **HIDL设计**目标

HIDL的设计目标是为以后系统更新的时候不用重新编译HAL的模块，HALs被供应商或者SOC厂商负责被编译成vendor.img烧录到系统的/vendor 分区。这样子在后面系统更新的时候只需要更新vendor分区，不需要更新整个OTA包。

##### HIDL的设计理念平衡一下三个要素：

**可互操作性：**在进程中创建可靠的可互相调用的接口，这种接口偶可以通过不同的交叉编译和和配置来编译成不同的架构。HIDL接口是通过版本来定义的，不同版本的接口可以在发布之后改变。

**效率：**HIDL的设计在进程通信间尽可能最小化拷贝的动作。HIDL定义的数据在C++标准布局数据结构中被传递到C++代码，这些数据结构可以在不解包的情况下使用。由于使用进程间通信会比直接调用来的慢，HIDL也提供了共享内存的方式，HIDL除了RPC通信外，还提供另外两种方式进行数据传输：内存共享和快速消息队列（FMQ）。

**直观性：**HIDL只有在两个进程需要RPC通信时才使用，从而避免了内存所有权的问题，数据值可以被方法或者回调函数有效的得到返回。将数据传递到HIDL进行传输或者从HIDL接收数据都不会更改数据的所有权，而始终保留在调用函数中。数据只需要在被调用函数的持续时间内保持，并且可以在被调用函数返回后立即销毁。

**HIDL语法详细参考下面链接：**

[https://source.android.com/devices/architecture/hidl](https://source.android.com/devices/architecture/hidl)
