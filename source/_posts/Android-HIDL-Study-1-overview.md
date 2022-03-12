---
title: 'Android HIDL学习（1） ----  简介'
date: 2019-12-15 20:59:57
categories:
- Android
- HIDL
tags:
cover: /img/android-hidl.png
---
#### **HIDL**

HAL接口定义语言（简称HIDL）适用于指定HAL和其用户之间的接口的一种接口描述语言（IDL）。HIDL允许指定类型和方法调用。从更更烦的意义上来说HIDL适用于在独立编程的代码库之间通信的系统。

HIDL旨在用于进程间通信（IPC）。进程之间的通信经过Binder化。对于必须与进程相关联的代码库，还可以使用直通模式。

HIDL可指定数据结构和方法签名，这些内容会整理归类到接口中，而接口会汇集到软件包中。尽管HIDL具有一系列不同的关键字，C++和JAVA程序员对HIDL的语法并不陌生。此外，HIDL还是用JAVA样式和注释。

<!--more-->

#### **扯淡一下**

好吧，上面这段HIDL的简介是从Android开发者官网抄过来的[https://source.android.com/devices/architecture/hidl/](https://source.android.com/devices/architecture/hidl/)，大家读上去可能会比较别扭，没办法本来英文挺好的，被翻译成中文就是这个鸟样了，我不是说中文不好，我负责Android开发官网中文翻译的肯定是个老外，或者就是Google翻译软件自动翻译的，反正一般人是理解不了的。

OK，我也不想从官网上去炒一堆文字来忽悠大家，毕竟大家的时间都是很宝贵的，那种垃圾翻译文章就let it go，当然啦，官方的文档固然是好的，但是写的比较粗和简单，人家都是站在一个不知道多高的高度来写的，所以的，我这边会记录Android HIDL开发的一些知识，当然了因为Android 8.0出来也没有多久，网上相关的文档寥寥无几，特别是想我这样子手把手的指导写代码的技术文档啦。

#### **HIDL设计**

其实HIDL的出现是为了更好的服务于Treble这个项目，不了解Treble的可以先从网上找一下相关的资料，我这里简单做说明。

严重的碎片化：

![](https://s2.ax1x.com/2020/03/10/8CfmsH.png)

由于Android的发展比较迅猛，各大手机厂商和芯片厂商都在做，Google当然作为一个领导者在指引我们做出更好的手机操作系统，但是由于版本太多，Android版本的碎片化越来越严重，而且系统的更新又是一个耗时和复杂的过程，Google试图来解决这个问题而引入了Treble，大家都知道做手机的，比如：小米，华为，VIVO等厂商，他们维护自己的BSP，基本上他们的BSP包含几部分：

![](https://s2.ax1x.com/2020/03/10/8CfRm9.png)

注意，这里的Framework是vendor修改过的，这样子的话，这四部分都耦合在一起，因为Google每次更新Android大版本，基本上都是framework的升级，与vendor改的代码理论上是可以独立开来的，所以Google尝试通过Treble来独立更新system.img来帮助vendor更快的移植新的Android版本。

以前HAL是以so的形式存在的，作为一堆标准接口，供Android framework调用，无论是通过jni还是别的途径，如果要被framework调用，那这些so就一定要存在于system分区，但是我们现在要把system分区独立开来，这样子，vendor修改的代码全部要在vendor分区，所以，引入了HIDL来解决这个问题，vendor设计的HAL都以独立的service存在，每一个HAL模块都是一个独立的binder server进程，Android framework想用调用HAL的接口就必须作为binder的client来调用，后面会详细描述，这里大家只要记住这个概念就OK。

![](https://s2.ax1x.com/2020/03/10/8Cf5Y6.png)

#### **结尾**

介绍了一些简单的概念，后面我们会以一个简单的例子给大家诠释。
