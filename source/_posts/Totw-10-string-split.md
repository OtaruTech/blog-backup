---
title: Tip of the Week 10 分割字符串，不必拘小节
date: 2022-03-21 21:23:12
tags:
- C++11
- StrSplit
- C++Tips
categories:
- C++ Tips of the Week
cover: /img/C++Tips-10.jpg
---

将字符串分割为子串是任何通用编程语言的常见操作，C++也不例外。这个需求出现在Google的时候，很多工程师发现他（她）们掉进了一堆（且不断增加的）“分割函数”填成的泥沼。你必须搜索那个输入参数、输出参数和语义都满足你要求的函数。在考察过600多行的头文件理的里的50多个函数之后，你也许会最终决定使用一个命名拐弯抹角的函数：

`SplitStringViewToDequeueIfStringAllowEmpty()`。

为了解决这个麻烦，C++库团队实现了一个新的API来分割字符串，放在了`absl/strings/str_split.h`里。

这个新的API用一个`absl::StrSplit()`函数取代了所有类似的分割函数。`absl::StrSplit()`接受一个准备被分割的字符串和一个分割符参数。其返回的子串集合可以自动适配为调用者接受的容器类型。`absl::StrSplit()`的内部实现使用`absl::string_view`，因此非常高效。除非调用者显式要求将结果存储为字符串对象集合，否则字符串内容不会被复制。

来看下下面的例子，比较直截了当：

```cpp
// 用逗号分割。结果存储为vector<string_view>（没有字符串内容的复制）。
std::vector<absl::string_view> v = absl::StrSplit("a,b,c", ',');

// 用逗号分割。结果存储为vector<string>（字符串内容复制一次）。
std::vector<std::string> v = absl::StrSplit("a,b,c", ',');

// 用子串"=>"分割（并非"="和">"选其一）
std::vector<absl::string_view> v = absl::StrSplit("a=>b=>c", "=>");

// 用任一给定的字符分割（','或';'）
using absl::ByAnyChar;
std::vector<std::string> v = absl::StrSplit("a,b;c", ByAnyChar(",;"));

// 结果存储在各种容器中（string可以替换为absl::string_view）
std::set<std::string> s = absl::StrSplit("a,b,c", ',');
std::multiset<std::string> s = absl::StrSplit("a,b,c", ',');
std::list<std::string> li = absl::StrSplit("a,b,c", ',');

// 等价于文中虚构的SplitStringViewToDequeOfStringAllowEmpty()
std::deque<std::string> d = absl::StrSplit("a,b,c", ',');

// 返回"a"->"1", "b"->"2", "c"->"3"
std::map<std::string, std::string> m = absl::StrSplit("a,1,b,2,c,3", ',');
```

本期字数较少，更多信息请参阅[absl/strings/str_split.h](https://github.com/abseil/abseil-cpp/blob/master/absl/strings/str_split.h)获取API的使用方法，参阅[absl/strings/str_split_test.cc](https://github.com/abseil/abseil-cpp/blob/master/absl/strings/str_split_test.cc)获取更多示例。