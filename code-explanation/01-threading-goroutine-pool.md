# go-zero 源码解读（一）：threading - 协程池与 Worker 模式

> 本文是 go-zero 源码解读系列的第一篇，我们将深入分析 `core/threading` 包的实现原理，学习如何优雅地管理 Go 协程的生命周期。

## 一、为什么需要协程池？

### 1.1 常见的协程管理问题

在 Go 开发中，我们经常会遇到以下问题：

```go
// ❌ 问题代码：协程泄漏
func handleRequests(requests []Request) {
    for _, req := range requests {
        go processRequest(req) // 无法知道何时完成，无法优雅关闭
    }
    // 函数返回，但协程可能还在运行
}

// ❌ 问题代码：无限制创建协程
func handleHighTraffic(requests chan Request) {
    for req := range requests {
        go processRequest(req) // 高并发时可能创建数万个协程
    }
}

// ❌ 问题代码：错误处理困难
func batchProcess(items []Item) error {
    for _, item := range items {
        go func(i Item) {
            if err := process(i); err != nil {
                // 如何将错误传递给调用者？
            }
        }(item)
    }
    return nil // 无法知道是否有错误发生
}
```

### 1.2 go-zero 的解决方案

go-zero 的 `threading` 包提供了三个核心组件：

- **TaskRunner**：控制并发数量的任务执行器
- **WorkerGroup**：有界任务队列的 Worker 池
- **RoutineGroup**：带等待和错误处理的协程组

## 二、源码结构概览

```
core/threading/
├── taskrunner.go       # 任务执行器
├── workergroup.go      # Worker 池
├── routinegroup.go     # 协程组
└── *_test.go          # 测试文件
```

## 三、TaskRunner 深度解析

### 3.1 设计思想

TaskRunner 的核心思想是：**限制同时运行的协程数量**，类似于有界线程池。

### 3.2 核心数据结构

```go
// 源码位置：core/threading/taskrunner.go

type TaskRunner struct {
    limitChan chan lang.PlaceholderType  // 用于控制并发数
}

func NewTaskRunner(concurrency int) *TaskRunner {
    return &TaskRunner{
        limitChan: make(chan lang.PlaceholderType, concurrency),
    }
}
```

**设计亮点**：
- 使用 channel 的容量特性自然地限制并发数
- `lang.PlaceholderType` 是空结构体，不占用内存
- 简单优雅，符合 Go 的设计哲学

### 3.3 核心方法实现

#### Schedule 方法

```go
func (r *TaskRunner) Schedule(task func()) {
    r.limitChan <- lang.Placeholder  // 获取令牌，满了会阻塞

    go func() {
        defer func() {
            <-r.limitChan  // 释放令牌
        }()

        task()
    }()
}
```

**执行流程**：
1. 向 `limitChan` 写入令牌，如果满了则阻塞
2. 获取令牌后启动协程执行任务
3. 任务完成后通过 defer 释放令牌

**优势分析**：
- ✅ 自动限流：并发数不会超过设定值
- ✅ 背压处理：调用方会感知到系统压力
- ✅ 资源保护：防止创建过多协程导致系统崩溃

### 3.4 使用示例

```go
package main

import (
    "fmt"
    "time"
    "github.com/zeromicro/go-zero/core/threading"
)

func main() {
    // 创建最多同时运行 3 个任务的 TaskRunner
    runner := threading.NewTaskRunner(3)

    // 提交 10 个任务
    for i := 0; i < 10; i++ {
        taskID := i
        runner.Schedule(func() {
            fmt.Printf("Task %d started\n", taskID)
            time.Sleep(time.Second)
            fmt.Printf("Task %d completed\n", taskID)
        })
    }

    // 等待所有任务完成
    runner.Wait()
}
```

**输出分析**：
- 前 3 个任务会立即开始
- 第 4 个任务会等待前面的任务完成
- 始终只有 3 个任务在并发执行

### 3.5 应用场景（需要控制并发数量）

1. **批量 API 调用**
```go
runner := threading.NewTaskRunner(10)
for _, userID := range userIDs {
    id := userID
    runner.Schedule(func() {
        user, err := fetchUserAPI(id)
        // 处理结果
    })
}
```

2. **文件批量处理**
```go
runner := threading.NewTaskRunner(5)
for _, file := range files {
    f := file
    runner.Schedule(func() {
        processFile(f)
    })
}
```

3. **数据库批量操作**
```go
runner := threading.NewTaskRunner(20)
for _, record := range records {
    r := record
    runner.Schedule(func() {
        db.Insert(r)
    })
}
```

## 四、WorkerGroup 深度解析

### 4.1 设计思想

WorkerGroup 是一个非常简单但实用的工具，它的核心思想是：**启动固定数量的 worker 协程，并行执行相同的任务函数**。

与 TaskRunner 的区别：

| 特性 | TaskRunner | WorkerGroup |
|------|-----------|-------------|
| 协程创建 | 每个任务创建一个协程 | 启动固定数量的协程 |
| 任务定义 | 每次 Schedule 传入不同任务 | 所有 worker 执行相同的 job |
| 等待机制 | 无等待（需手动处理） | 内置等待所有 worker 完成 |
| 适用场景 | 不同的任务需要并发执行 | 多个 worker 执行相同逻辑 |

### 4.2 核心数据结构

```go
// 源码位置：core/threading/workergroup.go

type WorkerGroup struct {
    job     func()  // 任务处理函数
    workers int     // worker 数量
}

func NewWorkerGroup(job func(), workers int) WorkerGroup {
    return WorkerGroup{
        job:     job,
        workers: workers,
    }
}
```

**设计亮点**：
- 极简设计，只有两个字段
- 职责单一，只负责启动多个相同的 worker

### 4.3 核心方法实现

```go
func (wg WorkerGroup) Start() {
    group := NewRoutineGroup()
    for i := 0; i < wg.workers; i++ {
        group.RunSafe(wg.job)
    }
    group.Wait()
}
```

**实现分析**：
1. 创建一个 RoutineGroup 用于管理协程
2. 启动指定数量的 worker，每个执行相同的 job 函数
3. 使用 RunSafe 保证 panic 不会导致程序崩溃
4. Wait 等待所有 worker 完成

**设计巧妙之处**：
- 复用了 RoutineGroup 的能力（等待、panic 恢复）
- 代码简洁，易于理解和维护
- Start() 会阻塞直到所有 worker 完成

### 4.4 使用示例

#### 示例 1：并发处理消息队列

```go
package main

import (
    "fmt"
    "time"
    "github.com/zeromicro/go-zero/core/threading"
)

func main() {
    messageQueue := make(chan string, 100)

    // 生产者：向队列写入消息
    go func() {
        for i := 0; i < 50; i++ {
            messageQueue <- fmt.Sprintf("message-%d", i)
        }
        close(messageQueue)
    }()

    // 创建 5 个 worker 并发消费消息
    wg := threading.NewWorkerGroup(func() {
        for msg := range messageQueue {
            fmt.Printf("Processing: %s\n", msg)
            time.Sleep(100 * time.Millisecond)
        }
    }, 5)

    // 启动并等待所有 worker 完成
    wg.Start()
    fmt.Println("All messages processed")
}
```

#### 示例 2：并发初始化多个资源

```go
func initializeResources() {
    resources := []string{"database", "redis", "mq", "cache", "logger"}
    index := 0

    // 3 个 worker 并发初始化资源
    wg := threading.NewWorkerGroup(func() {
        // 注意：这里需要使用原子操作或 channel 来分配任务
        // 这个示例仅为演示 WorkerGroup 的用法
        for index < len(resources) {
            resource := resources[index]
            index++
            initResource(resource)
        }
    }, 3)

    wg.Start()
}
```

### 4.5 实际应用场景

**场景 1：Web 爬虫**

```go
func crawlWebsites(urls []string) {
    urlChan := make(chan string, len(urls))
    for _, url := range urls {
        urlChan <- url
    }
    close(urlChan)

    // 10 个 worker 并发爬取
    wg := threading.NewWorkerGroup(func() {
        for url := range urlChan {
            crawl(url)
        }
    }, 10)

    wg.Start()
}
```

**场景 2：批量数据处理**

```go
func processBatch(items []Item) {
    itemChan := make(chan Item, len(items))
    for _, item := range items {
        itemChan <- item
    }
    close(itemChan)

    // 5 个 worker 并发处理
    wg := threading.NewWorkerGroup(func() {
        for item := range itemChan {
            processItem(item)
        }
    }, 5)

    wg.Start()
}
```

**场景 3：并行测试或基准测试**

```go
func runLoadTest() {
    // 模拟 100 个并发用户
    wg := threading.NewWorkerGroup(func() {
        for i := 0; i < 10; i++ {
            sendRequest()
        }
    }, 100)

    start := time.Now()
    wg.Start()
    duration := time.Since(start)

    fmt.Printf("Load test completed in %v\n", duration)
}
```

## 五、RoutineGroup 深度解析

### 5.1 设计目标

RoutineGroup 解决了以下痛点：
1. ✅ 等待所有协程完成
2. ✅ 捕获协程中的 panic
3. ✅ 收集协程的错误

### 5.2 核心数据结构

```go
// 源码位置：core/threading/routinegroup.go

type RoutineGroup struct {
    waitGroup sync.WaitGroup
}

func NewRoutineGroup() *RoutineGroup {
    return &RoutineGroup{}
}
```

### 5.3 核心方法

#### Run 方法

```go
func (g *RoutineGroup) Run(fn func()) {
    g.waitGroup.Add(1)

    go func() {
        defer g.waitGroup.Done()
        fn()
    }()
}
```

#### RunSafe 方法

```go
func (g *RoutineGroup) RunSafe(fn func()) {
    g.waitGroup.Add(1)

    go func() {
        defer g.waitGroup.Done()
        defer rescue.Recover()  // 捕获 panic
        fn()
    }()
}
```

#### Wait 方法

```go
func (g *RoutineGroup) Wait() {
    g.waitGroup.Wait()  // 等待所有协程完成
}
```

### 5.4 使用示例

```go
package main

import (
    "fmt"
    "time"
    "github.com/zeromicro/go-zero/core/threading"
)

func main() {
    group := threading.NewRoutineGroup()

    // 启动多个协程
    for i := 0; i < 5; i++ {
        taskID := i
        group.RunSafe(func() {
            fmt.Printf("Task %d running\n", taskID)
            time.Sleep(time.Second)

            if taskID == 2 {
                panic("task 2 panicked!")  // 会被捕获
            }
        })
    }

    // 等待所有协程完成
    group.Wait()
    fmt.Println("All tasks completed")
}
```

### 5.5 应用场景

**场景 1：并行初始化**
```go
group := threading.NewRoutineGroup()

group.RunSafe(func() { initDatabase() })
group.RunSafe(func() { initRedis() })
group.RunSafe(func() { initMQ() })

group.Wait()  // 等待所有初始化完成
```

**场景 2：并行数据处理**
```go
group := threading.NewRoutineGroup()

for _, data := range dataList {
    d := data
    group.RunSafe(func() {
        processData(d)
    })
}

group.Wait()  // 等待所有处理完成
```

## 六、三者对比与选择

### 6.1 核心差异对比

| 维度 | TaskRunner | WorkerGroup | RoutineGroup |
|------|-----------|-------------|--------------|
| 任务定义 | 每次传入不同任务 | 所有 worker 执行同一 job | 每次传入不同任务 |
| 协程数量 | 限制并发数 | 固定 worker 数量 | 无限制 |
| 阻塞行为 | 达到上限时阻塞 | Start() 阻塞到完成 | 不阻塞 |
| 等待机制 | 无内置等待 | 内置等待 | 内置等待 |
| Panic 处理 | 无 | 有（通过 RoutineGroup） | 有（RunSafe） |

### 6.2 使用场景选择

| 场景 | 推荐组件 | 理由 |
|------|---------|------|
| 限制并发执行不同任务 | TaskRunner | 控制并发数，任务各不相同 |
| 多个 worker 消费队列 | WorkerGroup | 多个协程执行相同消费逻辑 |
| 并行执行不同任务并等待 | RoutineGroup | 灵活启动任务，等待全部完成 |
| 批量 API 调用（限流） | TaskRunner | 避免过多并发请求 |
| 消息队列消费 | WorkerGroup | 固定数量 worker 消费 |
| 并行初始化多个服务 | RoutineGroup | 每个服务初始化逻辑不同 |
| 爬虫并发抓取 | WorkerGroup | 多个 worker 执行相同抓取逻辑 |

### 6.3 代码对比示例

**场景：处理 100 个不同的任务**

```go
// 使用 TaskRunner：限制同时最多 10 个并发
runner := threading.NewTaskRunner(10)
for i := 0; i < 100; i++ {
    taskID := i
    runner.Schedule(func() {
        processTask(taskID)  // 每个任务逻辑不同
    })
}

// 使用 RoutineGroup：不限并发，等待全部完成
group := threading.NewRoutineGroup()
for i := 0; i < 100; i++ {
    taskID := i
    group.RunSafe(func() {
        processTask(taskID)
    })
}
group.Wait()

// 使用 WorkerGroup：10 个 worker 从队列消费
taskChan := make(chan int, 100)
for i := 0; i < 100; i++ {
    taskChan <- i
}
close(taskChan)

wg := threading.NewWorkerGroup(func() {
    for taskID := range taskChan {
        processTask(taskID)
    }
}, 10)
wg.Start()
```

## 七、最佳实践与注意事项

### 7.1 TaskRunner 最佳实践

**✅ 推荐做法**

```go
// 根据资源特点设置合理的并发数
runner := threading.NewTaskRunner(runtime.NumCPU() * 2)

// 使用 context 支持取消
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

for _, task := range tasks {
    t := task
    runner.Schedule(func() {
        select {
        case <-ctx.Done():
            return
        default:
            processTask(t)
        }
    })
}
```

**❌ 避免的做法**

```go
// 不要设置过大的并发数
runner := threading.NewTaskRunner(10000)  // ❌ 可能创建过多协程

// 不要忘记捕获循环变量
for _, task := range tasks {
    runner.Schedule(func() {
        processTask(task)  // ❌ 闭包捕获问题
    })
}
```

### 7.2 WorkerGroup 最佳实践

**✅ 推荐做法**

```go
// 使用 channel 分发任务给多个 worker
taskChan := make(chan Task, 100)
resultChan := make(chan Result, 100)

// 生产者
go func() {
    for _, task := range tasks {
        taskChan <- task
    }
    close(taskChan)
}()

// 消费者：多个 worker 并发处理
wg := threading.NewWorkerGroup(func() {
    for task := range taskChan {
        result := processTask(task)
        resultChan <- result
    }
}, 10)

// 在另一个协程中等待
go func() {
    wg.Start()
    close(resultChan)
}()

// 收集结果
for result := range resultChan {
    handleResult(result)
}
```

**❌ 避免的做法**

```go
// 不要在 worker 函数外访问共享变量而不加锁
var counter int  // ❌ 数据竞争

wg := threading.NewWorkerGroup(func() {
    counter++  // ❌ 并发不安全
}, 10)
```

### 7.3 RoutineGroup 最佳实践

**✅ 推荐做法**

```go
// 使用 RunSafe 防止 panic
group := threading.NewRoutineGroup()

for _, task := range tasks {
    t := task
    group.RunSafe(func() {
        if err := processTask(t); err != nil {
            log.Error(err)
        }
    })
}

group.Wait()
```

**❌ 避免的做法**

```go
// 不要启动过多协程
group := threading.NewRoutineGroup()
for i := 0; i < 1000000; i++ {  // ❌ 可能耗尽资源
    group.RunSafe(func() {
        // ...
    })
}
group.Wait()
```

## 八、设计思想总结

### 8.1 核心设计原则

1. **简单性优先**
   - 使用 Go 原生特性（channel、goroutine、sync.WaitGroup）
   - 代码简洁，易于理解和维护
   - 避免过度设计

2. **职责单一**
   - TaskRunner：只负责限制并发数
   - WorkerGroup：只负责启动固定数量的 worker
   - RoutineGroup：只负责协程生命周期管理

3. **组合优于继承**
   - WorkerGroup 内部使用 RoutineGroup
   - 通过组合实现功能复用

4. **安全性保证**
   - RunSafe 捕获 panic
   - 提供优雅的等待机制
   - 防止协程泄漏

### 8.2 Go 哲学的体现

**"Do not communicate by sharing memory; instead, share memory by communicating"**
- TaskRunner 使用 channel 的容量特性控制并发
- WorkerGroup 建议通过 channel 分发任务
- 避免直接共享状态

**"Less is more"**
- WorkerGroup 只有两个字段，极简设计
- 每个组件功能单一，易于理解
- 通过组合实现复杂功能

**"Clear is better than clever"**
- 代码逻辑清晰直观
- 没有复杂的抽象和技巧
- 行为可预测

### 8.3 适用性分析

| 组件 | 优势 | 劣势 | 最适合 |
|------|------|------|--------|
| TaskRunner | 简单灵活，自动限流 | 无等待机制 | 需要控制并发的场景 |
| WorkerGroup | 极简设计，内置等待 | 所有 worker 执行相同逻辑 | 消费者模式 |
| RoutineGroup | 灵活，有等待和 panic 保护 | 不限制并发数 | 并行执行不同任务 |

## 九、进阶话题预告

在后续文章中，我们将探讨：

1. **executors** - 更高级的任务执行器
   - `ChunkExecutor`：批量执行器
   - `PeriodicalExecutor`：周期执行器
   - `DelayExecutor`：延迟执行器

2. **syncx** - 同步原语扩展
   - `SingleFlight`：防缓存击穿
   - `SharedCalls`：请求合并
   - `Pool`：对象池实现

3. **fx** - 函数式流处理
   - 如何用声明式方式处理并发
   - Stream API 设计理念

## 十、参考资料

- [go-zero 官方文档](https://go-zero.dev/)
- [Go 并发编程实战](https://golang.org/doc/effective_go#concurrency)
- [go-zero GitHub 仓库](https://github.com/zeromicro/go-zero)
- [Go 并发模式](https://go.dev/blog/pipelines)

---

## 总结

通过本文，我们深入分析了 go-zero `threading` 包的三个核心组件：

- **TaskRunner**：通过 channel 容量限制并发数，简单高效
- **WorkerGroup**：启动固定数量的 worker 执行相同任务，适合消费者模式
- **RoutineGroup**：管理协程生命周期，提供等待和 panic 保护

这些组件虽然简单，但充分体现了 go-zero 的工程化思维：

✅ **简单易用**：符合 Go 语言习惯，易于理解
✅ **职责单一**：每个组件专注解决一个问题
✅ **安全可靠**：内置 panic 保护，防止协程泄漏
✅ **场景清晰**：根据不同场景选择合适的组件

掌握这些基础组件，是深入理解 go-zero 框架的第一步！

---

**下一篇预告**：《go-zero 源码解读（二）：syncx - 高级同步原语》

我们将深入分析 `SingleFlight`、`SharedCalls` 等防击穿利器的实现原理！
