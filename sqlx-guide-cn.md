# go-zero sqlx：数据库操作详解

## 前言

go-zero 的 `sqlx` 包是对标准库 `database/sql` 的增强封装，提供了更简洁的 API、内置的连接池管理、慢查询日志和链路追踪集成。结合 goctl 生成的 Model 层代码，可以极大减少数据库操作的样板代码。

## 1. 连接配置

```yaml
# config.yaml
DataSource: "user:password@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"
```

```go
// internal/svc/servicecontext.go
import "github.com/zeromicro/go-zero/core/stores/sqlx"

type ServiceContext struct {
    Config    config.Config
    DB        sqlx.SqlConn
}

func NewServiceContext(c config.Config) *ServiceContext {
    return &ServiceContext{
        Config: c,
        DB:     sqlx.NewMysql(c.DataSource),
    }
}
```

## 2. goctl 生成 Model（推荐方式）

```sql
-- 建表 SQL
CREATE TABLE `user` (
    `id`         bigint NOT NULL AUTO_INCREMENT,
    `username`   varchar(64) NOT NULL UNIQUE,
    `email`      varchar(128) NOT NULL,
    `status`     tinyint NOT NULL DEFAULT 1,
    `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

```bash
# 根据 SQL 文件生成 Model（含缓存）
goctl model mysql ddl -src user.sql -dir ./internal/model --cache

# 不带缓存
goctl model mysql ddl -src user.sql -dir ./internal/model
```

生成的 Model 包含以下方法：
- `Insert(ctx, data)` — 插入
- `FindOne(ctx, id)` — 按主键查询
- `FindOneByUsername(ctx, username)` — 按唯一键查询（自动生成）
- `Update(ctx, data)` — 更新
- `Delete(ctx, id)` — 删除

## 3. 自定义查询

goctl 生成的代码只包含主键和唯一键查询，复杂查询需要手动编写：

```go
// internal/model/usermodel_extra.go（扩展方法，不会被 goctl 覆盖）

// 分页查询
func (m *defaultUserModel) FindByStatus(ctx context.Context,
    status int, page, pageSize int) ([]*User, error) {

    var users []*User
    query := fmt.Sprintf(
        "SELECT %s FROM %s WHERE `status` = ? ORDER BY id DESC LIMIT ? OFFSET ?",
        userRows, m.table)

    err := m.conn.QueryRowsCtx(ctx, &users, query, status, pageSize, (page-1)*pageSize)
    if errors.Is(err, sqlx.ErrNotFound) {
        return nil, nil
    }
    return users, err
}

// 统计数量
func (m *defaultUserModel) CountByStatus(ctx context.Context, status int) (int64, error) {
    var count int64
    query := fmt.Sprintf("SELECT COUNT(*) FROM %s WHERE `status` = ?", m.table)
    err := m.conn.QueryRowCtx(ctx, &count, query, status)
    return count, err
}

// 关联查询
type UserWithOrders struct {
    User
    OrderCount int `db:"order_count"`
}

func (m *defaultUserModel) FindUsersWithOrderCount(ctx context.Context,
    startTime, endTime time.Time) ([]*UserWithOrders, error) {

    var result []*UserWithOrders
    query := `
        SELECT u.*, COUNT(o.id) as order_count
        FROM user u
        LEFT JOIN ` + "`order`" + ` o ON u.id = o.user_id
            AND o.created_at BETWEEN ? AND ?
        GROUP BY u.id
        ORDER BY order_count DESC
        LIMIT 100
    `
    err := m.conn.QueryRowsCtx(ctx, &result, query, startTime, endTime)
    return result, err
}
```

## 4. 原生 sqlx 操作

当不使用 goctl 生成的 Model 时，直接使用 `sqlx.SqlConn`：

```go
// QueryRowCtx：查询单行，扫描到结构体
var user User
err := conn.QueryRowCtx(ctx, &user,
    "SELECT id, username, email FROM user WHERE id = ?", userId)
if errors.Is(err, sqlx.ErrNotFound) {
    return nil, ErrUserNotFound
}

// QueryRowsCtx：查询多行
var users []*User
err = conn.QueryRowsCtx(ctx, &users,
    "SELECT id, username, email FROM user WHERE status = ? LIMIT 100", 1)

// ExecCtx：执行 INSERT/UPDATE/DELETE
result, err := conn.ExecCtx(ctx,
    "UPDATE user SET status = ? WHERE id = ?", newStatus, userId)
rowsAffected, _ := result.RowsAffected()

// 扫描到 map（灵活但类型不安全）
var m sqlx.NullStringMap
err = conn.QueryRowCtx(ctx, &m,
    "SELECT * FROM user WHERE id = ?", userId)
```

## 5. 批量操作

```go
// 批量插入（使用 BulkInsert）
import "github.com/zeromicro/go-zero/core/stores/sqlx"

type Inserter struct {
    *sqlx.BulkInserter
}

func NewBatchInserter(conn sqlx.SqlConn) *Inserter {
    inserter, _ := sqlx.NewBulkInserter(conn,
        "INSERT INTO product(name, price, stock) VALUES (?, ?, ?)")
    return &Inserter{inserter}
}

// 使用
inserter := NewBatchInserter(conn)
for _, p := range products {
    inserter.Insert(p.Name, p.Price, p.Stock)
}
inserter.Flush() // 强制刷新剩余数据（通常不需要手动调用）
```

## 6. 事务操作

```go
// TransactCtx：事务包装
err = conn.TransactCtx(ctx, func(ctx context.Context, tx sqlx.Session) error {
    // tx 实现了 sqlx.Session 接口，可以使用同样的查询方法
    _, err := tx.ExecCtx(ctx,
        "UPDATE account SET balance = balance - ? WHERE user_id = ? AND balance >= ?",
        amount, fromUser, amount)
    if err != nil {
        return err // 自动回滚
    }

    _, err = tx.ExecCtx(ctx,
        "UPDATE account SET balance = balance + ? WHERE user_id = ?",
        amount, toUser)
    return err // 返回 nil 则提交，否则回滚
})
```

## 7. 慢查询日志

go-zero sqlx 自动记录慢查询：

```yaml
# config.yaml
Log:
  SlowThreshold: 500ms  # 超过 500ms 的查询会被记录
```

慢查询日志示例：
```
{"level":"error","content":"slow sql","duration":"1.234s",
 "sql":"SELECT * FROM order WHERE user_id = ? LIMIT 100",
 "args":[42]}
```

## 8. 结合 Redis 缓存（goctl 生成）

goctl 生成的 Model 自动集成 Redis 缓存：

```go
// 使用缓存 Model
rds := redis.MustNewRedis(c.RedisConf)
conn := sqlx.NewMysql(c.DataSource)
userModel := model.NewUserModel(conn, rds)

// FindOne 会先查缓存，缓存miss才查DB，并回写缓存
user, err := userModel.FindOne(ctx, userId)

// Insert/Update/Delete 会自动清缓存
err = userModel.Update(ctx, updatedUser)
```

## 9. 总结

| 操作 | 方法 |
|------|------|
| 查询单行 | `QueryRowCtx` |
| 查询多行 | `QueryRowsCtx` |
| 执行语句 | `ExecCtx` |
| 事务 | `TransactCtx` |
| 批量插入 | `BulkInserter` |

最佳实践：
1. 优先使用 goctl 生成 Model，减少样板代码
2. 复杂查询在 `*model_extra.go` 中扩展，避免 goctl 重新生成时覆盖
3. 始终传递 `ctx`，享受链路追踪和超时控制
4. 利用 goctl 的缓存集成，减少对数据库的压力

相关文章：
- [数据库事务：保证数据一致性](db-transaction-cn.md)
- [数据库读写分离](db-readwrite-split-cn.md)
- [缓存设计最佳实践](cache-design-cn.md)
