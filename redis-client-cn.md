# go-zero Redis 客户端：从基础操作到高级用法

## 前言

Redis 是现代微服务中不可或缺的基础设施：缓存热点数据、实现分布式锁、限流计数、消息发布/订阅。go-zero 内置了对 Redis 的高质量封装，支持单节点、哨兵（主从）、集群三种模式，并对常见操作提供了类型安全的 API。

## 1. 配置与初始化

```yaml
# config.yaml
Redis:
  Host: "127.0.0.1:6379"
  Pass: "password"      # 可为空
  Type: "node"          # node（单节点）| sentinel（哨兵）| cluster（集群）
  # 哨兵模式额外配置
  # Type: "sentinel"
  # MasterName: "mymaster"
  # 连接池（go-zero 自动管理，通常不需要手动配置）
```

```go
// internal/svc/servicecontext.go
import "github.com/zeromicro/go-zero/core/stores/redis"

type ServiceContext struct {
    Config config.Config
    Redis  *redis.Redis
}

func NewServiceContext(c config.Config) *ServiceContext {
    rds := redis.MustNewRedis(c.Redis)
    return &ServiceContext{
        Config: c,
        Redis:  rds,
    }
}
```

## 2. 基础操作

### 2.1 字符串操作

```go
// SET / GET
err := rds.Set("key", "value")
err  = rds.Setex("key", "value", 3600)  // 带过期时间（秒）

val, err := rds.Get("key")
// val == "" 且 err == nil 表示 key 不存在（go-zero 的约定）

// SET NX（如果不存在才设置）—— 用于分布式锁
ok, err := rds.Setnx("key", "value")
ok, err  = rds.SetnxEx("key", "value", 60)  // 带过期时间

// DEL
_, err = rds.Del("key1", "key2")

// TTL / EXPIRE
ttl, err := rds.Ttl("key")
err = rds.Expire("key", 3600)

// 计数器
val, err := rds.Incr("counter")
val, err  = rds.Incrby("counter", 10)
```

### 2.2 哈希操作

```go
// HSET / HGET
err = rds.Hset("user:1", "name", "Alice")
val, err = rds.Hget("user:1", "name")

// HMSET / HMGET
err = rds.Hmset("user:1", map[string]string{
    "name":  "Alice",
    "email": "alice@example.com",
    "age":   "28",
})
vals, err := rds.Hmget("user:1", "name", "email")

// HGETALL
all, err := rds.Hgetall("user:1")
// all: map[string]string{"name":"Alice","email":"alice@example.com","age":"28"}

// HDEL / HEXISTS
_, err = rds.Hdel("user:1", "age")
exists, err := rds.Hexists("user:1", "name")

// HINCRBY
count, err := rds.Hincrby("user:1", "loginCount", 1)
```

### 2.3 列表操作

```go
// LPUSH / RPUSH（头插/尾插）
_, err = rds.Lpush("queue", "item1", "item2")
_, err = rds.Rpush("queue", "item3")

// LPOP / RPOP
val, err = rds.Lpop("queue")
val, err = rds.Rpop("queue")

// LRANGE
vals, err := rds.Lrange("queue", 0, -1)  // 获取全部元素

// LLEN
length, err := rds.Llen("queue")
```

### 2.4 集合操作

```go
// SADD / SREM
_, err = rds.Sadd("tags:product:1", "电子", "手机", "苹果")
_, err = rds.Srem("tags:product:1", "苹果")

// SMEMBERS / SISMEMBER
members, err := rds.Smembers("tags:product:1")
ok, err := rds.Sismember("tags:product:1", "电子")

// SCARD（集合大小）
count, err := rds.Scard("tags:product:1")

// 集合运算：交集、并集、差集
inter, err := rds.Sinter("set1", "set2")
union, err := rds.Sunion("set1", "set2")
diff, err  := rds.Sdiff("set1", "set2")
```

### 2.5 有序集合（Sorted Set）

```go
// ZADD（添加分数成员）
err = rds.Zadd("leaderboard", 1000, "user1")
err = rds.Zadd("leaderboard", 2000, "user2")

// ZRANK / ZSCORE
rank, err  := rds.Zrank("leaderboard", "user1")  // 正序排名（0-based）
score, err := rds.Zscore("leaderboard", "user1")

// ZRANGE（按排名范围获取）
members, err := rds.Zrange("leaderboard", 0, 9)     // 前10名（正序）
members, err  = rds.Zrevrange("leaderboard", 0, 9)  // 前10名（倒序，分数高的在前）

// ZRANGEBYSCORE（按分数范围）
members, err = rds.Zrangebyscore("leaderboard", "1000", "2000")

// ZINCRBY（增加分数）
newScore, err := rds.Zincrby("leaderboard", 500, "user1")

// ZCARD / ZCOUNT
total, err := rds.Zcard("leaderboard")
count, err  = rds.Zcount("leaderboard", "1000", "+inf")
```

## 3. 管道与批量操作

管道可以将多个命令批量发送，减少网络往返次数：

```go
// 使用 Pipeline 批量操作
results, err := rds.Pipelined(func(pipe redis.Pipeliner) error {
    pipe.Set(ctx, "key1", "val1", time.Hour)
    pipe.Set(ctx, "key2", "val2", time.Hour)
    pipe.Incr(ctx, "counter")
    return nil
})
// 一次网络往返完成三个命令
```

## 4. Lua 脚本

Lua 脚本在 Redis 服务端原子执行，解决多命令的原子性问题：

```go
// 原子性地：如果值等于期望值，则删除（用于分布式锁释放）
releaseScript := `
    if redis.call('GET', KEYS[1]) == ARGV[1] then
        return redis.call('DEL', KEYS[1])
    else
        return 0
    end
`
result, err := rds.Eval(releaseScript, []string{"lock:key"}, "unique_value")
released := result.(int64) == 1

// 原子性地：获取计数并判断是否超限（用于限流）
limitScript := `
    local current = redis.call('INCR', KEYS[1])
    if current == 1 then
        redis.call('EXPIRE', KEYS[1], ARGV[1])
    end
    return current
`
count, err := rds.Eval(limitScript, []string{"ratelimit:user:1"}, "60")
```

## 5. Pub/Sub 发布订阅

```go
// 发布消息
err = rds.Publish("order:events", `{"orderId":"123","status":"paid"}`)

// 订阅（通常在 goroutine 中运行）
pubsub := rds.Subscribe(ctx, "order:events")
defer pubsub.Close()

for msg := range pubsub.Channel() {
    logx.Infof("收到消息: %s", msg.Payload)
}
```

## 6. 连接健康检查

```go
// 健康检查端点中使用
func (l *HealthLogic) Health() error {
    if _, err := l.svcCtx.Redis.Ping(); err != nil {
        return fmt.Errorf("Redis 连接异常: %w", err)
    }
    return nil
}
```

## 7. 配置参考

```go
// go-zero Redis 配置字段
type RedisConf struct {
    Host        string        // 地址（单节点或哨兵master）
    Type        string        // node | sentinel | cluster
    Pass        string        // 密码（可选）
    Tls         bool          // 是否启用 TLS
    NonBlock    bool          // 非阻塞连接（避免启动卡住）
    PingTimeout time.Duration // Ping 超时（默认1秒）
    // 哨兵专用
    MasterName  string
}
```

## 8. 总结

go-zero Redis 客户端封装了 Redis 的常用操作，提供了类型安全的 API：

| 数据结构 | 典型用途 |
|---------|---------|
| String | 缓存、计数器、分布式锁、Session |
| Hash | 用户信息、商品属性（对象存储）|
| List | 消息队列、操作历史 |
| Set | 标签系统、去重、好友关系 |
| Sorted Set | 排行榜、延迟队列、优先级队列 |
| Pub/Sub | 实时通知、事件广播 |
| Lua Script | 原子性复合操作（限流、分布式锁）|

相关文章：
- [分布式锁：基于 Redis 的并发控制](distributed-lock-cn.md)
- [限流机制详解：令牌桶与计数器](rate-limit-cn.md)
- [缓存设计最佳实践](cache-design-cn.md)
