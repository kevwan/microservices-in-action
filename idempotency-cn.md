# go-zero 幂等性设计：杜绝重复操作的副作用

## 前言

幂等性（Idempotency）是指：**同一操作执行一次和多次，结果相同**。在分布式系统中，网络重试、客户端超时重传、消息队列重复投递都可能导致同一请求被处理多次。如果接口不具备幂等性，用户可能被重复扣款、订单被重复创建、库存被重复扣减。

## 1. 哪些场景需要幂等

| 场景 | 风险 | 是否需要幂等 |
|------|------|------------|
| 创建订单 | 网络超时重试 → 重复下单 | ✅ 必须 |
| 支付扣款 | 支付超时重试 → 重复扣款 | ✅ 必须 |
| 发送短信 | 消息队列重投 → 重复发短信 | ✅ 必须 |
| 库存扣减 | 网络失败重试 → 超卖 | ✅ 必须 |
| 查询接口 | 重复查询结果相同 | ✅ 天然幂等 |
| 更新状态 | SET status=paid 多次结果相同 | ✅ 通常幂等 |

## 2. 幂等性实现方案

### 2.1 Token 方案（通用，推荐）

```
客户端                    服务端                    Redis
  │                          │                        │
  │── GET /api/idempotent ──▶ │                        │
  │                          │── SET token:xxx "1" ──▶ │
  │◀── token: "abc-123" ───── │                        │
  │                          │                        │
  │── POST /api/orders ─────▶ │                        │
  │   X-Idempotency-Key: abc-123                       │
  │                          │── GET token:abc-123 ──▶ │
  │                          │◀── "1" ──────────────── │
  │                          │── DEL token:abc-123 ──▶ │（原子操作：获取并删除）
  │                          │   执行业务逻辑            │
  │◀── 201 Created ────────── │                        │
  │                          │                        │
  │── POST /api/orders ─────▶ │（重复请求）              │
  │   X-Idempotency-Key: abc-123                       │
  │                          │── GET token:abc-123 ──▶ │
  │                          │◀── nil（已删除）──────── │
  │◀── 409 Conflict ────────── │（拒绝重复请求）         │
```

#### 步骤1：生成幂等 Token

```go
// internal/handler/idempotent/gettokenhandler.go
func GetIdempotentToken(svcCtx *svc.ServiceContext) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        token := uuid.NewString()
        // 存入 Redis，有效期 10 分钟
        err := svcCtx.Redis.Setex("idempotent:"+token, "1", 600)
        if err != nil {
            httpx.Error(w, err)
            return
        }
        httpx.OkJson(w, map[string]string{"token": token})
    }
}
```

#### 步骤2：幂等校验中间件

```go
// internal/middleware/idempotent.go
func IdempotentMiddleware(rds *redis.Redis) func(http.HandlerFunc) http.HandlerFunc {
    return func(next http.HandlerFunc) http.HandlerFunc {
        return func(w http.ResponseWriter, r *http.Request) {
            token := r.Header.Get("X-Idempotency-Key")
            if token == "" {
                httpx.WriteJsonCtx(r.Context(), w, http.StatusBadRequest,
                    map[string]any{"code": 30000, "message": "缺少 X-Idempotency-Key"})
                return
            }

            // 原子操作：尝试删除 token（成功=首次请求，失败=重复请求）
            // 使用 Lua 脚本保证原子性
            script := `
                local val = redis.call('GET', KEYS[1])
                if val then
                    redis.call('DEL', KEYS[1])
                    return 1
                end
                return 0
            `
            result, err := rds.Eval(script, []string{"idempotent:" + token})
            if err != nil {
                // Redis 故障，fail-open（放行请求）
                logx.Errorf("幂等检查失败: %v", err)
                next(w, r)
                return
            }

            if result.(int64) == 0 {
                // Token 不存在或已使用
                httpx.WriteJsonCtx(r.Context(), w, http.StatusConflict,
                    map[string]any{"code": 40900, "message": "请勿重复提交"})
                return
            }

            next(w, r)
        }
    }
}
```

### 2.2 业务唯一键方案（数据库层）

在数据库表上建唯一索引，依靠数据库约束保证幂等：

```sql
-- 订单表：以 (user_id, product_id, client_order_id) 建唯一索引
CREATE TABLE `order` (
    `id`              bigint NOT NULL AUTO_INCREMENT,
    `user_id`         bigint NOT NULL,
    `product_id`      bigint NOT NULL,
    `client_order_id` varchar(64) NOT NULL COMMENT '客户端生成的唯一ID',
    `status`          varchar(32) NOT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_client_order_id` (`client_order_id`)  -- 唯一约束
);
```

```go
func (l *CreateOrderLogic) CreateOrder(req *types.CreateOrderReq) (*types.CreateOrderResp, error) {
    // client_order_id 由客户端生成（通常是 UUID），重试时携带相同 ID
    result, err := l.svcCtx.OrderModel.Insert(l.ctx, &model.Order{
        UserId:        getUserId(l.ctx),
        ProductId:     req.ProductId,
        ClientOrderId: req.ClientOrderId, // 客户端传入的幂等键
        Status:        "pending",
    })

    if err != nil {
        // 检查是否是唯一键冲突
        if isDuplicateKeyError(err) {
            // 返回已存在的订单
            order, err := l.svcCtx.OrderModel.FindByClientOrderId(l.ctx, req.ClientOrderId)
            if err != nil {
                return nil, err
            }
            return &types.CreateOrderResp{OrderId: fmt.Sprintf("%d", order.Id)}, nil
        }
        return nil, err
    }

    orderId, _ := result.LastInsertId()
    return &types.CreateOrderResp{OrderId: fmt.Sprintf("%d", orderId)}, nil
}

func isDuplicateKeyError(err error) bool {
    if mysqlErr, ok := err.(*mysql.MySQLError); ok {
        return mysqlErr.Number == 1062  // Duplicate entry
    }
    return false
}
```

### 2.3 状态机方案（支付等关键流程）

```go
// 支付幂等：通过状态机防止重复扣款
func (l *PayOrderLogic) PayOrder(req *types.PayOrderReq) error {
    // 查询订单当前状态（加锁）
    order, err := l.svcCtx.OrderModel.FindOneForUpdate(l.ctx, req.OrderId)
    if err != nil {
        return err
    }

    // 状态机校验：只有 pending 状态的订单才能支付
    switch order.Status {
    case "pending":
        // 可以支付，继续
    case "paid":
        // 已支付，返回成功（幂等）
        return nil
    case "cancelled":
        return errorx.New(errorx.CodeOrderCancelled, "订单已取消")
    default:
        return errorx.New(errorx.CodeInvalidStatus, "订单状态异常")
    }

    // 执行支付
    return l.doPayOrder(order, req.PaymentMethod)
}
```

## 3. 消息消费幂等

消息队列的消费者必须实现幂等，因为消息可能被重复投递：

```go
// 基于 Redis SET NX 的消费幂等
func (c *OrderConsumer) Consume(ctx context.Context, key, value string) error {
    // 使用消息 offset 或业务 ID 作为幂等键
    idempotentKey := fmt.Sprintf("consumed:order:%s", key)

    // SET NX：如果 key 不存在则设置（返回 true），否则返回 false
    set, err := c.redis.SetnxEx(idempotentKey, "1", 3600*24) // 24小时
    if err != nil {
        return err // Redis 故障，重试
    }
    if !set {
        // 已消费过，跳过
        logx.WithContext(ctx).Infof("重复消息，跳过: key=%s", key)
        return nil
    }

    // 首次消费，执行业务逻辑
    var event OrderCreatedEvent
    json.Unmarshal([]byte(value), &event)
    return c.sendSMS(ctx, event)
}
```

## 4. 幂等键的设计原则

```
✅ 好的幂等键设计：
- 客户端生成的 UUID：{uuid}
- 业务对象 + 操作：order:{orderId}:pay
- 用户 + 时间窗口（防抖）：user:{userId}:create_order:{timestamp/60}
- 消息 offset：kafka:{topic}:{partition}:{offset}

❌ 不好的幂等键设计：
- 纯时间戳（精度不够，可能重复）
- 随机数（每次不同，失去幂等意义）
- 仅用用户ID（粒度太粗，会误拒合法请求）
```

## 5. 总结

| 方案 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| Token + Redis | 通用 HTTP 接口 | 灵活，可控 | 需要额外获取 token 步骤 |
| 数据库唯一键 | 写入类操作 | 简单，无外部依赖 | 需要预先设计 schema |
| 状态机 | 支付、审批流 | 语义清晰 | 需要加锁，有竞争 |
| Redis SETNX | 消息消费 | 简单高效 | 需要合理设计 key 的 TTL |

幂等性设计的核心：**在合适的层面拦截重复操作，并对已处理的请求返回原始结果**，而不是错误。
