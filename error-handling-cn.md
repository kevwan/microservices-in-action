# go-zero 错误处理与统一响应：打造一致的 API 体验

## 前言

API 设计中，错误处理往往是最容易被忽视的部分。当系统出现异常时，客户端收到的是 `{"error":"sql: no rows in result set"}` 这样的内部错误，还是 `{"code":40400,"message":"订单不存在"}` 这样友好的业务错误？前者会暴露内部实现细节，增加安全风险；后者才是生产级 API 应有的样子。

go-zero 提供了完整的错误处理体系，本文将介绍如何构建一致、安全、易维护的 API 响应格式。

## 1. go-zero 的默认响应格式

默认情况下，go-zero 的 Handler 会将 Logic 层返回的 `error` 直接序列化为 JSON 响应：

```go
// 默认行为（不推荐用于生产环境）
func (l *GetOrderLogic) GetOrder(req *types.GetOrderReq) (*types.GetOrderResp, error) {
    // 如果返回 err，客户端会收到：
    // HTTP 400 {"message":"sql: no rows in result set"}
    // 这暴露了内部实现！
}
```

我们需要的是**自定义错误码体系**，让客户端收到有意义的错误信息。

## 2. 定义错误码体系

### 2.1 错误码设计规范

```go
// internal/errorx/code.go
package errorx

// 错误码规范：
// 1xxxx  系统级错误
// 2xxxx  认证/鉴权错误
// 3xxxx  参数错误
// 4xxxx  业务逻辑错误
// 5xxxx  第三方服务错误

const (
    // 系统错误
    CodeSuccess         = 0
    CodeServerError     = 10000 // 服务器内部错误
    CodeTimeout         = 10001 // 请求超时

    // 认证错误
    CodeUnauthorized    = 20000 // 未登录
    CodeForbidden       = 20001 // 无权限
    CodeTokenExpired    = 20002 // Token 已过期
    CodeTokenInvalid    = 20003 // Token 无效

    // 参数错误
    CodeParamInvalid    = 30000 // 参数校验失败
    CodeParamMissing    = 30001 // 缺少必填参数

    // 业务错误
    CodeNotFound        = 40000 // 资源不存在
    CodeOrderNotFound   = 40001 // 订单不存在
    CodeUserNotFound    = 40002 // 用户不存在
    CodeProductSoldOut  = 40003 // 商品售罄
    CodeDuplicateOrder  = 40004 // 重复下单
    CodeOrderCancelled  = 40005 // 订单已取消，不可操作

    // 第三方错误
    CodePaymentFailed   = 50000 // 支付失败
    CodeSMSFailed       = 50001 // 短信发送失败
)
```

### 2.2 自定义 Error 类型

```go
// internal/errorx/error.go
package errorx

import "net/http"

// CodeError 业务错误，携带错误码
type CodeError struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
}

func (e *CodeError) Error() string {
    return e.Message
}

// 构造函数
func New(code int, message string) *CodeError {
    return &CodeError{Code: code, Message: message}
}

// 常用快捷构造函数
func NewNotFound(msg string) *CodeError {
    return &CodeError{Code: CodeNotFound, Message: msg}
}

func NewUnauthorized(msg string) *CodeError {
    return &CodeError{Code: CodeUnauthorized, Message: msg}
}

func NewForbidden(msg string) *CodeError {
    return &CodeError{Code: CodeForbidden, Message: msg}
}

func NewParamError(msg string) *CodeError {
    return &CodeError{Code: CodeParamInvalid, Message: msg}
}

func NewServerError() *CodeError {
    return &CodeError{Code: CodeServerError, Message: "服务器内部错误"}
}

// HTTPStatus 将业务错误码映射到 HTTP 状态码
func (e *CodeError) HTTPStatus() int {
    switch {
    case e.Code == CodeUnauthorized, e.Code == CodeTokenExpired, e.Code == CodeTokenInvalid:
        return http.StatusUnauthorized
    case e.Code == CodeForbidden:
        return http.StatusForbidden
    case e.Code == CodeNotFound || (e.Code >= 40001 && e.Code < 41000):
        return http.StatusNotFound
    case e.Code >= 30000 && e.Code < 40000:
        return http.StatusBadRequest
    default:
        return http.StatusInternalServerError
    }
}
```

## 3. 自定义统一响应格式

### 3.1 定义统一响应结构

```go
// internal/response/response.go
package response

import (
    "net/http"

    "github.com/zeromicro/go-zero/rest/httpx"
    "myapp/internal/errorx"
)

// Response 统一响应体
type Response struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
    Data    any    `json:"data,omitempty"`
}

// Ok 成功响应
func Ok(w http.ResponseWriter, data any) {
    httpx.OkJsonCtx(r.Context(), w, Response{
        Code:    errorx.CodeSuccess,
        Message: "ok",
        Data:    data,
    })
}

// Fail 失败响应
func Fail(w http.ResponseWriter, r *http.Request, err error) {
    var codeErr *errorx.CodeError

    // 类型断言：判断是否是业务错误
    if e, ok := err.(*errorx.CodeError); ok {
        codeErr = e
    } else {
        // 非业务错误（数据库错误、RPC 错误等）：
        // 记录原始错误，但对外只返回通用错误信息（安全）
        logx.WithContext(r.Context()).Errorf("internal error: %v", err)
        codeErr = errorx.NewServerError()
    }

    httpx.WriteJsonCtx(r.Context(), w, codeErr.HTTPStatus(), Response{
        Code:    codeErr.Code,
        Message: codeErr.Message,
    })
}
```

### 3.2 注册全局错误处理器

go-zero 支持通过 `httpx.SetErrorHandlerCtx` 注册全局错误处理器，这样所有 Handler 都自动使用统一的错误响应格式：

```go
// main.go
func main() {
    var c config.Config
    conf.MustLoad("etc/config.yaml", &c)

    server := rest.MustNewServer(c.RestConf)
    defer server.Stop()

    // 注册全局错误处理器（核心！）
    httpx.SetErrorHandlerCtx(func(ctx context.Context, err error) (int, any) {
        var codeErr *errorx.CodeError
        if e, ok := err.(*errorx.CodeError); ok {
            return e.HTTPStatus(), response.Response{
                Code:    e.Code,
                Message: e.Message,
            }
        }
        // 内部错误：记录日志，对外隐藏细节
        logx.WithContext(ctx).Errorf("internal error: %v", err)
        return http.StatusInternalServerError, response.Response{
            Code:    errorx.CodeServerError,
            Message: "服务器内部错误",
        }
    })

    // 注册统一 OK 响应（可选，统一成功响应格式）
    httpx.SetOkHandler(func(ctx context.Context, w http.ResponseWriter, resp any) {
        httpx.WriteJsonCtx(ctx, w, http.StatusOK, response.Response{
            Code:    errorx.CodeSuccess,
            Message: "ok",
            Data:    resp,
        })
    })

    ctx := svc.NewServiceContext(c)
    handler.RegisterHandlers(server, ctx)
    server.Start()
}
```

### 3.3 Logic 层使用自定义错误

注册全局处理器后，Logic 层直接返回业务错误即可：

```go
// internal/logic/order/getorderlogic.go
func (l *GetOrderLogic) GetOrder(req *types.GetOrderReq) (*types.GetOrderResp, error) {
    order, err := l.svcCtx.OrderModel.FindOne(l.ctx, req.OrderId)
    if err != nil {
        if errors.Is(err, model.ErrNotFound) {
            // 返回业务错误，框架自动转换为 404 + {"code":40001,"message":"订单不存在"}
            return nil, errorx.New(errorx.CodeOrderNotFound, "订单不存在")
        }
        // 数据库错误：框架会记录日志并返回 500 + {"code":10000,"message":"服务器内部错误"}
        return nil, err
    }

    if order.UserId != l.ctx.Value("userId").(int64) {
        return nil, errorx.NewForbidden("无权查看该订单")
    }

    return &types.GetOrderResp{
        OrderId:     fmt.Sprintf("%d", order.Id),
        Status:      order.Status,
        TotalAmount: order.Amount,
        CreatedAt:   order.CreatedAt.Unix(),
    }, nil
}
```

客户端收到的响应：

```json
// 成功
HTTP 200
{
  "code": 0,
  "message": "ok",
  "data": {
    "orderId": "12345",
    "status": "pending",
    "totalAmount": 9900,
    "createdAt": 1714320000
  }
}

// 订单不存在
HTTP 404
{
  "code": 40001,
  "message": "订单不存在"
}

// 无权限
HTTP 403
{
  "code": 20001,
  "message": "无权查看该订单"
}

// 内部错误（对外隐藏细节）
HTTP 500
{
  "code": 10000,
  "message": "服务器内部错误"
}
```

## 4. 参数校验

### 4.1 使用 go-zero 内置校验标签

```go
// 在 .api 文件中或 types.go 中定义校验规则
type CreateOrderReq {
    ProductId int64  `json:"productId"  validate:"required,gt=0"`
    Quantity  int    `json:"quantity"   validate:"required,min=1,max=100"`
    Address   string `json:"address"    validate:"required,min=5,max=500"`
}
```

go-zero 会在进入 Logic 层之前自动校验参数，校验失败会触发全局错误处理器，返回 400 错误。

### 4.2 自定义参数校验

对于复杂的业务校验（需要查询数据库），在 Logic 层手动校验：

```go
func (l *CreateOrderLogic) CreateOrder(req *types.CreateOrderReq) (*types.CreateOrderResp, error) {
    // 检查商品库存
    product, err := l.svcCtx.ProductModel.FindOne(l.ctx, req.ProductId)
    if err != nil {
        if errors.Is(err, model.ErrNotFound) {
            return nil, errorx.NewParamError("商品不存在")
        }
        return nil, err
    }

    if product.Stock < int64(req.Quantity) {
        return nil, errorx.New(errorx.CodeProductSoldOut,
            fmt.Sprintf("库存不足，当前库存：%d", product.Stock))
    }

    // 检查重复下单（幂等性）
    existing, err := l.svcCtx.OrderModel.FindByUserAndProduct(l.ctx, userId, req.ProductId)
    if err == nil && existing != nil {
        return nil, errorx.New(errorx.CodeDuplicateOrder, "您已下过相同订单，请勿重复提交")
    }

    // ... 继续业务逻辑
}
```

## 5. RPC 层的错误传递

RPC 服务（gRPC）的错误需要使用 `status` 包包装，才能正确传递到 HTTP 网关层：

```go
// order-rpc/internal/logic/getorderlogic.go（RPC 服务端）
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

func (l *GetOrderLogic) GetOrder(in *order.GetOrderRequest) (*order.GetOrderResponse, error) {
    o, err := l.svcCtx.OrderModel.FindOne(l.ctx, in.OrderId)
    if err != nil {
        if errors.Is(err, model.ErrNotFound) {
            // 使用 gRPC 状态码包装错误
            return nil, status.Errorf(codes.NotFound, "订单不存在: %s", in.OrderId)
        }
        return nil, status.Errorf(codes.Internal, "数据库错误")
    }
    // ...
}
```

```go
// API 网关层调用 RPC 并处理错误
func (l *GetOrderLogic) GetOrder(req *types.GetOrderReq) (*types.GetOrderResp, error) {
    resp, err := l.svcCtx.OrderRpc.GetOrder(l.ctx, &order.GetOrderRequest{
        OrderId: req.OrderId,
    })
    if err != nil {
        // 将 gRPC 错误转换为业务错误
        if st, ok := status.FromError(err); ok {
            switch st.Code() {
            case codes.NotFound:
                return nil, errorx.New(errorx.CodeOrderNotFound, st.Message())
            case codes.PermissionDenied:
                return nil, errorx.NewForbidden(st.Message())
            }
        }
        return nil, err  // 其他错误由全局处理器兜底
    }
    // ...
}
```

## 6. 响应格式对比

| 场景 | 默认响应 | 自定义统一响应 |
| ---- | ------- | ------------ |
| 成功 | `{"orderId":"123"}` | `{"code":0,"message":"ok","data":{"orderId":"123"}}` |
| 业务错误 | `{"message":"订单不存在"}` | `{"code":40001,"message":"订单不存在"}` |
| 内部错误 | `{"message":"sql: no rows"}` | `{"code":10000,"message":"服务器内部错误"}` |
| 参数错误 | `{"message":"Key: 'Quantity' Error:..."}` | `{"code":30000,"message":"quantity must be ≥1"}` |

## 7. 总结

构建一致的错误处理体系需要三步：

1. **定义错误码**：用数字常量区分不同的错误类型，便于客户端按错误码处理
2. **注册全局处理器**：`httpx.SetErrorHandlerCtx` + `httpx.SetOkHandler`，统一所有接口的响应格式
3. **Logic 层使用业务错误**：返回 `*errorx.CodeError`，框架自动转换为正确的 HTTP 状态码和响应体

这样做的核心价值：

- **安全**：内部错误细节不外泄
- **一致**：所有接口的响应格式统一，前端/移动端开发体验好
- **可维护**：错误码集中管理，易于排查问题
