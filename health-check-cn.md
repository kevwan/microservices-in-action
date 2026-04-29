# go-zero 健康检查：生产级服务探针设计

## 前言

健康检查是云原生应用的基础设施。Kubernetes 的存活探针（Liveness）和就绪探针（Readiness）依赖健康检查端点来决定是否重启 Pod 或将流量路由到该实例。go-zero 提供了基础健康检查端点，并支持自定义检查逻辑。

## 1. go-zero 内置健康检查

go-zero 的 REST 服务会提供基础健康检查端点：

```
GET /healthz      → 服务存活检查（Liveness）
GET /readyz       → 服务就绪检查（Readiness）
```

```bash
# 测试内置端点
curl http://localhost:8080/healthz
# 响应: "OK"

curl http://localhost:8080/readyz
# 响应: "OK"
```

## 2. 自定义就绪检查

默认的就绪检查只确认 HTTP 服务已启动，生产环境需要检查依赖服务（数据库、Redis等）：

```go
// internal/handler/health/healthhandler.go
type HealthChecker struct {
    db    sqlx.SqlConn
    redis *redis.Redis
}

type HealthStatus struct {
    Status     string            `json:"status"`      // ok | degraded | unhealthy
    Components map[string]string `json:"components"`  // 各组件状态
    Timestamp  int64             `json:"timestamp"`
}

func (h *HealthChecker) ReadinessHandler() http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
        defer cancel()

        status := &HealthStatus{
            Status:     "ok",
            Components: make(map[string]string),
            Timestamp:  time.Now().Unix(),
        }

        // 检查数据库
        if err := h.checkDB(ctx); err != nil {
            status.Components["database"] = "unhealthy: " + err.Error()
            status.Status = "unhealthy"
        } else {
            status.Components["database"] = "ok"
        }

        // 检查 Redis
        if err := h.checkRedis(ctx); err != nil {
            status.Components["redis"] = "unhealthy: " + err.Error()
            // Redis 故障可能只是"降级"而不是"不可用"
            if status.Status == "ok" {
                status.Status = "degraded"
            }
        } else {
            status.Components["redis"] = "ok"
        }

        code := http.StatusOK
        if status.Status == "unhealthy" {
            code = http.StatusServiceUnavailable
        }
        httpx.WriteJsonCtx(r.Context(), w, code, status)
    }
}

func (h *HealthChecker) checkDB(ctx context.Context) error {
    var result int
    return h.db.QueryRowCtx(ctx, &result, "SELECT 1")
}

func (h *HealthChecker) checkRedis(ctx context.Context) error {
    _, err := h.redis.Ping()
    return err
}
```

## 3. 注册自定义健康检查路由

```go
// main.go
server := rest.MustNewServer(c.RestConf)
defer server.Stop()

checker := &HealthChecker{db: svcCtx.DB, redis: svcCtx.Redis}

// 覆盖默认就绪探针，添加自定义检查
server.AddRoute(rest.Route{
    Method:  http.MethodGet,
    Path:    "/readyz",
    Handler: checker.ReadinessHandler(),
})

handler.RegisterHandlers(server, serverCtx)
server.Start()
```

## 4. Kubernetes 探针配置

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: order-api
          image: order-api:latest
          ports:
            - containerPort: 8080

          # 存活探针：服务崩溃时 K8s 重启 Pod
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10   # 启动后等待10秒再检查
            periodSeconds: 15          # 每15秒检查一次
            timeoutSeconds: 5          # 超时时间
            failureThreshold: 3        # 连续失败3次才重启

          # 就绪探针：服务未就绪时 K8s 停止向其发送流量
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 5    # 启动后5秒开始检查
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3       # 连续失败3次才从 Service 中摘除
            successThreshold: 2       # 连续成功2次才重新加入

          # 启动探针：处理启动慢的服务（防止存活探针误杀）
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            failureThreshold: 30      # 给30*10=300秒启动时间
            periodSeconds: 10
```

## 5. 启动时的就绪状态管理

服务启动时需要等待所有依赖初始化完毕后才接受流量：

```go
// 使用 ready flag 控制就绪状态
var ready atomic.Bool

// 就绪探针
server.AddRoute(rest.Route{
    Method: http.MethodGet,
    Path:   "/readyz",
    Handler: func(w http.ResponseWriter, r *http.Request) {
        if !ready.Load() {
            w.WriteHeader(http.StatusServiceUnavailable)
            w.Write([]byte("initializing"))
            return
        }
        w.Write([]byte("OK"))
    },
})

server.Start() // 异步启动（实际上是 goroutine）

// 初始化耗时操作（加载缓存、预热连接等）
if err := warmUpCache(svcCtx); err != nil {
    log.Fatal(err)
}
if err := preConnectDB(svcCtx); err != nil {
    log.Fatal(err)
}

// 所有初始化完成，标记为就绪
ready.Store(true)
logx.Info("服务就绪，开始接受流量")
```

## 6. 健康检查响应格式

```json
// 健康时（200 OK）
{
    "status": "ok",
    "components": {
        "database": "ok",
        "redis": "ok"
    },
    "timestamp": 1715000000
}

// 降级时（200 OK，仍可服务但有组件异常）
{
    "status": "degraded",
    "components": {
        "database": "ok",
        "redis": "unhealthy: connection refused"
    },
    "timestamp": 1715000000
}

// 不可用时（503 Service Unavailable）
{
    "status": "unhealthy",
    "components": {
        "database": "unhealthy: context deadline exceeded",
        "redis": "ok"
    },
    "timestamp": 1715000000
}
```

## 7. 总结

| 探针类型 | 检查内容 | 失败后果 |
|---------|---------|---------|
| Liveness（存活）| HTTP 服务是否响应 | Pod 重启 |
| Readiness（就绪）| 依赖服务是否可用 | 从 Service 摘除，停止接收流量 |
| Startup（启动）| 服务是否启动完成 | 防止 Liveness 过早误杀慢启动服务 |

健康检查设计原则：
1. 存活探针尽量简单（只检查服务进程本身）
2. 就绪探针检查关键依赖（DB、Redis）
3. 设置合理的 timeout（建议 ≤ 3s）
4. 降级（degraded）≠ 不可用，可以继续服务

相关文章：
- [优雅停机：零中断滚动发布](graceful-shutdown-cn.md)
- [Prometheus 监控：打造生产级可观测性](prometheus-monitoring-cn.md)
