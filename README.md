# microservices-in-action

The collection of articles on design principles and best practices in microservices with go-zero.

## Table of contents

### 可靠性与弹性

- [自适应降载](adaptive-shedding-cn.md)
- [自适应熔断](breaker-cn.md)
- [HTTP 超时控制与防 DoS 攻击](timeout-dosattack-cn.md)
- [限流机制详解：从配额设计到分布式治理](rate-limit-cn.md)
- [重试与退避：从可用性收益到重试风暴治理](retry-backoff-cn.md)
- [幂等性设计：杜绝重复操作的副作用](idempotency-cn.md)

### 可观测性

- [Continuous Profiling](continuous-profiling-cn.md)
- [分布式链路追踪：OpenTelemetry 实战](tracing-cn.md)
- [Prometheus 监控：从指标采集到告警设计](prometheus-monitoring-cn.md)
- [logx 结构化日志最佳实践](logx-guide-cn.md)
- [日志脱敏](log-desensitization-cn.md)
- [性能调优实战：从分析到优化](performance-tuning-cn.md)

### 数据层

- [数据库读写分离](db-readwrite-split-cn.md)
- [数据库事务：保证数据一致性](db-transaction-cn.md)
- [sqlx 数据库操作详解](sqlx-guide-cn.md)
- [缓存设计最佳实践：从单机到分布式](cache-design-cn.md)
- [Redis 客户端：从基础操作到高级用法](redis-client-cn.md)
- [分布式锁：基于 Redis 的并发控制](distributed-lock-cn.md)
- [分页查询：游标分页与偏移分页](pagination-cn.md)

### API 与 HTTP

- [JWT 认证与鉴权：从 Token 校验到权限治理](jwt-auth-cn.md)
- [CORS 跨域处理：原理与最佳实践](cors-cn.md)
- [请求参数校验：从基础到自定义规则](request-validation-cn.md)
- [文件上传：本地存储与对象存储](file-upload-cn.md)
- [错误处理与统一响应：打造一致的 API 体验](error-handling-cn.md)
- [中间件进阶：链式调用、复用与最佳实践](middleware-advanced-cn.md)

### gRPC 与服务通信

- [gRPC 拦截器：统一处理横切关注点](grpc-interceptors-cn.md)
- [gRPC 流式传输：处理实时数据与大量数据](grpc-streaming-cn.md)
- [WebSocket：实时双向通信](websocket-cn.md)
- [负载均衡：P2C 算法深度解析](load-balancing-p2c-cn.md)

### 服务治理

- [服务注册与发现：从配置连接到生产治理](service-discovery-cn.md)
- [优雅停机：从进程退出到流量切换闭环](graceful-shutdown-cn.md)
- [服务健康检查端点](health-check-cn.md)
- [定时任务：cron 与后台 Worker](cron-job-cn.md)
- [go-queue 异步任务处理](message-queue-cn.md)

### 安全

- [TLS/mTLS：服务间双向认证](tls-cn.md)

### 配置与代码生成

- [配置管理：多环境支持与动态加载](config-management-cn.md)
- [goctl 代码生成：从零到微服务的极速体验](goctl-guide-cn.md)

### 测试与部署

- [单元测试与集成测试](testing-guide-cn.md)
- [CI/CD 流水线：GitHub Actions 自动化部署](cicd-pipeline-cn.md)

### AI 与前沿

- [让大模型成为你的 go-zero 专家](ai-native-framework-cn.md)
