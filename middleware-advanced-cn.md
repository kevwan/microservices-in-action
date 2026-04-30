# go-zero 中间件进阶：链式调用、复用与最佳实践

## 前言

中间件（Middleware）是 go-zero HTTP 服务的扩展机制，用于在请求到达业务逻辑前后注入横切关注点：日志、鉴权、限流、链路追踪、请求验证等。go-zero 的中间件系统设计简洁，基于标准的 `http.Handler` 装饰器模式。

## 1. 中间件基础

### 1.1 中间件签名

go-zero 中间件是一个将 `http.HandlerFunc` 包装为另一个 `http.HandlerFunc` 的函数：

```go
type Middleware func(next http.HandlerFunc) http.HandlerFunc
```

### 1.2 最简单的中间件

```go
func LogMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        logx.WithContext(r.Context()).Infof("→ %s %s", r.Method, r.URL.Path)

        next(w, r)  // 调用下一个处理器

        logx.WithContext(r.Context()).Infof("← %s %s [%v]",
            r.Method, r.URL.Path, time.Since(start))
    }
}
```

## 2. 注册中间件的三种方式

### 2.1 全局中间件（所有路由生效）

```go
server := rest.MustNewServer(c.RestConf)

// 全局中间件：所有路由都会经过
server.Use(middleware.LogMiddleware)
server.Use(middleware.RequestIDMiddleware)
server.Use(middleware.CorsMiddleware)

handler.RegisterHandlers(server, ctx)
```

### 2.2 路由组中间件（.api 文件中声明）

```api
// 需要鉴权的路由组
@server (
    middleware: AuthCheck, RateLimiter
    jwt: Auth
)
service order-api {
    @handler CreateOrder
    post /orders (CreateOrderReq) returns (CreateOrderResp)
}

// 公开路由组（无中间件）
@server ()
service order-api {
    @handler HealthCheck
    get /health
}
```

go-zero 会生成 ServiceContext 中的中间件字段：

```go
// internal/svc/servicecontext.go
type ServiceContext struct {
    Config      config.Config
    AuthCheck   rest.Middleware
    RateLimiter rest.Middleware
}

func NewServiceContext(c config.Config) *ServiceContext {
    return &ServiceContext{
        Config:      c,
        AuthCheck:   middleware.NewAuthCheckMiddleware(c).Handle,
        RateLimiter: middleware.NewRateLimiterMiddleware(c).Handle,
    }
}
```

### 2.3 单路由中间件（Handler 内部）

```go
// 在特定 Handler 中手动包装
func SpecialHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
    return middleware.AdminOnlyMiddleware(
        func(w http.ResponseWriter, r *http.Request) {
            l := logic.NewSpecialLogic(r.Context(), svcCtx)
            resp, err := l.Special()
            // ...
        },
    )
}
```

## 3. 常用中间件实现

### 3.1 请求 ID 中间件

```go
// internal/middleware/requestid.go
package middleware

import (
    "net/http"

    "github.com/google/uuid"
    "github.com/zeromicro/go-zero/core/logx"
)

const RequestIDKey = "X-Request-Id"

func RequestIDMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // 优先使用客户端传入的 Request-ID（用于跨服务追踪）
        requestID := r.Header.Get(RequestIDKey)
        if requestID == "" {
            requestID = uuid.NewString()
        }

        // 写入响应头，方便客户端排查
        w.Header().Set(RequestIDKey, requestID)

        // 注入到 Context（后续中间件和 Logic 层可读取）
        ctx := context.WithValue(r.Context(), RequestIDKey, requestID)

        // 为当前请求的所有日志添加 requestId 字段
        ctx = logx.ContextWithFields(ctx, logx.Field("requestId", requestID))

        next(w, r.WithContext(ctx))
    }
}
```

### 3.2 访问日志中间件（结构化）

```go
// internal/middleware/accesslog.go

type responseWriter struct {
    http.ResponseWriter
    statusCode int
    bodySize   int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
    n, err := rw.ResponseWriter.Write(b)
    rw.bodySize += n
    return n, err
}

func AccessLogMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        rw := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

        next(rw, r)

        duration := time.Since(start)
        logx.WithContext(r.Context()).Infow("access",
            logx.Field("method", r.Method),
            logx.Field("path", r.URL.Path),
            logx.Field("query", r.URL.RawQuery),
            logx.Field("status", rw.statusCode),
            logx.Field("bodySize", rw.bodySize),
            logx.Field("duration", duration.Milliseconds()),
            logx.Field("ip", getRealIP(r)),
            logx.Field("userAgent", r.UserAgent()),
        )
    }
}

func getRealIP(r *http.Request) string {
    // 处理反向代理
    if ip := r.Header.Get("X-Real-IP"); ip != "" {
        return ip
    }
    if ips := r.Header.Get("X-Forwarded-For"); ips != "" {
        return strings.Split(ips, ",")[0]
    }
    return r.RemoteAddr
}
```

### 3.3 IP 白名单中间件

```go
// internal/middleware/ipwhitelist.go

func NewIPWhitelistMiddleware(allowedIPs []string) func(http.HandlerFunc) http.HandlerFunc {
    // 构建允许 IP 集合
    allowed := make(map[string]struct{})
    for _, ip := range allowedIPs {
        allowed[ip] = struct{}{}
    }

    return func(next http.HandlerFunc) http.HandlerFunc {
        return func(w http.ResponseWriter, r *http.Request) {
            clientIP := getRealIP(r)
            // 去掉端口
            if host, _, err := net.SplitHostPort(clientIP); err == nil {
                clientIP = host
            }

            if _, ok := allowed[clientIP]; !ok {
                http.Error(w, "forbidden", http.StatusForbidden)
                return
            }
            next(w, r)
        }
    }
}
```

### 3.4 请求体大小限制中间件

```go
// 防止大请求体耗尽内存
func MaxBodyMiddleware(maxBytes int64) func(http.HandlerFunc) http.HandlerFunc {
    return func(next http.HandlerFunc) http.HandlerFunc {
        return func(w http.ResponseWriter, r *http.Request) {
            r.Body = http.MaxBytesReader(w, r.Body, maxBytes)
            next(w, r)
        }
    }
}
```

### 3.5 Panic Recovery 中间件

```go
func RecoveryMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                // 打印堆栈
                buf := make([]byte, 64<<10)
                buf = buf[:runtime.Stack(buf, false)]
                logx.WithContext(r.Context()).Errorf("panic: %v\n%s", rec, buf)

                http.Error(w, `{"code":10000,"message":"服务器内部错误"}`,
                    http.StatusInternalServerError)
            }
        }()
        next(w, r)
    }
}
```

## 4. 中间件执行顺序

中间件的执行顺序遵循**洋葱模型**：注册顺序决定执行顺序，先注册的中间件在最外层。

```
请求进入
    │
    ▼
RequestID（注册第1个）
    │
    ▼
AccessLog（注册第2个）
    │
    ▼
Auth（注册第3个）
    │
    ▼
RateLimiter（注册第4个）
    │
    ▼
业务 Handler
    │
    ▼（响应返回，逆序执行后续代码）
RateLimiter after
    │
    ▼
Auth after
    │
    ▼
AccessLog after（记录耗时）
    │
    ▼
RequestID after（无）
    │
    ▼
响应返回客户端
```

推荐的中间件注册顺序：

```go
server.Use(middleware.RequestIDMiddleware)  // 1. 最外层：分配请求ID
server.Use(middleware.RecoveryMiddleware)   // 2. Panic 恢复
server.Use(middleware.AccessLogMiddleware)  // 3. 访问日志（记录完整耗时）
server.Use(middleware.CorsMiddleware)       // 4. CORS（在鉴权之前，OPTIONS 不需要鉴权）
// 路由组级别：Auth / RateLimiter（在 .api 文件中配置）
```

## 5. 有状态中间件（带依赖注入）

当中间件需要依赖 Redis、数据库等外部资源时，使用结构体封装：

```go
// internal/middleware/ratelimiter.go
type RateLimiterMiddleware struct {
    rdb     *redis.Redis
    limit   int
    window  time.Duration
}

func NewRateLimiterMiddleware(rdb *redis.Redis, limit int, window time.Duration) *RateLimiterMiddleware {
    return &RateLimiterMiddleware{rdb: rdb, limit: limit, window: window}
}

// Handle 方法满足 rest.Middleware 类型
func (m *RateLimiterMiddleware) Handle(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        userID := r.Header.Get("X-User-Id")
        key := fmt.Sprintf("rate:%s", userID)

        count, err := m.rdb.Incr(key)
        if err != nil {
            // 限流失败开放（fail-open）
            next(w, r)
            return
        }
        if count == 1 {
            m.rdb.Expire(key, int(m.window.Seconds()))
        }
        if count > int64(m.limit) {
            httpx.WriteJsonCtx(r.Context(), w, http.StatusTooManyRequests,
                map[string]any{"code": 42900, "message": "请求过于频繁"})
            return
        }

        next(w, r)
    }
}
```

## 6. 中间件测试

```go
// internal/middleware/requestid_test.go
func TestRequestIDMiddleware(t *testing.T) {
    // 构造测试 Handler
    called := false
    handler := RequestIDMiddleware(func(w http.ResponseWriter, r *http.Request) {
        called = true
        // 验证 Context 中有 RequestID
        id := r.Context().Value(RequestIDKey)
        assert.NotEmpty(t, id)
    })

    // 发起测试请求
    req := httptest.NewRequest(http.MethodGet, "/test", nil)
    w := httptest.NewRecorder()
    handler(w, req)

    assert.True(t, called)
    assert.NotEmpty(t, w.Header().Get(RequestIDKey))
}

func TestRequestIDMiddleware_PreserveExisting(t *testing.T) {
    existingID := "test-request-id-123"
    handler := RequestIDMiddleware(func(w http.ResponseWriter, r *http.Request) {
        id := r.Context().Value(RequestIDKey)
        assert.Equal(t, existingID, id)
    })

    req := httptest.NewRequest(http.MethodGet, "/test", nil)
    req.Header.Set(RequestIDKey, existingID)  // 客户端传入已有 ID
    w := httptest.NewRecorder()
    handler(w, req)

    assert.Equal(t, existingID, w.Header().Get(RequestIDKey))
}
```

## 7. 总结

| 注册方式 | 适用场景 | 示例 |
|---------|---------|------|
| `server.Use()` | 全局横切关注点 | 请求ID、CORS、Panic恢复 |
| `.api` 文件 `middleware` | 路由组级别 | 鉴权、业务限流 |
| Handler 内手动包装 | 单个路由特殊逻辑 | 管理员专属权限 |

设计原则：
- **单一职责**：每个中间件只做一件事
- **无副作用**：不修改请求体（需要时先 copy）
- **快速失败**：尽早拒绝非法请求，减少下游压力
- **可测试**：使用 `httptest` 独立测试每个中间件
