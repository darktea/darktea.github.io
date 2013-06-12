---
layout: post
title: Linux Page Cache and Buffer Cache
category: notes
---

# 〇, 目的

Linux Kernel 为存取速度慢的 Block 设备提供了两种比较通用的 Cache 机制:

* Page Cache: 为以页 (Page) 为单位的操作提供 Cache
* Buffer Cache: 为以块 (Block) 为单位的操作提供 Cache

本文的目的就是介绍这两种 Cache 的相关知识.

Cache 机制和 内存管理, 文件管理, 设备管理都紧密相关. 所以会先介绍一下这些相关知识.

# 一, 内存管理

## 1. 进程地址空间

从每个用户空态进程的角度来看, 它能使用的内存是一个独享的平坦线性的 32位 或 64位的虚拟地址空间 (一个大数组), 即进程地址空间 (Process Address Space).

而这个大的内存空间 (32位的话, 有 4G) 又可以被分成很多区域 (Memory Areas), 每个 Memory Area 是一个连续的地址空间.

对进程地址空间 (Process Address Space) 中的不同的 Memory Areas,  用户进程有不同的操作权限 (可读, 可写, 可执行). 例如, 一块区域是代码段, 那么, 用户进程对这块区域只有可读和可执行的权限.

Kernel 用一个数据结构来描述进程地址空间: struct mm_struct, 其中包含了进程地址空间的所有信息 (每个进程对应一个 task 结构, 其中有一个 mm field 指向这个进程对应的 mm_struct).

下面特别介绍 mm_struct 的 3 个关键 fields:

* mmap/mm_rb: 这两个 fields 描述同一个东西: 本地址空间包含的所有的 Memory Areas. mmap 是一个链表, 当遍历所有 Memory Areas 时使用. 而 mm_rb 是一个红黑树, 用来快速查找某个 Memory Areas;
* pgd: 指向 Page Table 的指针 (Page Table 后面会详细讲)

----

在 Linux Kernel 中, Memory Areas 被简称为 VMAs. VMA 的种类:

* 和文件相关的 VMA: 代码/库, 数据文件, 共享内存, 设备; 这些 VMA 的内容都是来至于文件的.
* 匿名 VMA: Stack, Heap, CoW pages; 这些 VMA 的内容都是用户程序管理的.

Linux Kernel 用  struct vm_area_struct 来描述 Memory Area.  每个 Memory Area 对应一个 vm_area_struct,  下面介绍 vm_area_struct 的关键 fields:

* vm_mm: 指向 VMA 对应的 mm_struct;
* vm_start: vma 的起始位置 (低位)
* vm_end: vma 的结束位置 (高位). vm_end - vm_start 就是这个 VMA 的 size.
* 一组函数指针; 这些函数实现了在这个 VMA 上的各种操作 (page fault, open, close ...)

Figure 4-6 展示了管理一个进程地址空间 (Process Address Space) 怎么管理和它相关的 VMAs.

## 2. Page

Linux 按 Page 来管理内存. 所以进程地址空间 (Process Address Space), 虽然被分成很多内存区域 (Memory Areas), 但每个 Memory Area 由有多个 Pages 组成.

每个 Page 的大小是固定的 (32位的体系结构中, 一个 Page 的大小是 4K Bytes).

物理内存也是按 Page 来管理的: 物理内存被分成很多页, 每页大小也是 4K Bytes, 一般来说, 物理 Page 有一个名字: Page Frame.

对每个物理 Page Frame, Kernel 用 struct page 来描述. 下面介绍一个关键 fields:

* mapping: 这个 Page Frame 被 Page Cache 使用时 (关于 Page Cache 的详细情况后面会讲), 这个 field 指向和这个 Page Frame 联系的 Page Cache 对象.

## 3.  内存空间映射

用户进程地址空间只是一个虚拟内存, 当进程在运行时, 操作系统必须把进程正在使用的虚拟内存和物理内存对应起来.

Linux 按页管理内存, 当用户进程要存取某个 Page, 但这个 Page 还没有存在在物理内存中, Linux 触发一次 Page Fault, 把这个 Page 和 物理 Page Frame 对应起来.

Linux Kernel 利用 Page Table 来做虚拟内存到物理内存的映射 (每个进程都有自己的 Page Table). 

一般会用3级页表: PGD->PMD->PTE; 操作系统只要先做好设置, 一般来说, 硬件 (CPU) 会自动做虚拟内存到物理内存的映射 (MMU);

而且会利用 TLB 来加速映射 (映射缓存到 TLB).

一旦切换进程, 也要切换 Page Table, 同时可能也要刷新 TLB. 也就是说, 各个进程的页表是隔离的, 不会互相影响.

Figure 15.1 Virtual-to-physical address lookup 展示了利用页表把虚拟页映射到一个 Page Frame.

## 4. 反向 Mapping

上一节讲了了虚拟内存 Page 到 物理内存 Page Frame 的映射. 

而很多时候, 某个 Page Frame 可能会被多个 Page 映射 (例如一个共享库被多个进程共享); 这种物理页被多个虚拟页共享的机制可以节约内存.

但是同时也引入了新的问题: 

* 如果这个物理页长时间没有被使用, 当系统内存紧张时, 可能需要把这个物理页换出到外存. 此时也需要通知那些映射到这个物理页的进程修改页表.
* 那么怎么才能知道某个 Page Frame 被哪些 Page 映射了呢?
* 这就需要引入一个 反向映射 (反向 Mapping) 机制. Kernel 会利用一个数据结构来记录这个物理页被哪些 VMA 使用.

反向 Mapping 这里只是先提一下. 后面介绍 Page Cache 时会介绍更多的细节.

# 二, 虚拟文件系统 (VFS)

在同一个目录结构中, 可以挂载着若干种不同的文件系统. VFS 隐藏了它们的实现细节, 为使用者提供统一的接口.

## 1. files_struct 结构

从进程的角度来看文件会使用到 files_struct 结构.

VFS 的使用者是进程. 描述进程的 task_struct 结构中 files 指针指向了一个 files_struct 结构, 而 files_struct 描述了进程已打开的文件集合.
这个集合中的每个已经打开的文件用 struct file 来描述.

files_struct 结构最后会对应一个 inode (下面介绍 inode)

## 2. inode

文件的 metadata 一般包含以下几种信息:
* 文件名字
* unique number (文件在文件系统的唯一标识符)
* 文件在磁盘上的位置
* 文件大小
* 权限控制 (可读, 可写, 可执行)
* 时间, 日期

Linux 中用 inode 来存储文件的 metadata.

多个 files_struct 结构 可能会对应同一个 inode.

# 三, 设备管理



# 四, 什么是 Page Cache?

Linux Kernel 利用 Page Cache 来尽量少的缩减磁盘 IO.

磁盘 cache 是现代操作系统的一个重要组成部分, 有两个原因:

1. 存取内存的速度比存取磁盘的速度快好几个数量级
2. 时间局部原理

Page Cache 的任务就是利用一些物理内存来加速对块设备的以页为单位的操作.

Page Cache 由内存中的一些物理 Pages 组成. 其中存储了磁盘上物理 Blocks 的内容.

Page Cache 的总大小是动态的, Linux 的理念就是尽量的利用空闲的内存来缩减磁盘的 IO. 同时, 当系统内存紧张时, 又会缩减 Page Cache 的大小, 腾出更多的内存.

# 五, 读操作相关的 Page Cache

Kernel 做读操作时 (例如, 某个 process 调用 read 调用) 先会检查 Page Cache 是否已经存放了要读的数据. 如果已经存在, 就是 **cache hit**; 

否则, 就是 **cache miss**, 出现 cache miss 时, kernel 必须利用 Block IO 操作从磁盘上读取数据; 同时这些数据也会放到 Page Cache 中, 之后需要读取这些数据的时候, 就 cache hit 了.

# 六, 写操作相关的 Page Cache

写操作相关的 Cache 策略有3种:

1. no-write: 写操作时不做 cache, 会把数据直接写到磁盘, 同时, Page Cache 中, 之前被 cache 了的数据需要被清理掉. 现代操作系统中, 很少使用这种策略;
2. write-through: 写操作时, 数据写到磁盘上,同时会更新 page cache 中的数据. 这种策略实现比较简单, 但是性能差;
3. write-back: 写操作只把数据更新到 Page Cache 中, 但磁盘中的数据不会立即被更新. 这样, 就会造成 Page Cache 中某些页的内容和磁盘数据不一致. 这些 Pages 被标记为"脏页".  Kernel 会周期性的把脏页的数据写回到磁盘, 使两者的内容达到一致;

write-back 策略比 write-through 策略的性能要好.

# 七, Page Cache 的管理

Kernel 需要快速的确认某个 Page 是否 存在在 Page Cache. 为了提高在 Page Cache 查找 Page 的速度, 利用了 Radix Tree.

Linux Kernel 利用 Radix Tree 来管理 Page Cache 中 Pages.

# Buffer Cache

和在物理内存中的 page 不同, Block 的大小更小, 而且对于不同的块设备, 可能每块的大小也会不同.

另外一个问题, 就是前面也提到过, 现在新版本的 Kernel 里面, 单独操作一个块比较少了, 大多数时候都是用 Page Cache. 

只要在少数场景会用到 Buffer Cache (一个典型的例子就是读文件系统的元数据会用到 Buffer Cache, 而读文件中的数据用的是 Page Cache)

Buffer cache 的数据结构包含两种结构:

*  buffer head.  buffer head  指向一块物理内存, 其中包含了 buffer 状态的相关信息 (例如: block number, block size 等等)
* 被 cache 的数据
 * 被存放在 page cache 中. page cache 中每个 page 被平均划分为多块, 每块都由一个 buffer head 指向 (一般来说, page 有 4k 大, 而每个 block 有 512 大).
 * 如果是直接按块为单位来操作块设备, 就不会用 page cache, 而是用专门的 buffer cache 来存放被 cache 的数据.

这里特别注意的就是, 实际上 cache 的数据 和 buffer head 所用的数据不是 同一块数据.

这里可以看出, 虽然, buffer cache 是由一些 buffer head 来管理的, 但实际上其 cache 的数据是按块组合起来以后存放到 page cache 中的.

# 五, Cache 回收

Linux 的 Cache 回收策略:

1. 首先会选择回收 "干净" 的页
2. 如果只回收 "干净" 的页还不够, 那就需要对一些 "脏页" 做 write-back. 而用什么策略来选择哪些 "脏页" 做 write-back? Linux 使用  two-list 策略 (LRU策略的改进) 来挑选 "脏页" 做 write-back.

# 六, Linux 中 Page Cache 的实现

Page cache 中的一页可能包含多个不连续的物理磁盘的 blocks (例如, 一个 page 是 4K, 而一个 Block 是 512, 一个  page 可能有 8 个 blocks, 而这 8 个 blocks 并不一定是 8 个相邻的 blocks).

事实上, 如果只是实现针对物理文件的 cache 的话, 只需要对 inode 的 structures 进行扩展就行了. 但 Linux 的目的是实现一个更通用的 cache, 所以 Linux 定义了一个新的 structure 来实现 page cache.

对文件的 cache, 只需要在 inode 的 structure 中用一个 field 指向这个新的 structure 就行了 (每个文件对应一个 page cache).

下面叫介绍这个数据结构: address_space (这个名字不太准确, 叫 "page_cache_entity" 什么的可能更贴切):

struct address_space { 
    struct inode            *host;              /* owning inode */ 
    ...
    struct prio_tree_root   i_mmap;             /* list of all mappings */ 
    ...
    unsigned long           nrpages;            /* total number of pages */ 
    ...
};

* host: 如果这个 cache 和一个 inode 关联, host 指向这个 inode 节点; 否则 host 为 NULL
* i_mmap: kerner 利用这个结构来寻找哪些虚拟页和这个 cached 文件联系
* nrpages: 一共有多少个页

由于大多数时候, Kernel 中的操作大多是以 Page 为单位, 所以 Page Cache 会比较常用.

但按块 (Block) 操作也是有的, 比如读一些元数据信息的时候 (Metadata Blockwise) 按块更容易处理. 

随着 Kernel 的不断发展, 趋向于更多的使用 Page Cache. 但总的来说, Buffer Cache 的保留也不仅仅只是为了兼容老版本, 有的地方还是用 Buffer Cache 更合适 (后面会详细讲).