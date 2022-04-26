---
title: Tip of the Week 88 初始化：=,()和{}
date: 2022-04-25 11:55:36

tags:
- C++11
- C++Tips
- 初始化
categories:
- C++ Tips of the Week
cover: /img/C++Tips-88.png
---

# Tip of the Week#88: 初始化：=,()和{}

C++11提供了一种称为”统一初始化语法”的新语法，它被认为统一所有不同风格的初始化，避免最麻烦人的解析，并避免窄化转换。这种新机制意味着我们现在有另一种初始化语法，它有自己的权衡。

## C++11括号初始化

一些统一初始化语法的支持者会建议我们使用{}和直接初始化(不适用’=’，尽管在多数情况下两种形式调用相同的构造函数)来初始化所有类型：

```cpp
int x{2};
std::string foo{"Hello World"};
std::vector<int> v{1, 2, 3};
```

对比（例如）：

```cpp
int x = 2;
std::string foo = "Hello World";
std::vector<int> v = {1, 2, 3};
```

这种方法有两个特点。首先，“统一”是一个延伸概念：在某些情况下，在调用什么以及如何调用这方面依然存在歧义(针对普通读者，而不是编译器)。

```cpp
std::vector<std::string> strings{2}; // 两个空字符串的vector
std::vector<int> ints{2};            // 仅仅一个整数2的vector
```

其次：这种语法并不完全直观：没有其他通用语法像这样使用。这种语言当然可以引入新的令人惊讶的语法，并且在某些情况下有必要使用它的技术原因——尤其是在通用代码中。重要的问题是：我们应该在多大程度上改变我们的习惯和语言理解来利用这种改变？改变我们的习惯或存在的代码，这种代码是否值得？对于统一初始化语法，我们一般不认为利大于弊。

## 关于初始化的最佳时间

相反，我们推荐下面的指南来进行“如何初始化一个变量？”，既可以再你自己的代码中遵循，也可以在你的代码审查中引用：

- 当用预期的字面量(例如：`int`, `float`或`std::string`值)、智能指针(例如：`std::shared_ptr`, `std::unique_ptr`)、容器(`std::vector`, `std::map`等)来直接初始化时，当执行结构体初始化或进行深拷贝时，请使用赋值语法。

```cpp
int x = 2;
std::string foo = "Hello World";
std::vector<int> v = {1, 2, 3};
std::unique_ptr<Matrix> matrix = NewMatrix(rows, cols);
MyStruct x = {true, 5.0};
MyProto copied_proto = original_proto;
```

而不是：

```cpp
// 糟糕的代码
int x{2};
std::string foo{"Hello World"};
std::vector<int> v{1, 2, 3};
std::unique_ptr<Matrix> matrix{NewMatrix(rows, cols)};
MyStruct x{true, 5.0};
MyProto copied_proto{original_proto};
```

- 当初始化正在执行一些活动的逻辑时，使用传统的构造函数语法(用括号)，而不是简单的将值组合在一起。

```cpp
Frobber frobber(size, &bazzer_to_duplicate);
std::vector<double> fifty_pies(50, 3.14);
```

对比

```cpp
// 糟糕的代码
// 可以调用一个初始化列表构造函数，或者一个两个参数的构造函数 
Frobber frobber{size, &bazzer_to_duplicate};

// 制作一个2个double的vector
std::vector<double> fifty_pies{50, 3.14};
```

- 只有以上选项不能编译时，才使用{}初始化，而不用=

```cpp
class Foo {
 public:
  Foo(int a, int b, int c) : array_{a, b, c} {}

 private:
  int array_[5];
  // 因为构造函数标记为explicit并且是非拷贝类型，因此需要{}
  EventManager em{EventManager::Options()};
};
```

- 绝不滥用{}和auto
    
    例如，不要这样子：
    

```cpp
// 糟糕的代码
auto x{1};
auto y = {2}; // 这是std::initializer_list<int>!
```

（对于语言专家而言：如果可行，有限使用拷贝初始化而不是直接初始化，并在直接初始化时使用圆括号而不是大括号。）

也许对这个问题最完整的描述是**Herb Sutter的GotW贴文**。尽管他展示了一个直接用大括号初始化`int`的示例，但是他最终的建议大致与我们再次提出的建议相符：Herb说“在哪里你更愿意只看到-符号”，我们毫不含糊的有限确定地看到那个。结合更加一直的在多参数构造函数上使用explicit，这提供了在可读性、确切性和正确性之间的平衡。

## 结论

统一初始化语法的权衡结果通常是不值得的：我们的编译器已经警告了最烦人的解析(你可以使用大括号初始化或者添加括号来解决这个问题)，并且窄化转换的安全性不值得为可读性去使用大括号初始化(我们最终需要一个不同的解决方案来进行窄化转换)。风格仲裁者任务这个问题不足以制作一个正式的规则，特别是因为在某些情况下(尤其是在通用代码中)大括号初始化可能是合理的。