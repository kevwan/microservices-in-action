# go-zero TLS/mTLS：服务间双向认证

## 前言

微服务之间的通信默认是明文的，在生产环境中存在数据泄露和中间人攻击的风险。TLS（传输层安全）加密通信内容，mTLS（双向 TLS）不仅加密，还要求客户端也提供证书，实现服务间的双向身份认证，是零信任安全架构的核心技术。

## 1. TLS vs mTLS

```
TLS（单向认证）：              mTLS（双向认证）：
客户端验证服务端证书            服务端和客户端互相验证证书

Client ──▶ 验证服务端 CA ──▶ Server    Client ──▶ 验证服务端 CA ──▶ Server
           建立加密连接                          ◀── 验证客户端 CA ──
                                               建立加密连接
```

| 特性 | HTTP | TLS | mTLS |
|------|------|-----|------|
| 加密传输 | ❌ | ✅ | ✅ |
| 服务端身份验证 | ❌ | ✅ | ✅ |
| 客户端身份验证 | ❌ | ❌ | ✅ |
| 防止 IP 欺骗 | ❌ | ❌ | ✅ |

## 2. 生成证书（开发环境）

```bash
# 生成 CA 根证书
openssl req -x509 -newkey rsa:4096 -keyout ca.key -out ca.crt \
    -days 3650 -nodes -subj "/CN=MyCA/O=MyOrg"

# 生成服务端证书（由 CA 签发）
openssl req -newkey rsa:4096 -keyout server.key -out server.csr \
    -nodes -subj "/CN=order-api/O=MyOrg"
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
    -CAcreateserial -out server.crt -days 365 \
    -extfile <(echo "subjectAltName=DNS:localhost,DNS:order-api,IP:127.0.0.1")

# 生成客户端证书（用于 mTLS）
openssl req -newkey rsa:4096 -keyout client.key -out client.csr \
    -nodes -subj "/CN=user-api/O=MyOrg"
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key \
    -CAcreateserial -out client.crt -days 365
```

## 3. REST 服务启用 TLS

```yaml
# config.yaml
RestConf:
  Host: 0.0.0.0
  Port: 8443
  CertFile: /etc/certs/server.crt
  KeyFile:  /etc/certs/server.key
```

go-zero 读取配置后自动启用 HTTPS：

```go
// main.go
server := rest.MustNewServer(c.RestConf)  // 自动使用 TLS
defer server.Stop()
// 其余代码不变
```

## 4. gRPC 服务启用 TLS

### 4.1 服务端配置

```yaml
# config.yaml
RpcServerConf:
  ListenOn: 0.0.0.0:9443
  CertFile: /etc/certs/server.crt
  KeyFile:  /etc/certs/server.key
```

go-zero zRPC 服务端自动加载证书：

```go
// main.go
server := zrpc.MustNewServer(c.RpcServerConf, func(s *grpc.Server) {
    pb.RegisterOrderServiceServer(s, server.NewOrderServiceServer(ctx))
})
defer server.Stop()
server.Start()
```

### 4.2 客户端配置

```yaml
# config.yaml（调用方服务）
OrderRpcConf:
  Target: k8s://namespace/order-service:9443  # 或 dns:///order-service:9443
  CertFile: /etc/certs/ca.crt   # 验证服务端证书的 CA
  ServerName: order-api          # 证书中的 CN/SAN 名称
```

```go
// 创建 zRPC 客户端（自动 TLS）
conn := zrpc.MustNewClient(c.OrderRpcConf)
orderClient := pb.NewOrderServiceClient(conn.Conn())
```

## 5. mTLS 配置

mTLS 要求客户端也提供证书，服务端验证客户端身份：

```go
// 服务端：要求客户端证书（mTLS）
import (
    "crypto/tls"
    "crypto/x509"
    "os"
)

func createMTLSConfig() (*tls.Config, error) {
    // 加载服务端证书
    serverCert, err := tls.LoadX509KeyPair("server.crt", "server.key")
    if err != nil {
        return nil, err
    }

    // 加载 CA，用于验证客户端证书
    caCert, err := os.ReadFile("ca.crt")
    if err != nil {
        return nil, err
    }
    caPool := x509.NewCertPool()
    caPool.AppendCertsFromPEM(caCert)

    return &tls.Config{
        Certificates: []tls.Certificate{serverCert},
        ClientCAs:    caPool,
        ClientAuth:   tls.RequireAndVerifyClientCert,  // 强制验证客户端证书
        MinVersion:   tls.VersionTLS13,                 // 只允许 TLS 1.3
    }, nil
}
```

```go
// 在 gRPC 服务端注入 mTLS
tlsConfig, _ := createMTLSConfig()
creds := credentials.NewTLS(tlsConfig)

server := grpc.NewServer(grpc.Creds(creds))
```

```go
// gRPC 客户端（mTLS：也需要提供客户端证书）
clientCert, _ := tls.LoadX509KeyPair("client.crt", "client.key")
caCert, _ := os.ReadFile("ca.crt")
caPool := x509.NewCertPool()
caPool.AppendCertsFromPEM(caCert)

tlsConfig := &tls.Config{
    Certificates: []tls.Certificate{clientCert},  // 客户端证书
    RootCAs:      caPool,                           // 验证服务端的 CA
    ServerName:   "order-api",                      // 服务端 CN/SAN
}
creds := credentials.NewTLS(tlsConfig)

conn, _ := grpc.Dial("order-api:9443", grpc.WithTransportCredentials(creds))
```

## 6. Kubernetes 中的证书管理

```yaml
# 使用 cert-manager 自动颁发和续期证书
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: order-api-tls
spec:
  secretName: order-api-tls
  issuerRef:
    name: ca-issuer
    kind: ClusterIssuer
  dnsNames:
    - order-api
    - order-api.default.svc.cluster.local
  duration: 24h       # 证书有效期
  renewBefore: 1h     # 提前1小时续期
```

```yaml
# Deployment：挂载证书 Secret
spec:
  template:
    spec:
      containers:
        - name: order-api
          volumeMounts:
            - name: tls-certs
              mountPath: /etc/certs
              readOnly: true
      volumes:
        - name: tls-certs
          secret:
            secretName: order-api-tls
```

## 7. 证书轮换

```go
// 动态加载证书（不重启服务即可更新证书）
type DynamicCertLoader struct {
    certFile string
    keyFile  string
    cert     *tls.Certificate
    mu       sync.RWMutex
}

func (d *DynamicCertLoader) GetCertificate(_ *tls.ClientHelloInfo) (*tls.Certificate, error) {
    d.mu.RLock()
    defer d.mu.RUnlock()
    return d.cert, nil
}

func (d *DynamicCertLoader) Reload() error {
    cert, err := tls.LoadX509KeyPair(d.certFile, d.keyFile)
    if err != nil {
        return err
    }
    d.mu.Lock()
    d.cert = &cert
    d.mu.Unlock()
    return nil
}

// 使用动态加载器
loader := &DynamicCertLoader{certFile: "server.crt", keyFile: "server.key"}
loader.Reload()

// 定时检查证书更新
go func() {
    ticker := time.NewTicker(1 * time.Hour)
    for range ticker.C {
        if err := loader.Reload(); err != nil {
            logx.Errorf("证书重载失败: %v", err)
        }
    }
}()
```

## 8. 总结

| 场景 | 推荐配置 |
|------|---------|
| 公网 HTTP 服务 | TLS（单向），使用 Let's Encrypt |
| 服务间内部通信 | mTLS（双向），使用内部 CA |
| K8s 集群内 | cert-manager 自动管理 + Istio 自动注入 |
| 开发环境 | 自签证书或 mkcert |

go-zero 已内置 TLS 接入能力：配置证书后即可启用加密传输；mTLS 还需要额外配置客户端证书校验与证书轮换策略。mTLS 的核心价值是实现零信任：**永不信任，始终验证**。

相关文章：
- [JWT 认证与鉴权：从入门到生产](jwt-auth-cn.md)
- [gRPC 拦截器：统一处理横切关注点](grpc-interceptors-cn.md)
