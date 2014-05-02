---
layout: post
title: Redis PSYNC
category: notes
---

本文主要介绍 Redis 2.8 新引入的增量同步 PSYNC 机制.

### 一, 动机

Redis 的 2.8 版本已经进入 beta 阶段. 这个版本最重要的特性就是新增了增量同步的功能. 

之前 Redis 在生产环境中最让人质疑的就是它的全量同步, 也就是每次从节点连上主节点的时候, 都会把主节点的所有数据同步过来, 哪怕是只是一次网络瞬断.

而 Redis 做一次全量同步的代价是巨大的. 接下来先简单的回顾一下 Redis 原来的全量同步 SYNC 实现, 然后再会介绍增量同步的实习原理.

### 二, 全量同步 (SYNC)

目前 Redis 主从同步的简要流程:

1. 每次 slave 连到 master 后, master bgsave 到 rdb 文件
2. master 发送 rdb 文件给客户端
3. slave 收到 rdb 文件后复制
4. master 开始向 slave 发送写命令 (master 作为 slave 一个特殊的 client)

整个同步过程是一个状态机:

* slave 有6个状态: REDIS_REPL_NONE, REDIS_REPL_CONNECT, REDIS_REPL_CONNECTING, REDIS_REPL_RECEIVE_PONG, REDIS_REPL_TRANSFER, REDIS_REPL_CONNECTED
* slave 接收到 slave 命令, 或者启动的时候从配置文件里面读到 slave 配置. 做初始化工作 (例如: 断开所有的slaves). 初始化结束后, 状态为: REDIS_REPL_CONNECT
* Redis 周期的调用 replicationCron 函数
* REDIS_REPL_CONNECT 状态时, replicationCron 中会创建一个socket fd 连接到 master, 并在上面创建读写事件. slave 状态为  REDIS_REPL_CONNECTING. 
* 消息循环中, socket fd 会阻塞发送一个 PING 消息给master (发送成功后, delete fd 的写事件) slave 状态为 REDIS_REPL_RECEIVE_PONG
* 消息循环中, 如果 slave 状态为 REDIS_REPL_RECEIVE_PONG. 开始阻塞读取 master 返回的响应. 并删除读事件. 并认证. 认证成功了阻塞发送 SYNC 命令请求同步. 
* 发送了 SYNC 命令后, slave 建临时文件准备接收 rdb 文件. 创建读文件时间, 状态为: REDIS_REPL_TRANSFER. 开始在消息循环中获取 rdb 内容.
* 完全获取到 rdb 文件后, 把 master 作为 一个特殊的 client, 开始接收写命令. slave 状态为: REDIS_REPL_CONNECTED

* master 有4个状态: REDIS_REPL_WAIT_BGSAVE_START, REDIS_REPL_WAIT_BGSAVE_END, REDIS_REPL_SEND_BULK, REDIS_REPL_ONLINE
* master 接收到 SYNC 命令后, 线性检查是否正好有其他的 slave 正在做 bgsave. 如果有, 就直接开始传输 rdb 文件. 没有的话, 之后开始做 bgsave 来创建 rdb 文件 (创建过程中, master 的状态为: REDIS_REPL_WAIT_BGSAVE_START).  子进程中创建rdb文件
* rdb 创建好了以后, master 的状态为 REDIS_REPL_WAIT_BGSAVE_END. 开始发送 rdb 文件的内容给 slave.
* serverCron 里面检测到有bgsave子进程结束, 顺序检查 slaves 列表. 如果是 END 状态的 slave, 创建写事件. 把 rdb 文件的内容在消息循环中发送给 slave. master 为 REDIS_REPL_SEND_BULK
* rdb 文件内容发送完成以后, 删除原先的写事件, 新建发送写命令到 slave 的写事件. 状态为: REDIS_REPL_ONLINE

master 的同步过程中的关键点:

1. 在发送 rdb 文件的过程中, master 新获取到的写命令被缓冲在 client 对象的发送缓冲区中. 一旦进入 REDIS_REPL_ONLINE, 会把缓冲区的写命令发送给 slave 进行写操作
2. 多个阶段写事件的创建和删除
3. 只要是 REDIS_REPL_WAIT_BGSAVE_START 状态之后的写命令, 都会写入到 slave 的写缓冲区, 等待发送给 slaves
4. 线性遍历 slaves, 效率不太好


总之, 全量同步实现比较简单:

* 把所有对 master 的更改命令在 slave 上回放
* 对 slave 来说, master 只是一个 client, slave 接收并执行这个特殊的 client 发送过来的命令. 并没有用什么特殊的命令和消息格式来做复制
* 每次 slave 链接到 master 都要做一个 master 的镜像, 然后把镜像发送到 slave 这边, 再 load 进来.

采用这样的方式来实现同步主要是基于 Redis "简洁至上" 的原则, 也就是尽量少的引入复杂性来完成任务.
采用全量同步来实现主从间的同步, 代码改动比较少, 而且实现逻辑简单, 很容易的就保证了整个同步过程的一致性, 提供了高可靠.

但这种简单的实现方式也付出了代价: 在生产环境中, 如果主从间的网络不稳定, 会频繁的发生全量同步, 而全量同步时, 系统的性能会急剧下降 (从节点会阻塞, 主节点要用 COW 的方式来生成RDB文件)

事实上很多公司在线上大规模的使用 Redis 时, 都对 Reids 的同步机制做了优化. 而 Redis 的作者在使用者抱怨很久以后, 也在 2.8 中加入了增量同步功能, 希望能解决这个问题

下面介绍一下新增加的增量同步功能


### 三, 增量同步 (PSYNC)

在实现 PSYNC 的时候, Redis 的作者也先考虑怎样最简洁的来实现, 避免引入过多的复杂性:

最直接的实现想法就是用一个 AOF 文件记录 master 所有的数据, 这样当 slave 需要做增量同步时, 只要根据 offset 位置从 AOF 文件中拿到需要增量同步的数据发送给客户端就能实现增量同步;

但是这样的实现方式需要写 AOF 文件代价比较大. 作者最后实现的方法是用内存, 而不是用文件来解决问题!

而想用内存来解决问题, 就不可能保存太多的数据, 所以做了两个需求上的妥协:
slave 断开后需要尽早的连上来, 才能做增量同步, 如果间隔了太久才连上来, 只能做全量同步 (断开后很快就连上来, master 那边就不用保存太多的数据, 可以用内存来解决)
断开的原因不是 master 重启了 (重启之后, 内存里面的数据就都丢失了, 只能做全量同步)

做这两个妥协来实现增量同步还是划得来的, 因为我们之前讨论过, 增量同步主要时解决网络瞬断的问题

重连发生时, 大概的过程:

slave端:

```
[60051] 17 Jan 16:52:54.979 * Caching the disconnected master state.
[60051] 17 Jan 16:52:55.405 * Connecting to MASTER...
[60051] 17 Jan 16:52:55.405 * MASTER <-> SLAVE sync started
[60051] 17 Jan 16:52:55.405 * Non blocking connect for SYNC fired the event.
[60051] 17 Jan 16:52:55.405 * Master replied to PING, replication can continue...
[60051] 17 Jan 16:52:55.405 * Trying a partial resynchronization (request 6f0d582d3a23b65515644d7c61a10bf9b28094ca:30).
[60051] 17 Jan 16:52:55.406 * Successful partial resynchronization with master.
[60051] 17 Jan 16:52:55.406 * MASTER <-> SLAVE sync: Master accepted a Partial Resynchronization.
```

master端:

```
[59968] 17 Jan 16:52:55.406 * Slave asks for synchronization
[59968] 17 Jan 16:52:55.406 * Partial resynchronization request accepted. Sending 0 bytes of backlog starting from offset 30
```

整个过程就是先尝试做增量同步: 只要能从 master 端把增量同步offset点之后的数据拿到, 就能成功的做增量同步

具有了增量同步, Reids 会更好更强大 (作者原话). 当前 2.8 已经开发完成, 正在测试阶段, 相信很快会 release.

### 参考:

* http://antirez.com/news/47
* http://antirez.com/news/49