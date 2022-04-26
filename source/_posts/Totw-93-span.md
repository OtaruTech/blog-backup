---
title: Tip of the Week 93 使用absl::Span
date: 2022-04-25 09:40:05
tags:
- C++11
- C++Tips
- absl::Span
categories:
- C++ Tips of the Week
cover: /img/C++Tips-95.jpg
---
# Tip of the Week#93: 使用absl::Span

在google，当我们想处理没有owner的字符串时，通常会使用`absl::string_view`作为函数的参数和返回值。它能够使API更加灵活，并且它能够通过避免对`std::string`进行不必要的转换来提升性能。

`absl::string_view`有一个更加通用的表亲，被称为`absl::Span`。`absl::Span`对`std::vector`就像`absl::string_view`对`std::string`一样。它为`vector`的元素提供只读接口，但是它也可以从由非`vector`(如数组和初始化列表)来构造，并且不会产生拷贝元素的消耗。

`const`可以被删除，因此`absl::Span`是一个元素不能改变的数组的视图，absl::Span允许对元素进行非常量访问。然而，与`const`的跨度不同，这些需要显式构造。

## 关于`std::span/gsl::span`的注释

需要重点注意的是，虽然`absl::Span`在设计和目的上与`std::span`方案(以及现存的`gsl::span`引用实现)相似，但是`absl::Span`目前并不保证是一个对最终标准的随时替代品，因为`std::span`方案仍在开发和变化。

相反，`absl::Span`旨在拥有一个尽可能与`absl::string_view`类似的接口，而不是针对特定于字符串的功能。

## 作为函数参数

使用`absl::Span`作为函数参数的一些好处类似于使用`absl::string_view`的好处。

调用者可以传递出事`vector`的一个切片，或者传递一个纯数组。它也兼容其他类数组的容器，像`absl::InlinedVector`，`absl::FixedArray`，`google::protobuf::RepeatedField`等等。

与`absl::string_view`一样，当用作函数参数时，通常最好是按值传递`absl::Span`；这种方式比通过`const`引用(在大多数平台上)传递稍微快点，并且生成更小的代码。

### 示例：Span的基本用法

```cpp
void TakesVector(const std::vector<int>& ints);
void TakesSpan(absl::Span<const int> ints);

void PassOnlyFirst3Elements() {
  std::vector<int> ints = MakeInts();
  // 我们需要创建一个临时的vecotr,接着引起一个分配和拷贝
  TakesVector(std::vector<int>(ints.begin(), ints.begin() + 3));
  // 当使用Span时，没有拷贝和分配
  TakesSpan(absl::Span<const int>(ints.data(), 3));
}

void PassALiteral() {
  // 这会创建一个临时的std::vector<int>.
  TakesVector({1, 2, 3});
  // Span不需要临时的分配和拷贝，所以它更快
  TakesSpan({1, 2, 3});
}
void IHaveAnArray() {
  int values[10] = ...;
  // 临时的std::vector<int>再一次被创建
  TakesVector(std::vector<int>(std::begin(values), std::end(values)));
  // 仅传递数组。Span自动检测大小。没有拷贝产生。
  TakesSpan(values);
}
```

## 预防缓冲区溢出

因为`absl::Span`知道它自己的长度，相较于C风格的指针长度，API使用它会更安全。

### 示例：更安全的`memcpy()`

```cpp
// 糟糕的代码
void BadUser() {
  int src[] = {1, 2, 3};
  int dest[2];
  memcpy(dest, src, ABSL_ARRAYSIZE(src) * sizeof(int)); // 哎呀，目标溢出.
}
```

```cpp
// 一个简单的示例，但是复用Span已知的大小来阻止上述错误
template <typename T>
bool SaferMemCpy(absl::Span<T> dest, absl::Span<const T> src) {
  if (src.size() > dest.size()) {
    return false;
  }
  memcpy(dest.data(), src.data(), src.size() * sizeof(T));
  return true;
}

void GoodUser() {
  int src[] = {1, 2, 3}, dest[2];
  // 没有溢出！
  SaferMemCpy(absl::MakeSpan(dest), absl::Span<const int>(src));
}
```

## 关于指针的`vector`的`const`的正常性

传递`std::vector<T*>`的一个大问题是你不能再不改变容器类型的情况下使得指针指向的内容为`const`。

任何带有`const std::vector<T*>`的函数都不能修改`vector`，但是它能够修改T类型的值。这也适用于返回`const std::vector<T*>&`的访问器。你不能阻止调用者修改T类型的值。

通常的“解决方案”包括拷贝或者转换`vector`为正确的类型。这些解决方案是慢的(对于拷贝)或者未定义的行为(对于转换)，应该避免它们。相反，请使用`absl::Span`

### 示例：函数参数

考虑这些`Frob`变体：

```cpp
void FrobFastWeak(const std::vector<Foo*>& v);
void FrobSlowStrong(const std::vector<const Foo*>& v);
void FrobFastStrong(absl::Span<const Foo* const> v);
```

从一个需要`Frob`的`const std::vector<Foo*>& V`开始，你有两个不完美的选项和一个完美的。

```cpp
// 更快更容易输入但是不安全
FrobFastWeak(v);
// 更慢更混乱，但是更安全
FrobSlowStrong(std::vector<const Foo*>(v.begin(), v.end()));
// fast, safe, and clear!
FrobFastStrong(v);
```

### 示例：访问器

```cpp
// 糟糕的代码
class DontDoThis {
 public:
  // 不要修改我的Foos，拜托了。
  const std::vector<Foo*>& shallow_foos() const { return foos_; }

 private:
  std::vector<Foo*> foos_;
};

void Caller(const DontDoThis& my_class) {
  // Modifies a foo even though my_class is a reference-to-const
  my_class->foos()[0]->SomeNonConstOp();
}
```

```cpp
// 好的代码
class DoThisInstead {
 public:
  absl::Span<const Foo* const> foos() const { return foos_; }

 private:
  std::vector<Foo*> foos_;
};

void Caller(const DoThisInstead& my_class) {
  // 这个不能编译
  // my_class.foos()[0]->SomeNonConstOp();
}
```

## 结论

在恰当使用时，`absl::Span`可以提供解耦，常量正确性和性能优势。

需要重点注意的是，`absl::Span`的行为非常像`absl::string_view`，在引用一些外部占有的数据上。所有相同的警告都适用。特别的是，`absl::Span`不能比它引用的数据寿命更长。