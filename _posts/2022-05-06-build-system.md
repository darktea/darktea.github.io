---
title: Build Systems
date: 2022-05-06 00:00:00 +0800
categories: [工具]
tags: [cmake, makefile, bazel]
---

## 一. CMake

### 1. Modern CMake

**Modern CMake**: 围绕着 targets 来管理对应用的构建。同时要成功构建 target，也需要配置 target 对应的 properties。

* **Target**：target 就是被构建应用的各个组件的抽象。一个可执行文件是一个 target，一个库是一个 target
  * 构建一个应用就是构建一系列 targets 的集合。而且这些 targets 可能互相依赖
* **Property**：需要对被构建的 target 配置 properties，才能成功进行构建：
  * target 是从哪些 source files 构建而来的
  * target 构建时的编译选项
  * target 需要链接哪些库

总之，**Modern CMake** 的目的就是创建一系列的 targets，同时定义这些 targets 必要的 properties。

### 2. Properties Scopes

**Target properties** 可以被定义在 2 种 scopes 中：**INTERFACE** and **PRIVATE**：

* **INTERFACE**：暴露给使用该 target 的用户使用的 properties
* **PRIVATE**：只是在内部构建该 target 时使用的 properties

另外，如果一个 property 同时是 **INTERFACE** 和 **PRIVATE** 的，那么这个 property 为 **PUBLIC**。

同时，**INTERFACE** property 会被透传给 target 的使用者；相反的，**PRIVATE** 的 property 不会被透传给 target 的使用者。

### 3. CMake 的职责

管理 C/C++ 应用。其中包括对从构建到发布整个生命周期中的各个步骤进行管理：

* Compiling executables and libraries
* Managing dependencies
* Testing
* Installing
* Packaging
* Producing documentation
* Testing some more

用户需要利用「CMake 脚本」来实现对应用的管理。内容为「CMake 脚本」的文件叫 **listfile**：

* CMakeLists.txt：在项目的顶层目录下，cmake 执行时从这个文件开始执行
* 以 cmake 为文件后缀的文件

### 4. 最佳实践

* 把一个大的 Project 进行拆分；而且是拆分成不同的目录，每个目录为项目的一个组成部分。例如：
  * 代码、测试、文档、外部依赖，脚本等
  * 而每一个组成部分又可以进行进一步拆分，代码目录可以再拆分成库代码，执行程序代码等等

> 可以使用 CMake 提供的 **add_subdirectory** 命令实现项目的拆分

### 5. 更多细节

我们给出了一个 CMake 项目的常用结构，可以直接参考（其中的 CMakeLists.txt 中有详细的注释说明）：

<https://github.com/darktea/cmake-full-project>

更多细节备忘：

* CMake 中可以使用 list；同时提供了 **list 命令**来操控 list。例如：

```cmake
# 把 "${CMAKE_SOURCE_DIR}/cmake" 添加（append）到 CMAKE_MODULE_PATH 路径中去
# 其中：
# CMAKE_SOURCE_DIR 是当前项目的顶级目录
# CMAKE_MODULE_PATH 是 cmake module（cmake 模块）所在的目录
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")
```

* CMake 的变量（variable）的作用域（scope）比较复杂；分为 2 种作用域：
  * Function Scope
  * Directory Scope

我们这里只简单的通过一个例子简单说明一下 Function 作用域：

```cmake
cmake_minimum_required(VERSION 3.20.0)
project(Scope)

function(Inner)
# 进入 inner 函数后，变量 V 的值还是上一层函数的值 2
  message("  > Inner: ${V}")
# 在函数中使用 set 命令会把 V 的值改为 3
  set(V 3)
# V 的当前值为 3
  message("  < Inner: ${V}")
endfunction()

function(Outer)
# 进入函数后，全局变量 V 的值不变，还是 1
  message(" > Outer: ${V}")
# 在函数中使用 set 命令会把 V 的值改为 2
  set(V 2)
# 调用下一级函数 inner
  Inner()
# 从 inner 函数返回以后，V 的值恢复到 2
  message(" < Outer: ${V}")
endfunction()

# 设置全局变量 V 的值为 1
set(V 1)
message("> Global: ${V}")

# 调用函数 Outer
Outer()

# 从函数 Outer 返回后，V 的值恢复到 1
message("< Global: ${V}")
```

运行这个例子可以得到结果：

```shell
> Global: 1
 > Outer: 1
  > Inner: 2
  < Inner: 3
 < Outer: 2
< Global: 1
```

另外，如果上面例子的 inner 函数改为：

```cmake
function(Inner)
  message("  > Inner: ${V}")
# 使用 PARENT_SCOPE 修改上一层函数中 V 的值；
# 但不会修改本函数和全局的 V 的值
  set(V 3 PARENT_SCOPE)
  message("  < Inner: ${V}")
endfunction()
```

运行结果如下（Outer 函数的打印为 Out: 3）：

```shell
> Global: 1
 > Outer: 1
  > Inner: 2
  < Inner: 2
 < Outer: 3
< Global: 1
```

