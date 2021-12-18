---
layout: blog
title: 'Android ION 内存管理'
date: 2019-12-16 11:14:25
categories:
- Android
- ION内存管理
tags:
---
##### **ION的设计初衷**

Android为了更好的针对移动设备内存的管理，设计出了ION内存管理机制，主要是为了解决以下几个问题：

- 预留大块连续内存，比如camera，display，GPU等模块
- 避免内存随便花
- 用户控件和硬件之间实现"零拷贝"(zero-copy)的内存共享
<!--more-->
做Android系统的，特别是跟Display，camera模块相关的

ION的官方介绍和历史由来查看下面的介绍：

https://lwn.net/Articles/480055/

##### **ION的实现**

Android系统的ION实现依赖于不同的`CPU/GPU`硬件，Android提供了ION的框架供CPU厂商(NXP, Qualcomm, MTK, 海思等)在自己的`BSP`里面实现`ION`机制。

默认的ION驱动会提供以下三种不同的`ION heaps`实现：

- **ION_HEAP_TYPE_SYSTEM**: memory allocated via vmalloc_user()
- **ION_HEAP_TYPE_SYSTEM_CONFIG**: memory allocated via kzalloc
- **ION_HEAP_TYPE_CARVEOUT**: carveout memory is physically contiguous and set aside at boot

CPU厂商会根据自己CPU的特性实现更多的`ION heaps`，比如`NVIDIA`提交了一种`ION_HEAP_TYPE_IOMMU`的heap。

不管哪一种`ION heaps`实现，都必须实现如下核心接口

```C
struct ion_heap_ops {
	int (*allocate)(struct ion_heap *heap,
			struct ion_buffer *buffer, unsigned long len,
			unsigned long align, unsigned long flags);
	void (*free)(struct ion_buffer *buffer);
	int (*phys)(struct ion_heap *heap, struct ion_buffer *buffer,
		    ion_phys_addr_t *addr, size_t *len);
	struct sg_table * (*map_dma)(struct ion_heap *heap,
				     struct ion_buffer *buffer);
	void (*unmap_dma)(struct ion_heap *heap, struct ion_buffer *buffer);
	void * (*map_kernel)(struct ion_heap *heap, struct ion_buffer *buffer);
	void (*unmap_kernel)(struct ion_heap *heap, struct ion_buffer *buffer);
	int (*map_user)(struct ion_heap *mapper, struct ion_buffer *buffer,
			struct vm_area_struct *vma);
	int (*shrink)(struct ion_heap *heap, gfp_t gfp_mask, int nr_to_scan);
	void (*unmap_user) (struct ion_heap *mapper, struct ion_buffer *buffer);
	int (*print_debug)(struct ion_heap *heap, struct seq_file *s,
			   const struct list_head *mem_map);
};
```

简单概括一下

- `allocate()`和`free()`分别用来从heap中**分配**或者**释放**一个ion_buffer对象
- 连续的物理内存，`phys()`用来得到`ion_buffer`对象的**物理内存**地址和**大小**
- `map_kernel()`和`unmap_kernel()`分别用来把物理内存映射到内核虚拟地址空间
- `map_user()`用来把物理内存映射到用户空间，当用户空间的文件描述符被释放的时候会自动取消映射，所以没有提供类似于`unmap_user()`的函数

用户空间进程使用ION的情形

![](https://s2.ax1x.com/2020/03/10/8CTnOS.png)

##### **主要数据结构**

在整个Android操作系统中，可以发现，比较好的框架，先去了解数据结构，就能猜到是如何实现的整个架构。

**ion_device**

`ion_device`是ION驱动的最基础的一个对象，用来描述一个ion设备，其实在一个Android操作系统中，一般只有一个实例存在，用来统一管理ION Heaps，由CPU厂商自己的代码来创建。

```c
struct ion_device {
	struct miscdevice dev;
	struct rb_root buffers;
	struct mutex buffer_lock;
	struct rw_semaphore lock;
	struct plist_head heaps;
	long (*custom_ioctl)(struct ion_client *client, unsigned int cmd,
			     unsigned long arg);
	struct rb_root clients;
	struct dentry *debug_root;
	struct dentry *heaps_debug_root;
	struct dentry *clients_debug_root;
	struct semaphore vm_sem;
	atomic_t page_idx;
	struct vm_struct *reserved_vm_area;
	pte_t **pte;
	...
};
```

`struct ion_device`是ION的核心结构体，但是在用户空间是不可见的，是由CPU厂商的代码通过`ion_device_create()`函数分配、初始化。

- **dev成员**，是一个`struct miscdevice`类型的杂项字符设备，用来连同内核和用户空间，open/ioctl
- **heaps成员**，用来描述所有的`struct ion_heap`实例
- **clients成员**，用来管理所有`struct ion_client`

**ion_client**

`struct ion_client`是由`ion_client_create()`创建，是通过`struct ion_device`来创建的。

```c
struct ion_client {
	struct rb_node node;
	struct ion_device *dev;
	struct rb_root handles;
	struct idr idr;
	struct mutex lock;
	const char *name;
	char *display_name;
	int display_serial;
	struct task_struct *task;
	pid_t pid;
	struct dentry *debug_root;
};
```

- **node成员**，用于将`struct ion_client`实例加入到`struct ion_device`中`ion_device::clients`
- **device成员**，指向设备的`ion_device`
- **handles成员**，是一个红黑树的根，用来管理它所拥有的handle，即`struct ion_handle`实例。一个`struct ion_handle`实例代表一个buffer。`ion_buffer`是从`struct ion_heap`中分配的。

**ion_heap**

`struct ion_heap`表示ION中的heap，被ion_device中的`ion_device::heaps`所管理

```c
struct ion_heap {
	struct plist_node node;
	struct ion_device *dev;
	enum ion_heap_type type;
	struct ion_heap_ops *ops;
	unsigned long flags;
	unsigned int id;
	const char *name;
	struct shrinker shrinker;
	struct list_head free_list;
	size_t free_list_size;
	spinlock_t free_lock;
	wait_queue_head_t waitqueue;
	struct task_struct *task;

	int (*debug_show)(struct ion_heap *heap, struct seq_file *, void *);
};
```

**ion_handle**

`struct ion_handle`其实就是buffer，用户永健用它来表示自己的buffer，通过`ion_handle_create()`函数分配获得实例

```c
struct ion_handle {
	struct kref ref;
	struct ion_client *client;
	struct ion_buffer *buffer;
	struct rb_node node;
	unsigned int kmap_cnt;
	int id;
};
```

- **ref成员**，是`struct kref`结构体类型，在kernel中被广泛的用来表示引用计数，用来记录handle被引用的次数，当引用计数为0时自动销毁。
- **client成员**，指向这个handle所属的client
- **buffer成员**，指向真正的buffer

**ion_buffer**

这个结构体很重要，通过ION分配的内存就是通过它表示的。

```c
struct ion_buffer {
	struct kref ref;
	...
	struct ion_device *dev;
	struct ion_heap *heap;
	unsigned long flags;
	unsigned long private_flags;
	size_t size;
	union {
		void *priv_virt;
		ion_phys_addr_t priv_phys;
	};
	struct mutex lock;
	int kmap_cnt;
	void *vaddr;
	int dmap_cnt;
	struct sg_table *sg_table;
	struct page **pages;
	struct list_head vmas;
	struct list_head iovas;
	...
};
```

- **ref成员**，是`struct kref`结构体实例，维护了本 `ion_buffer` 的引用计数。当引用计数为 0 时会释放该 buffer，即`struct ion_heap_ops::free `会被调用。分配用 `ION_IOC_ALLOC` 型 ioctl 系统调用，相应的释放用 `ION_IOC_FREE` 型 ioctl 系统调用。
- **size 成员**，当然是本 buffer 所表示的空间的大小，用字节表示。
- **priv_virt 成员**，是所分配内存的虚拟地址啦，它常与 `struct sg_table`，或者封装它的结构，有关。它不是我们在内核中读写时所需的内核虚拟地址啦，内核虚拟地址使用 vaddr 成员来表示的。一般而言，物理内存不连续的，使用本字段；否则使用下面的 priv_phys 字段，如 `struct ion_heap_ops contig_heap_ops`。
- **priv_phys 成员**，表示所分配的内存的物理地址。它适用于分配的物理内存是连续的 ion heap。这种连续的物理内存：在将其映射到用户空间时，即获取用户空间虚拟地址，可以使用 `remap_pfn_range()` [memory.c] 这个方便的接口；在将其映射到内核空间时，即获取内核虚拟地址，可以使用 `vmap()` [vmalloc.c] 这个方便的接口。例子详见 `struct ion_heap_ops contig_heap_ops [exynos_ion.c]`。priv_virt 成员和 priv_phys 成员组成了一个联合体，其实都表示地址，只不过不同的场景下具体用的不一样而已。
- **kmap_cnt 成员**，记录本 buffer 被映射到内核空间的次数。
- **vaddr 成员**，是本 buffer 对应的内核虚拟地址。当 kmap_cnt 不为 0 时有效。可以通过 `ion_map_kernel()` [ion.c] 来获取本 buffer 对应的内核虚拟地址。`ion_map_kernel() `[ion.c] 实际上调用的是相应 `struct ion_heap_ops::map_kernel` 回调函数获取相应的虚拟地址的。
- **dmap_cnt 成员**，记录本 buffer 被 mapped for DMA 的次数。
- **sg_table 成员**，是 `struct sg_table` 结构体类型的指针。本字段与 DMA 操作有关，而且仅仅在 dmap_cnt 成员变量不为 0 时是有效的。可以通过 `ion_buffer_create()` [ion.c] 来初始化本成员变量，该函数实际上是调用相应 ion_heap 所属的 `struct ion_heap_ops::map_dma` 回调函数获取本字段的值的。
- **dirty 成员**，表示 bitmask。即以位图表示本 buffer 的哪一个 page 是 dirty 的，即不能直接用于 DMA。dirty 表示 DMA 的不一致性，即 CPU 缓存中的内容与内存中的实际内容不一样。

![](https://s2.ax1x.com/2020/03/10/8CHBxx.png)

用一张图来展示这些数据结构之间的基本关系。后面一起来看看如何在用户空间和内核空间来使用ION 共享内存机制。
