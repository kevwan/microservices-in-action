# go-zero HTTP 超时控制与防 DoS 攻击

## 前言

HTTP 服务的超时控制是微服务稳定性的第一道防线。没有合理的超时控制，一个慢请求可能会耗尽服务器的连接池、goroutine 资源，最终导致整个服务不可用。而 DoS（拒绝服务）攻击则是通过大量恶意请求来耗尽服务器资源的攻击手段。

go-zero 内置了完整的 HTTP 超时控制机制，并通过多层防护手段有效抵御常见的 DoS 攻击场景。本文将深入解析 go-zero 的超时控制设计原理及防 DoS 攻击机制。

## 1. 为什么需要超时控制？

### 1.1 没有超时控制的危害

假设你的 HTTP 服务没有任何超时控制，客户端可以这样攻击你：

```bash
# 发起一个永不完成的请求（只发送请求头不发送请求体）
curl -v --no-buffer -X POST http://your-service/api/upload \
  -H "Content-Type: application/json" \
  -H "Content-Length: 1000000" \
  --data-binary @/dev/zero &
```

服务器会一直等待请求体的到来，而这个连接永远不会完成。如果有足够多这样的连接，服务器的 goroutine 会被耗尽，正常请求将无法得到响应。

### 1.2 典型的超时攻击场景

| 攻击类型 | 描述 | 危害 |
| ------- | ---- | ---- |
| Slowloris 攻击 | 发送不完整的 HTTP 请求头，每隔几秒发送一个字节 | 耗尽服务器连接数 |
| 慢速 POST 攻击 | 声明大 Content-Length 但极慢发送请求体 | 占用 goroutine 和内存 |
| 永久连接攻击 | 建立连接后不发送任何数据 | 耗尽文件描述符 |
| 大请求体攻击 | 发送超大 JSON/表单数据 | 耗尽内存和 CPU |

### 1.3 go-zero 的四层超时防护

```text
客户端请求
    │
    ▼
┌───────────────────────────────┐
│  1. 读取请求头超时               │  (ReadHeaderTimeout)
│     防止 Slowloris 攻击         │
├───────────────────────────────┤
│  2. 读取请求体超时               │  (ReadTimeout)
│     防止慢速 POST 攻击           │
├───────────────────────────────┤
│  3. 写响应超时                  │  (WriteTimeout)
│     防止慢速客户端拖垮服务器       │
├───────────────────────────────┤
│  4. 业务逻辑超时                 │  (Timeout middleware)
│     防止业务处理超时消耗资源       │
└───────────────────────────────┘
```

## 2. go-zero 超时控制的实现

### 2.1 服务器级超时配置

go-zero 的 `RestConf` 提供了完整的超时配置项：

```yaml
Name: my-service
Host: 0.0.0.0
Port: 8888

# 超时配置
Timeout: 3000          # 业务逻辑超时，单位毫秒，默认 3000ms
ReadTimeout: 5000      # 读取请求超时，单位毫秒
WriteTimeout: 5000     # 写入响应超时，单位毫秒

# 请求限制
MaxBytes: 1048576      # 最大请求体大小，单位字节，默认 1MB
```

对应的 Go 配置结构体：

```go
type RestConf struct {
    service.ServiceConf
    Host     string `json:",default=0.0.0.0"`
    Port     int
    CertFile string `json:",optional"`
    KeyFile  string `json:",optional"`
    Verbose  bool   `json:",optional"`
    MaxConns int    `json:",default=10000"`    // 最大并发连接数
    MaxBytes int64  `json:",default=1048576"`  // 最大请求体 1MB
    // 单位毫秒
    Timeout      int64 `json:",default=3000"`
    CpuThreshold int64 `json:",default=900,range=[0:1000]"`
    // 以下用于服务注册
    // ...
}
```

### 2.2 底层 HTTP Server 超时设置

go-zero 在创建 `http.Server` 时会正确设置 `ReadHeaderTimeout`，这是防止 Slowloris 攻击最关键的配置：

```go
// go-zero 内部实现（简化版）
server := &http.Server{
    Addr:              fmt.Sprintf("%s:%d", host, port),
    Handler:           handler,
    ReadTimeout:       time.Duration(c.ReadTimeout) * time.Millisecond,
    WriteTimeout:      time.Duration(c.WriteTimeout) * time.Millisecond,
    ReadHeaderTimeout: 5 * time.Second,  // 固定 5 秒，防止 Slowloris
    IdleTimeout:       60 * time.Second,
}
```

> **为什么 `ReadHeaderTimeout` 很重要？**
>
> Go 标准库的 `ReadTimeout` 从连接建立开始计时，如果设置得过短，在高延迟网络中会导致大量合法请求超时。而 `ReadHeaderTimeout` 只计算读取 HTTP 头的时间，这样可以精确地防御 Slowloris 攻击而不影响合法的慢速连接。

### 2.3 请求体大小限制（MaxBytes 中间件）

go-zero 会自动注入 `MaxBytesHandler` 中间件，限制请求体大小：

```go
// 内部实现原理
func MaxBytesHandler(n int64) func(http.Handler) http.Handler {
    if n <= 0 {
        return func(next http.Handler) http.Handler {
            return next
        }
    }

    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if r.ContentLength > n {
                // 请求体声明大小超限，直接拒绝
                http.Error(w, "Request Entity Too Large", http.StatusRequestEntityTooLarge)
                return
            }
            // 包装 Body，实际读取时也限制大小
            r.Body = http.MaxBytesReader(w, r.Body, n)
            next.ServeHTTP(w, r)
        })
    }
}
```

这样可以防止客户端通过发送超大请求体来耗尽服务器内存。

### 2.4 业务超时中间件

这是 go-zero 超时控制中最精妙的部分。业务超时不仅仅是简单的 context deadline，还需要正确处理超时后的响应：

```go
// go-zero 业务超时中间件的设计思路（简化）
func TimeoutHandler(duration time.Duration) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        if duration <= 0 {
            return next
        }

        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx, cancel := context.WithTimeout(r.Context(), duration)
            defer cancel()

            // 使用 channel 同步请求处理结果
            done := make(chan struct{})
            tw := &timeoutWriter{w: w, h: make(http.Header)}

            go func() {
                defer close(done)
                next.ServeHTTP(tw, r.WithContext(ctx))
            }()

            select {
            case <-done:
                // 请求正常完成，刷写响应
                tw.mu.Lock()
                defer tw.mu.Unlock()
                tw.writeHeader()

            case <-ctx.Done():
                // 超时，返回 503
                tw.mu.Lock()
                defer tw.mu.Unlock()
                if !tw.wroteHeader {
                    w.WriteHeader(http.StatusServiceUnavailable)
                }
            }
        })
    }
}
```

**关键设计要点：**

1. **并发安全**：使用互斥锁保护响应写入，防止超时后业务 goroutine 仍在写响应导致的数据竞争
2. **资源清理**：使用 `defer cancel()` 确保 context 一定被取消，避免 goroutine 泄漏
3. **状态码正确**：超时时返回 503（Service Unavailable）而不是 504（Gateway Timeout），因为这是服务端主动超时

## 3. 防 DoS 攻击实战

### 3.1 连接数限制

仅仅超时控制还不够，go-zero 还提供了最大并发连接数限制（`MaxConns`）。当活跃连接数超过阈值，新连接会被立即拒绝：

```yaml
MaxConns: 10000  # 默认值，根据服务容量调整
```

```go
// 内部使用信号量实现连接数限制
func MaxConns(n int) func(http.Handler) http.Handler {
    if n <= 0 {
        return func(next http.Handler) http.Handler {
            return next
        }
    }

    // 使用 channel 作为计数信号量
    sem := make(chan struct{}, n)

    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            select {
            case sem <- struct{}{}:
                defer func() { <-sem }()
                next.ServeHTTP(w, r)
            default:
                // 超过最大连接数，直接返回 503
                w.WriteHeader(http.StatusServiceUnavailable)
            }
        })
    }
}
```

### 3.2 完整的防 DoS 配置示例

```yaml
Name: my-api
Host: 0.0.0.0
Port: 8888

# 超时配置：防止慢连接和慢业务耗尽资源
Timeout: 3000        # 业务超时 3s，超时返回 503
ReadTimeout: 10000   # 读取请求超时 10s（包含慢网络的宽容）
WriteTimeout: 10000  # 写入响应超时 10s

# 请求限制：防止大请求耗尽内存
MaxBytes: 2097152    # 最大请求体 2MB，根据业务调整

# 连接限制：防止连接耗尽
MaxConns: 10000      # 根据服务器资源调整

# 过载保护：防止 CPU 被打满
CpuThreshold: 900    # CPU 超过 90% 开始过载保护
```

### 3.3 与过载保护联动

超时控制和过载保护（自适应降载）是协同工作的。当系统处于过载状态时：

1. 过载保护先拒绝部分请求（返回 503）
2. 超时控制确保剩余请求能在限定时间内完成
3. 熔断器记录超时请求，防止下游故障蔓延

```go
// 在 server 端，中间件的执行顺序至关重要
// go-zero 的中间件链（从外到内）：
// MaxConns -> MaxBytes -> Shedding(过载保护) -> Breaker -> Timeout -> 业务Handler
```

这个顺序很重要：

- `MaxConns` 最外层，超过连接数直接拒绝，不消耗任何 goroutine
- `MaxBytes` 早期拒绝大请求，避免读取请求体消耗内存
- `Shedding` 在 CPU 超载时拒绝请求，保护 CPU
- `Timeout` 为业务逻辑设置最终期限

## 4. 超时传播：从服务端到客户端

### 4.1 超时上下文的传播

在微服务架构中，超时不应该只在服务端控制，还需要在整个调用链中传播：

```go
// 服务端业务逻辑中，将超时 context 传给下游调用
func (l *OrderLogic) CreateOrder(req *types.OrderRequest) (*types.OrderResponse, error) {
    // l.ctx 已经包含了超时 deadline
    ctx := l.ctx

    // 调用用户服务，超时会自动传播
    user, err := l.svcCtx.UserRpc.GetUser(ctx, &user.GetUserRequest{
        UserId: req.UserId,
    })
    if err != nil {
        // 如果是超时错误，会向上传播
        return nil, err
    }

    // 查询数据库，也受超时控制
    order, err := l.svcCtx.OrderModel.Insert(ctx, &model.Order{
        UserId:    req.UserId,
        ProductId: req.ProductId,
        Amount:    req.Amount,
    })
    if err != nil {
        return nil, err
    }

    return &types.OrderResponse{OrderId: order.LastInsertId()}, nil
}
```

### 4.2 gRPC 超时传播

go-zero 的 gRPC 客户端会自动将 context 的 deadline 通过 gRPC metadata 传递给下游服务：

```go
// go-zero 的 RPC 客户端自动处理超时传播
// 上游 HTTP 请求的超时 deadline 会自动传递给 RPC 调用
// 不需要额外配置，框架自动处理
```

这意味着如果上游 HTTP 请求设置了 3 秒超时，那么整个调用链（包括 RPC 调用、数据库查询等）都必须在 3 秒内完成，否则会统一返回超时错误。

## 5. 最佳实践

### 5.1 超时值的设置原则

| 场景 | 推荐超时 | 原因 |
| ---- | ------- | ---- |
| 普通 CRUD API | 1-3 秒 | 业务逻辑简单 |
| 含第三方调用 | 5-10 秒 | 第三方服务不稳定 |
| 文件上传 | 30-60 秒 | 文件传输需要时间 |
| 报表生成 | 可用异步 | 同步超时无法避免 |
| 数据库查询 | 3-5 秒 | 避免慢查询拖垮服务 |

```yaml
# 针对不同 API 可以通过路由级别单独设置
# 但 go-zero 全局配置对所有路由生效
# 对于特殊路由，可以在业务逻辑中单独控制 context
```

### 5.2 超时错误的处理

```go
func (l *Logic) DoSomething(ctx context.Context) error {
    // 检查 context 是否已超时，避免做无用功
    if err := ctx.Err(); err != nil {
        return err
    }

    // 执行耗时操作
    result, err := l.callExternalService(ctx)
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            // 记录超时日志，便于排查
            logx.WithContext(ctx).Errorf("外部服务调用超时: %v", err)
            return errors.New("服务暂时不可用，请稍后重试")
        }
        return err
    }

    return processResult(ctx, result)
}
```

### 5.3 监控超时率

超时率是衡量服务健康状态的重要指标。go-zero 会自动上报请求指标，可以通过 Prometheus + Grafana 监控：

```yaml
# go-zero 自动上报的 metrics 中包含：
# http_server_requests_duration_ms_bucket - 请求耗时分布
# http_server_requests_total - 请求总数（含状态码）

# 超时请求会以 503 状态码上报
# 可以通过以下 PromQL 计算超时率：
# rate(http_server_requests_total{code="503"}[5m]) / rate(http_server_requests_total[5m])
```

## 6. 总结

go-zero 的超时控制体系是一个多层次、协同配合的防御系统：

1. **`ReadHeaderTimeout`**：防 Slowloris 攻击，默认 5 秒
2. **`ReadTimeout` / `WriteTimeout`**：防慢连接攻击，保护服务器资源
3. **`MaxBytes` 中间件**：防大请求体内存耗尽攻击
4. **`MaxConns` 中间件**：防连接耗尽，快速失败
5. **业务 `Timeout` 中间件**：保证业务处理的时间边界
6. **自适应过载保护**：CPU 过载时主动降级

这六层防护从不同维度保护服务不被 DoS 攻击拖垮，同时通过 context 超时传播，确保整个调用链的超时语义一致。

合理配置这些参数，配合[自适应过载保护](adaptive-shedding-cn.md)和[熔断器](breaker-cn.md)，可以打造出高度稳定的微服务体系。
