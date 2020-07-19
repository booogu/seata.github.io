---
title: 分布式事务如何实现？深入解读 Seata 的 XA 模式
keywords: Seata,分布式事务,XA,AT
description: 深入解读 Seata 的 XA 模式
author: 煊檍
date: 2020/04/28
---

# 分布式事务如何实现？深入解读 Seata 的 XA 模式

作者简介：煊檍，GitHub ID：sharajava，阿里巴巴中件间 GTS 研发团队负责人，SEATA 开源项目发起人，曾在 Oracle 北京研发中心多年，从事 WebLogic 核心研发工作。长期专注于中间件，尤其是分布式事务领域的技术实践。

Seata 1.2.0 版本重磅发布新的事务模式：XA 模式，实现对 XA 协议的支持。

这里，我们从三个方面来深入解读这个新的特性：

- 是什么（What）：XA 模式是什么？
- 为什么（Why）：为什么支持 XA？
- 怎么做（How）：XA 模式是如何实现的，以及怎样使用？

# 1. XA 模式是什么？

这里有两个基本的前置概念：

1. 什么是 XA？
2. 什么是 Seata 定义的所谓 事务模式？

基于这两点，再来理解 XA 模式就很自然了。

## 1.1 什么是 XA？

XA 规范 是 X/Open 组织定义的分布式事务处理（DTP，Distributed Transaction Processing）标准。

XA 规范 描述了全局的事务管理器与局部的资源管理器之间的接口。 XA规范 的目的是允许的多个资源（如数据库，应用服务器，消息队列等）在同一事务中访问，这样可以使 ACID 属性跨越应用程序而保持有效。

XA 规范 使用两阶段提交（2PC，Two-Phase Commit）来保证所有资源同时提交或回滚任何特定的事务。

XA 规范 在上世纪 90 年代初就被提出。目前，几乎所有主流的数据库都对 XA 规范 提供了支持。

## 1.2 什么是 Seata 的事务模式？

Seata 定义了全局事务的框架。

全局事务 定义为若干 分支事务 的整体协调：

1. TM 向 TC 请求发起（Begin）、提交（Commit）、回滚（Rollback）全局事务。
2. TM 把代表全局事务的 XID 绑定到分支事务上。
3. RM 向 TC 注册，把分支事务关联到 XID 代表的全局事务中。
4. RM 把分支事务的执行结果上报给 TC。（可选）
5. TC 发送分支提交（Branch Commit）或分支回滚（Branch Rollback）命令给 RM。

![img](https://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapOODXhMq7lSW4fHRRa0EvINCYVjnMIibW6xSIrocTZPGUj0669qvTQUEOohd3eIa0NnJOhpSl9vhMtQ/640?wx_fmt=png)

Seata 的 全局事务 处理过程，分为两个阶段：

- 执行阶段 ：执行 分支事务，并 保证 执行结果满足是 *可回滚的（Rollbackable）* 和 *持久化的（Durable）*。
- 完成阶段： 根据 执行阶段 结果形成的决议，应用通过 TM 发出的全局提交或回滚的请求给 TC，TC 命令 RM 驱动 分支事务 进行 Commit 或 Rollback。

Seata 的所谓 事务模式 是指：运行在 Seata 全局事务框架下的 分支事务 的行为模式。准确地讲，应该叫作 分支事务模式。

不同的 事务模式 区别在于 分支事务 使用不同的方式达到全局事务两个阶段的目标。即，回答以下两个问题：

- 执行阶段 ：如何执行并 保证 执行结果满足是 *可回滚的（Rollbackable）* 和 *持久化的（Durable）*。
- 完成阶段： 收到 TC 的命令后，如何做到分支的提交或回滚？

以我们 Seata 的 AT 模式和 TCC 模式为例来理解：

AT 模式

![img](https://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapOODXhMq7lSW4fHRRa0EvINCL2xqvpPKZ9JGFCHBjNMicmU3b4hEhDlq93TJ3fL2zRIWAYMJrDRsDgQ/640?wx_fmt=png)

- 执行阶段：

- - 可回滚：根据 SQL 解析结果，记录回滚日志
  - 持久化：回滚日志和业务 SQL 在同一个本地事务中提交到数据库

- 完成阶段：

- - 分支提交：异步删除回滚日志记录
  - 分支回滚：依据回滚日志进行反向补偿更新

TCC 模式

![img](https://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapOODXhMq7lSW4fHRRa0EvINCeicwxNibuIYy5clHCWPC9cN5YFtibPmJibm29ZzxsDyudBXRn0BVDMKgrQ/640?wx_fmt=png)

- 执行阶段：

- - 调用业务定义的 Try 方法（完全由业务层面保证 *可回滚* 和 *持久化*）

- 完成阶段：

- - 分支提交：调用各事务分支定义的 Confirm 方法
  - 分支回滚：调用各事务分支定义的 Cancel 方法

## 1.3 什么是 Seata 的 XA 模式？

XA 模式：

在 Seata 定义的分布式事务框架内，利用事务资源（数据库、消息服务等）对 XA 协议的支持，以 XA 协议的机制来管理分支事务的一种 事务模式。

![img](https://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapOODXhMq7lSW4fHRRa0EvINCBBfPegN1aYkRXEmxMnEFt2f8xV5pyVbY9L6pTmhQXM7OZTBUs7P3Lw/640?wx_fmt=png)

- 执行阶段：

- - 可回滚：业务 SQL 操作放在 XA 分支中进行，由资源对 XA 协议的支持来保证 *可回滚*
  - 持久化：XA 分支完成后，执行 XA prepare，同样，由资源对 XA 协议的支持来保证 *持久化*（即，之后任何意外都不会造成无法回滚的情况）

- 完成阶段：

- - 分支提交：执行 XA 分支的 commit
  - 分支回滚：执行 XA 分支的 rollback

# 2. 为什么支持 XA？

为什么要在 Seata 中增加 XA 模式呢？支持 XA 的意义在哪里呢？

## 2.1 补偿型事务模式的问题

本质上，Seata 已经支持的 3 大事务模式：AT、TCC、Saga 都是 补偿型 的。

补偿型 事务处理机制构建在 事务资源 之上（要么在中间件层面，要么在应用层面），事务资源 本身对分布式事务是无感知的。

![img](https://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapOODXhMq7lSW4fHRRa0EvINC2KZeXbRyd6rPV6wUzQfakLIo9WD4wTugNmbSWqiagsFISsGicL0R4GeQ/640?wx_fmt=png)

事务资源 对分布式事务的无感知存在一个根本性的问题：无法做到真正的 全局一致性 。

比如，一条库存记录，处在 补偿型 事务处理过程中，由 100 扣减为 50。此时，仓库管理员连接数据库，查询统计库存，就看到当前的 50。之后，事务因为异外回滚，库存会被补偿回滚为 100。显然，仓库管理员查询统计到的 50 就是 脏 数据。

可以看到，补偿型 分布式事务机制因为不要求 事务资源 本身（如数据库）的机制参与，所以无法保证从事务框架之外的全局视角的数据一致性。

## 2.2 XA 的价值

与 补偿型 不同，XA 协议 要求 事务资源 本身提供对规范和协议的支持。

![img](https://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapOODXhMq7lSW4fHRRa0EvINCibTpm0VkkPpFy34dAeMnof17L9IzAH3IeMLmh7hiaOj2U68W0QrU3f1w/640?wx_fmt=png)

因为 事务资源 感知并参与分布式事务处理过程，所以 事务资源（如数据库）可以保障从任意视角对数据的访问有效隔离，满足全局数据一致性。

比如，上一节提到的库存更新场景，XA 事务处理过程中，中间态数据库存 50 由数据库本身保证，是不会仓库管理员的查询统计 *看* 到的。（当然隔离级别需要 读已提交 以上）

除了 全局一致性 这个根本性的价值外，支持 XA 还有如下几个方面的好处：

1. 业务无侵入：和 AT 一样，XA 模式将是业务无侵入的，不给应用设计和开发带来额外负担。
2. 数据库的支持广泛：XA 协议被主流关系型数据库广泛支持，不需要额外的适配即可使用。
3. 多语言支持容易：因为不涉及 SQL 解析，XA 模式对 Seata 的 RM 的要求比较少，为不同语言开发 SDK 较之 AT 模式将更 *薄*，更容易。
4. 传统基于 XA 应用的迁移：传统的，基于 XA 协议的应用，迁移到 Seata 平台，使用 XA 模式将更平滑。

## 2.3 XA 广泛被质疑的问题

不存在某一种分布式事务机制可以完美适应所有场景，满足所有需求。

XA 规范早在上世纪 90 年代初就被提出，用以解决分布式事务处理这个领域的问题。

现在，无论 AT 模式、TCC 模式还是 Saga 模式，这些模式的提出，本质上都源自 XA 规范对某些场景需求的无法满足。

XA 规范定义的分布式事务处理机制存在一些被广泛质疑的问题，针对这些问题，我们是如何思考的呢？

1. **数据锁定**：数据在整个事务处理过程结束前，都被锁定，读写都按隔离级别的定义约束起来。

> 思考：
>
> 数据锁定是获得更高隔离性和全局一致性所要付出的代价。
>
> 补偿型 的事务处理机制，在 执行阶段 即完成分支（本地）事务的提交，（资源层面）不锁定数据。而这是以牺牲 隔离性 为代价的。
>
> 另外，AT 模式使用 *全局锁* 保障基本的 *写隔离*，实际上也是锁定数据的，只不过锁在 TC 侧集中管理，解锁效率高且没有阻塞的问题。

2. **协议阻塞**：XA prepare 后，分支事务进入阻塞阶段，收到 XA commit 或 XA rollback 前必须阻塞等待。

> 思考：
>
> 协议的阻塞机制本身并不是问题，关键问题在于 协议阻塞 遇上 数据锁定。
>
> 如果一个参与全局事务的资源 “失联” 了（收不到分支事务结束的命令），那么它锁定的数据，将一直被锁定。进而，甚至可能因此产生死锁。
>
> 这是 XA 协议的核心痛点，也是 Seata 引入 XA 模式要重点解决的问题。
>
> 基本思路是两个方面：避免 “失联” 和 增加 “自解锁” 机制。（这里涉及非常多技术细节，暂时不展开，在后续 XA 模式演进过程中，会专门拿出来讨论）

3. **性能差**：性能的损耗主要来自两个方面：一方面，事务协调过程，增加单个事务的 RT；另一方面，并发事务数据的锁冲突，降低吞吐。

> 思考：
>
> 和不使用分布式事务支持的运行场景比较，性能肯定是下降的，这点毫无疑问。
>
> 本质上，事务（无论是本地事务还是分布式事务）机制就是拿部分 性能的牺牲 ，换来 编程模型的简单 。
>
> 与同为 业务无侵入 的 AT 模式比较：
>
> 首先，因为同样运行在 Seata 定义的分布式事务框架下，XA 模式并没有产生更多事务协调的通信开销。
>
> 其次，并发事务间，如果数据存在热点，产生锁冲突，这种情况，在 AT 模式（默认使用全局锁）下同样存在的。
>
> 所以，在影响性能的两个主要方面，XA 模式并不比 AT 模式有非常明显的劣势。
>
> AT 模式性能优势主要在于：集中管理全局数据锁，锁的释放不需要 RM 参与，释放锁非常快；另外，全局提交的事务，完成阶段 异步化。

# 3. XA 模式如何实现以及怎样用？

## 3.1 XA 模式的设计

### 3.1.1 设计目标

XA 模式的基本设计目标，两个主要方面：

1. 从 场景 上，满足 全局一致性 的需求。
2. 从 应用上，保持与 AT 模式一致的无侵入。
3. 从 机制 上，适应分布式微服务架构的特点。

整体思路：

1. 与 AT 模式相同的：以应用程序中 本地事务 的粒度，构建到 XA 模式的 分支事务。
2. 通过数据源代理，在应用程序本地事务范围外，在框架层面包装 XA 协议的交互机制，把 XA 编程模型 透明化。
3. 把 XA 的 2PC 拆开，在分支事务 执行阶段 的末尾就进行 XA prepare，把 XA 协议完美融合到 Seata 的事务框架，减少一轮 RPC 交互。

### 3.1.2 核心设计

#### 1. 整体运行机制

XA 模式 运行在 Seata 定义的事务框架内：

![img](https://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapOODXhMq7lSW4fHRRa0EvINCgesnUic0eEoBM4cHgbcybuXX8MRs4v9M7gDPM4Zib83W7wxP7r2qYgmA/640?wx_fmt=png)

- 执行阶段（E xecute）：

- - XA start/XA end/XA prepare + SQL + 注册分支

- 完成阶段（F inish）：

- - XA commit/XA rollback

#### 2. 数据源代理

XA 模式需要 XAConnection。

获取 XAConnection 两种方式：

- 方式一：要求开发者配置 XADataSource
- 方式二：根据开发者的普通 DataSource 来创建

第一种方式，给开发者增加了认知负担，需要为 XA 模式专门去学习和使用 XA 数据源，与 透明化 XA 编程模型的设计目标相违背。

第二种方式，对开发者比较友好，和 AT 模式使用一样，开发者完全不必关心 XA 层面的任何问题，保持本地编程模型即可。

我们优先设计实现第二种方式：数据源代理根据普通数据源中获取的普通 JDBC 连接创建出相应的 XAConnection。

类比 AT 模式的数据源代理机制，如下：

![img](https://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapOODXhMq7lSW4fHRRa0EvINCBp8qwfiadnF3VpdXVGY5YMFzt4QFo7Nas81qFB4japicMeD376Kxuu3Q/640?wx_fmt=png)

但是，第二种方法有局限：无法保证兼容的正确性。

实际上，这种方法是在做数据库驱动程序要做的事情。不同的厂商、不同版本的数据库驱动实现机制是厂商私有的，我们只能保证在充分测试过的驱动程序上是正确的，开发者使用的驱动程序版本差异很可能造成机制的失效。

这点在 Oracle 上体现非常明显。参见 Druid issue：https://github.com/alibaba/druid/issues/3707

综合考虑，XA 模式的数据源代理设计需要同时支持第一种方式：基于 XA 数据源进行代理。

类比 AT 模式的数据源代理机制，如下：

![img](https://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapOODXhMq7lSW4fHRRa0EvINCcqvY2bOEC0NoCLRlg0humicdTaaF7kFawH8xXLE2s1xvB9T4vibCRmtg/640?wx_fmt=png)

#### 3. 分支注册

XA start 需要 Xid 参数。

这个 Xid 需要和 Seata 全局事务的 XID 和 BranchId 关联起来，以便由 TC 驱动 XA 分支的提交或回滚。

目前 Seata 的 BranchId 是在分支注册过程，由 TC 统一生成的，所以 XA 模式分支注册的时机需要在 XA start 之前。

将来一个可能的优化方向：

把分支注册尽量延后。类似 AT 模式在本地事务提交之前才注册分支，避免分支执行失败情况下，没有意义的分支注册。

这个优化方向需要 BranchId 生成机制的变化来配合。BranchId 不通过分支注册过程生成，而是生成后再带着 BranchId 去注册分支。

#### 4. 小结

这里只通过几个重要的核心设计，说明 XA 模式的基本工作机制。

此外，还有包括 *连接保持*、*异常处理* 等重要方面，有兴趣可以从项目代码中进一步了解。

以后会陆续写出来和大家交流。

### 3.1.3 演进规划

XA 模式总体的演进规划如下：

1. 第 1 步（已经完成）：首个版本（1.2.0），把 XA 模式原型机制跑通。确保只增加，不修改，不给其他模式引入的新问题。
2. 第 2 步（计划 5 月完成）：与 AT 模式必要的融合、重构。
3. 第 3 步（计划 7 月完成）：完善异常处理机制，进行上生产所必需的打磨。
4. 第 4 步（计划 8 月完成）：性能优化。
5. 第 5 步（计划 2020 年内完成）：结合 Seata 项目正在进行的面向云原生的 Transaction Mesh 设计，打造云原生能力。

## 3.2 XA 模式的使用

从编程模型上，XA 模式与 AT 模式保持完全一致。

可以参考 Seata 官网的样例：seata-xa

样例场景是 Seata 经典的，涉及库存、订单、账户 3 个微服务的商品订购业务。

在样例中，上层编程模型与 AT 模式完全相同。只需要修改数据源代理，即可实现 XA 模式与 AT 模式之间的切换。

```
    @Bean("dataSource")
    public DataSource dataSource(DruidDataSource druidDataSource) {
        // DataSourceProxy for AT mode
        // return new DataSourceProxy(druidDataSource);

        // DataSourceProxyXA for XA mode
        return new DataSourceProxyXA(druidDataSource);
    }
```

# 4. 总结

在当前的技术发展阶段，不存一个分布式事务处理机制可以完美满足所有场景的需求。

一致性、可靠性、易用性、性能等诸多方面的系统设计约束，需要用不同的事务处理机制去满足。

Seata 项目最核心的价值在于：构建一个全面解决分布式事务问题的 标准化 平台。

基于 Seata，上层应用架构可以根据实际场景的需求，灵活选择合适的分布式事务解决方案。

![img](https://mmbiz.qpic.cn/mmbiz_png/8XkvNnTiapOODXhMq7lSW4fHRRa0EvINCu7AcpdcyMglc3YEUTpZkDxGhicPq7G1G58sZic9wib4Ps2jV5icgrU1OVQ/640?wx_fmt=png)

XA 模式的加入，补齐了 Seata 在 全局一致性 场景下的缺口，形成 AT、TCC、Saga、XA 四大 事务模式 的版图，基本可以满足所有场景的分布式事务处理诉求。

当然 XA 模式和 Seata 项目本身都还不尽完美，有很多需要改进和完善的地方。非常欢迎大家参与到项目的建设中，共同打造一个标准化的分布式事务平台。
