---
title: Tip of the Week 3 字符串拼接，operator+ vs StrCat()
date: 2022-03-21 13:13:23
tags:
- C++11
- StrCat
- C++Tips
categories:
- C++ Tips of the Week
cover: /img/C++Tips-3.jpg
---

当审查代码的人对程序员说“**别用字符串拼接操作，太低效了**”，程序员尝尝会一脸吃惊：`std::string::operator+`怎么会低效呢？难道这不是很难出错的方式吗？

事实证明，这种低效没有那么简单明了。在实践中，下面两段代码执行时间相差不多：

```cpp
std::string foo = LongString1();
std::string bar = LongString2();
std::string foobar = foo + bar;
std::string foo = LongString1();
std::string bar = LongString2();
std::string foobar = absl::StrCat(foo, bar);
```

然而，同样的结论却不适用于下面两段代码：

```cpp
std::string foo = LongString1();
std::string bar = LongString2();
std::string baz = LongString3();
string foobar = foo + bar + baz;

std::string foo = LongString1();
std::string bar = LongString2();
std::string baz = LongString3();
std::string foobar = absl::StrCat(foo, bar, baz);
```

为什么这两个例子中的执行时间比较会不同？我们可以通过拆解表达式`foo + bar + baz`得到答案。C++中没有三个参数的操作符重载。因此这个表达式必须调用两次`string::operator+`。在这两次调用过程中，会有一个临时字符串被构造（和存储）。所以`std::string foobar = foo +bar + baz`等价于：

```cpp
std::string temp = foo + bar;
std::string foobar = std::move(temp) + baz;
```

请注意，在`foobar`被赋值以前，`foo`和`bar`存储的内容必须被复制到一个临时区域中。

C++11至少允许第二个拼接不需要创建一个新的`string`对象：`std::move(temp) + baz`等价于`std::move(temp.append(baz))`。但是，那个临时字符串(`temp`)最初申请的空间可能不足以存下最终的字符串内容，这种情况下会重新申请一次内存(`reallocation`)（以及复制一次）。因此，在最坏情况下，`n`次字符串拼接需要重新申请`O(n)`次内存。

这时最好改用`absl::StrCat()`，一个不错的辅助函数（定义在`absl/strings/str_cat.h`中）。它会计算必要的字符串长度，按此预留空间，然后将所有输入数据放进输出的空间中（优化过的`O(n)`时间复杂度）。类似地，对于如下情况：

```cpp
foobar += foo + bar + baz;
```

可以使用`absl::StrAppend()`（实现了类似优化）：

```cpp
absl::StrAppend(&foobar, foo, bar, baz);
```

另外，`absl::StrCat()`和`absl::StrAppend()`还接收非字符串类型：你可以用`absl::StrCat/absl::StrAppend`转换`int32_t`，`uint32_t`，`int64_t`，`uint64_t`，`float`，`double`，`const char*`和`string_view`（转换为`string`），例如：

```cpp
std::string foo = absl::StrCat("The year is ", year);
```