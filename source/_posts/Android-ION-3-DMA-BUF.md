---
layout: blog
title: 'Android ION内存管理(3) -- dma-buf'
date: 2021-02-16 21:42:30
categories:
- Android
- ION内存管理
tags:
- Android
- ION
- 内存管理
cover: /img/android-ion.png
---
前面讲了如何在内核空间以及用户空间利用`ion`的接口分配内存、共享内存、同步内存数据，但是最近在浏览最新的`AOSP`的系统变更的时候发现接口都发生了改变：

https://source.android.com/devices/architecture/kernel/ion_abi_changes

<!--more-->

![](https://s3.ax1x.com/2021/02/16/ygm9mQ.png)

意思就是在kernel 4.14版本和之后ION的接口有一个重大的重构，使得在使用kernel 4.14和之后的版本之后的一些代码都需要注意很多地方需要修改。特别是Display，Camera，DRM等模块的kernel驱动和硬件抽象层的代码。

下面来看看有哪些改动的地方。

------

其实在kernel 4.12已经开始重构ion的内核代码了，清除了一些和Android 框架层强耦合的代码，一些ion的ioctl接口被移除了。

#### 移除ION clients和handles

在kernel 4.12之前，通过打开`dev/ion`获取到`ion client`，然后通过`IOC_ION_ALLOC` ioctl 来分配一块新的buffer，到用户空间就是返回一个`ion handle`，这个handle就代表了一块buffer。为了把这块buffer分享给别的进程，通常需要调用`IOC_ION_SHARE` ioctl来把buffer重新导出为dma-buf。

在kernel 4.12及以后，`IOC_ION_ALLOC`直接返回的是dm-buf的文件描述符，这个dma-buf的文件描述符是直接可以用来共享的，也不会帮到定某个ion handle和ion client上，所以`IOC_ION_SHARE`就不在需要了。

#### 添加了cache-coherency接口

在内核4.12之前，Ion提供了一个`ION_IOC_SYNC` ioctl来同步文件描述符和内存。这个ioctl的文档写的很差，而且缺乏灵活性。因此，许多供应商实现了自定义ioctl来执行缓存维护。

kernel 4.12使用了`DMA_BUF_IOCTL_SYNC ioctl`来代替`ION_IOC_SYNC`，定义在`linux/dma-buf.h`。在CPU的代码访问内存之前和之后调用`DMA_BUF_IOCTL_SYNC`，并指定是要读还是写。虽然说`DMA_BUF_IOCTL_SYNC`也会显得有一些冗余，但是这个接口还是更用户空间更多的选择来处理缓存同步的问题。

#### 移植代码到Android kernel 4.12+

![](https://s3.ax1x.com/2021/02/16/ygmPTs.png)

对于内核的客户端使用，因为Ion不再导出任何面向内核的API，所以以前使用`ion_import_dma_buf_fd()`的内核中Ion内核API的驱动程序必须转换为使用dma_buf_get()的内核中dma-buf API。

