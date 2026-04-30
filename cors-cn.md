# go-zero CORS 跨域处理：原理与最佳实践

## 前言

当你的前端应用（比如运行在 `https://app.example.com`）尝试调用后端 API（`https://api.example.com`）时，浏览器会检测到这是一个**跨域请求**，并触发 **CORS（Cross-Origin Resource Sharing，跨源资源共享）** 检查。如果服务端没有正确配置 CORS，浏览器会拦截响应，前端会收到类似 `Access to fetch... has been blocked by CORS policy` 的错误。

go-zero 通过中间件机制提供了灵活且安全的 CORS 处理方案。本文将深入解析 CORS 原理，并给出 go-zero 中的完整实现示例。

## 1. CORS 原理

### 1.1 什么是跨域

**同源策略**要求协议、域名、端口三者完全相同才算同源：

| 请求来源 | 目标地址 | 是否跨域 |
| ------- | ------- | ------- |
| `https://app.com` | `https://app.com/api` | 否（同源） |
| `https://app.com` | `https://api.app.com` | 是（子域不同） |
| `https://app.com` | `http://app.com` | 是（协议不同） |
| `https://app.com:443` | `https://app.com:8888` | 是（端口不同） |

### 1.2 简单请求 vs 预检请求

**简单请求**（直接发送，同时附带 `Origin` 头）：

- 方法：GET、POST、HEAD
- Content-Type 仅限：`text/plain`、`application/x-www-form-urlencoded`、`multipart/form-data`

**预检请求**（Preflight，先发送 OPTIONS 请求确认服务端允许）：

- PUT、DELETE、PATCH 等方法
- Content-Type 为 `application/json`
- 携带自定义请求头（如 `Authorization`）

```text
预检请求流程：

浏览器                                          服务器
  │                                               │
  │── OPTIONS /api/order ──────────────────────→  │
  │   Origin: https://app.example.com             │
  │   Access-Control-Request-Method: POST         │
  │   Access-Control-Request-Headers: Authorization│
  │                                               │
  │←─ 200 OK ──────────────────────────────────── │
  │   Access-Control-Allow-Origin: https://app.example.com
  │   Access-Control-Allow-Methods: POST, GET, OPTIONS
  │   Access-Control-Allow-Headers: Authorization, Content-Type
  │   Access-Control-Max-Age: 3600                │
  │                                               │
  │ （预检通过，发送实际请求）                       │
  │── POST /api/order ──────────────────────────→  │
  │   Authorization: Bearer xxx                   │
```

### 1.3 关键响应头含义

| 响应头 | 含义 | 示例 |
| ------ | ---- | ---- |
| `Access-Control-Allow-Origin` | 允许的来源域名 | `https://app.example.com` 或 `*` |
| `Access-Control-Allow-Methods` | 允许的 HTTP 方法 | `GET, POST, PUT, DELETE` |
| `Access-Control-Allow-Headers` | 允许的请求头 | `Authorization, Content-Type` |
| `Access-Control-Allow-Credentials` | 是否允许携带 Cookie | `true` |
| `Access-Control-Max-Age` | 预检结果缓存时间（秒） | `3600` |
| `Access-Control-Expose-Headers` | 允许前端读取的响应头 | `X-Request-Id, X-RateLimit-Remaining` |

## 2. go-zero 中实现 CORS 中间件

go-zero 没有内置 CORS 中间件（因为 CORS 策略高度依赖业务需求），但通过 `server.Use()` 添加自定义中间件非常简单。

### 2.1 基础 CORS 中间件

```go
// internal/middleware/corsmiddleware.go
package middleware

import (
    "net/http"
)

// CorsMiddleware 允许所有来源（仅适合开发环境或公开 API）
func CorsMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS, PATCH")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization, X-Request-Id")

        // 处理预检请求
        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusNoContent)
            return
        }

        next(w, r)
    }
}
```

在 `main.go` 中注册：

```go
server := rest.MustNewServer(c.RestConf)
defer server.Stop()

// 注册 CORS 中间件（在所有路由之前执行）
server.Use(middleware.CorsMiddleware)

handler.RegisterHandlers(server, ctx)
server.Start()
```

### 2.2 生产级 CORS 中间件（白名单控制）

生产环境中应严格限制允许的来源，避免安全风险：

```go
// internal/middleware/corsmiddleware.go
package middleware

import (
    "net/http"
    "strings"
)

type CorsConfig struct {
    // 允许的来源列表（精确匹配）
    AllowedOrigins []string
    // 是否允许携带 Cookie / Authorization（设为 true 时 AllowedOrigins 不能包含 *）
    AllowCredentials bool
    // 预检结果缓存时间（秒），减少 OPTIONS 请求数量
    MaxAge int
    // 允许前端读取的额外响应头
    ExposeHeaders []string
}

func NewCorsMiddleware(cfg CorsConfig) func(http.HandlerFunc) http.HandlerFunc {
    // 构建 origins 查找表（O(1) 查找）
    originSet := make(map[string]struct{}, len(cfg.AllowedOrigins))
    for _, origin := range cfg.AllowedOrigins {
        originSet[strings.ToLower(origin)] = struct{}{}
    }

    maxAge := "86400" // 默认缓存 1 天
    if cfg.MaxAge > 0 {
        maxAge = fmt.Sprintf("%d", cfg.MaxAge)
    }

    return func(next http.HandlerFunc) http.HandlerFunc {
        return func(w http.ResponseWriter, r *http.Request) {
            origin := r.Header.Get("Origin")

            // 没有 Origin 头，不是跨域请求，直接通过
            if origin == "" {
                next(w, r)
                return
            }

            // 检查来源是否在白名单中
            if _, allowed := originSet[strings.ToLower(origin)]; !allowed {
                // 来源不在白名单，不添加 CORS 头，浏览器将拦截
                // 注意：不要返回 403，否则会暴露接口存在
                if r.Method == http.MethodOptions {
                    w.WriteHeader(http.StatusNoContent)
                    return
                }
                next(w, r)
                return
            }

            // 来源合法，设置 CORS 响应头
            w.Header().Set("Access-Control-Allow-Origin", origin)
            w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS, PATCH")
            w.Header().Set("Access-Control-Allow-Headers",
                "Content-Type, Authorization, X-Request-Id, X-App-Version")
            w.Header().Set("Access-Control-Max-Age", maxAge)

            if cfg.AllowCredentials {
                w.Header().Set("Access-Control-Allow-Credentials", "true")
            }

            if len(cfg.ExposeHeaders) > 0 {
                w.Header().Set("Access-Control-Expose-Headers",
                    strings.Join(cfg.ExposeHeaders, ", "))
            }

            // Vary 头：告知缓存服务器按 Origin 缓存不同的响应
            w.Header().Add("Vary", "Origin")

            // 处理预检请求
            if r.Method == http.MethodOptions {
                w.WriteHeader(http.StatusNoContent)
                return
            }

            next(w, r)
        }
    }
}
```

在 `main.go` 中使用：

```go
server := rest.MustNewServer(c.RestConf)
defer server.Stop()

// 生产环境：只允许指定来源
corsMiddleware := middleware.NewCorsMiddleware(middleware.CorsConfig{
    AllowedOrigins: []string{
        "https://app.example.com",
        "https://admin.example.com",
    },
    AllowCredentials: true,   // 允许携带 Cookie 和 Authorization
    MaxAge:           3600,
    ExposeHeaders:    []string{"X-Request-Id", "X-RateLimit-Remaining"},
})

server.Use(corsMiddleware)
handler.RegisterHandlers(server, ctx)
server.Start()
```

### 2.3 从配置文件读取 CORS 设置

将 CORS 配置纳入配置文件，方便不同环境使用不同策略：

```yaml
# etc/config.yaml
Name: my-api
Host: 0.0.0.0
Port: 8888

Cors:
  AllowedOrigins:
    - "https://app.example.com"
    - "https://admin.example.com"
  AllowCredentials: true
  MaxAge: 3600
  ExposeHeaders:
    - "X-Request-Id"
    - "X-RateLimit-Remaining"
```

```go
// internal/config/config.go
type Config struct {
    rest.RestConf

    Cors struct {
        AllowedOrigins   []string
        AllowCredentials bool   `json:",default=false"`
        MaxAge           int    `json:",default=86400"`
        ExposeHeaders    []string
    }
}
```

```go
// main.go
corsMiddleware := middleware.NewCorsMiddleware(middleware.CorsConfig{
    AllowedOrigins:   c.Cors.AllowedOrigins,
    AllowCredentials: c.Cors.AllowCredentials,
    MaxAge:           c.Cors.MaxAge,
    ExposeHeaders:    c.Cors.ExposeHeaders,
})
server.Use(corsMiddleware)
```

## 3. 按路由分组配置 CORS

有时不同的 API 路由需要不同的 CORS 策略（比如公开 API 允许所有来源，内部管理 API 只允许管理后台）：

```api
// api.api

// 公开 API - 允许所有来源
@server (
    middleware: PublicCors
)
service my-api {
    @handler GetProduct
    get /api/v1/products/:id returns (ProductResp)
}

// 管理 API - 只允许管理后台域名
@server (
    middleware: AdminCors
    jwt: Auth
)
service my-api {
    @handler ListAllOrders
    get /admin/orders returns (OrderListResp)
}
```

```go
// internal/svc/servicecontext.go
type ServiceContext struct {
    Config     config.Config
    PublicCors rest.Middleware
    AdminCors  rest.Middleware
}

func NewServiceContext(c config.Config) *ServiceContext {
    return &ServiceContext{
        Config: c,
        // 公开 API CORS：允许所有来源
        PublicCors: middleware.NewCorsMiddleware(middleware.CorsConfig{
            AllowedOrigins: []string{"*"},
        }),
        // 管理 API CORS：严格限制
        AdminCors: middleware.NewCorsMiddleware(middleware.CorsConfig{
            AllowedOrigins:   []string{"https://admin.example.com"},
            AllowCredentials: true,
        }),
    }
}
```

## 4. 常见 CORS 问题排查

### 4.1 Cookie 无法携带

**现象**：前端 `credentials: 'include'` 后 Cookie 仍然不传

**原因**：

1. 服务端没有设置 `Access-Control-Allow-Credentials: true`
2. `Access-Control-Allow-Origin` 设置为 `*`（使用 credentials 时不能用通配符）

**解决**：

```go
// 必须同时满足两个条件：
w.Header().Set("Access-Control-Allow-Origin", "https://app.example.com")  // 不能是 *
w.Header().Set("Access-Control-Allow-Credentials", "true")
```

### 4.2 自定义请求头被拦截

**现象**：前端发送 `X-App-Version` 头后请求被 CORS 拦截

**原因**：自定义请求头需要在 `Access-Control-Allow-Headers` 中明确声明

**解决**：

```go
w.Header().Set("Access-Control-Allow-Headers",
    "Content-Type, Authorization, X-Request-Id, X-App-Version")
//                                              ↑ 必须包含所有自定义头
```

### 4.3 响应头无法读取

**现象**：前端 `response.headers.get('X-Request-Id')` 返回 `null`

**原因**：浏览器默认只暴露简单响应头，其他头需要通过 `Access-Control-Expose-Headers` 声明

**解决**：

```go
w.Header().Set("Access-Control-Expose-Headers", "X-Request-Id, X-RateLimit-Remaining")
```

### 4.4 反向代理的 CORS 配置

如果使用 Nginx/Traefik 作为反向代理，**应该在代理层统一处理 CORS，而不是在应用层**：

```nginx
# nginx.conf
server {
    location /api/ {
        # 在 Nginx 层处理 CORS，应用层无需处理
        add_header 'Access-Control-Allow-Origin' 'https://app.example.com' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type' always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;

        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Max-Age' '3600';
            return 204;
        }

        proxy_pass http://backend;
    }
}
```

## 5. CORS 安全注意事项

### 5.1 不要过度开放

```go
// ❌ 危险：允许所有来源 + 允许 Credentials
// 这意味着任何网站都可以用用户的 Cookie 调用你的 API
w.Header().Set("Access-Control-Allow-Origin", "*")
w.Header().Set("Access-Control-Allow-Credentials", "true")

// ✅ 正确：要么限制来源，要么不允许 Credentials
w.Header().Set("Access-Control-Allow-Origin", "https://app.example.com")
w.Header().Set("Access-Control-Allow-Credentials", "true")
```

### 5.2 来源验证要精确

```go
// ❌ 危险：使用 strings.Contains 验证来源
// 攻击者可以注册 "evil-example.com" 来绕过
if strings.Contains(origin, "example.com") {
    w.Header().Set("Access-Control-Allow-Origin", origin)
}

// ✅ 正确：使用精确匹配或合法的通配符（子域名）
allowedOrigins := map[string]bool{
    "https://app.example.com":   true,
    "https://admin.example.com": true,
}
if allowedOrigins[origin] {
    w.Header().Set("Access-Control-Allow-Origin", origin)
}
```

### 5.3 不要暴露敏感响应头

`Access-Control-Expose-Headers` 只应包含前端真正需要读取的头，不要暴露 `Set-Cookie`、`Authorization` 等敏感头。

## 6. 总结

go-zero 通过中间件机制处理 CORS，灵活且不侵入框架核心。核心要点：

1. **开发环境**：可以使用 `*` 允许所有来源，快速开发
2. **生产环境**：严格使用白名单，精确匹配域名
3. **携带 Cookie**：`Allow-Origin` 必须是精确域名，`Allow-Credentials: true`
4. **预检缓存**：设置合理的 `Max-Age`（如 3600 秒），减少预检请求开销
5. **反向代理**：优先在代理层统一配置 CORS，应用层可不处理
6. **安全第一**：不要用 `strings.Contains` 验证来源，必须精确匹配

配合 go-zero 的 [JWT 认证](jwt-auth-cn.md)和[限流机制](rate-limit-cn.md)，构建完整的 API 安全体系。
