---
title: Tip of the Week 77 临时变量，移动，复制
date: 2022-03-30 21:03:10
tags:
- C++11
- C++Tips
- move
categories:
- C++ Tips of the Week
cover: /img/C++Tips-77.jpg
---

我们正在尝试给语言专家意外的人解释清楚C++11会怎样改变代码世界，因此在“什么时候发生复制？”系列中又增加了一篇温江，这也是更广泛的尝试之中的一部分，试图用一套更简单的规则简化C++中的变量复制规则。

## 你会数到2吗？

你会？太好了。记住“名字规则”意味着你能为一个资源指定的所有名字，会影响该对象有多少份拷贝。

## 简单解释数名字

如果你在担心发生了变量复制，那我们驾驶你在担心几行特定的代码。在你认为被复制的数据上，有几个名字存在？只有三种情况需要考虑：

### 1. 两个名字：复制

这个容易：如果你在给同一份数据第二个名字，那就是复制。

```cpp
std::vector<int> foo;
FillAVectorOfIntsByOutputParameterSoNobodyThinksAboutCopies(&foo);
std::vector<int> bar = foo;     // 没错，这就是复制。

std::map<int, string> my_map;
string forty_two = "42";
my_map[5] = forty_two;          // 也是个复制：my_map[5]也算一个名字。
```

### 2. 一个名字：移动

这货有点意外：C++11发现如果你不能再引用一个名字，那你就再也不会关心这份数据了。编程语言必须小心行事，以防博怀那些你依赖析构函数的情况（比如，`absl::MutexLock`），所以`return`是个容易识别的情况。

```cpp
std::vector<int> GetSomeInts() {
  std::vector<int> ret = {1, 2, 3, 4};
  return ret;
}

// 仅移动：数据要么在"ret"里，要么在"foo"里，但不可能同时在两个变量里。
std::vector<int> foo = GetSomeInts();
```

还有一种方法可以告诉编译器放弃一个名字：调用`std::move()`。

```cpp
std::vector<int> foo = GetSomeInts();
// 不是复制，move允许编译器将foo当做临时变量，所以这里会调用std::vector<int>的移动构造函数。
// 注意一点，是构造函数而不是std::move在执行移动操作。调用std::move仅是允许foo被当做临时变量（而不是一个有名字的对象）。
std::vector<int> bar = std::move(foo);
```

### 3. 零个名字：临时变量

临时变量也挺特殊的：如果你想避免复制，尽量别给一个变量命名：

```cpp
void OperatesOnVector(const std::vector<int>& v);

// 没有复制：GetSomeInts()返回的vector会被移动（O(1)）到两次函数调用之间的临时变量，然后以引用传给OperatesOnVector()。
OperatesOnVector(GetSomeInts());
```

## 当心：僵尸变量

以上（除了`std::move()`自己）都还算挺直观地，只是我们在C++11之前的那么多年，都在建立各种关于复制的器官观念。对于一门没有垃圾回收的语言，这样的算计使我们得以完美混合效率与清晰度。然而，这也不是没有危险，其中最大的危险是这个：一个值被移走之后还剩下什么？

```cpp
T bar = std::move(foo);
CHECK(foo.empty()); // 合法吗？也许吧，但别抱太高指望。
```

这是最大的难题之一：我们对剩下的值可以下什么断言？对于大部分标准库类型，这样的值处于“合法但是未指定状态。”非标准类型通常秉持同样的规则。安全的方法是远离这类对象：你可以给他们重新赋值，或者任其自己离开作用域，但是别对其状态做其它的假设。

clang-tidy的misc-use-after-move提供了一些静态检查来捕获“移动后使用”的情况。然而，静态分析永远也不可能找到所有的情况——自己小心。做代码审查的时候帮坐着指出来，而且自己的代码里尽量避免。远离僵尸变量。

## 等会儿，`std::move`不移动东西？

是的呢，另一件需要注意的事儿就是，调用`std::move()`本身并不是一个移动，它就是强制转换到右值引用。而接收右值引用的移动构造函数或移动赋值运算符才是真正干活的人。

```cpp
std::vector<int> foo = GetSomeInts();
std::move(foo); // 什么都没干。
// 调用std::vector<int>的移动构造函数。
std::vector<int> bar = std::move(foo);
```

这几乎不会发生，而且你也许不应该为此良妃脑容量。我们提到它是怕`std::move()`和移动构造函数之间的联系给你整蒙圈了。

## 嗷！太复杂了！为什么！？

首先：真的没有那么差。既然我们手头主要的值类型（包括protobuf）都有了移动操作，我们可以绕开各种“这时复制吗？这样效率高吗？”的讨论，而仅仅依赖于数名字：两个名字，有复制。少于两个名字：没复制。

忽略掉复制的问题，值语义的分析更清楚也更简单。考虑如下两个操作：

```cpp
void Foo(std::vector<string>* paths) {
  ExpandGlob(GenerateGlob(), paths);
}

std::vector<string> Bar() {
  std::vector<string> paths;
  ExpandGlob(GenerateGlob(), &paths);
  return paths;
}
```

他们一样吗？如果`*path`里本来就有数据又如何？你怎么知道的？读者分析值语义要比分析输入/输出参数容易多了。后者你需要考虑（和写文档说明）已有的数据会发生什么，以及可能的是否有指针所有权转移发生。

因为处理值（而非指针）有更简单的生命周期和使用方式的保证，编译器的优化器能够更容易的处理这种风格的代码。被良好管理的值语义也会最小化内存分配器（allocator）的调用（便宜但不免费）。一旦我们理解了移动语义如何帮我们避免复制，编译器的优化器就能更好的分析对象类型，生命期，虚表分派（virtual dispatch），和一坨其他的能帮助生成更高效的机器代码的问题。

既然现在大部分的工具代码都能识别移动语义了，我们就应该别再操心复制和指针语义了，而应该专注于写简单的容易看懂的代码。请确认你理解新的规则：不是所有你遇到的遗留接口会被更新为以值返回（而不是以输出参数返回），所以两种风格混搭回事常态。理解什么时候哪种风格更合适会对你很重要。