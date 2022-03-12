---
layout: blog
title: 'Android ION内存管理(2) -- 共享内存使用'
date: 2019-12-16 21:42:30
categories:
- Android
- ION内存管理
tags:
cover: /img/android-ion.png
---
内存共享和大块内存的使用，在实际场景下面的需求是很多的，这里，举三个简单的应用场景：

- 用户态和内核态共享内存
- 用户态不同进程内存共享
- 内核态中使用ION分配buffer

##### 用户态和内核态共享内存
<!--more-->

在Android的BSP代码中有一个ion的library封装了一些对ion驱动设备操作的接口`system/core/libion/`

```c++
int ion_open();
int ion_close(int fd);
int ion_alloc(int fd, size_t len, size_t align, unsigned int heap_mask,
              unsigned int flags, ion_user_handle_t *handle);
int ion_alloc_fd(int fd, size_t len, size_t align, unsigned int heap_mask,
              unsigned int flags, int *handle_fd);
int ion_sync_fd(int fd, int handle_fd);
int ion_free(int fd, ion_user_handle_t handle);
int ion_map(int fd, ion_user_handle_t handle, size_t length, int prot,
            int flags, off_t offset, unsigned char **ptr, int *map_fd);
int ion_share(int fd, ion_user_handle_t handle, int *share_fd);
int ion_import(int fd, int share_fd, ion_user_handle_t *handle);
```

可以参考改目录下的`ion.c`实现代码，都是对ion驱动设备的open/close/ioctl等操作。

初始化ion获取文件描述符，分配内存，获取内存共享的文件描述符

```c++
void init_ion()
{
    size_t size = HON_ZONE_SIZE;
    int ret = -1;

    ion_fd = ion_open();
    if(ion_fd < 0) {
        ALOGE("Failed to open ion device\n");
        ret = ion_fd;
        goto out;
    }

    ret = ion_alloc(ion_fd, size, 0, (1 << ION_SYSTEM_HEAP_ID),
            ION_FLAG_CACHED,
            &ion_hdl);
    if(ret < 0) {
        ALOGE("Failed to allocate ion buffer\n");
        goto close_fd;
    }

    ret = ion_share(ion_fd, ion_hdl, &shared_fd);
    if(ret < 0) {
        ALOGE("Failed to shared the ion fd\n");
        goto close_ion;
    }
}

void release_ion()
{
    ion_free(ion_fd, ion_hdl);
    close(ion_fd);
}
```

- `ion_open()`函数打开`/dev/ion`设备获取文件描述符，后面对文件的操作都是基于这个文件描述符
- `ion_alloc()`指定分配的buffer大小和类型，需要注意的是buffer的size，在camera等一些应用场景下经常会被要求buffer要4K对齐，返回的是`ion_handler`
- `ion_share()`函数把指定`ion_handler`内存共享出去，获取到内存共享文件描述符，把`shared_fd`发送到另外一个进程中，然后通过`mmap`函数进行内存映射。

通过`mmap`函数把`ion_handle`转化为虚拟地址

```c++
    g_buf = (char *)mmap(NULL, HON_ZONE_SIZE,
            PROT_READ | PROT_WRITE, MAP_SHARED, shared_fd, 0);
    if(g_buf == MAP_FAILED) {
        ALOGE("Failed to mmap buffer\n");
    }
```

本例中需要把buffer传到内核空间去，然后在kernel空间对buffer进行读写，首先需要把`shared_fd`传到kernel空间，然后通过`shared_fd`获取到`ion_handler`最后得到内核虚拟机地址。

使用`ioctl`把shared_fd传到内核空间

```c++
void bind_driver()
{
    int fd = -1;
    int ret;

    fd = open("/dev/hon_zone", O_RDWR);
    if(fd < 0) {
        ALOGE("Failed to open hon_zone device\n");
        return;
    }

    ret = ioctl(fd, HON_ZONE_ION_INIT, &shared_fd);
    if(ret < 0) {
        ALOGE("Failed to do ioctl\n");
        close(fd);
        return;
    }

    zone_fd = fd;
    ALOGD("zone fd = %d", zone_fd);
}
```

本文中自己实现了一个字符驱动`hon_zone`用来配合用户空间的进程读写用户空间分配的ion buffer，用户态使用ioctl把shared_fd传下去，来看kernel是如何处理的：

```c
        case HON_ZONE_ION_INIT:
            ZONE_MTX_LOCK();
            if(copy_from_user(&ion_fd, (int *)argp, sizeof(int))) {
                LOGE("%s: failed to get fd from user", __func__);
                rc = -EINVAL;
                break;
            }
            if(zone->client == NULL) {
                zone->client = msm_ion_client_create("hon_zone");
                if(!zone->client) {
                    LOGE("%s: failed to create ion client", __func__);
                    rc = -ENOMEM;
                    break;
                }
                zone->handle = ion_import_dma_buf(zone->client, ion_fd);
            } else {
                rc = -EBUSY;
                break;
            }

            zone->buffer = ion_map_kernel(zone->client, zone->handle);
            ion_handle_get_size(zone->client, zone->handle, &len);
            zone->size = len;
            zone->is_registered = true;
            LOGD("%s: virtual address is %p", __func__, (void *)zone->buffer);
```

- `copy_from_user`拿到用户态传下来的`shared_fd`
- 创建`ion_client`这里使用的是高通的内核，封装的函数`msm_ion_client_create`获取`ion_client`
- `ion_import_dma_buf`把`shared_fd`转化为`ion_handle`
- 通过`ion_map_kernel`传入`ion_handle`获取到虚拟地址，与用户态的`mmap`函数功能类似

##### 用户态不同进程共享内存

写一个简单的例子介绍如何在不同进程中进行内存共享

![](https://s2.ax1x.com/2020/03/10/8C7yNj.png)

创建`socketpair`，获取两个可以连通的socket文件描述符，这两个文件描述符分别给父子进程使用，一个写一个读，这里其实和使用管道是类似的作用，但是管道是单向的，`socketpair`是双向读写的。

```c++
    int pipefd[2];
    /*创建父子进程管道*/
    int ret = socketpair(PF_UNIX, SOCK_DGRAM, 0, pipefd);
```

`fork`子进程，在子进程中获取ION buffer的共享内存文件描述符

```c++
    int pid = fork();
    if(pid == 0)
    {
        close(pipefd[0]);
        int fd_to_pass = getIONBuffer();
        if(fd_to_pass == -1)
        {
            return -1;
        }
        send_fd(pipefd[1], (fd_to_pass > 0) ? fd_to_pass : 0);
    }
```

获取shared_fd然后通过socket发送给父进程

```c++
int getIONBuffer()
{
    int ion_fd = -1;
    struct ion_fd_data sFdData = {};
    struct ion_handle_data sHandleData = {};
    struct ion_allocation_data sAllocInfo;
    int pagesize = 4096;
    int err;
    int size = 0;
    int bo_width = BO_W;
    int bo_height = BO_H;
    ion_fd=open("/dev/ion", O_RDWR);
    size = AL(bo_width,32) * bo_height * 4;

    char buffer[50] = "qwertysdfgh";
    if (ion_fd < 0) {
        return -1;;
    }

    /* allocate ion buffer */
    sAllocInfo.len = 50*sizeof(char);
    sAllocInfo.align = 50*sizeof(char);
    //sAllocInfo.heap_id_mask  = ION_HEAP_TYPE_DMA_MASK;
    sAllocInfo.heap_id_mask = (1 << 25);
    sAllocInfo.flags = ION_FLAG_CACHED;
    // sAllocInfo.flags |= (1 << 17); // write combined
    err = ioctl(ion_fd, ION_IOC_ALLOC, &sAllocInfo);
    if(err) {
            return -1;
    }
    sFdData.handle = sAllocInfo.handle;
    sHandleData.handle = sAllocInfo.handle;

    err = ioctl(ion_fd, ION_IOC_SHARE, &sFdData);
    if(err) {
            ioctl(ion_fd, ION_IOC_FREE, &sHandleData);
            return -1;
    }

    void *data;
    data = mmap(NULL, sizeof(buffer), PROT_READ | PROT_WRITE, MAP_SHARED, sFdData.fd, 0);
    if (data == MAP_FAILED) {
          printf("!!! Child mmap failed: %m\n");
          return -1;
    }
    memcpy(data,buffer,50*sizeof(char));
    munmap(data,sizeof(buffer));
    err = ioctl(ion_fd, ION_IOC_FREE, &sHandleData);
    if(err) {
            close(sFdData.fd);
            return -1;
    }
    return sFdData.fd;
}
```

- 打开`/dev/ion`获取client文件描述符
- 调用`ioctl`来分配ION buffer获取ion handle
- 利用`mmap`映射虚拟地址，最终操作的是虚拟地址数据
- 返回shared_fd

```c++
void send_fd(int fd, int fd_to_send)
{
    struct iovec iov[1];
    struct msghdr msg;
    char buf[0];

    iov[0].iov_base = buf;
    iov[0].iov_len = 1;
    msg.msg_name = NULL;
    msg.msg_namelen = 0;
    msg.msg_iov = iov;
    msg.msg_iovlen = 1;

    struct cmsghdr cm;
    cm.cmsg_len = CONTROL_LEN;
    cm.cmsg_level = SOL_SOCKET;
    cm.cmsg_type = SCM_RIGHTS;
    *(int*)CMSG_DATA(&cm) = fd_to_send;
    msg.msg_control = &cm; /*设置辅助数据*/
    msg.msg_controllen = CONTROL_LEN;
    sendmsg(fd, &msg, 0);
}
```

通过`socket`进程间通信把shared_fd发送给父进程

父进程中获取共享buffer使用

```c++
        int pagesize = 4096;
        char buff_data[50];
        int c_ion_fd = -1;
        int s_error;
        int bo_width = BO_W;
        int bo_height = BO_H;
        int size = AL(bo_width,32) * bo_height * 4;
        struct ion_fd_data stFdData = {};
        close(pipefd[1]);
        fd_to_pass = recv_fd(pipefd[0]);
        c_ion_fd = open("/dev/ion", O_RDWR);
        stFdData.fd = fd_to_pass;
        s_error = ioctl(c_ion_fd,ION_IOC_IMPORT,&stFdData);
        if(s_error) {
            return -1;
        }
        void *data;
        data = mmap(NULL, sizeof(buff_data), PROT_READ | PROT_WRITE, MAP_SHARED, stFdData.fd, 0);

        memset(buff_data,0,50*sizeof(char));
        memcpy(buff_data,data,50*sizeof(char));
        close(fd_to_pass);
```

- 通过`recv_fd`函数获取到子进程发送过来的shared_fd
- 调用`ioctl`传入shared_fd获取ion handle
- 调用`mmap`映射成可以操作的虚拟地址
- 对数据进行读写操作

可以看到在父子进程中都对`/dev/ion`设备打开，而且在不同的进程中制允许有一个ion_client实例。

```c++
int recv_fd(int fd)
{
    struct iovec iov[1];
    struct msghdr msg;
    char buf[0];

    iov[0].iov_base = buf;
    iov[0].iov_len = 1;
    msg.msg_name = NULL;
    msg.msg_namelen = 0;
    msg.msg_iov = iov;
    msg.msg_iovlen = 1;

    struct cmsghdr cm;
    msg.msg_control = &cm;
    msg.msg_controllen = CONTROL_LEN;

    recvmsg(fd, &msg, 0);

    int fd_to_read = *(int*)CMSG_DATA(&cm);
    return fd_to_read;
}
```

##### 总结

总的来说，在Android中使用ION机制进行共享内存还是很方便的。其中比较有意思的是如何把`shared_fd`传到不同的进程中，本例使用的是`socketpair`机制，在Android 8或以上的版本中，Android官方推荐使用binder机制通过把shared_fd封装成`native_handler`发送给别的进程。

![](https://s2.ax1x.com/2020/03/10/8C764s.png)

有兴趣的可以去尝试一下。