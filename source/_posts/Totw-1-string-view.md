---
title: Tip of the Week #1: string_view
date: 2022-03-16 20:35:20
tags:
- C++11
- string_view
- C++Tips
categories:
- C++ Tips of the Week
cover: /img/C++Tips-1.png
---

参考链接：[***abseil.io/tips/1***](https://abseil.io/tips/1)

## 什么是string_view，为什么需要string_view

当你在把一个字符串（或者字符串常量）作为一个函数参数的时候，通常有三种选择：前面两种你肯定是知道的，剩下的你可能没有意识到：

```cpp
// C Convention
void TakesCharStar(const char* s);

// Old Standard C++ convention
void TakesString(const std::string& s);

// string_view C++ conventions
void TakesStringView(absl::string_view s);    // Abseil
void TakesStringView(std::string_view s);     // C++17
```

当调用者传入字符串格式的参数来调用函数，前面两种方式是可以正常工作的，但是在这个过程中是如何做转化的呢？无论是从`const char*` 到 `std::string`还是从`std::string`到`const char*`?

调用者需要把`std::string`转化为`const char*`，虽然说效率很高，但是还是需要调用`c_str()`函数，不是很方便：

```cpp
void AlreadyHasString(const std::string& s) {
  TakesCharStar(s.c_str());               // explicit conversion
}
```

在下面的代码中，调用者需要把`const char*`转化为`std::string`虽然说在代码上不需要做额外的事情，但是在转化的过程中，会创建一个临时的字符串，并且拷贝到`string`对象中：

```cpp
void AlreadyHasCharStar(const char* s) {
  TakesString(s); // compiler will make a copy
}
```

## 该怎么做呢？

Google建议选择使用`string_view`来处理这种字符串参数。标准C++17是支持`std::string_view`的，在这之前可以使用`absl::string_view`，但是需要注意的是，他们不是完全兼容的，不能直接做替换。

一个`string_view`实例可以看作是一个字符串缓存的描述。明确的说，一个`string_view`包含了一个指向字符串buffer的指针和字符串长度值，也就是说这个`string_view`不是这个字符串的所有者并且不能修改字符串内容。因此，复制一个`string_view`是一个很轻量级的操作：没有字符串数据拷贝！

`string_view`有一个隐式的构造函数传入`const char*`或者`const std::string&`，由于`string_view`没有做拷贝的动作，所以在内存空间消耗为O(n)。当传入的是一个`const std::string&`，构造函数时间消耗为O(1)，当传入的是一个`const char*`类型时，构造函数会调用strlen()来获取字符串长度。

```cpp
void AlreadyHasString(const std::string& s) {
  TakesStringView(s); // no explicit conversion; convenient!
}

void AlreadyHasCharStar(const char* s) {
  TakesStringView(s); // no copy; efficient!
}
```

因为`string_view`没有真正拥有字符串数据，任何指向字符串的`string_view`对象必须知道指向数据的生命周期，必须必`string_view`长。这就意味着用`string_view`来做存储是有问题的：必须证明真正的数据的生命周期比`string_view`长。

如果你的API在一个函数调用中只需要引用字符串数据，不需要修改数据时，使用`string_view`是可以的。如果还需要修改字符串数据，可以隐式的使用`std::string(my_string_view)`来转化为一个C++的`string`对象。

在一个现有项目的代码中添加`string_view`往往不是一个很好的方案，在函数的参数中把`std::string`替换为`string_view`往往会提升效率，但是最好在项目初期就考虑使用`string_view`。

## 一些额外需要注意的地方

- 不像其他的字符串类型，在函数传递的时候应该直接传入`string_view`的值，就像`int`和`double`一样，因为`string_view`是个很小的值。
- 将`string_view`设置为`const`只会影响`string_view`对象本身是否可以修改，不会影响`string_view`指向的真正字符串数据是否可以修改。这和`const char*`不能用来修改字符数据的方式是一样的，即使指针本身可以修改。
- 对于函数参数，不要再函数中不要使用`const`来声明`string_view`参数。
- string_view不一定是以空字符结尾的，因此下面的写法是不安全的：
    
        `printf("%s\n", sv.data()); // DON’T DO THIS`
    
    用一下方式代替：
        `absl::PrintF("%s\n", sv);`
    
- 使用如下方式打印`string_view`日志
    `std::cout << "Took '" << s << "'";`
- 在大多数情况下把`const std::string&`或者以空结尾的字符串转化为`string_view`都是可以接受的。唯一有风险的是当被用作函数指针的时候。
- `string_view`有一个`constexpr`的构造函数和不是很重要的析构函数，所以在静态和全局变量使用的时候需要记住这一点。