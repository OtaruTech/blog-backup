---
title: Tip of the Week 116 保留多参数的引用
date: 2022-05-05 21:33:29
tags:
- C++11
- C++Tips
categories:
- C++ Tips of the Week
cover: /img/C++Tips-116.jpg
---

> **从绘画到图像，从图像到文本，从文本到声音，一种虚构的指针指示、展示、固定、定位、强加引用系统，试图稳定一个独立空间。——米歇尔·福柯创作的这不是一只烟斗**
> 

## const引用与指向const的指针

`const`引用用作函数的参数，与指向`const`的指针相比，它有几个优点：它们不能为`null`。并且很明显该函数没有取得对象的所有权。但是它们有其他的有事会出现问题的差异：它们更加隐式（即在调用点没有任何内容表明我们正在接受引用），并且它们可以被绑定到临时变量。

## 类中悬空引用的风险

将以下类作为示例：

```cpp
class Foo {
 public:
  explicit Foo(const std::string& content) : content_(content) {}
  const std::string& content() const { return content_; }

 private:
  const std::string& content_;
};
```

它看起来是合理的。但是如果我们从`string`字面量构建一个`Foo`会发生什么呢？

```cpp
void Func() {
  Foo foo("something");
  std::cout << foo.content();  // 轰隆!
}
```

当创建`foo`时，`content_`成员被绑定到临时`std::string`对象，该对象创建自字面量并传递给构造函数。临时字符串在创建的那一行结尾就离开了作用域。现在`foo.content_`是一个已经不存在的对象的引用。访问它是未定义的行为，任何事情都可能发生，在测试中工作正常，到实际生产中出现错误。

## 解决方案：使用指针

在我们的示例中，最简单的方案可能是按值来传递和保存`string`。但是假定我们需要参考原始的参数，例如它不是一个`string`，而是一些更有趣的类型。解决方案是按指针传递参数：

```cpp
class Foo {
 public:
  // 不要忘记这个注释：
  // 不要接管content的所有权,content必须引用比这个对象更长生存期的有效字符串
  explicit Foo(const std::string* content) : content_(content) {}
  const std::string& content() const { return *content_; }

 private:
  const std::string* const content_;  // 不拥有，不能为空
};
```

现在下面的内容将直接编译失败：

```cpp
std::string GetString();
void Func() {
  Foo foo1(&GetString());  // 错误：拥有类型'std::string临时对象的地址
  Foo foo2(&"something");  // 错误：与'Foo'初始化构造函数不匹配
}
```

这下面的调用处的对象保留参数的地址是相当清晰地：

```cpp
void Func2() {
  std::string content = GetString();
  Foo foo(&content);
}
```

## 更进一步，少一个评论：存储引用

你可能注意到，我们讲了两次关于指针不能为空并且它也不被拥有，一次是在构造函数的文档中，一次是在关于实例变量的注释中，这是必要的吗？考虑一下：

```cpp
class Baz {
 public:
  // 不拥有任意所有权，并且所有的指针必须引用超过构造的生命周期的有效对象
  explicit Baz(const Arg1* arg1, Arg2* arg2) : arg1_(*arg1), arg2_(*arg2) {}

 private:
  // 现在很明显我们没有所有权并且引用不能为空
  const Arg1& arg1_;
  Arg2& arg2_;  // 是的，非常量引用是合适的风格。
};
```

引用类型的成员的一个缺点是不能重新赋值，意味着你的类没有一个拷贝赋值运算符（拷贝构造函数是正常的），但是显式地删除它以遵循3规则可能是有意义的。如果你的类应该是可分配的，那么你需要一个非常量指针，仍旧只想一个常量对象。

如果你想深入防御并认为调用者可能意外的传递一个空指针，那么可以使用“`ABSL_DIE_IF_NULL(arg1)`”来产生一个崩溃。请注意，仅仅解引用空指针并不像通常认为的那样会保证崩溃；相反它是未定义行为并且不应该被依赖。这里可能发生的是，当有人实际访问这个字段时，由于引用被作为指针来执行，它将被拷贝并且随后将崩溃。

## 结论

如果参数是拷贝的，或者只是在构造函数中使用并且在构造函数对象中保留它的引用，那么通过常量引用传递参数到构造函数是可行的。在其他情况中，考虑以指针传（无论是否`const`）递参数。还请记住，如果你实际正在传递对象的所有权，则应该将其作为`std::unique_ptr`来传递。

最后，此处讨论的内容不限定于构造函数：以某种方式保留参数之一的别名的函数，无论是通过放置一个指针在缓存中还是将参数绑定到分离函数中，都应该通过指针获取该函数。