# go-zero 数据库事务：保证数据一致性

## 前言

在电商、支付等业务场景中，一个操作往往需要同时修改多张表，必须保证这些修改要么全部成功，要么全部失败。数据库事务是保证这种**原子性**的核心机制。go-zero 的 sqlx 包提供了简洁的事务 API，支持链式事务和嵌套事务模式。

## 1. 基础事务用法

### 1.1 sqlx.TransactCtx（推荐）

```go
// go-zero 的事务 API
import "github.com/zeromicro/go-zero/core/stores/sqlx"

func (m *OrderModel) CreateOrderWithDeduction(ctx context.Context,
    order *Order, productId int64, quantity int) error {

    return m.conn.TransactCtx(ctx, func(ctx context.Context, session sqlx.Session) error {
        // 1. 创建订单（使用 session，而非 m.conn）
        result, err := session.ExecCtx(ctx,
            "INSERT INTO order (user_id, product_id, quantity, amount, status) VALUES (?,?,?,?,?)",
            order.UserId, order.ProductId, order.Quantity, order.Amount, "pending")
        if err != nil {
            return err  // 返回 error 自动回滚
        }

        orderId, _ := result.LastInsertId()

        // 2. 扣减库存（同一事务中）
        result, err = session.ExecCtx(ctx,
            "UPDATE product SET stock = stock - ? WHERE id = ? AND stock >= ?",
            quantity, productId, quantity)
        if err != nil {
            return err  // 自动回滚
        }

        affected, _ := result.RowsAffected()
        if affected == 0 {
            // 库存不足，手动返回 error 触发回滚
            return errors.New("库存不足")
        }

        // 3. 创建库存扣减日志
        _, err = session.ExecCtx(ctx,
            "INSERT INTO stock_log (order_id, product_id, quantity, type) VALUES (?,?,?,?)",
            orderId, productId, quantity, "deduct")
        return err  // nil 则自动提交，error 则自动回滚
    })
}
```

### 1.2 事务中的查询

```go
func (m *OrderModel) TransferBalance(ctx context.Context, fromUserId, toUserId int64, amount int64) error {
    return m.conn.TransactCtx(ctx, func(ctx context.Context, session sqlx.Session) error {
        // 事务中查询（使用 FOR UPDATE 锁行）
        var fromBalance int64
        err := session.QueryRowCtx(ctx, &fromBalance,
            "SELECT balance FROM user_wallet WHERE user_id = ? FOR UPDATE",
            fromUserId)
        if err != nil {
            return err
        }

        if fromBalance < amount {
            return errors.New("余额不足")
        }

        // 扣减转出方余额
        _, err = session.ExecCtx(ctx,
            "UPDATE user_wallet SET balance = balance - ? WHERE user_id = ?",
            amount, fromUserId)
        if err != nil {
            return err
        }

        // 增加转入方余额
        _, err = session.ExecCtx(ctx,
            "UPDATE user_wallet SET balance = balance + ? WHERE user_id = ?",
            amount, toUserId)
        return err
    })
}
```

## 2. 事务与 Model 层集成

### 2.1 在 Model 层暴露事务方法

```go
// internal/model/ordermodel.go

type OrderModel interface {
    Insert(ctx context.Context, data *Order) (sql.Result, error)
    // 事务方法：接受 sqlx.Session
    InsertTx(ctx context.Context, session sqlx.Session, data *Order) (sql.Result, error)
}

// 实现
func (m *customOrderModel) InsertTx(ctx context.Context,
    session sqlx.Session, data *Order) (sql.Result, error) {

    query := fmt.Sprintf("insert into %s (%s) values (?, ?, ?, ?, ?)",
        m.table, orderRowsExpectAutoSet)
    return session.ExecCtx(ctx, query,
        data.UserId, data.ProductId, data.Quantity, data.Amount, data.Status)
}
```

### 2.2 跨 Model 的事务（Logic 层协调）

```go
// internal/logic/order/createorderlogic.go
func (l *CreateOrderLogic) CreateOrder(req *types.CreateOrderReq) (*types.CreateOrderResp, error) {
    var orderId int64

    // Logic 层协调跨 Model 的事务
    err := l.svcCtx.DB.TransactCtx(l.ctx, func(ctx context.Context, session sqlx.Session) error {
        // 1. 创建订单
        result, err := l.svcCtx.OrderModel.InsertTx(ctx, session, &model.Order{
            UserId:    getUserId(l.ctx),
            ProductId: req.ProductId,
            Quantity:  int64(req.Quantity),
            Amount:    calculateAmount(req.ProductId, req.Quantity),
            Status:    "pending",
        })
        if err != nil {
            return err
        }
        orderId, _ = result.LastInsertId()

        // 2. 扣减库存
        affected, err := l.svcCtx.ProductModel.DeductStockTx(ctx, session,
            req.ProductId, int64(req.Quantity))
        if err != nil {
            return err
        }
        if affected == 0 {
            return errorx.New(errorx.CodeProductSoldOut, "库存不足")
        }

        // 3. 记录积分（可选，失败不影响主流程时可用 savepoint）
        return l.svcCtx.PointModel.AddPointsTx(ctx, session,
            getUserId(l.ctx), calculatePoints(req.Quantity))
    })

    if err != nil {
        return nil, err
    }

    return &types.CreateOrderResp{
        OrderId: fmt.Sprintf("%d", orderId),
    }, nil
}
```

## 3. 事务隔离级别

go-zero 支持设置事务隔离级别：

```go
// 使用指定隔离级别的事务（默认是 READ COMMITTED）
err := m.conn.TransactCtx(ctx, func(ctx context.Context, session sqlx.Session) error {
    // 业务逻辑
    return nil
}, sql.LevelSerializable)  // 最高隔离级别：可串行化
```

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 适用场景 |
|---------|------|-----------|------|---------|
| READ UNCOMMITTED | ✓ | ✓ | ✓ | 几乎不用 |
| READ COMMITTED | ✗ | ✓ | ✓ | 默认，大多数场景 |
| REPEATABLE READ | ✗ | ✗ | ✓ | MySQL 默认（InnoDB 用 MVCC 解决幻读） |
| SERIALIZABLE | ✗ | ✗ | ✗ | 高一致性要求（性能最差） |

## 4. 分布式事务（Saga 模式）

当事务需要跨多个微服务时，数据库事务无法满足需求，需要使用分布式事务模式：

```
单体应用：                          微服务：
┌────────────────────┐             ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Begin Transaction  │             │ order-rpc│  │ stock-rpc│  │ point-rpc│
│   INSERT order     │   →         │ 创建订单  │  │ 扣减库存  │  │ 增加积分  │
│   UPDATE stock     │             │          │  │          │  │          │
│   INSERT points    │             └──────────┘  └──────────┘  └──────────┘
│ Commit             │               各服务各自的数据库，无法共享事务
└────────────────────┘
```

Saga 补偿事务：

```go
// 简化的 Saga 实现
func (l *CreateOrderLogic) CreateOrderSaga(req *types.CreateOrderReq) (*types.CreateOrderResp, error) {
    var createdOrderId string
    var stockDeducted bool

    // Step 1: 创建订单
    orderResp, err := l.svcCtx.OrderRpc.CreateOrder(l.ctx, &pb.CreateOrderRequest{...})
    if err != nil {
        return nil, err
    }
    createdOrderId = orderResp.OrderId

    // Step 2: 扣减库存（失败需要补偿步骤1）
    _, err = l.svcCtx.StockRpc.DeductStock(l.ctx, &pb.DeductStockRequest{
        ProductId: req.ProductId, Quantity: int32(req.Quantity),
    })
    if err != nil {
        // 补偿：取消订单
        l.svcCtx.OrderRpc.CancelOrder(l.ctx, &pb.CancelOrderRequest{OrderId: createdOrderId})
        return nil, err
    }
    stockDeducted = true

    // Step 3: 增加积分（失败不补偿，积分可以异步补充）
    _, err = l.svcCtx.PointRpc.AddPoints(l.ctx, &pb.AddPointsRequest{...})
    if err != nil {
        logx.WithContext(l.ctx).Errorf("积分添加失败，将异步补偿: orderId=%s", createdOrderId)
        // 通过消息队列异步补偿，不回滚订单和库存
        l.svcCtx.MQ.Push(l.ctx, `{"type":"add_points","orderId":"`+createdOrderId+`"}`)
    }

    _ = stockDeducted  // 成功使用
    return &types.CreateOrderResp{OrderId: createdOrderId}, nil
}
```

## 5. 事务常见问题

### 5.1 事务超时

```go
// 为事务设置超时（避免长事务锁住资源）
ctx, cancel := context.WithTimeout(l.ctx, 5*time.Second)
defer cancel()

err := m.conn.TransactCtx(ctx, func(ctx context.Context, session sqlx.Session) error {
    // 如果 5 秒内未完成，ctx.Done() 触发，驱动层报错并回滚
    return nil
})
```

### 5.2 避免长事务

```go
// ❌ 错误：在事务中调用 HTTP/RPC（可能导致长事务）
err := m.conn.TransactCtx(ctx, func(ctx context.Context, session sqlx.Session) error {
    session.ExecCtx(ctx, "INSERT INTO order...")

    // 这个 RPC 调用可能需要几秒钟，期间持有数据库锁
    payResp, err := paymentClient.Charge(ctx, ...)  // ❌
    return err
})

// ✅ 正确：先完成事务，再调用外部服务
orderId, err := createOrderInTransaction(ctx)
if err != nil {
    return err
}
// 事务已提交，再调用支付
payResp, err := paymentClient.Charge(ctx, orderId)
```

## 6. 总结

| 场景 | 方案 |
|------|------|
| 单服务多表操作 | `TransactCtx` 数据库事务 |
| 跨服务一致性 | Saga + 补偿事务 |
| 消息 + 数据库一致性 | Outbox 模式 |
| 高并发防超卖 | 事务 + `FOR UPDATE` 或乐观锁 |

go-zero 的 `TransactCtx` 通过 Context 传递事务会话，优雅地支持跨 Model 的事务操作，同时完全支持 Go 原生的 Context 超时和取消机制。
