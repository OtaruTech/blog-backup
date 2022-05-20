---
title: C++性能测试工具：google benchmark 入门(1)
date: 2022-05-13 21:36:43
categories:
- C/C++
- benchmark

tags:
- benchmark

cover: /img/benchmark01.png
---

如果你正在寻找一款C++性能测试工具，那么此文不容错过。

市面上的benchmark工具或多或少存在一些使用上的不方便，那么是否存在一个使用简便又功能强大的性能测试工具呢？答案是[google/benchmark](https://github.com/google/benchmark)。

google/benchmark是一个由Google开发的基于googletest框架的C++ benchmark工具，它易于安装和使用，并提供了全面的性能测试接口。

下面来介绍google/benchmark的安装并用一个简短的例子介绍它的简单实用。

## 安装google/benchmark

google/benchmark基于C++11标准和googletest框架，所以安装前需要先做一些准备工作。

首先是g++和cmake

```powershell
sudo apt install g++ cmake
```

确保你的g++版本在5.0以上，否则可能不能很好的支持C++11的某些特性。

然后是googletest框架，你可以选择单独安装，不过这里我选择将其作为benchmark源码树的依赖而不单独安装它，因为benchmark在编译安装时需要googletest，但是在使用时并不需要。

选择一个合适的目录，然后运行下面的命令：

```powershell
git clone https://github.com/google/benchmark.git
git clone https://github.com/google/googletest.git benchmark/googletest
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=RELEASE ../benchmark
make -j4
# 如果想全局安装就接着运行下面的命令
sudo make install
```

头文件会被安装到`/usr/local/include`，库文件会被安装到`/usr/local/lib`。

安装完成之后，来看一下benchmark如何使用。

## google/benchmark的简单使用

这个例子将会对比三种访问`std::array`容器内元素方法的性能，进行演示benchmark的使用方法。

先看代码：

```cpp
#include <benchmark/benchmark.h>
#include <array>
 
constexpr int len = 6;
 
// constexpr function具有inline属性，你应该把它放在头文件中
constexpr auto my_pow(const int i)
{
    return i * i;
}
 
// 使用operator[]读取元素，依次存入1-6的平方
static void bench_array_operator(benchmark::State& state)
{
    std::array<int, len> arr;
    constexpr int i = 1;
    for (auto _: state) {
        arr[0] = my_pow(i);
        arr[1] = my_pow(i+1);
        arr[2] = my_pow(i+2);
        arr[3] = my_pow(i+3);
        arr[4] = my_pow(i+4);
        arr[5] = my_pow(i+5);
    }
}
BENCHMARK(bench_array_operator);
 
// 使用at()读取元素，依次存入1-6的平方
static void bench_array_at(benchmark::State& state)
{
    std::array<int, len> arr;
    constexpr int i = 1;
    for (auto _: state) {
        arr.at(0) = my_pow(i);
        arr.at(1) = my_pow(i+1);
        arr.at(2) = my_pow(i+2);
        arr.at(3) = my_pow(i+3);
        arr.at(4) = my_pow(i+4);
        arr.at(5) = my_pow(i+5);
    }
}
BENCHMARK(bench_array_at);
 
// std::get<>(array)是一个constexpr function，它会返回容器内元素的引用，并在编译期检查数组的索引是否正确
static void bench_array_get(benchmark::State& state)
{
    std::array<int, len> arr;
    constexpr int i = 1;
    for (auto _: state) {
        std::get<0>(arr) = my_pow(i);
        std::get<1>(arr) = my_pow(i+1);
        std::get<2>(arr) = my_pow(i+2);
        std::get<3>(arr) = my_pow(i+3);
        std::get<4>(arr) = my_pow(i+4);
        std::get<5>(arr) = my_pow(i+5);
    }
}
BENCHMARK(bench_array_get);
 
BENCHMARK_MAIN();
```

可以看到每一个benchmark测试用例都是一个类型为`std::function<void(benchmark::State&)>`的函数，其中`benchmark::State&`负责测试的运行及额外参数的传递。

随后我们使用`for(auto _ : state) {}`来运行需要测试的内容，state会选择合适的次数来运行循环，时间的计算从循环内的语句开始，所以我们可以选择像例子中一样在for循环之外初始化测试环境，然后再循环体内编写需要测试的代码。

测试用例编写完成之后我们需要使用`BENCHMARK(<function_name>);`将我们的测试用例注册进benchmark，这样程序运行时才会执行我们的测试。

最后用`BENCHMARK_MAIN();`替代直接编写的main函数，它会处理命令行参数并运行所有注册过的测试用例生成测试结果。

示例中大量使用了`constexpt`，这时为了能再编译期计算出需要的数值避免对测试产生太多噪音。

编译测试程序：

```powershell
g++ -Wall -std=c++14 benchmark_example.cpp -pthread -lbenchmark
```

benchmark需要链接libbenchmark.so，所以需要指定-lbenchmark，此外还需要thread的支持，因为libstdc++不提供thread的底层实现，我们需要pthread。

运行测试程序：

```bash
2022-05-11T17:17:16+08:00
Running ./a.out
Run on (24 X 3800 MHz CPU s)
CPU Caches:
  L1 Data 32 KiB (x12)
  L1 Instruction 32 KiB (x12)
  L2 Unified 512 KiB (x12)
  L3 Unified 16384 KiB (x4)
Load Average: 2.91, 3.07, 3.10
***WARNING*** CPU scaling is enabled, the benchmark real time measurements may be noisy and will incur extra overhead.
---------------------------------------------------------------
Benchmark                     Time             CPU   Iterations
---------------------------------------------------------------
bench_array_operator       19.8 ns         19.8 ns     35558602
bench_array_at             19.8 ns         19.8 ns     35103622
bench_array_get            20.2 ns         20.1 ns     34736601
```

显示的警告信息表示在当前系统环境有一些噪音可能导致结果不太准确，并不影响我们的测试。

测试结果与预期基本相符，`std::get`最快，`at()`最慢。