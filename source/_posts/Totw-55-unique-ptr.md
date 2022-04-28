---
title: Tip of the Week 55 名字计数与unique_ptr
date: 2022-04-28 22:52:13
tags:
- C++11
- C++Tips
categories:
- C++ Tips of the Week
cover: /img/C++Tips-55.png
---
> “尽管我们可能以一千种名字来认识他，但是他于我们所有人而言是同一个人。”——圣雄甘地
> 

通俗来讲，值的“名字”是在任意作用域内保持特定数值的任意值类型变量（不是指针也不是引用）。（对于规范准则而言，如果我们说“名字”，实质上我们谈论的是左值。）因为`std::unique_ptr`的特定行为的要求，我们需要确保在`std::unique_ptr`中保存的任意值仅有一个名字。

请务必注意，C++委员会为`std::unique_ptr`选择了一个非常恰当的名字。在任何时候，存储在`std::unique_ptr`中的任何非空指针值都只能在一个`std::unique_ptr`中出现；标准库以强制执行此操作来设计的。许多编译代码中使用`std::unique_ptr`的常见问题能够通过学习如何计算`std::unique_ptr`的数量来解决：一个名字是可以的，但是相同的指针值对应多个名字则不行。

让我们数一些名字。在每一行，计算在位置（无论是否在作用域内）上包含相同指针的`std::unique_ptr`并且存在的名字数量。如果你在任何一行发现关于相同指针值的名字多余一个，那是一个错误！

```cpp
std::unique_ptr<Foo> NewFoo() {
  return std::unique_ptr<Foo>(new Foo(1));
}

void AcceptFoo(std::unique_ptr<Foo> f) { f->PrintDebugString(); }

void Simple() {
  AcceptFoo(NewFoo());
}

void DoesNotBuild() {
  std::unique_ptr<Foo> g = NewFoo();
  AcceptFoo(g); // 不能编译
}

void SmarterThanTheCompilerButNot() {
  Foo* j = new Foo(2);
  // 可以编译，但是违反规则并且在运行时会重复删除
  std::unique_ptr<Foo> k(j);
  std::unique_ptr<Foo> l(j);
}
```

在`Simple()`中，随`NewFoo()`分配的唯一指针只有一个可以引用的名字：在`AcceptFoo()`中的“f”。

与`DoesNotBuild()`相比，随`NewFoo()`分配的唯一指针有两个引用它的名字：`DoesNotBuild()`中的”g”和`AcceptFoo()`中的“f”。

这时一个典型的违反唯一性：在执行中的任何给定点，通过`std::unique_ptr`（更确切的说，任何仅能够移动的类型）拥有的任何值都只能由单一明显的名字来引用。任何看起来像引用的额外名字的拷贝，都是禁止的，并且不能编译。

```cpp
scratch.cc: 错误: 调用std::unique_ptr<Foo>被废弃了的构造函数
AcceptFoo(g);
```

即便编译器没有抓到你的问题，`std::unique_ptr`的运行时行为也会发生。每当你“自已为超过”编译器，然后引入多个`std::unique_ptr`名字，它可能能够编译，但是你将遇到运行时内存问题。

现在问题变成了：我们如何移除名字？C++11以`std::move()`的形式提供了一种解决方案。

```cpp
void EraseTheName() {
   std::unique_ptr<Foo> h = NewFoo();
   AcceptFoo(std::move(h)); // 用std::move来修复DoesNotBuild
}
```

调用`std::move`实际上市一种名字消除器：从概念上讲，你可以停止计数关于指针值“h”。现在，这通过distinct-names规则：在`NewFoo()`分配的唯一指针有单个的名字(”h”)，并且在调用`AcceptFoo()`中，这里又仅有单个名字(”f”)。通过使用`std::move`，我们保证直到为它分配了新值，才会再次读取“h”。

在现代C++中，对于那些不熟悉左值、右值等细节的人而言，名字计数是一种方便的技巧：它能够帮助你识别不必要拷贝的可能性，并且它能够帮助你恰当的使用`std::unique_ptr`。名字计数以后，如果你在一个地方发现过多的名字，请使用`std::move`来删除不再需要的名字。