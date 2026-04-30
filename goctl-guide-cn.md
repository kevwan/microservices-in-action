# goctl 代码生成：从零到微服务的极速体验

## 前言

go-zero 最令人印象深刻的特性之一是 **goctl** —— 一个强大的代码生成工具。定义好接口规范，goctl 就能自动生成完整的项目骨架：路由、Handler、Logic、Model（带缓存）、RPC 桩代码，甚至 Dockerfile 和 Kubernetes 配置。

本文将系统介绍 goctl 的核心功能，带你体验"定义即代码"的高效开发模式。

## 1. 安装 goctl

```bash
# 安装 goctl
go install github.com/zeromicro/go-zero/tools/goctl@latest

# 验证安装
goctl --version
# goctl version 1.7.x

# 安装 protoc 及插件（生成 gRPC 代码所需）
goctl env check --install --verbose
```

## 2. API 服务生成

### 2.1 .api 文件格式

.api 文件是 go-zero 的接口定义语言（IDL），描述 HTTP 接口的完整信息：

```api
// order.api

syntax = "v1"

// 请求/响应结构体
type CreateOrderReq {
    ProductId int64  `json:"productId"`
    Quantity  int    `json:"quantity"`
    Address   string `json:"address"`
}

type CreateOrderResp {
    OrderId   string `json:"orderId"`
    TotalAmount int64 `json:"totalAmount"`
}

type GetOrderReq {
    OrderId string `path:"orderId"`
}

type GetOrderResp {
    OrderId     string `json:"orderId"`
    Status      string `json:"status"`
    TotalAmount int64  `json:"totalAmount"`
    CreatedAt   int64  `json:"createdAt"`
}

// 路由分组：公开接口（无鉴权）
@server (
    group: order
    prefix: /api/v1
)
service order-api {
    @doc "创建订单"
    @handler CreateOrder
    post /orders (CreateOrderReq) returns (CreateOrderResp)
}

// 路由分组：需要 JWT 鉴权的接口
@server (
    group:      order
    prefix:     /api/v1
    jwt:        Auth
    middleware: TraceMiddleware
)
service order-api {
    @doc "查询订单"
    @handler GetOrder
    get /orders/:orderId (GetOrderReq) returns (GetOrderResp)

    @doc "取消订单"
    @handler CancelOrder
    delete /orders/:orderId
}
```

### 2.2 生成完整项目

```bash
# 生成 API 服务项目骨架
goctl api go \
  --api order.api \
  --dir ./order-api \
  --style gozero

# 生成的目录结构
order-api/
├── etc/
│   └── order-api.yaml          # 配置文件
├── internal/
│   ├── config/
│   │   └── config.go           # 配置结构体
│   ├── handler/
│   │   ├── order/
│   │   │   ├── createorderhandler.go  # HTTP Handler（路由绑定）
│   │   │   ├── getorderhandler.go
│   │   │   └── cancelorderhandler.go
│   │   └── routes.go           # 路由注册
│   ├── logic/
│   │   └── order/
│   │       ├── createorderlogic.go   # 业务逻辑（你只需填这里）
│   │       ├── getorderlogic.go
│   │       └── cancelorderlogic.go
│   ├── middleware/
│   │   └── tracemiddleware.go  # 自定义中间件桩代码
│   ├── svc/
│   │   └── servicecontext.go   # 依赖注入容器
│   └── types/
│       └── types.go            # 请求/响应结构体
└── order.go                    # main.go
```

### 2.3 生成后只需填写业务逻辑

goctl 生成的代码中，**你只需关注 logic 层**，其他都是自动生成的框架代码：

```go
// internal/logic/order/createorderlogic.go（goctl 已生成框架，你填业务）
func (l *CreateOrderLogic) CreateOrder(req *types.CreateOrderReq) (resp *types.CreateOrderResp, err error) {
    // TODO: 填写业务逻辑
    // 框架已处理：路由、参数解析、响应序列化、JWT 验证、中间件调用

    order, err := l.svcCtx.OrderModel.Insert(l.ctx, &model.Order{
        ProductId: req.ProductId,
        Quantity:  int64(req.Quantity),
        Address:   req.Address,
        Status:    "pending",
    })
    if err != nil {
        return nil, err
    }

    return &types.CreateOrderResp{
        OrderId:     fmt.Sprintf("%d", order.LastInsertId),
        TotalAmount: calculateAmount(req.ProductId, req.Quantity),
    }, nil
}
```

## 3. Model 层生成（带缓存）

### 3.1 从 SQL 建表语句生成

```sql
-- order.sql
CREATE TABLE `order` (
  `id`          bigint NOT NULL AUTO_INCREMENT,
  `user_id`     bigint NOT NULL COMMENT '用户ID',
  `product_id`  bigint NOT NULL COMMENT '商品ID',
  `quantity`    int NOT NULL DEFAULT 1,
  `amount`      bigint NOT NULL COMMENT '金额（分）',
  `status`      varchar(32) NOT NULL DEFAULT 'pending',
  `address`     varchar(500) NOT NULL DEFAULT '',
  `created_at`  timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at`  timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `idx_user_id` (`user_id`),
  KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

```bash
# 从 SQL 文件生成 Model（带 Redis 缓存）
goctl model mysql ddl \
  --src order.sql \
  --dir ./internal/model \
  --cache        # 启用缓存（生成带 Redis 缓存的 CRUD）
  --style gozero

# 生成的文件
internal/model/
├── ordermodel.go          # 接口定义
├── ordermodel_gen.go      # 自动生成的 CRUD 实现
└── vars.go                # 表名等常量
```

### 3.2 生成的 Model 提供完整 CRUD

```go
// 生成的 Model 接口（ordermodel.go）
type OrderModel interface {
    Insert(ctx context.Context, data *Order) (sql.Result, error)
    FindOne(ctx context.Context, id int64) (*Order, error)
    FindOneByUserId(ctx context.Context, userId int64) (*Order, error) // 索引字段自动生成
    Update(ctx context.Context, data *Order) error
    Delete(ctx context.Context, id int64) error
}

// 使用示例
order, err := svcCtx.OrderModel.FindOne(ctx, orderId)
if err == model.ErrNotFound {
    return nil, errorx.NewNotFound("订单不存在")
}
```

### 3.3 从数据库直接生成（datasource 模式）

```bash
# 直接连接数据库生成 Model
goctl model mysql datasource \
  --url "root:password@tcp(127.0.0.1:3306)/shop" \
  --table "order,user,product" \
  --dir ./internal/model \
  --cache \
  --style gozero
```

## 4. RPC 服务生成

### 4.1 定义 .proto 文件

```protobuf
// order.proto
syntax = "proto3";

package order;
option go_package = "./order";

message CreateOrderRequest {
    int64 userId    = 1;
    int64 productId = 2;
    int32 quantity  = 3;
    string address  = 4;
}

message CreateOrderResponse {
    string orderId     = 1;
    int64 totalAmount  = 2;
}

message GetOrderRequest {
    string orderId = 1;
}

message GetOrderResponse {
    string orderId     = 1;
    string status      = 2;
    int64 totalAmount  = 3;
    int64 createdAt    = 4;
}

service OrderService {
    rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
    rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
}
```

### 4.2 生成 RPC 服务

```bash
# 生成 gRPC 服务项目
goctl rpc protoc order.proto \
  --go_out=./order-rpc \
  --go-grpc_out=./order-rpc \
  --zrpc_out=./order-rpc \
  --style gozero

# 生成的目录结构
order-rpc/
├── etc/
│   └── order.yaml
├── internal/
│   ├── config/config.go
│   ├── logic/
│   │   ├── createorderlogic.go   # 填写业务逻辑
│   │   └── getorderlogic.go
│   ├── server/
│   │   └── orderserviceserver.go # gRPC server 实现（自动生成）
│   └── svc/
│       └── servicecontext.go
├── order/                         # protoc 生成的 pb.go 文件
│   ├── order.pb.go
│   └── order_grpc.pb.go
└── order.go                       # main.go
```

## 5. 常用 goctl 命令速查

### 5.1 API 相关

```bash
# 生成 API 项目
goctl api go --api foo.api --dir ./

# 验证 .api 文件语法
goctl api check --api foo.api

# 生成 API 文档（Markdown）
goctl api doc --dir . --o docs/

# 生成 Postman 集合
goctl api postman --api foo.api --o postman.json
```

### 5.2 数据库 Model 相关

```bash
# 从 SQL 文件生成（最常用）
goctl model mysql ddl --src schema.sql --dir ./model --cache

# 从数据库连接生成
goctl model mysql datasource --url "dsn" --table "%" --dir ./model --cache

# 生成 MongoDB Model
goctl model mongo --type User --dir ./model
```

### 5.3 RPC 相关

```bash
# 生成 RPC 服务
goctl rpc protoc foo.proto --go_out=. --go-grpc_out=. --zrpc_out=.

# 新增接口后，只更新 logic 层（不覆盖已有代码）
goctl rpc protoc foo.proto --go_out=. --go-grpc_out=. --zrpc_out=. --idea
```

### 5.4 基础设施生成

```bash
# 生成 Dockerfile
goctl docker --go order.go --port 8888

# 生成 Kubernetes Deployment
goctl kube deploy \
  --name order-api \
  --namespace default \
  --image myapp/order-api:v1.0.0 \
  --port 8888 \
  --replicas 3 \
  --o deploy.yaml
```

## 6. 增量更新：已有项目添加新接口

开发过程中最常见的需求是在已有项目中**添加新接口**。goctl 支持增量更新，不会覆盖已有业务代码：

```bash
# 在 order.api 中添加新接口后，重新生成
goctl api go --api order.api --dir ./

# goctl 的行为：
# ✅ 新增 Handler 文件（新接口）
# ✅ 新增 Logic 文件（新接口）
# ✅ 更新 routes.go（添加新路由）
# ✅ 更新 types.go（添加新结构体）
# ❌ 不会覆盖已有 Logic 文件中的业务逻辑
```

## 7. 项目模板自定义

可以自定义生成代码的模板，以适应团队规范：

```bash
# 导出默认模板
goctl template init --home ./tpl

# 修改模板（例如在 handler 中添加统一日志）
vim ./tpl/api/handler.tpl

# 使用自定义模板生成
goctl api go --api order.api --dir ./ --home ./tpl
```

## 8. 与 goctl 配合的开发流程

```text
1. 定义接口                  2. 生成代码                3. 填写业务逻辑
┌────────────────┐          ┌─────────────────┐         ┌────────────────┐
│ 编写 .api 文件  │          │ goctl api go    │         │ 填写 logic 层  │
│ 定义请求/响应   │  ──────▶  │ goctl model ddl │  ────▶  │ 调用 model/rpc │
│ 定义路由分组    │          │ goctl rpc proto │         │ 添加业务逻辑    │
└────────────────┘          └─────────────────┘         └────────────────┘
                                                                │
4. 部署                      5. 生成基础设施                    │
┌────────────────┐          ┌─────────────────┐               │
│ 构建镜像        │          │ goctl docker    │  ◀────────────┘
│ kubectl apply  │  ◀──────  │ goctl kube      │
└────────────────┘          └─────────────────┘
```

## 9. 总结

goctl 大幅降低了 go-zero 的上手成本，让开发者专注于业务逻辑而非框架胶水代码：

| 功能 | 命令 | 效果 |
| ---- | ---- | ---- |
| API 服务 | `goctl api go` | 完整 HTTP 服务骨架，含路由、Handler、Logic |
| 数据库 Model | `goctl model mysql` | CRUD + Redis 缓存，支持主键/索引查询 |
| RPC 服务 | `goctl rpc protoc` | 完整 gRPC 服务骨架，含 Server、Logic |
| Docker 化 | `goctl docker` | 多阶段构建 Dockerfile |
| K8s 部署 | `goctl kube deploy` | 生产就绪的 Deployment YAML |

推荐的最佳实践：将 `.api` 和 `.proto` 文件纳入版本控制，作为服务契约的唯一来源，团队成员各自生成代码，保证接口定义的一致性。
