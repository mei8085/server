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

文件：[app.go](app.go#L29-L60)

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

文件：[config/config.go](config/config.go#L11-L93)

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
    return conf
}
```

**configFiles() 分支逻辑**（[config/config.go#L71-L76](config/config.go#L71-L76)）：

```go
func configFiles() []string {
    if mode.Get() == mode.TestDev {
        return []string{"config.yml"}                      // ← 测试模式：仅当前目录
    }
    return []string{"config.yml", "/etc/gotify/config.yml"} // ← 生产/开发模式：两处
}
```

| 运行模式 | configFiles() 返回值 | 效果 |
|---------|---------------------|------|
| `testdev` | `["config.yml"]` | 测试只读当前目录，避免读到系统级残留配置 |
| `dev` | `["config.yml", "/etc/gotify/config.yml"]` | 开发模式也能读系统配置 |
| `prod` | `["config.yml", "/etc/gotify/config.yml"]` | 生产标准路径 |

> ⚠️ 之前文档将 `configFiles()` 简写为 `["config.yml", "/etc/gotify/config.yml"]`，未区分 TestDev 分支。实际上 TestDev 模式下**仅读** `config.yml`，不读 `/etc/gotify/config.yml`。

**配置优先级（从高到低）**：
1. 环境变量：`GOTIFY_DEFAULTUSER_NAME` / `GOTIFY_DEFAULTUSER_PASS`
2. `config.yml` 的 `defaultUser.name` / `defaultUser.pass`（当前目录）
3. `/etc/gotify/config.yml`（仅 dev/prod 模式）
4. struct tag 默认值：`admin` / `admin`

---

## 四、数据库初始化核心：database.New

文件：[database/database.go](database/database.go#L32-L109)

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

[database/database.go#L159-L167](database/database.go#L159-L167)

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

[database/database.go#L90-L92](database/database.go#L90-L92)

**GORM AutoMigrate 行为**：
- 表不存在 → CREATE TABLE
- 表已存在 → 对比字段，仅 ALTER TABLE 增列（不删列、不改类型）
- 外键约束因配置被禁用（`DisableForeignKeyConstraintWhenMigrating: true`）

**迁移后的 User 表结构**（来自 [model/user.go](model/user.go#L6-L15)）：

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

[database/database.go#L94-L98](database/database.go#L94-L98)

**幂等性保证**：双重条件 `createDefaultUserIfNotExist == true` 且 `userCount == 0`，确保：
- 数据库已有任何用户时（不管是不是 admin），**绝不会重复注入**
- 即使服务意外重启，也不会产生重复账户

### 4.7 密码加密：password.CreatePassword

文件：[auth/password/password.go](auth/password/password.go#L5-L12)

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

## 五、MySQL/Postgres 多副本首启竞态分析

### 5.1 问题场景

在 Kubernetes 等编排平台上，多个 Gotify Pod 可能**同时**对同一个 MySQL/Postgres 数据库执行首次启动。此时 `database.New` 中的 Count-then-Create 模式存在经典 TOCTOU（Time-of-Check-to-Time-of-Use）窗口：

```
  副本 A                              副本 B
──────────┬──────────────────────────────┬──────────
           │  Count() → 0                │
           │  判定: userCount==0         │
           │                              │  Count() → 0
           │                              │  判定: userCount==0
           │  Create(admin)               │
           │  ★ 成功                      │  Create(admin)
           │                              │  ★ 违反唯一索引！
```

### 5.2 代码现状：无显式事务保护

查看 [database/database.go#L94-L98](database/database.go#L94-L98)，Count 和 Create 之间**没有**包裹在数据库事务中：

```go
userCount := int64(0)
db.Find(new(model.User)).Count(&userCount)    // ← READ
if createDefaultUserIfNotExist && userCount == 0 {
    db.Create(&model.User{                    // ← WRITE
        Name: defaultUser, Pass: ..., Admin: true,
    })
}
```

对比同文件中 `fillMissingSortKeys` 和 `fillMissingCreatedAt` 使用了 `db.Transaction(..., &sql.TxOptions{Isolation: sql.LevelSerializable})`，而管理员注入**没有使用同等隔离级别**。

### 5.3 实际安全网：唯一索引兜底

竞态的**最终后果**并不严重，因为 `User.Name` 字段有数据库级唯一索引：

```go
Name string `gorm:"type:varchar(180);uniqueIndex:uix_users_name"`
```

（见 [model/user.go#L8](model/user.go#L8)）

三方言下的实际表现：

| 方言 | 竞态时第二个 Create 的结果 | 对启动的影响 |
|------|--------------------------|------------|
| **SQLite3** | 不可能竞态（`SetMaxOpenConns(1)` 强制串行） | 无影响 |
| **MySQL** | `INSERT` 触发 `uix_users_name` 唯一约束报错 | GORM 返回 error，但 `database.New` **未检查此 error**，函数正常返回 |
| **Postgres** | 同上，触发 `23505 unique_violation` | 同上，error 被静默忽略 |

关键细节：[database/database.go#L97](database/database.go#L97) 的 `db.Create(...)` 返回值**未被检查**：

```go
db.Create(&model.User{Name: defaultUser, Pass: ..., Admin: true})
// ↑ 返回值 *gorm.DB 被丢弃，Error 字段未被断言
```

这意味着：
- 副本 A 成功写入 → 正常
- 副本 B 违反唯一索引 → `db.Create` 返回 error，但**不阻断启动**
- 两个副本均正常进入 `router.Create` → 服务可用

### 5.4 竞态结论

| 维度 | 判定 |
|------|------|
| **数据正确性** | ✅ 安全。唯一索引保证至多一条 admin 记录，不会产生重复管理员 |
| **启动健壮性** | ✅ 安全。Create 失败的 error 被忽略，不影响后续流程 |
| **逻辑严谨性** | ⚠️ 有瑕疵。Count→Create 非原子操作，理论上是 TOCTOU 反模式；更严谨的做法是将整段包裹在 `SERIALIZABLE` 事务中，或使用 `INSERT ... WHERE NOT EXISTS` 的 upsert 模式 |
| **实际风险** | 极低。多副本同时首启的场景本身罕见，且唯一索引兜底后唯一副作用是日志中出现一条无害的 gorm 写入错误 |

---

## 六、`strength` 与 `createDefaultUserIfNotExist` 形参的设计权衡

### 6.1 `strength` 为什么是形参而非硬编码

[database/database.go#L33](database/database.go#L33) 中 `strength int` 作为 `New` 的形参传入，而不是在 `password.CreatePassword` 内部写死，背后有三层考量：

**（a）性能与安全的可调节点**

bcrypt cost 每增加 1，哈希耗时近似翻倍。不同部署环境对"启动时花 100ms 还是 400ms 做 admin 密码哈希"的容忍度不同：

| cost | 典型耗时 | 适用场景 |
|------|---------|---------|
| 5 | ~6ms | 自动化测试（database_test.go 中实际使用 `strength=5`） |
| 10 | ~100ms | 生产默认值 |
| 12 | ~400ms | 高安全要求环境 |

将 `strength` 提升为形参，使得 `database.New` 的调用方（[app.go#L47](app.go#L47)）从配置中读取 `PassStrength`，最终来源是 `GOTIFY_PASSSTRENGTH` 环境变量或 `passStrength` 配置项。用户无需重编译即可调节。

**（b）测试友好**

[database/database_test.go#L33](database/database_test.go#L33) 和 [database/migration_test.go#L46](database/migration_test.go#L46) 都使用 `strength=5` 调用 `New`：

```go
db, err := New("sqlite3", ..., "defaultUser", "defaultPass", 5, true, fixedNow)
```

如果 `password.CreatePassword` 内部硬编码 cost=10，每个测试用例仅初始化数据库就要多花 10 倍时间。形参化后测试可以降 cost，CI 速度不受 bcrypt 影响。

**（c）职责分离**

`database.New` 不应替业务方决定"多安全算够安全"。密码策略属于运维领域，应归入配置层；`database.New` 只负责"接收强度值、传给哈希函数"，保持数据层的单一职责。

### 6.2 `createDefaultUserIfNotExist` 为什么是形参而非内部判定

[database/database.go#L33](database/database.go#L33) 中 `createDefaultUserIfNotExist bool` 的设计意图：

**（a）使"是否注入"可被测试显式控制**

如果不设此参数，`database.New` 将永远执行空库检测并注入——测试中无法关闭此行为。形参化后，测试可以传 `false` 来创建不含默认用户的干净数据库，再验证自定义用户逻辑。

**（b）预留禁用入口**

虽然当前 [app.go#L47](app.go#L47) 始终传入 `true`，但形参的存在使得：
- 运维可通过修改一行调用关闭自动注入，强制走 OIDC 或 LDAP 的外部身份源
- 未来版本可引入 `--no-bootstrap` 命令行参数，透传到此处为 `false`

**（c）为什么不反过来——在 `New` 内部检测配置**

一种替代设计是在 `database.New` 内部读取全局配置来决定是否注入。但这样做的代价是：
- `database` 包对 `config` 包产生反向依赖，破坏分层（`config` → `database` 的单向依赖变为双向）
- `New` 函数签名隐去了注入开关，调用方无法从签名上理解行为，可读性变差

将控制权以形参形式上提至调用方，保持了 `database` 包的独立性和 `New` 函数的"纯数据层"定位。

**（d）与 `now func() time.Time` 同一设计哲学**

`now` 参数和 `createDefaultUserIfNotExist` 遵循同一原则：**将所有可能影响行为的外部因素显式化为形参**，使得 `New` 的行为完全由入参决定，无隐式的全局状态依赖。这是 Go 标准库风格（如 `httputil.ReverseProxy` 的 `Director` 参数）的惯用做法。

---

## 七、迁移测试验证：migration_test.go

文件：[database/migration_test.go](database/migration_test.go#L14-L74)

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

## 八、路由与服务启动（后续衔接）

### 8.1 router.Create

文件：[router/router.go](router/router.go#L28-L248)

数据库就绪后，路由层将 `db` 注入各 API Handler：

```go
authentication := auth.Auth{DB: db, SecureCookie: conf.Server.SecureCookie}
userHandler := api.UserAPI{DB: db, PasswordStrength: conf.PassStrength, ...}
// 登录路由：/auth/local/login → sessionHandler.Login
g.POST("/auth/local/login", sessionHandler.Login)
```

此时数据库中已存在首位管理员，用户可通过 Web UI 或 API 以 `admin/admin` 登录。

### 8.2 runner.Run

文件：[runner/runner.go](runner/runner.go#L20-L61)

启动 HTTP(S) 监听器，将 gin engine 挂载到 `http.Server`，开始对外提供服务。

---

## 九、模块衔接时序图

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

## 十、关键设计要点总结

| 设计点 | 实现方式 | 代码位置 |
|--------|---------|---------|
| **凭据来源多层级** | 环境变量 > config.yml > struct tag 默认值 | [config.go#L79-L87](config/config.go#L79-L87) |
| **幂等注入** | 双重条件：`createDefaultUserIfNotExist && userCount == 0` | [database.go#L94-L98](database/database.go#L94-L98) |
| **多副本竞态兜底** | `User.Name` 唯一索引 + Create error 被静默忽略 | [user.go#L8](model/user.go#L8)、[database.go#L97](database/database.go#L97) |
| **密码安全存储** | bcrypt 哈希，默认 cost=10，明文不落库 | [password.go#L5-L12](auth/password/password.go#L5-L12) |
| **强度可配置** | `strength` 形参化，测试用 5，生产用 10 | [database.go#L33](database/database.go#L33) |
| **注入开关可控制** | `createDefaultUserIfNotExist` 形参化，避免全局状态耦合 | [database.go#L33](database/database.go#L33) |
| **迁移与注入顺序** | AutoMigrate → 计数 → 注入（先建表再写数据） | [database.go#L90-L98](database/database.go#L90-L98) |
| **数据库方言兼容** | MySQL / PostgreSQL / SQLite3 统一入口 | [database.go#L52-L59](database/database.go#L52-L59) |
| **SQLite 并发保护** | `SetMaxOpenConns(1)` 强制串行写 | [database.go#L74-L80](database/database.go#L74-L80) |

---

## 十一、操作指引：自定义首位管理员

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
