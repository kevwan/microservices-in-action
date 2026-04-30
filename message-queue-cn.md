# go-zero 消息队列：go-queue 异步任务处理

## 前言

在微服务架构中，并非所有操作都需要同步完成。用户下单后，发送短信通知、更新搜索索引、生成账单报表——这些操作如果放在下单请求的主链路中，不仅增加延迟，还会因为短信服务/搜索服务的故障影响下单成功率。

**消息队列**将这些操作解耦为异步任务：下单成功后投递一条消息，其他服务各自消费，互不影响。go-zero 提供了 **go-queue**，基于 Kafka 和 Beanstalkd 实现了高性能的消息队列客户端，与 go-zero 生态深度集成。

## 1. 消息队列的核心价值

```text
同步模式（耦合、脆弱）：                  异步模式（解耦、弹性）：

用户下单请求                              用户下单请求
    │                                         │
    ├─ 扣减库存  ─────────────────             ├─ 扣减库存
    ├─ 发送短信  （短信超时 → 下单失败❌）      ├─ 投递消息 ──── MQ ───▶ 短信服务（异步）
    ├─ 更新搜索  （搜索挂了 → 下单失败❌）      └─ 返回成功✅        ├──▶ 搜索服务（异步）
    └─ 生成账单  （串行 → 响应慢 ❌）                               └──▶ 账单服务（异步）
```

消息队列的三大优势：

- **解耦**：生产者不关心谁在消费，消费者故障不影响生产者
- **削峰**：高峰期消息积压在队列，消费者按能力消费，保护下游
- **异步**：主链路只做最关键的操作，非核心逻辑异步处理

## 2. 安装与配置 Kafka

```bash
# 使用 Docker Compose 启动 Kafka
cat > docker-compose.yml << 'EOF'
version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
EOF

docker compose up -d
```

安装 go-queue：

```bash
go get github.com/zeromicro/go-queue
```

## 3. 消息生产：投递消息

### 3.1 配置 Kafka 生产者

```yaml
# etc/config.yaml
KqOrderTopic:
  Brokers:
    - localhost:9092
  Topic: order-events      # Kafka Topic 名称
```

```go
// internal/config/config.go
type Config struct {
    rest.RestConf
    KqOrderTopic kq.KqConf
}
```

### 3.2 在 ServiceContext 中初始化生产者

```go
// internal/svc/servicecontext.go
import "github.com/zeromicro/go-queue/kq"

type ServiceContext struct {
    Config       config.Config
    OrderPusher  *kq.Pusher
}

func NewServiceContext(c config.Config) *ServiceContext {
    return &ServiceContext{
        Config:      c,
        OrderPusher: kq.NewPusher(c.KqOrderTopic.Brokers, c.KqOrderTopic.Topic),
    }
}
```

### 3.3 投递订单事件

```go
// internal/logic/order/createorderlogic.go
import (
    "encoding/json"
    "github.com/zeromicro/go-queue/kq"
)

// 消息结构体（与消费者共享，建议放在独立包中）
type OrderCreatedEvent struct {
    OrderId     string `json:"orderId"`
    UserId      int64  `json:"userId"`
    ProductId   int64  `json:"productId"`
    Quantity    int    `json:"quantity"`
    TotalAmount int64  `json:"totalAmount"`
    UserPhone   string `json:"userPhone"`
    CreatedAt   int64  `json:"createdAt"`
}

func (l *CreateOrderLogic) CreateOrder(req *types.CreateOrderReq) (*types.CreateOrderResp, error) {
    // 1. 核心业务：创建订单（同步）
    order, err := l.svcCtx.OrderModel.Insert(l.ctx, &model.Order{
        UserId:    getUserId(l.ctx),
        ProductId: req.ProductId,
        Quantity:  int64(req.Quantity),
        Amount:    calculateAmount(req.ProductId, req.Quantity),
        Status:    "pending",
    })
    if err != nil {
        return nil, err
    }

    // 2. 投递消息：非核心操作异步处理
    event := OrderCreatedEvent{
        OrderId:     fmt.Sprintf("%d", order.LastInsertId),
        UserId:      getUserId(l.ctx),
        ProductId:   req.ProductId,
        Quantity:    req.Quantity,
        TotalAmount: calculateAmount(req.ProductId, req.Quantity),
        UserPhone:   req.Phone,
        CreatedAt:   time.Now().Unix(),
    }
    
    payload, err := json.Marshal(event)
    if err != nil {
        // 注意：消息投递失败不应该让下单失败
        // 可以记录日志，后续通过定时任务补偿
        logx.WithContext(l.ctx).Errorf("投递订单消息失败: %v", err)
    } else {
        // Pusher.Push 是同步的，确保消息写入 Kafka 后才返回
        if err = l.svcCtx.OrderPusher.Push(l.ctx, string(payload)); err != nil {
            logx.WithContext(l.ctx).Errorf("Kafka Push 失败: %v", err)
            // 同上：消息投递失败不中断主流程，后续补偿
        }
    }

    return &types.CreateOrderResp{
        OrderId: fmt.Sprintf("%d", order.LastInsertId),
    }, nil
}
```

## 4. 消息消费：处理异步任务

### 4.1 短信通知服务

```go
// sms-service/main.go
package main

import (
    "context"
    "encoding/json"

    "github.com/zeromicro/go-queue/kq"
    "github.com/zeromicro/go-zero/core/conf"
    "github.com/zeromicro/go-zero/core/logx"
    "github.com/zeromicro/go-zero/core/service"
)

type Config struct {
    Kafka kq.KqConf
}

// OrderCreatedEvent 与生产者共享的消息结构
type OrderCreatedEvent struct {
    OrderId     string `json:"orderId"`
    UserId      int64  `json:"userId"`
    TotalAmount int64  `json:"totalAmount"`
    UserPhone   string `json:"userPhone"`
    CreatedAt   int64  `json:"createdAt"`
}

// SMSConsumer 实现 kq.ConsumeHandler 接口
type SMSConsumer struct {
    // 注入短信 SDK 等依赖
}

func (c *SMSConsumer) Consume(ctx context.Context, key, value string) error {
    var event OrderCreatedEvent
    if err := json.Unmarshal([]byte(value), &event); err != nil {
        // 解析失败：记录日志，返回 nil 跳过（避免无限重试）
        logx.WithContext(ctx).Errorf("消息解析失败: %v, payload: %s", err, value)
        return nil
    }

    // 发送短信
    msg := fmt.Sprintf("您的订单 %s 已创建，金额 %.2f 元，感谢您的购买！",
        event.OrderId, float64(event.TotalAmount)/100)
    
    if err := c.sendSMS(event.UserPhone, msg); err != nil {
        logx.WithContext(ctx).Errorf("短信发送失败: orderId=%s, phone=%s, err=%v",
            event.OrderId, event.UserPhone, err)
        // 返回 error：go-queue 会重试（根据配置的重试策略）
        return err
    }

    logx.WithContext(ctx).Infof("短信发送成功: orderId=%s, phone=%s",
        event.OrderId, event.UserPhone)
    return nil
}

func main() {
    var c Config
    conf.MustLoad("etc/config.yaml", &c)

    // 创建 Kafka 消费队列
    q := kq.MustNewQueue(c.Kafka, &SMSConsumer{})

    // 使用 ServiceGroup 管理消费者生命周期（支持优雅停机）
    group := service.NewServiceGroup()
    group.Add(q)
    defer group.Stop()

    group.Start()
}
```

### 4.2 消费者配置

```yaml
# sms-service/etc/config.yaml
Kafka:
  Brokers:
    - localhost:9092
  Topic: order-events
  Group: sms-consumer-group       # 消费者组（同一 Topic 可有多个消费者组）
  Processors: 8                   # 并发消费者数量（根据 CPU 核数调整）
  MinBytes: 1                     # 最小拉取字节数
  MaxBytes: 10485760              # 最大拉取字节数（10MB）
  Offset: first                   # 从最早的消息开始消费（first/last）
```

### 4.3 多消费者组：同一消息多方处理

同一个 Topic 可以有多个消费者组，每个消费者组都会**独立消费所有消息**：

```text
Kafka Topic: order-events
    │
    ├──── sms-consumer-group      → 短信服务（发送通知）
    ├──── search-consumer-group   → 搜索服务（更新索引）
    ├──── billing-consumer-group  → 账单服务（生成报表）
    └──── stats-consumer-group    → 统计服务（更新销量）
```

每个服务只需将自己的 `Group` 配置为不同的名称，即可独立消费。

## 5. 延迟队列（Beanstalkd）

对于需要延迟处理的任务（如订单超时自动取消），go-queue 还支持基于 Beanstalkd 的延迟队列：

### 5.1 安装 Beanstalkd

```bash
docker run -d --name beanstalkd -p 11300:11300 schickling/beanstalkd
```

### 5.2 投递延迟任务

```go
import "github.com/zeromicro/go-queue/dq"

// 配置
type Config struct {
    DqOrderTimeout dq.DqConf
}
```

```yaml
# etc/config.yaml
DqOrderTimeout:
  Beanstalks:
    - Endpoint: localhost:11300
      Tube: order-timeout          # Beanstalkd tube 名称
  Redis:
    Host: localhost:6379
    Type: node
```

```go
// 投递延迟任务：30 分钟后触发订单超时检查
producer := dq.NewProducer(c.DqOrderTimeout.Beanstalks)

// 序列化任务
payload := fmt.Sprintf(`{"orderId":"%s","action":"timeout_check"}`, orderId)

// 延迟 30 分钟执行
delay := 30 * time.Minute
_, err := producer.Delay([]byte(payload), delay)
if err != nil {
    logx.Errorf("延迟任务投递失败: %v", err)
}
```

### 5.3 消费延迟任务

```go
// order-timeout-service/main.go
consumer := dq.NewConsumer(c.DqOrderTimeout)
consumer.Consume(func(body []byte) {
    var task struct {
        OrderId string `json:"orderId"`
        Action  string `json:"action"`
    }
    if err := json.Unmarshal(body, &task); err != nil {
        logx.Errorf("任务解析失败: %v", err)
        return
    }
    
    // 检查订单是否已支付
    order, err := orderModel.FindOne(context.Background(), task.OrderId)
    if err != nil || order.Status != "pending" {
        return  // 订单已处理，跳过
    }
    
    // 超时取消订单
    orderModel.UpdateStatus(context.Background(), task.OrderId, "cancelled")
    logx.Infof("订单 %s 超时自动取消", task.OrderId)
})
```

## 6. 消息可靠性保障

### 6.1 生产者保障

| 策略 | 说明 | 实现方式 |
| ---- | ---- | ------- |
| 同步发送 | 等待 Kafka 确认写入后返回 | `Pusher.Push()` 默认同步 |
| 本地消息表 | 消息与业务数据在同一事务中保存 | 先写 DB，定时任务扫描发送 |
| 幂等生产 | 重试时不产生重复消息 | Kafka 开启 `enable.idempotence` |

### 6.2 消费者保障

```go
// 消费者幂等处理：使用 Redis 记录已处理的消息 ID
func (c *SMSConsumer) Consume(ctx context.Context, key, value string) error {
    // 使用 Kafka offset 或消息中的业务 ID 作为幂等键
    idempotentKey := fmt.Sprintf("sms:processed:%s", key)
    
    // 检查是否已处理
    existed, err := c.redis.SetNX(idempotentKey, "1", 24*time.Hour)
    if err != nil {
        return err  // Redis 故障，重试
    }
    if !existed {
        // 已处理过，跳过（幂等）
        return nil
    }
    
    // 发送短信
    return c.sendSMS(...)
}
```

### 6.3 死信队列

消费失败多次后，将消息转移到死信队列，避免阻塞正常消费：

```go
func (c *SMSConsumer) Consume(ctx context.Context, key, value string) error {
    retryCount := getRetryCount(ctx)  // 获取当前重试次数
    
    err := c.sendSMS(...)
    if err != nil {
        if retryCount >= 3 {
            // 超过最大重试次数：写入死信队列，人工处理
            c.deadLetterPusher.Push(ctx, value)
            logx.WithContext(ctx).Errorf("消息进入死信队列: %s", value)
            return nil  // 返回 nil 避免继续重试
        }
        return err  // 返回 error 触发重试
    }
    return nil
}
```

## 7. 监控消费进度

```bash
# 查看消费者组的 Lag（积压消息数量）
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group sms-consumer-group

# 输出示例：
# TOPIC          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# order-events   0          1000            1050            50
# order-events   1          2000            2000            0
```

在 Prometheus 中监控 Lag：

```promql
# 消费者 Lag 告警（积压超过 1000 条）
kafka_consumer_group_lag{group="sms-consumer-group"} > 1000
```

## 8. 总结

go-queue 为 go-zero 微服务提供了轻量、高效的消息队列解决方案：

| 场景 | 方案 | 特点 |
| ---- | ---- | ---- |
| 实时异步处理 | Kafka + kq | 高吞吐、消费者组隔离 |
| 延迟任务 | Beanstalkd + dq | 精准延迟、适合超时补偿 |
| 定时任务 | go-zero cron | 简单周期性任务 |

最佳实践：

1. 主链路只做最关键的操作，非核心操作通过消息队列异步化
2. 消费者实现幂等处理，保证消息被处理**至少一次**且不产生副作用
3. 消息投递失败不中断主业务流程，通过定时补偿兜底
4. 监控消费 Lag，及时扩容消费者应对积压

相关文章：[优雅停机](graceful-shutdown-cn.md) — go-queue 消费者在停机时会等待当前正在处理的消息完成后再退出。
