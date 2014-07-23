---
layout: post
title: On Designing and Deploying Internet-Scale Services
category: notes
---

<On Designing and Deploying Internet-Scale Services>
    -- James Hamilton – Windows Live Services Platform

# 0. 目的

本文是设计和部署易于运维的服务最佳实践的概括和总结.


先提出3个原则:

1. Expect failures; 所依赖的任何服务都可能发生故障, 需要优雅的处理所有的故障.
2. Keep things simple; 复杂性会引入问题; 任何一个 server 出故障不应该影响到 data center 的其他 servers;
3. Automate everything; 引入自动化以后, 系统更可测试, 更可修复, 更可依赖;

这 3 个原则会贯穿整篇文章

接下来分 10 个方面讨论怎样设计和部署易于运维的服务.

## 1. Overall Application Design

80% 以上的运维事故的根源在于设计和开发. 所以 Overall Application Design 是最重要, 因为要从根源上解决这些问题的手段通常就是在设计上和开发上做改进.

而且开发团队, 测试团队, 运维团队需要尽可能的一起工作, 这样能减少运维的成本.

下面列举为了实现一个易于运维的系统, Overall Application Design 时最最重要的 5 大原则:

### 1) Design for failure

* 比如说量上去以后, 每天都会有硬盘会坏; 
* 故障恢复路径必须是非常简单的路径, 而且故障恢复功能必须易于测试. 
* 测试故障最好的方法就是不要按正常流程来 shutdown 服务, 而是要 hard-fail it;


### 2) Redundancy and fault recovery

* 要到达 5 个 9, redundancy 是必须的; 靠单台服务要达到 4 个 9 也是非常困难的 (尽管这个观点已经是业界共识, 但是还是经常会看到很多服务构建在非 redundant 的数据层上) 
* 为了设计一个能达到 SLA 的系统必须非常仔细; 为了完全满足这个设计原则的 acid test 指的是: 运维团队有意愿, 有能力在任意时刻 shutdown 整个服务中的任意服务器, 同时不会降低整个系统的 work load
* 这里推荐一种设计手段来发现和纠正系统潜在的安全问题: security threat modeling (仔细考虑每种可能的 security threat, 针对每种 security threat, 实现合适的应对)
* 把所有想到的故障记录成文档; 确保每种故障时, 整个服务可以继续工作, 同时保证服务质量损失在可接受范围内; 
* 判断一些组合故障是 "unusual" 时, 必须很谨慎; 因为当规模上去以后,  Rare combinations 故障 can become commonplace.


### 3) Commodity hardware slice

* 用大规模的 commodity servers 来替换 少量的大型服务器会便宜很多
* server performance 一直会比 I/O 性能增长快; 对于同样数量的存储空间 (disks), 用小 server, 整个系统会更加平衡
* 电力消耗是和 server 数量成线性增长的; 但是和 时钟频率 成 cubically 级增长; 所以用高 performance 的 server 电费会更贵;
* 小 server 挂了以后影响的范围也小


### 4) Single-version software

* 如果一个服务的软件只需要以 single internal deployment 为目标, 同时软件 (面向企业的产品) 的旧版本不需要支持 10 年; 部署和升级的代价会非常小;
* 要做到Single-version software 需要: 1) 每次 release 时保证用户的体验不会变化 2) a willingness to allow customers that need this level of control to either host internally or switch to an application service provider willing to provide this people-intensive multi-version support

**备注**: 这里提到的 people-intensive 型的服务指的是: Those businesses tend to be more people intensive and more willing to run complex, customer specific configurations.

### 4) Multi-tenancy

* Multi-tenancy 优先的理念和 Single-version software 优先是一脉相承的
* 对自动化和大规模化有更好的支持; 节约服务成本


除了以上的 5 大原则, 还有其他一些设计易运维服务时的 best practices: 

* Quick service health check. 可以在开发环境快速跑的测试, 确保服务不会被破坏; 这种测试可以不包含各种边界测试, 但是要求 quick; 跑通以后就可以 check in 代码;
* Develop in the full environment. 整套服务能在单台 server 上搭起来, 这样开发人员可以容易的单元测试他负责的 components, 同时也可以测试这些 components 上的改动对整个 service 的影响
* Zero trust of underlying components. 假设所有依赖的服务都可能出故障, 同时要确认这些依赖都可以恢复重新提供服务. 一些通用的恢复技术: 
 * 继续在只读模式下访问 cached 的数据
 * 除了少部分用户外的绝大多数用户能够继续使用服务 (利用 redundant copy)
* Do not build the same functionality in multiple components; 前期的时候确实可能会一份代码到处 copy; 但如果长期不关注这个, 代码会快速恶化 (随着服务的快速增长, 进化)
* One pod or cluster should not affect another pod or cluster; 虽然很难做到, 但是还是要尽可能的让一个 cluster 所有的依赖都在这个 cluster 内;
* Allow (rare) emergency human intervention; 
 * 目标是所有场景都不需要人工操作; 
 * 但还是有极少的场景 (某些突发事件导致) 会需要人工处理 (这种场景下人为的错误往往是灾难的根源); 
 * 这些场景的处理办法不要用文档的形式来准备, 而是事先准备好脚本
 * 需要定期搞 "实弹演习", 验证这些脚本是可以 work 的
 * If the service-availability risk of a drill is excessively high, then insufficient investment has been made in the design, development, and testing of the tools
* Keep things simple and robust
 * 采用简单, 傻瓜的解决方法在 high-scale service 中比采用复杂的算法好
 * 其原则是, 如果复杂的方法能带来的 magnitude 级别的改进才值得上; 如果只是 percentage or even small factor 的改进就算了;
* Enforce admission control at all levels
 * 好点的系统都会在最前端有一个流量进入控制; 
 * 但也需要在所有重要的组件上有 流量控制机制 (有可能总的流量没有过载, 但是由于某种原因, 某个组件上的流量过载了, 如果没有相应的控制, 最终会导致整个服务受到影响)
 * 通用准则是: 尝试优雅的 degrading, 而不是 hard failing (block entry to the service before giving uniform poor service to all users); 后面会详细展开讨论这方面的内容
* Partition the service; Partition 的粒度必须把握好; 很多时候不能简单的按照真实世界的界限来做 Partition (按用户, 公司...); 合理粒度划分出来的 Partition 可以在各个 servers 间自由迁移;
* Understand the network design; 开发人员必须 understand 网络设计; 必须尽早的和运维团队 review networking specialists; 必须尽早的 understand 同机架, 不同机架, 不同 data center 间的带宽;
* Analyze throughput and latency; 
 * 必须做 UI 相关的核心服务的 throughput 和 latency 的分析工作; 了解其影响;
 * 特别是在一些日常运维操作 (数据库维护, 配置更新, 服务 debugging...) 发生的同时也对 throughput 和 latency 进行分析; 这样能帮忙定位这些操作带来的问题;
 * 对每个服务需要度量容量规划指标. 例如: 每秒用户请求数; 并行在线用户数, 以及其他一些通过 work load 来反映资源需求量的指标
* Treat operations utilities as part of the service; 所有的 Operations utilities 必须 code review 过, 提交到代码库中, 被测试过; (实际上 Operations utilities 经常被当作低优先级的事情来处理, 也没有被测试过)
* Understand access patterns: 对每个新 feature 都要考虑到它对后端 load 的影响; 上线之前必须量化和验证这种影响;
* Version everything: 前面提了, 目标是 single version software; 但实际上多版本是不可避免的 (联调测试环境...) 所有组件的版本 n 和 版本 n+1 需要能共存;
* Keep the unit/functional tests from the last release: 不仅仅用单元测试和功能测试来验证前一个版本的功能没有出问题; 在生产环境也跑单元测试来对服务进行验证 (这个后面会详细讲)
* Avoid single points of failure: 
 * 无状态的实现优先; 
 * 不要把某个 client 的请求限制在某个 server 上; 
 * 基于静态hash之类的负载均衡可能也会有问题; 
 * 用好的 Partition 策略 (前面提过了)
 * 尽量少做跨 Partition 的操作
 * All database state is stored redundantly (on at least one) fully redundant hot standby server and failover is tested frequently in production


## 2. Automatic Management and Provisioning

* 许多 service 使用: 故障检测->报警->人工恢复 的模式;
* 这种模式的问题在于在压力下, 20%的时候可能会出现人为的错误
* 结果就是运维成本高, 而且降低了整个服务的可靠性


但是 Designing for automation 也会给服务 model 带来很大的限制 (例如, 数据库主从切换时可能丢数据, 要保证一致性 automation 的方式可能处理不好);
所以为了 automation, 服务质量上可能付出一定的代价;


下面列出一些 automation 设计的 best practices:

* Be restartable and redundant: 服务可重启; 存储有冗余;
* Support geo-distribution: 好像我们这里提到的 automation 都可以没有 geo-distribution; 但实际上, 如果一个 high-scale 的服务不支持 geo-distribution 的话, 由于运维上的限制, 成本会高很多
* Automatic provisioning and installation: 加快部署速度, 降低部署成本
* Configuration and code as a unit: 需要确保 1) 开发团队把代码和配置作为一个整体来发布 2) 在测试中采用和部署时同样的部署流程 3) 运维团队把代码和配置作为一个整体来部署
* 如果对线上系统做了更新, 有 log 记录所有的更新内容 (更新了什么? 什么时候更新的? 谁做的操作? 哪些服务器上做了操作?); 同时要经常 scan 所有服务器的状态是否正确 (确保配置什么的没有问题)
* Manage server roles or personalities rather than servers: 部署到哪些 servers 上要看部署服务的角色和身份, 并不会限定服务应该被部署到哪些具体的 servers 上;
* Multi-system failures are common:  Expect failures of many hosts at once (power, net switch, and rollout)
* Recover at the service level: 在 service 层做 recovery, 而不是在底层的软件层 (比如在 service 层做冗余, 而不是在底层的软件层做)
* Never rely on local storage for non-recoverable information: Always replicate all the non-ephemeral service state
* Keep deployment simple: 不要用复杂的安装脚本; 通过 文件 copy 来部署是最理想的部署模式; 要避免: 同一个 component 的不同版本, 或者不同的 components 如果不能跑在同一个 server 上;
* Fail services regularly:  
 * 定期通过 take down data centers, shut down racks, and power off servers 等手段来暴露服务, 系统, 网络的弱点; 
 * Those unwilling to test in production aren't yet confident that the service will continue operating through failures
 * And, without production testing, recovery won't work when called upon


## 3. Dependency Management

As a general rule, dependence on small components or services doesn't save enough to justify the complexity of managing them.
两种依赖是可以被 justified 的:

* 正在被依赖的 components 数量多和复杂度高 (例如: 存储服务, 一致性算法等)
* 正在被依赖的 service 作为一个 single, central instance (例如: 认证管理系统)

假设现有的 dependencies 满足上面的那些规则, 那么 dependencies are justified. 

对这些 可以被 justified 的 dependencies, 下面列举一些管理它们的 best practices:

* Expect latency: 设置 timeout;
* Isolate failures: 必须防止 cascading 式的故障, 尽量 "fail fast"
* Use shipping and proven components; 宁愿用版本老一点, 但是稳定的软硬件; (就算新版本性能好, 功能多)
* Implement inter-service monitoring and alerting; 出问题的时候要能联系到人来解决
* Dependent services require the same design point; 相同的 SLA;
* Decouple components: 其他 components 故障的时候, 要能继续工作 (可能在 degraded 模式)


## 4. Release Cycle and Testing

* 对所有的 internet-scale 服务, 在线上环境进行测试是一种必备 QA 的手段
* 尽管很多服务都有一个模拟线上环境的测试环境 (会在这个环境上模拟线上负载来进行测试); 但实际上, 无论怎么样, 这个模拟环境不能完美的模拟线上环境
* 推荐的做法是在模拟环境 (尽可能的模拟线上环境) 测试之后, 可以在线上环境 (会做一些限制) 做最后阶段的测试; 
* 但需要注意的是, 我们不希望在线上环境做测试会影响到线上服务. 所以必须非常小心, 同时遵照以下规则:
 * 线上环境必须要充分的冗余; 出现问题可以快速切换
 * 必须保证数据不会被破坏, 状态相关的故障不会发生
 * 错误必须能检测到; 开发团队 (而不是运维团队) 必须对系统进行监控
 * 必须能够快速回滚; 而且回滚动作本身之前必须测试通过

听起来比较危险, 但是实际上效果很好, 而且可以和灰度发布配合起来
灰度发布实际上会降低风险 (对比一下子把更新上线)

另外, 做做到了上面这些规则之后, 在白天上线比在半夜上线好:

* 半夜操作更容易犯错
* 出了问题半夜找人更困难

下面列出 Release Cycle and Testing 相关的 best practices:

* 更频繁的发布; 可以促进发布的质量的提高, 用户体验会更好. 发布周期大于 3 个月是危险的 (有的团队现在已经做到了 Week 级别的发布周期)
* Use production data to find problems: 大规模系统的 QA 问题已经不仅仅是测试问题, 而是 data-mining 和 visualization 问题. 需要从线上环境采集大量的数据:
 * 测量发布标准: 根据用户体验来定义发布规格; 并持续对这个指标进行监控 (例如, 可用性如果被设置为 99%, 就持续监控报警是否真的达到这个标准)
 * 不要纠结需要把标准定为多少, 99%? 99.9%? 而是设置一个可接受的目标, 随后不断进行提升
 * 收集真实的数值, 而不是用 summary reports (例如, 用红灯, 绿灯什么的) 来体现
 * 避免过度报警;
 * Analyze trends; 用来预估问题
 * Make the system health highly visible; 有一个内部运维系统, 大家都可以了解当前服务的状态;
 * Monitor continuously; 如果每天都要看所有的监控数据, 时间久了就会失去效果; 可以搞成一个明确的任务, 大家轮流做 (每个人做一段时间)
 * Invest in engineering; 不懂?
 * Support version roll-back; 必须有回滚 (所有的部署都要有回滚, 就像保险带); 而且必须被测试过证明过; 
 * Stress test for load: 在线上环境的一个子集上跑 2 倍的压力
 * Perform capacity and performance testing prior to new releases; 在 service level 上做这个; 也在每个 component 上做这个;
 * Build and deploy shallowly and iteratively; 在开发的早期就引入 a skeleton version of the full service;
 * Test with real data; tcp copy 测试
 * Run system-level acceptance tests; 不懂?
 * Test and develop in full environments; 用预留的硬件搭建测试环境, 尽量保持和线上的硬件配置一致, 最大化投资效果 (以后这些机器可以直接用到线上环境)


## 5. Hardware Selection and Standardization

用 services fabric 模式来进行硬件采购和管理. 使用 services fabric 模式保证了两个关键原则:

* 对所有的 services (即使规模比较小) 进行自动管理和自动运维;
* 新的 service 能被更快速的测试和部署

下面列举硬件选型相关的 best practices:

* Use only standard SKUs; 只用一种或少数几种 SKU; 这样可以保证线上环境中的各种 services 的资源能够容易的互相调配; 最有效的方式就是部署一套标准的 service-hosting framework, 其中包括自动管理, 自动运维, 硬件, 共享 services; 
* Purchase full racks; 购买全配置好, 而且全测试好的 racks; Racking and stacking costs are inexplicably high in most data centers, so let the system manufacturers do it and wheel in full racks.
* Write to a hardware abstraction; services 不能绑定在具体的硬件 SKU 上面; 而是按照 hardware abstraction 来做 services (CPU数, 内存数, 硬盘大小...)
* Abstract the network and naming; using DNS and CNAMEs; 而不是用 机器名, If you need to avoid flushing the DNS cache, remember to set Time To Live sufficiently low to ensure that changes are pushed as quickly as needed


## 6. Operations and Capacity Planning

高效对 services 进行运维的关键在于: 构建一个系统来消除绝大多数的运维交互操作; 这个系统的目标就是要用 8\*5 /周的人力来运维一个高可用, 7\*24 的服务;

开发团队预先准备好针对紧急情况的恢复动作 (可以用脚本); 而且事先要在线上环境对这些脚本进行过测试; 如果这些脚本在线上环境测试风险太大, 那么到出现紧急情况的时候, 这些脚本也不安全, 不可用;

如果一个小问题没有按预先计划的那样被自动恢复, 在很多时候会引发大灾难

下面列举相关的  best practices:

* Make the development team responsible; "谁开发的谁运维". 这个可能太激进了, 不过方向是这个方向; 要推动开发团队改进易运维性
* Soft delete only; Never delete anything; 用标记删除; 如果被误操作删除了就没有办法恢复了. RAID, 镜像什么的都不能服务这种错误; 
* Track resource allocation; services 都会定义一些负载指标 (同时在线用户数, QPS...), 关键是必须了解这些负载指标和硬件资源消耗的关系; 运维团队把这些数据反馈给市场和销售团队; 不同的 services 要求不同的采购周期; 
* Make one change at a time; 更新线上环境时, 每次只做一个更新; 
* Make Everything Configurable; 所有需要在线上环境修改的东西都做成可配置的, 避免改代码; 甚至在不知道这个值是否会在线上被修改的时候, 如果方便的话, 也尽量的做成可配置的 (不能随意在线上进行更动, 改动之前需要测好).


## 7. Auditing, Monitoring and Alerting

对所有的线上操作进行记录. 一旦有问题, 可以从最近动了哪些东西来找原因; 

Alerting is an art. 报得太多, 就会被忽略; 可以跟踪两个指标来判断报警是否合理:

* alerts-to-trouble ticket ratio; 越接近 1 越好
* number of systems health issues without corresponding alerts; 越接近 0 越好; 

下面列举一些 best practices:

* Instrument everything; 测量每个用户操作和事务, 报告其中的异常 (可以用一个程序模拟用户的操作, 但可能还是不够)
* Data is the most valuable asset; 从系统中收集数据来了解系统运行的状况 (是否真正的工作正常)
* Have a customer view of service; 可以用前面提到的模拟用户操作的程序来覆盖路径是否正常 (也是一种报警, 所以也需要合理报警)
* Instrumentation required for production testing; 在线上环境做测试, 详尽的报警和监控是必须的;
* Latencies are the toughest problem; 很慢但是没有失败的请求最难被发现; 需要确保能监控到这种状况;
* Have sufficient production data (哪些数据需要被监控?):
 * Use performance counters for all operations: 至少要记录每秒的 latency 和 QPS; 
 * Audit all operations: 一方面可以发现问题; 另外一方面可以发现用户在做什么 (但是最好运维人员不要用公共账号来操作)
 * Track all fault tolerance mechanisms; fault tolerance 掩盖错误, 所以需要记录每次 retry 事件, 每次副本复制; 
 * Track operations against important entities; 记录成文档, 进行数据分析, 发现数据中的异常; 如果在项目后期加这个比较困难, 要早点加进去;
 * Asserts; 多用 Asserts 来定位问题;
 * Keep historical data: 历史日志和历史performance都需要
* Configurable logging; 可以配置 level
* Expose health information for monitoring; 主动暴露一些接口提供信息供监控程序使用
* Make all reported errors actionable; 报警信息的内容要是 actionable 的;
* Enable quick diagnosis of production problems; 
 * Give enough information to diagnose; "10 queries returned no results" 不够充分, 应该补充"and here is the list, and the times they happened" 
 * Chain of evidence: 需要给开发者提供一条完成的路径来诊断问题
 * Debugging in production; 最好不要到线上操作, 而是把信息 (内存镜像...) 导出来 debug; 如果真的要到线上调式, 最好的操作人选是开发人员 (之前必须被训练过); 不过不到万不得以还是推荐不要到线上去;
 * Record all significant actions; This includes both when a user sends a command and what the system internally does; 更重要的是, 开发 mining 工具发现有用的 aggregates (例如: 用户正在查的流行词汇)


## 8. Graceful Degradation and Admission Control

2 个 best practices:

* 'big red switch'
* admission control

针对这两点, 每个 services 都要度身定做; 这两点都非常重要:

* big red switch 是一种事先设计好的, 可测试的动作; 当 service 不能满足 SLA, 或者是即将不能满足 SLA 时,  可以执行这些动作; 
* big red switch 的理念就是保证关键服务继续, 同时暂时 shedding 或 delaying 不重要的 workload
* queue 之类的地方就是 big red switch 的候选
* 关键点是在系统有问题时, 什么是保证系统运行的最低要求;
* 需要测试 degrade 之后, 系统能切回正常模式
* admission control 具体怎么好可以根据具体的系统来设计 (只放 vip 用户进来? 不再 queue 请求等等)
* Meter admission; 在做了 admission control 以后, 有一个关键点是之后能够修改这个控制点; 例如, 服务正常以后, 通过调整 admission control, 慢慢的恢复系统 (用多大的粒度恢复服务或发布新的release也是很重要的, 通过和用户的沟通, 引导用户的期望)
* Another client-side trick that can be used to prevent them all synchronously hammering the server is to introduce intentional jitter and per-entity automatic backup?


## 9. Customer and Press Communication Plan

系统有问题时 (故障, 延迟比较大) 必须通知用户; 和用户的沟通需要有多种渠道; 

有 client 的软件可以做很多有益的事情...

如果没有 client 而是 web page 的应用, 也可以有很多方式来和用户交流; 用户对系统状态了解更多, 用户的满意度就会更高 (系统的 owner 往往倾向于不告知用户系统的故障, 但我们的经验如果告诉用户, 用户满意度会高很多)

需要在平时提前准备好沟通计划; 每类事件需要提前计划好, 到时候 who to call, when to call them, and how to handle communications


## 10. Customer Self-Provisioning and Self-Help

用户自服务可以节约成本, 提升用户的满意度; 例如用户能登录 web page, 自己输入自己需要的数据, 然后就可以开始使用服务; 