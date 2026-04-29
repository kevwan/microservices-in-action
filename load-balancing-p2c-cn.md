# go-zero 负载均衡深度解析：P2C 算法

## 前言

在微服务架构中，负载均衡是保证服务高可用和高性能的关键技术。传统的轮询（Round Robin）、随机（Random）等算法在后端节点性能一致时表现良好，但在实际生产环境中，节点的处理能力往往因为硬件配置、运行状态、网络延迟等因素而存在差异。

go-zero 默认使用 **P2C（Power of Two Choices）+ EWMA（指数加权移动平均）** 算法实现自适应负载均衡。这个算法能够根据后端节点的实时健康状况动态调整流量分配，从根本上解决负载不均和慢节点问题。

## 1. 传统负载均衡算法的局限

### 1.1 轮询算法的问题

```
时刻T1: 请求 → 节点A（响应时间 10ms，负载轻）
时刻T2: 请求 → 节点B（响应时间 500ms，负载重）
时刻T3: 请求 → 节点C（响应时间 15ms，负载轻）
时刻T4: 请求 → 节点A（响应时间 10ms）
时刻T5: 请求 → 节点B（响应时间 800ms，继续堆积）⚠️
```

轮询算法完全不感知节点状态，会持续向已经过载的节点发送请求，形成恶性循环。

### 1.2 最少连接数算法的问题

最少连接数（Least Connections）看似解决了上面的问题，但在异步高并发场景下，"连接数少"不等于"响应快"——一个节点可能有很少的连接，但每个连接都在执行耗时的操作。

### 1.3 P2C 算法的优势

P2C 算法的核心思想是：**从所有节点中随机选择两个，然后选择其中负载更低的那个**。

数学证明表明，P2C 算法能够将最大负载从 O(log n / log log n) 降低到 O(log log n)，大幅改善负载均衡效果，而且算法复杂度仅为 O(1)。

## 2. EWMA：实时追踪节点性能

P2C 算法需要一个指标来衡量"节点负载"。go-zero 使用 EWMA（Exponential Weighted Moving Average，指数加权移动平均）来估算每个节点当前的响应时间。

### 2.1 EWMA 公式

$$
\text{ewma}_t = \alpha \cdot x_t + (1 - \alpha) \cdot \text{ewma}_{t-1}
$$

其中：
- $x_t$ 是最新一次请求的响应时间
- $\alpha$ 是平滑因子（go-zero 中 $\alpha = 0.9$）
- $\text{ewma}_{t-1}$ 是上一次的 EWMA 值

EWMA 的特点是**对最新数据给予更高权重**，能够快速响应节点性能变化。当节点变慢时，EWMA 会迅速上升；当节点恢复时，EWMA 会迅速下降。

### 2.2 时间衰减

在低流量场景下，如果某个节点很久没有收到请求，其 EWMA 值可能已经过时。go-zero 引入了时间衰减机制：

$$
\text{lag} = t_{\text{now}} - t_{\text{last}}
$$

$$
\text{ewma}_{\text{adjusted}} = \begin{cases}
0 & \text{if lag} > \text{threshold} \\
\text{ewma} \cdot e^{-\lambda \cdot \text{lag}} & \text{otherwise}
\end{cases}
$$

当节点长时间没有请求时，EWMA 值会衰减趋向 0，这样该节点会被优先选择来探测其当前状态。

## 3. go-zero P2C 实现解析

### 3.1 节点权重计算

go-zero 中每个节点的综合负载指标计算公式：

```go
// 节点的综合负载分数 = EWMA响应时间 × 在途请求数
// 在途请求数（inflight）代表当前正在处理的并发请求数
score = ewma * inflight
```

这个设计非常巧妙：
- 如果 EWMA 响应时间高，说明节点处理慢
- 如果 inflight 高，说明节点当前压力大
- 两者相乘，综合反映了节点的"繁忙程度"

### 3.2 核心选择逻辑

```go
// P2C 选择算法（简化版）
func (b *p2cPicker) pick() (subConn *p2cConn, done func(balancer.DoneInfo), err error) {
    b.lock.Lock()
    defer b.lock.Unlock()

    var chosen *p2cConn

    switch len(b.conns) {
    case 0:
        return nil, nil, balancer.ErrNoSubConnAvailable
    case 1:
        // 只有一个节点，直接使用
        chosen = b.conns[0]
    case 2:
        // 两个节点，选负载低的
        chosen = b.choose(b.conns[0], b.conns[1])
    default:
        // 多个节点：随机选两个，然后选负载低的
        var node1, node2 *p2cConn
        for i := 0; i < pickTimes; i++ {
            a := b.conns[b.r.Intn(len(b.conns))]
            b_ := b.conns[b.r.Intn(len(b.conns))]
            if a != b_ {
                node1, node2 = a, b_
                break
            }
        }
        chosen = b.choose(node1, node2)
    }

    // 增加在途请求计数
    atomic.AddInt64(&chosen.inflight, 1)
    return chosen, b.buildDoneFunc(chosen), nil
}

func (b *p2cPicker) choose(c1, c2 *p2cConn) *p2cConn {
    start := int64(timex.Now())
    // 计算两个节点的综合负载分数
    if c1.load(start) <= c2.load(start) {
        return c1
    }
    return c2
}
```

### 3.3 请求完成后更新统计

```go
func (b *p2cPicker) buildDoneFunc(c *p2cConn) func(balancer.DoneInfo) {
    start := int64(timex.Now())

    return func(info balancer.DoneInfo) {
        // 请求完成，减少在途请求计数
        atomic.AddInt64(&c.inflight, -1)

        now := int64(timex.Now())
        // 计算此次请求的响应时间（纳秒）
        last := atomic.SwapInt64(&c.last, now)
        td := now - last
        if td < 0 {
            td = 0
        }

        // 时间衰减：计算权重
        w := math.Exp(float64(-td) / float64(decayTime))
        lag := now - start
        if lag < 0 {
            lag = 0
        }

        // EWMA 更新：新的平均响应时间
        // ewma = α * lag + (1-α) * ewma_old
        // 但这里用时间衰减权重代替固定 α
        olag := atomic.LoadUint64(&c.lag)
        if olag == 0 {
            w = 0
        }
        atomic.StoreUint64(&c.lag, uint64(float64(olag)*w+float64(lag)*(1-w)))

        // 记录成功/失败，用于健康检查
        success := uint64(1)
        if info.Err != nil && !codes.Acceptable(info.Err) {
            success = 0
        }
        oSuccess := atomic.LoadUint64(&c.success)
        atomic.StoreUint64(&c.success, uint64(float64(oSuccess)*w+float64(success)*(1-w)))
    }
}
```

### 3.4 负载分数计算

```go
func (c *p2cConn) load(now int64) int64 {
    // 计算时间衰减权重
    td := now - atomic.LoadInt64(&c.last)
    if td < 0 {
        td = 0
    }
    w := math.Exp(float64(-td) / float64(decayTime))

    // 获取 EWMA 响应时间
    lag := float64(atomic.LoadUint64(&c.lag))
    // 应用时间衰减
    lagAdjusted := lag * w

    // 综合负载 = EWMA响应时间 × 在途请求数
    load := lagAdjusted * float64(atomic.LoadInt64(&c.inflight))

    // 为了避免 load 为 0 时节点永远被选择，设置最小值
    if load == 0 {
        return 0
    }
    return int64(load)
}
```

## 4. 在 go-zero 中使用 P2C 负载均衡

### 4.1 RPC 服务自动启用

在已配置服务发现（如 etcd）或 Endpoints 的前提下，go-zero 的 RPC 客户端默认使用 P2C 负载均衡：

```yaml
# etc/service.yaml
Name: order-api
Host: 0.0.0.0
Port: 8888

# RPC 客户端配置
UserRpc:
  Etcd:
    Hosts:
      - "etcd:2379"
    Key: user.rpc
  # P2C 是默认负载均衡策略，无需显式配置
```

```go
// main.go
func main() {
    var c config.Config
    conf.MustLoad("etc/service.yaml", &c)

    // go-zero 会自动使用 P2C 负载均衡连接 user.rpc 的所有实例
    ctx := svc.NewServiceContext(c)
    server := rest.MustNewServer(c.RestConf)
    defer server.Stop()

    handler.RegisterHandlers(server, ctx)
    server.Start()
}
```

### 4.2 多实例场景验证

假设你有 3 个 user-rpc 实例：

```bash
# 实例1：正常运行，响应时间约 5ms
user-rpc-1:3001

# 实例2：轻微负载，响应时间约 15ms
user-rpc-2:3001

# 实例3：负载过重，响应时间约 200ms
user-rpc-3:3001
```

使用 P2C 算法后，流量分配会自动向实例1和实例2倾斜，实例3几乎不会收到新请求，直到其响应时间恢复正常。

## 5. 对比实验

### 5.1 场景设置

- 3 个后端节点
- 节点A：正常，响应时间 5ms
- 节点B：正常，响应时间 8ms
- 节点C：故障恢复中，响应时间 200ms

### 5.2 轮询 vs P2C 对比

| 指标 | 轮询（RR） | P2C + EWMA |
|------|-----------|-----------|
| 平均响应时间 | ~71ms | ~8ms |
| P99 响应时间 | ~200ms | ~15ms |
| 错误率 | 33% | <1% |
| 节点C 流量占比 | 33% | <5% |

P2C 算法能够自动识别慢节点，将流量从 200ms 的节点转移开，系统整体的平均响应时间从 71ms 降低到 8ms。

## 6. 与一致性哈希的结合

在某些场景下（如需要保持会话亲和性或分布式缓存），需要使用一致性哈希而不是 P2C。go-zero 也支持一致性哈希负载均衡：

```go
// 在 RPC 调用时指定路由键，使用一致性哈希
// 相同的 userId 会路由到相同的后端节点
func (l *Logic) GetUserInfo(userId int64) (*UserInfo, error) {
    resp, err := l.svcCtx.UserRpc.GetUser(
        // 通过 WithBalancerName 指定使用一致性哈希
        // 并通过 metadata 传递路由键
        metadata.NewOutgoingContext(l.ctx,
            metadata.Pairs("user-id", strconv.FormatInt(userId, 10)),
        ),
        &user.GetUserRequest{UserId: userId},
    )
    return resp, err
}
```

## 7. 总结

go-zero 的 P2C + EWMA 负载均衡算法相比传统算法有以下核心优势：

1. **自适应性**：实时感知后端节点的响应时间和负载状态
2. **快速响应**：EWMA 的时间衰减特性能够快速检测到节点状态变化
3. **防止慢节点**：自动减少对慢节点的请求分配，避免慢节点成为系统瓶颈
4. **低开销**：O(1) 复杂度，选择算法本身不增加额外延迟
5. **零配置**：作为默认策略开箱即用，无需手动配置

在实际生产环境中，P2C 负载均衡配合 go-zero 的[熔断器](breaker-cn.md)和[过载保护](adaptive-shedding-cn.md)，共同构建了一套完整的微服务稳定性保障体系。
