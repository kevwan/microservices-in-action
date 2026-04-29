# go-zero gRPC 拦截器：统一处理 RPC 横切关注点

## 前言

gRPC 拦截器（Interceptor）是 RPC 服务的"中间件"，与 HTTP 中间件的作用相同：在请求进入业务逻辑前后注入横切逻辑。go-zero 内置了多个拦截器（链路追踪、日志、熔断、指标、恢复等），同时支持自定义拦截器扩展 RPC 行为。

## 1. 拦截器类型

gRPC 有两种拦截器：

- **UnaryInterceptor**：拦截普通（一元）RPC 调用（最常用）
- **StreamInterceptor**：拦截流式 RPC 调用

服务端和客户端各自可以注册拦截器：

```
客户端拦截器链                    服务端拦截器链

请求 → 限流 → 重试 → 压缩 →   网络   → 认证 → 日志 → 追踪 → 业务逻辑

                              ← 错误处理 ← 响应 ←
```

## 2. go-zero 内置拦截器

go-zero 的 RPC 服务端（`zrpc.MustNewServer`）默认启用以下拦截器：

| 拦截器 | 功能 |
|--------|------|
| `prometheusUnaryInterceptor` | 上报 RPC 指标到 Prometheus |
| `otelUnaryInterceptor` | OpenTelemetry 链路追踪 |
| `logUnaryInterceptor` | 请求/响应日志 |
| `recoverInterceptor` | Panic 恢复 |
| `breaker` | 熔断保护 |

## 3. 服务端自定义拦截器

### 3.1 添加服务端拦截器

```go
// main.go
s := zrpc.MustNewServer(c.RpcServerConf, func(grpcServer *grpc.Server) {
    pb.RegisterOrderServiceServer(grpcServer, server.NewOrderServer(ctx))
})

// 添加自定义服务端拦截器
s.AddUnaryInterceptors(
    AuthInterceptor(c.Auth.AccessSecret),
    AuditInterceptor(),
)

defer s.Stop()
s.Start()
```

### 3.2 认证拦截器

```go
// interceptors/auth.go
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/metadata"
    "google.golang.org/grpc/status"
)

func AuthInterceptor(secret string) grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any,
        info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {

        // 从 metadata 获取 token
        md, ok := metadata.FromIncomingContext(ctx)
        if !ok {
            return nil, status.Error(codes.Unauthenticated, "缺少认证信息")
        }

        tokens := md.Get("authorization")
        if len(tokens) == 0 {
            return nil, status.Error(codes.Unauthenticated, "缺少 Authorization header")
        }

        token := strings.TrimPrefix(tokens[0], "Bearer ")
        claims, err := validateJWT(token, secret)
        if err != nil {
            return nil, status.Errorf(codes.Unauthenticated, "Token 无效: %v", err)
        }

        // 将 claims 注入 Context
        ctx = context.WithValue(ctx, "userId", claims.UserId)
        return handler(ctx, req)
    }
}
```

### 3.3 审计日志拦截器

```go
func AuditInterceptor() grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any,
        info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {

        start := time.Now()
        userId, _ := ctx.Value("userId").(int64)

        resp, err := handler(ctx, req)

        // 记录审计日志
        logx.WithContext(ctx).Infow("rpc-audit",
            logx.Field("method", info.FullMethod),
            logx.Field("userId", userId),
            logx.Field("duration", time.Since(start).Milliseconds()),
            logx.Field("error", fmt.Sprintf("%v", err)),
        )

        return resp, err
    }
}
```

### 3.4 请求验证拦截器

```go
// 自动调用请求结构体的 Validate() 方法
func ValidationInterceptor() grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any,
        info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {

        // 如果请求实现了 Validator 接口，自动校验
        if v, ok := req.(interface{ Validate() error }); ok {
            if err := v.Validate(); err != nil {
                return nil, status.Errorf(codes.InvalidArgument, "参数校验失败: %v", err)
            }
        }

        return handler(ctx, req)
    }
}
```

## 4. 客户端自定义拦截器

### 4.1 添加客户端拦截器

```go
// 在创建 RPC 客户端时添加拦截器
conn, err := zrpc.NewClient(
    c.OrderRpc,
    zrpc.WithUnaryClientInterceptor(
        MetadataInjector(),
        RetryInterceptor(3),
        TimeoutInterceptor(2*time.Second),
    ),
)
```

### 4.2 Metadata 注入拦截器（透传请求头）

```go
// 将 HTTP 请求的 requestId、traceId 透传给 RPC 调用
func MetadataInjector() grpc.UnaryClientInterceptor {
    return func(ctx context.Context, method string, req, reply any,
        cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {

        // 从 Context 获取 requestId 并注入到 gRPC metadata
        md := metadata.New(nil)

        if requestId, ok := ctx.Value("requestId").(string); ok {
            md.Set("x-request-id", requestId)
        }

        // 将服务名注入，方便服务端日志识别调用来源
        md.Set("x-caller-service", "api-gateway")

        ctx = metadata.NewOutgoingContext(ctx, md)
        return invoker(ctx, method, req, reply, cc, opts...)
    }
}
```

### 4.3 重试拦截器

```go
func RetryInterceptor(maxRetries int) grpc.UnaryClientInterceptor {
    return func(ctx context.Context, method string, req, reply any,
        cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {

        var lastErr error
        for i := 0; i <= maxRetries; i++ {
            if i > 0 {
                // 指数退避
                backoff := time.Duration(math.Pow(2, float64(i-1))) * 100 * time.Millisecond
                select {
                case <-time.After(backoff):
                case <-ctx.Done():
                    return ctx.Err()
                }
                logx.WithContext(ctx).Infof("RPC 重试 %d/%d: method=%s", i, maxRetries, method)
            }

            lastErr = invoker(ctx, method, req, reply, cc, opts...)
            if lastErr == nil {
                return nil
            }

            // 只重试可重试的错误（网络超时、临时不可用）
            st, ok := status.FromError(lastErr)
            if !ok {
                return lastErr  // 非 gRPC 错误，不重试
            }
            switch st.Code() {
            case codes.Unavailable, codes.DeadlineExceeded:
                continue  // 可重试
            default:
                return lastErr  // 业务错误（NotFound、PermissionDenied 等），不重试
            }
        }
        return lastErr
    }
}
```

## 5. 流式拦截器

```go
// 流式 RPC 服务端拦截器
func StreamLoggingInterceptor() grpc.StreamServerInterceptor {
    return func(srv any, ss grpc.ServerStream, info *grpc.StreamServerInfo,
        handler grpc.StreamHandler) error {

        start := time.Now()
        logx.Infof("流式 RPC 开始: method=%s, isClientStream=%v, isServerStream=%v",
            info.FullMethod, info.IsClientStream, info.IsServerStream)

        err := handler(srv, ss)

        logx.Infof("流式 RPC 结束: method=%s, duration=%v, error=%v",
            info.FullMethod, time.Since(start), err)
        return err
    }
}

// 注册流式拦截器
s.AddStreamInterceptors(StreamLoggingInterceptor())
```

## 6. 拦截器链执行顺序

拦截器按注册顺序形成链，请求从第一个拦截器进入，响应从最后一个返回：

```go
// 注册顺序决定执行顺序
s.AddUnaryInterceptors(
    RecoveryInterceptor(),  // 1. 最外层：Panic 恢复
    LoggingInterceptor(),   // 2. 日志（记录完整耗时）
    AuthInterceptor(),      // 3. 认证
    ValidationInterceptor(), // 4. 参数校验
    AuditInterceptor(),     // 5. 审计（最靠近业务）
)

// 执行顺序：Recovery → Logging → Auth → Validation → Audit → Handler
// 返回顺序：Audit ← Validation ← Auth ← Logging（记录耗时）← Recovery
```

## 7. 总结

| 场景 | 拦截器类型 | 位置 |
|------|-----------|------|
| JWT/Token 验证 | UnaryServerInterceptor | 服务端 |
| 审计日志 | UnaryServerInterceptor | 服务端 |
| 参数校验 | UnaryServerInterceptor | 服务端 |
| 请求头透传 | UnaryClientInterceptor | 客户端 |
| 自动重试 | UnaryClientInterceptor | 客户端 |
| 超时控制 | UnaryClientInterceptor | 客户端 |

go-zero 已内置了最常用的拦截器（追踪、日志、熔断、Prometheus），自定义拦截器专注于业务需求，保持简洁。
