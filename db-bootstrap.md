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

## 五、失败场景深度分析

### 5.1 启动失败线（Bootstrap Failure）

启动失败发生在 `main()` 执行到 `runner.Run()` 之前，服务尚未接受请求。所有启动失败都会导致进程终止。

---

#### 5.1.1 配置加载失败

| 项 | 详情 |
|----|------|
| **触发条件** | 1. config.yml 文件不存在且格式错误<br>2. 环境变量格式不符合预期<br>3. 配置文件 YAML 语法错误 |
| **错误传播链** | `configor.Load()` → `return error` → `panic(err)` → 进程退出 |
| **错误类型** | panic |
| **自动恢复** | ❌ 否（进程直接终止） |
| **错误日志位置** | stderr，直接输出 panic 堆栈 |
| **运维动作** | 1. 检查 `config.yml` 文件是否存在且格式正确<br>2. 验证所有 `GOTIFY_*` 环境变量格式<br>3. 修复后手动重启服务 |

**代码调用栈**:
```
main()
  → config.Get()
    → configor.Load()
      → 检测到错误 → panic(err)
  → 进程终止，退出码 != 0
```

---

#### 5.1.2 应用目录创建失败（Plugins/Images）

| 项 | 详情 |
|----|------|
| **触发条件** | 1. 数据目录权限不足（无法写入）<br>2. 磁盘空间已满<br>3. 文件系统挂载为只读 |
| **错误传播链** | `os.MkdirAll()` → `return error` → `panic(err)` → 进程退出 |
| **错误类型** | panic |
| **自动恢复** | ❌ 否 |
| **错误日志位置** | stderr |
| **运维动作** | 1. 检查 `data/` 目录权限（UID/GID映射）<br>2. 验证磁盘空间 `df -h`<br>3. 检查挂载点是否为只读<br>4. 修复后手动重启 |

---

#### 5.1.3 SQLite 数据目录创建失败

| 项 | 详情 |
|----|------|
| **触发条件** | 1. SQLite 数据库文件所在目录不存在且无法创建<br>2. 同上权限/磁盘问题 |
| **错误传播链** | `createDirectoryIfSqlite()` → `mkdirAll()` → `panic(err)` → 进程退出 |
| **错误类型** | panic |
| **自动恢复** | ❌ 否 |
| **错误日志位置** | stderr |
| **运维动作** | 1. 检查 `data/gotify.db` 上级目录权限<br>2. 对于 Docker 部署，检查 volume 挂载<br>3. 修复后手动重启 |

---

#### 5.1.4 数据库连接建立失败

| 项 | 详情 |
|----|------|
| **触发条件** | 1. MySQL/PostgreSQL 服务未启动<br>2. 连接字符串格式错误<br>3. 用户名/密码认证失败<br>4. 网络不可达（防火墙/安全组）<br>5. 数据库不存在（需要预先创建） |
| **错误传播链** | `gorm.Open()` → `return error` → `database.New()` → `return nil, err` → `main()` → `panic(err)` → 进程退出 |
| **错误类型** | panic（经 error 传递后） |
| **自动恢复** | ❌ 否（进程终止）<br>⚠️ K8s 环境下：Deployment 会自动重启 Pod |
| **错误日志位置** | stderr + 数据库驱动日志 |
| **运维动作** | 1. 验证数据库服务状态 `telnet host port`<br>2. 检查连接字符串格式<br>3. 手动验证数据库凭据<br>4. 检查网络连通性/防火墙规则<br>5. 确保数据库已创建<br>6. 修复后手动重启 |

**调用栈传播**:
```
main()
  → database.New()
    → gorm.Open(mysql.Open(DSN))
      → 驱动连接失败 → return error
    → 收到 error → return nil, err
  → main() 收到 err → panic(err)
→ 进程终止
```

---

#### 5.1.5 连接池配置失败

| 项 | 详情 |
|----|------|
| **触发条件** | 1. `db.DB()` 获取底层 sql.DB 失败（极罕见）<br>2. GORM 内部状态异常 |
| **错误传播链** | `db.DB()` → `return error` → `database.New()` → `return nil, err` → `panic(err)` → 进程退出 |
| **错误类型** | panic |
| **自动恢复** | ❌ 否 |
| **错误日志位置** | stderr |
| **运维动作** | 1. 这是内部错误，通常伴随连接失败同时发生<br>2. 检查 GORM 版本兼容性<br>3. 重启服务重试 |

---

#### 5.1.6 AutoMigrate 自动迁移失败

| 项 | 详情 |
|----|------|
| **触发条件** | 1. 表结构变更与现有数据冲突<br>2. 数据库用户缺少 ALTER/CREATE 权限<br>3. 磁盘空间不足无法创建索引<br>4. 外键约束问题（已禁用但仍可能发生） |
| **错误传播链** | `db.AutoMigrate()` → `return error` → `database.New()` → `return nil, err` → `panic(err)` → 进程退出 |
| **错误类型** | panic |
| **自动恢复** | ❌ 否（需要人工介入修复数据） |
| **错误日志位置** | stderr + 数据库日志 |
| **运维动作** | 1. 检查数据库日志获取具体失败的 SQL<br>2. 验证数据库用户权限<br>3. 如需手动迁移，先备份数据库<br>4. 手动执行 DDL 修复后重启服务 |

---

#### 5.1.7 默认用户创建失败（隐式）

> ⚠️ **重要**: 此步骤**不会导致启动失败**

| 项 | 详情 |
|----|------|
| **触发条件** | 1. `db.Find().Count()` 查询失败<br>2. `db.Create()` 用户插入失败 |
| **错误传播链** | ❌ 错误被忽略，无返回，无 panic |
| **错误类型** | 静默失败（Silent Failure） |
| **自动恢复** | ⚠️ 下次重启时重试 |
| **错误日志位置** | ❌ 无日志（需要改进） |
| **运维动作** | 1. 检查 `/api/user` 端点是否返回正常<br>2. 如无法登录，检查 users 表<br>3. 手动创建管理员用户 |

---

#### 5.1.8 Sort Key 数据迁移失败

| 项 | 详情 |
|----|------|
| **触发条件** | 1. 事务执行失败（死锁/超时）<br>2. 现有数据导致 fracdex 生成失败<br>3. 序列化事务隔离级别冲突 |
| **错误传播链** | `db.Transaction()` → `return error` → `database.New()` → `return nil, err` → `panic(err)` → 进程退出 |
| **错误类型** | panic |
| **自动恢复** | ❌ 否 |
| **错误日志位置** | stderr |
| **运维动作** | 1. 检查 applications 表中 sort_key 字段状态<br>2. 手动清理异常数据<br>3. 重启服务 |

---

### 5.2 运行时失败线（Runtime Failure）

运行时失败发生在 `runner.Run()` 之后，服务已启动并接受请求。

---

#### 5.2.1 数据库连接池耗尽

| 项 | 详情 |
|----|------|
| **触发条件** | 1. 并发请求超过 10（默认连接池大小）<br>2. 慢查询导致连接长时间占用<br>3. 连接泄漏（未释放） |
| **错误传播链** | `db.Exec/Query()` → 等待连接超时 → `return error` → API handler → HTTP 500 |
| **错误类型** | 返回 error（单请求失败） |
| **自动恢复** | ✅ 是（连接释放后自动恢复） |
| **错误日志位置** | GORM 慢查询日志（Warn 级别） |
| **运维动作** | 1. 监控连接池状态（需额外指标）<br>2. 优化慢查询<br>3. 考虑增大 `SetMaxOpenConns`<br>4. 高峰期无需重启，自动恢复 |

---

#### 5.2.2 数据库连接临时中断（网络闪断）

| 项 | 详情 |
|----|------|
| **触发条件** | 1. 数据库服务重启<br>2. 网络临时中断<br>3. 防火墙会话超时 |
| **错误传播链** | 查询执行 → 检测到坏连接 → `return error` → 连接池丢弃坏连接 → 下次请求自动重连 |
| **错误类型** | 返回 error（单请求失败） |
| **自动恢复** | ✅ 是（sql.DB 内置连接池自动重连） |
| **错误日志位置** | HTTP 500 错误日志 |
| **运维动作** | 1. 无需立即介入，观察是否自动恢复<br>2. 如持续失败，检查数据库状态<br>3. 必要时重启服务 |

**恢复机制时序**:
```
T0: 连接中断，查询失败 → 返回 500
T1: sql.DB 检测到坏连接，从池中移除
T2: 下一个请求到来 → 创建新连接（gorm.Open 逻辑）
T3: 新连接建立成功 → 恢复正常
```

---

#### 5.2.3 MySQL wait_timeout 连接过期

| 项 | 详情 |
|----|------|
| **触发条件** | 连接空闲超过 9 分钟（小于 MySQL 默认 10 分钟） |
| **错误传播链** | ✅ 已通过 `SetConnMaxLifetime(9min)` 预防 |
| **错误类型** | ✅ 不会发生（设计预防） |
| **自动恢复** | ✅ 连接池自动回收重建 |
| **错误日志位置** | 无 |
| **运维动作** | 无需处理，自动管理 |

---

#### 5.2.4 Health Check 检测到数据库异常

| 项 | 详情 |
|----|------|
| **触发条件** | 1. 所有连接都失效<br>2. 数据库长时间不可用<br>3. `/health` 被调用时 Ping() 失败 |
| **错误传播链** | `Health()` → `db.Ping()` → `return error` → HTTP 500 + Health: orange, Database: red |
| **错误类型** | HTTP 错误响应（服务仍在运行） |
| **自动恢复** | ⚠️ 部分恢复（依赖 K8s）<br>1. Kubernetes livenessProbe 检测到 500 → 杀死 Pod<br>2. Pod 重启 → 重新走完整初始化流程 |
| **错误日志位置** | HTTP 访问日志 + Gin 错误日志 |
| **运维动作** | 1. 监控告警触发后确认数据库状态<br>2. 如数据库正常但服务异常，手动触发重启<br>3. 检查 Pod 重启原因 `kubectl describe pod` |

---

#### 5.2.5 数据库查询执行失败

| 项 | 详情 |
|----|------|
| **触发条件** | 1. SQL 语法错误（代码 bug）<br>2. 数据完整性约束违反<br>3. 死锁/锁等待超时 |
| **错误传播链** | GORM 执行 → `return error` → Handler 处理 → HTTP 500/400 |
| **错误类型** | 单请求失败 |
| **自动恢复** | ⚠️ 视情况：<br>✅ 死锁: 下次请求自动重试成功<br>❌ 代码 bug: 永久失败，需发版修复 |
| **错误日志位置** | GORM 错误日志 |
| **运维动作** | 1. 区分是偶发还是持续失败<br>2. 持续失败需开发介入修复<br>3. 偶发死锁无需干预 |

---

## 六、恢复决策总表

### 6.1 从 main 到 health 的完整恢复决策矩阵

| 失败阶段 | 检测点 | 触发条件 | 错误传播方式 | 进程是否终止 | K8s 自动重启 | 手动恢复动作 | 预计恢复时间 |
|---------|--------|---------|-------------|-------------|--------------|-------------|-------------|
| **配置加载** | `config.Get()` | 配置文件/环境变量错误 | panic | ✅ 是 | ❌ 配置错误重启也没用 | 修正配置后重启 | 取决于问题定位速度 |
| **目录创建** | `os.MkdirAll()` | 权限/磁盘问题 | panic | ✅ 是 | ⚠️ 目录问题需人工修复 | 修复权限/磁盘后重启 | 分钟级 |
| **SQLite 目录** | `createDirectoryIfSqlite()` | 目录无法创建 | panic | ✅ 是 | ⚠️ 同上 | 检查 volume 挂载后重启 | 分钟级 |
| **数据库连接** | `gorm.Open()` | DB 不可达/认证失败 | return error → panic | ✅ 是 | ✅ 数据库恢复后自动重启成功 | 检查数据库状态，确认恢复后等待自动重启 | 数据库恢复 + Pod 重启（秒-分钟级） |
| **连接池配置** | `db.DB()` | 内部错误 | return error → panic | ✅ 是 | ✅ 偶发问题重启可能解决 | 检查 GORM 日志后重启 | 分钟级 |
| **AutoMigrate** | `db.AutoMigrate()` | DDL 执行失败 | return error → panic | ✅ 是 | ❌ 需手动修复数据库 | 备份 + 手动迁移 + 重启 | 小时级（视数据量） |
| **默认用户创建** | `db.Create()` | 用户创建失败 | 静默失败 | ❌ 否 | ❌ 无检测 | 手动创建用户 | 分钟级 |
| **Sort Key 迁移** | `db.Transaction()` | 事务失败 | return error → panic | ✅ 是 | ❌ 需手动修复数据 | 清理异常数据后重启 | 分钟级 |
| **连接池耗尽** | 查询时 | 高并发/慢查询 | return error → HTTP 500 | ❌ 否 | ❌ 服务仍存活 | 优化查询/扩容 | 自动恢复秒级，优化需时间 |
| **连接临时中断** | 查询时 | 网络闪断 | return error → HTTP 500 | ❌ 否 | ❌ 服务仍存活 | 无需操作，自动恢复 | 秒级（连接池重建） |
| **Health Check 失败** | `/health` 端点 | 数据库不可用 | HTTP 500 响应 | ⚠️ K8s 会杀死后重启 | ✅ livenessProbe 触发重启 | 确认数据库状态，如正常则无需操作 | Pod 重启周期（~30秒） |
| **查询执行失败** | API 处理中 | 代码 bug/数据问题 | return error → HTTP 500 | ❌ 否 | ❌ 服务仍存活 | 偶发忽略，持续需修复代码 | 取决于修复周期 |

---

### 6.2 恢复决策树

```
检测到异常
    │
    ├─ 服务是否还在运行？
    │   ├─ 否（进程已终止）
    │   │   ├─ 查看退出时 panic 日志
    │   │   ├─ 识别失败阶段（见上表）
    │   │   ├─ 是配置/权限/磁盘问题吗？
    │   │   │   ├─ 是 → 人工修复根因后重启
    │   │   │   └─ 否 → 是数据库连接/迁移问题吗？
    │   │   │       ├─ 数据库正常 → 重启服务即可
    │   │   │       └─ 数据库异常 → 先修复数据库，再重启服务
    │   │   └─ K8s 环境：检查重启是否解决，未解决则人工介入
    │   │
    │   └─ 是（服务仍在运行）
    │       ├─ 访问 /health 检查状态
    │       │   ├─ HTTP 200 green → 正常，可能是偶发问题
    │       │   └─ HTTP 500 red/orange → 数据库问题
    │       │       ├─ 数据库本身正常吗？
    │       │       │   ├─ 是 → 这是连接池问题 → 手动重启服务加速恢复
    │       │       │   └─ 否 → 先修复数据库，再决定是否重启服务
    │       │       └─ K8s 环境：等待 livenessProbe 自动重启
    │       │
    │       └─ 检查错误率
    │           ├─ 100% 失败 → 立即重启服务
    │           ├─ 部分失败 → 观察是否自动恢复
    │           └─ 个别请求失败 → 记录日志，无需立即操作
    │
    └─ 恢复验证
        ├─ 服务重启后访问 /health 确认 green
        ├─ 验证核心功能（登录、发送消息）
        └─ 监控错误率是否恢复正常
```

---

### 6.3 关键发现与改进建议

#### 🔴 风险点 1: 默认用户创建静默失败
```go
// database/database.go:87-91
userCount := int64(0)
db.Find(new(model.User)).Count(&userCount)  // ❌ 错误未检查
if createDefaultUserIfNotExist && userCount == 0 {
    db.Create(&model.User{...})  // ❌ 错误未检查
}
```
**问题**: 这两步的错误都被忽略了，如果数据库此时异常，服务能正常启动但没有管理员用户，导致无法登录。

**建议**: 添加错误检查和日志输出。

---

#### 🔴 风险点 2: 启动阶段无重试机制
所有启动失败都直接 panic 终止，没有重试逻辑。在 K8s 环境中，数据库可能比 Gotify 启动慢，导致 Gotify 反复 CrashBackOff。

**建议**: 添加数据库连接重试逻辑，支持指数退避。

---

#### 🟡 优化点 1: 健康检查粒度不够
当前 `/health` 只做 Ping()，无法区分：
- 数据库完全不可用
- 部分连接失效
- 性能问题（慢查询）

**建议**: 扩展健康检查，增加连接池状态指标。

---

#### 🟡 优化点 2: 缺少数据库监控指标
当前没有暴露连接池使用率、等待队列长度等指标。

**建议**: 集成 Prometheus metrics 暴露数据库相关指标。

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
