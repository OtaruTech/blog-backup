---
layout: blog
title: 'Android Input子系统 -- Linux'
date: 2019-12-17 20:58:40
categories:
- Android
- Input子系统
tags:
cover: /img/android-input.jpg
---
#### 前言

上一节有展示Android Input子系统的架构图，这里我们关心Linux kernel层

![](https://s2.ax1x.com/2020/03/10/8CIFne.png)
<!--more-->

可以看到kernel层分为三层：

**输入子系统设备驱动**：处理与硬件相关的信息，调用input API注册输入设备，并把数据往上报

**输入子系统核心层**：为事件处理层和设备驱动层提供API接口调用

**输入子系统事件处理**：通过核心层的API获取输入事件上报的数据，定义input API与应用层交互

#### 数据结构

| 数据结构                                                    | 代码位置                               | 描述                                     |
| :---------------------------------------------------------- | -------------------------------------- | ---------------------------------------- |
| `struct input_dev`                                          | input.h                                | input设备驱动中的实例                    |
| `struct evdev`<br />`struct mousedev`<br />`struct keybdev` | evdev.c<br />mousedev.c<br />keybdev.c | Event Handler层逻辑input设备的数据结构   |
| `struct input_handler`                                      | input.h                                | Event Handler的结构，handler层实例化对象 |
| `struct input_handle`                                       | input.h                                | 用于创建驱动层input_dev和handler链表     |

```c
//kernel/include/linux/input.h
/* 描述输入设备 */
struct input_dev {			//代表一个输入设备
	const char *name;		//设备名字，sys文件名
	//...
	struct input_id id;		//与handler匹配：总线类型、厂商、版本等
	/* 输入设备支持时间的位图bitmap */
	unsigned long evbit[BITS_TO_LONGS(EV_CNT)];			//所有事件
	unsigned long keybit[BITS_TO_LONGS(KEY_CNT)];		//按键事件
	unsigned long relbit[BITS_TO_LONGS(REL_CNT)];		//相对位移事件
	//...
	unsigned int keycodemax;		//支持按键值个数
	int (*setkeycode)();			//修改当前keymap
	int (*getkeycode)();			//检索keymap

    unsigned int repeat_key;		//记录最近一次按键值
	struct timer_list timer;

	int rep[REP_CNT];

	struct input_mt *mt;

	struct input_absinfo *absinfo;

	unsigned long key[BITS_TO_LONGS(KEY_CNT)];	//当前按键值状态
	unsigned long led[BITS_TO_LONGS(LED_CNT)];
	unsigned long snd[BITS_TO_LONGS(SND_CNT)];
	unsigned long sw[BITS_TO_LONGS(SW_CNT)];

	int (*open)(struct input_dev *dev);
	void (*close)(struct input_dev *dev);
	int (*flush)(struct input_dev *dev, struct file *file);	//处理传递给设备的事件
	int (*event)(struct input_dev *dev, unsigned int type, unsigned int code, int value);

	struct input_handle __rcu *grab;		//当前占用该设备的input_handle
	//...
	struct list_head	h_list;				//handle链表，链接此input_dev
	struct list_head	node;				//链入input_dev_list
	//...
};

/*事件处理*/
struct input_handler {
	void *private;
	void (*event)();		//处理设备驱动报告的事件
	void (*events)();
	bool (*filter)();
	bool (*match)();
	int (*connect)();		//连接handler和input_dev
	void (*disconnect)();	//断开连接
	void (*start)();		//启动指定handle的handler函数

	bool legacy_minors;
	int minor;
	const char *name;		//handler名

	const struct input_device_id *id_table;	//输入设备id列表，匹配input_dev设备信息

	struct list_head	h_list;	//链入handle链表
	struct list_head	node;	//链入input_handler_list
};

/* 
 * 连接 input_dev 和 handler 的桥梁
 * 一个 input_dev 可以对应多个 handler ， 一个 handler 也可以对应多个dev
*/
struct input_handle {
	int open; // 设备打开次数（上层访问次数）
	const char *name;

	struct input_dev *dev;  // 所属 input_dev
	struct input_handler *handler; // 所属 handler

	struct list_head	d_node; // 链入对应 input_dev 的 h_list
	struct list_head	h_node; // 链入对应 handler 的 h_list
};
/* 事件载体，输入子系统的事件包装为 input_event 上传到 Framework*/
struct input_event {
 struct timeval time; // 时间戳
 __u16 type;  // 事件类型
 __u16 code;  // 事件代码
 __s32 value;  // 事件值，如坐标的偏移值
};
```

对于handler和device，分别用链表`input_handler_list`和`input_device_list`进行维护，这两条是全局链表

`input_handle`结构体代表一个成功配对的`input_dev`和`input_handler`。`input_handle`没有一个全局的链表，它注册的时候将自己分别挂在`input_device_list`和`input_handler_list`的`h_list`上；同时，`input_handle`的成员.dev，关联到`input_dev`结构，.handler关联到`input_handler`结构。

![](https://s2.ax1x.com/2020/03/10/8CI236.png)

#### **输入子系统流程**

- 子系统入口函数：`subsys_initcall(input_init); `

  - ​	`class_register(&input_class)`：在/sys/class下创建input类
  - ​	`input_proc_init()`：在/proc下建立相关文件
  - ​	`register_chrdev_region(MKDEV(INPUT_MAJOR, 0), INPUT_MAX_CHAR_DEVICES, "input")`：申请字符设备主设备号为13

- 注册input设备

  - ​	添加设备
  - ​	把输入设备挂到输入设备链表`input_dev_list`中
  - ​	遍历`input_handler_list`链表，查找并匹配输入设备对应的时间处理层，如果匹配上，就调用`handler`的`connect`函数进行连接。

- 事件处理入口函数：`module_init(evdev_init);` -> `input_register_handler`

  - 把设备处理器挂到全局的input子系统设备链表`input_handler_list`上
  - 遍历input_dev_list，与每一个input_dev进行匹配：`input_attach_handler`
  - `input_attach_handler`
    - ​	`input_match_device`
    - ​	`handler->connect`
    - 申请此设备号：`input_get_new_monor`
    - 设置设备节点名称/dev/eventX
    - 设置应用层使用的设备号
    - `input_dev`设备驱动和handler事件处理层的关联：`input_register_handler`
    - 将设备加入到Linux设备模型：`device_add`

- 上报事件

  - `input_report_xx()`
    - `input_event()`
  - `input_sync`
    - `input_event()`

- 举例一个简单的input设备驱动：

  https://landlock.io/linux-doc/landlock-v7/input/input-programming.html