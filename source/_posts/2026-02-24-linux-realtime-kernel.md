---
title: Linux内核实时性能调优
date: 2026-02-24 22:29:00
tags:
  - Linux
  - 实时系统
  - 内核
  - 性能
  - 调度
categories:
  - 系统开发
---

# Linux内核实时性能调优

> Linux 内核的实时性能对于自动驾驶、工业控制等场景至关重要

<!--more-->

## 引言

Linux 内核的实时性能对于自动驾驶、工业控制等场景至关重要。

---

## 1. 实时Linux概述

### 实时性指标

```
┌──────────────────────────────────────────┐
│          实时性级别                       │
├──────────────────────────────────────────┤
│  硬实时：确定性延迟 < 1ms                │
│  ├─ 确定性响应                            │
│  └─ 延迟抖动 < 10%                       │
├──────────────────────────────────────────┤
│  软实时：平均延迟 < 10ms                 │
│  ├─ 延迟可接受                           │
│  └─ 偶尔超时可接受                       │
├──────────────────────────────────────────┤
│  普通Linux：延迟变化大                    │
│  └─ 不适合实时应用                       │
└──────────────────────────────────────────┘
```

---

## 2. PREEMPT_RT 补丁

### 内核配置

```bash
# .config 配置
CONFIG_PREEMPT=y              #  Voluntary Preemption
CONFIG_PREEMPT_VOLUNTARY=y    #  Voluntary Preemption
CONFIG_PREEMPT=y              #  Full Preemption
CONFIG_PREEMPT_RT=y           #  PREEMPT_RT 补丁
```

---

## 3. CPU调度优化

### 实时调度策略

```c
// 设置SCHED_RR (轮转调度)
struct sched_param param;
param.sched_priority = 99;  // 最高优先级
sched_setscheduler(0, SCHED_RR, &param);

// 或使用SCHED_FIFO (先进先出)
sched_setscheduler(0, SCHED_FIFO, &param);
```

### CPU亲和性

```c
#define _GNU_SOURCE
#include <sched.h>

// 设置CPU亲和性
cpu_set_t cpuset;
CPU_ZERO(&cpuset);
CPU_SET(0, &cpuset);  // 仅使用CPU 0
CPU_SET(1, &cpuset);  // 添加CPU 1

sched_setaffinity(0, sizeof(cpu_set_t), &cpuset);

// 锁定内存（防止换页）
mlockall(MCL_CURRENT | MCL_FUTURE);
```

---

## 4. 中断处理优化

### 中断亲和性

```bash
# 查看中断
cat /proc/interrupts

# 设置中断亲和性
echo 1 > /proc/irq/irq_number/smp_affinity
```

---

## 5. 锁优化

### 无锁编程

```c
// 原子操作
#include <stdatomic.h>

atomic_int counter(0);
atomic_fetch_add(&counter, 1);

// 自旋锁
typedef struct {
    atomic_flag lock = ATOMIC_FLAG_INIT;
} spinlock_t;

void spin_lock(spinlock_t *lock) {
    while (atomic_flag_test_and_set(&lock->lock)) {
        cpu_relax();  // PAUSE指令
    }
}
```

---

## 6. 内存管理

### 大页内存

```bash
# 启用大页
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

# 使用大页
void *ptr = mmap(NULL, size, PROT_READ | PROT_WRITE,
                 MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB,
                 -1, 0);
```

---

## 7. 延迟测量

### cyclictest

```bash
# 基本测试
cyclictest -t 1 -p 80

# 详细输出
cyclictest -t 1 -p 80 -i 1000 -l 10000 -h 30
```

### ftrace

```bash
# 启用追踪
echo 1 > /sys/kernel/debug/tracing/on

# 追踪延迟
echo function_graph > /sys/kernel/debug/tracing/current_tracer
```

---

## 8. 调试工具

| 工具 | 用途 |
|------|------|
| cyclictest | 延迟测试 |
| latencytop | 延迟分析 |
| ftrace | 函数追踪 |
| perf | 性能分析 |
| LTTng | 跟踪分析 |

---

## 总结

| 优化层级 | 技术 | 效果 |
|----------|------|------|
| 内核 | PREEMPT_RT | 10-100x |
| 调度 | FIFO/RR | 确定性 |
| 内存 | 大页/池 | 减少抖动 |
| 中断 | 亲和性 | 避免干扰 |

---

*本文档基于 Epteck Linux 实时优化方案扩展而成*
