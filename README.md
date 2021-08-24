# BrightDB

## 项目介绍

BrightDB 是个极高性能的支持分布式事务的分布式对象数据库（缓存命中情况下性能相比redis高3个数量级以上）。

BrightDB 的设计目标是游戏**系统功能**的解决方案，提供高并发、高可用、分布式事务、实时持久化（意味着服务无状态化）的特性。它易于上手、开发友好、极为可靠，大幅提升了游戏项目的服务器质量。

BrightDB 设计之初就考虑到让它很容易地集成到任何现有的服务器架构体系。它没有特殊的依赖和要求，只要少量的代码，就能良好地集成到项目之中，与现有的框架协同工作。

## 特性
- 提供一个简单易用的对象数据库。支持复杂的对象结构，学习成本低、开发高效、代码简洁。
- 在对象数据库基础上提供分布式事务支持，简化了游戏社交系统功能中常见的分布式系统下不同节点之间数据的一致性的问题。
- 基于乐观锁的无死锁事务，对业务层完全屏蔽锁相关的操作，彻底规避了死锁问题。
- 基于分布式一致性缓存的**极高性能**分布式事务，相比传统的事务延迟大幅降低，同时性能有极大提升，跟**单机内存事务**相近。
- 多种隔离级别及持久化策略。支持捕捉事务的变化日志，持久化到消息队列。以较小的代价实现数据**实时持久化**。
- 事务日志的实时持久化而实现的无状态化，像互联网服务那样很容易扩容缩容，负载均衡。
- 高可用
- 数据与逻辑分离，因此很容易集成自动化测试，减少项目后期回归测试的成本。
- 基于事务变化日志的连续回档，可以将部分甚至全体数据回滚到任意一个时刻点。
- 基于事务的BI分析。由于日志精确包含所有变化，BI更容易分析玩家的所有操作数据。
- 支持服务器状态日志与监控，方便实时了解节点内部状态。
- 支持 c#, java, go 三种主流服务器语言（考虑到安全性，只支持这种内存的语言）
### benchmark

    不单独对缓存测试（缓存命中情况下实在太快了，没有测试必要），只测试事务性能。

    处理器 Intel(R) Core(TM) i5-4460 CPU @3.2G 4核

    内存 8G

    Read 表示事务内读取几个记录，Write表示事务内修改了几个记录，QPS表示每秒成功执行的事务数(单位 万/秒)

| Read | Write |  QPS (w/s) |
| --- | --- | ---- |
| 1  | 0 | 500 |
| 5 | 0 | 115 |
| 20 | 0  | 32 |
| 0  | 1 | 300 |
| 0 | 5 | 71 |
|0 | 20 | 17 |
| 20 | 5 | 19 |

## 使用展示

- kv 范式
```c#
    var storage = new KVCoherenceStorage(...);
    var ctx = new PlayerContext(1001);
    using(var rec = storage.AcquireAsync("user.User", "1001"))
    {
        ReadOnlyMemory<byte> value = rec.Value;
        var json = JSON.Decode(value);
        Console.WriteLine("name:{0}", json["name"]);
        // 给客户端回复一条消息
        ctx.Notify(new UserInfo() { Name=(string)json["name"], Level=(int)json["level"], Exp=(int)json["exp"]});
    }
```

- k-object 范式
```c#
    var storage = new KVCoherenceStorage(...);
    var userTable = new ObjectStorage<long, User>(storage, "user.TbUser");
    var ctx = new PlayerContext(1001);
    using(var rec = userTable.AcquireAsync(1001))
    {
        User user = rec.Value;
        Console.WriteLine("name:{0}", user.Name);
        // 给客户端回复一条消息
        ctx.Notify(new UserInfo() { Name=user.Name, Level=user.Level, Exp=user.Exp});
    }
```

- distribution transaction k-object 范式
```c#
    var storage = new KVCoherenceStorage(...);
    var userTable = new DistributedTransactionManager(storage, ...);

    ProcedureUtil.RunTransactionAsync((PlayerTxnContext ctx) =>
    {
        db.User user = await db.user.TbUser.GetAsync(1001);
        Console.WriteLine("name:{0}", user.Name);
        user.Exp += 5;
        // 给客户端回复一条消息
        ctx.Notify(new UserInfo() { Name=user.Name, Level=user.Level, Exp=user.Exp});
        return true;
    });
```

## 支持与联系
    
    QQ群：732436871 BrightDB开发群