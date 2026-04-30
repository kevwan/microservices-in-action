# go-zero CI/CD 流水线：GitHub Actions 自动化部署

## 前言

手动部署既费时又容易出错。现代微服务交付依赖 CI/CD 流水线：代码提交后自动触发测试、构建镜像、推送到仓库、部署到 Kubernetes。本文以 GitHub Actions 为例，搭建 go-zero 微服务的完整 CI/CD 流水线。

## 1. 流水线总览

```
Developer Push → GitHub Actions
                    ├── 代码检查（golangci-lint）
                    ├── 单元测试（go test）
                    ├── 构建 Docker 镜像
                    ├── 推送到 Container Registry
                    └── 部署到 Kubernetes
                         ├── staging（main 分支）
                         └── production（release 标签）
```

## 2. 完整 GitHub Actions 配置

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
    tags: ['v*']
  pull_request:
    branches: [main]

env:
  GO_VERSION: "1.22"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # =================== 代码质量检查 ===================
  lint:
    name: Code Quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: golangci-lint
        uses: golangci/golangci-lint-action@v4
        with:
          version: latest
          args: --timeout=5m

  # =================== 单元测试 ===================
  test:
    name: Unit Tests
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: testdb
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s
      redis:
        image: redis:7
        ports:
          - 6379:6379
    steps:
      - uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Run Tests
        run: |
          go test -v -race -coverprofile=coverage.out \
            -covermode=atomic ./...

      - name: Upload Coverage
        uses: codecov/codecov-action@v4
        with:
          file: coverage.out

  # =================== 构建镜像 ===================
  build:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: [lint, test]
    if: github.event_name != 'pull_request'
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Docker Metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-,format=short

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and Push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # =================== 部署到 Staging ===================
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure kubectl
        run: |
          echo "${{ secrets.KUBE_CONFIG_STAGING }}" | base64 -d > $HOME/.kube/config

      - name: Deploy to Staging
        run: |
          # 更新镜像 tag
          kubectl set image deployment/order-api \
            order-api=${{ needs.build.outputs.image-tag }} \
            -n staging

          # 等待滚动更新完成
          kubectl rollout status deployment/order-api \
            -n staging --timeout=300s

      - name: Smoke Test
        run: |
          sleep 10
          curl -f https://staging.example.com/healthz || exit 1

  # =================== 部署到生产 ===================
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build
    if: startsWith(github.ref, 'refs/tags/v')
    environment:
      name: production
      url: https://example.com
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure kubectl
        run: |
          echo "${{ secrets.KUBE_CONFIG_PROD }}" | base64 -d > $HOME/.kube/config

      - name: Deploy to Production
        run: |
          kubectl set image deployment/order-api \
            order-api=${{ needs.build.outputs.image-tag }} \
            -n production

          kubectl rollout status deployment/order-api \
            -n production --timeout=600s
```

## 3. Dockerfile 多阶段构建

```dockerfile
# Dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app

# 先复制 go.mod 利用缓存（只有依赖变化才重新下载）
COPY go.mod go.sum ./
RUN go mod download

# 复制源码并构建
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-w -s \
    -X main.Version=${VERSION} \
    -X main.BuildTime=$(date +%Y%m%d%H%M%S)" \
    -o /app/server ./cmd/order-api/main.go

# 最终镜像：仅包含二进制
FROM gcr.io/distroless/static-debian12
WORKDIR /app
COPY --from=builder /app/server .
COPY --from=builder /app/etc /app/etc
EXPOSE 8080
ENTRYPOINT ["/app/server", "-f", "/app/etc/config.yaml"]
```

## 4. golangci-lint 配置

```yaml
# .golangci.yml
linters:
  enable:
    - errcheck      # 检查未处理的错误
    - gosimple      # 简化代码建议
    - govet         # go vet 检查
    - ineffassign   # 无效赋值
    - staticcheck   # 静态分析
    - unused        # 未使用的代码
    - gofmt         # 格式检查
    - goimports     # import 排序
    - gosec         # 安全检查
    - revive        # 代码规范

linters-settings:
  gosec:
    excludes:
      - G601  # 切片元素地址（Go 1.22后已修复）

issues:
  exclude-rules:
    - path: _test\.go
      linters: [gosec, errcheck]  # 测试文件宽松一些
```

## 5. 环境配置管理

```yaml
# k8s/configmap.yaml（非敏感配置）
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-api-config
data:
  config.yaml: |
    Name: order-api
    Host: 0.0.0.0
    Port: 8080
    Log:
      Level: info
      Mode: json

# k8s/secret.yaml（敏感配置，通过 Sealed Secrets 或 Vault 管理）
apiVersion: v1
kind: Secret
metadata:
  name: order-api-secrets
type: Opaque
stringData:
  DB_PASSWORD: "${DB_PASSWORD}"
  REDIS_PASSWORD: "${REDIS_PASSWORD}"
  JWT_SECRET: "${JWT_SECRET}"
```

## 6. 部署回滚

```bash
# 快速回滚到上一个版本
kubectl rollout undo deployment/order-api -n production

# 回滚到指定版本
kubectl rollout history deployment/order-api -n production
kubectl rollout undo deployment/order-api --to-revision=3 -n production
```

## 7. 总结

| 阶段 | 工具 | 触发条件 |
|------|------|---------|
| 代码检查 | golangci-lint | 所有 push 和 PR |
| 单元测试 | go test | 所有 push 和 PR |
| 构建镜像 | Docker Buildx | merge 到 main/release |
| 部署 Staging | kubectl | merge 到 main |
| 部署生产 | kubectl | 打 v* 标签 |

CI/CD 最佳实践：
1. **PR 必须通过所有 CI 检查才能合并**
2. **镜像使用不可变标签**（sha256 digest 或 commit sha）
3. **环境晋升**：先部署 staging 验证，再部署生产
4. **自动回滚**：通过 `kubectl rollout undo` 快速恢复

相关文章：
- [优雅停机：零中断滚动发布](graceful-shutdown-cn.md)
- [单元测试与集成测试](testing-guide-cn.md)
