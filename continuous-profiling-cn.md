在微服务架构日益复杂、业务流量不断攀升的背景下，系统的稳定性成为我们追求的核心目标。而性能问题的排查，往往需要结合指标监控、日志、tracing，还少不了最难搞的 CPU/内存 Profiling。

现在，go-zero 支持原生集成 Continuous Profiling（持续性能分析），通过集成 Pyroscope，你可以方便地在生产环境中 实时采集性能数据，做到：

✅ 定位性能瓶颈
✅ 追踪 CPU/内存/协程异常
✅ 分析线上热点函数
✅ 降低系统维护成本

## 为什么使用持续 Profiling？

传统的 `pprof` 工具虽然强大，但使用成本高：手动触发、文件下载、手动分析，难以做到自动化、实时性、对用户透明。

而「持续 Profiling」具有以下优势：
- **动态采集性能数据**：支持按 CPU 利用率门限触发，也可持续上报
- **精确定位异常代码路径**：通过火焰图聚焦问题函数
- **对开发者透明**：无需侵入业务代码
- **支持 Grafana 可视化集成**

## 本地部署 Pyroscope 可视化服务

使用 Docker 一键启动：

```bash
docker pull grafana/pyroscope
docker run -it -p 4040:4040 grafana/pyroscope
```

访问 `http://localhost:4040` 即可查看火焰图等可视化分析数据。

详细参考：https://grafana.com/docs/pyroscope/latest/get-started/

## 如何启用？

### 1. 快速创建示例项目

> 请确保 go-zero 版本 >= v1.8.4

使用 goctl 快速创建一个 HTTP 服务器项目：

```bash
# 创建示例项目
goctl quickstart -t mono

# 进入项目目录
cd greet/api
```

这将生成一个完整的 go-zero 单体应用示例，包含基本的 HTTP API 和配置文件。

### 2. 配置文件启用 Profiling

在生成的 `etc/greet.yaml` 配置文件中添加 Profiling 配置：

```yaml
Name: ping
Host: localhost
Port: 8888
Log:
  Level: error
# 添加 Profiling 配置
Profiling:
  ServerAddr: http://localhost:4040  # 必须项
  CpuThreshold: 0                    # 设置为 0 表示持续采集，便于演示
```

默认配置下，设置 `CpuThreshold: 0` 表示持续采集性能数据，便于演示和测试 Profiling 功能。

### 3. 支持参数说明

| 参数名 | 默认值 | 含义 |
|--------|---------|------|
| CpuThreshold | 700 | 即 70%，超过触发采集；为 0 表示持续采集 |
| UploadDuration | 15s | 上报间隔 |
| ProfilingDuration | 2m | 每次采集时长 |
| ProfileType | | 支持 CPU、内存、协程、互斥锁等类型 |

可以通过配置控制采集内容：

```go
ProfileType struct {
    CPU        bool `json:",default=true"`
    Memory     bool `json:",default=true"`
    Goroutines bool `json:",default=true"`
    Mutex      bool `json:",default=false"` // 会影响性能，默认关闭
    Block      bool `json:",default=false"` // 会影响性能，默认关闭
}
```

`pyroscope` 中可以看到采集的性能指标：
![](https://cdn.learnku.com/uploads/images/202506/11/73865/lZHny8citu.jpg!large)

### 4. 模拟CPU负载测试

为了更好地演示 Profiling 效果，我们先在 `internal/logic/pinglogic.go` 中添加一些CPU密集型操作：

```go
package logic

import (
	"context"

	"greet/api/internal/svc"
	"greet/api/internal/types"

	"github.com/zeromicro/go-zero/core/logx"
)

type PingLogic struct {
	logx.Logger
	ctx    context.Context
	svcCtx *svc.ServiceContext
}

func NewPingLogic(ctx context.Context, svcCtx *svc.ServiceContext) *PingLogic {
	return &PingLogic{
		Logger: logx.WithContext(ctx),
		ctx:    ctx,
		svcCtx: svcCtx,
	}
}

func (l *PingLogic) Ping() (resp *types.Resp, err error) {
	// 模拟CPU密集型操作
	simulateCPULoad()

	return &types.Resp{
		Msg: "pong",
	}, nil
}

// 模拟CPU负载的函数
func simulateCPULoad() {
	for i := 0; i < 1000000; i++ {
		_ = i * i * i
	}
}
```

### 5. 启动服务测试

启动服务并测试 Profiling 功能：

```bash
# 启动服务
go run greet.go -f etc/greet.yaml

# 在另一个终端发送请求产生负载
# hey 是个压测工具
hey -c 100 -z 60m "http://localhost:8888/ping"
```

由于设置了 `CpuThreshold: 0`，服务启动后会立即开始持续采集性能数据并上报到 Pyroscope。

### 6. 查看CPU负载Profiling图

访问 http://localhost:4040 ，在 Pyroscope 界面中可以看到实时的性能数据：

![](https://cdn.learnku.com/uploads/images/202506/11/73865/fTno1IUZcr.jpg!large)

火焰图中可以清楚地看到：
- `simulateCPULoad` 函数占用了大量CPU时间
- `rand.Intn` 和相关的随机数生成函数调用频繁
- 可以精确定位到具体的代码热点

点击 `simulateCPULoad` 可以呈现调用所在位置，如图：
![](https://cdn.learnku.com/uploads/images/202506/11/73865/wFripAotUx.jpg!large)

### 7. 移除模拟负载代码

现在我们移除模拟CPU负载的代码，将 `internal/logic/pinglogic.go` 恢复为简单版本：

```go
func (l *PingLogic) Ping() (resp *types.Response, err error) {
	return &types.Response{
		Message: "pong",
	}, nil
}

// 删除 simulateCPULoad 函数
```

重新启动服务后，再次查看 Pyroscope：

![](https://cdn.learnku.com/uploads/images/202506/11/73865/dmk51S21gs.jpg!large)

可以看到CPU使用率显著降低，主要的性能消耗集中在HTTP处理和JSON序列化等正常操作上。

## 使用场景推荐

- CPU 使用突然升高，无法重现？
- 有内存泄漏，定位不到是哪段逻辑？
- 想知道服务 QPS 提高后瓶颈在哪？

配合 go-zero 的 continuous profiling，你可以：
- 快速回溯当时执行路径
- 可视化展示性能变化趋势
- 实现运维与研发协同分析

## 总结

持续 Profiling 的引入，是 go-zero 在稳定性方面的重要升级。我们将继续优化系统可观测能力，让开发者 **更早发现问题、更快解决问题、更少线上事故**。

