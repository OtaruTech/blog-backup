---
title: 编译器优化完全指南
date: 2026-02-24 22:24:00
tags:
  - 编译器
  - GCC
  - Clang
  - 优化
categories:
  - C++开发
cover: /img/compiler-optimization.jpg

  - C++开发
---

# 编译器优化完全指南

> 编译器优化是提升 C++ 性能最"免费"的方式

<!--more-->

## 引言

编译器优化是提升 C++ 性能最"免费"的方式。本文汇总了最新的编译器优化技术。

---

## 1. GCC/Clang 优化选项

### 基本优化级别

```bash
# -O0: 无优化（调试用）
g++ -O0 -g file.cpp

# -O1: 基本优化
g++ -O1 file.cpp

# -O2: 标准优化（常用）
g++ -O2 file.cpp

# -O3: 激进优化
g++ -O3 file.cpp

# -Os: 优化代码大小
g++ -Os file.cpp

# -Ofast: 激进数值优化
g++ -Ofast file.cpp
```

### 架构特定优化

```bash
# 针对本地CPU优化
g++ -O3 -march=native file.cpp

# 针对特定架构
g++ -O3 -march=skylake file.cpp
g++ -O3 -march=armv8-a file.cpp

# 向量化选项
g++ -O3 -mavx2 -mfma file.cpp  # AVX2 + FMA
g++ -O3 -mavx512f file.cpp     # AVX-512
```

---

## 2. Link-Time Optimization (LTO)

### 跨编译单元优化

```bash
# 完整LTO
g++ -O3 -flto file1.cpp file2.cpp

# Thin LTO（更快编译）
g++ -O3 -flto=thin file1.cpp file2.cpp
```

### CMake 配置

```cmake
# 启用 LTO
include(CheckIPOSupported)
check_ipo_supported(RESULT ipo_supported)
if(ipo_supported)
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
endif()

set(CMAKE_CXX_FLAGS_RELEASE "-O3 -flto")
```

---

## 3. 配置文件引导优化 (PGO)

### 使用流程

```bash
# 1. 编译 + 生成prof数据
g++ -O3 -fprofile-generate -fprofile-info-section=.text.fdo main.cpp
./a.out  # 运行，生成 gmon.out

# 2. 使用prof数据优化编译
g++ -O3 -fprofile-use -fprofile-info-section=.text.fdo main.cpp
```

---

## 4. 常用优化选项详解

### 内联优化

```bash
# 强制内联
__attribute__((always_inline)) void func() {}

// 禁用内联
__attribute__((noinline)) void func() {}

// 提示内联级别
__attribute__((flatten)) void func() {}  # 尽量展平
```

### 向量化

```bash
# 强制向量化
g++ -O3 -ftree-vectorize -fopt-info-vec-optimized

# 报告向量化信息
g++ -O3 -fopt-info-vec-optimized -fopt-info-vec-missed
```

---

## 5. Clang 特有优化

### Clang 特定选项

```bash
# 时间优化
clang -O3 -Rpass=.* file.cpp  # 报告所有优化

# 循环优化
clang -O3 -mllvm -loop-vectorize file.cpp

# SLP 向量化
clang -O3 -mllvm -slp-vectorize file.cpp

# ThinLTO
clang -O3 -flto=thin -fuse-ld=lld
```

---

## 6. 调试优化代码

### 查看汇编

```bash
# 生成汇编
g++ -O2 -S file.cpp -o file.s

# 带源码的汇编
g++ -O2 -g -S file.cpp -o file.s

# 使用 objdump
objdump -d -S a.out > dump.s
```

### 使用 Godbolt

访问 [godbolt.org](https://godbolt.org) 实时查看编译输出。

---

## 7. 编译器 intrinsic

### SIMD intrinsic

```cpp
#include <immintrin.h>

// AVX2 加法
__m256d a, b;
__m256d result = _mm256_add_pd(a, b);

// AVX2 矩阵乘法
__m256d a1 = _mm256_load_pd(A);
__m256d b1 = _mm256_set1_pd(b[i]);
__m256d prod = _mm256_mul_pd(a1, b1);
__m256d sum = _mm256_add_pd(prod, sum);
```

---

## 8. 优化最佳实践

### CMake 建议

```cmake
# Release 优化配置
set(CMAKE_CXX_FLAGS_RELEASE
    "-O3 -march=native -flto -fno-stack-protector")

# Debug 不优化
set(CMAKE_CXX_FLAGS_DEBUG
    "-O0 -g -Wall")

# 统一设置
add_compile_options(-Wall -Wextra)
```

---

## 总结

| 优化方式 | 效果 | 成本 |
|----------|------|------|
| -O2/-O3 | 基础 | 免费 |
| -march=native | 10-30% | 免费 |
| LTO | 10-20% | 编译时间 |
| PGO | 10-30% | 两阶段编译 |
| Intrinsic | 依赖实现 | 编码复杂度 |

---

*本文档基于 xania.org Compiler Advent 扩展而成*
