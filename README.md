# BrightDB

## 使用展示

### kv 范式

```c#
    var storage = new KVCoherenceStorage(...);
    var ctx = new PlayerContext(1001);
    using(var rec = storage.AcquireAsync("user.User", "1001"))
    {
        ReadOnlyMemory<byte> value = rec.Value;
        var json = JSON.Decode(value);
        Console.WriteLine("name:{0}", json["name"]);
        // 给客户端回复一条消息
        ctx.Notify(new UserInfo() { Name=(string)json["name"], Level=(int)json["level"],});
    }
```

### k-object 范式

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

### distribution transaction k-object 范式

#### 示例1： 从队伍中踢除第一个成员


```c#
    var storage = new KVCoherenceStorage(...);
    var userTable = new DistributedTransactionManager(storage, ...);

    // 对任意一些数据的操作，在任意节点执行，甚至并发执行，都能得到正确的结果。
    // 演示一个队长踢除队伍第一个成员的操作（未仔细检查边界条件，请忽略这些细节）。
    ProcedureUtil.RunTransactionAsync((PlayerTxnContext ctx) =>
    {
        // bright db事务的锁粒度为行（即表中一个记录）
        // 以下访问记录的顺序为 user1, team, user2。
        // 如果基于悲观锁实现的数据库，是有潜在死锁风险的。
        // 而bright db能保证不死锁，并且在激烈冲突的情况下也能2次以内完成事务(优秀!!!!!!)
        db.User user = await db.user.TbUser.GetAsync(1001);
        db.Team team = await db.team.TbTeam.GetAsync(user.TeamId);
        long kickedUserId = team.Members[0].UserId;
        // 队列中移除第一个成员
        team.Members.RemoveAt(0);
        db.User kickedUser = await db.user.TbUser.GetAsync(kickedUserId);
        // 被踢除的角色,设置队伍id=0
        kickedUser.TeamId = 0;
        return true;
    });
```

#### 示例2： 两个节点，同时执行给角色1001的经验+1。最终结果为角色经验+2

```c#
    ProcedureUtil.RunTransactionAsync((PlayerTxnContext ctx) =>
    {
        db.User user = await db.user.TbUser.GetAsync(1001);
        user.Exp += 1;
        return true;
    });
```

## 支持与联系
    
    QQ群：732436871 BrightDB开发群
