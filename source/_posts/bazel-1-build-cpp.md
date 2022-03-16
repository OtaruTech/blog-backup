---
title: bazel构建C++工程
date: 2021-12-24 08:01:40
categories:
- C/C++
- bazel

tags:
- bazel
cover: /img/bazel.jpg
---

## 1. bazel介绍

Bazel是一个开源的构建和测试工具，类似于Make、Maven和Gradel。Bazel支持多种语言的项目，并未多种平台构建输出。Bazel支持跨多个存储库和大量用户的大型代码库。

<!--more-->

## 2. bazel安装

bazel安装有两种方法，一种是通过安装，另一个是通过下载安装包本地安装。ubuntu系统建议使用第一种安装方式。

第一步：添加bazel分发url作为包源

```bash
sudo apt install curl gnupg
curl https://bazel.build/bazel-release.pub.gpg | sudo apt-key add -
echo "deb [arch=amd64] https://storage.googleapis.com/bazel-apt stable jdk1.8" | sudo tee /etc/apt/sources.list.d/bazel.list
```

第二步：安装或者更新bazel

```bash
sudo apt update && sudo apt install bazel
```

如果已经安装了bazel，需要升级到最新版本的bazel

```bash
sudo apt update && sudo apt full-upgrade
```

安装特定版本的bazel，例如安装bazel-1.0.0

```bash
sudo apt install bazel-1.0.0
```

## 3. bazel构建工程

bazel里面有构建C++工程的例子，可以通过命令`git clone [https://github.com/bazelbuild/examples](https://github.com/bazelbuild/examples)`下载，需要的代码位于example/cpp-tutorial目录中，目录结构如下：

```bash
cpp-tutorial/
|-- README.md
|-- stage1
|   |-- README.md
|   |-- WORKSPACE
|   `-- main
|       |-- BUILD
|       `-- hello-world.cc
|-- stage2
|   |-- README.md
|   |-- WORKSPACE
|   `-- main
|       |-- BUILD
|       |-- hello-greet.cc
|       |-- hello-greet.h
|       `-- hello-world.cc
`-- stage3
    |-- README.md
    |-- WORKSPACE
    |-- lib
    |   |-- BUILD
    |   |-- hello-time.cc
    |   `-- hello-time.h
    `-- main
        |-- BUILD
        |-- hello-greet.cc
        |-- hello-greet.h
        `-- hello-world.cc
```

下面对这个C++工程进行解析。

### 3.1 设置工作区

在构建项目之前，需要设置项目工作区，这个工作区就是一个包含项目源文件的目录和bazel构建输出，bazel识别`WORKSPACE`和`BUILD`这两个文件。

- WORKSPACE文件将目录及其内容识别为Bazel工作区，位于项目目录结构的根目录中
- BUILD文件会有一个或者多个，分别构建项目的不同部分

### 3.2 理解BUILD文件

`BUILD`文件包含Bazel的几种不同类型的指令，最重要的啥类型是构建规则，它告诉bazel如何构建所需的输出，比如可执行的二进制文件或者动态/静态库。构建文件中构建规则的每个实例称为目标，并指向一组特定的源文件和依赖项。一个目标也可以指向其它目标。

看一下`cpp-tutorial/stage1/main`中BUILD文件：

```python
cc_binary(
    name = "hello-world",
    srcs = ["hello-world.cc"],
)
```

在示例中，`hello-world`目标实例化了bazel的内置`cc_binary`规则。规则告诉bazel建立一个独立的可执行二进制`hello-world.cc`源文件没有依赖性。

### 3.3 使用bazel编译项目

构建示例项目，切换到cpp-tutorial/stage1目录，运行以下命令：

```python
bazel build //main:hello-world
```

注意目标标签 —— `//main:`部分是构建文件相当于工作区根的位置，`helllo-world`是我们在构建文件中命名的目标。

```bash
bazel build //main:hello-world
Starting local Bazel server and connecting to it...
INFO: Analyzed target //main:hello-world (15 packages loaded, 57 targets configured).
INFO: Found 1 target...
Target //main:hello-world up-to-date:
  bazel-bin/main/hello-world
INFO: Elapsed time: 2.342s, Critical Path: 0.30s
INFO: 6 processes: 4 internal, 2 processwrapper-sandbox.
INFO: Build completed successfully, 6 total actions
```

测试输出如下：

```bash
./bazel-bin/main/hello-world
Hello world
Thu Dec 23 13:24:20 2021
```

### 3.4 查看依赖图

成功的构建将在构建文件中现实的生命其所有依赖项。Bazel使用这些语句创建项目的依赖关系图，从而实现精确的增量构建。将示例项目的依赖关系可视化。首先，生成依赖关系图的文本表示：

```bash
bazel query --notool_deps --noimplicit_deps "deps(//main:hello-world)" \
  --output graph
```

上面的命令告诉bazel寻找target `//main:hello-world`的所有依赖项，并将输出格式化为图形，然后可以使用`GraphViz`查看。

可以通过管道直接输出到xdot来生成和查看图形：

```bash
sudo apt update && sudo apt install graphviz xdot
```

生成图形：

```bash
xdot <(bazel query --notool_deps --noimplicit_deps "deps(//main:hello-world)" \
  --output graph)
```

依赖图如下：

![bazel.drawio.png](bazel%E6%9E%84%E5%BB%BAC++%E5%B7%A5%E7%A8%8B%20986aa39efa9347cfbbc0723640f6d72d/bazel.drawio.png)

### 3.5 多个目标文件(target)编译

将示例项目构建分成两个目标。看一下`cpp-tutorial/stage2/`主目录中的构建文件：

```python
cc_library(
    name = "hello-greet",
    srcs = ["hello-greet.cc"],
    hdrs = ["hello-greet.h"],
)

cc_binary(
    name = "hello-world",
    srcs = ["hello-world.cc"],
    deps = [
        ":hello-greet",
    ],
)
```

对于这个构建文件，bazel首先构建`hello-greet`库(使用bazel内置的cc_library规则)，然后构建`hello-world`二进制文件。`hello-world`目标中`deps`属性告诉bazel需要`hello-greet`库来构建`hello-world`二进制文件。

切换到cpp-tutorial/stage2目录。运行：

```bash
bazel build //main:hello-world
Starting local Bazel server and connecting to it...
INFO: Analyzed target //main:hello-world (15 packages loaded, 60 targets configured).
INFO: Found 1 target...
Target //main:hello-world up-to-date:
  bazel-bin/main/hello-world
INFO: Elapsed time: 1.974s, Critical Path: 0.20s
INFO: 7 processes: 4 internal, 3 processwrapper-sandbox.
INFO: Build completed successfully, 7 total actions
```

依赖关系如下：

![bazel2.drawio.png](bazel%E6%9E%84%E5%BB%BAC++%E5%B7%A5%E7%A8%8B%20986aa39efa9347cfbbc0723640f6d72d/bazel2.drawio.png)

### 3.6 多个项目包(packages)编译

将项目分离成多个包，看一下cpp-tutorial/stage3目录的内容：

```bash
|-- README.md
|-- WORKSPACE
|-- lib
|   |-- BUILD
|   |-- hello-time.cc
|   `-- hello-time.h
`-- main
    |-- BUILD
    |-- hello-greet.cc
    |-- hello-greet.h
    `-- hello-world.cc
```

现在有两个子目录，每个子目录都包含一个`BUILD`文件。因此，对于bazel来说，工作空间现在包含两个包，`lib`和`main`，先看一下`lib/BUILD`文件

```python
load("@rules_cc//cc:defs.bzl", "cc_library")

cc_library(
    name = "hello-time",
    srcs = ["hello-time.cc"],
    hdrs = ["hello-time.h"],
    visibility = ["//main:__pkg__"],
)
```

还有`main/BUILD`文件：

```python
load("@rules_cc//cc:defs.bzl", "cc_binary", "cc_library")

cc_library(
    name = "hello-greet",
    srcs = ["hello-greet.cc"],
    hdrs = ["hello-greet.h"],
)

cc_binary(
    name = "hello-world",
    srcs = ["hello-world.cc"],
    deps = [
        ":hello-greet",
        "//lib:hello-time",
    ],
)
```

主包中的`hello-world`目标依赖于lib包中的hello-time目标(因此是目标标签`//lib:hello-time`)——bazel通过`deps`属性知道这一点。依赖关系：

![bazel3.drawio.png](bazel%E6%9E%84%E5%BB%BAC++%E5%B7%A5%E7%A8%8B%20986aa39efa9347cfbbc0723640f6d72d/bazel3.drawio.png)

注意，为了使构建成功，使用`visibility`属性使得`lib/BUILD`中的`//lib:hello-time`目标对于`main/BUILD`中的目标显示可见。这是因为默认情况下，目标只对同一构建文件中的其它目标可见。(Bazel使用目标可见性来防止诸如库中包含的实现细节泄漏到公共API等问题)

编译和运行和上面一致：

```bash
bazel build //main:hello-world
Starting local Bazel server and connecting to it...
INFO: Analyzed target //main:hello-world (16 packages loaded, 63 targets configured).
INFO: Found 1 target...
Target //main:hello-world up-to-date:
  bazel-bin/main/hello-world
INFO: Elapsed time: 2.743s, Critical Path: 0.20s
INFO: 8 processes: 4 internal, 4 processwrapper-sandbox.
INFO: Build completed successfully, 8 total actions

./bazel-bin/main/hello-world
Hello world
Thu Dec 23 13:58:17 2021
```

## 参考

1. [Google开源构建工具Bazel](https://docs.bazel.build/versions/3.4.0/install-ubuntu.html)
2. [Introduction to Bazel: Build a C++ Project](https://docs.bazel.build/versions/3.4.0/tutorial/cpp.html)