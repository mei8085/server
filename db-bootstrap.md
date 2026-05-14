# 数据库启动到就绪全流程报告

## 一、概述

本文档详细描述 Gotify Server 服务启动过程中的数据库连通性探测、首次初始化流程，以及失败时的恢复路径。

---

## 二、服务启动入口与整体流程

### 2.1 主入口文件

**文件位置**: `app.go:26-54`

服务启动的主流程：
```go
func main() {
    // 1. 初始化版本信息
    vInfo := &model.VersionInfo{...}
    
    // 2. 加载配置
    conf := config.Get()
    
    // 3. 创建必要的目录（插件、图片存储）
    os.MkdirAll(conf.PluginsDir, 0o755)
    os.MkdirAll(conf.UploadedImagesDir, 0o755)
    
    // 4. 数据库初始化（核心）
    db, err := database.New(...)
    if err != nil {
        panic(err)  // 失败直接终止
    }
    defer db.Close()
    
    // 5. 创建路由
    engine, closeable := router.Create(db, vInfo, conf)
    
    // 6. 启动 HTTP/HTTPS 服务
    runner.Run(engine, conf)
}
```

### 2.2 关键时序

```
服务启动
    ↓
配置加载 (config.Get())
    ↓
目录创建 (PluginsDir, UploadedImagesDir)
    ↓
数据库初始化 (database.New())
    ├─ SQLite 目录预处理
    ├─ 建立数据库连接
    ├─ 连接池配置
    ├─ 自动迁移 (AutoMigrate)
    ├─ 默认用户创建
    └─ 数据迁移（填充 sort_key）
    ↓
路由创建与服务启动
```

---

## 三、数据库连通性探测机制

### 3.1 启动时隐式探测

**文件位置**: `database/database.go:45-56`

在 `database.New()` 函数中，通过 `gorm.Open()` 建立连接时会隐式进行连通性探测：

```go
switch dialect {
case "mysql":
    db, err = gorm.Open(mysql.Open(connection), gormConfig)
case "postgres":
    db, err = gorm.Open(postgres.Open(connection), gormConfig)
case "sqlite3":
    db, err = gorm.Open(sqlite.Open(connection), gormConfig)
}

if err != nil {
    return nil, err  // 连接失败直接返回错误
}
```

### 3.2 运行时健康检查探测

#### 3.2.1 Ping 接口实现

**文件位置**: `database/ping.go:3-10`

```go
// Ping 验证数据库连接是否存活
func (d *GormDatabase) Ping() error {
    sqldb, err := d.DB.DB()
    if err != nil {
        return err
    }
    return sqldb.Ping()
}
```

#### 3.2.2 Health API 端点

**文件位置**: `api/health.go:8-46`

```go
// Health 返回健康信息
func (a *HealthAPI) Health(ctx *gin.Context) {
    if err := a.DB.Ping(); err != nil {
        ctx.JSON(500, model.Health{
            Health:   model.StatusOrange,
            Database: model.StatusRed,
        })
        return
    }
    ctx.JSON(200, model.Health{
        Health:   model.StatusGreen,
        Database: model.StatusGreen,
    })
}
```

#### 3.2.3 健康状态定义

**文件位置**: `model/health.go:21-27`

| 状态值 | 说明 |
|--------|------|
| `green` | 正常 |
| `orange` | 部分异常 |
| `red` | 异常 |

### 3.3 探测时机

1. **启动时**: `gorm.Open()` 建立连接时自动探测
2. **运行时**: 通过 `/health` 端点主动探测（可用于 Kubernetes liveness/readiness probe）

---

## 四、数据库首次初始化全流程

**核心文件**: `database/database.go:27-98`

### 4.1 阶段一：SQLite 目录预处理

**文件位置**: `database/database.go:133-141`

```go
func createDirectoryIfSqlite(dialect, connection string) {
    if dialect == "sqlite3" {
        if _, err := os.Stat(filepath.Dir(connection)); os.IsNotExist(err) {
            if err := mkdirAll(filepath.Dir(connection), 0o777); err != nil {
                panic(err)  // 目录创建失败直接 panic
            }
        }
    }
}
```

**说明**:
- 仅对 SQLite3 数据库生效
- 递归创建数据库文件所在目录
- 目录创建失败时直接 panic 终止服务

### 4.2 阶段二：建立数据库连接

**文件位置**: `database/database.go:45-56`

支持三种数据库方言：
- **MySQL**: `gorm.io/driver/mysql`
- **PostgreSQL**: `gorm.io/driver/postgres`
- **SQLite3**: `gorm.io/driver/sqlite`

**GORM 配置** (`database.go:30-40`):
- 慢查询阈值: 200ms
- 日志级别: Warn
- 忽略 RecordNotFound 错误
- 禁用外键约束迁移
- 启用错误翻译

### 4.3 阶段三：连接池配置

**文件位置**: `database/database.go:58-81`

```go
sqldb, err := db.DB()
if err != nil {
    return nil, err
}

// 全局连接数限制：最多 10 个连接
sqldb.SetMaxOpenConns(10)

// SQLite 特殊处理：并发写入限制为 1 个连接
if dialect == "sqlite3" {
    sqldb.SetMaxOpenConns(1)
}

// MySQL 连接生命周期：9 分钟（小于 wait_timeout 默认值 10 分钟）
if dialect == "mysql" {
    sqldb.SetConnMaxLifetime(9 * time.Minute)
}
```

**设计考量**:
- MySQL 的 `wait_timeout` 默认为 10 分钟，设置 9 分钟避免"断开的连接"问题
- SQLite 不支持并发写入，限制为单连接
- 全局连接数限制防止"too many connections"错误

### 4.4 阶段四：自动迁移（AutoMigrate）

**文件位置**: `database/database.go:83-85`

```go
if err := db.AutoMigrate(
    new(model.User),
    new(model.Application),
    new(model.Message),
    new(model.Client),
    new(model.PluginConf),
); err != nil {
    return nil, err
}
```

**迁移内容**:
- 创建/更新表结构
- 创建索引
- 不删除已有数据

### 4.5 阶段五：默认管理员用户创建

**文件位置**: `database/database.go:87-91`

```go
userCount := int64(0)
db.Find(new(model.User)).Count(&userCount)

if createDefaultUserIfNotExist && userCount == 0 {
    db.Create(&model.User{
        Name:  defaultUser,
        Pass:  password.CreatePassword(defaultPass, strength),
        Admin: true,
    })
}
```

**说明**:
- 仅当数据库中无用户时才创建
- 默认用户名/密码由配置决定（默认: admin/admin）
- 密码使用 bcrypt 加密，强度可配置

### 4.6 阶段六：数据迁移 - 填充 Sort Key

**文件位置**: `database/database.go:93-131`

```go
if err := db.Transaction(fillMissingSortKeys, &sql.TxOptions{
    Isolation: sql.LevelSerializable,
}); err != nil {
    return nil, err
}
```

**功能**:
- 为缺少 `sort_key` 的应用记录生成分层排序键
- 使用可序列化事务隔离级别保证数据一致性
- 按用户分组，每个用户内部独立排序

---

## 五、失败场景与恢复路径

### 5.1 启动时失败场景

#### 5.1.1 配置加载失败

**位置**: `config/config.go:79-87`

```go
func Get() *Configuration {
    conf := new(Configuration)
    err := configor.New(...).Load(conf, configFiles()...)
    if err != nil {
        panic(err)  // 直接终止
    }
    return conf
}
```

**失败原因**:
- 配置文件格式错误
- 环境变量格式错误

**恢复路径**:
1. 服务立即终止（panic）
2. 检查 stderr 日志获取详细错误信息
3. 修正配置文件或环境变量
4. 手动重启服务

#### 5.1.2 目录创建失败

**位置**: `app.go:33-40` 与 `database/database.go:136-138`

**失败原因**:
- 权限不足
- 磁盘空间不足
- 文件系统只读

**恢复路径**:
1. 服务立即终止（panic）
2. 检查数据目录权限
3. 确保磁盘有足够空间
4. 手动重启服务

#### 5.1.3 数据库连接失败

**位置**: `database/database.go:54-56`

**失败原因**:
- 数据库服务未启动
- 连接字符串错误
- 认证失败（用户名/密码错误）
- 网络不可达
- 数据库不存在（MySQL/PostgreSQL）

**恢复路径**:
1. 服务立即终止，返回错误
2. 检查数据库服务状态
3. 验证数据库连接配置
4. 确保网络连通性
5. 手动重启服务

#### 5.1.4 自动迁移失败

**位置**: `database/database.go:83-85`

**失败原因**:
- 表结构变更冲突
- 数据库权限不足
- 磁盘空间不足

**恢复路径**:
1. 服务立即终止
2. 检查数据库日志
3. 可能需要手动修复表结构
4. 手动重启服务

#### 5.1.5 数据迁移（Sort Key）失败

**位置**: `database/database.go:93-95`

**失败原因**:
- 事务执行失败
- 数据一致性问题

**恢复路径**:
1. 服务立即终止
2. 检查错误日志
3. 可能需要手动修复数据
4. 手动重启服务

### 5.2 运行时失败场景

#### 5.2.1 数据库连接中断

**检测方式**:
- 通过 `/health` 端点进行健康检查
- 返回 HTTP 500 状态码

**响应内容**:
```json
{
    "health": "orange",
    "database": "red"
}
```

**恢复路径**:
1. 监控系统检测到健康检查失败
2. 容器编排系统（如 Kubernetes）自动重启服务
3. 重启时重新执行完整的初始化流程
4. GORM 内置连接池会自动重连（不重启也可能恢复）

#### 5.2.2 数据库查询失败

**行为**:
- 单个请求失败，返回 500 错误
- 服务不会整体崩溃
- 连接池会尝试重新建立连接

---

## 六、配置参数说明

**文件位置**: `config/config.go:47-54`

```go
Database struct {
    Dialect    string `default:"sqlite3"`  // 数据库类型: sqlite3, mysql, postgres
    Connection string `default:"data/gotify.db"`  // 连接字符串/文件路径
}
DefaultUser struct {
    Name string `default:"admin"`  // 默认管理员用户名
    Pass string `default:"admin"`  // 默认管理员密码
}
PassStrength int `default:"10"`  // bcrypt 加密强度
```

**环境变量前缀**: `GOTIFY_`

示例：
- `GOTIFY_DATABASE_DIALECT`
- `GOTIFY_DATABASE_CONNECTION`
- `GOTIFY_DEFAULTUSER_NAME`
- `GOTIFY_DEFAULTUSER_PASS`

---

## 七、生产环境部署建议

### 7.1 Kubernetes 部署

**健康检查配置**:
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 7.2 数据库高可用

1. **MySQL/PostgreSQL**: 使用主从复制或集群部署
2. **SQLite**: 仅适合单实例部署，不适合高可用场景
3. **连接池**: 当前配置为 10 个连接，根据负载调整

### 7.3 监控告警

- 监控 `/health` 端点返回状态
- 告警阈值：连续 3 次返回非 green 状态
- 关注数据库连接错误日志

---

## 八、关键代码索引

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| 主入口 | `app.go` | 26-54 |
| 数据库初始化 | `database/database.go` | 27-98 |
| SQLite 目录创建 | `database/database.go` | 133-141 |
| 连接池配置 | `database/database.go` | 58-81 |
| Ping 方法 | `database/ping.go` | 3-10 |
| 健康检查 API | `api/health.go` | 8-46 |
| 健康状态模型 | `model/health.go` | 1-28 |
| 配置加载 | `config/config.go` | 78-87 |

---

## 九、总结

### 9.1 启动流程特点

1. **全有或全无**: 任何初始化步骤失败都会导致服务终止
2. **无重试机制**: 启动时失败直接终止，不进行重试
3. **幂等设计**: 初始化流程可安全重复执行

### 9.2 恢复策略

| 失败阶段 | 恢复方式 | 自动化程度 |
|---------|---------|-----------|
| 配置加载 | 手动修正配置后重启 | 手动 |
| 目录创建 | 修复权限/磁盘后重启 | 手动 |
| 数据库连接 | 修复数据库后重启 | 手动/自动（K8s） |
| 自动迁移 | 修复数据库结构后重启 | 手动 |
| 运行时中断 | 健康检查失败后重启 | 自动（K8s） |

### 9.3 设计考量

- 简单可靠：失败即终止，避免不一致状态
- 幂等安全：重复初始化不会破坏数据
- 运维友好：通过健康检查端点支持容器编排
