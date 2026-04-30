# go-zero 文件上传：本地存储与对象存储

## 前言

文件上传是 Web 服务的常见需求：用户头像、商品图片、合同附件、数据导入文件。go-zero 提供了标准的 `multipart/form-data` 处理，可以轻松集成本地存储、阿里云 OSS、AWS S3 等对象存储服务。

## 1. 基础文件上传

### 1.1 定义上传接口（.api 文件）

```api
// file.api
@server (
    jwt: Auth
)
service file-api {
    @doc "上传头像"
    @handler UploadAvatar
    post /api/user/avatar returns (UploadResp)

    @doc "上传商品图片（多文件）"
    @handler UploadProductImages
    post /api/product/images returns (BatchUploadResp)
}

type UploadResp {
    Url  string `json:"url"`
    Size int64  `json:"size"`
    Name string `json:"name"`
}

type BatchUploadResp {
    Urls []string `json:"urls"`
}
```

### 1.2 Handler 层处理文件

```go
// internal/handler/file/uploadavatarhandler.go
func UploadAvatarHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // 限制请求体大小（2MB）
        r.Body = http.MaxBytesReader(w, r.Body, 2<<20)

        // 解析 multipart 表单
        if err := r.ParseMultipartForm(2 << 20); err != nil {
            httpx.Error(w, errorx.NewParamError("文件大小超过 2MB 限制"))
            return
        }

        // 获取文件
        file, header, err := r.FormFile("file")
        if err != nil {
            httpx.Error(w, errorx.NewParamError("请选择要上传的文件"))
            return
        }
        defer file.Close()

        // 调用 Logic 层处理
        l := logic.NewUploadAvatarLogic(r.Context(), svcCtx)
        resp, err := l.UploadAvatar(file, header)
        if err != nil {
            httpx.Error(w, err)
            return
        }
        httpx.OkJson(w, resp)
    }
}
```

### 1.3 Logic 层处理业务

```go
// internal/logic/file/uploadavatarlogic.go
func (l *UploadAvatarLogic) UploadAvatar(
    file multipart.File,
    header *multipart.FileHeader,
) (*types.UploadResp, error) {

    // 1. 校验文件类型（通过 Magic Bytes，而非扩展名）
    buf := make([]byte, 512)
    if _, err := file.Read(buf); err != nil {
        return nil, err
    }
    contentType := http.DetectContentType(buf)
    if !isAllowedImageType(contentType) {
        return nil, errorx.NewParamError("只支持 JPG、PNG、WEBP 格式图片")
    }
    // 重置读取位置
    file.Seek(0, io.SeekStart)

    // 2. 生成存储路径（按日期分目录，UUID 避免重名）
    ext := getExtByContentType(contentType)
    userId := l.ctx.Value("userId").(int64)
    objectKey := fmt.Sprintf("avatar/%s/%d%s",
        time.Now().Format("2006/01"), userId, ext)

    // 3. 上传到对象存储
    url, err := l.svcCtx.Storage.Upload(l.ctx, objectKey, file, header.Size, contentType)
    if err != nil {
        return nil, fmt.Errorf("文件上传失败: %w", err)
    }

    // 4. 更新用户头像
    if err := l.svcCtx.UserModel.UpdateAvatar(l.ctx, userId, url); err != nil {
        // 尝试删除已上传的文件（最终一致）
        l.svcCtx.Storage.Delete(l.ctx, objectKey)
        return nil, err
    }

    return &types.UploadResp{
        Url:  url,
        Size: header.Size,
        Name: header.Filename,
    }, nil
}

func isAllowedImageType(contentType string) bool {
    allowed := map[string]bool{
        "image/jpeg": true,
        "image/png":  true,
        "image/webp": true,
        "image/gif":  true,
    }
    return allowed[contentType]
}
```

## 2. 对象存储集成

### 2.1 抽象存储接口

```go
// internal/storage/storage.go
type Storage interface {
    Upload(ctx context.Context, key string, reader io.Reader, size int64, contentType string) (url string, err error)
    Delete(ctx context.Context, key string) error
    GetPresignedURL(ctx context.Context, key string, expire time.Duration) (string, error)
}
```

### 2.2 阿里云 OSS 实现

```go
// internal/storage/oss.go
import "github.com/aliyun/aliyun-oss-go-sdk/oss"

type OSSStorage struct {
    client     *oss.Client
    bucket     *oss.Bucket
    bucketName string
    cdnDomain  string
}

func NewOSSStorage(endpoint, accessKey, secretKey, bucketName, cdnDomain string) (*OSSStorage, error) {
    client, err := oss.New(endpoint, accessKey, secretKey)
    if err != nil {
        return nil, err
    }
    bucket, err := client.Bucket(bucketName)
    if err != nil {
        return nil, err
    }
    return &OSSStorage{
        client: client, bucket: bucket,
        bucketName: bucketName, cdnDomain: cdnDomain,
    }, nil
}

func (s *OSSStorage) Upload(ctx context.Context, key string,
    reader io.Reader, size int64, contentType string) (string, error) {

    err := s.bucket.PutObject(key, reader,
        oss.ContentType(contentType),
        oss.ContentLength(size),
    )
    if err != nil {
        return "", err
    }
    return fmt.Sprintf("https://%s/%s", s.cdnDomain, key), nil
}
```

### 2.3 AWS S3 实现

```go
// internal/storage/s3.go
import (
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

type S3Storage struct {
    client    *s3.Client
    bucket    string
    cdnDomain string
}

func (s *S3Storage) Upload(ctx context.Context, key string,
    reader io.Reader, size int64, contentType string) (string, error) {

    _, err := s.client.PutObject(ctx, &s3.PutObjectInput{
        Bucket:        aws.String(s.bucket),
        Key:           aws.String(key),
        Body:          reader,
        ContentType:   aws.String(contentType),
        ContentLength: aws.Int64(size),
    })
    if err != nil {
        return "", err
    }
    return fmt.Sprintf("https://%s/%s", s.cdnDomain, key), nil
}
```

## 3. 预签名 URL：客户端直传

对于大文件，让客户端直接上传到 OSS/S3，绕过服务器，节省带宽：

```
传统方式：                        预签名直传：
客户端 → 服务器 → OSS             客户端 → 服务器（获取签名URL）
   文件走两次网络                   客户端 → OSS（直接上传）
                                      只有签名走服务器
```

```go
// Logic 层生成预签名 URL
func (l *GetUploadURLLogic) GetUploadURL(req *types.GetUploadURLReq) (*types.GetUploadURLResp, error) {
    // 校验文件类型
    if !isAllowedType(req.FileType) {
        return nil, errorx.NewParamError("不支持的文件类型")
    }

    // 生成唯一的对象 key
    userId := l.ctx.Value("userId").(int64)
    ext := mimeToExt[req.FileType]
    objectKey := fmt.Sprintf("uploads/%s/%d-%s%s",
        time.Now().Format("2006/01/02"), userId, uuid.NewString(), ext)

    // 生成 15 分钟有效的预签名上传 URL
    presignedURL, err := l.svcCtx.Storage.GetPresignedUploadURL(
        l.ctx, objectKey, req.FileType, 15*time.Minute)
    if err != nil {
        return nil, err
    }

    return &types.GetUploadURLResp{
        UploadUrl: presignedURL,
        ObjectKey: objectKey,
        // 上传完成后客户端用此 key 通知服务端
    }, nil
}
```

## 4. 多文件上传

```go
func UploadProductImagesHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // 最多 9 张，每张最大 5MB
        r.Body = http.MaxBytesReader(w, r.Body, 9*5<<20)
        if err := r.ParseMultipartForm(9 * 5 << 20); err != nil {
            httpx.Error(w, errorx.NewParamError("文件总大小超过 45MB"))
            return
        }

        files := r.MultipartForm.File["files"]
        if len(files) == 0 {
            httpx.Error(w, errorx.NewParamError("请选择图片"))
            return
        }
        if len(files) > 9 {
            httpx.Error(w, errorx.NewParamError("最多上传9张图片"))
            return
        }

        l := logic.NewUploadProductImagesLogic(r.Context(), svcCtx)
        resp, err := l.UploadProductImages(files)
        if err != nil {
            httpx.Error(w, err)
            return
        }
        httpx.OkJson(w, resp)
    }
}
```

## 5. 安全注意事项

```go
// ❌ 危险：通过扩展名判断文件类型
ext := filepath.Ext(header.Filename)
if ext != ".jpg" { ... }  // 攻击者可以把 shell.php 改名为 shell.jpg

// ✅ 安全：通过 Magic Bytes 判断
buf := make([]byte, 512)
file.Read(buf)
contentType := http.DetectContentType(buf)  // 读取文件头，无法伪造

// ✅ 安全：对象 key 不使用用户传入的文件名
// 攻击者可能传入 ../../etc/passwd 等路径穿越路径
objectKey := fmt.Sprintf("avatar/%s/%s", time.Now().Format("2006/01"), uuid.NewString())

// ✅ 安全：限制文件大小（防止 DoS）
r.Body = http.MaxBytesReader(w, r.Body, maxSize)
```

## 6. 总结

go-zero 文件上传实现要点：

1. **类型校验**：用 Magic Bytes，不信任扩展名和 Content-Type 头
2. **大小限制**：用 `MaxBytesReader` 在网络层截断大文件
3. **安全路径**：服务端生成 UUID 路径，不使用用户传入文件名
4. **大文件**：使用预签名 URL 让客户端直传，绕过服务器
5. **存储接口**：抽象 `Storage` 接口，方便切换存储后端（本地/OSS/S3）
