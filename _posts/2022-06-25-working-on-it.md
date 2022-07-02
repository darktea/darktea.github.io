---
title: Working on it
date: 2022-06-25 00:00:00 +0800
categories: [notes]
tags: [notes]
---

分 2 部分介绍：

* 「日常需求」的完整工作流程
* 「线上服务」的运维

## 一. 流程

前提：

* 效率相关（相关指标）
* 故障相关
  * 定义什么是故障
  * 定义故障等级（受影响用户数 + 时长）
  * 设定目标

------

工作流程包括以下环节：

* 技术评审
* 变更管理：
  * 代码分支管理
  * 代码提交卡点：代码规范检测，安全审核，中间件版本检测，CR，单测通过。。。
  * 数据库（或中间件）变更规范
  * 系统变更管理
* 推进：开发环境 -> 联调环境 -> 测试环境 -> 预发环境
* 测试：自测        ->  联调        -> QA 测试   -> QA 验证通过 -> 是否压测？
* 发布：灰度 -> 引流 -> 分批发布
* 回滚

### 1. 技术评审

* 需求的开发负责人组织评审工作
* 形式可以多样，但涉及的相关人员都要确认（特别是 QA 人员的确认）
* 基于「技术评审」的结果确认所有工作的时间表（特别是 QA 人员的测试时间表）

### 2. 变更管理

* 代码分支管理：主干开发模式 vs. 迭代分支模式
* 代码规范 & 安全审核 & 中间件版本检测 & 单测通过
* CR 卡点时机？
* 数据库变更规范 & 中间件变更规范
* 数据库/中间件/系统变更的回滚方案

### 3. 环境 & 效率

* 建设稳定的开发/测试环境的目标是效率
  * 自测环境 -> 开发效率
  * 联调环境 -> 联调效率
  * 测试环境 -> QA 测试效率

### 4. 测试流程

* 提测时间点
* 在哪个环境提测的讨论：
  * 「测试环境」vs.「预发环境」（需要同时测试多个需求？）
* QA 人员发布「测试报告」
  * 是否压测在「技术评审」时确定

### 5. 发布

* 「发布」和「运维」相互关联
  * 「红线」
* 「发布方案」也是「技术评审」的一部分
* 依据「效率」&「故障」指标来制定对应的「发布流程规范」
  * 设置各个发布阶段的卡点策略（灰度 -> 引流 -> 分批发布）
  * 需要相关「运维」工具的支持
    * 日志/监控/报警
    * 限流/降级
* 回滚流程（应急手册 + 事先演练）

### 6. 中间件的迭代

* 监控和统计
  * 版本分布/接口分布（日志和统计接口）
* CR 策略
* 测试策略
  * 回归测试/压力测试（自动化测试用例集合）
* 发布策略

## 二. 运维

### 1. 运维粒度

* 业务级别 / 服务级别 / 应用级别。。。
* 系统级别 / 单机级别 / 容器级别。。。

### 2. 运维指标

各个「运维粒度」对应的运维指标：

* 业务级别：业务指标
* 服务级别：QPS/耗时/失败率
* 应用级别：CPU / 负载 / 内存 / 错误数。。。
* 系统级别：上下游错误数
* 单机 / 容器级别：CPU / 负载 / 内存 / 错误数。。。

但最重要的核心指标还是围绕服务级别：QPS/耗时/失败率

### 3. 链路梳理

* 强弱依赖/系统瓶颈/系统容量 -> 压测 -> 风险评估
  * 配合具体业务目标进行梳理
* 压测相关
  * 压测的风险：也是故障的普遍来源之一
  * 压测流量的区分
  * 压测策略的选择（单机/接口级/应用级/单链路级/全链路级别）

### 4. 报警监控应急

监控 -> 报警 -> 应急手册

依赖各种运维工具：

* 监控相关
  * 业务指标跟踪
  * 链路跟踪 + Metric（基于中间件日志）
  * 中间件日志/应用日志错误实时分析
  * 系统指标监控（CPU，内存，负载，网络，JVM。。。）
* 报警相关
  * 依据「故障」指标来对报警进行设置
  * 核心指标：出现故障到相关人员上线处理的时长
* 应急相关（基于之前的「链路梳理」，「监控」，「报警」）
  * 预案可能需要包括以下方面：
    * 人员 & 沟通
    * 限流
    * 降级
    * 回滚
  * 应急演练（例如进行压测的同时，对各种预案进行演练）

## 三. 重点

### 1. 代码规范检测

Java 语言的话，可以使用一些静态代码分析工具，常用的有：

* 阿里编码规约
* PMD
* SonarLint

具体使用方法，一般有 2 种使用方法：

* 安装对应的 Intellij 插件。开发人员在编码和 CR 时，利用这些 IDE 插件确认代码能通过这些检测
* 利用对应的命令行工具（可能是 maven 插件的形式）对代码库中的代码自动扫描

### 2. 技术评审

「日常需求任务」的技术评审主要包括以下要素：

* 任务背景和任务内容的介绍
* 涉及到的系统的工作细节（例如：「时序图」）
* 涉及到的相关「接口」的文档
* 本次任务的实现方案（选择该方案的考量？计划怎么做？涉及哪些系统的改动？）
* 列举需要变更的各个系统的改动点（可以精确到代码级别）
  * 各个改动的负责人也要在这个时候明确
* 埋点+统计方案和大概的验证方法（如果需要）
* 中间件的改动（如果需要。可能包括：DB，缓存，RPC。。。）
* 性能影响和压测方案（如果需要）
* 发布方案（如果需要）
* 风险评估及其应对方案（如果需要）
* 开发时间预估
  * 特别是给出**最终关键时间节点**的「时间」
    * 例如：「什么时候能确定**提测时间**」

### 3. 基础设施变更的回滚

也就是类似「数据库变更」，「中间件版本变更」，「系统软件变更」等基础设施变更的回滚方法。

先讲一下「数据库变更」的回滚：

* 所谓「数据库变更」最常见的场景其实就 3 种：新增表，加字段，改字段类型。这 3 种变更的回滚其实还好
  * 「加索引」要谨慎（变更和回滚方案要和 DBA 确认）
  * 其他不常见的场景，就需要事先和 DBA 制定详细的计划了（例如在「技术评审」阶段）

------

再讲一下「其他基础设施变更」的回滚，例如「中间件版本」，「系统软件」等设施的变更：

* 在做回滚之前，先要确认「交付物」是什么：
  * 如果是当前流行的：镜像（Docker 镜像这种），Mesh，Serverless，比较简单，使用对应的成熟方案即可
  * 如果还是传统的「二进制包」，就需要发布人员有一套稳定的「脚本工具」来实施对应的发布/回滚等工作
* 另外，回滚方案可能需要演练

### 4. 中间件版本/接口管理

方法如下：

* 中间件日志的目录（例如：/home/admin/logs）和应用服务的目录分开
* 一般中间件目录下会重点记录以下日志（这些日志会记录到不同的日志文件）：
  * trace 信息
  * 实施监控统计信息（看具体情况，可以到分钟级，30 秒级等。。。）
  * 错误信息
* 然后可以通过这些日志掌握当前中间件运行的状况，基于这些状况可以实现监控，报警
* 甚至可以暴露 HTTP 接口来查询到中间件的运行状况

有了以上这些单机的信息之后，就可以汇总得到中间件「可视化大盘」

### 5. 监控指标

这里重点介绍一下需要重点关注（监控+记录）的单机指标（有了单机指标后，汇总起来就能得到整体指标，并可以组合出各种报警）：

* 系统（物理机和容器）级别的指标：
  * CPU 相关（用户级，系统级，等待，切换，steal。。。）
  * 内存 相关
  * 磁盘 相关
  * Load（负载）
  * IO 相关：「网络流量」，「网络错误」，「磁盘 IO」，「磁盘错误」等
* JVM 指标：
  * GC 相关
  * 堆相关
  * 线程数
* 接口级别的指标（本身接口和依赖的上下游接口）：
  * 实时流量（秒级，分钟级。。。）
  * 实时失败数
  * 实时失败率（「成功率」）
  * 实时 RT（也就是延时）：平均，最大，百分比
* 应用级别的指标：
  * 实时日志中的 error 数
  * 实时业务指标
  * 实时资损（如果有的话）
* 中间件级别的指标：
  * 包括 DB，缓存，RPC，消息队列。。。
  * 限流相关监控
* 测试用例定时执行（预发环境 + 线上环境）