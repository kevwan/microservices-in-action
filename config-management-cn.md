# go-zero 配置管理：多环境支持与动态加载

## 前言

配置管理是微服务中最基础也最容易出问题的部分。go-zero 提供了稳健的配置加载能力，支持 YAML/JSON/TOML 格式、环境变量覆盖、结构体默认值和强类型解析。对于动态配置场景，可结合 etcd 等配置中心在业务层扩展监听逻辑。

## 1. 基础配置加载

### 1.1 conf.MustLoad 自动加载

```go
// main.go
var c config.Config
conf.MustLoad("etc/config.yaml", &c)
// 加载失败时直接 panic，适合服务启动阶段
```

### 1.2 支持多种格式

```bash
# YAML（推荐）
conf.MustLoad("config.yaml", &c)

# JSON
conf.MustLoad("config.json", &c)

# TOML
conf.MustLoad("config.toml", &c)
```

## 2. 配置结构体定义

### 2.1 结构体标签

go-zero 支持 `json` 标签（复用 JSON 序列化规则）和 `default`/`optional` 扩展标签：

```go
// internal/config/config.go
package config

import (
    "github.com/zeromicro/go-zero/rest"
    "github.com/zeromicro/go-zero/zrpc"
    "github.com/zeromicro/go-zero/core/stores/cache"
)

type Config struct {
    rest.RestConf  // 内嵌 HTTP 服务配置（Host/Port/Timeout 等）

    // 数据库
    DB struct {
        DataSource string  // 必填，无默认值
    }

    // Redis（可选，带默认值）
    Cache cache.CacheConf

    // JWT（带默认值）
    Auth struct {
        AccessSecret  string
        AccessExpire  int64 `json:",default=86400"`  // 默认 1 天
        RefreshSecret string `json:",optional"`       // 可选字段
        RefreshExpire int64  `json:",default=604800"` // 默认 7 天
    }

    // 第三方服务（可选整个块）
    SMS struct {
        ApiKey    string `json:",optional"`
        ApiSecret string `json:",optional"`
        SignName  string `json:",default=MyApp"`
    } `json:",optional"`

    // 下游 RPC 服务
    OrderRpc zrpc.RpcClientConf
    UserRpc  zrpc.RpcClientConf

    // 业务配置
    Order struct {
        MaxRetry    int     `json:",default=3"`
        TimeoutSecs int     `json:",default=30"`
        MaxAmount   float64 `json:",default=99999.99"`
    }
}
```

### 2.2 嵌套配置示例

```yaml
# etc/config.yaml
Name: order-api
Host: 0.0.0.0
Port: 8888
MaxConns: 100
Timeout: 3000

DB:
  DataSource: "root:password@tcp(127.0.0.1:3306)/shop?charset=utf8mb4&parseTime=true"

Cache:
  - Host: 127.0.0.1:6379
    Type: node
    Pass: ""
    DB: 0

Auth:
  AccessSecret: "your-super-secret-key-min-32-chars"
  AccessExpire: 86400

OrderRpc:
  Etcd:
    Hosts:
      - 127.0.0.1:2379
    Key: order.rpc

Order:
  MaxRetry: 3
  TimeoutSecs: 30
```

## 3. 环境变量覆盖

go-zero 支持用环境变量覆盖配置文件中的值，格式为 `${ENV_VAR_NAME:default_value}`：

```yaml
# etc/config.yaml（使用环境变量占位符）
Name: order-api
Host: 0.0.0.0
Port: ${PORT:8888}          # 未设置时默认 8888

DB:
  DataSource: ${DB_DSN}     # 必填，无默认值，未设置会报错

Auth:
  AccessSecret: ${JWT_SECRET:dev-secret-not-for-production}

Cache:
  - Host: ${REDIS_HOST:127.0.0.1}:${REDIS_PORT:6379}
    Pass: ${REDIS_PASS:}    # 空字符串作为默认值
```

```bash
# 生产环境启动
export PORT=8080
export DB_DSN="root:prod_pass@tcp(prod-db:3306)/shop?parseTime=true"
export JWT_SECRET="$(cat /run/secrets/jwt_secret)"
export REDIS_HOST="redis.prod.internal"

./order-api -f etc/config.yaml
```

## 4. 多环境配置

### 4.1 方案一：多配置文件（推荐）

```
etc/
├── config.yaml          # 公共配置（不含敏感信息）
├── config.dev.yaml      # 开发环境
├── config.staging.yaml  # 预发布环境
└── config.prod.yaml     # 生产环境（通过 CI/CD 注入，不入代码库）
```

```go
// main.go
func main() {
    env := flag.String("env", "dev", "运行环境: dev/staging/prod")
    flag.Parse()

    configFile := fmt.Sprintf("etc/config.%s.yaml", *env)
    var c config.Config
    conf.MustLoad(configFile, &c)
    // ...
}
```

### 4.2 方案二：goflags 命令行参数

go-zero 内置了 `-f` 参数指定配置文件：

```go
var configFile = flag.String("f", "etc/config.yaml", "配置文件路径")

func main() {
    flag.Parse()
    var c config.Config
    conf.MustLoad(*configFile, &c)
}
```

```bash
# 不同环境指定不同配置文件
./order-api -f etc/config.prod.yaml
```

### 4.3 方案三：环境变量 + 统一配置文件（K8s 推荐）

在 Kubernetes 中，使用 ConfigMap + Secret 注入环境变量，统一使用一个配置文件：

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-api-config
data:
  PORT: "8888"
  REDIS_HOST: "redis.default.svc.cluster.local"

---
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: order-api-secrets
stringData:
  DB_DSN: "root:prod_pass@tcp(mysql:3306)/shop?parseTime=true"
  JWT_SECRET: "prod-jwt-secret-32-chars-minimum"

---
# k8s/deployment.yaml（引用 ConfigMap 和 Secret）
spec:
  containers:
    - name: order-api
      envFrom:
        - configMapRef:
            name: order-api-config
        - secretRef:
            name: order-api-secrets
```

## 5. 配置热更新

### 5.1 监听配置文件变化

```go
// 监听配置文件变化，配置更新时重新加载（适用于非核心配置）
conf.LoadConfig("etc/config.yaml", &c)

watcher, err := fsnotify.NewWatcher()
if err != nil {
    log.Fatal(err)
}

go func() {
    for {
        select {
        case event := <-watcher.Events:
            if event.Op&fsnotify.Write == fsnotify.Write {
                conf.LoadConfig("etc/config.yaml", &c)
                logx.Info("配置已重新加载")
            }
        case err := <-watcher.Errors:
            logx.Errorf("配置文件监听错误: %v", err)
        }
    }
}()

watcher.Add("etc/config.yaml")
```

### 5.2 集成 etcd 配置中心（动态配置）

```go
// 使用 go-zero 的 etcd 配置监听
import "github.com/zeromicro/go-zero/core/discov"

// 监听 etcd 中的配置 key
client, err := discov.NewEtcdClient([]string{"127.0.0.1:2379"})
if err != nil {
    log.Fatal(err)
}

// 监听配置变化
var featureFlags struct {
    EnableNewCheckout bool
    MaxOrderAmount    float64
}

// 初始加载
value, err := client.GetValue("/config/feature-flags")
json.Unmarshal([]byte(value), &featureFlags)

// 监听变化
client.WatchValue("/config/feature-flags", func(value string) {
    if err := json.Unmarshal([]byte(value), &featureFlags); err == nil {
        logx.Infof("功能开关已更新: %+v", featureFlags)
    }
})
```

## 6. 配置校验

在服务启动时提前校验配置，避免运行时才发现配置错误：

```go
// internal/config/config.go
func (c *Config) Validate() error {
    if len(c.Auth.AccessSecret) < 32 {
        return fmt.Errorf("JWT AccessSecret 长度不足 32 字符，存在安全风险")
    }
    if c.Order.MaxAmount <= 0 {
        return fmt.Errorf("Order.MaxAmount 必须大于 0")
    }
    if c.Order.TimeoutSecs > 300 {
        return fmt.Errorf("Order.TimeoutSecs 不应超过 300 秒")
    }
    return nil
}

// main.go
conf.MustLoad(*configFile, &c)
if err := c.Validate(); err != nil {
    log.Fatalf("配置校验失败: %v", err)
}
```

## 7. 敏感配置安全实践

| 配置类型 | 存储方式 | 备注 |
|---------|---------|------|
| 数据库密码 | K8s Secret / Vault | 绝不写入代码库 |
| JWT Secret | K8s Secret / 环境变量 | 最小 32 字符 |
| API Key | K8s Secret | 定期轮换 |
| 服务端口 | ConfigMap / 配置文件 | 非敏感，可入库 |
| 超时配置 | 配置文件 | 非敏感，可入库 |

```bash
# 安全检查：确保敏感配置不入代码库
echo "etc/config.prod.yaml" >> .gitignore
echo "etc/*.prod.*" >> .gitignore
echo "etc/*secret*" >> .gitignore
```

## 8. 总结

go-zero 的配置系统简洁而实用：

1. **强类型**：结构体映射消除运行时类型错误
2. **默认值**：`json:",default=xxx"` 减少冗余配置
3. **环境变量覆盖**：`${ENV:default}` 语法天然支持 12-Factor App
4. **多环境**：多配置文件或统一文件 + 环境变量两种方式
5. **安全**：敏感配置通过环境变量注入，不入代码库
6. **可扩展**：动态配置通常通过 etcd/Nacos SDK 或自定义组件实现监听与热更新

推荐在 K8s 环境中使用 ConfigMap + Secret 注入环境变量，配合统一的配置文件模板，实现开发/预发布/生产的配置隔离。
