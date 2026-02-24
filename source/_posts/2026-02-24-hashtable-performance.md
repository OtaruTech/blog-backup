---
title: 哈希表性能分析实战
date: 2026-02-24 00:17:00
tags:
  - 性能优化
  - C++
  - 哈希表
  - Abseil
categories:
  - 性能优化
cover: /images/hashtable-performance.jpg?w=800
---

## 哈希表性能分析

**关键观点**: 哈希表问题很难从 CPU profile 中发现，需要专门的profiler。

### 核心指标

| 指标 | 含义 | 阈值 |
|------|------|------|
| Stuck bits | 哈希函数熵不足 | 应为 0 |
| Probe length | 平均探针长度 | > 0.1 表示有碰撞 |
| Rehashes | 重新哈希次数 | 过多说明没提前 reserve() |

### 案例分析

#### 1. 使用 absl::Hash

推荐使用 `absl::Hash` 而非自定义弱哈希函数：

```cpp
// 推荐
absl::flat_hash_map<Key, Value> map;

// 避免自定义弱哈希
```

#### 2. 为分片添加 Salt

为分片哈希添加 salt，避免底部 bit 冲突：

```cpp
// 为每个分片添加不同的 salt
uint64_t salt = GetShardSalt(shard_id);
size_t hash = MixHash(base_hash, salt);
```

### 实践建议

1. **提前分配**: 使用 `reserve()` 预分配内存
2. **选择合适类型**: 优先使用 `absl::flat_hash_map`
3. **监控指标**: 定期检查探针长度和 rehash 次数
4. **避免自定义**: 尽量使用标准库或 Abseil 的哈希实现

### 总结

哈希表是 C++ 项目中最常用的数据结构之一，但其性能问题往往隐藏在表面之下。通过专业的哈希表 profiler，我们可以发现 CPU profile 无法揭示的问题，从而进一步优化系统性能。

---
*来源: Abseil 性能优化指南*
