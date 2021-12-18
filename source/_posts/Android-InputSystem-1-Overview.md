---
layout: blog
title: 'Android Input子系统开篇'
date: 2019-12-17 08:26:58
categories:
- Android
- Input子系统
tags:
---
### 前言

Input子系统在整个Android 系统中主要管理一些输入设备：按键、触摸屏鼠标等，他是建立在Linux的input子系统上的一套应用层软件架构，主要是处理用户的一些输入行为，反馈给前台的应用或者系统窗口。

Linux的input子系统的范围要更广，包含sensor等设备。
<!--more-->

### Input子系统系统框架

![](https://s2.ax1x.com/2020/03/10/8C5H6U.png)

从框架上看出来，主要分为三部分

- **Linux 输入设备驱动**：处理硬件的输入事件，通过文件系统发送到用户态程序
- **Android EventHub**：通过监控/dev/input/设备节点，来获取Linux输入事件，转化为Android的KeyCode
- **Android Input Manager Service**：将EventHub转化的Android KeyCode发送到合适的window上

分别对应下面的文章进行阐述。

