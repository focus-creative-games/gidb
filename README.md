# GiDB

核心设计目标为**游戏系统功能高性能无状态化**的解决方案，实现了接近本地内存事务的性能，性能数据超出常规的 `无状态服务器+redis` 一个量级以上。

## 特性

- 提供一个简单易用的对象数据库接口
- 实现了高性能的CRUD分布式事务，无竞争情况下性能与本地内存事务接近。分布式事务解决了大多数游戏分布式系列下的数据一致性痛点
- 基于乐观锁的无死锁事务机制，深度改良了重做机制，恶劣竞争情况下也能2次内完成事务，避免饥饿及激烈冲突造成的大幅性能下降
- 实时增量持久化，玩家所家即所存，不再有数据丢失
- 高并发、高可用、高可靠

## 使用展示

### 两个节点，同时执行给角色1001的经验+1。最终结果为角色经验+2

```c#
    async Task<bool> Handle_AddPlayerExpAsync(PlayerTxnContext ctx)
    {
        db.User user = await db.user.TbUser.GetAsync(1001);
        user.Exp += 1;
        return true;
    });
```


### 角色1001 送100元宝给角色1002

```c#
    async Task<bool> Handle_TranferMoneyAsync(PlayerTxnContext ctx)
    {
        db.User user1 = await db.user.TbUser.GetAsync(1001);
        user1.Gold -= 100;
        db.User user2 = await db.user.TbUser.GetAsync(1002);
        user2.Gold += 100;
        return true;
    }
```

### 角色1002与1002结成情侣，消耗角色1001 100元宝


```c#
    async Task<bool> Handle_EngageAsync(PlayerTxnContext ctx)
    {
        db.User user1 = await db.user.TbUser.GetAsync(1001);
        if (user1.Gold < 100)
        {
            ctx.ResponseError(ErrorCode.GOLD_NOT_ENOUGH);
            return false;
        }
        // 注意，虽然此处先扣了钱，如果后续检查符合条件，导致事务失败，则会自动回滚。
        // 很多情况下有极大的便利性，减轻程序员心智负担。
        user1.Gold -= 100;
        
        db.Lover lover1 = await db.sns.TbLover.GetAsync(1001);
        if (lover1.LoverId != 0)
        {
            ctx.ResponseError(ErrorCode.YOU_HAVE_LOVER);
            return false;
        }
        db.Lover lover2 = await db.sns.TbLover.GetAsync(1002);
        if (lover2.LoverId != 0)
        {            
            ctx.ResponseError(ErrorCode.PEER_HAVE_LOVER);
            return false;
        }
        lover1.LoverId = 1002;
        lover2.LoverId = 1001;
        return true;
    }
```

### 从队伍中踢除第一个成员

```c#
    // 对任意一些数据的操作，在任意节点执行，甚至并发执行，都能得到正确的结果。
    // 演示一个队长踢除队伍第一个成员的操作（未仔细检查边界条件，请忽略这些细节）。
    async Task<bool> Handle_KickTeamMemberAsync(PlayerTxnContext ctx)
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
    }
```



## 支持与联系
    
    QQ群：732436871 GiDB开发群
