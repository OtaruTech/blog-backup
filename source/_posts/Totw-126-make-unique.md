---
title: Tip of the Week 126 ‘make_unique’是新的‘new’
date: 2022-05-08 10:36:36
tags:
- C++11
- C++Tips
categories:
- C++ Tips of the Week
cover: /img/C++Tips-126.jpg
---
随着代码库的扩展，越来越难以了解你依赖的每件事的细节。需要深入的知识无法扩展：我们必须依靠接口和契约来知道代码是正确的，无论是在写还是在审查代码。在许多情况下，类型系统可以用一种通用的方式来提供这些契约。类型系统契约使用的一致性，通过识别在堆上分配的对象存在潜在风险分配或所有权转移的位置，可以更轻松的编写和审查代码。

虽然在C++中，我们可以通过使用纯值来减少动态内存分配的需求，但是有时我们需要对象的声明周期超过其作用域。在动态分配对象时，C++代码应该优先使用智能指针（最常见的`std::unique_ptr`）而不是原生指针。这提供了关于分配和所有权转移一致性，并在那些需要更仔细审查代码所有权问题的地方留下了更清晰地视觉提示。满足C++14之后，在外部如何分配及异常安全上的副作用只是小事。

关于此的两个关键工具是`absl::make_unique()`（C++14的`std::make_unique`的C++11实现，用于免泄漏动态分配）和`absl::WrapUnique()`（用于包装拥有指向相应`std::unique_ptr`类型的原生指针）。他们可以再`absl/memory/memory.h`中找到。

## 为什么要避免new？

为什么代码要优先使用智能指针和分配函数而不是原生指针和`new`呢？

1. 如果可能，所有权最好表达在类型系统中。这允许审查者几乎完全通过本地检查来验证正确性（没有泄漏和重复删除）。（在堆性能异常敏感的代码中，这可能是可能原谅的：虽然代码小，但是由于`ABI`的约束，以值的方式阔函数传递`std::unique_ptr`的开销是非零。这不够重要到证明需要避免它。）
2. 有点像优先`push_back()`而不是`emplace_back()`的原因（TotW 112），`absl::make_unique()`直接表达了目的并且只能做一件事（使用公共构造函数分配，返回指定类型的`std::unique_ptr`）。这里没有类型转换或隐式行为。`absl::make_unique`在做明面上所讲的。
3. 同样可以用`std::unique_ptr my_t(new T(args))`来完成；但是这时多余的（重复类型名称T），对于某些人来说，尽量减少对`new`的调用是有价值的。
4. 如果所有分配都通过`absl::make_unique()`或工厂调用处理，则将`absl::WrapUnique()`用于实现这些工厂调用，用于与不依赖`std::unique_ptr`所有权的传统方法交互的代码转移，以及在极少数情况下需要通过聚合初始化动态分配（`absl::WrapUnique(new MyStruct{3.141, “pi”})`）。在代码审查中，很容易发现`absl::WrapUnique`调用并评估“表达式看起来像所有权转移吗？”通常很明显（例如，它是一些工厂函数）。当它不明显时，我们需要检查函数以确保它实际上是原始指针所有权转移。
5. 如果我们主要依赖`std::unique_ptr`的构造函数，那么我们会看到如下的调用：`std::unique_ptr foo(Blah());std::unique_ptr bar(new T());`只需要片刻的检查就可以看到后者是安全的(没有泄露，没有重复删除)。前者呢？它取决于:如果`Blah()`返回一个`std::unique_ptr`,那很好，尽管如此，如果写成下面那样会更安全`std::unique_ptr foo = Blah();`如果`Blah()`是返回一个传递了所有权的原生指针，那也是可以的。如果`Blah()`只返回一些随机指针(没有传递)，那么就有问题了。依赖`absl::make_unique()` 和 `absl::WrapUnique()`(避免构造函数)为我们担心的地方提供了额外的视觉提示()(调用 `absl::WrapUnique()`，并且仅仅如此)。

## 我们应该如何选择使用哪一个呢？

1. 默认情况下，使用`absl::make_unique()`（或者用`std::make_shared()`来处理那些共享所有权的烧友的情况下是合适的）来动态分配。例如，使用`auto bar = absl::make_unique()`替代`std::unique_ptr bar(new T());`并且用`bar = absl::make_unique()`; 来替代`bar.reset(new T());`
2. 在使用非公有函数的工厂函数中，返回`std::unique_ptr`，在实际中使用`absl::WrapUnique(new T(...))`。
3. 当动态分配一个需要用大括号初始化（典型的结构体，数据和容器）的对象时，使用`absl::WrapUnique(new T{...})`。
4. 当调用一个通过`T*`接受所有权的传统API时，要么提前使用`absl::make_unique`分配对象然后调用`ptr.release()`，要么直接在函数参数中使用`new`。
5. 当调用一个通过`T`返回所有权的传统API时，请立即使用`WrapUnique`构造一个智能指针（处分你打算立即传递一个指针到另外一个接受T形式所有权的传统API）。

## 总结

相较于`absl::WrapUnique()`请优先使用`absl::make_unique()`，相较于原生`new`优先使用`absl::WrapUnique()`。