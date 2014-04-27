---
layout: post
title: Linux 内存管理
category: notes
---

# 内存管理

本文简单介绍 Linux Kernel 怎么管理内存. 包含内核物理内存管理和用户空间内存管理两大部分.

## 1. Page Frame

Linux Kernel 以物理 Page (即 Page Frame) 为基本单位来进行物理内存管理.
而对不同的 Architecture, Page Frame 的大小也不一样, 不过一般的 32 位的 Architecture 的 Page Frame 大小为 4K Bytes.
Kernel 用一个数据结构 struct page 来表示系统中的一个 Page Frame (下面只列出了 4 个重要的 fields):

{% highlight c %}
struct page { 
    unsigned long            flags;
    atomic_t                 _count; 
    struct address_space     *mapping; 
    struct list_head         lru; 
};
{% endhighlight %}

4个重点字段:

* flags: 标识物理页面的状态 (例如, 是否是脏页)
* _count: 物理页面的引用计数, 一旦为 0, 说明这个页面没有被使用, 是 free 的
* lru: 链表头. 多个 Page Frames 可以用链表组织起来, 例如, active 链表 和 inactive 链表 (以后介绍页面回收的时候会详细介绍)
* mapping 分两种情况: 
  
  如果物理页面被 Page Cache 使用, 并映射到一个文件, 那么 mapping 指向映射的文件的 inode 的 address_space 对象 (以后介绍 Page Cache 时会有更详细的介绍)

  如果物理页面是一个匿名页, 那么 mapping 指向 anon_vma 对象, 而不是 address_space 对象 (关于匿名页, 以后会有详细介绍)

## 2. 进程地址空间

从每个用户空态进程的角度来看, 它能使用的内存是一个独享的平坦线性的 32位 或 64位的虚拟地址空间 (一个大数组), 即进程地址空间 (Process Address Space).

而这个大的内存空间 (32位的话, 有 4G) 又可以被分成很多内存区域 (Memory Areas), 每个 Memory Area 是一个连续的地址空间, 而且各个 Memory Area 不会重叠.

对进程地址空间 (Process Address Space) 中的不同的 Memory Areas,  用户进程有不同的操作权限 (可读, 可写, 可执行). 例如, 一块区域是代码段, 那么, 用户进程对这块区域只有可读和可执行的权限.

Kernel 用一个数据结构来描述进程地址空间: struct mm_struct, 其中包含了进程地址空间的所有信息 (每个进程对应一个 task 结构, 其中有一个 mm field 指向这个进程对应的 mm_struct).

下面特别介绍 mm_struct 的 4 个关键 fields:

* mmap: 本地址空间包含的所有的 Memory Areas 的集合. mmap 是一个链表, 当遍历所有 Memory Areas 时使用;
* mm_rb: 和 mmap 字段一样, mm_rb 也是本地址空间所有 Memory Areas 的集合,不过不是链表, 而是一个 红黑树, 用来快速查找某个 Memory Area;
* pgd: 指向本进程 Page Table 的指针 (Page Table 后面会详细讲)
* mm_users: 有几个进程正在使用这个地址空间 (例如: 如果两个线程共享这个地址空间, 那么 mm_users 为 2)

## 3. VMA

在 Linux Kernel 中, 组成进程地址空间的 Memory Areas 被简称为 VMAs. VMA 的种类:

* 和文件相关的 VMA: 代码/库, 数据文件, 共享内存, 设备; 这些 VMA 的内容都是来至于文件的.
* 匿名 VMA: Stack, Heap, CoW pages; 这些 VMA 的内容都是用户程序管理的.

Linux Kernel 用  struct vm_area_struct 来描述 Memory Area.  每个 Memory Area 对应一个 vm_area_struct,  下面介绍 vm_area_struct 的关键 fields:

* vm_mm: 指向 VMA 对应的 mm_struct;
* vm_start: vma 的起始位置 (低位)
* vm_end: vma 的结束位置 (高位). vm_end - vm_start 就是这个 VMA 的 size.
* 一组函数指针; 这些函数实现了在这个 VMA 上的各种操作 (page fault, open, close ...)

下图展示了管理一个进程地址空间 (Process Address Space) 怎么管理和它相关的 VMAs.

![img](https://lh6.googleusercontent.com/-LxNqiy-7JyY/Ub29VuG1ElI/AAAAAAAAADw/mR9sIlibH9M/w761-h271-no/vma.png)

## 4. Page Table

用户进程地址空间只是一个虚拟地址空间, 当进程在运行时, 操作系统必须把进程正在使用的虚拟地址和物理地址对应起来.

Linux 按页管理内存, 当用户进程要存取某个 Page, 但这个 Page 还没有存在在物理内存中, Linux 触发一次 Page Fault, 把这个 Page 和 物理 Page Frame 对应起来.

Linux Kernel 利用 Page Table 来做虚拟地址到物理内存的地址映射 (每个进程都有自己的 Page Table). 

Linux Kernel 使用 4 级页表: PGD->PUD -> PMD->PTE; 操作系统只要先做好设置, 一般来说, 硬件 (CPU) 会自动做虚拟地址到物理地址的映射 (MMU);

而且会利用 TLB 来加速映射 (映射缓存到 TLB).

一旦切换进程, 也要切换 Page Table, 同时可能也要刷新 TLB. 也就是说, 各个进程的页表是隔离的, 不会互相影响.

下图展示了利用页表把虚拟页映射到一个 Page Frame.

![img](https://lh3.googleusercontent.com/--RtO9tp7hjk/Ub29UOAWGcI/AAAAAAAAADo/fo9LwK3s-YE/w896-h308-no/pdg.png)

## 5. 反向 Mapping

上一节讲了了虚拟内存 Page 到 物理内存 Page Frame 的映射. 

而很多时候, 某个 Page Frame 可能会被多个 Page 映射 (例如一个共享库被多个进程共享); 这种物理页被多个虚拟页共享的机制可以节省内存.

但是同时也引入了新的问题: 

* 如果这个物理页长时间没有被使用, 当系统内存紧张时, 可能会把这个物理页换出到外存. 此时也需要通知那些映射到这个物理页的进程修改页表.
* 那么怎么才能知道某个 Page Frame 被哪些 Pages 映射了呢?
* 这就需要引入一个 反向映射 (反向 Mapping) 机制. Kernel 会利用一个数据结构来记录这个物理页被哪些 VMAs 使用.

反向 Mapping 这里只是先提一下. 以后介绍 Page Cache 时会介绍更多的细节.
