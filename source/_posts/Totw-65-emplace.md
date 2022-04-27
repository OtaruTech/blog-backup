---
title: Tip of the Week 65 就地安放
date: 2022-04-27 22:16:15
tags:
- C++11
- C++Tips
categories:
- C++ Tips of the Week
cover: /img/C++Tips-65.jpg
---

> “让我解释一下。不，这太多了。让我总结一下。” —— 埃尼戈·蒙托亚
> 

C++11添加了一种元素插入标准容器的新方法：`emplace()`方法系列。这些方法直接在容器中创建对象，而不是创建一个临时的对象，然后拷贝或者移动这个对象进容器。避免这些副本对于绝大多数对象而言都是更有效率的，并且在标准容器中存储只移动的对象（例如`std::unique_ptr`）也会更容易。

## 旧方法和新方法

让我们来看一下两种使用`vector`的方法，第一个示例使用C++11之前的代码：

```cpp
class Foo {
 public:
  Foo(int x, int y);
  …
};

void addFoo() {
  std::vector<Foo> v1;
  v1.push_back(Foo(1, 2));
}
```

使用这些旧的`push_bach`方法，有两个foo对象被构造：这临时的参数和在vector中由临时对象通过移动构造函数构造对象。

我们能够用C++11中的`emplace_back`代替，并且只有一个对象在vector的内存中被直接构造。由于“emplace”系列函数转发他们的参数给接下来的对象构造函数，因此我们直接提供构造函数参数，避免需要创建一个临时Foo

```cpp
void addBetterFoo() {
  std::vector<Foo> v2;
  v2.emplace_back(1, 2);
}
```

## 对只移动操作使用emplace方法

到目前为止，我们已经看到了许多`emplace`方法提供性能的情形，但是它们也能使得之前不可能的代码变得可行，例如在容器中存储像`std::unique_ptr`这种只能移动的类型。例如：

```cpp
std::vector<std::unique_ptr<Foo>> v1;
```

那么你将如何插入值到这种vector呢？一种方法是使用`push_back`，并且直接在他的参数中构造值：

```cpp
v1.push_back(std::unique_ptr<Foo>(new Foo(1, 2)));
```

这语法能够工作，但是有点累赘。不幸的是，绕过这种混淆的传统方法充满了复杂性。

```cpp
Foo *f2 = new Foo(1, 2);
v1.push_back(std::unique_ptr<Foo>(f2));
```

这段代码能够编译，但是在插入之前，它使得原生指针的所有权不清晰。更糟糕的是，vector现在拥有了此对象，但是f2依然保持有效，并且可能在以后被意外的删除。对于不知情的读者而言，这种所有权模式可能令人困扰，特别是如果构造和插入不是如上述的连续事件。

其他解决方甚至无法编译通过，因为`unique_ptr`不能复制。

```cpp
std::unique_ptr<Foo> f(new Foo(1, 2));
v1.push_back(f);             // Does not compile!
v1.push_back(new Foo(1, 2)); // Does not compile!
```

使用`emplace`方法可以更直接的在创建对象时插入它。在其他情况下，如果你需要移动`unique_ptr`进入vector，可以：

```cpp
std::unique_ptr<Foo> f(new Foo(1, 2));
v1.emplace_back(new Foo(1, 2));
v1.push_back(std::move(f));
```

结合`emplace`和标准`iterator`，你也能够在vector任意位置插入对象：

```cpp
v1.emplace(v1.begin(), new Foo(1, 2));
```

也就是说，实际上我们不想看到这些构造`unique_ptr`的方法——使用`std::make_unique`或`absl::make_unique`

## 结论

我们在本贴中使用vector作为示例，但是`emplace`方法也可以用于map，list和其他STL容器。当与`unique_ptr`组合使用时，`emplace`能有更好的封装，并且使得堆分配对象使用权语义这种方式变得更清晰，这在以前是不能可能的。