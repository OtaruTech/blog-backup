---
title: 现代C++性能优化实战（上）
date: 2026-02-24 22:23:00
tags:
  - C++
  - 现代C++
  - 性能优化
  - 移动语义
categories:
  - C++开发
cover: /img/modern-cpp.jpg

  - C++开发
---

# 现代C++性能优化实战（上）

> 移动语义、智能指针、Lambda、并行算法详解

<!--more-->

## 引言

本文介绍现代 C++（C++11/14/17/20/23）中的性能优化技术，这些技术使得代码既安全又高效。

> 🎯 **核心理念**：现代 C++ 的设计让高性能与安全性可以兼得！

---

## 1. 移动语义 (Move Semantics)

### 避免不必要的拷贝

```cpp
// C++98：总是拷贝
std::vector<int> getData() {
    std::vector<int> v;
    // 填充数据
    return v;  // 拷贝整个vector！
}

// C++11：移动语义
std::vector<int> getData() {
    std::vector<int> v;
    // 填充数据
    return v;  // 移动，只移动指针
}

// 等价于
return std::move(v);  // 显式移动
```

### 移动 vs 拷贝

```cpp
class Buffer {
    char* data_;
    size_t size_;
    
public:
    // 移动构造函数
    Buffer(Buffer&& other) noexcept 
        : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;  // 避免析构函数释放
        other.size_ = 0;
    }
    
    // 移动赋值
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data_;
            data_ = other.data_;
            size_ = other.size_;
            other.data_ = nullptr;
            other.size_ = 0;
        }
        return *this;
    }
};
```

---

## 2. 完美转发 (Perfect Forwarding)

```cpp
// ❌ 只能接受特定类型
template<typename T>
void processValue(T v) {
    doProcess(v);
}

// ✅ 完美转发：保持参数特性
template<typename T>
void processValue(T&& v) {
    doProcess(std::forward<T>(v));  // 转发左值/右值
}

// 使用
processValue(42);           // 作为右值传递
int x = 42;
processValue(x);            // 作为左值传递
processValue(std::move(x)); // 强制右值
```

---

## 3. 智能指针与内存管理

### unique_ptr

```cpp
// ❌ 原始指针管理
class Widget {
    Resource* resource_;
public:
    ~Widget() { delete resource_; }  // 容易忘记删除
};

// ✅ unique_ptr 自动管理
class Widget {
    std::unique_ptr<Resource> resource_;
public:
    // 无需手动析构
};

// 创建
auto widget = std::make_unique<Widget>();
```

### shared_ptr 与 weak_ptr

```cpp
// 共享所有权
auto sp1 = std::make_shared<Widget>();
auto sp2 = sp1;  // 引用计数+1

// 打破循环引用
class Node {
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> parent;  // 不增加引用计数
};
```

---

## 4. Lambda 表达式优化

### 捕获策略

```cpp
int main() {
    int value = 42;
    
    // [=] 按值捕获：拷贝
    auto f1 = [=]() { return value; };
    
    // [&] 按引用捕获：可能悬空
    auto f2 = [&]() { return value; };
    
    // [=, &x] 混合：value拷贝，x引用
    auto f3 = [=, &value]() { value++; };
    
    // [this] 捕获this指针
    auto f4 = [this]() { return member_; };
}
```

### 性能考虑

```cpp
// ❌ 捕获大型对象
std::vector<int> largeData(1000000, 1);
auto f = [=]() { return largeData[0]; };  // 拷贝整个vector！

// ✅ 按引用捕获或移动
std::vector<int> largeData(1000000, 1);
auto f = [&largeData]() { return largeData[0]; };  // 只捕获指针
// 或
auto f = [data = std::move(largeData)]() { return data[0]; };  // 移动
```

---

## 5. 容器优化

### vector 预分配

```cpp
// ❌ 默认构造 + push_back
std::vector<int> v;
for (int i = 0; i < 1000; ++i) {
    v.push_back(i);  // 可能多次重分配
}

// ✅ reserve 预分配
std::vector<int> v;
v.reserve(1000);  // 一次性分配
for (int i = 0; i < 1000; ++i) {
    v.push_back(i);
}

// ✅ emplace_back 就地构造
std::vector<std::pair<int, std::string>> v;
v.emplace_back(1, "hello");  // 直接构造，避免临时对象
```

---

## 6. 并行算法 (C++17)

```cpp
#include <algorithm>
#include <execution>

std::vector<int> data(1000000);

// ❌ 串行排序
std::sort(data.begin(), data.end());

// ✅ 并行排序 (C++17)
std::sort(std::execution::par, data.begin(), data.end());

// ✅ 其他并行算法
std::for_each(std::execution::par, data.begin(), data.end(), [](int& x) {
    x = x * 2;
});

std::transform(std::execution::par, src.begin(), src.end(), dst.begin(), 
    [](int x) { return x * 2; });

std::reduce(std::execution::par, data.begin(), data.end(), 0);
```

---

## 7. constexpr 与编译时计算

```cpp
// C++14: constexpr 函数
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}

int arr[factorial(5)];  // 编译时计算，arr[120]

// C++17: constexpr lambda
constexpr auto add = [](int a, int b) constexpr { 
    return a + b; 
};

// C++20: constexpr 容器和算法
constexpr std::array<int, 5> arr = {1, 2, 3, 4, 5};
constexpr auto sum = std::accumulate(arr.begin(), arr.end(), 0);
```

---

## 总结

| 技术 | 优势 | 适用场景 |
|------|------|----------|
| 移动语义 | 避免拷贝 | 返回大型对象 |
| 完美转发 | 通用参数 | 模板库 |
| unique_ptr | 自动内存管理 | 独占资源 |
| Lambda | 代码内联 | 短回调 |
| 并行算法 | 多核加速 | 大数据处理 |
| constexpr | 编译时计算 | 常量计算 |

---

*本文档基于 drlongnecker 现代C++性能优化系列扩展而成*
