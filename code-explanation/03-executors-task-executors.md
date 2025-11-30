# go-zero 源码解读（三）：批量聚合、防抖节流，三种任务执行器的奥秘

> 深入剖析 go-zero 任务执行器的实现原理：ChunkExecutor 通过双触发机制实现批量聚合，PeriodicalExecutor 用命令通道模式支持优雅关闭，DelayExecutor 用简洁设计实现防抖。掌握这三种执行器，让异步任务处理更高效。

## 1. 概述

在微服务开发中，我们经常需要处理各种异步任务，如批量写入日志、定期清理缓存、延迟执行任务等。go-zero 的 `executors` 包提供了三种高性能的任务执行器，帮助我们优雅地处理这些场景。

本文将深入剖析 go-zero 中三种任务执行器的实现原理：
- **ChunkExecutor** - 批量执行器
- **PeriodicalExecutor** - 周期执行器
- **DelayExecutor** - 延迟执行器

## 2. ChunkExecutor - 批量执行器

### 2.1 设计背景

在日志收集、指标上报等场景中，如果每产生一条数据就立即处理，会导致：
- 频繁的 I/O 操作，性能低下
- 网络请求次数过多，浪费资源
- 无法利用批量接口的性能优势

ChunkExecutor 的设计思想是**批量聚合**：将多个任务累积到一定数量或时间后，一次性批量执行。

### 2.2 核心结构

```go
// core/executors/chunkexecutor.go
type ChunkExecutor struct {
	execute Execute           // 批量执行函数
	container *chunkContainer // 任务容器
}

type chunkContainer struct {
	tasks         []Task        // 待执行任务队列
	execute       Execute       // 批量执行函数
	size          int           // 批量大小阈值
	interval      time.Duration // 时间间隔阈值
	lock          sync.Mutex    // 保护并发访问
	executionLock sync.Mutex    // 保护执行过程
	timer         *time.Timer   // 定时器
}

// Task 任务接口
type Task interface{}

// Execute 批量执行函数签名
type Execute func(tasks []Task)
```

### 2.3 工作原理

ChunkExecutor 采用**双触发机制**：

```
┌─────────────┐
│  Add Task   │
└──────┬──────┘
       │
       ▼
  ┌─────────┐
  │ Buffer  │ ◄─── 累积任务
  └────┬────┘
       │
       ├──► 数量达到阈值 ─────┐
       │                      │
       └──► 时间达到间隔 ─────┤
                              │
                              ▼
                      ┌───────────────┐
                      │ Batch Execute │
                      └───────────────┘
```

**触发条件**：
1. **数量触发**：任务数量达到 `size` 阈值
2. **时间触发**：距离上次执行超过 `interval` 时间

### 2.4 源码实现

#### 创建执行器

```go
func NewChunkExecutor(execute Execute, opts ...ChunkOption) *ChunkExecutor {
	container := &chunkContainer{
		execute:  execute,
		size:     defaultChunkSize,    // 默认 1024
		interval: defaultFlushInterval, // 默认 1 秒
	}

	for _, opt := range opts {
		opt(container)
	}

	executor := &ChunkExecutor{
		execute:   execute,
		container: container,
	}

	return executor
}
```

#### 添加任务

```go
func (ce *ChunkExecutor) Add(task Task) error {
	ce.container.addTask(task)
	return nil
}

func (c *chunkContainer) addTask(task Task) {
	c.lock.Lock()
	defer c.lock.Unlock()

	// 添加任务到缓冲区
	c.tasks = append(c.tasks, task)

	// 检查是否达到批量大小阈值
	if len(c.tasks) >= c.size {
		// 立即执行
		c.doExecute()
		return
	}

	// 启动或重置定时器
	c.ensureTimer()
}
```

#### 定时器管理

```go
func (c *chunkContainer) ensureTimer() {
	if c.timer == nil {
		// 首次创建定时器
		c.timer = time.AfterFunc(c.interval, c.execute)
	} else {
		// 重置定时器
		c.timer.Reset(c.interval)
	}
}
```

#### 批量执行

```go
func (c *chunkContainer) execute() {
	c.lock.Lock()
	defer c.lock.Unlock()

	// 停止定时器
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}

	// 执行任务
	c.doExecute()
}

func (c *chunkContainer) doExecute() {
	// 注意：此时已持有 c.lock

	if len(c.tasks) == 0 {
		return
	}

	// 复制任务列表
	tasks := c.tasks
	c.tasks = nil

	// 释放锁后执行，避免阻塞后续添加
	c.lock.Unlock()

	// 使用执行锁保护批量执行过程
	c.executionLock.Lock()
	defer c.executionLock.Unlock()

	// 执行批量任务
	c.execute(tasks)

	// 重新获取锁
	c.lock.Lock()
}
```

### 2.5 关键设计

#### 1. 双锁机制

```go
lock          sync.Mutex  // 保护 tasks 队列
executionLock sync.Mutex  // 保护执行过程
```

**为什么需要两把锁？**

- `lock`: 保护任务队列的并发访问
- `executionLock`: 保护批量执行过程，确保同一时刻只有一个批次在执行

**执行流程**：
```
1. 持有 lock
2. 复制 tasks，清空队列
3. 释放 lock（允许新任务继续添加）
4. 持有 executionLock
5. 执行批量任务
6. 释放 executionLock
```

#### 2. 定时器优化

使用 `time.AfterFunc` 而非 `time.Ticker`：
- 更灵活，可以随时 Reset
- 避免空轮询，只在有任务时才触发
- 自动停止，不需要手动管理 channel

### 2.6 使用示例

#### 批量日志写入

```go
// 日志批量写入器
type LogBatcher struct {
	executor *executors.ChunkExecutor
}

func NewLogBatcher() *LogBatcher {
	return &LogBatcher{
		executor: executors.NewChunkExecutor(
			func(tasks []executors.Task) {
				// 批量写入日志
				logs := make([]string, 0, len(tasks))
				for _, task := range tasks {
					logs = append(logs, task.(string))
				}

				// 一次性写入文件
				writeLogsToFile(logs)
			},
			executors.WithChunkBytes(1024*1024), // 1MB 触发
			executors.WithFlushInterval(time.Second), // 或 1 秒触发
		),
	}
}

func (l *LogBatcher) Log(msg string) {
	l.executor.Add(msg)
}
```

#### 批量指标上报

```go
// 指标批量上报
metricsExecutor := executors.NewChunkExecutor(
	func(tasks []executors.Task) {
		metrics := make([]Metric, 0, len(tasks))
		for _, task := range tasks {
			metrics = append(metrics, task.(Metric))
		}

		// 批量上报到监控系统
		reportMetrics(metrics)
	},
	executors.WithChunkBytes(100),           // 100 条触发
	executors.WithFlushInterval(time.Second), // 或 1 秒触发
)

// 添加指标
metricsExecutor.Add(Metric{Name: "qps", Value: 1000})
```

## 3. PeriodicalExecutor - 周期执行器

### 3.1 设计背景

PeriodicalExecutor 是 ChunkExecutor 的增强版，增加了以下特性：
- **自动容器清理**：执行后自动清空任务列表
- **异步执行**：不阻塞添加操作
- **优雅关闭**：支持 Wait 等待所有任务完成

**适用场景**：
- 定期刷新缓存
- 周期性数据同步
- 定时任务调度

### 3.2 核心结构

```go
type PeriodicalExecutor struct {
	commander chan interface{}  // 命令通道
	interval  time.Duration      // 执行间隔
	container *taskContainer     // 任务容器
	waitGroup sync.WaitGroup     // 等待组
}

type taskContainer struct {
	tasks    []Task            // 任务列表
	execute  Execute           // 执行函数
	interval time.Duration     // 执行间隔
	lock     sync.Mutex        // 并发锁
}

const (
	cmdSubmit = iota  // 提交任务命令
	cmdFlush          // 立即执行命令
)
```

### 3.3 工作原理

```
                  ┌──────────────┐
                  │   Add Task   │
                  └──────┬───────┘
                         │
                         ▼
                 ┌───────────────┐
                 │   Commander   │ ◄── 命令通道
                 │    Channel    │
                 └───────┬───────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  ┌─────────┐    ┌──────────┐    ┌──────────┐
  │ Submit  │    │  Flush   │    │  Timer   │
  └────┬────┘    └────┬─────┘    └────┬─────┘
       │              │                │
       └──────────────┴────────────────┘
                      │
                      ▼
              ┌───────────────┐
              │    Execute    │
              └───────────────┘
```

### 3.4 源码实现

#### 创建执行器

```go
func NewPeriodicalExecutor(interval time.Duration, execute Execute) *PeriodicalExecutor {
	executor := &PeriodicalExecutor{
		commander: make(chan interface{}, 1), // 带缓冲，避免阻塞
		interval:  interval,
		container: &taskContainer{
			execute:  execute,
			interval: interval,
		},
	}

	// 启动后台协程
	executor.waitGroup.Add(1)
	go executor.backgroundExecutor()

	return executor
}
```

#### 后台执行协程

```go
func (pe *PeriodicalExecutor) backgroundExecutor() {
	defer pe.waitGroup.Done()

	// 创建定时器
	ticker := time.NewTicker(pe.interval)
	defer ticker.Stop()

	for {
		select {
		case cmd := <-pe.commander:
			switch cmd {
			case cmdSubmit:
				// 收到提交命令，继续等待
				continue

			case cmdFlush:
				// 立即执行
				pe.container.execute(true)
				return // 退出协程

			default:
				// 忽略未知命令
			}

		case <-ticker.C:
			// 定时触发
			pe.container.execute(false)
		}
	}
}
```

#### 任务容器执行

```go
func (tc *taskContainer) execute(force bool) {
	tc.lock.Lock()

	// 获取当前任务
	tasks := tc.tasks
	tc.tasks = nil // 清空队列

	tc.lock.Unlock()

	if len(tasks) == 0 {
		return
	}

	// 执行批量任务
	tc.execute(tasks)
}
```

#### 添加任务

```go
func (pe *PeriodicalExecutor) Add(task Task) {
	// 加入任务队列
	pe.container.addTask(task)

	// 发送提交命令（非阻塞）
	select {
	case pe.commander <- cmdSubmit:
	default:
		// 通道满，忽略（已经有待处理的命令）
	}
}

func (tc *taskContainer) addTask(task Task) {
	tc.lock.Lock()
	defer tc.lock.Unlock()

	tc.tasks = append(tc.tasks, task)
}
```

#### 优雅关闭

```go
func (pe *PeriodicalExecutor) Flush() {
	// 发送立即执行命令
	pe.commander <- cmdFlush
}

func (pe *PeriodicalExecutor) Wait() {
	// 等待后台协程结束
	pe.waitGroup.Wait()
}
```

### 3.5 关键设计

#### 1. 命令通道模式

```go
commander chan interface{}  // 带缓冲的命令通道
```

**设计优点**：
- 解耦：添加任务和执行任务分离
- 非阻塞：使用 select + default 避免阻塞
- 灵活：可扩展不同类型的命令

#### 2. 定时器 vs Ticker

```go
ticker := time.NewTicker(pe.interval)
```

使用 Ticker 而非 AfterFunc：
- 需要周期性执行，Ticker 更合适
- 配合 select 使用，代码更清晰
- 不需要手动 Reset

#### 3. 自动清理机制

每次执行后自动清空任务列表：
```go
tasks := tc.tasks
tc.tasks = nil  // 自动清空
```

### 3.6 使用示例

#### 定期刷新缓存

```go
// 缓存刷新器
type CacheRefresher struct {
	executor *executors.PeriodicalExecutor
}

func NewCacheRefresher() *CacheRefresher {
	return &CacheRefresher{
		executor: executors.NewPeriodicalExecutor(
			time.Second*5, // 每 5 秒刷新一次
			func(tasks []executors.Task) {
				// 批量刷新缓存
				keys := make([]string, 0, len(tasks))
				for _, task := range tasks {
					keys = append(keys, task.(string))
				}

				// 从数据库加载最新数据
				refreshCache(keys)
			},
		),
	}
}

func (c *CacheRefresher) MarkDirty(key string) {
	c.executor.Add(key)
}

func (c *CacheRefresher) Shutdown() {
	c.executor.Flush()
	c.executor.Wait()
}
```

#### 日志批量落盘

```go
// 日志落盘器
logExecutor := executors.NewPeriodicalExecutor(
	time.Second, // 每秒落盘一次
	func(tasks []executors.Task) {
		logs := make([]string, 0, len(tasks))
		for _, task := range tasks {
			logs = append(logs, task.(string))
		}

		// 批量写入磁盘
		writeLogsToDisk(logs)
	},
)

// 记录日志
logExecutor.Add("User login: user123")

// 程序退出时，确保所有日志落盘
defer func() {
	logExecutor.Flush()
	logExecutor.Wait()
}()
```

## 4. DelayExecutor - 延迟执行器

### 4.1 设计背景

在某些场景下，我们需要延迟执行任务，但又希望：
- **合并相同任务**：相同 key 的任务只执行最后一次
- **自动取消**：新任务到来时取消旧任务
- **延迟执行**：任务在指定时间后执行

**典型场景**：
- 搜索框输入防抖
- 配置变更后延迟生效
- 延迟通知推送

### 4.2 核心结构

```go
type DelayExecutor struct {
	callback func()         // 回调函数
	delay    time.Duration  // 延迟时间
	lock     sync.Mutex     // 并发锁
	timer    *time.Timer    // 延迟定时器
}
```

### 4.3 工作原理

```
第一次触发：
  Trigger() ─┬─► 创建 Timer(delay) ─► delay 后 ─► Execute
             │
             └─► 记录 Timer

第二次触发（delay 内）：
  Trigger() ─┬─► 停止旧 Timer
             ├─► 创建新 Timer(delay) ─► delay 后 ─► Execute
             └─► 更新 Timer

效果：最后一次触发后延迟执行
```

**防抖原理**：
```
Input:  T0      T1      T2      T3      T4
        │       │       │       │       │
        ▼       ▼       ▼       ▼       ▼
Trigger ●───────●───────●───────●───────●

        ├───────┤ delay
        ├───────────────┤ delay
                ├───────────────┤ delay
                        ├───────────────┤ delay
                                ├───────────────┤ delay
                                                └──► Execute (只执行一次)
```

### 4.4 源码实现

#### 创建执行器

```go
func NewDelayExecutor(callback func(), delay time.Duration) *DelayExecutor {
	return &DelayExecutor{
		callback: callback,
		delay:    delay,
	}
}
```

#### 触发执行

```go
func (de *DelayExecutor) Trigger() {
	de.lock.Lock()
	defer de.lock.Unlock()

	// 停止旧的定时器
	if de.timer != nil {
		de.timer.Stop()
	}

	// 创建新的定时器
	de.timer = time.AfterFunc(de.delay, func() {
		de.callback()
	})
}
```

#### 立即执行

```go
func (de *DelayExecutor) Flush() {
	de.lock.Lock()
	defer de.lock.Unlock()

	// 停止定时器
	if de.timer != nil {
		de.timer.Stop()
		de.timer = nil
	}

	// 立即执行回调
	de.callback()
}
```

### 4.5 关键设计

#### 1. 简单高效

DelayExecutor 的实现非常简洁：
- 只有一个定时器
- 使用 `Stop()` + 重新创建实现重置
- 无需复杂的队列管理

#### 2. 防抖机制

每次 Trigger 都会重置定时器：
```go
timer.Stop()  // 取消旧任务
timer = time.AfterFunc(delay, callback)  // 创建新任务
```

这确保了只有最后一次触发会真正执行。

### 4.6 使用示例

#### 搜索框防抖

```go
// 搜索防抖
type SearchDebouncer struct {
	executor *executors.DelayExecutor
}

func NewSearchDebouncer(searchFunc func(keyword string)) *SearchDebouncer {
	var currentKeyword string

	return &SearchDebouncer{
		executor: executors.NewDelayExecutor(
			func() {
				// 延迟执行搜索
				searchFunc(currentKeyword)
			},
			300*time.Millisecond, // 300ms 防抖
		),
	}
}

func (s *SearchDebouncer) OnInput(keyword string) {
	currentKeyword = keyword
	s.executor.Trigger() // 每次输入都触发，但只有最后一次会执行
}

// 使用
debouncer := NewSearchDebouncer(func(keyword string) {
	fmt.Println("搜索:", keyword)
})

debouncer.OnInput("g")
debouncer.OnInput("go")
debouncer.OnInput("gol")
debouncer.OnInput("gola")
debouncer.OnInput("golan")
debouncer.OnInput("golang")
// 300ms 后只执行一次: "搜索: golang"
```

#### 配置热更新

```go
// 配置热更新器
type ConfigReloader struct {
	executor *executors.DelayExecutor
	config   *Config
}

func NewConfigReloader(config *Config) *ConfigReloader {
	reloader := &ConfigReloader{
		config: config,
	}

	reloader.executor = executors.NewDelayExecutor(
		func() {
			// 延迟重新加载配置
			reloader.reload()
		},
		time.Second, // 1 秒后生效
	)

	return reloader
}

func (cr *ConfigReloader) OnConfigChange() {
	// 配置文件变化时触发
	cr.executor.Trigger()
}

func (cr *ConfigReloader) reload() {
	// 重新加载配置
	newConfig := loadConfigFromFile()
	cr.config.Update(newConfig)
	fmt.Println("配置已更新")
}

// 使用场景：
// 用户频繁修改配置文件，但只在停止修改 1 秒后才重新加载
```

#### 批量保存

```go
// 文档自动保存
type AutoSaver struct {
	executor *executors.DelayExecutor
	document *Document
}

func NewAutoSaver(doc *Document) *AutoSaver {
	saver := &AutoSaver{
		document: doc,
	}

	saver.executor = executors.NewDelayExecutor(
		func() {
			// 保存文档
			saver.save()
		},
		2*time.Second, // 2 秒后保存
	)

	return saver
}

func (as *AutoSaver) OnEdit() {
	// 每次编辑都触发
	as.executor.Trigger()
}

func (as *AutoSaver) save() {
	// 保存到磁盘
	as.document.SaveToDisk()
	fmt.Println("文档已保存")
}

// 用户不断编辑文档，但只在停止编辑 2 秒后才保存
```

## 5. 三种执行器对比

| 特性 | ChunkExecutor | PeriodicalExecutor | DelayExecutor |
|------|---------------|-------------------|---------------|
| **触发方式** | 数量/时间双触发 | 周期性触发 | 延迟触发 |
| **任务合并** | ❌ 不合并 | ❌ 不合并 | ✅ 合并（防抖） |
| **自动清理** | ❌ 需手动管理 | ✅ 自动清理 | N/A |
| **异步执行** | ❌ 同步 | ✅ 异步 | ✅ 异步 |
| **优雅关闭** | ❌ 不支持 | ✅ Flush + Wait | ✅ Flush |
| **使用场景** | 批量写入、上报 | 定期刷新、同步 | 防抖、延迟执行 |

## 6. 最佳实践

### 6.1 选择合适的执行器

```go
// 场景 1：批量写日志（追求高吞吐）
chunkExecutor := executors.NewChunkExecutor(
	batchWrite,
	executors.WithChunkBytes(1000),
	executors.WithFlushInterval(time.Second),
)

// 场景 2：定期清理过期数据
periodicalExecutor := executors.NewPeriodicalExecutor(
	5*time.Minute,
	cleanExpiredData,
)

// 场景 3：搜索框防抖
delayExecutor := executors.NewDelayExecutor(
	search,
	300*time.Millisecond,
)
```

### 6.2 优雅关闭

```go
type Service struct {
	executor *executors.PeriodicalExecutor
}

func (s *Service) Shutdown() {
	// 立即执行剩余任务
	s.executor.Flush()

	// 等待执行完成
	s.executor.Wait()

	log.Println("Service shutdown gracefully")
}
```

### 6.3 错误处理

```go
executor := executors.NewChunkExecutor(
	func(tasks []executors.Task) {
		// 使用 defer 捕获 panic
		defer func() {
			if r := recover(); r != nil {
				log.Printf("Executor panic: %v", r)
			}
		}()

		// 批量处理
		for _, task := range tasks {
			if err := process(task); err != nil {
				log.Printf("Task failed: %v", err)
				// 记录失败任务，但继续处理其他任务
			}
		}
	},
)
```

### 6.4 性能优化

```go
// 1. 预分配容量
executor := executors.NewChunkExecutor(
	func(tasks []executors.Task) {
		items := make([]Item, 0, len(tasks))
		for _, task := range tasks {
			items = append(items, task.(Item))
		}
		batchProcess(items)
	},
	executors.WithChunkBytes(1000),
)

// 2. 合理设置批量大小
// - 太小：频繁执行，性能差
// - 太大：内存占用高，延迟大
// 建议：根据实际测试调整，一般 100-1000 之间
```

## 7. 源码精华

### 7.1 锁的粒度控制

```go
// ❌ 错误：持锁时间过长
func (c *container) execute() {
	c.lock.Lock()
	defer c.lock.Unlock()  // 整个执行过程都持锁

	// 执行可能很耗时
	c.executeFn(c.tasks)
	c.tasks = nil
}

// ✅ 正确：最小化持锁时间
func (c *container) execute() {
	c.lock.Lock()
	tasks := c.tasks
	c.tasks = nil
	c.lock.Unlock()  // 快速释放锁

	// 无锁执行
	c.executeFn(tasks)
}
```

### 7.2 非阻塞命令发送

```go
// ✅ 使用 select + default 避免阻塞
select {
case pe.commander <- cmdSubmit:
	// 发送成功
default:
	// 通道满，忽略（已有待处理命令）
}
```

### 7.3 定时器的选择

```go
// 一次性延迟：使用 AfterFunc
timer := time.AfterFunc(delay, callback)
timer.Reset(delay)  // 可重置

// 周期性触发：使用 Ticker
ticker := time.NewTicker(interval)
defer ticker.Stop()

select {
case <-ticker.C:
	// 周期性执行
}
```

## 8. 总结

go-zero 的 `executors` 包提供了三种精心设计的任务执行器：

1. **ChunkExecutor**：双触发机制（数量+时间），适合批量写入场景
2. **PeriodicalExecutor**：命令通道模式，支持优雅关闭，适合周期性任务
3. **DelayExecutor**：简洁的防抖实现，适合延迟执行场景

**核心设计思想**：
- 📦 **批量聚合**：提高 I/O 效率
- ⏰ **时间控制**：平衡延迟与性能
- 🔒 **并发安全**：细粒度锁控制
- 🎯 **职责单一**：每种执行器专注一个场景

**工程实践**：
- 合理选择执行器类型
- 关注优雅关闭
- 控制批量大小
- 做好错误处理

掌握这三种执行器，能够让我们在微服务开发中更优雅地处理异步任务，提升系统性能和用户体验。

---

**下一篇预告**：go-zero 源码解读（四）：fx - 函数式流处理的设计与实现

我们将深入探讨 go-zero 的流式处理框架，分析其惰性求值、并行处理等高级特性的实现原理。
