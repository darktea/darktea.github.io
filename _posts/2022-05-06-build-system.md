---
title: Build Systems
date: 2022-05-06 00:00:00 +0800
categories: [工具]
tags: [cmake, makefile, bazel, c]
---

## 一. CMake

### 1. Modern CMake

**Modern CMake**: 围绕着 targets 来管理应用的构建。同时，要成功构建 target，也需要配置 target 对应的 properties。

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

* CMake 提供了一个 **Unit Test 规范（CTest）**，只要按照 CTest 规范进行配置，可以集成各种 Unit Test 框架。从而实现 Unit Test 的自动化运行
  * 推荐使用 GTest 框架，能够和 CMake 有很好的配合
  * C 语言的话，我现在使用 [Tau](https://github.com/jasmcaus/tau)

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

### 6. Reference

* [Modern CMake is like inheritance](https://kubasejdak.com/modern-cmake-is-like-inheritance)

## 二. C 语言相关

### 概念

#### Value

* 从「可移植」方面来考量的话，就 C 语言中的数据来说，作为一个 programmer，应该尽可能的只关注 [value](#value) 本身，而不是 [value](#value) 的具体 **representation**
  * C programs primarily reason about [value](#value) and **NOT** about their **representation**.
  * 这个道理就类似现实世界中，同一个数在不同的语言（例如用罗马数字来表示数字）中有不同的表达方式，但我们关注的是数本身，而不是表达方式

* C 中，[value](#value) 是抽象实体，抽象于特定的实现，和特定的运行之上
* [value](#value) 和 representation 之前的相互转换是编译器应该关注的事情
* C 程序可以抽象成「有序抽象状态机」。「抽象状态机」的运行理想上是和平台无关的，「状态机」中和平台无关的相关概念包括 [value](#value)、object、[type](#type)：
  * 所谓「有序抽象状态机」就是用来操作 [value](#value)（在给定时刻，程序的变量，variable，拥有特定的 [value](#value)）
  * **[value](#value)**：就是「值」的抽象概念，和具体的程序是无关的；而且 C 中的 [value](#value) 都是「数值」（或者可以被转换为「数值」）

「状态机」状态的转换由 3 者决定： [value](#value)、[type](#type)、[Binary Representation](#binary-representation)

#### Type

* [type](#type)：C 语言中，[type](#type) 是 [value](#value) 的属性
  
  * 所有的 [value](#value) 的 [type](#type) 都静态的决定（非运行时决定）；所谓的「静态类型」
  * 在一个 [value](#value) 上可以执行什么操作，由 [value](#value) 的 [type](#type) 决定
  * [value](#value) 上操作的结果是什么会被 [type](#type) 决定（但也不能完全被 [type](#type) 决定，结果也受 [Binary Representation](#binary-representation) 的影响）
  * 编译器是否能做一些优化，也受到 [type](#type) 的影响（例如，对 unsigned int 上的操作做优化时，可以不考虑 overflow）
  
#### Object

澄清几个概念：

* **[object](#object)**：可以被看成一种 box 本身
* **identifier**：用来标识 box 的 name
* **[type](#type)**：如果 [object](#object) 看成是 box 的话，那么 [type](#type) 就是 box 的 specification
* **[value](#value)**：如果 [object](#object) 看成 box 的话，[value](#value) 就是放在 box 中的 content

> 赋值语句可以理解为：把 value 赋值给 object（把 content 放进 box）
>
> C 语言中，可能是不需要引入「左值」、「右值」的概念；没有那么复杂，直接用 object、value 的概念就行了。
>
> 为什么？请参考下面的讨论……

C 语言里面的「左右值」的讨论：

* 先不考虑 function 的场景，C 语言中的 lvalue 中的 “l” 代表的意思不是「左」的意思，是「地址可被定位」的意思：
  * lvalue simply means an [object](#object) that has an identifiable **location** in memory
  * 「左值」就是 [object](#object)
  * 所谓「右值」，在 C 语言中，其实是「非左值」

一个 C 语言中「非左值」的例子：

```c
typedef struct S {
  int i;
} S;

S f() {
  S s;
  s.i = 123;
  return s;
}

int main() {
  S s;      // 这里的 s 是「左值」
  s = f();  // 这里的 f() 就是一个「非左值」
}
```

#### Binary Representation

* [Binary Representation](#binary-representation)：[type](#type) 的二进制表示。例如对一个 16 位的 unsigned int，其二进制表示就很简单：直接用 16 位 bits（b<sub>0</sub>, b<sub>1</sub>...b<sub>15</sub>）表示就行了。一般来说是具体平台无关，也是一种可抽象的概念
  * 但由于各种平台的差异，C 的标准并不能完全控制所有的具体实现细节；也就是说，即使遵循了 C 标准，也不能完全保证同样的操作在所有平台上的结果完全一致。
  * [Binary Representation](#binary-representation) 也会对在 [value](#value) 上的操作的结果造成影响
  * 但这里的 [Binary Representation](#binary-representation) 还是一个抽象的概念，并没有决定 [value](#value) 物理上具体怎么存储的。[value](#value) 物理上怎么存储的是 [Object Representation](#object-representation) 相关的概念

举个例子：

> C 标准说明了 `size_t` 这个 [type](#type) 是一个大于 0 的整数，据此可以推演出 `size_t` 这个 [type](#type) 的各种操作；但同时，由于在不同的平台上，`size_t` 的 `SIZE_MAX` 并不一样，所以在 `size_t` 上的操作结果也是和平台相关的，并不能仅仅由  [type](#type)  就决定其操作结果；和平台相关的部分需要由 [Binary Representation](#binary-representation) 来决定。

总之：

> 「抽象状态机」状态的转换由 3 者决定： [value](#value)、[type](#type)、[Binary Representation](#binary-representation)

其他一些 tips：

* 尽量使用 unsigned 类型，避免可能的 UB 行为
* 尽量使用 uint8_t, uint16_t, uint32_t……

#### Object Representation

* [Object Representation](#object-representation) 是和平台相关的概念，不是抽象的，具体由编译器来决定

总之，

> 「抽象状态机」状态的迁移只由 3 者决定：[value](#value)，[type](#type)，[Object Representation](#object-representation)

### Derived data types

4 种非基本类型：

* 2 种组合类型
  * Array
  * Structure
* 2 种非组合类型
  * Pointer
  * Union

#### Array

* Array 作为函数参数时，数组的长度信息会**丢失**，但可以使用 `static` 关键字进行说明。例如：

```c
// static 2 表示作为函数入参的数组的程度「大于等于」 2
void swap_double(double a[static 2]) {
  double tmp = a[0];
  a[0] = a[1];
  a[1] = tmp;
}
```

* `void func(double a[static 7]);` 中的 `static 7` 标明 func 的参数是一个至少包含 7 个元素的数组（或指针）
* 「字符串」是最后一个元素为 0 的 char 数组。下面分别给出 2 类例子，第一类是「字符串」，第二类不是「字符串」：

```c
// 下面 4 个是字符串（最后一个元素为 0）
char jay0[] = ”jay”;
char jay1[] = { ”jay” };
char jay2[] = { 'j', 'a', 'y', 0, };
char jay3[4] = { 'j', 'a', 'y', };

// 下面 2 个不是字符串
char jay4[3] = { 'j', 'a', 'y', };
char jay5[3] = ”jay”;
```

* 对字符串函数（例如：`strlen(s)`）使用非字符串是 UB

2 种数组：**FLA** 和 **VLA**（VLA 是到了 C99 才有的）

VLA 的一些特点：

* VLA 没有初始化
* VLA 不能在 function 之外定义（只能在函数内部定义使用）

VLA 的一个声明：

```c
 void func(size_t n, double a[n]);
```

#### Structure

* 在声明 `struct` 时，使用 `typedef`，起到简化的作用（不需要每次都要加上 struct 关键字）：

```c
typedef struct toto toto;
struct toto {
  // ...
};

// 不需要加上 struct
toto one;
```

* **opaque structure**：就是不在头文件里面定义 struct。只在头文件里面声明 struct，同时头文件里面的函数的声明只会使用到这个 struct 的指针。例如：

```c
typedef struct toto toto;

void toto_doit(toto*, unsigned);
```

* 但是，注意，不要 typedef 指向 struct 的指针

```c
// 不要这样做
typedef struct toto_s* toto;
void toto_doit(toto, unsigned);
```

### 线程

本节介绍 C11 标准里使用[线程](#线程)时会用到的几个基本库函数：

```c
#include <threads.h>
typedef int (*thrd_start_t)(void*);

/*
 * thrd_create 的 3 个入参说明如下：
 * thrd_t*：线程创建成功后的线程 id
 * thrd_start_t：线程执行的函数
 * void* 需要传入给线程的数据
 */
int thrd_create(thrd_t*, thrd_start_t, void*);

int thrd_join(thrd_t, int *);
```

一般的使用方法：

* 在 `main` 函数中，使用 `thrd_create` 函数对不同的任务创建相应的线程
* 然后在 `main` 函数中使用 `thrd_join` 对这些[线程](#线程)做 `join`，等待这些[线程](#线程)的结束
* 同时，某个[线程](#线程)执行其对应的 `thrd_start_t` 函数，直到这个函数 `return`，该[线程](#线程)的工作结束

一个简单的例子：

```c
/* 先准备好需要提供给线程的数据：
   Create an object that holds the game's data. */
life L = LIFE_INITIALIZER;
life_init(&L, n0, n1, M);

/* Creates four threads that all operate on that same object
   and collects their IDs in ”thrd” */
thrd_t thrd[4];
thrd_create(&thrd[0], update_thread, &L);
thrd_create(&thrd[1], draw_thread, &L);
thrd_create(&thrd[2], input_thread, &L);
thrd_create(&thrd[3], account_thread, &L);

/* Waits for the update thread to terminate */
thrd_join(thrd[0], 0);

/* Tells everybody that the game is over */
L.finished = true;
ungetc('q', stdin);

/* Waits for the other threads */
thrd_join(thrd[1], 0);
thrd_join(thrd[2], 0);
thrd_join(thrd[3], 0);

// 只有当这 4 个线程都结束时，main 函数才会结束
```

使用线程的要点：

* 如果某个[线程](#线程)对一个「非原子」变量进行了写操作，那么其他[线程](#线程)**同时**对这个变量的「读写操作」会导致[线程](#线程)执行的 UB（未定义行为）
* 使用 `_Atomic(T)` 语法定义「原子」变量。具体用法参考文档，这里只给简单的例子：

```c
 // 不能作用到数组上。Invalid: atomic cannot be applied to arrays.
_Atomic(double[45]) C;

// Valid: atomic can be applied to array base.
_Atomic(double) D[45];
```

### Tips

#### 类型

* C 语言中，有几种类型是不能直接做**算术操作**的。所谓的 **narrow types**：unsigned char / unsigned short /  char / signed char / signed short / bool
* 在算术表达式中，narrow types 会被做一次 promote，成为 signed int 类型（而不是 unsigned int）；所以干脆最好不要在算术表达式中使用 narrow types
* 尽量使用 unsigned 类型；同时不要在算术表达式中把 unsigned 类型和 signed 类型混用
* C 标准中的 stdint.h 头文件里面定义了固定 size 的整数。例如：uint32_t，int32_t 等。但到了具体平台上，并不是每种这些类型都存在

```c
#include <inttypes.h>
#include <stdio.h>

uint32_t u = 23;

// 需要使用 PRIu32 来 printf 类型为 uint32_t 的值
printf("%" PRIu32 "\n", u);
```

* 非 scalar 类型的初始化必须使用 `{}`。例如：

```c
double A[] = { 7.8, };
double B[3] = { 2 * A[0], 7, 33, };
double C[] = { [0] = 6, [3] = 1, };
```

* 上面例子中数组 C 的初始化是 **Designated Initializers**（「指定初始化」，C99 标准支持）。尽量要使用 **Designated Initializers** 可以获得更好的 robust

* 另外，当不知道如何对类型 T 做初始化时，推荐使用 **Default Initializers**，`{ 0 }`。**只要不是 VLA 类型**，都可以使用 **Default Initializers**。例如：

```c
double C[] = { 0 };
```

* 「只读字符串」建议采用 `char const*const` 的形式，例如：

```c
char const*const bird[3] = { 
  ”raven”,
  ”magpie”,
  ”jay”,
};
```

#### Function

* Use `EXIT_SUCCESS` and `EXIT_FAILURE` as return values for `main` 函数
  * 这样做为移植性考虑：`main` 函数在有的平台期望 return 一个整数，而有的平台不希望 return 值
* 满足 2 个特性的 function 叫 **pure function**：
  * The function has no effects other than returning a value.
  * The function return value only depends on its parameters.
* 当不能用 pure function 解决问题时，需要使用「指针」作为函数参数来解决问题
  * pure function 就是只用值来传递参数，函数内部对参数的修改不会影响函数返回后入参的值
* 如果 function 除了 return 之外，还有其他方式来改变「抽象状态机」的状态，那么就不是 pure function。例如：
  * 修改了「全局变量」
  * 使用了 static 变量
  * 做了 IO 操作

#### 指针

* pointer 用来 hold objects 的地址。objects 的 names 就是 variables
* `*` 号用来定义指针类型时，最好靠左：`int* p ＝ ＆n`
* `*` 号用来「解引用」时，最好靠右：`int i ＝ *p`
* 如果一个指针指向一个 array object，那么 array object 的长度不能通过对这个指针做 sizeof 操作得到
  * 指针不是数组
  * 把指针作为函数的参数时，如果语义上这个指针是一个数组，需要也指定这个数组的长度
* 对指针之间做减法，只能是当这些指针指向的是同一个 array object 时才行
  * object 上做 sizeof，返回的结果的类型是：size_t
  * ptrdiff_t：指针之间相减以后的值的类型是 ptrdiff_t
  * 使用 printf 打印一个指针时，用 p％，而且要把这个指针 cast 成 void\*
* C 里面可以把一个类型的 object 强制解释成另外一个不同的类型。但不要这样做，这样做有一个专门的术语：**trap representation**，这是一种 UB 行为。一个指针只能指向 3 种目标：
  * 有效的 object
  * 有效位置之后的一个位置
  * 0（空值）
* 总之，对一个 pointer 做 dereferenced 时，这个 pointer 指向的 object 必须有效，同时这个 pointer 指向的 object 的 type 必须是明确的（**designated type**）
* C 里面有空指针的概念，但不能使用宏 NULL 来表示空指针（有坑。目前 C 标准对宏 NULL 的规定比较松散，不严格，底层具体的实现可能和平台相关）。目前推荐的做法是把指针赋值为 0 来表示空指针
* 数组可以被退化成指针。而且一旦退化成指针后，数组的其他信息就都丢失了

```c
// A 原先是一个数组，取地址后就退化成指针 p 了，后续使用 p 就是作为一个指针来使用
int A[10] = {0};
int* p = &A[0];
```

* 把数组作为参数传给函数时，在数组参数之前增加一个参数来表示数组的长度：

```c
// len 参数就是用来表示「数组参数」的长度
// 当然这个 len 的值是否正确，只能是靠程序员来保证了
double double_copy(size_t len, double target[len], double const source[len]);
```

* 能不用 `&` 操作符（取一个 object 的地址），就不要用；引起潜在的问题（可以参考 Rust 的思想）
* `void *`：无类型的指针。任何对象的指针**无需强转就**可以转换为 `void*`，但也会损失掉这个对象的 type 信息
  * 尽管损失了 type 信息，但转换过程中 object 对应的 storage 实例是不会动的。在用强转回来以后，还能恢复原先的类型和数据（值能保持一致）
  * 一种常见的做法是：函数的入参是一个 `void*` 参数和一个 `size_t` 的参数，这样的话，代表该函数要对一段固定长度（`size_t` 为单位）的内存做操作，而不关心这段内存中的数据的类型（例如：memcpy 和 memset）
  * 还有一种常见情况：创建「线程」时「程序员」需要传递数据到线程中，pthread_create 函数有一个参数的类型就是 `void*`，其含义就是「程序员」可以传入任何类型的数据到线程中。当然，「程序员」自己知道这个 `void*` 是什么类型的，「程序员」会在「线程」中做一次 recast 把数据转换回到正确的类型。当然，这种 recast 的正确性完成由「程序员」保证。
  * `void*` 还是能不用就尽量不用
  
* `restrict` 关键字：用 restrict 修饰指针类型告诉「编译器」两个指针不指向同一数据（开发人员保证）。也就是该指针只会指向一个 object，不会 aliasing 其他对象
  * **Pointer aliasing**：是指两个或以上的指针指向同一数据

一个例子：

```c
// memcpy 的定义使用了 restrict
void* memcpy(void*restrict s1, void const*restrict s2, size_t n);

// 而 memmove 不能使用 restrict（可能发生 Pointer aliasing）
void* memmove(void* s1, const void* s2, size_t n);
```

* 一般来说，函数的参数的类型都是「指针」，而不能是「数组」。唯一的例外只是在对函数进行「声明」的时候，下面三种写法等价：

```c
int func(int *a);     /*写法1*/
int func(int a[]);    /*写法2*/
int func(int a[10]);  /*写法3：编译器会忽略 10 */
```

* C99 中给出了「可变结构体」的标准定义。例如：

```c
typedef struct {
    int     length;
    Point   point[]; // 这里不指定长度
} Polyline;

// 利用 malloc 对其进行初始化一个长度为 5 的结构体：
Polyline* polyline = malloc(sizeof(Polyline) + sizeof(Point) * 5);
polyline->length = 5;
```

#### inline

* C99 引入了 inline 关键字：
  * A function definition that is declared with inline can be used in several TUs without causing a multiple-symbol-definition error
  * All pointers to the same inline function will compare as equal, even if obtained in different TUs
  * An inline function that is not used in a specific TU will be completely absent from the binary of that TU and, in particular, will not contribute to its size
* inline 的一个关键点：
  * An inline function definition is visible in all TUs. 但同时，编译器也不会保证一定会 emit 这个 inline 函数的符号。
    * 如果要确保该 inline 函数的符号被 emit，可以在 .c 文件中加一行不带 inline 的函数 declare 来确保 emit 函数的符号
* 总之，inline 函数的最佳实践如下：
  * An inline definition goes in a header file
  * An additional declaration without **inline** goes in exactly one TU

例如一个头文件 toto.h 文件：

```c
// toto.h 文件
// Inline definition in a header file.
// Function argument names and local variables are visible
// to the preprocessor and must be handled with care.
inline
toto* toto_init(toto* toto_x) {
  if (toto_x) {
    *toto_x = (toto){ 0 };
  }
  return toto_x;
}
```

其对应的 toto.c 文件：

```c
#include ”toto.h”
// Instantiate in exactly one TU.
// The parameter name is omitted to avoid macro replacement.
toto* toto_init(toto*);

```

#### 其他

* 再澄清一下几个概念
  * semantic type：也就是语义上的类型。例如 int32_t（即 32 位的整数）
  * basic type：C 语言中的基本类型。例如 signed int（事实上 int32_t 是用 typedef 到 signed int）
  * binary representation：二进制表示。signed int 就是 b31, b30, ... b7, ... b0（32 个bits，4 个 bytes）
  * object representation：C 中所有的 object 都可以用 unsigned char 来表示。如果在一个小端系统中，上面的 binary representation 对应的 object representation 就是 unsigned char[4] 数组，且该数组的 [0] 是最高位，[3] 是最低位（小端系统）
* 总之，C 对 object 的内存模型做了以下规定：
  * `sizeof(char)` 为 1（包括 3 种 char：unsigned char，signed char 和 char）
  * 类型为 A 的 object 的  object representation 是一个数组：`unsigned char[sizeof(A)]`
    * 但不要搞混了，object representation 的 char 是 unsigned char，不是 char
    * char 只能用于「字符类型」，或者「字符串类型」

* cast：**不要用 cast**，坑巨多
* Effective Type：对 object 的 access 进行限制。关键点如下：
  * Objects must be accessed through their **effective type** or through a pointer to a character type
    * union 是个例外：Any member of an object that has an effective union type can be accessed at any time, provided the byte representation amounts to a valid value of the access type
  * The effective type of a variable or compound literal is the type of its declaration
* Files that are written in binary mode (fread, fwrite) are **not portable** between platforms

#### 错误码

* 一般还是不要用 errno 那一套方法（多线程环境不能用）。而是利用「枚举」来定义错误码，把错误码作为函数的返回值返回给「调用方」，并采用以 API 文档的形式定义错误码的具体含义。
  * 并且要用 API 的形式告诉「调用方」如何处理对应的错误；如果和「调用方」无关（或者说是「调用方」无法处理的错误）就不需要返回错误码给「调用方」了
