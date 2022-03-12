---
title: 巧妙的kfifo
date: 2021-12-18 22:18:51
categories:
- Linux
- Kernel
tags:
- Linux
- kfifo
- RingBuffer
- FIFO
cover: /img/kfifo.jpg
---

在编写一个字符驱动的时候想用一个`Ring Buffer`的结构来做数据的读取/写入，自己去设计了一个`RingBuffer`的数据结构，然后使用一个`read/write`的指针来指向读取和写入`buffer`的地址，由于是`ring buffer`所以会考虑到写入超过size的时候回到起始地址的操作。

写完之后偶然发现Linux Kernel中有现成的kfifo实现，拜读了一下代码之后，发现自己的代码太low了，原来Linux Kernel的`kfifo`才是大神级别的代码，简洁且美观，没有一行多余的代码。

<!--more-->

下面我们来分析一下Linux Kernel的`kfifo`是如何实现和工作的。

### kfifo概述

kfifo是内核里面的一个First In First Out数据结构，它采用唤醒循环队列的数据结构来实现，但是它提供的是一个无边界的字节流，最重要的一点是，它使用并行无锁编程技术，也就是当它用于只有一个入队线程和一个出队线程的场景时，两个线程可以并行操作，不需要加锁就可以保证kfifo的线程安全。

OK，既然那么多特点，先来看数据结构：

```c
struct kfifo {
    unsigned char *buffer;    /* the buffer holding the data */
    unsigned int size;    /* the size of the allocated buffer */
    unsigned int in;    /* data is added at offset (in % size) */
    unsigned int out;    /* data is extracted from off. (out % size) */
    spinlock_t *lock;    /* protects concurrent modifications */
};
```

这是kfifo的数据结构，kfifo主要提供了两个操作，`__kfifo_put`(入队操作)和`__kfifo_get`(出队操作)。

- `buffer`: 用于存放数据的缓存
- `size`: buffer空间的大仙，在初始化的时候，会将它向上扩展成2的幂
- `lock`: 如果使用不能保证任何时间最多只有一个读线程和写线程，需要使用该锁来实施同步
- `in/out`:和buffer一起构成一个循环队列，in指向buffer中的队头，out指向buffer中的队尾

```c
+-------------------------------------------------------------------+
|            |<-------------data------------>|                      |
+-------------------------------------------------------------------+
             ^                               ^                      ^
             |                               |                      |
            out                             in                     size
```

内核的开发者使用了一种更好的计数处理了in,out和buffer的关系，这也是为什么我一开始说我看了内核中的实现发现自己写的代码low的原因。

### kfifo功能描述

对外主要提供的功能如下

1. 只支持一个读者和一个读者并发操作
2. 无阻塞的读写操作，如果空间不够，则返回实际访问空间

### kfifo_alloc分配kfifl内存和初始化工作

```c
struct kfifo *kfifo_alloc(unsigned int size, gfp_t gfp_mask, spinlock_t *lock)
{
    unsigned char *buffer;
    struct kfifo *ret;

    /*
     * round up to the next power of 2, since our 'let the indices
     * wrap' tachnique works only in this case.
     */
    if (size & (size - 1)) {
        BUG_ON(size > 0x80000000);
        size = roundup_pow_of_two(size);
    }

    buffer = kmalloc(size, gfp_mask);
    if (!buffer)
        return ERR_PTR(-ENOMEM);

    ret = kfifo_init(buffer, size, gfp_mask, lock);

    if (IS_ERR(ret))
        kfree(buffer);

    return ret;
}
```

这里有一个地方需要关注的是，`kfifo->size`的值总是把传入进来的`size`的基础上向2的幂扩展，这里是内核的一贯做法，这样子做的好处可以对`kfifo->size`取模运算转化为与运算，如下

```c
kfifo->in % kfifo->size可以转化为kfifo->in & (kfifo->size - 1)
```

与运算的计算量可比取模运算少的多了，这样子可以极大的提高使用`kfifo`的性能。

这里使用了`roundup_pow_of_two`来将之向上扩展为2的幂，有兴趣的可以去看一下这个函数的实现。

### __kfifo_put和_kfifo_get巧妙的入队和出队

`__kfifo_put`是入队操作，它先将数据放入`buffer`里面，最后才去修改in参数；`__kfifo_get`是出队操作，它先将数据从`buffer`中移走，最后才去修改`out`，这样子`in`和`out`就各司其职

具体代码如下：

```c
unsigned int __kfifo_put(struct kfifo *fifo,
             unsigned char *buffer, unsigned int len)
{
    unsigned int l;

    len = min(len, fifo->size - fifo->in + fifo->out);
    /*
     * Ensure that we sample the fifo->out index -before- we
     * start putting bytes into the kfifo.
     */
    smp_mb();
    /* first put the data starting from fifo->in to buffer end */
    l = min(len, fifo->size - (fifo->in & (fifo->size - 1)));
    memcpy(fifo->buffer + (fifo->in & (fifo->size - 1)), buffer, l);
    /* then put the rest (if any) at the beginning of the buffer */
    memcpy(fifo->buffer, buffer + l, len - l);
    /*
     * Ensure that we add the bytes to the kfifo -before-
     * we update the fifo->in index.
     */
    smp_wmb();
    fifo->in += len;

    return len;
}
```

这段代码十分简洁，使用`min`来替代`if-else`的分支判断，我们来稍作分析。

```c
len = min(len, fifo->size - fifo->in + fifo->out);
```

这段代码是计算可以放入多少数据，把入参len和剩余空间(`fifo->size - fifo->in + fifo->out`)做比较，可以看一下下面的图来做分析：

```c
+-------------------------------------------------------------------+
|            |<-------------data------------>|                      |
+-------------------------------------------------------------------+
             ^                               ^                      ^
             |                               |                      |
            out                             in                     size
```

认真的读一下下面两行代码：

```c
    /* first put the data starting from fifo->in to buffer end */
    l = min(len, fifo->size - (fifo->in & (fifo->size - 1)));
    memcpy(fifo->buffer + (fifo->in & (fifo->size - 1)), buffer, l);
    /* then put the rest (if any) at the beginning of the buffer */
    memcpy(fifo->buffer, buffer + l, len - l);
```

我们假设入队的指针到buffer的size这段数据的大小小于了要写入的数据长度，那么我们就要`memcpy`两次，第一次是尾部剩余的大小，第二次是头部要写入的剩余buffer。

我们可以看到最后in的长度是直接增加len，而不是去和size取模，配合size是2的次幂的特性接着往下看。

根据这个假设就要计算尾部剩余长度，看一下下面的图就明了了

```c
                                             kfifo_put(写)空间开始地址
                                             |
                                            \_/
XXXXXXX|                                     |XXXXXXXXXXXXXXXXXXXXXXX
+-------------------------------------------------------------------+
|            |<-------------data------------>|                      |
+-------------------------------------------------------------------+
             ^                               ^                      ^
             |                               |                      |
          out%size                        in%size                  size
       ^
       |
 写空间结束地址
```

第二种情况是写入的buffer长度小于尾部剩余空间，那么其实是不需要调用第二个`memcpy`的，也就是第二个`memcpy`的第三个参数会是负数，`memcpy`遇到拷贝负数的时候也是什么都不会去做的，也不会去报错。

同理查看`__kfifo_get`函数实现

```c
unsigned int __kfifo_get(struct kfifo *fifo,
             unsigned char *buffer, unsigned int len)
{
    unsigned int l;

    len = min(len, fifo->in - fifo->out);
    /*
     * Ensure that we sample the fifo->in index -before- we
     * start removing bytes from the kfifo.
     */
    smp_rmb();
    /* first get the data from fifo->out until the end of the buffer */
    l = min(len, fifo->size - (fifo->out & (fifo->size - 1)));
    memcpy(buffer, fifo->buffer + (fifo->out & (fifo->size - 1)), l);
    /* then get the rest (if any) from the beginning of the buffer */
    memcpy(buffer + l, fifo->buffer, len - l);
    /*
     * Ensure that we remove the bytes from the kfifo -before-
     * we update the fifo->out index.
     */
    smp_mb();
    fifo->out += len;

    return len;
}
```

那么还有一个问题，`in/out`的数据类型是`unsigned int`的，如果一直不断的写入读出，迟早会达到`unsigned int`的上限，这里也是巧妙的使用了C语言中达到最大值之后会绕回的特点，不会再去做取模的操作。而且始终满足如下特点：

```c
kfifo->in - fifo->out <= kfifo->Size
```

即使`kfifo->in`回绕到0的一端，这个性质仍然是保持的

### kfifo_get和kfifo_put无锁并发操作

上面说了kernel中kfifo的设计可以使得在单一线程读和单一线程写的使用场景下不需要加锁的妙处，我们上面看到了代码，get和put的操作只是更新自己的in和out数据，各不相干。

```c
|<--写入-->|
+--------------------------------------------------------------+
|                        |<----------data----->|               |
+--------------------------------------------------------------+
                         |<--读取-->|
                         ^                     ^               ^
                         |                     |               |
                        out                   in              size
```

为了避免读者看到**写者预计写入，但实际没有写入数据**的空间，写者必须保证以下的写入顺序

1. 往[`kfifo->in,kfifo>in + len`]空间写入数据
2. 更新`kfifo->in`指针为`fifo->in + len`

在操作1完成时，读者是还没有看到写入的信息的，因为`kfifo->in`没有变化，认为读者还没有开始写操作，只有更新`kfifo->in`之后，读者才能看到。

那么如何保证1必须在2之前完成，秘密就是使用内存屏障：`smp_mb()，smp_rmb(), smp_wmb(),` 来保证对方观察到内存操作顺序。

### 总结

Linux kernel中这种奇妙的代码还是很多的，适合好好深入去读一下，有的时候没写特性我们是知道的，但是如何巧妙的用在合适的地方，才是真正牛X的程序员应该去做的事情。