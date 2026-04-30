# go-zero gRPC 流式传输：处理实时数据与大量数据

## 前言

普通 gRPC（一元 RPC）是请求-响应模式：发一次请求，收一次响应。但有些场景需要持续发送或接收数据：实时日志推送、文件分块上传、股票行情订阅、进度报告等。**gRPC 流式传输**正是为这类场景设计的，go-zero 完整支持四种 gRPC 通信模式。

## 1. 四种 gRPC 通信模式

| 模式 | 请求 | 响应 | 典型场景 |
|------|------|------|---------|
| **Unary（一元）** | 单次 | 单次 | 普通查询 |
| **Server Streaming** | 单次 | 流式 | 实时推送、导出数据 |
| **Client Streaming** | 流式 | 单次 | 文件上传、批量写入 |
| **Bidirectional Streaming** | 流式 | 流式 | 聊天、实时协作 |

## 2. Proto 定义

```protobuf
// stream.proto
syntax = "proto3";
package stream;
option go_package = "./stream";

// 一元 RPC（对比用）
message GetOrderReq { string orderId = 1; }
message GetOrderResp { string orderId = 1; string status = 2; }

// 服务端流：订阅订单状态变更
message WatchOrdersReq { int64 userId = 1; }
message OrderEvent {
    string orderId = 1;
    string status  = 2;
    int64  time    = 3;
}

// 客户端流：批量上传商品
message UploadProductReq {
    string name  = 1;
    int64  price = 2;
    int64  stock = 3;
}
message UploadProductResp {
    int32 total    = 1;
    int32 success  = 2;
    int32 failed   = 3;
}

// 双向流：实时聊天
message ChatMessage {
    int64  userId  = 1;
    string content = 2;
    int64  time    = 3;
}

service StreamService {
    // 一元
    rpc GetOrder(GetOrderReq) returns (GetOrderResp);

    // 服务端流：客户端发一个请求，服务端持续推送
    rpc WatchOrders(WatchOrdersReq) returns (stream OrderEvent);

    // 客户端流：客户端持续发送，服务端最后返回汇总
    rpc UploadProducts(stream UploadProductReq) returns (UploadProductResp);

    // 双向流
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}
```

```bash
# 生成代码
goctl rpc protoc stream.proto \
  --go_out=. --go-grpc_out=. --zrpc_out=. --style gozero
```

## 3. 服务端流实现

### 3.1 服务端：持续推送订单事件

```go
// internal/logic/watchorderslogic.go
func (l *WatchOrdersLogic) WatchOrders(
    in *stream.WatchOrdersReq,
    streamServer stream.StreamService_WatchOrdersServer,
) error {

    // 订阅订单变更事件（通过 Redis Pub/Sub 或消息队列）
    channel := fmt.Sprintf("order:events:%d", in.UserId)
    sub := l.svcCtx.Redis.Subscribe(streamServer.Context(), channel)
    defer sub.Close()

    for {
        select {
        case msg := <-sub.Channel():
            // 解析事件
            var event stream.OrderEvent
            if err := json.Unmarshal([]byte(msg.Payload), &event); err != nil {
                continue
            }

            // 推送给客户端
            if err := streamServer.Send(&event); err != nil {
                // 客户端断开连接
                logx.WithContext(streamServer.Context()).Infof(
                    "客户端断开: userId=%d, err=%v", in.UserId, err)
                return nil
            }

        case <-streamServer.Context().Done():
            // 请求超时或客户端主动断开
            return nil
        }
    }
}
```

### 3.2 客户端：接收服务端推送

```go
// 客户端调用
client := stream.NewStreamServiceClient(conn)

streamResp, err := client.WatchOrders(ctx, &stream.WatchOrdersReq{UserId: 42})
if err != nil {
    log.Fatal(err)
}

for {
    event, err := streamResp.Recv()
    if err == io.EOF {
        // 服务端关闭了流
        break
    }
    if err != nil {
        log.Printf("接收错误: %v", err)
        break
    }
    fmt.Printf("订单事件: orderId=%s, status=%s\n", event.OrderId, event.Status)
}
```

## 4. 客户端流实现

### 4.1 服务端：接收批量上传

```go
// internal/logic/uploadproductslogic.go
func (l *UploadProductsLogic) UploadProducts(
    stream stream.StreamService_UploadProductsServer,
) error {

    var total, success, failed int32

    for {
        req, err := stream.Recv()
        if err == io.EOF {
            // 客户端发送完毕，返回汇总结果
            return stream.SendAndClose(&stream.UploadProductResp{
                Total:   total,
                Success: success,
                Failed:  failed,
            })
        }
        if err != nil {
            return err
        }

        total++

        // 写入数据库
        _, err = l.svcCtx.ProductModel.Insert(stream.Context(), &model.Product{
            Name:  req.Name,
            Price: req.Price,
            Stock: req.Stock,
        })
        if err != nil {
            logx.WithContext(stream.Context()).Errorf("商品写入失败: %v", err)
            failed++
        } else {
            success++
        }
    }
}
```

### 4.2 客户端：分块上传

```go
// 客户端：分块上传大量商品
uploadStream, err := client.UploadProducts(ctx)
if err != nil {
    log.Fatal(err)
}

products := loadProductsFromCSV("products.csv") // 假设有 10000 条
for _, p := range products {
    if err := uploadStream.Send(&stream.UploadProductReq{
        Name:  p.Name,
        Price: p.Price,
        Stock: p.Stock,
    }); err != nil {
        log.Printf("发送失败: %v", err)
        break
    }
}

// 发送完毕，等待服务端返回汇总
resp, err := uploadStream.CloseAndRecv()
if err != nil {
    log.Fatal(err)
}
fmt.Printf("上传完成: 总计=%d, 成功=%d, 失败=%d\n",
    resp.Total, resp.Success, resp.Failed)
```

## 5. 双向流实现

```go
// 服务端：聊天室
func (l *ChatLogic) Chat(stream stream.StreamService_ChatServer) error {
    userId, _ := stream.Context().Value("userId").(int64)
    room := l.svcCtx.ChatRoom

    // 注册到聊天室
    msgChan := room.Join(userId)
    defer room.Leave(userId)

    // 并发：接收客户端消息 + 推送其他人的消息
    errChan := make(chan error, 1)

    // goroutine 1: 接收客户端发来的消息，广播给其他人
    go func() {
        for {
            msg, err := stream.Recv()
            if err != nil {
                errChan <- err
                return
            }
            room.Broadcast(userId, msg)
        }
    }()

    // goroutine 2: 推送其他人的消息给客户端
    go func() {
        for msg := range msgChan {
            if err := stream.Send(msg); err != nil {
                errChan <- err
                return
            }
        }
    }()

    select {
    case err := <-errChan:
        if err == io.EOF {
            return nil
        }
        return err
    case <-stream.Context().Done():
        return nil
    }
}
```

## 6. 流式传输的注意事项

### 6.1 背压控制

```go
// 服务端推送速度过快时，客户端处理不过来会导致内存积压
// 使用带缓冲的 channel 并控制推送速率
func (l *WatchOrdersLogic) WatchOrders(...) error {
    // 限制推送速率：每秒最多 100 条
    limiter := rate.NewLimiter(rate.Limit(100), 10)

    for event := range eventChan {
        if err := limiter.Wait(streamServer.Context()); err != nil {
            return nil
        }
        streamServer.Send(event)
    }
    return nil
}
```

### 6.2 心跳保活

```go
// 长时间无数据推送时，发送心跳防止连接被网关断开
ticker := time.NewTicker(30 * time.Second)
defer ticker.Stop()

for {
    select {
    case event := <-eventChan:
        streamServer.Send(event)
    case <-ticker.C:
        // 发送心跳
        streamServer.Send(&stream.OrderEvent{Type: "ping"})
    case <-streamServer.Context().Done():
        return nil
    }
}
```

### 6.3 错误处理与重连

```go
// 客户端：自动重连
func subscribeWithRetry(ctx context.Context, client stream.StreamServiceClient, userId int64) {
    for {
        if err := subscribe(ctx, client, userId); err != nil {
            if ctx.Err() != nil {
                return  // 主动取消，退出
            }
            logx.Errorf("流断开，3秒后重连: %v", err)
            time.Sleep(3 * time.Second)
        }
    }
}
```

## 7. 与 WebSocket 的对比

| 特性 | gRPC Streaming | WebSocket |
|------|---------------|-----------|
| 协议 | HTTP/2 | HTTP/1.1 升级 |
| 类型安全 | ✅ Proto 定义 | ❌ 手动 JSON |
| 背压支持 | ✅ 内置 | ❌ 需自实现 |
| 浏览器支持 | ❌ 需 grpc-web | ✅ 原生 |
| 服务间通信 | ✅ 推荐 | ❌ 不适合 |

## 8. 总结

gRPC 流式传输让 go-zero 微服务能够处理实时数据场景：

- **Server Streaming**：适合服务端推送（消息通知、实时监控）
- **Client Streaming**：适合大量数据上传（批量导入、日志收集）
- **Bidirectional Streaming**：适合双向实时交互（聊天、游戏、协同编辑）

关键注意事项：背压控制、心跳保活、客户端自动重连，这三点决定了流式服务的稳定性。
