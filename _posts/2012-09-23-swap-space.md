---
layout: post
title: 系统性能监控之 Swap Space
category: notes
---


对线上服务器进行性能监控时, 需要关注各种性能指标, 从各个方面来对系统性能进行监控. 例如, 系统负载, cpu 占用, 内存占用, 网络带宽等. 其中 Swap Space 的使用状况也是值得关注的一项, 本文对在 Linux 环境中监控 Swap Space 的相关内容进行了总结.

### 一, 首先简单的介绍一下什么是 Swap Space

用户进程内存空间中数据有两种:

1. 从文件系统中读进来的数据 (主要有文件内容高速缓存, 程序代码和共享库)
2. 程序使用的堆栈空间

而Linux 是一个分页请求系统, 用户进程使用的所有内存需要映射到物理内存:

1. 应用程序的地址空间是虚拟空间, 是按被分成固定大小的页 (page) 来管理的
2. 物理内存也是按固定大小的页框 (page frames) 来组织
3. 虚拟页面 (page) 需要由操作系统映射到物理内存中的页框 (page frames)

为了更好的利用物理内存, Linux 会对在物理内存中的页面进行回收

* 如果存放的是程序代码或共享库, 可以直接回收 (以后需要的时候可以从文件里面直接读入)
* 如果缓存的是文件内容, 如果是脏页,  先写回磁盘后再回收; 如果不是脏页可以直接回收
* 而对于程序使用的堆栈空间, 由于没有对应的文件 (叫做 "匿名页", anonymous pages), 只能是备份到磁盘上一块专门的分区, 这个专门的分区就是 Swap Space.
* 一旦回收算法既不能从文件缓存回收内存, 又不能从正在使用的匿名页中回收内存, 而系统需要满足更多的内存请求, 这个时候只能用最后一招了: OOM kill

从 swap space 的原理得出以下结论:

* Swap Space 不会拖慢系统. 事实上, 不分配 swap space 并不代表就没有换进换出发生 (非匿名页还是可能会被换进换出).有了 Swap Space, 只是说在试图回收内存时, Linux 有了更多的选择
* Swap Space 只是匿名页专用的. 在任何情况下, 程序代码, 共享库, 文件系统cache不会使用 Swap Space
* 由上面的 1, 2 两点可以知道: "尽量给 Swap Space 分配很小的空间" 唯一的好处就是不浪费磁盘空间 (也就是说最小化 Swap Space 大小并不能提升系统性能)
* 系统监控时, 要特别注意是否发生了 OOM 的情况, 一旦 OOM 发生, Linux 会自己挑选一个消耗内存较大, 又不 "重要" 的程序 kill 掉来回收内存 (但很多时候被 kill 掉的正好是某个应用程序)

### 二, 对 Swap Space 进行监控

先利用 free 命令来监控 swap space 的使用率. 例如:

![img](https://lh5.googleusercontent.com/-GhzyxKVX8Js/U2Ly3_RyQlI/AAAAAAAAAEQ/tvsJtJ1P4EE/w615-h64-no/1.jpg)

最后一行显示 swap space 的使用状况:

* total 显示 swap space 的总容量 (2G)
* used 显示已用空间大小 (43M 左右)
* free 显示空闲空间大小 (1.95G 左右)

对系统进行监控时, 可以监控 used占用了total的比例.

但是特别需要注意的是, 这个比值较高时, 并不等于系统内存紧张了.  只有当在 Swap Space 上有大量的换入换出操作时才说明出现了内存紧张.

例如, 某个程序启动时使用了大量的内存, 造成系统的 Swap Space 消耗过大. 但启动完成之后, 进入平稳运行期时, 并不需要很多内存 (从这个例子也可以看出来, Swap Space 分配过小可能造成有时候本来可以启动起来的程序启动不起来).

可以用 vmstat  或 sar -W 命令来监控换入换出操作的状况

例如, 使用 sar -W:

![img](https://lh6.googleusercontent.com/-Aw3NZL7AFR0/U2LzAeNK0iI/AAAAAAAAAEw/TybuNWPW_uQ/w262-h79-no/2.jpg)

 或者使用 vmstat:

![img](https://lh4.googleusercontent.com/-jIsbfP4oJdE/U2LzIRFB6cI/AAAAAAAAAFc/ptkqeeSR_j8/w656-h96-no/3.jpg)


在这个例子里面, 可以看出, 虽然 Swap Space 被使用了一点, 但是没有在 Swap Space 上的换入换出, 所以系统性能还是不错的.

总之, 系统性能可能出现以下几种情况:

* 极好:   Swap Space 没有被使用. 物理内存的使用也很少.
* 好:      Swap Space 用了一点, 但是在 swap space 上没有换进换出的操作
* 还行:    Swap Space 用了一点. 有少量的在 Swap Space 上的 换进换出的操作. 不过系统的吞吐量还是不错 (CPU在 User 上的消耗还比较高, 也就是说明此时在 Swap Space 上的换入换出还没有造成系统瓶颈)
* 有问题: 在 Swap Space 上有大量的换入换出操作, 同时 CPU 在 Sys 和 Wait 上的消耗很高 (这种情况就是说明内存不够用了, 需要查看是否有程序发生了内存泄漏)
* 有问题: 系统运行良好, 但是启动新程序会由于内存不足而失败 (这种情况是 Swap Space 分配得太小了)

### 三, 和 Swap Space 相关的系统参数的调节

vm.swappiness (缺省值是60). 设置成0, Linux 会尽可能的避免把内存交换出去 (可以让系统有好的响应时间). 相反, 如果设置成100, Linux 会尽量的使用 swap space.

vm.overcommit_memory (缺省值是0):

* 0: 允许 overcommit (当收到申请内存的请求时, Linux 用一套heuristic方法来决定是否要 overcommit), 但是就像前面提到的, overcommit 之后, 当真正需要用到这些内存时, 如果回收算法又不能回收到任何物理内存, 只好OOM kill 掉一个进程
* 1: 总是允许 overcommit (Redis 就建议这样设置)
* 2: 不允许进行 overcommit, 一旦申请的空间超出总空间 (总空间 = 物理空间 + swap space size x overcommit_ratio, overcommit_ratio 缺省值是 50%), 就报错

###  参考

* http://www.linuxjournal.com/article/10678
* http://hi.baidu.com/_kouu/item/4c73532902a05299b73263d0