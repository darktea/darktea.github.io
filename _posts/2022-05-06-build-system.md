---
title: Build Systems
date: 2022-05-06 00:00:00 +0800
categories: [notes]
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

### 4. 更多细节

我们给出了一个 CMake 项目的常用结构，可以直接参考（其中的 CMakeLists.txt 中有详细的注释说明）：

<https://github.com/darktea/cmake-full-project>

更多细节备忘：

* CMake 中可以使用 list；同时提供了 **list 命令**来操控 list。例如：

```cmak
# 把 "${CMAKE_SOURCE_DIR}/cmake" 添加（append）到 CMAKE_MODULE_PATH 路径中去
# 其中：
# CMAKE_SOURCE_DIR 是当前项目的顶级目录
# CMAKE_MODULE_PATH 是 cmake module（cmake 模块）所在的目录
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")
```
