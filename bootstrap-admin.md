# 首位管理员引导入库流程分析

本文档逐段分析 Gotify 推送服务首次启动时，命令行工具与服务进程协同完成首位管理员入库的完整代码流程。

---

## 一、整体流程总览

```
命令行启动 (app.go:main)
    │
    ├─ 1. 加载配置 (config.Get) → 获取默认用户名/密码
    │
    ├─ 2. 创建数据目录
    │
    ├─ 3. 初始化数据库 (database.New)
    │       ├─ 3.1 SQLite 目录创建
    │       ├─ 3.2 建立数据库连接
    │       ├─ 3.3 设置连接池参数
    │       ├─ 3.4 AutoMigrate 表结构迁移
    │       ├─ 3.5 检查用户表是否为空
    │       ├─ 3.6 空库时注入首位管理员（bcrypt 加密）
    │       └─ 3.7 补充缺失字段（sort_key / created_at）
    │
    ├─ 4. 创建路由引擎 (router.Create)
    │
    └─ 5. 启动 HTTP 服务 (runner.Run)
```

---

## 二、启动入口：app.go:main

文件：[app.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-server/app.go#L29-L60)

### 核心调用链

```go
func main() {
    // 初始化日志
    log.Logger = log.Output(zerolog.ConsoleWriter{...})

    // 设置运行模式
    mode.Set(Mode)

    // ★ 第1步：加载配置（含默认管理员凭据）
    conf := config.Get()

    // 第2步：创建必要目录
    os.MkdirAll(conf.PluginsDir, 0o755)
    os.MkdirAll(conf.UploadedImagesDir, 0o755)

    // ★ 第3步：初始化数据库，传入默认用户名/密码
    // 参数 createDefaultUserIfNotExist = true
    db, err := database.New(
        conf.Database.Dialect,
        conf.Database.Connection,
        conf.DefaultUser.Name,   // 默认 "admin"
        conf.DefaultUser.Pass,   // 默认 "admin"
        conf.PassStrength,       // 默认 bcrypt cost=10
        true,                    // createDefaultUserIfNotExist
        time.Now,
    )

    // 第4步：创建路由
    engine, closeable := router.Create(db, vInfo, conf)

    // 第5步：启动服务进程
    runner.Run(engine, conf)
}
```

**关键衔接点**：`main` 函数将配置模块解析出的 `DefaultUser.Name` 和 `DefaultUser.Pass` 直接透传给 `database.New`，由数据库模块决定是否执行注入。

---

## 三、配置模块：config.Get

文件：[config/config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-server/config/config.go#L11-L93)

### Configuration 结构体关键片段

```go
type Configuration struct {
    Database struct {
        Dialect    string `default:"sqlite3"`
        Connection string `default:"data/gotify.db"`
    }
    // ★ 默认管理员凭据
    DefaultUser struct {
        Name string `default:"admin"`   // 标签默认值
        Pass string `default:"admin"`   // 标签默认值
    }
    PassStrength int `default:"10"`     // bcrypt 哈希强度
    // ...
}
```

### 加载逻辑

```go
func Get() *Configuration {
    conf := new(Configuration)
    // configor 库按优先级加载：环境变量 > config.yml > struct tag 默认值
    // 环境变量前缀 GOTIFY_，如 GOTIFY_DEFAULTUSER_NAME
    err := configor.New(&configor.Config{
        ENVPrefix: "GOTIFY",
        Silent:    true,
    }).Load(conf, configFiles()...)
    // configFiles() = ["config.yml", "/etc/gotify/config.yml"]
    return conf
}
```

**配置优先级（从高到低）**：
1. 环境变量：`GOTIFY_DEFAULTUSER_NAME` / `GOTIFY_DEFAULTUSER_PASS`
2. 当前目录 `config.yml` 的 `defaultUser.name` / `defaultUser.pass`
3. `/etc/gotify/config.yml`
4. struct tag 默认值：`admin` / `admin`

---

## 四、数据库初始化核心：database.New

文件：[database/database.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-server/database/database.go#L32-L109)

这是首位管理员入库的**核心函数**，逐段拆解如下。

### 4.1 函数签名

```go
func New(
    dialect, connection,                 // 数据库类型与连接串
    defaultUser, defaultPass string,     // 从配置传入的默认凭据
    strength int,                         // bcrypt 强度
    createDefaultUserIfNotExist bool,     // 是否允许注入（首启时为 true）
    now func() time.Time,                 // 时间函数（便于测试）
) (*GormDatabase, error)
```

### 4.2 阶段一：SQLite 目录创建

```go
createDirectoryIfSqlite(dialect, connection)
```

[database/database.go#L159-L167](file:///d:/fz/0601-2/solo-dogfeeding/code/5-server/database/database.go#L159-L167)

仅当使用 `sqlite3` 时，确保 `data/gotify.db` 所在的 `data/` 目录存在，否则 `gorm.Open` 会失败。

### 4.3 阶段二：建立数据库连接

```go
switch dialect {
case "mysql":
    db, err = gorm.Open(mysql.Open(connection), gormConfig)
case "postgres":
    db, err = gorm.Open(postgres.Open(connection), gormConfig)
case "sqlite3":
    db, err = gorm.Open(sqlite.Open(connection), gormConfig)
}
```

支持三种方言，通过 DSN 建立连接。GORM 配置中禁用了外键约束自动创建，便于迁移。

### 4.4 阶段三：连接池调优

```go
sqldb, _ := db.DB()
sqldb.SetMaxOpenConns(10)            // 全局限制 10 连接
if dialect == "sqlite3" {
    sqldb.SetMaxOpenConns(1)         // SQLite 只支持单写，强制串行
}
if dialect == "mysql" {
    sqldb.SetConnMaxLifetime(9 * time.Minute)  // 规避 wait_timeout
}
```

### 4.5 阶段四：★ 数据库表结构迁移 (AutoMigrate)

```go
// ★ 核心迁移调用
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

[database/database.go#L90-L92](file:///d:/fz/0601-2/solo-dogfeeding/code/5-server/database/database.go#L90-L92)

**GORM AutoMigrate 行为**：
- 表不存在 → CREATE TABLE
- 表已存在 → 对比字段，仅 ALTER TABLE 增列（不删列、不改类型）
- 外键约束因配置被禁用（`DisableForeignKeyConstraintWhenMigrating: true`）

**迁移后的 User 表结构**（来自 [model/user.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-server/model/user.go#L6-L15)）：

```go
type User struct {
    ID           uint   `gorm:"primaryKey;autoIncrement"`
    Name         string `gorm:"type:varchar(180);uniqueIndex:uix_users_name"`  // 唯一索引
    Pass         []byte   // bcrypt 哈希后的密文
    Admin        bool     // 是否管理员
    CreatedAt    time.Time
    Applications []Application
    Clients      []Client
    Plugins      []PluginConf
}
```

### 4.6 阶段五：★ 空库检测与首位管理员注入

这是**最关键的业务逻辑**：

```go
// 1. 统计用户总数
userCount := int64(0)
db.Find(new(model.User)).Count(&userCount)

// 2. 条件判断：允许注入 AND 用户数为 0
if createDefaultUserIfNotExist && userCount == 0 {
    // 3. 密码加密后写入
    db.Create(&model.User{
        Name:  defaultUser,                                    // 明文用户名
        Pass:  password.CreatePassword(defaultPass, strength), // bcrypt 密文
        Admin: true,                                           // ★ 标记为管理员
    })
}
```

[database/database.go#L94-L98](file:///d:/fz/0601-2/solo-dogfeeding/code/5-server/database/database.go#L94-L98)

**幂等性保证**：双重条件 `createDefaultUserIfNotExist == true` 且 `userCount == 0`，确保：
- 数据库已有任何用户时（不管是不是 admin），**绝不会重复注入**
- 即使服务意外重启，也不会产生重复账户

### 4.7 密码加密：password.CreatePassword

文件：[auth/password/password.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-server/auth/password/password.go#L5-L12)

```go
func CreatePassword(pw string, strength int) []byte {
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(pw), strength)
    if err != nil {
        panic(err)  // 加密失败直接终止启动
    }
    return hashedPassword
}
```

- 使用 Go 标准 `golang.org/x/crypto/bcrypt`
- `strength` 即 bcrypt cost，配置默认值为 10（约 100ms 哈希耗时）
- 返回值是 `[]byte`，直接存入 `model.User.Pass` 字段

### 4.8 阶段六：数据补全迁移（兼容旧版本）

```go
// 补全 Application 缺失的 sort_key（排序用）
db.Transaction(fillMissingSortKeys, &sql.TxOptions{Isolation: sql.LevelSerializable})

// 补全各表缺失的 created_at
db.Transaction(func(tx *gorm.DB) error { return fillMissingCreatedAt(tx, now()) }, ...)
```

这两步与管理员注入无直接关系，但属于初始化流程的一部分，确保旧版本升级后数据完整性。

---

## 五、迁移测试验证：migration_test.go

文件：[database/migration_test.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-server/database/migration_test.go#L14-L74)

测试用例 `TestMigration` 完整验证了引导逻辑：

```go
// BeforeTest：手工创建一个旧库，只含 User 表，并插入 test_user
db.Migrator().CreateTable(new(model.User))
db.Create(&model.User{Name: "test_user", Admin: true})

// TestMigration：调用 database.New，传入 createDefaultUserIfNotExist=true
db, err := New("sqlite3", ..., "admin", "admin", 6, true, fixedNow)

// 断言1：AutoMigrate 已创建 Application 表
assert.True(s.T(), db.DB.Migrator().HasTable(new(model.Application)))

// 断言2：因 test_user 已存在，userCount!=0，故 NOT 注入 admin
if user, err := db.GetUserByName("admin"); assert.NoError(s.T(), err) {
    assert.Nil(s.T(), user)  // admin 不存在
}

// 断言3：原有用户 test_user 保留，且 Admin=true
if user, err := db.GetUserByName("test_user"); assert.NoError(s.T(), err) {
    assert.Equal(s.T(), true, user.Admin)
}
```

这个测试证明：**当数据库已有任何用户时，默认管理员不会被注入**，幂等性得到保障。

---

## 六、路由与服务启动（后续衔接）

### 6.1 router.Create

文件：[router/router.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-server/router/router.go#L28-L248)

数据库就绪后，路由层将 `db` 注入各 API Handler：

```go
authentication := auth.Auth{DB: db, SecureCookie: conf.Server.SecureCookie}
userHandler := api.UserAPI{DB: db, PasswordStrength: conf.PassStrength, ...}
// 登录路由：/auth/local/login → sessionHandler.Login
g.POST("/auth/local/login", sessionHandler.Login)
```

此时数据库中已存在首位管理员，用户可通过 Web UI 或 API 以 `admin/admin` 登录。

### 6.2 runner.Run

文件：[runner/runner.go](file:///d:/fz/0601-2/solo-dogfeeding/code/5-server/runner/runner.go#L20-L61)

启动 HTTP(S) 监听器，将 gin engine 挂载到 `http.Server`，开始对外提供服务。

---

## 七、模块衔接时序图

```
  用户命令行                         app.go                       config                        database
──────┬──────────────────────────────┬────────────────────────────┬───────────────────────────────┬──────
      │  ./gotify server             │                            │                               │
      │─────────────────────────────>│                            │                               │
      │                              │  config.Get()              │                               │
      │                              │───────────────────────────>│                               │
      │                              │                            │ 加载 GOTIFY_* 环境变量        │
      │                              │                            │ 加载 config.yml               │
      │                              │                            │ struct tag 默认值             │
      │                              │<───────────────────────────│                               │
      │                              │  conf.DefaultUser.Name     │                               │
      │                              │  conf.DefaultUser.Pass     │                               │
      │                              │  conf.PassStrength         │                               │
      │                              │                            │                               │
      │                              │  database.New(...,         │                               │
      │                              │      defaultUser,          │                               │
      │                              │      defaultPass,          │                               │
      │                              │      strength,             │                               │
      │                              │      createDefault=true)   │                               │
      │                              │───────────────────────────────────────────────────────────>│
      │                              │                            │                               │ 1. 建立连接
      │                              │                            │                               │ 2. SetMaxOpenConns
      │                              │                            │                               │ 3. AutoMigrate(User, ...)
      │                              │                            │                               │ 4. Count(model.User)
      │                              │                            │                               │    → userCount = 0
      │                              │                            │                               │ 5. CreatePassword(pass, 10)
      │                              │                            │                               │ 6. db.Create(User{Admin:true})
      │                              │                            │                               │ 7. fillMissingSortKeys
      │                              │                            │                               │ 8. fillMissingCreatedAt
      │                              │<───────────────────────────────────────────────────────────│
      │                              │                            │                               │
      │                              │  router.Create(db)         │                               │
      │                              │  runner.Run(engine)        │                               │
      │                              │  :80 监听启动              │                               │
      │                              │                            │                               │
      │  POST /auth/local/login      │                            │                               │
      │  {name:"admin",pass:"admin"} │                            │                               │
      │─────────────────────────────>│───────────────────────────────────────────────────────────>│
      │                              │                            │                               │ SELECT * FROM users WHERE name='admin'
      │                              │                            │                               │ bcrypt.CompareHashAndPassword
      │<─────────────────────────────│<───────────────────────────────────────────────────────────│
      │  200 OK + Set-Cookie         │                            │                               │
```

---

## 八、关键设计要点总结

| 设计点 | 实现方式 | 代码位置 |
|--------|---------|---------|
| **凭据来源多层级** | 环境变量 > config.yml > struct tag 默认值 | [config.go#L79-L87](file:///d:/fz/0601-2/solo-dogfeeding/code/5-server/config/config.go#L79-L87) |
| **幂等注入** | 双重条件：`createDefaultUserIfNotExist && userCount == 0` | [database.go#L94-L98](file:///d:/fz/0601-2/solo-dogfeeding/code/5-server/database/database.go#L94-L98) |
| **密码安全存储** | bcrypt 哈希，默认 cost=10，明文不落库 | [password.go#L5-L12](file:///d:/fz/0601-2/solo-dogfeeding/code/5-server/auth/password/password.go#L5-L12) |
| **迁移与注入顺序** | AutoMigrate → 计数 → 注入（先建表再写数据） | [database.go#L90-L98](file:///d:/fz/0601-2/solo-dogfeeding/code/5-server/database/database.go#L90-L98) |
| **数据库方言兼容** | MySQL / PostgreSQL / SQLite3 统一入口 | [database.go#L52-L59](file:///d:/fz/0601-2/solo-dogfeeding/code/5-server/database/database.go#L52-L59) |
| **SQLite 并发保护** | `SetMaxOpenConns(1)` 强制串行写 | [database.go#L74-L80](file:///d:/fz/0601-2/solo-dogfeeding/code/5-server/database/database.go#L74-L80) |
| **唯一性约束** | `Name` 字段 `uniqueIndex`，即使并发也不重复 | [user.go#L8](file:///d:/fz/0601-2/solo-dogfeeding/code/5-server/model/user.go#L8) |

---

## 九、操作指引：自定义首位管理员

无需修改代码，通过以下任一方式覆盖默认凭据：

### 方式 A：环境变量（推荐容器化部署）
```bash
export GOTIFY_DEFAULTUSER_NAME="myadmin"
export GOTIFY_DEFAULTUSER_PASS="StrongP@ss123"
./gotify server
```

### 方式 B：config.yml（推荐裸机部署）
在 `config.yml` 中配置：
```yaml
defaultUser:
  name: myadmin
  pass: StrongP@ss123
passStrength: 12  # 可选，提高 bcrypt 强度
```

### 方式 C：依赖默认值（仅用于开发/测试）
不做任何配置，首次启动后默认：
- 用户名：`admin`
- 密码：`admin`

> ⚠️ **生产环境必须修改默认凭据**，否则任何人可通过 80 端口登录管理员后台。
