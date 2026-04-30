# go-zero 分布式锁：基于 Redis 的并发控制

## 前言

在分布式系统中，多个服务实例可能同时处理同一数据。比如多个订单服务实例同时扣减同一商品的库存，如果没有并发控制，就会出现超卖问题。**分布式锁**是解决这类问题的核心工具，go-zero 内置了基于 Redis 的分布式锁实现。

## 1. go-zero 内置分布式锁

go-zero 在 `core/stores/redis` 包中提供了 `RedisLock`：

```go
import "github.com/zeromicro/go-zero/core/stores/redis"

// 创建锁（指定 Redis 客户端和锁 key）
lock := redis.NewRedisLock(rds, "order:stock:12345")

// 设置锁过期时间（防止程序崩溃导致死锁）
lock.SetExpire(10) // 10 秒后自动释放

// 尝试获取锁
acquired, err := lock.Acquire()
if err != nil {
    return err
}
if !acquired {
    // 锁已被其他实例持有
    return errors.New("操作太频繁，请稍后重试")
}

// 确保释放锁（即使出现 panic）
defer lock.Release()

// 执行临界区代码
// ...
```

## 2. 完整的库存扣减示例

```go
// internal/logic/order/createorderlogic.go
func (l *CreateOrderLogic) deductStock(productId int64, quantity int) error {
    // 1. 创建分布式锁（以商品 ID 为 key，保证同一商品串行扣减）
    lockKey := fmt.Sprintf("stock:lock:%d", productId)
    lock := redis.NewRedisLock(l.svcCtx.Redis, lockKey)
    lock.SetExpire(5) // 5 秒超时，防止死锁

    // 2. 获取锁（非阻塞，立即返回）
    acquired, err := lock.Acquire()
    if err != nil {
        return fmt.Errorf("获取分布式锁失败: %w", err)
    }
    if !acquired {
        return errorx.New(errorx.CodeConcurrentConflict, "商品正在处理中，请稍后重试")
    }
    defer lock.Release()

    // 3. 查询最新库存（锁内读取，避免读到脏数据）
    product, err := l.svcCtx.ProductModel.FindOne(l.ctx, productId)
    if err != nil {
        return err
    }
    if product.Stock < int64(quantity) {
        return errorx.New(errorx.CodeProductSoldOut, "库存不足")
    }

    // 4. 扣减库存
    return l.svcCtx.ProductModel.DeductStock(l.ctx, productId, int64(quantity))
}
```

## 3. 带重试的分布式锁

当业务允许等待一段时间时，可以实现带重试的锁：

```go
// 带超时的锁获取（最多等待 3 秒）
func acquireWithRetry(lock *redis.RedisLock, timeout time.Duration) (bool, error) {
    deadline := time.Now().Add(timeout)
    interval := 50 * time.Millisecond  // 每 50ms 重试一次

    for time.Now().Before(deadline) {
        acquired, err := lock.Acquire()
        if err != nil {
            return false, err
        }
        if acquired {
            return true, nil
        }
        // 未获取到锁，等待后重试
        time.Sleep(interval)
        // 指数退避：逐渐增加等待时间
        if interval < 500*time.Millisecond {
            interval = interval * 2
        }
    }
    return false, nil  // 超时未获取到锁
}
```

## 4. 防重复提交（幂等锁）

分布式锁还可以用于防止接口重复提交：

```go
func (l *CreateOrderLogic) CreateOrder(req *types.CreateOrderReq) (*types.CreateOrderResp, error) {
    userId := l.ctx.Value("userId").(int64)

    // 以用户ID + 请求特征 作为幂等键
    // 防止同一用户在 5 秒内重复提交相同订单
    idempotentKey := fmt.Sprintf("idempotent:create_order:%d:%d:%d",
        userId, req.ProductId, req.Quantity)

    lock := redis.NewRedisLock(l.svcCtx.Redis, idempotentKey)
    lock.SetExpire(5) // 5 秒内不允许重复提交

    acquired, err := lock.Acquire()
    if err != nil {
        // fail-open：Redis 故障时放行
        logx.WithContext(l.ctx).Errorf("幂等锁检查失败: %v", err)
    } else if !acquired {
        return nil, errorx.New(errorx.CodeDuplicateRequest, "请勿重复提交，请稍后再试")
    }
    // 注意：幂等锁不 release，让它自然过期（5秒内锁住重复请求）

    // 继续创建订单...
    return l.doCreateOrder(req)
}
```

## 5. 超时问题与防护

### 5.1 锁过期时间选择

锁过期时间需要根据业务逻辑的最长执行时间来设置：

```
锁过期时间 = 业务最长执行时间 × 安全系数（建议 2-3 倍）

示例：
- 库存扣减（纯内存 + DB）：业务约 100ms → 锁过期 3 秒
- 扣减库存 + 第三方支付：业务约 3 秒 → 锁过期 15 秒
- 生成账单报表：业务约 30 秒 → 考虑用任务队列替代分布式锁
```

### 5.2 锁续期（Watchdog）

对于执行时间不确定的任务，可以实现自动续期：

```go
// 启动 watchdog goroutine，定期续期
func acquireWithWatchdog(ctx context.Context, lock *redis.RedisLock, expire int) (context.CancelFunc, error) {
    acquired, err := lock.Acquire()
    if err != nil || !acquired {
        return nil, err
    }

    // 每 expire/3 秒续期一次
    renewCtx, cancel := context.WithCancel(ctx)
    go func() {
        ticker := time.NewTicker(time.Duration(expire/3) * time.Second)
        defer ticker.Stop()
        for {
            select {
            case <-ticker.C:
                // 续期（重新设置过期时间）
                if err := lock.(*redis.RedisLock).Expire(expire); err != nil {
                    logx.Errorf("锁续期失败: %v", err)
                    cancel()
                    return
                }
            case <-renewCtx.Done():
                return
            }
        }
    }()

    return cancel, nil
}
```

## 6. 分布式锁 vs 数据库乐观锁

| 方案 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| Redis 分布式锁 | 高并发短事务 | 性能好，毫秒级 | 依赖 Redis 可用性 |
| 数据库乐观锁（version） | 低并发，读多写少 | 无外部依赖 | 高并发下重试多 |
| 数据库悲观锁（FOR UPDATE） | 强一致性要求 | 简单可靠 | 长事务影响吞吐 |
| 队列串行化 | 超高并发抢购 | 无锁，吞吐最高 | 实现复杂 |

乐观锁示例（对比用）：

```sql
-- 带 version 的库存扣减
UPDATE product
SET stock = stock - ?, version = version + 1
WHERE id = ? AND stock >= ? AND version = ?
```

```go
// Go 代码中检查影响行数
result, err := db.ExecContext(ctx, updateSQL, quantity, productId, quantity, currentVersion)
affected, _ := result.RowsAffected()
if affected == 0 {
    // version 不匹配，说明被其他实例修改，需要重试
    return errors.New("并发冲突，请重试")
}
```

## 7. 总结

go-zero 的 `RedisLock` 实现了 Redis 分布式锁的核心语义：

- 原子性：使用 `SET key value NX PX expire` 原子操作
- 安全释放：只删除自己持有的锁（通过 Lua 脚本保证原子性）
- 防死锁：强制设置过期时间

**最佳实践**：
1. 锁粒度要细（按业务对象 ID 加锁，而非全局锁）
2. 过期时间要合理（业务时间的 2-3 倍）
3. 必须 defer release（防止忘记释放）
4. Redis 故障时 fail-open（根据业务选择）
5. 高并发场景（如秒杀）考虑用消息队列替代分布式锁
