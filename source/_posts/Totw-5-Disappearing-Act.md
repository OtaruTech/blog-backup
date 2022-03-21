---
title: Tip of the Week 5 消逝的演出
date: 2022-03-21 20:27:35
tags:
- C++11
- C++Tips
categories:
- C++ Tips of the Week
cover: /img/C++Tips-5.png
---

有时候，为了正确的运用C++的库，你既需要理解库本身，又需要理解这门语言。那么......下面的代码中的问题是什么？

```cpp
// 别这么干
std::string s1, s2;
...
const char* p1 = (s1 + s2).c_str();             // 别！
const char* p2 = absl::StrCat(s1, s2).c_str();  // 别！
```

`s1+s2`和`absl::StrCat(s1, s2)`都创建了临时对象（这里都是字符串对象，但同样的规则使用于任意对象）。成员函数`c_str()`返回指向底层数据的指针，而底层数据与临时对象生命周期一致。临时对象能活多长？根据C++17标准中的`[class temporary]`，“在临时对象创建点所在的完整表达式中，临时变量的销毁时表达式的最后一步。”，那么也就是说，当赋值运算符右边的表达式结束的时候，临时变量就被销毁了，`c_str()`的返回值就成了“野”指针。那么如何避免这类问题呢？

## 方法一，在完整表达式结束前用完临时对象：

```cpp
// 安全（虽然弱鸡了一点）
size_t len1 = strlen((s1 + s2).c_str());
size_t len2 = strlen(absl::StrCat(s1, s2).c_str());
```

## 方法2，存储临时对象：

既然你都（在栈上）创建对象了，干嘛不多留他一会呢？这可能比他初看上去便宜。因为一个叫“返回值优化”的玩意儿，临时变量会在赋值目标对象上直接构造，而不是复制：

```cpp
// 安全（且比你想象的更高效）
std::string tmp_1 = s1 + s2;
std::string tmp_2 = absl::StrCat(s1, s2);
// tmp_1.c_str()和tmp_2.c_str()是安全的。
```

## 方法3，存储一个指向临时变量的引用：

C++标准`[class temporary]`：“若临时变量绑定到引用，或临时变量的子对象绑定到引用，则临时变量声明周期会被延长到与该引用一致。”

因为返回值优化的存在，这种方式常并不比存储对象背身（方法2）更“便宜”，而且还有可能给人整蒙圈。

```cpp
// 同等安全：
const std::string& tmp_1 = s1 + s2;
const std::string& tmp_2 = absl::StrCat(s1, s2);
// tmp_1.c_str()和tmp_2.c_str()是安全的。

// 如下的行为徘徊在危险的边缘：
// 如果编译器能看出你是在存储一个指向临时对象内部的引用，它就会让整个对象活着。
// struct Person { string name; ... }
// GeneratePerson()返回一个对象；GeneratePerson().name显然是个子对象：
const std::string& person_name = GeneratePerson().name; // 安全

// 如果编译器看不出来，那你就危险了。
// class DiceSeries_DiceRoll { `const string&` nickname() ... }
// GenerateDiceRoll()返回一个对象；编译器可看不出来GenerateDiceRoll().nickname()是不是个子对象。
// 如下代码可能存储了一个悬挂引用：
const std::string& nickname = GenerateDiceRoll().nickname(); // 不好!
```

## 方法4，设计函数的时候就别返回对象？？？

很多函数遵循这条原则；但也有很多函数不遵守。相比要求调用者传进一个指向输出参数的指针，有时候返回个对象真的更好。在创建临时对象的地方多加小心，在操作临时对象的时候，任何返回对象内部的指针或者引用的东西都有可能出问题。`c_str()`是最明显的罪魁祸首，但是protobuf的（或其他的可修改）访问器（getter）和其他通常的访问器也同样有可能出问题。