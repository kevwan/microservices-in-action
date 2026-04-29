# go-zero 缓存设计最佳实践：从单机到分布式

## 前言

缓存是提升系统性能最直接有效的手段。然而，缓存的引入也带来了一系列新问题：缓存击穿、缓存穿透、缓存雪崩……任何一个问题处理不当，都可能导致数据库被打垮，系统崩溃。

go-zero 的 `cache` 包和 `sqlc`（带缓存的 SQL 组件）为这些问题提供了通用解法，但生效范围与接入方式仍取决于具体实现（例如是否使用 goctl 生成的 Model）。本文将深度解析 go-zero 的缓存设计思路和最佳实践。

## 1. 三大经典缓存问题

### 1.1 缓存穿透

**问题**：查询一个**不存在的数据**，缓存没有命中，请求穿透到数据库。如果恶意攻击者持续查询不存在的 ID，数据库会被打垮。

```
请求: GET /user/id=99999999（不存在）
缓存: Miss（没有这个 key）
数据库: 查询不到，返回空
缓存: 不缓存（因为没有数据）
下次同样请求: 继续穿透数据库
```

### 1.2 缓存击穿

**问题**：某个**热点 key** 的缓存在高并发时失效，大量请求同时穿透到数据库，产生巨大冲击。

```
T0: key "user:123" 缓存过期
T0: 1000 个并发请求同时发现缓存 Miss
T0: 1000 个请求同时查询数据库 SELECT * FROM users WHERE id=123
数据库：CPU 飙升，其他请求被阻塞
```

### 1.3 缓存雪崩

**问题**：大量缓存**同时过期**，导致短时间内数据库承受大量请求。

```
23:00:00 批量缓存同时过期（因为设置了相同的 TTL）
23:00:00 大量缓存 Miss，请求涌入数据库
数据库：短时间内被大量请求淹没
```

## 2. go-zero 的解决方案

### 2.1 针对缓存穿透：缓存空值

go-zero 在数据库查询返回空时，仍然缓存一个空标记：

```go
// go-zero sqlc 内部处理（简化版）
func (c *CachedConn) QueryRowCtx(ctx context.Context, v any, key string, query QueryCtxFn) error {
    return c.cache.TakeCtx(ctx, v, key, func(v any) error {
        err := query(ctx, v)
        if err == sql.ErrNoRows {
            // 数据不存在，缓存一个空标记（短 TTL）
            // 防止穿透
            return c.cache.SetWithExpireCtx(ctx, key, "*",
                time.Minute)  // 空标记只缓存 1 分钟
        }
        return err
    })
}
```

当再次查询同一个不存在的 ID 时，会命中缓存（空标记），直接返回空，不会穿透到数据库。

### 2.2 针对缓存击穿：singleflight

go-zero 使用 `singleflight` 模式确保同一个 key 在同一时刻只有一个请求穿透到数据库：

```go
// go-zero cache 内部使用 singleflight（简化版）
func (c *Cache) TakeCtx(ctx context.Context, v any, key string, query func(v any) error) error {
    val, err, _ := c.barrier.Do(key, func() (any, error) {
        // 1. 先查缓存
        if err := c.GetCtx(ctx, v, key); err == nil {
            return v, nil
        }

        // 2. 缓存 Miss，只有一个请求会到达这里（其他相同 key 的请求在等待）
        if err := query(v); err != nil {
            return nil, err
        }

        // 3. 回填缓存
        if err := c.SetCtx(ctx, key, v); err != nil {
            logx.Errorf("缓存写入失败: %v", err)
        }

        return v, nil
    })
    // singleflight 的所有等待者共享这一个结果
    // 无论有多少并发请求，只有 1 次数据库查询
    ...
}
```

即使 1000 个请求同时访问同一个失效的 key，也只有 1 个请求会查询数据库，其余 999 个等待结果。

### 2.3 针对缓存雪崩：TTL 随机抖动

go-zero 在设置缓存时，会在 TTL 上添加随机抖动，防止大量 key 同时过期：

```go
// go-zero 缓存 TTL 设置（简化版）
const (
    cacheSafeGapBetweenIndexAndPrimary = time.Second * 5
    // 默认缓存时间
    defaultCacheTTL = time.Hour
)

func (c *Cache) SetCtx(ctx context.Context, key string, val any) error {
    // TTL = 基础时间 + 随机偏移（±10%）
    // 例如基础 TTL=1h，则实际 TTL 在 54min~66min 之间随机
    ttl := defaultCacheTTL + time.Duration(rand.Int63n(int64(defaultCacheTTL/10)))
    return c.SetWithExpireCtx(ctx, key, val, ttl)
}
```

通过随机 TTL，大量 key 不会在同一时刻集中过期，避免雪崩。

## 3. go-zero 缓存组件使用

### 3.1 goctl 自动生成带缓存的 Model

go-zero 提供了 `goctl` 工具，可以自动生成带缓存的数据库操作代码：

```bash
# 根据 SQL 文件自动生成带 Redis 缓存的 Model
goctl model mysql ddl --src user.sql --dir ./internal/model -c

# -c 参数表示生成带缓存的 Model
```

生成的代码会自动处理缓存的读写和失效：

```go
// 自动生成的 user_model_gen.go（简化版）
package model

type defaultUserModel struct {
    sqlc.CachedConn  // 包含缓存功能的 SQL 连接
    table string
}

// 查询：先查缓存，缓存 Miss 则查数据库并回填缓存
func (m *defaultUserModel) FindOne(ctx context.Context, id int64) (*User, error) {
    userIdKey := fmt.Sprintf("%s%v", cacheUserIdPrefix, id)
    var resp User
    err := m.QueryRowCtx(ctx, &resp, userIdKey, func(ctx context.Context, conn sqlx.SqlConn, v any) error {
        query := fmt.Sprintf("select %s from %s where `id` = ? limit 1", userRows, m.table)
        return conn.QueryRowCtx(ctx, v, query, id)
    })
    return &resp, err
}

// 更新：更新数据库，并删除对应缓存（让缓存自然失效，下次读取时重建）
func (m *defaultUserModel) Update(ctx context.Context, data *User) error {
    userIdKey := fmt.Sprintf("%s%v", cacheUserIdPrefix, data.Id)
    _, err := m.ExecCtx(ctx, func(ctx context.Context, conn sqlx.SqlConn) (sql.Result, error) {
        query := fmt.Sprintf("update %s set %s where `id` = ?", m.table, userRowsWithPlaceHolder)
        return conn.ExecCtx(ctx, query, data.Nickname, data.Mobile, data.Id)
    }, userIdKey)  // 更新完成后自动删除这个 key 的缓存
    return err
}
```

### 3.2 直接使用 Cache 组件

```go
package svc

import (
    "context"
    "encoding/json"
    "time"

    "github.com/zeromicro/go-zero/core/stores/cache"
    "github.com/zeromicro/go-zero/core/stores/redis"
)

type ProductCache struct {
    cache cache.Cache
}

func NewProductCache(rds *redis.Redis) *ProductCache {
    // 创建缓存组件
    c := cache.New(
        []cache.NodeConf{
            {
                RedisConf: redis.RedisConf{
                    Host: "localhost:6379",
                    Type: "node",
                },
                Weight: 100,  // 一致性哈希权重
            },
        },
        nil,        // 统计对象（可选）
        cache.WithExpiry(time.Hour),          // 默认 TTL
        cache.WithNotFoundExpiry(time.Minute), // 空值 TTL（防穿透）
    )
    return &ProductCache{cache: c}
}

func (p *ProductCache) GetProduct(ctx context.Context, productId int64) (*Product, error) {
    key := fmt.Sprintf("product:%d", productId)
    var product Product

    err := p.cache.TakeCtx(ctx, &product, key, func(v any) error {
        // 缓存 Miss 时从数据库加载
        return loadProductFromDB(ctx, productId, v.(*Product))
    })

    return &product, err
}

func (p *ProductCache) InvalidateProduct(ctx context.Context, productId int64) error {
    key := fmt.Sprintf("product:%d", productId)
    return p.cache.DelCtx(ctx, key)
}
```

## 4. 多级缓存架构

对于极高读取频率的数据，可以构建多级缓存：本地内存缓存 + Redis 缓存：

### 4.1 使用 go-zero 的 collection 包

```go
import (
    "github.com/zeromicro/go-zero/core/collection"
    "time"
)

type TwoLevelCache struct {
    // 一级缓存：本地内存（LRU）
    localCache *collection.Cache
    // 二级缓存：Redis
    redisCache cache.Cache
}

func NewTwoLevelCache(rds *redis.Redis) *TwoLevelCache {
    // 本地 LRU 缓存：最多 1000 条，每条过期时间 30 秒
    local, _ := collection.NewCache(30*time.Second, collection.WithLimit(1000))

    return &TwoLevelCache{
        localCache: local,
        redisCache: cache.New(/* redis 配置 */),
    }
}

func (c *TwoLevelCache) Get(ctx context.Context, key string, dest any) error {
    // 1. 先查本地内存缓存
    if val, ok := c.localCache.Get(key); ok {
        // 本地命中，直接返回（无网络开销）
        return copyValue(val, dest)
    }

    // 2. 查 Redis 缓存
    err := c.redisCache.TakeCtx(ctx, dest, key, func(v any) error {
        // Redis Miss，查数据库
        return loadFromDB(ctx, key, v)
    })

    if err == nil {
        // Redis 命中，回填本地缓存
        c.localCache.Set(key, deepCopy(dest))
    }

    return err
}
```

### 4.2 多级缓存的一致性处理

多级缓存面临的最大挑战是数据更新时的一致性：

```go
func (c *TwoLevelCache) Invalidate(ctx context.Context, key string) error {
    // 1. 删除本地缓存（立即生效）
    c.localCache.Del(key)

    // 2. 删除 Redis 缓存
    if err := c.redisCache.DelCtx(ctx, key); err != nil {
        return err
    }

    // 3. 如果有多个服务实例，需要通过消息广播通知其他实例清除本地缓存
    // 例如通过 Redis Pub/Sub 或 消息队列
    return c.publisher.Publish(ctx, "cache:invalidate", key)
}

// 订阅缓存失效事件
func (c *TwoLevelCache) StartInvalidateListener(ctx context.Context) {
    c.subscriber.Subscribe(ctx, "cache:invalidate", func(key string) {
        c.localCache.Del(key)
    })
}
```

## 5. 缓存 Key 设计规范

### 5.1 Key 命名规范

```go
// 推荐的 Key 格式：{服务}:{资源}:{id}
const (
    // 用户信息
    cacheUserKey = "user:info:%d"        // user:info:123
    // 用户会话
    cacheUserSessionKey = "user:session:%s"  // user:session:abc123
    // 商品信息
    cacheProductKey = "product:detail:%d"    // product:detail:456
    // 商品库存
    cacheStockKey = "product:stock:%d"       // product:stock:456
    // 排行榜（带版本，方便清理）
    cacheRankKey = "rank:v2:%s:%d"           // rank:v2:daily:20240101
)

func getCacheUserKey(userId int64) string {
    return fmt.Sprintf(cacheUserKey, userId)
}
```

### 5.2 避免大 Key

```go
// ❌ 不好的做法：将大列表存入单个 key
func getBadUserList(ctx context.Context) ([]User, error) {
    key := "user:all_list"  // 可能包含几百万用户！
    var users []User
    cache.TakeCtx(ctx, &users, key, ...)
    return users, nil
}

// ✅ 好的做法：分页或分片存储
func getGoodUserList(ctx context.Context, page, pageSize int) ([]User, error) {
    key := fmt.Sprintf("user:list:page:%d:size:%d", page, pageSize)
    var users []User
    cache.TakeCtx(ctx, &users, key, ...)
    return users, nil
}
```

## 6. 缓存监控

go-zero 的缓存组件内置了统计功能，可以监控命中率：

```go
// 创建带统计的缓存
stat := cache.NewStat("product-cache")

c := cache.New(
    nodeConfs,
    stat,  // 传入统计对象
    cache.WithExpiry(time.Hour),
)

// stat 会自动记录：
// - 总请求数
// - 缓存命中数
// - 缓存 Miss 数
// - 命中率

// go-zero 会自动定期打印统计信息到日志：
// {"level":"stat","ts":"2024-01-01T12:00:00Z",
//  "caller":"cache/stat.go:42",
//  "stat":"product-cache","total":10000,"hit":9500,"miss":500,"hitRatio":"95.00%"}
```

## 7. 最佳实践总结

| 问题 | go-zero 解决方案 | 配置方式 |
|------|-----------------|---------|
| 缓存穿透 | 缓存空值（短 TTL） | `WithNotFoundExpiry` |
| 缓存击穿 | singleflight 合并请求 | 内置，无需配置 |
| 缓存雪崩 | TTL 随机抖动 | 内置，无需配置 |
| 数据一致性 | 删除缓存（而非更新） | goctl 自动生成 |
| 大 key | 分页/分片存储 | 业务层控制 key 设计 |
| 命中率监控 | 内置 Stat 统计 | `cache.NewStat()` |

**核心原则**：
1. **Cache Aside 模式**：读时先查缓存，Miss 则查数据库并回填；写时先写数据库，再删缓存
2. **删除而非更新**：缓存更新时删除 key，而不是直接更新缓存值（避免并发写带来的不一致）
3. **使用 goctl 生成**：让框架自动处理缓存逻辑，减少手动编写缓存代码的错误
4. **监控命中率**：命中率 < 90% 时需要排查是否存在缓存设计问题

通过 go-zero 的缓存组件，可以快速构建具备生产级稳定性的缓存层，从框架层面规避缓存三大经典问题，让开发者专注于业务逻辑。
