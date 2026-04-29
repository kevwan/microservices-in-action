# go-zero 分布式链路追踪：OpenTelemetry 实战

## 前言

在微服务架构中，一个用户请求往往需要经过多个服务的协同处理。当出现性能问题或错误时，如何快速定位问题出在哪个服务、哪个环节，成为运维和开发的最大挑战。

分布式链路追踪（Distributed Tracing）正是为解决这个问题而生的。go-zero 基于 **OpenTelemetry** 标准提供链路追踪能力：配置 `Telemetry` 后可自动追踪 HTTP 请求、gRPC 调用、数据库操作等，并在追踪后端可视化调用链路。

## 1. 链路追踪核心概念

### 1.1 Trace、Span 和 Context

理解链路追踪，首先需要理解三个核心概念：

```
Trace（调用链）
└── Span A: HTTP /api/order (duration: 120ms)
    ├── Span B: gRPC UserService.GetUser (duration: 30ms)
    │   └── Span D: MySQL SELECT users (duration: 5ms)
    ├── Span C: gRPC InventoryService.CheckStock (duration: 80ms)
    │   ├── Span E: Redis GET stock:123 (duration: 2ms)
    │   └── Span F: MySQL SELECT inventory (duration: 20ms)
    └── Span G: MySQL INSERT orders (duration: 10ms)
```

- **Trace**：完整的一次请求链路，由全局唯一的 `TraceID` 标识
- **Span**：链路中的一个工作单元（如一次函数调用、一次 DB 查询），包含开始时间、持续时间、操作名称等
- **SpanContext**：在服务间传递的上下文，包含 `TraceID` 和 `SpanID`

### 1.2 go-zero 支持的追踪点

go-zero 内置了以下追踪点（启用 Telemetry 后自动生效）：

| 类型 | 追踪内容 |
|------|---------|
| HTTP 服务端 | 请求路径、方法、状态码、耗时 |
| HTTP 客户端 | 请求 URL、方法、状态码、耗时 |
| gRPC 服务端 | 服务名、方法名、状态码、耗时 |
| gRPC 客户端 | 服务名、方法名、状态码、耗时 |
| SQL 查询 | 数据库名、SQL 语句、耗时 |
| Redis 操作 | 命令、key、耗时 |

## 2. 快速上手

### 2.1 部署 Jaeger（可视化追踪系统）

```bash
# 使用 Docker 一键启动 Jaeger
docker run -d --name jaeger \
  -e COLLECTOR_OTLP_ENABLED=true \
  -p 16686:16686 \
  -p 4317:4317 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest

# 访问 Jaeger UI
open http://localhost:16686
```

也可以使用 Zipkin 或 Tempo（Grafana 生态）：

```bash
# Zipkin
docker run -d -p 9411:9411 openzipkin/zipkin

# Grafana Tempo + Grafana
docker run -d -p 3200:3200 -p 4317:4317 grafana/tempo
```

### 2.2 配置 go-zero 启用链路追踪

只需在配置文件中添加 `Telemetry` 配置块：

```yaml
Name: order-api
Host: 0.0.0.0
Port: 8888

# 链路追踪配置
Telemetry:
  Name: order-api           # 服务名称，在 Jaeger 中显示
  Endpoint: http://localhost:4318/v1/traces  # OTLP HTTP 端点
  # 或者使用 gRPC 端点：
  # Endpoint: grpc://localhost:4317
  Sampler: 1.0              # 采样率 1.0 = 100%，生产环境建议 0.1
  Batcher: otlphttp         # 使用 OTLP HTTP 协议（otlpgrpc 用于 gRPC）
```

完成该配置后，go-zero 会自动为 HTTP 和 gRPC 请求启用链路追踪。

### 2.3 RPC 服务也需要配置

每个参与追踪的服务都需要配置 Telemetry：

```yaml
# user-rpc etc/user.yaml
Name: user.rpc
ListenOn: 0.0.0.0:3001

Telemetry:
  Name: user-rpc
  Endpoint: http://localhost:4318/v1/traces
  Sampler: 1.0
  Batcher: otlphttp
```

go-zero 会通过 gRPC metadata 自动传播 TraceID，将跨服务的调用链串联起来。

## 3. 手动添加自定义 Span

自动追踪覆盖了大多数场景，但有时需要追踪特定的业务逻辑：

### 3.1 在业务逻辑中添加 Span

```go
package logic

import (
    "context"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"

    "github.com/zeromicro/go-zero/core/logx"
    "myapp/internal/svc"
    "myapp/internal/types"
)

type CreateOrderLogic struct {
    logx.Logger
    ctx    context.Context
    svcCtx *svc.ServiceContext
}

func (l *CreateOrderLogic) CreateOrder(req *types.OrderRequest) (*types.OrderResponse, error) {
    // 创建自定义 Span
    tracer := otel.Tracer("order-logic")
    ctx, span := tracer.Start(l.ctx, "CreateOrder")
    defer span.End()

    // 添加业务相关属性
    span.SetAttributes(
        attribute.Int64("user.id", req.UserId),
        attribute.Int64("product.id", req.ProductId),
        attribute.Int("quantity", req.Quantity),
    )

    // 校验库存
    ctx, stockSpan := tracer.Start(ctx, "CheckInventory")
    available, err := l.checkInventory(ctx, req.ProductId, req.Quantity)
    stockSpan.End()
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "库存检查失败")
        return nil, err
    }
    if !available {
        span.SetAttributes(attribute.Bool("inventory.available", false))
        return nil, errors.New("库存不足")
    }

    // 创建订单
    ctx, dbSpan := tracer.Start(ctx, "InsertOrder")
    orderId, err := l.svcCtx.OrderModel.Insert(ctx, &model.Order{
        UserId:    req.UserId,
        ProductId: req.ProductId,
        Quantity:  req.Quantity,
        Status:    "pending",
    })
    dbSpan.End()
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "创建订单失败")
        return nil, err
    }

    // 在 Span 中记录关键信息
    span.SetAttributes(attribute.Int64("order.id", orderId))
    span.SetStatus(codes.Ok, "")

    return &types.OrderResponse{OrderId: orderId}, nil
}
```

### 3.2 在 Span 中记录事件

```go
// 记录关键事件（时间点）
span.AddEvent("payment.initiated", trace.WithAttributes(
    attribute.Float64("amount", order.Amount),
    attribute.String("currency", "CNY"),
))

// 一段时间后...
span.AddEvent("payment.completed")
```

### 3.3 传播 TraceID 到日志

将 TraceID 注入日志，实现日志与链路追踪的关联：

```go
func (l *OrderLogic) ProcessOrder(req *types.OrderRequest) (*types.OrderResponse, error) {
    // go-zero 的 logx 会自动从 context 中提取 TraceID 并注入日志
    // 使用 logx.WithContext 即可
    logx.WithContext(l.ctx).Infof("开始处理订单: userId=%d, productId=%d",
        req.UserId, req.ProductId)

    // 上面的日志会自动包含 trace_id 字段，例如：
    // {"level":"info","ts":"2024-01-01T12:00:00Z","trace_id":"abc123...","msg":"开始处理订单..."}

    // 这样可以直接在日志系统中用 TraceID 找到对应的调用链，
    // 也可以从 Jaeger 的 Span 中找到对应的日志行
    return l.createOrder(l.ctx, req)
}
```

## 4. 采样策略

在高流量生产环境中，100% 采样的追踪数据量会非常大。go-zero 支持多种采样策略：

### 4.1 固定比例采样

```yaml
Telemetry:
  Name: my-service
  Endpoint: http://jaeger:4318/v1/traces
  Sampler: 0.1    # 10% 采样，即每 10 个请求采样 1 个
  Batcher: otlphttp
```

### 4.2 采样策略选择建议

| 环境 | 推荐采样率 | 说明 |
|------|----------|------|
| 开发环境 | 1.0 (100%) | 完整追踪，方便调试 |
| 测试环境 | 1.0 (100%) | 完整追踪，验证链路 |
| 预发布环境 | 0.1 (10%) | 代表性样本 |
| 生产环境（低流量）| 0.1-0.5 | 根据流量调整 |
| 生产环境（高流量）| 0.01-0.05 | 减少存储和网络开销 |
| 异常请求 | 1.0 (100%) | 通过头部标记强制采样 |

### 4.3 强制采样（Baggage）

对于需要完整追踪的特殊请求（如调试某个用户的请求），可以通过 HTTP Header 强制采样：

```bash
# 在 HTTP 请求中添加强制采样头
curl -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01" \
  http://your-service/api/order
```

## 5. 与 Grafana 集成

在现代可观测性体系中，通常将 Traces、Metrics 和 Logs 集成到 Grafana 进行统一可视化：

### 5.1 Grafana + Tempo + Loki 架构

```
go-zero 服务
    ├── Traces → Grafana Tempo
    ├── Metrics → Prometheus
    └── Logs → Loki (通过 logx 的 logrus/zap 格式)

Grafana
    ├── 展示 Tempo 的 Traces
    ├── 展示 Prometheus 的 Metrics
    └── 展示 Loki 的 Logs
    └── 三者可以通过 TraceID 互相跳转
```

### 5.2 docker-compose 示例

```yaml
version: '3'
services:
  tempo:
    image: grafana/tempo:latest
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "3200:3200"   # Tempo 查询端口
    command: ["-config.file=/etc/tempo.yaml"]
    volumes:
      - ./tempo.yaml:/etc/tempo.yaml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
```

### 5.3 go-zero 服务配置连接 Grafana Tempo

```yaml
Telemetry:
  Name: order-api
  Endpoint: http://tempo:4318/v1/traces
  Sampler: 0.1
  Batcher: otlphttp
```

## 6. 实际调试案例

### 6.1 定位慢请求

假设你收到告警：`/api/checkout` 接口 P99 延迟达到 2 秒。

1. 打开 Jaeger UI，搜索 `/api/checkout` 的慢 Trace（时间 > 1s）
2. 展开 Trace，查看各 Span 的耗时占比：

```
POST /api/checkout                           2100ms
├── gRPC UserService.GetUser                   25ms
├── gRPC CartService.GetCart                   30ms
├── gRPC InventoryService.CheckStock         1800ms  ⚠️ 这里有问题！
│   ├── Redis GET stock:product:456             2ms
│   └── MySQL SELECT inventory               1790ms  ⚠️ 慢查询！
│       (sql="SELECT * FROM inventory WHERE product_id=456 AND warehouse_id IN (...)")
└── gRPC PaymentService.CreatePayment         200ms
```

3. 通过 Trace 立即定位到是 `inventory` 表的某个查询没有索引，导致全表扫描！

### 6.2 追踪异常传播

```go
// 当 span 记录了错误，在 Jaeger 中会用红色标记显示
// 可以快速看到错误在哪个 Span 产生，以及错误信息
span.RecordError(fmt.Errorf("支付失败: 余额不足"))
span.SetStatus(codes.Error, "payment_failed")
```

## 7. 最佳实践

1. **为关键业务操作创建 Span**：支付、下单、核销等关键路径应有独立 Span
2. **在 Span 中记录关键参数**：用户ID、订单ID等有助于过滤和关联分析
3. **Span 名称语义化**：使用 `service.operation` 格式，如 `order.create`、`payment.charge`
4. **不要记录敏感信息**：密码、支付卡号等不应出现在 Span 属性中
5. **生产环境控制采样率**：避免追踪系统成为性能瓶颈
6. **将 TraceID 透传到日志**：通过 `logx.WithContext(ctx)` 自动注入 TraceID

## 8. 总结

go-zero 的链路追踪方案基于行业标准的 OpenTelemetry，具有以下特点：

- **接入简单**：配置 Telemetry 后即可启用框架层自动埋点
- **全链路覆盖**：HTTP、gRPC、SQL、Redis 均有内置追踪点
- **上下文自动传播**：TraceID 在服务间自动传递，无需手动处理
- **标准兼容**：支持 Jaeger、Zipkin、Grafana Tempo 等主流追踪后端
- **低性能开销**：通过采样控制追踪数据量，对业务影响极小

配合 go-zero 的日志系统（`logx`）和 Metrics（Prometheus），可以构建完整的可观测性体系，让微服务的运维从"凭经验猜测"变为"数据驱动决策"。
