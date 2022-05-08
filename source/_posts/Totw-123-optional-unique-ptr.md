---
title: Tip of the Week 123 absl::optional和std::unique_ptr
date: 2022-05-08 10:30:03
tags:
- C++11
- C++Tips
categories:
- C++ Tips of the Week
cover: /img/C++Tips-123.png
---
## 如何存储值？

此贴士讨论了集中存储值得方法。此处我们使用类成员变量作为示例，但是以下的许多同样也适用于局部变量。

```cpp
#include <memory>
#include "third_party/absl/types/optional.h"
#include ".../bar.h"

class Foo {
  ...
 private:
  Bar val_;
  absl::optional<Bar> opt_;
  std::unique_ptr<Bar> ptr_;
};
```

## 作为一个纯对象

这是最简单的方法，`val_`分别在`Foo`的构造函数的开头和`Foo`析构函数的末尾被构造和销毁。如果`Bar`有一个默认的构造函数，那么它甚至不需要显式初始化。

`val_`使用起来非常安全，因为它的值不能为`null`。这小出了一类潜在的错误。

但是`bar`对象不是很灵活：

- `val_`的生命周期基本与它的父`Foo`对象的生命周期相关，这有时并不是想要的。如果Bar支持移动或交换操作，那么能够通过这些操作替换val_的内容，然而对`val_`的任何现有的指针或引用继续指向或引用相同的`val_`对象（作为容器），而不是存储在其中的值。
- 需要传递给`Bar`的构造函数的任何参数都需要在`Foo`的构造函数的初始化列表中进行计算，如果涉及复杂的表达式，这可能是困难的。

## 作为absl::optional

纯对象的简单性和`std::unique_ptr`的灵活性。对象存储在Foo中，但与纯对象不同，absl::optional可以为空。它可以随时通过赋值（`opt_=`）或通过在适当位置构造对象来输入。

因为是内联存储的，所以在栈上分配大对象的常见警告同样适用。另外请注意，空`absl::optional`使用的内存和输入的内存一样多。

与纯对象相比，`absl::optional`有一些缺点：

- 对于读者来说，对象的构造和析构的地方都不明显
- 对访问不存在对象的风险

## 作为std::unique_ptr

这是最灵活的方法。对象存储在`Foo`外，就像`absl::optional`一样，`std::unique_ptr`能够为空。然而，不像`absl::optional`，它可以将对象的所有权转移给其他对象（通过移动操作），从其他对象获取对象的所有权（通过构造函数或者赋值函数），或者假定对某个对象的原生指针的所有权（在构造或通过`ptr_=absl::WrapUnique(...)`），参见TotW126

当`std::unique_ptr`作为`null`时，它没有分配对象，只消耗指针1的大小。

如果对象可能需要超出`std::unique_ptr`的作用域（所有权转移），那么`std::unique_ptr`中包装对象时必须的。

这种灵活性伴随着一些成本：

- 增加读者的认知负担：
    - 不容易知道里面存储了什么（`Bar`，或从`Bar`派生的）。然而，它同样也减少认知负担，因为读者只聚集于所持有的基本接口
    - 在对象构造或析构的地方，它甚至比absl::optional更不明显，因为对象的所有权可以转移
- 与`absl::optional`一样，有访问不存在对象的风险——著名的空指针解引用
- 指针引用了额外的间接层，那么需要进行对分配，并且对CPU缓存不友好；重要与否依赖于特定的用例
- `std::unique_ptr`即便是不可复制的。这依然可以放置Foo可复制

## 结论

与往常一样，努力避免不必要的复杂性，并使用最简单的东西。如果适用你的情况，优先穿对象。否则尝试`absl::optional`，请适用`std::unique_ptr`作为最后的方法。