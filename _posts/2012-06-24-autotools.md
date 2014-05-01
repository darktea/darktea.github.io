---
layout: post
title: Autotools 使用入门
category: notes
---

# Autotools 使用入门

介绍 Autotools 的相关文章网上已经太多了，但这些文章要么是把一大堆工具列举出来，要么就是来一个 step by step 的例子。

但 Autotools 这套工具本身比较杂乱，罗列一堆工具，看了以后多半没有什么印象。

而 step by step 的例子做完一遍之后，也多半不知其所以然。

最好的入门材料还是 Autoconf 和 Automake 的官方文档：

* http://www.gnu.org/software/autoconf/
* http://www.gnu.org/software/automake/

可惜看初学者一上来就要面对几百页的英文文档，又有点下不了手。

本文的目的就是使得初学者可以快速入门，马上能在自己的项目中使用 Autotools 工具。

然后可以在以后的使用过程中，通过查询官方文档来解决实际遇到的问题，在实践中深入全面掌握其高阶用法。

 
这个入门分两部分，

第一部分介绍基础概念；

第二部分展现了一个完整的例子，可以参照这个例子把已有项目改成用 Autotools 工具来 build。

 
GNU Autotools 一般指的是3个 GNU 工具包：Autoconf，Automake 和 Libtool (本文先介绍前两个工具，Libtool留到今后介绍)

它们能解决什么问题，要先从 GNU 开源软件的 Build 系统说起。一般来说。GNU 软件的安装过程都是：

1. 解压源代码包
2. ./configure
3. make
4. make install

这个过程中， 需要有一个 configure 脚本，同时也需要一个 Makefile 文件。

而 Autoconf 和 Automake 就是一套自动生成 configure 脚本和 Makefile 文件的工具。


## 一. Autoconf 解决什么问题?

最早的时候，程序员完成源代码开发以后，发布代码包时，一般会附带相应的 Makefile 文件。然后就可以 make && make install 来编译工程。当时并不需要这个运行 configure 的步骤。

 

但是如果一个程序被广泛使用以后 (特别是成功的开源软件)，可能需要被安装到不同的平台上使用。这个时候，在不同的平台做 build 的时，一方面可能需要对 Makefile 文件进行调整 (最常见的例子就是：编译器的名字在不同的平台可能不同)。另外一方面，可能需要用一个替代函数来替换当前平台所不支持的函数 (例如：有的平台上不支持strdup这个调用)，需要在程序里面给每个平台写#define。

 

为了避免手工做这些调整，人们开始写 configure 脚本来自动做这些调整工作 (现在在 make 之前先运行 configure 是 GNU Code Style 标准所规定的)。

 

configure 脚本一般会先检查目前的环境，然后生成一个config.h 文件 (里面带了各种各样的#define) ，同时会生成一个 针对当前平台的 Makefile 文件，之后，make 命令就会使用到这个 Makefile文件。


另外，GNU的 build  系统还有一些"乱七八糟"的功能，用户在使用 configure 这个脚本的时候，可能会使用到这些功能 (最常见的就是用 --prefix 来指定安装路径，用configure --help来查看说明等等)。

 

但是，过了一段时间以后，人们发现靠人手工写这个 configure 脚本工作量太巨大了，而且以后维护这个 configure也比较麻烦：一旦发现项目在某个平台的移植性有问题，就需要更新这个 configure 脚本，比较繁琐。于是，人们就开发了 Autoconf 这个工具集来自动生成 configure 脚本。

 

总之，相对于手工维护 configure 脚本，用 Autoconf 工具来生成 configure 有以下好处：

1. 不需要手工写configure来支持 GNU Build System 规定的一些必须的功能。例如：在运行configure的时候对目录参数进行设置 (最常见的就是 --prefix=...)。或者可以设置 build 选项, 例如, 设置 CFLAGS 什么的
2. 用 Autoconf 自动生成的 configure 能完美的支持各种不同的平台上 (手工写几乎是不可能的)。当 GNU Coding Style 更新的时候, Autoconf 会统一做相应的更新, 不需要所有的项目都自己去手工调整 configure 脚本。


## 二, Automake 解决什么问题?

上面介绍了 Autoconf 的功能，接下来介绍 Automake。

有了 Autoconf 工具以后，人们能自动生成 configure 脚本，configure 脚本一般会做以下工作：

1. 检查用户的环境是否满足 GNU 程序的 Build 环境 (Autoconf 提供了很多macro来做各种各样的检查)
2. 替换模版文件，生成最后 Build 工程所需要的 Makefile，config.h 等文件 (其中 Makefile 的模版文件是 Makefile.in, 而 config.h 的模版文件是 config.h.in)

这里要特别说明的是 configure 是通过替换模版文件中的一些平台相关变量和 Build 选项来生成最后的 Makefile 文件 (和当前平台相关的 Makefile 文件)。

虽然 configure 文件是 Autoconf 自动生成的，但是模版文件 Makefile.in 还是要程序员自己写。下面是一个简单的 Makefile.in 的片段 (基本上就是普通 Makefile 文件的样子，除了 configure 脚本会替换掉那些用@包含的变量)：

```shell
# Package-specific substitution variables
package  = @PACKAGE_NAME@
version  = @PACKAGE_VERSION@
tarname  = @PACKAGE_TARNAME@
distdir  = $(tarname)-$(version)

# Prefix-specific substitution variables
prefix  = @prefix@
exec_prefix = @exec_prefix@
bindir  = @bindir@

# Tool-specific substitution variables
CFLAGS  ?= -g -O2

# VPATH-specific substitution variables
srcdir  = @srcdir@
VPATH  = @srcdir@

all: jupiter

hello: main.c
    $(CC) $(CFLAGS) $(CPPFLAGS) -I. -I$(srcdir) -I.. -o $@ $(srcdir)/main.c

...
```

而写这个 Makefile.in 估计每个程序员都会一开始自己写一个，之后做其他项目的时候，就在已有 Makefile.in 的基础上改一下直接用了。

这种做法有一些弊端，一方面都是手工操作，自动化不够；另外一方面在 copy 修改过程中可能会出现一些问题。

而 Automake 的目的就是可以帮助程序员自动生成模版文件 Makefile.in。而程序员只需要提供一个简单的 Makefile.am 文件来描述依赖关系就行了

例如, 只要在 Makefile.am 中写两行, :

```shell
bin_PROGRAMS = hello (目标)
hello_SOURCES = main.c (所依赖的源文件)
```

然后 Automake 就可以根据 Makefile.am 的描述自动生成 Makefile.in 模版文件

## 三, 例子

提供一个简单利用 Autotools 来 build 项目的例子。这个例子重点展示的是怎么把一个目录结构有点怪的已有项目改成用 Autotools 工具来 build 系统。


先说明一下，本文所使用的各个工具的版本 (因为 Autotools 工具集经常更新，老的版本可能和新版本有细微的差异)


* autoconf 2.67 版
* automake 1.11.1 版


然后介绍一下项目 (假设项目名称为 earth) 的目录结构：

```
           earth
      /               \
   src              test
   /                      \
fun.h fun.c          funtest
                                \
                             funtest1.c funtest2.c

```

项目根目录下有两个子目录 src 和 test。其中 src 目录放源代码，而 test 目录下面放测试代码；而测试代码又被分成很多组，每组是一个测试目录。

例如, test 目录下的 funtest 目录是一个测试组 (这个例子中只有一组测试)，用来对 fun.h/fun.c 进行测试，其中包含了两个测试用例，两个测试程序都可以链接出可执行文件 (funtest1.c 包含了一个 main 函数，funtest2.c 也包含了一个 main 函数)。

 

使用 Autotools 就是要利用 Autoconf 来生成 configure 脚本，同时利用 Automake 来生成 Makefile.in 和 config.h (因为这个例子程序比较简单, 没有用到什么特殊的调用, config.h 中也没有什么特殊的定义)

所以首先要提供这两个工具所需的输入文件：

1. configure.ac：Autoconf 所需 (就是一组宏定义，Autoconf 使用这些宏进行平台相关的检查, 并生成 configure脚本)
2. Makefile.am： Automake 所需 (特别注意，Automake 不单独使用，会和 Autoconf 配合使用。因此 configure.ac 里面也有 Automake 相关的宏定义)


### 1. configure.ac


一般来说，尽量直接使用 Autoconf 工具提供的 macro 来做各种各样的检查，最好不要自己写macro来做这些检查。因为Autoconf 提供的这些 macro 已经经过千锤百炼，适用不同的平台不同的系统，自己写多半不能做到像它这样全面。

earth 项目的 configure.ac：

```shell
AC_INIT([earth], [1.0], [bug-report@earth.com])
AM_INIT_AUTOMAKE([foreign -Werror])

AC_PROG_CC

AC_CONFIG_SRCDIR([test/funtest/funtest1.c])
AC_CONFIG_HEADERS([config.h])
AC_CONFIG_FILES([Makefile test/Makefile test/funtest/Makefile])

AC_OUTPUT
```

这里只特别说明一下 AC_CONFIG_FILES 这个宏的作用 (其他宏是必须的, 但是含义比较简单, 先直接照葫芦画瓢就行了, 具体细节可以查询 Autoconf 的手册)


AC_CONFIG_FILES 指明了需要根据模版生成的 Makefile 文件. 在这个例子中, 需要生成3个Makefile文件, 每个 Mafile 文件都需要一个模版:

* 根目录下需要有模版文件 Makefile.in
* test 目录下需要有模版文件 Makefile.in
* test/funtest 目录下需要有模版文件 Makefile.in

而这3个 Makefile.in 需要用 Automake 通过 各自的 Makefile.am 来生成。

特别注意，这个例子比较特殊，不需要在 src 目录下面进行 build， 因为这个例子只需要在测试用例目录下面生成测试程序的可执行文件即可。


### 2. Makefile.am

在介绍如何编写 Makefile.am之前，先说明一下 earth 项目为什么需要3个 Makefile 文件。


一般来说，都是在项目根目录下面运行 make && make install 的。而具体到earth这个项目，希望运行 make 以后，会在 test/funtest 下生成测试程序，因此需要make递归运行：
先在根目录运行make，再在 test 目录下运行 make，最后才在test/funtest 下运行 make，生成两个测试程序的可执行文件。


可以通过配置 Makefile.am 来控制这个递归过程。其中：


根目录下的 Makefile.am (SUBDIRS 指明需要在哪些子目录递归生成模版文件 Makefile.in) 只需要一行：

```
SUBDIRS=test
```
 

test目录下的 Makefile.am 只需要一行，指明在子目录 funtest 下执行 make：

```
SUBDIRS =  funtest
```

 

test/funtest 目录下的Makefile.am (test/funtest 目录下不需要再往下递归，所以不用设置SUBDIRS)，

由于需要描述出依赖关系，这个 Makefile.am 会比较复杂一点，有5行：

```
bin_PROGRAMS = funtest1 funtest2

funtest1_SOURCES = funtest1.c ../../src/fun.h ../../src/fun.c
funtest1_CPPFLAGS = -I$(top_srcdir)/src

funtest2_SOURCES = funtest2.c ../../src/fun.h ../../src/fun.c
funtest2_CPPFLAGS = -I$(top_srcdir)/src
```
 

其中：

* bin_PROGRAMS 指明要生成那些可执行文件
* funtest1_SOURCES 指明 funtest1 所依赖的源文件 (特别注意, 利用相对路径指明了所依赖的src目录下的源文件)
* funtest1_CPPFLAGS = -I$(top_srcdir)/src 指明了编译时需要指定的头文件路径

 

### 3. 利用 Autotools 生成configure脚本和Makefile.in模版

在configure.ac 和 Makefile.am 做好以后。在项目根目录运行：

```
autoreconf --install
```

就可以生成 configure 脚本和所有的模版文件 Makefile.in

然后再运行

```
./configure && make 
```

就可以得到测试程序的可执行文件。


这里特别说明的是，Autoconf 和 Automake 是两个工具包。其中包含了多个工具软件，而这些工具软件要互相配合才能工作。

而这些工具软件间的关系比较复杂，单独运行各个工具软件也可以得到最后的 configure 脚本和模版文件 Makefile.in。

不过使用 autoreconf 这个命令可以自动安排各个工具软件的执行顺，所以一般推荐直接运行 autoreconf，而不是单独运行各个单独的工具软件。

 

# 参考文档：

* [autoconf 官方手册](http://www.gnu.org/software/autoconf)
* [automake 官方手册](http://www.gnu.org/software/automake)
* [比较流行的一个入门文档](http://www.lrde.epita.fr/~adl/autotools.html) (这个人做过一段时间的维护工作)
* 最新的一本关于Autotools的书: Autotools: A Practioner's Guide to GNU Autoconf, Automake, and Libtool

