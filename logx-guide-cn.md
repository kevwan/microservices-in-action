# go-zero logx：结构化日志最佳实践

## 前言

日志是可观测性三支柱（Logs、Metrics、Traces）中最直接的工具。当生产故障发生时，日志往往是第一手资料。go-zero 内置的 `logx` 包提供了开箱即用的结构化日志，支持 JSON 输出、日志级别、Context 传播、字段注入、多 Writer 等特性。

## 1. 基本用法

### 1.1 六个日志级别

```go
import "github.com/zeromicro/go-zero/core/logx"

// 调试信息（Debug 级别以下默认不打印）
logx.Debug("调试信息")
logx.Debugf("用户 %d 登录", userID)

// 普通信息
logx.Info("服务启动完成")
logx.Infof("订单 %s 创建成功", orderID)

// 结构化字段
logx.Infow("订单创建",
    logx.Field("orderId", orderID),
    logx.Field("userId", userID),
    logx.Field("amount", amount),
)

// 慢请求警告
logx.Slow("查询超时") // 慢日志，会标记 level=slow

// 错误
logx.Error("数据库连接失败")
logx.Errorf("支付失败: %v", err)

// Fatal（打印后调用 os.Exit(1)）
logx.Fatal("无法绑定端口，退出")
```

### 1.2 默认 JSON 输出格式

```json
{
  "@timestamp": "2024-01-01T12:00:00.000Z",
  "level": "info",
  "content": "订单创建成功",
  "trace": "abc123def456",
  "span": "123456",
  "caller": "logic/createorderlogic.go:45"
}
```

## 2. Context 日志（推荐）

在 Handler 和 Logic 中，始终使用 `logx.WithContext(ctx)` 打印日志，自动携带链路追踪 ID：

```go
// Logic 层
func (l *CreateOrderLogic) CreateOrder(req *types.CreateOrderReq) (*types.CreateOrderResp, error) {
    // 使用 ctx 日志：自动注入 traceId、spanId
    logx.WithContext(l.ctx).Infof("开始创建订单: userId=%d, productId=%d",
        req.UserId, req.ProductId)

    // ...

    logx.WithContext(l.ctx).Infow("订单创建完成",
        logx.Field("orderId", orderId),
        logx.Field("duration", time.Since(start).Milliseconds()),
    )
    return resp, nil
}
```

输出示例：

```json
{
  "@timestamp": "2024-01-01T12:00:00.123Z",
  "level": "info",
  "content": "订单创建完成",
  "trace": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span": "00f067aa0ba902b7",
  "orderId": "ORD-2024-001",
  "duration": 45
}
```

## 3. 配置 logx

```yaml
# etc/config.yaml
Log:
  ServiceName: order-api       # 服务名（注入到每条日志）
  Level: info                  # 日志级别: debug/info/error
  Mode: console                # 输出模式: console（彩色）/ file / volume
  Encoding: json               # 编码: json / plain
  TimeFormat: "2006-01-02T15:04:05.000Z07:00"
  Path: /var/log/order-api     # 文件模式下的日志目录
  MaxSize: 100                 # 单文件最大 100MB，超出自动轮转
  MaxBackups: 10               # 保留 10 个历史日志文件
  MaxAge: 7                    # 保留最近 7 天的日志
  Compress: true               # gzip 压缩历史日志
  Stat: false                  # 是否开启 stat 日志（默认 false）
```

```go
// main.go 中加载日志配置
var c config.Config
conf.MustLoad(*configFile, &c)
logx.MustSetup(c.Log)  // 应用日志配置
```

## 4. 自定义字段（全局 + 请求级别）

### 4.1 全局字段（所有日志都携带）

```go
// 在程序启动时设置全局字段
logx.AddGlobalFields(
    logx.Field("env", "production"),
    logx.Field("region", "cn-hangzhou"),
    logx.Field("version", "v1.2.3"),
)
```

### 4.2 请求级别字段（注入到 Context）

```go
// 在中间件中注入请求级别字段
func RequestContextMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx := logx.ContextWithFields(r.Context(),
            logx.Field("requestId", r.Header.Get("X-Request-Id")),
            logx.Field("userId", r.Header.Get("X-User-Id")),
            logx.Field("clientVersion", r.Header.Get("X-App-Version")),
        )
        next(w, r.WithContext(ctx))
    }
}
```

后续所有 `logx.WithContext(ctx)` 的日志都会自动携带这些字段：

```json
{
  "level": "info",
  "content": "订单创建完成",
  "requestId": "req-abc-123",
  "userId": "42",
  "clientVersion": "3.1.0",
  "env": "production",
  "region": "cn-hangzhou"
}
```

## 5. 慢日志

go-zero 内置了慢日志机制，自动记录超过阈值的请求：

```yaml
# 配置慢日志阈值
Log:
  Stat: true          # 开启统计日志
```

```go
// 手动记录慢操作
func (l *SearchLogic) Search(req *types.SearchReq) (*types.SearchResp, error) {
    start := time.Now()
    result, err := l.elasticsearch.Search(req.Query)

    duration := time.Since(start)
    if duration > 500*time.Millisecond {
        // 超过 500ms 记录慢日志
        logx.WithContext(l.ctx).Slowf("ElasticSearch 查询慢: query=%s, duration=%dms",
            req.Query, duration.Milliseconds())
    }
    // ...
}
```

## 6. 日志聚合：接入 ELK / Loki

### 6.1 Filebeat → Elasticsearch

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    paths:
      - /var/log/order-api/*.log
    fields:
      service: order-api
    json.keys_under_root: true
    json.add_error_key: true

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "go-zero-logs-%{+yyyy.MM.dd}"
```

### 6.2 Promtail → Loki（K8s 推荐）

```yaml
# promtail-config.yaml（K8s DaemonSet）
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    pipeline_stages:
      - json:
          expressions:
            level: level
            trace: trace
            service: "@service"
      - labels:
          level:
          service:
```

在 Grafana 中查询：

```
# 查询 order-api 的所有错误日志
{service="order-api"} |= "error" | json

# 查询特定 traceId 的所有日志（跨服务）
{namespace="default"} |= "4bf92f3577b34da6a3ce929d0e0e4736"
```

## 7. 禁用日志（测试场景）

```go
// 测试时禁用日志输出，避免干扰测试结果
func TestMain(m *testing.M) {
    logx.Disable()  // 禁用所有日志输出
    os.Exit(m.Run())
}
```

## 8. 总结

| 特性 | 用法 |
|------|------|
| Context 日志 | `logx.WithContext(ctx).Infof(...)` |
| 结构化字段 | `logx.Infow("msg", logx.Field("key", value))` |
| 全局字段 | `logx.AddGlobalFields(...)` |
| 请求级字段 | `logx.ContextWithFields(ctx, ...)` |
| 慢日志 | `logx.Slowf(...)` |
| 日志轮转 | 配置 `MaxSize`/`MaxBackups`/`MaxAge` |
| 禁用日志 | `logx.Disable()`（测试用） |

核心原则：**始终用 Context 日志**，确保同一请求的所有日志携带相同的 traceId，方便在 Grafana/Kibana 中一键过滤出完整请求链路。
