# go-zero 定时任务：cron 与后台 Worker

## 前言

微服务中有很多需要定期执行的任务：账单对账、过期数据清理、统计报表生成、库存同步等。go-zero 本身不提供独立 cron 调度器，但提供了 `threading` 等基础能力，通常与 `robfig/cron` 组合实现生产级定时任务。

## 1. go-zero + cron 的推荐组合

### 1.1 使用 go-zero 的 threading.GoSafe

go-zero 提供了 `threading.GoSafe` 用于安全启动后台 goroutine（自动 Panic 恢复）：

```go
import "github.com/zeromicro/go-zero/core/threading"

// 安全启动后台 goroutine（panic 会被 recover 并打印日志）
threading.GoSafe(func() {
    // 定时任务逻辑
    for {
        // ...
    }
})
```

### 1.2 集成 robfig/cron（推荐）

go-zero 项目通常结合 `robfig/cron` 库实现复杂的定时任务调度：

```bash
go get github.com/robfig/cron/v3
```

```go
// jobs/scheduler.go
package jobs

import (
    "github.com/robfig/cron/v3"
    "github.com/zeromicro/go-zero/core/logx"
    "github.com/zeromicro/go-zero/core/threading"
)

type Scheduler struct {
    c    *cron.Cron
    svc  *svc.ServiceContext
}

func NewScheduler(svc *svc.ServiceContext) *Scheduler {
    c := cron.New(
        cron.WithSeconds(),              // 支持秒级 cron 表达式
        cron.WithLogger(cron.PrintfLogger(logx.Infof)),
    )
    return &Scheduler{c: c, svc: svc}
}

func (s *Scheduler) Start() {
    // 注册所有定时任务
    s.registerJobs()
    s.c.Start()
    logx.Info("定时任务调度器已启动")
}

func (s *Scheduler) Stop() {
    ctx := s.c.Stop()
    <-ctx.Done()  // 等待正在执行的任务完成
    logx.Info("定时任务调度器已停止")
}

func (s *Scheduler) registerJobs() {
    // 每天凌晨 2 点对账
    s.c.AddFunc("0 0 2 * * *", s.reconcileOrders)

    // 每小时清理过期会话
    s.c.AddFunc("0 0 * * * *", s.cleanExpiredSessions)

    // 每 5 分钟同步库存
    s.c.AddFunc("0 */5 * * * *", s.syncInventory)

    // 每天早 8 点发送日报
    s.c.AddFunc("0 0 8 * * *", s.sendDailyReport)
}
```

### 1.3 在主服务中启动调度器

```go
// main.go
func main() {
    var c config.Config
    conf.MustLoad(*configFile, &c)

    svcCtx := svc.NewServiceContext(c)

    // 启动 HTTP 服务
    server := rest.MustNewServer(c.RestConf)
    defer server.Stop()

    // 启动定时任务调度器（与 HTTP 服务并行）
    scheduler := jobs.NewScheduler(svcCtx)
    scheduler.Start()
    defer scheduler.Stop()

    handler.RegisterHandlers(server, svcCtx)
    server.Start()
}
```

## 2. 具体定时任务实现

### 2.1 订单对账任务

```go
// jobs/reconcile.go
func (s *Scheduler) reconcileOrders() {
    ctx := context.Background()
    logx.WithContext(ctx).Info("开始订单对账...")
    start := time.Now()

    // 查询昨天的待对账订单
    yesterday := time.Now().AddDate(0, 0, -1)
    orders, err := s.svc.OrderModel.FindByDateAndStatus(ctx,
        yesterday.Format("2006-01-02"), "paid")
    if err != nil {
        logx.WithContext(ctx).Errorf("对账查询失败: %v", err)
        return
    }

    var successCount, failCount int
    for _, order := range orders {
        // 向支付平台查询实际支付状态
        payStatus, err := s.svc.PaymentClient.QueryOrder(ctx, order.PaymentId)
        if err != nil {
            logx.WithContext(ctx).Errorf("支付查询失败: orderId=%s, err=%v",
                order.Id, err)
            failCount++
            continue
        }

        // 状态不一致，触发异常处理
        if payStatus != order.Status {
            logx.WithContext(ctx).Warnf("对账异常: orderId=%s, dbStatus=%s, payStatus=%s",
                order.Id, order.Status, payStatus)
            s.svc.AlertService.Send(ctx, fmt.Sprintf("对账异常: 订单 %s", order.Id))
        }
        successCount++
    }

    logx.WithContext(ctx).Infow("对账完成",
        logx.Field("total", len(orders)),
        logx.Field("success", successCount),
        logx.Field("fail", failCount),
        logx.Field("duration", time.Since(start).Seconds()),
    )
}
```

### 2.2 过期数据清理任务

```go
// jobs/cleanup.go
func (s *Scheduler) cleanExpiredSessions() {
    ctx := context.Background()

    // 批量删除（避免一次性删除过多导致数据库压力）
    const batchSize = 1000
    var totalDeleted int

    for {
        // 每次删除最多 batchSize 条过期 Session
        deleted, err := s.svc.SessionModel.DeleteExpiredBatch(ctx,
            time.Now(), batchSize)
        if err != nil {
            logx.WithContext(ctx).Errorf("清理过期 Session 失败: %v", err)
            break
        }
        totalDeleted += deleted

        if deleted < batchSize {
            break  // 已删完所有过期数据
        }

        // 批次间休眠，避免持续占用数据库
        time.Sleep(100 * time.Millisecond)
    }

    if totalDeleted > 0 {
        logx.WithContext(ctx).Infof("清理过期 Session: 共删除 %d 条", totalDeleted)
    }
}
```

### 2.3 库存同步任务（分布式环境下的单节点执行）

在多实例部署时，同一定时任务可能在多个实例上同时执行。使用分布式锁确保任务只在一个实例上运行：

```go
func (s *Scheduler) syncInventory() {
    ctx := context.Background()

    // 获取分布式锁，确保多实例中只有一个在执行
    lock := redis.NewRedisLock(s.svc.Redis, "job:sync_inventory")
    lock.SetExpire(300) // 锁过期时间 5 分钟

    acquired, err := lock.Acquire()
    if err != nil || !acquired {
        // 其他实例正在执行，跳过
        return
    }
    defer lock.Release()

    logx.WithContext(ctx).Info("开始库存同步...")

    // 从 ERP 系统拉取最新库存数据
    inventories, err := s.svc.ERPClient.GetInventory(ctx)
    if err != nil {
        logx.WithContext(ctx).Errorf("ERP 库存拉取失败: %v", err)
        return
    }

    // 批量更新本地数据库
    for _, inv := range inventories {
        if err := s.svc.ProductModel.UpdateStock(ctx, inv.ProductId, inv.Stock); err != nil {
            logx.WithContext(ctx).Errorf("库存更新失败: productId=%d, err=%v",
                inv.ProductId, err)
        }
    }

    logx.WithContext(ctx).Infof("库存同步完成: 共同步 %d 条商品", len(inventories))
}
```

## 3. 独立的 Cron Service

对于任务较多的系统，可以将定时任务单独部署为一个独立服务：

```go
// cron-service/main.go
package main

import (
    "github.com/zeromicro/go-zero/core/conf"
    "github.com/zeromicro/go-zero/core/service"
)

func main() {
    var c config.Config
    conf.MustLoad("etc/config.yaml", &c)

    svcCtx := svc.NewServiceContext(c)

    // 使用 ServiceGroup 管理服务生命周期
    group := service.NewServiceGroup()

    // 将调度器作为一个 Service 加入 Group
    group.Add(jobs.NewSchedulerService(svcCtx))

    // 可以同时运行多个 Service
    group.Add(jobs.NewWorkerService(svcCtx))

    defer group.Stop()
    group.Start()
}
```

实现 `service.Service` 接口：

```go
// jobs/scheduler_service.go
type SchedulerService struct {
    scheduler *Scheduler
}

func (s *SchedulerService) Start() {
    s.scheduler.Start()
}

func (s *SchedulerService) Stop() {
    s.scheduler.Stop()
}
```

## 4. Cron 表达式速查

```
# 格式（开启 WithSeconds 后）：
# 秒 分 时 日 月 周

# 每分钟执行
*/1 * * * *

# 每天凌晨 2 点
0 0 2 * * *

# 每 5 分钟
0 */5 * * * *

# 每周一早上 9 点
0 0 9 * * 1

# 每月 1 号凌晨 0 点
0 0 0 1 * *

# 每天早 8 点、晚 8 点
0 0 8,20 * * *

# 工作日（周一至周五）早 9 点
0 0 9 * * 1-5
```

## 5. 监控定时任务

```go
// 为每个任务记录执行时间和状态到 Prometheus
var (
    jobDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Name: "cron_job_duration_seconds",
        Help: "定时任务执行时间",
    }, []string{"job"})

    jobErrors = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "cron_job_errors_total",
        Help: "定时任务失败次数",
    }, []string{"job"})
)

func instrumentedJob(name string, fn func()) func() {
    return func() {
        start := time.Now()
        defer func() {
            jobDuration.WithLabelValues(name).Observe(time.Since(start).Seconds())
        }()

        defer func() {
            if r := recover(); r != nil {
                jobErrors.WithLabelValues(name).Inc()
                logx.Errorf("定时任务 panic: job=%s, err=%v", name, r)
            }
        }()

        fn()
    }
}

// 注册时包装
s.c.AddFunc("0 0 2 * * *", instrumentedJob("reconcile_orders", s.reconcileOrders))
```

## 6. 总结

| 场景 | 方案 | 特点 |
|------|------|------|
| 简单定时任务 | `threading.GoSafe` + `time.Ticker` | 无外部依赖 |
| 复杂调度（cron 表达式） | `robfig/cron` | 灵活，支持分钟/秒级 |
| 多实例部署的单点任务 | 分布式锁 + cron | 防止重复执行 |
| 大量任务、独立运维 | 独立 cron service | 资源隔离，独立扩缩容 |
| 分布式任务调度 | Temporal / asynq | 更复杂场景 |

使用 go-zero 的 `service.ServiceGroup` 管理定时任务服务，可以享受与主服务相同的优雅停机能力，确保正在执行的任务完成后才退出。
