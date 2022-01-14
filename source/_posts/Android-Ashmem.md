---
title: Android匿名共享内存ashmem
date: 2022-01-03 09:08:10
categories:
- Android
- Ashmem
tags:
- 内存
- ashmem
- 共享内存
---

## 1. 进程间通信之共享内存

Android匿名共享内存—ashmem，其本质是一种共享内存实现方案，而共享内存是操作系统实现进程间通信(IPC)的一种手段。众所周知，binder驱动是Android系统运用最广泛的一种IPC，那么为什么还需要ashmem呢？我么将在文章的最后给出答案，在此之前，我们将其作为一种共享内存机制来说明其实现原理。

先来看一下共享内存的原理。

<!--more-->

### 1.1 什么是虚拟内存

虚拟内存—vm，不是一块实际存在的内存，为了环节物理内存不够用的问题，往往从磁盘空间中开辟一块空间作为物理内存的扩展，这块开辟的磁盘空间称之为交换区—swap。

但是我们都知道，CPU是不能直接访问磁盘空间的，所有的数据都必须先从磁盘空间读入物理内存之后才能被CPU访问，所以swap的工作方式并不是简单的当物理内存用完之后直接从交换区读取数据，而是有一个换出(swap out)换入(swap in)的过程。

具体来说，根据局部访问原理，我们总是认为物理内存中存储的总是最常访问的数据，所以当我们要访问数据时，大多数时候可以直接从物理内存中访问到。但是，当访问数据不在物理内存中时，内核需要将数据从磁盘载入物理内存(换入)，同时将最近不常使用的数据从内存中暂存到磁盘交换区以保留出足够的空间来存储换入的数据(换出)。

这样，即使我们需要运行的程序和数据量超过了物理内存的上限，但是我们仍然感觉内存空间比实际物理内存要大，而这块最大的内存并不是因为实际物理内存有那么大，而是通过动态的换入换出过程虚拟出来的。由于实际不是一块真实的内存，所以将其称为虚拟空间更为合适，这个虚拟空间的大小为：

**vm = 物理内存 + swap**

对于进程来说，它能看到的是虚拟空间，所以虚拟空间需要进行统一编址。一个进程在访问一个虚拟内存页的时候，实际是要去访问其对用的物理内存页，所以虚拟地址和物理地址之间需要一张映射表去根据虚拟地址查找到对应的物理地址，这个工作由内存管理单元—MMU(一般集成在CPU中)来完成。如果MMU映射到的物理页面存在，则直接访问，否则触发缺页终端，需要启动磁盘执行换入换出操作，重新简历该页的映射关系。

![ashmem-1.drawio.png](Android-Ashmem/ashmem-1.drawio.png)

### 1.2 什么是内存共享

通常情况下，各个进程的内存空间是相互独立的，这不光是指其虚拟空间独立，同时也意味着其对应的物理空间也是独立的。这样，一个进程是不能访问到另外一个进程的内存空间的，如下所示：

![ashmem-2.drawio.png](Android-Ashmem/ashmem-2.drawio.png)

为了简单器件省略了MMU部分，并且我们把物理内存空间统一标识为DRAM + swap，即物理内存和交换区，隐藏不必要的缺页细节。

若两个进程需要通过内存共享来实现IPC，那么两个进程的虚拟空间仍然是独立的，而物理空间则出现重叠，即来自两个进程不同的虚拟页面共同指向了物理空间的同一个页面，如下：

![ashmem-3.drawio.png](Android-Ashmem/ashmem-3.drawio.png)

这样一来，V1和V2两个页面共同指向了物理空间页面，当进程1往V1中写入数据的时候，实际就是往共享内存中写入数据，而进程V2读取到的也是共享内存中的数据。这样便实现了共享内存IPC。

像这样共享的物理页面，根据应用的不同，有两种织带：一是指相同的只读数据，不如两个不同的进程都从磁盘载入相同的可运行代码，为了节省内存，在物理空间只需要维护一个副本；二是手动创建一个新的内存映射，将同一部分物理页面分别映射到不同进程的虚拟空间中，其核心的实现往往需要借助`mmap`系统调用完成，而这才是共享内存最广泛的应用。

### 1.3 基于mmap实现的内存共享

`mmap`系统调用将创建一个物理空间与调用进程虚拟空间之间的新映射。下面是`mmap`的函数定义：

```cpp
void *mmap(
    void* addr, 
    size_t length, 
    int prot, 
    int flags, 
    int fd, 
    off_t offset
);
```

- `addr` 参数指定映射被放置的虚拟地址，实际上内核只将其作为参考，尽可能的进行匹配，可以传 `NULL` 完全由内核决定。
- `length` 参数指定了映射的字节数，会被向上提升为一个分页大小的下一个倍数，因为映射大小是以页作为单元的。
- `prot` 参数是一个位掩码组合（用 OR 组合），指定了映射上的保护信息（可读、可写、可执行）。
- `flags` 参数是映射控制码。若为 `MAP_PRIVATE` 表示私有映射，映射只发生在当前进程中，变更只对当前进程可见，对于文件映射，变更也不会修改相应的底层文件；若为 `MAP_SHARED` 表示共享映射，即对相同映射都设置了 `MAP_SHARED` 的不同进程，变更对所有进程都是可见的，对于文件映射，其变更将反映到底层文件；若为 `MAP_ANONYMOUS` 表示匿名映射。
- `fd` 参数表示文件描述符，如果需要将一个文件所表示的区域映射到进程虚拟空间时，可以设置一个有效的文件描述符。
- `offset` 只对于文件映射有效，表示文件映射区域的起始位置。

`mmap`系统调用成功后，将返回新映射的起始虚拟地址。

我们把基于`mmap`实现共享内存的讨论范围进行一下约定：

- 由于`MAP_PRIVATE`私有映射往往用于内存分配场景，不具备共享内存的功能，所以我们只考虑`MAP_SHARED`共享映射。
- 另外Linux自带的匿名映射会忽略`fd`参数，此时`fd`可以设置为-1，那么基于`MAP_SHARED|MAP_ANONYMOUS`和`fd=-1`就可以实现匿名共享内存。但是这样的共享内存只限定在具有继承关系和相关进程之间。比如父进程通过`mmap`返回一个匿名共享内存映射地址，然后通过`fork`系统调用产生一个子进程，子进程将继承父进程的映射地址，由此可以实现父子进程间的内存共享。而我们讨论的不仅仅限制在相关进程之间，所以Linux原生基于`mmap`的匿名共享内存也不加以深究。

**以上，我们只讨论基于mmap的共享文件映射。**

我们先来理解一下Linux下的设备，因为这里讨论的文件，都是依赖于具体的设备的。

**Linux设备**

Linux下有两种类型的设备：字符设备和块设备。字符设备每次智能处理一个字符，比如终端和键盘；块设备每次处理一块数据，比如磁盘设备。另外，设备可以是实际存在的物理实体设备，也可以是虚拟出来的设备，对于操作系统来说，不管是实体设备还是虚拟设备都提供统一的设备处理接口，都是抽象的设备。

在Linux下，一切皆文件，于上层来说，简化了接口，隐藏了细节。设备也不例外，每个设备都被抽象成一个文件，叫做设备文件。每个设备都对应`/dev`目录下的一个文件节点，且都有一个主设备号和次设备号。主设备号相同的设备（文件），通常使用相同的驱动程序来处理对设备I/O请求，这些驱动程序都有对应的系统调用（比如`open`、`close`、`read`、`write`、`mmap`以及`ioctl`）。

举个例子，比如主设备号为8的设备是SCSI磁盘设备，而磁盘设备根据分区又分为很多子设备，其文件名均以`/dev/sd**`开头，每个子设备都通过不同的次设备号来区分。那么当对一个普通文件进行`open`、`read`、`mmap`等操作时，最终都关联到主设备号为8的设备驱动程序来处理。

在系统初始化的时候，每个设备都会被注册进内核，同时设置该设备的驱动程序。此外，内核中有一张常驻的`inode`表，里面记录了所有已创建文件的基本信息和索引信息，每个文件对应一个`inode`结构。就磁盘设备来说，当创建一个普通磁盘文件的时候，会在内核中为该文件增加一个`inode`结构，特别的，会初始化`inode`结构中的属性`i_fop`，他是一个`file_operations`结构体，其中记录了所有的设备驱动函数。

所以，理论上，对于一个文件进行的所有I/O操作，都可以定向到该文件在内核中的`inode`结构中指向的其对应设备的驱动程序来处理。这不光适用于普通文件，甚至也适用于设备文件本身，因为设备文件也是一个文件，在内核中也会维护一个`inode`结构。

所以我们看Linux下的设备和文件有一个有趣的现象：**设备皆文件，文件隶属于设备**。前者简化设备的操作，后者是文件I/O驱动特点使然。

下面看主要看一下`mmap`的执行过程

**mmap执行文件映射过程**

1. **执行open系统调用**，打开一个文件，初始化一个与该文件相关联的file结构体以及指向该结构体的文件描述符fd，fd和file作为调用进程文件描述符表的一个新项被插入。

![ashmem-4.drawio.png](Android-Ashmem/ashmem-4.drawio.png)

几个要点：

open的作用就是根据inode表初始化文件的一个file结构体，然后将其记录在内核为该进程维护的文件描述符表中，而用于索引file的文件描述符fd则由进程用户空间维护。

`file`表示一个打开的文件，里面记录的信息是动态的，比如读写位置变化等；`inode`表示一个文件的基本信息，是静态不变的。

值得注意的是，`file`中也有一个`file_operations`的属性`f_op`，是基于inode中的`i_fop`初始化的，所以打开后的文件，执行后续的I/O操作都可以直接通过file中的`f_op`定向到其对应的设备驱动函数，比如`mmap`。

1. **执行mmap系统调用**，在调用进程用户空间中开辟一块指定大小的虚拟内存快vma，并将vma与上一步打开的文件进行关联，即将文件指定区域映射到虚拟空间中。

![ashmem-5.drawio.png](Android-Ashmem/ashmem-5.drawio.png)

从图中可以看到，`mmap`只是简单的将虚拟空间和文件指定区域简历了对应关系，这在逻辑上表示以后可以像读写虚拟内存vma一样读写文件的指定区域。但是实际上，还并没有为文件上该块区域分配对应的物理内存。

在Linux中，给进程分配的虚拟空间vma由`vm_area_struct`结构体来表示，该结构体中有一个成员属性vm_file，他指向的就是被映射的文件的file结构。

逻辑上来讲，mmap系统调用完成三个步骤：

1. 根据要映射的空间大小创建并初始化虚拟内存结构vma，表示分配了一块虚拟内存
2. 根据传递给mmap的fd从当前进程的文件描述符表中找到对应的file结构体
3. 调用`file->f_op->mmap(file*, struct vm_area_struct*)`定向到mmap驱动函数完成映射过程

而驱动函数`mmap(file*, struct vm_area_struct*)`的主要任务就是把一个合适的文件指定到`vma->vm_file`上，以完成映射。为什么说是合适的文件呢？因为实际映射的文件有可能是系统调用`mmap`中传递的fd指向的文件（如普通磁盘文件）；而对于某些设备文件往往不是直接映射为该设备文件，因为这些设备文件可能并不具备直接存取数据的能力，而是会单独映射到一个新建的可进行存取的文件中，比如从内存文件系统`tmpfs`中申请一个文件。而Android匿名共享内存恰好就是采用的第二种映射方案，后面会详细描述。

既然mmap并没有为映射文件分配物理内存，那么如何对映射的文件区域进行访问呢？这就要提到前面说得MMU转义和缺页中断机制了：

![ashmem-6.drawio.png](Android-Ashmem/ashmem-6.drawio.png)

如图所示，`mmap`会返回虚拟空间的起始地址addr，这实际也是分配的vma链表中的第一个vma块的起始地址，即`vma->vm_start`。对于调用进程而言，可以直接往addr指向的虚拟地址进行读写，而不同通过read/write系统调用来读写映射的文件了。

- 首先，向addr指向的虚拟地址读写数据
- 由MMU将addr转义成具体的物理地址，但是因为没有为文件分配物理空间，无法找到对应的物理地址，发生缺页，调用缺页中断程序。
- 缺页中断先试图从swap中寻找对应的页面，如果找到，说明文件被载入过且因为其它页的换入操作被换出过了，于是直接从swap中换入缺失的页面到物理页。
- 若swap中也没有对应的页面，说明文件从未被载入过，于是从磁盘空间载入指定文件的区域到物理内存。
- 返回物理内存地址，实现虚拟页到屋里也的映射关系。

以上，缺页发生后无论是从swap中载入缺页还是从文件中载入缺页，物理内存都可能已满，这时缺页机制会执行换出操作，将其它最近最不常使用的页面换出到swap分区。

到目前为止，我们已经知道了`mmap`实现共享文件映射的过程，只不过只分析了一个进程在共享模式下映射一个文件的原理，那么要实现两个进程的内存共享，我们也不难分析其过程：

1. 假设要映射的文件名为`fname`，一般是一个路径，共享区域大小为`len`
2. 进程1对文件执行`open`和`mmap`系统调用：

```cpp
int fd1 = open(fname, ...);
void* addr1 = mmap(NULL, len, ..., MAP_SHARED, fd1, 0);
```

1. 进程2同样对文件执行open和mmap系统调用：

```cpp
int fd2 = open(fname, ...);
void* addr2 = mmap(NULL, len, ..., MAP_SHARED, fd2, 0);
```

这样，进程1中的`addr1`表示的虚拟地址和进程2中的`addr2`表示的虚拟地址都指向了同一块相同的物理内存区域，于是便实现了进程1和进程2的内存共享。

需要特别注意的是，进程1打开文件返回的文件描述符`fd1`只对进程1有效，而不能直接给进程2使用，因为不同的进程都有自己独立的文件描述符表。在进程1中，`open`之后，会给进程1的文件描述符表中分配一个表项，并生成一个可用的文件描述符`fd1`，然后指向文件结构体`file`。这时，进程2中如果没有打开过`fname`，则其文件描述符表中根本没有`fname`的表项，还没有为其分配可用的文件描述符，如果直接把`fd1`拿过来用要么找不到`fname`的`file`结构体，要么找到的根本不是`fname`的file结构体，从而产生不可预知的错误。这就是为什么在进程2中也需要对`fname`先执行一遍`open`调用了。

## 2 Linux POSIX共享内存结构shm_open

在Linux POSIX标准中有一个专用的共享内存函数：`shm_open()`。如果我们使用共享内存进行IPC时没有数据持久化的需求，则可以使用`shm_open`来实现共享内存。实际上`shm_open`对应的是我们上一届实现共享内存过程的`open`系统调用，所以其实现共享内存的过程也类似，即先调用`shm_open`打开一个共享文件并返回文件描述符，然后调用`mmap`完成映射。

通常情况下`shm_open`只是对`open`进行了一下简单的改造，下面是实现源码：

```cpp
int
shm_open (const char *name, int oflag, mode_t mode)
{
  SHM_GET_NAME (EINVAL, -1, "");
  // ... 省略
  int fd = open (shm_name, oflag, mode);
  // ... 省略
  return fd;
}
```

可以看到，内部实际上也是通过系统调用`open`来实现的，其不同之处在于指定打开的文件是定制的，说明打开的一个特殊的文件。注意里面的`shm_name`变量，该变量的初始化用宏定义`SHM_GET_NAME`获取，其实际作用就是给参数`name`加上一个共享文件目录的前缀：`/dev/shm/`

`/dev/shm/`有何特别之处呢？其表示的实际上是Linux的临时文件系统tmpfs。`shm_open`一个文件，相当于就在tmpfs文件系统中打开了一个文件。

### 2.1 tmpfs临时文件系统

tmpfs从功能上来说，正如其名，即存储在tmpfs上的数据都是暂时的、临时的，这意味着它与内存有着相同的特性：临时存储。这样的数据没有持久存储的需求，实际上，tmpfs文件系统中的数据就是存储在物理内存和swap分区，它与虚拟空间的物理指向很相似，只不过是以文件系统的方式来管理的。

当在tmpfs下创建一个文件后，在该文件上进行读写操作的性能非常高，因为就是直接操作的内存，无需进行I/O，虽然在物理内存满的时候也需要使用swap分区进行换入操作，但总体性能还是比直接操作普通文件快很多。

所以，当我们使用`shm_open`实现共享内存的时候，理论上就相当于直接将一块物理内存映射到了进程虚拟空间中，而很多基于内存共享的IPC是没有数据持久化的需求的，`shm_open`将是一个实现内存共享的首选。

Android ashmem共享内存也是基于tmpfs文件系统来实现的，与`shm_open`方案有异曲同工之妙。

## 3 Android ashmem共享内存原理

前面介绍了那么多Linux共享内存原理及相关实现原理，都是为了本节分析Android匿名共享内存做铺垫，因为Android匿名共享内存（ashmem）的底层逻辑与前述Linux共享内存的实现原理是一致的，只是在实现细节上结合了一些Android框架的特性以及自带锁保护机制实现共享内存的同步访问。

**ashmem以mmap实现共享文件映射，以tmpfs临时文件系统作为共享文件，以binder驱动作为进程间传递共享文件句柄的桥梁。**

### 3.1 ashmem设备

**纽带文件：**在前面分析基于共享文件实现共享内存的时候，文件是一个关键点，先由open打开文件，然后再由mmap将文件进行映射，两个通信的进程都以同一个文件作为纽带。如前分析，这里的文件既可以是普通的磁盘文件，也可以是直接是一个设备文件，但是真正被共享的文件却不一定就是传入的这个文件，传入的文件其主要作用是作为通信进程间的纽带，或者说有了这个纽带文件，进行通信的多个进程就能通过这个纽带文件去找到真正的共享文件。

也就是说，从应用程序的监督看`open`和`mmap`系统调用中的文件名和文件描述符，逻辑上来讲可以将纽带文件当成是共享文件，但从物理层面上看纽带文件不一定是共享文件。

ashmem中的纽带文件是一个设备，叫ashmem设备，这是一个虚拟的字符设备，隶属于杂项设备（misc设备，主设备号为10），其文件名为`/dev/ashmem`。Android系统启动是便会对ashmem设备进行注册和初始化，在内核`inode`表中将生成设备的`inode`节点以便后续打开该设备文件所用。一下是初始化关键代码：

```cpp
static int __init ashmem_init(void)
{
    int ret;
    ashmem_area_cachep = kmem_cache_create("ashmem_area_cache",
                      sizeof(struct ashmem_area),
                      , , NULL);
    // ...省略

    ret = misc_register(&ashmem_misc);  

    // ...省略  

    return;                                                         
}                                                                     
                                                                      
static struct file_operations ashmem_fops = {
    .owner = THIS_MODULE,
    .open = ashmem_open,
    .mmap = ashmem_mmap, 
    // ...省略
}; 

static struct miscdevice ashmem_misc = {
    .minor = MISC_DYNAMIC_MINOR,
    .name = "ashmem",
    .fops = &ashmem_fops,
};
```

其初始化中我们重点关注两个部分：第一是ashmem设备的`open`和`mmap`这两个驱动函数，其对应的实现分别是`ashmem_open`和`ashmem_mmap`；第二是`ashmem_area_cachep`这个全局变量，其用于分配ashmem共享内存，准确的说是一个用于表示ashmem共享内存的数据结构，叫做`ashmem_area`。ashmem与上一节描述的Linux共享内存在实现上的差异主要就是体现在这两部分。

### 3.2 ashmem_open

在执行系统调用`open("/dev/ashmem", ...)`之后，最终会进入到驱动函数`ashmem_open`

ashmem设备的open驱动主要完成两件事：

1. 由ashmem_area_cachep分配一块ashmem共享内存asma，asma是一个ashmem_area结构体，用于描述ashmem共享内存：

```cpp
struct ashmem_area {
    char name[ASHMEM_FULL_NAME_LEN]; /* 共享内存名字，记录在 /proc/pid/maps */
    struct list_head unpinned_list; /* 记录所有共享内存块，用于内存管理 */
    struct file *file;        /* 真正的共享文件指针 */
    size_t size;            /* 共享内存大小，字节 */
    unsigned long prot_mask;    /* 控制掩码 */
};
```

当中最重要的一个字段是`file`，这个才是ashmem共享内存中真正的共享文件，从tmpfs文件系统分配。

1. 将asma记录在设备文件的私有数据字段，这样后续只要能够找到ashmem设备文件，就能找到ashma共享内存，就能找到真正的共享文件。

代码如下：

```cpp
static int ashmem_open(
    struct inode *inode, // 表示 ashmem设备文件的 inode 结构体
    struct file *file // 表示 ashmem设备文件的 file 结构体
) {
    struct ashmem_area *asma;

    // ...省略

    // 分配 asma
    asma = kmem_cache_zalloc(ashmem_area_cachep, GFP_KERNEL);

    // ...省略

    // 将 asma 记录在设备文件中
    file->private_data = asma;
    return 0;
}
```

### 3.3 ashmem_mmap

ashmem设备的mmap驱动也主要完成两件事：

1. 向tmpfs文件系统申请创建一个共享文件，并记录在asma共享内存中。

我们在第2节分析POSIX共享内存函数shm_open的原理的时候知道，可以在tmpfs目录（`/dev/shm`）下指定一个文件名来打开并创建一个tmpfs文件。即直接执行`shm_open`系统调用即可。不过这种方法返回的是共享文件的文件描述符，而这里我们不需要通过文件描述符再去访问文件，而是直接指定共享文件的file结构体就可以了。所以，`ashmem_mmap`函数直接调用Linux内核共享文件创建函数`struct file* shmem_file_setup(const char *name, loff_t size, unsigned long flags)`来从tmpfs文件系统创建一个共享文件，并直接返回共享文件的file结构体。

1. 将共享文件映射到`mmap`系统调用分配的虚拟空间中。

我们在1.3节中曾经解释过，设备驱动函数`mmap`的一个主要作用就是把一个合适的文件指定到`vma->vm_file`上，已完成映射。这里，合适的文件便是指的上一步创建的tmpfs共享文件。

```cpp
static int ashmem_mmap(
    struct file *file, // ashmem设备文件
    struct vm_area_struct *vma // mmap系统调用分配的虚拟空间
) {
    // 从设备文件取共享内存
    struct ashmem_area *asma = file->private_data;
    int ret = 0;

    // 互斥锁保护
    mutex_lock(&ashmem_mutex);

    // ...省略

    if (!asma->file) { // 共享文件只需要设置一次
        char *name = ASHMEM_NAME_DEF;
        struct file *vmfile;

        if (asma->name[ASHMEM_NAME_PREFIX_LEN] != '\0')
            name = asma->name;

        // 向 tmpfs文件系统申请并创建一个共享文件，asma->size 必须提前通过 ioctl 设置，过程略
        vmfile = shmem_file_setup(name, asma->size, vma->vm_flags);

        // ...省略

        asma->file = vmfile;
    }

    // ...省略

    if (vma->vm_flags & VM_SHARED)
        // 共享文件映射，主要功能是指定 vma->vm_file = asma->file
        shmem_set_file(vma, asma->file);
    else {
        if (vma->vm_file)
            fput(vma->vm_file);
        vma->vm_file = asma->file;
    }
    vma->vm_flags |= VM_CAN_NONLINEAR;

out:
    mutex_unlock(&ashmem_mutex);
    return ret;
}
```

### 3.4 ashmem设备在进程间的传递

到目前为止，我们已经分析了单个进程如何实现ashmem文件映射。理论上，根据第一节的分析，要实现ashmem共享内存，只需要两个进程分别对ashmem设备执行一遍`open`和`mmap`系统调用即可。但是前面我们分析了`ashmem_open`每次调用都会分配一个新的`asma`共享内存，所以如果有两个进程都调用了`open(”/dev/ashmem”, ...)`之后，会生成两个设备文件`file`结构体，他们各自拥有独立的`asma`共享内存，也就是说这两个进程之间根本没有共享内存可言：

![ashmem-7.drawio.png](Android-Ashmem/ashmem-7.drawio.png)

其实这样的设置对Android系统而言也是合理的，因为我们要实现一个共享内存方案，不光需要有一组进程可以进行共享内存IPC，而且我们需要实现同时有很多进程组的共享内存IPC，每个进程组之间可以通过共享内存实现IPC，且各个进程组之间相互隔离。

所以，我们如何才能实现两个进程间asma真正的共享呢？也就是说，当一个服务进程ServiceProcess通过open和mmap映射了一块ashmem共享内存的时候，如何荣客户端进程ClientProcess也从内核中取到由ServiceProcess打开的ashmem设备文件结构体ashmem file呢？

ashmem在这里通过binder驱动来实现的，基本原理如下：

- Android进程间普遍通过binder驱动实现IPC，而binder驱动中维护了所有待通信的Android进程，这一点大家要先明白。也就是说，所有通过binder进行IPC的进程，对于binder驱动都是可见的
- 假设Server端进程`SP`经完成了ashmem共享内存的映射，这时binder驱动可以读取到`SP`的ashmem设备文件描述符`sfd`；假设Client进程`CP`想向`SP`请求共享内存通信，并通过binder向`SP`发送这一请求
- binder根据`CP`的进程文件描述符表，计算出一个空闲的文件描述符`cfd`，用于之后索引ashmem设备文件
- binder根据`sfd`从内核取到ashmem设备文件`file`结构体`afile`
- binder将`cfd`和`afile`插入到`CP`的进程文件描述符表中
- `SP`通过binder最终响应`CP`的共享内存通信请求，以binder的方式将`cfd`返回给`CP`

这时，在`CP`的进程描述符表中增加了一项ashmem设备文件的索引项，且其file结构体与SP进程描述附表中的ashmem设备文件`file`结构体指向相同。

最后，`CP`只要执行系统调用`mmap(..., cfd, ...)`便将同一个共享文件映射到了自己的虚拟空间，从而实现了和`SP`的共享内存。

![ashmem-8.drawio.png](Android-Ashmem/ashmem-8.drawio.png)

总结来说，binder做了两件事，一是将Server进程打开的ashmem设备文件，即纽带文件挂载到客户端进程的文件描述符中；二是为Client进程生成新的纽带文件描述符并返回给Client端。

至于为什么ashmem叫匿名共享内存，实际上对于Server进程而言，从打开ashmem设备文件的角度，ashmem设备文件是有文件名的。这里的匿名主要针对的是Client端进程，对Client端进程来说，因为不需要执行open系统调用来得到纽带文件描述符，所以不需要知道纽带文件名（即ashmem设备文件名），而是通过binder将新的纽带文件描述符回传给客户端来实现共享内存。

## 4 Android ashmem的使用场景

ashmem 是基于 tmpfs 文件系统来实现的。在 Linux 下，tmpfs 文件系统的大小默认为物理内存的一半，而 ashmem 在向 tmpfs 申请空间的时候也并没有对申请大小做出特别的限制，同时 tmpfs 又是内存文件系统，所以理论上 ashmem 实现的共享内存 IPC 在时间上不输 binder 通信，在空间上又没有 binder 通信的限制，这很适合在需要传递大量数据的实时通信场景。

一个典型的应用是 Android 窗口刷新。Android 窗口刷新也是基于 CS 模式的。Android 的窗口一般会由几个客户端进程来提供，比如提供顶部状态栏窗口的进程、提供底部虚拟按键栏窗口的进程、以及提供中间主体内容窗口的进程。每个进程都负责屏幕各自区域的图像数据刷新，在 VSYNC 信号驱动下，各个客户端进程会将各自的窗口数据传递给服务端窗口进程来合成，服务端进程将合成的窗口数据刷新到屏幕上：

![ashmem-9.drawio.png](Android-Ashmem/ashmem-9.drawio.png)

这种情况下，窗口图像数据一般都比较大，而且有实时性能要求，而 ashmem 正好满足这两个要求。