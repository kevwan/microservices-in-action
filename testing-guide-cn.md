# go-zero 单元测试与集成测试

## 前言

微服务的可靠性离不开完善的测试体系。go-zero 的分层架构（Handler → Logic → Model）天然适合单元测试：Logic 层通过依赖注入隔离外部依赖，可以轻松用 Mock 替代真实的数据库和 RPC 调用。

## 1. 测试分层策略

```
┌────────────────────────────────────┐
│          E2E 测试（少量）            │  真实环境，验证完整流程
├────────────────────────────────────┤
│          集成测试（适量）            │  真实数据库/Redis，测试数据层
├────────────────────────────────────┤
│          单元测试（大量）            │  Mock 外部依赖，测试业务逻辑
└────────────────────────────────────┘
```

## 2. Logic 层单元测试

### 2.1 用 mockgen 生成 Mock

```bash
# 安装 mockgen
go install github.com/golang/mock/mockgen@latest

# 为 Model 接口生成 Mock
mockgen -source=internal/model/ordermodel.go \
        -destination=internal/model/mock/ordermodel_mock.go \
        -package=mock
```

### 2.2 测试创建订单逻辑

```go
// internal/logic/order/createorderlogic_test.go
package logic

import (
    "context"
    "testing"

    "github.com/golang/mock/gomock"
    "github.com/stretchr/testify/assert"

    "myapp/internal/model/mock"
    "myapp/internal/svc"
    "myapp/internal/types"
)

func TestCreateOrderLogic_CreateOrder_Success(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    // 创建 Mock
    mockOrderModel := mock.NewMockOrderModel(ctrl)
    mockProductModel := mock.NewMockProductModel(ctrl)

    // 设置 Mock 期望
    mockProductModel.EXPECT().
        FindOne(gomock.Any(), int64(100)).
        Return(&model.Product{Id: 100, Stock: 50, Price: 9900}, nil)

    mockOrderModel.EXPECT().
        Insert(gomock.Any(), gomock.Any()).
        Return(&sqlResult{lastInsertId: 12345}, nil)

    // 构建 ServiceContext（注入 Mock）
    svcCtx := &svc.ServiceContext{
        OrderModel:   mockOrderModel,
        ProductModel: mockProductModel,
    }

    // 构建 Context（模拟 JWT 注入的 userId）
    ctx := context.WithValue(context.Background(), "userId", int64(42))

    // 执行被测逻辑
    logic := NewCreateOrderLogic(ctx, svcCtx)
    resp, err := logic.CreateOrder(&types.CreateOrderReq{
        ProductId: 100,
        Quantity:  2,
        Address:   "北京市海淀区中关村大街1号",
    })

    // 断言结果
    assert.NoError(t, err)
    assert.NotEmpty(t, resp.OrderId)
}

func TestCreateOrderLogic_CreateOrder_InsufficientStock(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    mockProductModel := mock.NewMockProductModel(ctrl)
    mockProductModel.EXPECT().
        FindOne(gomock.Any(), int64(100)).
        Return(&model.Product{Id: 100, Stock: 1}, nil) // 库存只有1

    svcCtx := &svc.ServiceContext{
        ProductModel: mockProductModel,
    }

    ctx := context.WithValue(context.Background(), "userId", int64(42))
    logic := NewCreateOrderLogic(ctx, svcCtx)
    _, err := logic.CreateOrder(&types.CreateOrderReq{
        ProductId: 100,
        Quantity:  5, // 请求5个，超过库存
    })

    // 断言返回库存不足错误
    assert.Error(t, err)
    var codeErr *errorx.CodeError
    assert.ErrorAs(t, err, &codeErr)
    assert.Equal(t, errorx.CodeProductSoldOut, codeErr.Code)
}
```

### 2.3 表驱动测试（覆盖多个场景）

```go
func TestCreateOrderLogic_Validate(t *testing.T) {
    tests := []struct {
        name      string
        req       *types.CreateOrderReq
        wantErr   bool
        errCode   int
    }{
        {
            name: "正常下单",
            req:  &types.CreateOrderReq{ProductId: 1, Quantity: 1, Address: "北京市..."},
            wantErr: false,
        },
        {
            name: "数量为0",
            req:  &types.CreateOrderReq{ProductId: 1, Quantity: 0, Address: "北京市..."},
            wantErr: true,
            errCode: errorx.CodeParamInvalid,
        },
        {
            name: "地址为空",
            req:  &types.CreateOrderReq{ProductId: 1, Quantity: 1, Address: ""},
            wantErr: true,
            errCode: errorx.CodeParamInvalid,
        },
        {
            name: "数量超限",
            req:  &types.CreateOrderReq{ProductId: 1, Quantity: 101, Address: "北京市..."},
            wantErr: true,
            errCode: errorx.CodeParamInvalid,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // ...
            _, err := logic.CreateOrder(tt.req)
            if tt.wantErr {
                assert.Error(t, err)
                var codeErr *errorx.CodeError
                if assert.ErrorAs(t, err, &codeErr) {
                    assert.Equal(t, tt.errCode, codeErr.Code)
                }
            } else {
                assert.NoError(t, err)
            }
        })
    }
}
```

## 3. Handler 层测试（HTTP 接口测试）

```go
// internal/handler/order/createorderhandler_test.go
func TestCreateOrderHandler(t *testing.T) {
    // 禁用日志输出
    logx.Disable()

    // 创建测试 Server
    server := rest.MustNewServer(rest.RestConf{})
    svcCtx := &svc.ServiceContext{
        // 注入 Mock ...
    }
    handler.RegisterHandlers(server, svcCtx)

    // 构造 HTTP 请求
    reqBody := `{"productId":100,"quantity":2,"address":"北京市海淀区"}`
    req := httptest.NewRequest(http.MethodPost, "/api/v1/orders",
        strings.NewReader(reqBody))
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer "+generateTestToken(42))

    w := httptest.NewRecorder()
    server.ServeHTTP(w, req)

    assert.Equal(t, http.StatusOK, w.Code)

    var resp map[string]any
    json.Unmarshal(w.Body.Bytes(), &resp)
    assert.Equal(t, float64(0), resp["code"])
    assert.NotEmpty(t, resp["data"].(map[string]any)["orderId"])
}
```

## 4. 数据层集成测试（使用真实数据库）

### 4.1 使用 TestContainers 启动测试数据库

```go
// 安装
// go get github.com/testcontainers/testcontainers-go

func TestMain(m *testing.M) {
    // 启动 MySQL 容器
    ctx := context.Background()
    req := testcontainers.ContainerRequest{
        Image:        "mysql:8.0",
        ExposedPorts: []string{"3306/tcp"},
        Env: map[string]string{
            "MYSQL_ROOT_PASSWORD": "test",
            "MYSQL_DATABASE":      "test",
        },
        WaitingFor: wait.ForLog("port: 3306  MySQL Community Server"),
    }

    mysqlContainer, err := testcontainers.GenericContainer(ctx,
        testcontainers.GenericContainerRequest{
            ContainerRequest: req,
            Started:          true,
        })
    if err != nil {
        log.Fatal(err)
    }
    defer mysqlContainer.Terminate(ctx)

    host, _ := mysqlContainer.Host(ctx)
    port, _ := mysqlContainer.MappedPort(ctx, "3306")
    dsn := fmt.Sprintf("root:test@tcp(%s:%s)/test?parseTime=true", host, port.Port())

    // 初始化数据库结构
    db, _ := sql.Open("mysql", dsn)
    db.Exec(createTableSQL)

    // 运行测试
    os.Exit(m.Run())
}
```

### 4.2 Model 集成测试

```go
func TestOrderModel_Insert_FindOne(t *testing.T) {
    // 使用真实数据库连接
    conn := sqlx.NewMysql(testDSN)
    model := NewOrderModel(conn, testCache)

    // 插入测试数据
    result, err := model.Insert(context.Background(), &Order{
        UserId:    42,
        ProductId: 100,
        Quantity:  2,
        Amount:    19800,
        Status:    "pending",
    })
    assert.NoError(t, err)

    orderId, _ := result.LastInsertId()

    // 查询验证
    order, err := model.FindOne(context.Background(), orderId)
    assert.NoError(t, err)
    assert.Equal(t, int64(42), order.UserId)
    assert.Equal(t, "pending", order.Status)

    // 清理测试数据
    model.Delete(context.Background(), orderId)
}
```

## 5. RPC 客户端 Mock 测试

### 5.1 为 RPC 客户端生成 Mock

```bash
# 基于 proto 生成的 gRPC 接口生成 Mock
mockgen -source=order/orderclient.go \
        -destination=mock/orderclient_mock.go \
        -package=mock
```

```go
// 测试调用 order-rpc 的 API Gateway Logic
func TestGetOrderDetailLogic(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    mockOrderClient := mock.NewMockOrderClient(ctrl)
    mockOrderClient.EXPECT().
        GetOrder(gomock.Any(), &pb.GetOrderRequest{OrderId: "12345"}).
        Return(&pb.GetOrderResponse{
            OrderId:     "12345",
            Status:      "paid",
            TotalAmount: 9900,
        }, nil)

    svcCtx := &svc.ServiceContext{
        OrderRpc: mockOrderClient,
    }

    logic := NewGetOrderDetailLogic(context.Background(), svcCtx)
    resp, err := logic.GetOrderDetail(&types.GetOrderDetailReq{OrderId: "12345"})

    assert.NoError(t, err)
    assert.Equal(t, "paid", resp.Status)
}
```

## 6. 测试覆盖率

```bash
# 运行测试并生成覆盖率报告
go test ./... -coverprofile=coverage.out -covermode=atomic

# 查看覆盖率摘要
go tool cover -func=coverage.out | tail -1

# 生成 HTML 报告（可视化哪些代码未被覆盖）
go tool cover -html=coverage.out -o coverage.html

# 覆盖率目标建议：
# - Logic 层：>80%（核心业务逻辑必须覆盖）
# - Model 层：>60%（集成测试覆盖）
# - Handler 层：>50%（接口契约测试）
```

## 7. CI 中的测试配置

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: test
          MYSQL_DATABASE: test
        ports:
          - 3306:3306
      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'

      - name: Run tests
        run: go test ./... -race -coverprofile=coverage.out

      - name: Coverage check
        run: |
          COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | tr -d '%')
          if (( $(echo "$COVERAGE < 70" | bc -l) )); then
            echo "覆盖率 $COVERAGE% 低于 70%"
            exit 1
          fi
```

## 8. 总结

| 测试类型 | 工具 | 适用层 | 速度 |
|---------|------|--------|------|
| 单元测试 | mockgen + testify | Logic | 极快（毫秒） |
| HTTP 接口测试 | httptest | Handler | 快（毫秒） |
| 集成测试 | TestContainers | Model | 较慢（秒级） |
| RPC Mock 测试 | mockgen | 调用方 | 快（毫秒） |

go-zero 的分层架构让测试变得简单：Logic 层接受接口而非具体实现，天然支持依赖注入和 Mock 替换，是单元测试友好型设计的典范。
