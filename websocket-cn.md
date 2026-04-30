# go-zero WebSocket：实时双向通信

## 前言

HTTP 是请求-响应协议，服务端无法主动推送消息给客户端。WebSocket 在单个 TCP 连接上实现了全双工通信，适合实时聊天、在线协作、实时通知、股票行情等场景。go-zero 的 REST 框架支持 WebSocket 升级，配合 `gorilla/websocket` 库可以构建完整的实时通信服务。

## 1. 依赖安装

```bash
go get github.com/gorilla/websocket
```

## 2. 基础 WebSocket 连接

### 2.1 服务端

```go
// internal/handler/ws/chathandler.go
import "github.com/gorilla/websocket"

var upgrader = websocket.Upgrader{
    ReadBufferSize:  4096,
    WriteBufferSize: 4096,
    // CheckOrigin：生产环境需要严格校验来源
    CheckOrigin: func(r *http.Request) bool {
        origin := r.Header.Get("Origin")
        // 允许的域名白名单
        allowed := map[string]bool{
            "https://example.com":  true,
            "https://www.example.com": true,
        }
        return allowed[origin]
    },
}

func ChatHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // JWT 认证
        userId, err := authutil.GetUserId(r.Context())
        if err != nil {
            http.Error(w, "未授权", http.StatusUnauthorized)
            return
        }

        // HTTP 升级到 WebSocket
        conn, err := upgrader.Upgrade(w, r, nil)
        if err != nil {
            logx.Errorf("WebSocket 升级失败: %v", err)
            return
        }

        // 注册客户端
        client := NewClient(conn, userId, svcCtx)
        svcCtx.Hub.Register(client)
        defer svcCtx.Hub.Unregister(client)

        // 启动读写
        go client.writePump()
        client.readPump()
    }
}
```

### 2.2 Client 连接管理

```go
// internal/ws/client.go
const (
    writeWait      = 10 * time.Second      // 写超时
    pongWait       = 60 * time.Second      // 等待 pong 超时
    pingPeriod     = (pongWait * 9) / 10   // 发送 ping 间隔（54s）
    maxMessageSize = 4096                   // 最大消息大小（字节）
)

type Client struct {
    conn   *websocket.Conn
    userId int64
    hub    *Hub
    send   chan []byte  // 待发送消息缓冲
}

func NewClient(conn *websocket.Conn, userId int64, svcCtx *svc.ServiceContext) *Client {
    return &Client{
        conn:   conn,
        userId: userId,
        hub:    svcCtx.Hub,
        send:   make(chan []byte, 256),
    }
}

// readPump：读取客户端消息（在 goroutine 中运行）
func (c *Client) readPump() {
    defer func() {
        c.hub.Unregister(c)
        c.conn.Close()
    }()

    c.conn.SetReadLimit(maxMessageSize)
    c.conn.SetReadDeadline(time.Now().Add(pongWait))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    for {
        _, message, err := c.conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err,
                websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
                logx.Errorf("WebSocket 异常断开: userId=%d, %v", c.userId, err)
            }
            break
        }

        // 处理消息
        c.hub.Broadcast(&Message{
            From:    c.userId,
            Content: message,
        })
    }
}

// writePump：向客户端写消息（在独立 goroutine 中运行）
func (c *Client) writePump() {
    ticker := time.NewTicker(pingPeriod)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case message, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if !ok {
                // Hub 关闭了 channel
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            w, err := c.conn.NextWriter(websocket.TextMessage)
            if err != nil {
                return
            }
            w.Write(message)

            // 将缓冲区中的消息一并发送（减少系统调用）
            n := len(c.send)
            for i := 0; i < n; i++ {
                w.Write([]byte{'\n'})
                w.Write(<-c.send)
            }

            if err := w.Close(); err != nil {
                return
            }

        case <-ticker.C:
            // 发送心跳
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}
```

## 3. 聊天室 Hub

```go
// internal/ws/hub.go
type Message struct {
    From    int64  `json:"from"`
    RoomId  string `json:"roomId"`
    Content []byte `json:"content"`
}

type Hub struct {
    // roomId → clients
    rooms      map[string]map[*Client]bool
    broadcast  chan *Message
    register   chan *Client
    unregister chan *Client
    mu         sync.RWMutex
}

func NewHub() *Hub {
    h := &Hub{
        rooms:      make(map[string]map[*Client]bool),
        broadcast:  make(chan *Message, 256),
        register:   make(chan *Client),
        unregister: make(chan *Client),
    }
    go h.run()
    return h
}

func (h *Hub) run() {
    for {
        select {
        case client := <-h.register:
            h.mu.Lock()
            if h.rooms[client.roomId] == nil {
                h.rooms[client.roomId] = make(map[*Client]bool)
            }
            h.rooms[client.roomId][client] = true
            h.mu.Unlock()

        case client := <-h.unregister:
            h.mu.Lock()
            if room, ok := h.rooms[client.roomId]; ok {
                if _, ok := room[client]; ok {
                    delete(room, client)
                    close(client.send)
                    if len(room) == 0 {
                        delete(h.rooms, client.roomId)
                    }
                }
            }
            h.mu.Unlock()

        case msg := <-h.broadcast:
            h.mu.RLock()
            room := h.rooms[msg.RoomId]
            h.mu.RUnlock()

            for client := range room {
                if client.userId == msg.From {
                    continue // 不回显给发送者
                }
                select {
                case client.send <- msg.Content:
                default:
                    // 发送缓冲区满，断开连接（客户端太慢）
                    close(client.send)
                    h.mu.Lock()
                    delete(h.rooms[msg.RoomId], client)
                    h.mu.Unlock()
                }
            }
        }
    }
}
```

## 4. 多实例部署（跨节点广播）

单机 Hub 无法处理多 Pod 部署，需要通过 Redis Pub/Sub 广播：

```go
// internal/ws/distributedhub.go
type DistributedHub struct {
    localHub *Hub
    redis    *redis.Redis
    nodeId   string
}

func (h *DistributedHub) Broadcast(msg *Message) {
    // 发布到 Redis，所有节点（包括自己）都会收到
    data, _ := json.Marshal(msg)
    h.redis.Publish("ws:chat:"+msg.RoomId, string(data))
}

// 订阅来自其他节点的消息
func (h *DistributedHub) Subscribe() {
    sub := h.redis.Subscribe(context.Background(), "ws:chat:*")
    for redisMsg := range sub.Channel() {
        var msg Message
        json.Unmarshal([]byte(redisMsg.Payload), &msg)
        // 分发给本节点的客户端
        h.localHub.Broadcast(&msg)
    }
}
```

## 5. 消息协议设计

```go
// 统一消息格式
type WSMessage struct {
    Type    string          `json:"type"`    // text | image | file | system
    RoomId  string          `json:"roomId"`
    From    int64           `json:"from"`
    Content json.RawMessage `json:"content"`
    Time    int64           `json:"time"`
}

// 系统消息（用户加入/离开）
type SystemContent struct {
    Action  string `json:"action"`  // join | leave
    UserId  int64  `json:"userId"`
    Message string `json:"message"`
}

// 服务端处理消息
func (c *Client) handleMessage(data []byte) {
    var msg WSMessage
    if err := json.Unmarshal(data, &msg); err != nil {
        return
    }

    switch msg.Type {
    case "text":
        c.hub.Broadcast(&msg)
    case "join":
        c.roomId = msg.RoomId
        c.hub.sendSystemMessage(msg.RoomId, c.userId, "join")
    case "leave":
        c.hub.sendSystemMessage(msg.RoomId, c.userId, "leave")
        c.roomId = ""
    }
}
```

## 6. 总结

| 场景 | 方案 |
|------|------|
| 单机聊天室 | 本地 Hub + goroutine |
| 多实例部署 | Redis Pub/Sub 广播 |
| 连接保活 | Ping/Pong 心跳机制 |
| 跨域安全 | CheckOrigin 白名单 |
| 慢客户端 | 发送缓冲区 + 断开保护 |

相关文章：
- [gRPC 流式传输：处理实时数据与大量数据](grpc-streaming-cn.md)
- [Redis 客户端：从基础操作到高级用法](redis-client-cn.md)
