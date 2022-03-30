---
title: Tip of the Week 11 返回策略
date: 2022-03-28 23:02:14
tags:
- C++11
- C++Tips
categories:
- C++ Tips of the Week
cover: /img/C++Tips-11.jpg
---

注：这条贴士—虽然仍然有效—但是写它的时候还没有C++11的移动语义。同时可以参阅[Totw #77](https://abseil.io/tips/77)。

很多老的C__代码库使用了“害怕复制对象”的范式。幸运的是，多亏了“[返回值优化(RVO](https://zh.wikipedia.org/wiki/%E8%BF%94%E5%9B%9E%E5%80%BC%E4%BC%98%E5%8C%96))”，我们可以“复制”但不真的复制。

RVO特性存在已久，几乎所有的C++编译器都实现了它。考虑如下的C++98代码，有一个复制构造函数和一个复制运算符。这些函数代价都很大，开发者让它们在每次调用时都打印一条消息：

```cpp
class SomeBigObject {
 public:
  SomeBigObject() { ... }
  SomeBigObject(const SomeBigObject& s) {
    printf("死贵的复制…\n", …);
    …
  }
  SomeBigObject& operator=(const SomeBigObject& s) {
    printf("死贵的赋值…\n", …);
    …
    return *this;
  }
  ~SomeBigObject() { ... }
  …
};
```

会被下面的代码吓一跳吗？

```cpp
static SomeBigObject SomeBigObjectFactory(...) {
  SomeBigObject local;
  ...
  return local;
}
```

看着挺低效的，是不是？如果我们跑下面这段代码会发生什么？

```cpp
SomeBigObject obj = SomeBigObject::SomeBigObjectFactory(...);
```

简单的答案：你也许会期望至少有两个对象被构造：一个是函数返回的对象，另一个是函数调用出的对象。两次都是复制，所以程序打印两条“死贵”的消息。实践的答案：没有消息被打印—因为复制构造函数和赋值运算符都没有被调用！

这时怎么回事？很多程序员为了写“高效代码”，创造一个对象，然后把对象地址传给函数，函数用指针或引用来操作原始对象。其实，上面的的情形中，编译器能把这些“低效的复制”转译成如前的“高效代码”。

当编译器在函数调用出喊道一个变量（来接收函数的返回值），在被调用的函数里看到另一个（将要被返回的）变量，那编译器就意识到它不需要两个变量。在实现中，编译器就会把函数调用处（函数外）的变量的地址传递给调用的函数。

引用C++98标准原文的说法，“当一个临时对象被复制构造函数复制时...实现被允许将原始对象和副本对象看做指向同一对象的两种方式，因此不需要执行复制操作—即使复制构造函数或析构函数有副作用。对于一个返回值为类的函数，如果返回语句中的表达式是局部对象的名字...实现被允许不必创建临时对象来保存函数返回值...“

担心“被允许”不是个很强的承诺，熏晕的是，所有现代C++编译器默认都会执行RVO，即使实在调试版本中，即使是非内联函数中。

## 你怎么保证编译器执行RVO？

被调用的函数应该定义唯一的返回值变量：

```cpp
SomeBigObject SomeBigObject::SomeBigObjectFactory(...) {
  SomeBigObject local;
  …
  return local;
}
```

函数调用处应该将返回值赋值给一个新的变量：

```cpp
// 不会打印“死贵”操作的消息：
SomeBigObject obj = SomeBigObject::SomeBigObjectFactory(...);
```

就这些！

如果函数调用处重用已经存在的变量来存储返回值，那么编译器没法执行RVO：

```cpp
// RVO不会发生；打印详细“死贵的赋值...”：
obj = SomeBigObject::SomeBigObjectFactory(s2);
```

如果被调用的函数使用多余一个变量作为返回值，那么编译器也没法执行RVO:

```cpp
// RVO不会执行：
static SomeBigObject NonRvoFactory(...) {
  SomeBigObject object1, object2;
  object1.DoSomethingWith(...);
  object2.DoSomethingWith(...);
  if (flag) {
    return object1;
  } else {
    return object2;
  }
}
```

但如果被调用的函数在多个地方返回同一个变量，编译器会执行RVO：

```cpp
// RVO会执行：
SomeBigObject local;
if (...) {
  local.DoSomethingWith(...);
  return local;
} else {
  local.DoSomethingWith(...);
  return local;
}
```

关于RVO，以上大概就是所有情形了。

## 另一个事情：临时变量

除了命名变量，RVO也对临时变量执行。当被调用的函数返回临时变量时，你仍然可以从RVO中获益：

```cpp
// RVO会执行：
SomeBigObject SomeBigObject::ReturnsTempFactory(...) {
  return SomeBigObject::SomeBigObjectFactory(...);
}
```

当函数调用处（以临时变量的形式）直接使用返回值时，你仍然可以冲RVO中获益：

```cpp
// 不会打印“死贵”操作的消息：
EXPECT_EQ(SomeBigObject::SomeBigObjectFactory(...).Name(), s);
```

最后的注解：如果你的代码需要执行复制，那么久执行复制，不论这些复制会不会被优化掉。**不要牺牲正确性来换取效率**。