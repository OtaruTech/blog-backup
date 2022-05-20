---
title: Linux性能分析60秒
date: 2022-05-20 23:23:03
categories:
- 系统性能分析
tags:
- System Performance
- 性能分析
cover: /img/system-perf-01.jpg
---
本文翻译自Netflix技术博客：

[Linux Performance Analysis in 60,000 Milliseconds | by Netflix Technology Blog | Netflix TechBlog](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55)

## Frist 60 Seconds：Summary

在本文中，Netfix的性能工程师团队会介绍在一上来的60秒内，在命令行中使用一套优化的性能调研，使用Linux系统中现成的工具。在头60内，你可以运行下面的命令来获取到一个High Level的系统资源利用率和正在运行的进程。首先是最容易理解的，查找一些错误信息和性能饱和度，然后才是资源利用率。饱和度是指一个资源已经超出了他本来应该持有的资源数量，可以理解为一个请求队列的长度，或者等待时间的消耗。

```bash
uptime
dmesg | tail
vmstat 1
mpstat -P ALL 1
pidstat 1
iostat -xz 1
free -m
sar -n DEV 1
sar -n TCP,ETCP 1
top
```

上面有一些工具需要使用到sysstat软件包，在ubuntu系统里面可以使用下面的命令去安装：

```bash
sudo apt install sysstat
```

这些命令给出的一些指标会帮助你完成一些[USE Method](https://www.brendangregg.com/usemethod.html)（过度资源使用和错误方法）：定位性能瓶颈的方法。这会涉及到检查所有资源的利用率，饱和度和错误信息（CPU，内存，磁盘等）。还要注意你是何时检查并释放了资源，因为通过释放的过程可以缩小研究目标，并帮助后续的调查。

以下会对这些命令做一些详细的介绍，也会涉及到一些例子，更多的信息可以查看这些工具的man页面。

### 1. uptime

```bash
$ uptime 
23:51:26 up 21:31, 1 user, load average: 30.02, 26.43, 19.02
```

这是一个快速查看平均负载的方法，并显示了多少个任务（processes）正在等待运行。在Linux系统上，这些包含想要在CPU上运行的进程数和进程被非中断I/O阻塞（通常是磁盘I/O）的进程。这可以给出一个High Level的系统资源的loading，但是没有其他工具的帮助下很难正确的理解系统真正的性能状态。比较值得快速的查看。

这三个数值分别是指在1分钟，5分钟，15分钟内的指数衰减和移动平均值。通过这三个数值可以了解系统loading的变化。举个例子，当你去检查一个服务器的问题时，如果1分钟的值远小于15分钟的值，那可能你登录的太晚了，问题已经不在现场了。

在这个例子中，1分钟的平均负载是30%，相比较15分钟的平均负载是19%，数值的变大意味有一些事情发生：可能的CPU需求；`vmstat`或者`mpstat`可以来确认。

### 2. dmesg | tail

```bash
$ dmesg | tail
[1880957.563150] perl invoked oom-killer: gfp_mask=0x280da, order=0, oom_score_adj=0
[...]
[1880957.563400] Out of memory: Kill process 18694 (perl) score 246 or sacrifice child
[1880957.563408] Killed process 18694 (perl) total-vm:1972392kB, anon-rss:1953348kB, file-rss:0kB
[2320864.954447] TCP: Possible SYN flooding on port 7001. Dropping request.  Check SNMP counters.
```

上面的命令可以查看到最新的10条内核日志。如果有一些错误的话，很可能会引起一些性能问题。上面的例子包含了一个OOM-Killer和一个TCP请求丢失。

dmesg信息往往值得检查，别轻易跳过了。

### 3. vmstat 1

```bash
$ vmstat 1
procs ---------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
34  0    0 200889792  73708 591828    0    0     0     5    6   10 96  1  3  0  0
32  0    0 200889920  73708 591860    0    0     0   592 13284 4282 98  1  1  0  0
32  0    0 200890112  73708 591860    0    0     0     0 9501 2154 99  1  0  0  0
32  0    0 200889568  73712 591856    0    0     0    48 11900 2459 99  0  0  0  0
32  0    0 200890208  73712 591860    0    0     0     0 15898 4840 98  1  1  0  0
```

一个快速查看虚拟内存状态的指令，通常Linux的设备上都有这个工具。每一行打印出来关键的一些服务统计信息。

上面的命令带有一个参数“1”，用来每一秒打印一条统计信息。第一条输出是从开机到现在的平均值。通常会跳过第一条信息。

- **r**：在CPU上运行并等待轮换的进程数量。这为确定CPU饱和度提供了比负载平均值更好的信息，因为这其中不包含I/O占用。
- **free**：以Kb为单位的空闲内存，数字越大代表空闲内存越多。后面要提到的第7条指令`“free -m”`可以看到更多空闲内存的信息。
- **si,so**：内存/缓存换入和换出。如果值不是0，代表系统内存不足了。
- **us,sy,id,wa,st**：所有CPU的拆分时间：用户态时间，系统时间（内核态），空闲时间，等待I/O和一些别别的子系统占用的时间。

使用CPU的时间（user+system）可以来确定CPU是否很忙。一个恒定的I/O等待率往往是因为磁盘瓶颈；这代表CPU在空闲，因为任务被阻塞在等待磁盘I/O。可以把等待I/O认为是另外一种CPU等待方式。

I/O的处理需要系统时间。一个高的平均系统时间（20%），可能可以挖掘出更多信息：可能内核正在处理的I/O是低效的。

上面的例子中，CPU的时间几乎被用户态全部占满，如果有性能问题就是应用程序级别的使用问题。CPU的使用率都超过了90%。这不一定就是一个必要的问题，可以查看r列的饱和度信息。

### 4. mpstat -P ALL 1

```bash
$ mpstat -P ALL 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015  _x86_64_ (32 CPU)

07:38:49 PM  CPU   %usr  %nice   %sys %iowait   %irq  %soft  %steal  %guest  %gnice  %idle
07:38:50 PM  all  98.47   0.00   0.75    0.00   0.00   0.00    0.00    0.00    0.00   0.78
07:38:50 PM    0  96.04   0.00   2.97    0.00   0.00   0.00    0.00    0.00    0.00   0.99
07:38:50 PM    1  97.00   0.00   1.00    0.00   0.00   0.00    0.00    0.00    0.00   2.00
07:38:50 PM    2  98.00   0.00   1.00    0.00   0.00   0.00    0.00    0.00    0.00   1.00
07:38:50 PM    3  96.97   0.00   0.00    0.00   0.00   0.00    0.00    0.00    0.00   3.03
```

这条指令打印出了每一个CPU的使用时间分布，可以用来查看系统的负载均衡问题，单个CPU可以认为是一个线程的占用。

### 5. pidstat 1

```bash
$ pidstat 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015    _x86_64_    (32 CPU)

07:41:02 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
07:41:03 PM     0         9    0.00    0.94    0.00    0.94     1  rcuos/0
07:41:03 PM     0      4214    5.66    5.66    0.00   11.32    15  mesos-slave
07:41:03 PM     0      4354    0.94    0.94    0.00    1.89     8  java
07:41:03 PM     0      6521 1596.23    1.89    0.00 1598.11    27  java
07:41:03 PM     0      6564 1571.70    7.55    0.00 1579.25    28  java
07:41:03 PM 60004     60154    0.94    4.72    0.00    5.66     9  pidstat

07:41:03 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
07:41:04 PM     0      4214    6.00    2.00    0.00    8.00    15  mesos-slave
07:41:04 PM     0      6521 1590.00    1.00    0.00 1591.00    27  java
07:41:04 PM     0      6564 1573.00   10.00    0.00 1583.00    28  java
07:41:04 PM   108      6718    1.00    0.00    0.00    1.00     0  snmp-pass
07:41:04 PM 60004     60154    1.00    4.00    0.00    5.00     9  pidstat
```

`pidstat`有点像`top`的单进程统计，但是打印的是滚动的摘要，而不是整屏幕的刷新。这对于观察一段时间内的场景很有用，还记录了你所看到的，可以很方便的拷贝到别的地方去深入调研。

上面的示例显示了两个非常耗CPU资源的两个java进程。%CPU列是所有CPU的消耗；1591%表示了这俩jiava进程几乎占满了16个CPU的时间。

### 6. iostat -xz 1

```bash
$ iostat -xz 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015  _x86_64_ (32 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
          73.96    0.00    3.73    0.03    0.06   22.21

Device:   rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
xvda        0.00     0.23    0.21    0.18     4.52     2.08    34.37     0.00    9.98   13.80    5.42   2.44   0.09
xvdb        0.01     0.00    1.02    8.94   127.97   598.53   145.79     0.00    0.43    1.78    0.28   0.25   0.25
xvdc        0.01     0.00    1.02    8.86   127.79   595.94   146.50     0.00    0.45    1.82    0.30   0.27   0.26
dm-0        0.00     0.00    0.69    2.32    10.47    31.69    28.01     0.01    3.23    0.71    3.98   0.13   0.04
dm-1        0.00     0.00    0.00    0.94     0.01     3.78     8.00     0.33  345.84    0.04  346.81   0.01   0.00
dm-2        0.00     0.00    0.09    0.07     1.35     0.36    22.50     0.00    2.55    0.23    5.62   1.78   0.03
[...]
^C
```

在统计应用的工作量和结果展示上，这是一个非常不错的工具来了解块设备的运作：

- **r/s, w/s, rkB/s, wkB/s**：这些是设备每秒读、写、读取千字节和写入千字节的值。这些指标往往表示了块设备的负载状态。很多性能问题可能是因为过多的负载导致。
- **await**：以毫秒为单位的平均I/O时间。这是应用程序等待的时间，包含了排队和被服务的时间。大于预期的平均等待时间可能是设备负载饱和或者问题导致的。
- **avgqu-sz**：向设备发出的平均请求数。大于1可能代表块设备已经饱和。
- **%util**：设备利用率。数值显示设备每秒工作的时间百分比。通常值大于60%会导致很差的性能，接近100%的话就代表了已经过载了。

如果存储设备是面向许多后端磁盘的逻辑磁盘设备，那么 100% 利用率可能只是意味着 100% 的时间正在处理某些 I/O，但是，后端磁盘可能远未饱和，并且可能能够处理更多的工作。

请记住，性能不佳的磁盘 I/O 不一定是应用程序问题。许多技术通常用于异步执行 I/O，这样应用程序就不会直接阻塞和遭受延迟（例如，读取的预读和写入的缓冲）。

### 7. free -m

```bash
$ free -m
             total       used       free     shared    buffers     cached
Mem:        245998      24545     221453         83         59        541
-/+ buffers/cache:      23944     222053
Swap:            0          0          0
```

- **buffers**：用于块设备IO的buffer缓冲区
- **cached**：用于文件系统的页缓存

如果数值解禁0，可能会导致更高的磁盘I/O时间和更差的性能。上面的例子似乎看着还可以。

### 8. sar -n DEV 1

```bash
$ sar -n DEV 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015     _x86_64_    (32 CPU)

12:16:48 AM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
12:16:49 AM      eth0  18763.00   5032.00  20686.42    478.30      0.00      0.00      0.00      0.00
12:16:49 AM        lo     14.00     14.00      1.36      1.36      0.00      0.00      0.00      0.00
12:16:49 AM   docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00

12:16:49 AM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
12:16:50 AM      eth0  19763.00   5101.00  21999.10    482.56      0.00      0.00      0.00      0.00
12:16:50 AM        lo     20.00     20.00      3.25      3.25      0.00      0.00      0.00      0.00
12:16:50 AM   docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
^C
```

使用此工具检查网络接口吞吐量：rxkB/s 和 txkB/s，作为工作量的衡量标准，并检查是否已达到任何限制。
在上面的示例中，eth0 接收达到 22 Mbytes/s，即 176 Mbits/sec（远低于 1 Gbit/sec 的限制）。

这个版本还有 %ifutil 用于设备利用率（全双工的最大双向），我们也使用 Brendan 的 nicstat 工具来测量。
和 nicstat 一样，这很难做到，而且在这个例子中似乎不起作用（0.00）。

### 9. sar -n TCP,ETCP 1

```bash
$ sar -n TCP,ETCP 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015    _x86_64_    (32 CPU)

12:17:19 AM  active/s passive/s    iseg/s    oseg/s
12:17:20 AM      1.00      0.00  10233.00  18846.00

12:17:19 AM  atmptf/s  estres/s retrans/s isegerr/s   orsts/s
12:17:20 AM      0.00      0.00      0.00      0.00      0.00

12:17:20 AM  active/s passive/s    iseg/s    oseg/s
12:17:21 AM      1.00      0.00   8359.00   6039.00

12:17:20 AM  atmptf/s  estres/s retrans/s isegerr/s   orsts/s
12:17:21 AM      0.00      0.00      0.00      0.00      0.00
^C
```

这是一些关键 TCP 指标的总结视图，这些包括：

- **active/s**: 每秒本地发起的TCP连接数（例如，通过connect函数）
- **passive/s**: 每秒远程发起的TPC连接数（例如，通过accept函数）
- **retrans/s**: 每秒TCP重传次数

主动和被动计数通常可用于粗略衡量服务器负载：新接受的连接数（被动）和下游连接数（主动）。将主动视为出站，将被动视为入站可能会有所帮助，但这并不完全正确（例如，考虑 localhost 到 localhost 的连接）。

重传是网络或服务器问题的标志；它可能是一个不可靠的网络（例如，公共互联网），或者可能是由于服务器过载并丢弃数据包。上面的例子每秒只显示一个新的 TCP 连接。

### 10. top

```bash
$ top
top - 00:15:40 up 21:56,  1 user,  load average: 31.09, 29.87, 29.92
Tasks: 871 total,   1 running, 868 sleeping,   0 stopped,   2 zombie
%Cpu(s): 96.8 us,  0.4 sy,  0.0 ni,  2.7 id,  0.1 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem:  25190241+total, 24921688 used, 22698073+free,    60448 buffers
KiB Swap:        0 total,        0 used,        0 free.   554208 cached Mem

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 20248 root      20   0  0.227t 0.012t  18748 S  3090  5.2  29812:58 java
  4213 root      20   0 2722544  64640  44232 S  23.5  0.0 233:35.37 mesos-slave
 66128 titancl+  20   0   24344   2332   1172 R   1.0  0.0   0:00.07 top
  5235 root      20   0 38.227g 547004  49996 S   0.7  0.2   2:02.74 java
  4299 root      20   0 20.015g 2.682g  16836 S   0.3  1.1  33:14.42 java
     1 root      20   0   33620   2920   1496 S   0.0  0.0   0:03.82 init
     2 root      20   0       0      0      0 S   0.0  0.0   0:00.02 kthreadd
     3 root      20   0       0      0      0 S   0.0  0.0   0:05.35 ksoftirqd/0
     5 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H
     6 root      20   0       0      0      0 S   0.0  0.0   0:06.94 kworker/u256:0
     8 root      20   0       0      0      0 S   0.0  0.0   2:38.05 rcu_sched
```

top 命令包括我们之前检查的许多指标。运行它可以很方便地查看是否有任何东西与之前的命令有很大不同，这表明负载是可变的。

top 的一个缺点是比较难看出来一段时间的变化，这在提供滚动输出的 vmstat 和 pidstat 等工具中可能更清楚。如果您没有足够快地暂停输出（Ctrl-S 暂停，Ctrl-Q 继续）并且屏幕清除，则间歇性问题的证据也可能丢失。

夜已深，睡觉^_^!