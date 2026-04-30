# go-zero 性能调优实战：从分析到优化

## 前言

性能问题往往在压测或生产流量增大时才暴露。go-zero 内置了 pprof 性能分析接入点，结合 Go 的性能分析工具链，可以系统地定位 CPU 热点、内存泄漏、goroutine 泄漏等问题。

## 1. 开启 pprof

go-zero 的 REST 服务支持集成 pprof：

```go
// main.go
import (
    _ "net/http/pprof"  // 注册 pprof 路由
    "net/http"
)

func main() {
    // 在独立端口暴露 pprof（不要暴露到公网！）
    go func() {
        logx.Info(http.ListenAndServe("localhost:6060", nil))
    }()

    // 正常启动 go-zero 服务
    server := rest.MustNewServer(c.RestConf)
    // ...
}
```

或者通过配置文件启用：

```yaml
# config.yaml
EnablePprof: true   # go-zero 会自动注册 /debug/pprof/ 路由
```

## 2. CPU 性能分析

### 2.1 采集 CPU Profile

```bash
# 采集 30 秒的 CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# 输出 pprof 文件
curl -o cpu.pprof http://localhost:6060/debug/pprof/profile?seconds=30
```

### 2.2 分析 CPU 热点

```bash
# 交互式分析
go tool pprof cpu.pprof

# 常用命令：
(pprof) top10           # 显示 CPU 占用最高的10个函数
(pprof) list FuncName   # 查看函数的行级 CPU 占用
(pprof) web             # 生成调用图（需要 graphviz）

# Web UI（推荐）
go tool pprof -http=:8090 cpu.pprof
```

### 2.3 典型 CPU 热点案例

```go
// ❌ 问题：每次请求都重新编译正则表达式
func ValidateEmail(email string) bool {
    re := regexp.MustCompile(`^[a-z0-9._%+\-]+@[a-z0-9.\-]+\.[a-z]{2,4}$`)
    return re.MatchString(email)
}

// ✅ 修复：包级变量预编译
var emailRegex = regexp.MustCompile(`^[a-z0-9._%+\-]+@[a-z0-9.\-]+\.[a-z]{2,4}$`)

func ValidateEmail(email string) bool {
    return emailRegex.MatchString(email)
}

// ❌ 问题：高频路径使用 fmt.Sprintf 格式化字符串
func BuildCacheKey(userId int64, productId int64) string {
    return fmt.Sprintf("user:%d:product:%d", userId, productId)
}

// ✅ 修复：使用 strings.Builder 或 strconv
func BuildCacheKey(userId int64, productId int64) string {
    var b strings.Builder
    b.WriteString("user:")
    b.WriteString(strconv.FormatInt(userId, 10))
    b.WriteString(":product:")
    b.WriteString(strconv.FormatInt(productId, 10))
    return b.String()
}
```

## 3. 内存分析

### 3.1 采集堆内存 Profile

```bash
# 堆内存快照（查看当前内存分配）
go tool pprof http://localhost:6060/debug/pprof/heap

# 查看分配次数（而不是大小）
go tool pprof -alloc_objects http://localhost:6060/debug/pprof/heap
```

### 3.2 常见内存问题

```go
// ❌ 问题：slice append 导致频繁扩容，大量临时内存
func ProcessItems(ids []int64) []Result {
    var results []Result  // 初始容量为0
    for _, id := range ids {
        results = append(results, processOne(id))  // 多次扩容
    }
    return results
}

// ✅ 修复：预分配容量
func ProcessItems(ids []int64) []Result {
    results := make([]Result, 0, len(ids))  // 预分配
    for _, id := range ids {
        results = append(results, processOne(id))
    }
    return results
}

// ❌ 问题：大对象使用后未及时置 nil，GC 无法回收
type Handler struct {
    largeCache map[string][]byte  // 很大的 map
}

// ✅ 修复：使用 sync.Pool 复用对象
var bufPool = sync.Pool{
    New: func() any {
        return make([]byte, 0, 4096)
    },
}

func ProcessRequest(data []byte) []byte {
    buf := bufPool.Get().([]byte)
    defer bufPool.Put(buf[:0])  // 归还时重置长度（保留容量）

    buf = append(buf, data...)
    // 处理...
    return buf
}
```

## 4. Goroutine 泄漏检测

```bash
# 查看 goroutine 数量和调用栈
curl http://localhost:6060/debug/pprof/goroutine?debug=1

# 采集 goroutine profile
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

```go
// ❌ 问题：goroutine 等待永远不会写入的 channel
func processAsync(ids []int64) {
    for _, id := range ids {
        ch := make(chan Result)  // unbuffered channel
        go func(id int64) {
            ch <- process(id)  // 如果没有读取方，goroutine 永久阻塞
        }(id)
        // 如果主 goroutine 提前返回，子 goroutine 泄漏
    }
}

// ✅ 修复：使用 context 控制生命周期
func processAsync(ctx context.Context, ids []int64) {
    ch := make(chan Result, len(ids))  // 缓冲 channel
    for _, id := range ids {
        id := id  // 避免闭包问题
        go func() {
            select {
            case ch <- process(id):
            case <-ctx.Done():  // context 取消时退出
            }
        }()
    }
}
```

## 5. 压测与基准测试

### 5.1 基准测试

```go
// BenchmarkBuildCacheKey 文件在 _test.go 中
func BenchmarkBuildCacheKeyFmt(b *testing.B) {
    b.ReportAllocs()  // 报告内存分配次数
    for i := 0; i < b.N; i++ {
        _ = fmt.Sprintf("user:%d:product:%d", int64(i), int64(i))
    }
}

func BenchmarkBuildCacheKeyBuilder(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        _ = BuildCacheKey(int64(i), int64(i))
    }
}
```

```bash
# 运行基准测试
go test -bench=BenchmarkBuildCacheKey -benchmem -count=5

# 对比两种实现
go test -bench=. -benchmem | tee result.txt
benchstat result.txt
```

### 5.2 HTTP 压测

```bash
# 使用 hey 压测
hey -n 10000 -c 100 -H "Authorization: Bearer $TOKEN" \
    http://localhost:8080/api/orders

# 结果解读
# Requests/sec: 吞吐量
# Average: 平均响应时间
# P99: 99分位响应时间
```

## 6. GC 调优

```bash
# 调整 GC 目标（触发 GC 的内存增长比例，默认100%）
# GOGC=200 表示内存增长到原来2倍时才触发GC，减少GC频率，但峰值内存更高
GOGC=200 ./order-api

# 设置软内存上限（Go 1.19+）
# 当内存接近上限时主动触发GC，防止OOM
GOMEMLIMIT=512MiB ./order-api
```

```go
// 在代码中设置
import "runtime/debug"

func init() {
    // 设置 512MB 内存上限
    debug.SetMemoryLimit(512 * 1024 * 1024)
}
```

## 7. go-zero 特有优化点

```yaml
# 1. 调整工作线程数（默认 = CPU 核数）
MaxConns: 10000    # 最大并发连接数

# 2. 限制请求体大小（防止大请求消耗内存）
MaxBytes: 1048576  # 1MB

# 3. 超时设置（快速释放资源）
Timeout: 5000  # 5秒
```

## 8. 总结

性能优化的方法论：

1. **测量**：用 pprof 找到真正的瓶颈，不要凭感觉优化
2. **CPU 热点**：正则预编译、减少字符串分配、避免反射
3. **内存**：预分配 slice、sync.Pool 复用、及时释放引用
4. **Goroutine**：context 控制生命周期，用 buffered channel 防阻塞
5. **GC**：GOGC 和 GOMEMLIMIT 根据业务特点调整

相关文章：
- [Continuous Profiling](continuous-profiling-cn.md)
- [Prometheus 监控：打造生产级可观测性](prometheus-monitoring-cn.md)
