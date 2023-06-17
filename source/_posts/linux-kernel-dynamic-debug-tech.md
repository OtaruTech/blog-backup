---
title: Linux内核调试之动态追踪
date: 2023-06-17 23:00:04
categories:
- 系统性能分析
tags:
- Linux
- 内核动态追踪
cover: /img/kernel-trace-debug.png
---
本文介绍几种常用的内核动态追踪技术，对ftrace、perf及eBPF的使用方法进行案例说明。

## 1. 什么是动态追踪

动态追踪技术，是通过探针机制来采集内核或者应用程序的运行信息，从而可以不用修改内核或应用程序的代码，就获得调试信息，对问题进行分析、定位。

通常在排查和调试异常问题时，我们首先想到的是使用GDB在程序运行路径上设置断点，然后结合命令进行分析定位；或者，在程序源码中增加新的日志，往往需要重新编译和部署。在面对偶先问题以及对时延要求严格的场景下，GDB和增加日志的方式就不能满足需求了。

动态追踪为这些问题提供了完美的方案：它既不需要停止服务，也不需要修改程序的代码；程序还按照原来的方式正常运行时，就可以分析出问题的根源。同时，相比以往的进程级跟踪方法（比如ptrace），动态追踪往往只会带来很小的性能损耗。

## 2. 动态追踪技术分类

动态追踪的工具很多，systemtap、perf、ftrace、ssydig、eBPF等。动态追踪的事件源根据事件类型不同，主要分为三类：静态探针，动态探针以及硬件事件。

- 硬件事件：通常由性能监控计数器OMC（Performance Monitoring Counter）产生，包括了各类硬件的性能情况，比如CPU的缓存、指令周期、分支预测等。
- 静态探针：事先在代码中定义好，并编译到应用程序或者内核中的探针。这些探针只有在开启探测功能时，才会被直行道；未开启时并不会执行。常见的静态探针包括内核中的跟踪点（tracepoints）和USDT（Userland Statically Defined Tracing）探针。
- 动态探针：没有实现在代码中定义，但却可以在运行时动态添加的探针，比如函数的调用和返回等。动态探针支持按需在内核或者应用程序中添加探测点，具有更高的灵活性。常见的动态探针有两种，即用于内核态的kprobes和用户态的uprobes。需要注意的是kprobes需要内核编译时开启CONFIG_KPROBE_EVENTS，uprobes则需要内核编译时开启CONFIG_UPROBE_EVENTS。

## 3. 动态追踪之ftrace

ftrace最早用于函数追踪，后来扩展支持了各种事件跟踪功能。ftrace通过debugfs以普通文件的形式，向用户空间提供访问接口，这种不需要额外的工具，就可以通过挂载点（通常为/sys/kernel/debug/tracing目录）的文件读写，来跟ftrace交互，跟踪内核或者应用程序的运行事件。

在使用ftrace之前，首先要确定当前系统是否已经挂载了debugfs，可以使用如下方式进行确认。

```bash
#方式一：查看挂载信息
mount | grep debugfs
#方式二：查看挂载点
ls /sys/kernel/debug/tracing
```

如果在上面的查找结果中有输出，说明当前系统已经挂载了debugfs，如果系统未挂载debugfs，则使用如下的命令进行挂载。

```bash
mount -t debugfs nodev /sys/kernel/debug
```

在/sys/kernel/debug/tracing/目录下提供了各种跟踪器（tracer）和时间（event），一些常用的选项如下。

- available_tracers: 当前系统支持的跟踪器；
- available_events: 当前系统支持的事件数；
- current_tracer: 当前正在使用的跟踪器；默认为nop，表示不做任何跟踪操作，使用echo命令把跟踪器的名字写入该文件，即可切换到不同的跟踪器；
- trace: 当前的跟踪信息；
- tracing_on: 用于开始活暂停跟踪；
- trace_options: 设置ftrace的一些相关选项；

ftrace提供了多个跟踪器，用于跟踪不同类型的信息，比如函数调用、中断关闭、进程调度等。具体支持的跟踪器取决于系统配置，使用如下的命令来查询当前系统支持的跟踪器。

```bash
root@ubuntu:/sys/kernel/debug/tracing# cat available_tracers 
hwlat blk mmiotrace function_graph wakeup_dl wakeup_rt wakeup function
```

以上是系统支持的所有跟踪器，function表示跟踪函数的执行，function_graph则是跟踪函数的调用关系，也就是生成直观的调用关系图，这是最常用的两种跟踪器。也可以通过配置内核，使系统支持更多类型的跟踪器。

在使用跟踪器前，需要确定好跟踪目标，包括内核函数和事件，函数是指内核中的函数名，事件是指内核中预先定义的跟踪点。可以使用如下方式查看内核支持跟踪的函数和事件。

```bash
#查看内核支持追踪的函数
cat available_filter_functions
#查看内核支持追踪的事件
cat available_events
```

接下来以一个简单的实例说明ftrace的基本用法，比如我们需要跟踪open的系统调用，open在内核中的视线接口为do_sys_open，所以需要将跟踪的函数设置为do_sys_open，以下是跟踪过程的步骤。

```bash
#设置函数跟踪点为 do_sys_open
echo do_sys_open > set_graph_function
#设置当前跟踪器为 function_graph
echo function_graph > current_tracer
#配置 trace 属性：显示当前的进程
echo funcgraph-proc > trace_options
#清除 trace 缓存
echo  > trace
#开启跟踪
echo 1 > tracing_on
#产生 do_sys_open 调用
ls
#关闭跟踪
echo 0 > tracing_on
```

在关闭跟踪后，使用cat命令查看跟踪结果，如下所示，在trace文件中保存了跟踪到的信息，第一列表示接口执行的CPU，第二列表示任务名称和进程PID，第三列是函数执行延迟，最后一列是函数调用关系图。

```bash
root@ubuntu:/sys/kernel/debug/tracing# cat trace
# tracer: function_graph
#
# CPU  TASK/PID         DURATION      FUNCTION CALLS
# |     |    |           |   |        |   |   |   |
 2)    ls-46073    |               |  do_sys_open() {
 2)    ls-46073    |               |    getname() {
 2)    ls-46073    |               |      getname_flags() {
 2)    ls-46073    |               |        kmem_cache_alloc() {
 2)    ls-46073    |               |          _cond_resched() {
 2)    ls-46073    |   0.119 us    |            rcu_all_qs();
 2)    ls-46073    |   0.416 us    |          }
 2)    ls-46073    |   0.095 us    |          should_failslab();
 2)    ls-46073    |   0.112 us    |          memcg_kmem_put_cache();
 2)    ls-46073    |   1.041 us    |        }
 2)    ls-46073    |               |        __check_object_size() {
 2)    ls-46073    |   0.097 us    |          check_stack_object();
 2)    ls-46073    |   0.100 us    |          __virt_addr_valid();
 2)    ls-46073    |   0.097 us    |          __check_heap_object();
 2)    ls-46073    |   0.662 us    |        }
 2)    ls-46073    |   2.109 us    |      }
 2)    ls-46073    |   2.414 us    |    }
```

ftrace的输出通过不同级别的缩进，直观展示了各函数间的调用关系，但是ftrace的使用需要好几个步骤，用起来并不方便，不过，trace-cmd已经把这些步骤给包装了起来，这样，就可以通过一行命令，完成上述所有过程。trace-cmd的安装方式如下

```bash
# Ubuntu
apt install trace-cmd
# CentOS
yum install trace-cmd
```

trace-cmd安装好之后，可以通过执行如下的命令输出类似的结果，值得注意的是 trace-cmd 的执行不能在 /sys/kernel/debug/tracing 路径，否则执行出出错，提示信息为“trace-cmd: Permission denied”。

```bash
trace-cmd record -p function_graph -g do_sys_open -O funcgraph-proc ls
trace-cmd report
```

ftrace的追踪功能远不止于此，它不仅能追踪到接口的调用关系，还能抓取接口调用的时间戳，用于性能分析；还可根据需要追踪的接口进行模糊过滤，众多的功能不在这里详细介绍，如果项目中需要用到在进行具体了解和总结。

## 4. 动态追踪之perf

perf的功能强大，可以统计分析出应用程序或者内核中的热点函数，从而用于程序性能分析；也可以用来分析CPU cache、CPU迁移、分支预测、指令周期等各种硬件事件；还可以对感兴趣的事件进行动态追踪。下面以do_sys_open为例，设置目标函数追踪。执行如下命令，可以查询所有支持的事件。

```bash
perf list
```

在perf的各个子命令中添加—event选项，设置追踪感兴趣的事件。如果这些预定义的事件不满足实际需求，可以使用perf probe来动态添加。而且，除了追踪内核事件外，perf还可以用来追踪用户空间的函数，执行如下代码添加do_sys_open探针。

```bash
#执行指令
perf probe --add do_sys_open
#输出
Added new event:
  probe:do_sys_open    (on do_sys_open)
You can now use it in all perf tools, such as:
  perf record -e probe:do_sys_open -aR sleep 1
```

探针添加成功后，就可以在所有的perf子命令中使用。比如，上述输出就是一个perf record的示例，执行它就可以对1s内的do_sys_open进行采样，如下所示。

```bash
#执行命令
perf record -e probe:do_sys_open -aR sleep 1
#输出
[ perf record: Woken up 1 times to write data ][ perf record: Captured and wrote 0.810 MB perf.data (18 samples) ]
```

执行如下命令显示采样结果，输出结果中列出了调用do_sys_open的任务名称、进程PID以及运行的CPU灯信息。

```bash
#执行命令
perf script
#输出
perf         3676 [003]  7619.618568: probe:do_sys_open: (ffffffffa92e78e0)
sleep        3677 [000]  7619.621118: probe:do_sys_open: (ffffffffa92e78e0)      
sleep        3677 [000]  7619.621130: probe:do_sys_open: (ffffffffa92e78e0)     
vminfo       1403 [001]  7619.864117: probe:do_sys_open: (ffffffffa92e78e0)
dbus-daemon   749 [001]  7619.864222: probe:do_sys_open: (ffffffffa92e78e0)
dbus-daemon   749 [001]  7619.864310: probe:do_sys_open: (ffffffffa92e78e0) 
irqbalance    743 [000]  7620.013548: probe:do_sys_open: (ffffffffa92e78e0) 
irqbalance    743 [000]  7620.013687: probe:do_sys_open: (ffffffffa92e78e0)
```

在使用结束后，使用如下命令删除探针。

```bash
#删除 do_sys_open 探针
perf probe --del probe:do_sys_open
```

## 5. 动态追踪之eBPF

eBPF 相对于 ftrace 和 perf 更加灵活，它可以通过 C 语言自由扩展，这些扩展通过 LLVM (Low Level Virtual Machine) 转换为 BPF 字节码后，加载到内核中执行。

5.1 搭建 eBPF 开发环境

虽然 Linux 内核很早就已经支持了 eBPF，但很多新特性都是在 4.x 版本中逐步增加的。所以，想要稳定运行 eBPF 程序，内核至少需要 4.9 或者更新的版本。而在开发和学习 eBPF 时，为了体验和掌握最新的 eBPF 特性，推荐使用更新的 5.x 内核，接下来的案例是基于 Ubuntu20.04 系统，内核版本为 5.15.0-56-generic，eBPF 开发和运行需要相关的开发工具如下。

- 将 eBPF 程序编译成字节码的 LLVM；
- C 语言程序编译工具 make；
- 流行的 eBPF 工具集 BCC 和它依赖的内核头文件；
- 内核代码仓库实时同步的 libbpf；
- 内核提供的 eBPF 程序管理工具 bpftool。

可运行如下命令进行相关工具的安装。

- 

```bash
apt install -y make clang llvm libelf-dev libbpf-dev bpfcc-tools libbpfcc-dev linux-tools-$(uname -r) linux-headers-$(uname -r)
```

5.2 开发 eBPF 程序的步骤

一般来说， eBPF 程序的开发分为如下 5 个步骤。

- 第一步，使用 C 语言开发一个 eBPF 程序；
- 第二步，借助 LLVM 把 eBPF 程序编译成 BPF 字节码；
- 第三步，通过 bpf 系统调用，把 BPF 字节码提交给内核；
- 第四步，内核验证并运行 BPF 字节码，并把相应的状态保存到 BPF 映射中；
- 第五步，用户程序通过 BPF 映射查询 BPF 字节码的运行状态。

[https://mmbiz.qpic.cn/sz_mmbiz_jpg/5hRosUANXGh7uUAOsLyN1rISKRE2vlmqG4UShvYzgOXdH9ibv0ibvl82iaXLcbXUjqQMESxMDu6fjKKHxotAUyhbg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/sz_mmbiz_jpg/5hRosUANXGh7uUAOsLyN1rISKRE2vlmqG4UShvYzgOXdH9ibv0ibvl82iaXLcbXUjqQMESxMDu6fjKKHxotAUyhbg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

eBPF 程序执行过程

以上的每一步，都可以自己动手去完成。但为了方便，推荐从 BCC（BPF Compiler Collection）开始学起。BCC 是一个 BPF 编译器集合，包含了用于构建 BPF 程序的编程框架和库，并提供了大量可以直接使用的工具。使用 BCC 的好处是，它把上述的 eBPF 执行过程通过内置框架抽象了起来，并提供了 Python、C++ 等编程语言接口。这样，就可以直接通过 Python 语言去跟 eBPF 的各种事件和数据进行交互。

5.3 借助 BCC 开发 eBPF 程序案例

接下来，就以跟踪 openat()（即打开文件）这个系统调用为例，说明如何开发并运行第一个 eBPF 程序。使用 BCC 开发 eBPF 程序，可以把上面的五步简化为下面的三步。

5.3.1 使用 C 开发一个 eBPF 程序

新建一个 trace_open.c 文件，输入如下内容。

```c
/* 包含头文件 */
#include <uapi/linux/openat2.h>
#include <linux/sched.h>

/* 定义数据结构 */
struct data_t {
  u32 pid;
  u64 ts;
  char comm[TASK_COMM_LEN];
  char fname[NAME_MAX];
};

/* 定义性能事件映射 */
BPF_PERF_OUTPUT(events);

/* 定义 kprobe 处理函数 */
int trace_open(struct pt_regs *ctx, int dfd, const char __user *filename, struct open_how *how)
{
  struct data_t data = {};
  
  /* 获取 PID 和时间 */
  data.pid = bpf_get_current_pid_tgid();
  data.ts = bpf_ktime_get_ns();
  
  /* 获取进程名 */
  if (bpf_get_current_comm(&data.comm, sizeof(data.comm)) == 0)
  {
    bpf_probe_read(&data.fname, sizeof(data.fname), (void *)filename);
  }
  
  /* 提交性能事件 */
  events.perf_submit(ctx, &data, sizeof(data));
  
  return 0;
}
```

BPF 程序可以利用 BPF 映射（map）进行数据存储，而用户程序也需要通过 BPF 映射，同运行在内核中的 BPF 程序进行交互。用户层为了获取内核打开文件名称时，就要引入 BPF 映射。为了简化 BPF 映射的交互，BCC 定义了一系列的库函数和辅助宏定义。

如下是对上述代码的说明。

- data_t 是用户自定义的数据结构，用于保存进程 PID、时间、执行命令及文件名称；
- 使用 BPF_PERF_OUTPUT 来定义一个 Perf 事件类型的 BPF 映射；
- bpf_get_current_pid_tgid 用于获取进程的 TGID 和 PID。定义的 data.pid 数据类型为 u32，所以高 32 位舍弃掉后就是进程的 PID；
- bpf_ktime_get_ns 用于获取系统自启动以来的时间，单位是纳秒；
- bpf_get_current_comm 用于获取进程名，并把进程名复制到预定义的缓冲区中；
- bpf_probe_read 用于从指定指针处读取固定大小的数据，这里则用于读取进程打开的文件名。
- perf_submit() 接口将填充好的 data 数据提交到定义的 BPF 映射中；

5.3.2 使用 Python 开发用户态程序

创建一个 trace_open.py 文件，并输入下面的内容。

```python
#!/usr/bin/env python3

# 1) import bcc library
from bcc import BPF

# 2) load BPF program
b = BPF(src_file="trace_open.c")

# 3) attach kprobe
b.attach_kprobe(event="do_sys_openat2", fn_name="trace_open")

# 4) print header
print("%-18s %-16s %-6s %-16s" % ("TIME(s)", "COMM", "PID", "FILE"))

# 5) define the callback for perf event
start = 0
def print_event(cpu, data, size):
  global start
  event = b["events"].event(data)
  if start == 0:
    start = event.ts
  time_s = (float(event.ts - start)) / 1000000000
  print("%-18.9f %-16s %-6d %-16s" % (time_s, event.comm, event.pid, event.fname))
  
# 6) loop with callback to print_event
b["events"].open_perf_buffer(print_event)
while 1:
  try:
    b.perf_buffer_poll()
  except KeyboardInterrupt:
    exit()
```

如下是对上述代码的说明。

- 第 1) 处导入了 BCC 库的 BPF 模块，以便接下来调用；
- 第 2) 处调用 BPF() 加载第一步开发的 BPF 源代码；
- 第 3) 处将 BPF 程序挂载到内核探针（简称 kprobe），其中 do_sys_openat2() 是系统调用 openat() 在内核中的实现；
- 第 4) 处输出显示标题；
- 第 5) 处的 print_event 定义一个数据处理的回调函数，打印进程的名字、PID 以及它调用 openat 时打开的文件；
- 第 6) 处的 open_perf_buffer 定义了名为 "events" 的 Perf 事件映射，而后通过一个循环调用 perf_buffer_poll 读取映射的内容，并执行回调函数输出进程信息。

5.3.3  执行 eBPF 程序

用户态程序开发完成之后，最后一步就是执行它了。需要注意的是，eBPF 程序需要以 root 用户来运行，非 root 用户需要加上 sudo 来执行，输入如下命令执行程序。

```python
python3 trace_open.py
```

命令执行后，可以在另一个终端中输入 cat 或 ls 命令（执行文件打开命令），然后回到运行 trace_open.py 脚本的终端，结果如下。

```bash
TIME(s)            COMM               PID    FILE
0.000000000        b'ls'              5171   b'/etc/ld.so.cache'
0.000021937        b'ls'              5171   b'/lib/x86_64-linux-gnu/libselinux.so.1'
......
2.088971702        b'gsd-housekeepin' 1803   b'/etc/fstab'
2.089056747        b'gsd-housekeepin' 1803   b'/proc/self/mountinfo'
......
2.741662512        b'cat'             5172   b'/etc/ld.so.cache'
2.741681539        b'cat'             5172   b'/lib/x86_64-linux-gnu/libc.so.6'
......
```

5.4 eBPF 开发方式简介

除了 BCC 之外，eBPF 还有可以使用其他的方式进行辅助开发，比如 bpftrace 和 libbpf，每种方法都有自己的优点和适用范围，以下是这三种方式的对比。

- bpftrace 通常用在快速排查和定位系统上，它支持用单行脚本的方式来快速开发并执行一个 eBPF 程序。不过，bpftrace 的功能有限，不支持特别复杂的 eBPF 程序，也依赖于 BCC 和 LLVM 动态编译执行。
- BCC 通常用在开发复杂的 eBPF 程序中，其内置的各种小工具也是目前应用最为广泛的 eBPF 小程序。不过，BCC 也不是完美的，它依赖于 LLVM 和内核头文件才可以动态编译和加载 eBPF 程序。
- libbpf 是从内核中抽离出来的标准库，用它开发的 eBPF 程序可以直接分发执行，这样就不需要每台机器都安装 LLVM 和内核头文件了。不过，它要求内核开启 BTF 特性，需要非常新的发行版才会默认开启（如 RHEL 8.2+ 和 Ubuntu 20.10+ 等）。

在实际应用中，可以根据内核版本、内核配置、eBPF 程序复杂度，以及是否允许安装内核头文件和 LLVM 编译工具等，来选择最合适的方案。