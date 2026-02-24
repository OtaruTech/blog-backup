---
title: 现代C++性能优化实战（下）
date: 2026-02-24 22:22:00
tags:
  - C++
  - 现代C++
  - 性能优化
  - 模板元编程
categories:
  - C++开发
cover: /img/modern-cpp.jpg

  - C++开发
---

# 现代C++性能优化实战（下）

> 模板元编程、内存管理、并发优化详解

<!--more-->

## 引言

本文是现代 C++ 性能优化系列的第二部分，涵盖更高级的优化技术。

---

## 1. 模板元编程

### 类型推导与 SFINAE

```cpp
// 检测类型是否有特定成员
template<typename T, typename = void>
struct has_size : std::false_type {};

template<typename T>
struct has_size<T, std::void_t<decltype(std::declval<T>().size())>> 
    : std::true_type {};

static_assert(has_size<std::vector<int>>::value, "vector has size");
static_assert(has_size<int>::value == false, "int has no size");
```

### 标签分发 (Tag Dispatch)

```cpp
// 根据迭代器类型选择最优算法
template<typename RandomIt>
RandomIt sortImpl(RandomIt first, RandomIt last, std::random_access_iterator_tag) {
    // 随机访问迭代器：快速排序
    return std::sort(first, last);
}

template<typename BidirectionalIt>
BidirectionalIt sortImpl(BidirectionalIt first, BidirectionalIt last, 
                         std::bidirectional_iterator_tag) {
    // 双向迭代器：插入排序
    return std::stable_sort(first, last);
}

template<typename Iterator>
Iterator sort(Iterator first, Iterator last) {
    return sortImpl(first, last, 
        typename std::iterator_traits<Iterator>::iterator_category{});
}
```

---

## 2. 内存管理优化

### 内存池

```cpp
template<typename T, size_t BlockSize = 4096>
class PoolAllocator {
    struct Block {
        Block* next;
        char data[BlockSize - sizeof(Block*)];
    };
    
    Block* head_ = nullptr;
    
public:
    T* allocate() {
        if (!head_) {
            // 分配新块
            Block* newBlock = new Block();
            // 将新块分成小块链接
            // ...
        }
        T* ptr = reinterpret_cast<T*>(head_->data);
        head_ = head_->next;
        return ptr;
    }
    
    void deallocate(T* ptr) {
        Block* block = reinterpret_cast<Block*>(ptr);
        block->next = head_;
        head_ = block;
    }
};
```

### 缓存对齐

```cpp
// C++17 alignas
struct alignas(64) CacheLineData {
    int counter;  // 避免伪共享
};

// 或者使用属性
struct Data {
    int value;
} __attribute__((aligned(64)));
```

---

## 3. 并发优化

### 无锁编程

```cpp
// 原子操作
std::atomic<int> counter(0);

counter.fetch_add(1);           // 原子加
counter.store(42);              // 原子写
int old = counter.load();        // 原子读

// 比较交换 (CAS)
bool compareExchange(std::atomic<int>& atomic, int expected, int desired) {
    return atomic.compare_exchange_strong(expected, desired);
}

// 无锁栈
template<typename T>
class LockFreeStack {
    struct Node {
        T data;
        Node* next;
    };
    
    std::atomic<Node*> head_;
    
public:
    void push(T data) {
        Node* newNode = new Node{data, head_.load()};
        while (!head_.compare_exchange_weak(newNode->next, newNode)) {
            // 重试
        }
    }
};
```

### 避免伪共享

```cpp
// ❌ 多个原子变量在同一缓存行
struct BadStruct {
    std::atomic<int> a;
    std::atomic<int> b;
    std::atomic<int> c;
    std::atomic<int> d;
};

// ✅ 缓存行对齐
struct GoodStruct {
    alignas(64) std::atomic<int> a;
    alignas(64) std::atomic<int> b;
    alignas(64) std::atomic<int> c;
    alignas(64) std::atomic<int> d;
};
```

---

## 4. IO 优化

### 缓冲IO

```cpp
// ❌ 每次写入都刷新
std::ofstream file("data.txt");
for (int i = 0; i < 10000; ++i) {
    file << i << "\n";  // 每次调用系统写入
}

// ✅ 使用缓冲
std::ofstream file("data.txt", std::ios::out | std::ios::binary);
file.rdbuf()->pubsetbuf(buffer, 100000);  // 设置缓冲
for (int i = 0; i < 10000; ++i) {
    file << i << "\n";  // 写入缓冲
}
file.close();  // 一次性刷新

// ✅ 内存映射文件
#include <sys/mman.h>
void* mapped = mmap(NULL, size, PROT_READ | PROT_WRITE, 
                    MAP_PRIVATE, fd, 0);
// 像访问内存一样访问文件
```

---

## 5. 编译优化选项

### GCC/Clang

```bash
# 基本优化
g++ -O2 file.cpp

# 更激进的优化
g++ -O3 file.cpp

# 链接时优化 (LTO)
g++ -O3 -flto file.cpp

# 配置文件引导优化 (PGO)
g++ -O3 -fprofile-generate main.cpp
./a.out  # 运行生成数据
g++ -O3 -fprofile-use main.cpp
```

---

## 6. 性能调优案例

### 案例：字符串处理优化

```cpp
// ❌ 频繁字符串拷贝
std::vector<std::string> split(const std::string& s, char delim) {
    std::vector<std::string> result;
    std::stringstream ss(s);
    std::string item;
    while (std::getline(ss, item, delim)) {
        result.push_back(item);  // 拷贝
    }
    return result;
}

// ✅ 使用 string_view (C++17)
#include <string_view>
std::vector<std::string_view> split(std::string_view s, char delim) {
    std::vector<std::string_view> result;
    size_t start = 0;
    while (true) {
        auto pos = s.find(delim, start);
        if (pos == std::string_view::npos) {
            result.push_back(s.substr(start));
            break;
        }
        result.push_back(s.substr(start, pos - start));
        start = pos + 1;
    }
    return result;
}
```

---

## 总结

| 优化技术 | 性能提升 | 复杂度 |
|----------|----------|--------|
| 模板元编程 | 中 | 高 |
| 内存池 | 2-10x | 中 |
| 无锁编程 | 高 | 高 |
| 缓冲 IO | 10-100x | 低 |
| LTO | 10-20% | 低 |
| PGO | 10-30% | 中 |

---

*本文档基于 drlongnecker 现代C++性能优化系列扩展而成*
