# go-zero 请求参数校验：从基础到自定义规则

## 前言

API 接口的第一道防线是参数校验。不合法的参数进入业务逻辑会导致各种错误：数据库约束违反、空指针崩溃、逻辑错误。go-zero 提供了请求解析与基础校验能力，本文示例基于 `validator/v10` 展示一套可扩展的参数校验实践。

## 1. .api 文件中的校验标签

```api
type CreateOrderReq {
    // validate 标签：声明校验规则
    ProductId  int64  `json:"productId"  validate:"required,gt=0"`
    Quantity   int    `json:"quantity"   validate:"required,min=1,max=100"`
    Address    string `json:"address"    validate:"required,min=5,max=500"`
    CouponCode string `json:"couponCode" validate:"omitempty,len=8,alphanum"`
    Phone      string `json:"phone"      validate:"required,e164"`
}
```

参数校验发生在 Handler 层，业务逻辑进入 Logic 层前已经过滤了非法参数。

## 2. 常用校验规则

### 2.1 基础校验

```go
// 必填与可选
validate:"required"          // 不能为零值（0、""、nil 都不行）
validate:"omitempty"         // 如果为空则跳过后续校验

// 字符串长度
validate:"min=5,max=100"     // 字符串长度范围
validate:"len=8"             // 精确长度

// 数字范围
validate:"gt=0"              // 大于 0
validate:"gte=0,lte=100"     // 0 到 100 之间（闭区间）
validate:"oneof=1 2 3"       // 只能是 1、2、3 之一

// 格式校验
validate:"email"             // 邮箱格式
validate:"url"               // URL 格式
validate:"e164"              // 电话号码（E.164格式，如 +8613800138000）
validate:"uuid4"             // UUIDv4 格式
validate:"ip"                // IP 地址
validate:"ipv4"              // IPv4 地址
validate:"alphanum"          // 只允许字母数字
validate:"numeric"           // 只允许数字
validate:"alpha"             // 只允许字母
```

### 2.2 实际业务示例

```go
type RegisterReq {
    Username    string `json:"username"    validate:"required,alphanum,min=3,max=20"`
    Password    string `json:"password"    validate:"required,min=8,max=64"`
    Email       string `json:"email"       validate:"required,email"`
    Phone       string `json:"phone"       validate:"omitempty,e164"`
    InviteCode  string `json:"inviteCode"  validate:"omitempty,alphanum,len=6"`
    BirthDate   string `json:"birthDate"   validate:"omitempty,datetime=2006-01-02"`
    Gender      int    `json:"gender"      validate:"required,oneof=1 2 3"`  // 1男 2女 3未知
}

type SearchReq {
    Keyword    string `json:"keyword"    validate:"required,min=1,max=100"`
    Page       int    `json:"page"       validate:"min=1,max=1000"`
    PageSize   int    `json:"pageSize"   validate:"min=1,max=100"`
    SortBy     string `json:"sortBy"     validate:"omitempty,oneof=price sales rating"`
    SortOrder  string `json:"sortOrder"  validate:"omitempty,oneof=asc desc"`
    MinPrice   int64  `json:"minPrice"   validate:"omitempty,min=0"`
    MaxPrice   int64  `json:"maxPrice"   validate:"omitempty,gtefield=MinPrice"` // 大于等于 MinPrice
}
```

## 3. 自定义校验规则

当内置规则无法满足业务需求时，可以注册自定义校验器：

### 3.1 注册自定义规则

```go
// internal/validator/validator.go
package validator

import (
    "regexp"

    "github.com/go-playground/validator/v10"
    "github.com/zeromicro/go-zero/rest/httpx"
)

var (
    // 中国大陆手机号
    cnPhoneRegex = regexp.MustCompile(`^1[3-9]\d{9}$`)
    // 中文姓名（2-10个汉字）
    cnNameRegex = regexp.MustCompile(`^[\p{Han}]{2,10}$`)
    // 中国身份证号（18位）
    cnIDCardRegex = regexp.MustCompile(`^\d{17}[\dX]$`)
)

func RegisterCustomValidators() {
    v := httpx.GetValidator() // 获取 go-zero 内部使用的 validator

    // 注册中国手机号校验
    v.RegisterValidation("cn_phone", func(fl validator.FieldLevel) bool {
        return cnPhoneRegex.MatchString(fl.Field().String())
    })

    // 注册中文姓名校验
    v.RegisterValidation("cn_name", func(fl validator.FieldLevel) bool {
        return cnNameRegex.MatchString(fl.Field().String())
    })

    // 注册身份证号校验（含最后一位校验位）
    v.RegisterValidation("cn_idcard", func(fl validator.FieldLevel) bool {
        id := fl.Field().String()
        if !cnIDCardRegex.MatchString(id) {
            return false
        }
        return validateIDCardChecksum(id)
    })
}

// 在 main.go 中调用
func main() {
    validator.RegisterCustomValidators()
    // ...
}
```

### 3.2 使用自定义规则

```go
type KYCReq {
    RealName string `json:"realName" validate:"required,cn_name"`
    IDCard   string `json:"idCard"   validate:"required,cn_idcard"`
    Phone    string `json:"phone"    validate:"required,cn_phone"`
}
```

## 4. 自定义错误消息

默认的校验错误消息是英文的，可以自定义：

```go
// 注册自定义错误消息
v := httpx.GetValidator()

// 为每个 tag 注册中文错误消息
v.RegisterTranslation("required", trans, func(ut ut.Translator) error {
    return ut.Add("required", "{0} 是必填项", true)
}, func(ut ut.Translator, fe validator.FieldError) string {
    t, _ := ut.T("required", fe.Field())
    return t
})

v.RegisterTranslation("min", trans, func(ut ut.Translator) error {
    return ut.Add("min", "{0} 最小值为 {1}", true)
}, func(ut ut.Translator, fe validator.FieldError) string {
    t, _ := ut.T("min", fe.Field(), fe.Param())
    return t
})
```

## 5. Logic 层的业务校验

参数格式校验由 Handler 层完成，但**业务逻辑校验**（需要查数据库的校验）应在 Logic 层进行：

```go
// internal/logic/user/registerlogic.go
func (l *RegisterLogic) Register(req *types.RegisterReq) (*types.RegisterResp, error) {
    // 参数格式校验：已在 Handler 层完成（validate 标签）

    // 业务校验1：用户名唯一性（需要查数据库）
    existing, err := l.svcCtx.UserModel.FindOneByUsername(l.ctx, req.Username)
    if err != nil && !errors.Is(err, model.ErrNotFound) {
        return nil, err
    }
    if existing != nil {
        return nil, errorx.NewParamError("用户名已被占用")
    }

    // 业务校验2：邀请码有效性
    if req.InviteCode != "" {
        inviter, err := l.svcCtx.UserModel.FindOneByInviteCode(l.ctx, req.InviteCode)
        if err != nil || inviter == nil {
            return nil, errorx.NewParamError("邀请码无效")
        }
    }

    // 业务校验3：手机号是否已注册
    if req.Phone != "" {
        phoneUser, err := l.svcCtx.UserModel.FindOneByPhone(l.ctx, req.Phone)
        if err != nil && !errors.Is(err, model.ErrNotFound) {
            return nil, err
        }
        if phoneUser != nil {
            return nil, errorx.NewParamError("手机号已被注册")
        }
    }

    // 通过所有校验，继续注册流程
    return l.doRegister(req)
}
```

## 6. 结构体嵌套校验

```go
type CreateShipmentReq {
    OrderId string      `json:"orderId"  validate:"required"`
    Address AddressInfo `json:"address"  validate:"required"`
}

type AddressInfo {
    Province string `json:"province" validate:"required,max=20"`
    City     string `json:"city"     validate:"required,max=20"`
    District string `json:"district" validate:"required,max=20"`
    Street   string `json:"street"   validate:"required,max=100"`
    ZipCode  string `json:"zipCode"  validate:"omitempty,len=6,numeric"`
}
```

注意：go-zero 对嵌套结构体会自动递归校验。

## 7. 数组/切片校验

```go
type BatchCreateOrderReq {
    Items []OrderItem `json:"items" validate:"required,min=1,max=20,dive"`
    //                                                              ↑ dive 表示校验数组元素
}

type OrderItem {
    ProductId int64  `json:"productId" validate:"required,gt=0"`
    Quantity  int    `json:"quantity"  validate:"required,min=1,max=100"`
}
```

## 8. 总结

| 校验类型 | 实现位置 | 工具 |
|---------|---------|------|
| 格式校验（必填、范围、格式）| Handler 层（自动） | `validate` 标签 |
| 自定义格式规则 | 注册自定义 validator | `RegisterValidation` |
| 业务逻辑校验（唯一性等） | Logic 层（手动） | 查数据库 + errorx |
| RPC 参数校验 | gRPC 拦截器 | ValidationInterceptor |

go-zero 的参数校验遵循单一职责：格式校验在框架层自动完成，业务校验在 Logic 层明确表达，两者职责清晰，代码易维护。
