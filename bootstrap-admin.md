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

## 三、Mode→ldflags→凭据来源链路（按代码执行顺序逐段拆解）

凭据从哪份配置文件读取，取决于 `configFiles()` 返回什么；而 `configFiles()` 的分支逻辑，取决于 `mode` 包中的全局状态 `mode.mode`。以下严格按**代码执行顺序**逐段拆开这条链路。

---

### 第①步：构建阶段 —— ldflags 在链接时注入 Mode 值

**代码位置**：[Makefile#L27](Makefile#L27)、[Makefile#L120](Makefile#L120)

在运行 `make` 或 Docker build 时，Makefile 调用 Go 编译器并传入 `-ldflags` 参数：

```makefile
# Makefile 第 27 行（test-js 目标）
go build -ldflags="-s -w -X main.Mode=prod" -o removeme/gotify app.go

# Makefile 第 120 行（Docker master 构建）
--build-arg LD_FLAGS="... -X main.Mode=prod"
```

`-X main.Mode=prod` 的含义是：**在链接阶段（linker，即 ld），将 `main` 包中变量 `Mode` 的值覆写为字符串 `"prod"`。**

- 这个动作发生在**编译时**，而非运行时
- 未传 `-ldflags` 时，`Mode` 保持 Go 源码中的初始化值
- 这是 Go 生态在二进制中嵌入版本号/构建模式的惯用手法

---

### 第②步：Go 源码中 Mode 变量的默认初始化

**代码位置**：[app.go#L18-L27](app.go#L18-L27)

```go
var (
    Version   = "unknown"
    Commit    = "unknown"
    BuildDate = "unknown"
    Mode      = mode.Dev   // ← 编译时默认值 = "dev"（见下方 mode 常量）
)
```

`mode.Dev` 的定义在 [mode/mode.go#L5-L12](mode/mode.go#L5-L12)：

```go
const (
    Dev     = "dev"       // 未构建 → 走此分支
    Prod    = "prod"      // ldflags 注入后 → Mode 变为此值
    TestDev = "testdev"   // 测试代码手动 mode.Set() 切换
)
```

**第①步和第②步的衔接**：
- `go build` **不传** `-ldflags` → `Mode` = `"dev"`（Go 源码初始化值）
- `go build` **传了** `-X main.Mode=prod` → 链接器在生成的二进制中将 `Mode` 改为 `"prod"`

---

### 第③步：进程启动执行 main() —— mode.Set(Mode)

**代码位置**：[app.go#L29-L36](app.go#L29-L36)

```go
func main() {
    // 前两行：日志初始化 + 组装版本信息
    log.Logger = log.Output(zerolog.ConsoleWriter{...})
    vInfo := &model.VersionInfo{Version: Version, Commit: Commit, BuildDate: BuildDate}

    // ★ 关键衔接点：第 33 行，将 Mode 值写入 mode 包
    mode.Set(Mode)

    // ★ 下一行就是读取配置 —— 此时 mode.mode 已被 Set 覆写
    log.Info().Str("version", vInfo.Version)...Msg("Gotify")
    conf := config.Get()
    // ...
}
```

---

### 第④步：mode.Set 写入包级状态

**代码位置**：[mode/mode.go#L14-L20](mode/mode.go#L14-L20)

`mode` 包有一个包级私有变量 `var mode = Dev`，所有 `Set/Get` 都围绕它操作：

```go
var mode = Dev   // ← 包级状态，mode.Dev = "dev"

func Set(newMode string) {
    mode = newMode      // ← 覆盖为 Mode 传入的值（"dev"/"prod"/"testdev"）
    updateGinMode()     // ← 顺带同步 gin 的运行模式（Debug/Release/Test）
}

func Get() string {
    return mode   // ← 读取当前值，后续 configFiles() 用的就是它
}
```

**第③步和第④步的衔接**：
- 如果 ldflags 注入了 `Mode="prod"` → `mode.Set("prod")` → 包级变量 `mode` 变成 `"prod"`
- 如果没注入 → `Mode="dev"` → `mode.Set("dev")` → 包级变量 `mode` 保持 `"dev"`（和默认值一样）
- 如果是 `go test` → 测试代码会手动 `mode.Set(mode.TestDev)` → 包级变量 `mode` 变成 `"testdev"`

---

### 第⑤步：config.Get() 调用 configFiles()

**代码位置**：[config/config.go#L78-L86](config/config.go#L78-L86)

```go
func Get() *Configuration {
    conf := new(Configuration)
    // ★ 关键：configFiles() 是在 Load 时展开的，而 configFiles() 又会读 mode.mode
    err := configor.New(&configor.Config{ENVPrefix: "GOTIFY", Silent: true}).
        Load(conf, configFiles()...)   // ← configFiles() 先执行，得到文件列表
    if err != nil {
        panic(err)
    }
    addTrailingSlashToPaths(conf)
    return conf
}
```

Go 的求值顺序：`configFiles()` 在 `Load(...)` 之前被调用，返回 `[]string`，再作为变参传入 `Load`。

---

### 第⑥步：configFiles() 读取 mode.mode 并分支

**代码位置**：[config/config.go#L71-L76](config/config.go#L71-L76)

```go
func configFiles() []string {
    // ★ 这里调用 mode.Get()，而 mode.Get() 返回的正是第④步被设置过的 mode.mode
    if mode.Get() == mode.TestDev {
        // 分支 A：测试模式 —— 只看当前目录，避免读系统级残留
        return []string{"config.yml"}
    }
    // 分支 B：dev / prod 模式 —— 当前目录 + 系统标准路径
    return []string{"config.yml", "/etc/gotify/config.yml"}
}
```

这就是 `Mode` 值如何决定"凭据从哪份文件读"的核心衔接：

| 第④步 `mode.mode` 值 | 进入哪个分支 | configFiles() 返回 | 读取哪些文件 |
|---|---|---|---|
| `"testdev"` | 分支 A（if 命中） | `["config.yml"]` | **仅**当前目录 config.yml |
| `"dev"` 或 `"prod"` | 分支 B（if 未命中） | `["config.yml", "/etc/gotify/config.yml"]` | 当前目录 + `/etc` |

---

### 第⑦步：configor.Load 解析配置 → conf.DefaultUser 获得最终值

**代码位置**：[config/config.go#L51-L55](config/config.go#L51-L55) + [config/config.go#L78-L86](config/config.go#L78-L86)

`configor` 库按如下**从高到低**的优先级填充 `Configuration` 结构体：

```
优先级 1：环境变量
          GOTIFY_DEFAULTUSER_NAME  覆盖 DefaultUser.Name
          GOTIFY_DEFAULTUSER_PASS  覆盖 DefaultUser.Pass
          GOTIFY_PASSSTRENGTH      覆盖 PassStrength

优先级 2：第 1 个配置文件
          config.yml 中的 defaultUser.name / defaultUser.pass

优先级 3：第 2 个配置文件（仅分支 B 存在）
          /etc/gotify/config.yml 中的同名字段（只覆盖优先级 2 中没出现的字段）

优先级 4：struct tag 默认值
          Name string `default:"admin"`
          Pass string `default:"admin"`
          PassStrength int `default:"10"`
```

最终得到的 `conf.DefaultUser.Name` / `conf.DefaultUser.Pass`，就是接下来 `database.New()` 用来注入首位管理员的凭据。

---

### Mode→凭据链路完整传导图

```
构建时链接阶段（Makefile 调用 go build）
  │
  │  -ldflags "-X main.Mode=prod"
  ▼
app.go: var Mode = mode.Dev   ← 被链接器覆写为 "prod"
  │
  ▼  main() 执行
app.go: mode.Set(Mode)        ← 传入 "prod"
  │
  ▼
mode.mode = "prod"            ← 包级状态被更新
  │
  ▼  config.Get() 执行
configFiles() → mode.Get() == "prod" != "testdev"
  │
  ▼
configFiles() = ["config.yml", "/etc/gotify/config.yml"]
  │
  ▼  configor.Load
conf.DefaultUser = {Name: "admin", Pass: "admin"}   ← 综合 4 层优先级后的结果
  │
  ▼
database.New(..., Name, Pass, ...)                   ← 透传给数据库模块做注入
```

**易错点澄清**（之前文档表述不够精确之处）：
1. 「`configFiles()` 读哪份文件」不是在编译时决定，而是在运行时 `config.Get()` 内部才决定，取决于当前 `mode.mode`
2. TestDev 模式不会由 ldflags 产生，而是测试代码 `mode.Set(mode.TestDev)` 显式切换
3. `Mode` 变量本身在构建后是只读的字符串常量；它的值如何，完全取决于构建时是否传入了 `-ldflags`

---

## 四、配置模块：config.Get

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

### 第⑦步之后：路径归一化 addTrailingSlashToPaths

`configor.Load` 解析完成后，还有一步收尾处理：

**代码位置**：[config/config.go#L89-L93](config/config.go#L89-L93)

```go
func addTrailingSlashToPaths(conf *Configuration) {
    if !strings.HasSuffix(conf.UploadedImagesDir, "/") && !strings.HasSuffix(conf.UploadedImagesDir, "\\") {
        conf.UploadedImagesDir += string(filepath.Separator)
    }
}
```

- 只处理 `UploadedImagesDir` 一个字段（`PluginsDir` 不处理）
- 作用：确保路径末尾总有一个分隔符，后面拼接文件名时不需要再加 `/`
- 与首位管理员注入无直接关系，但属于 `config.Get()` 完整流程的一环

---

## 五、数据库初始化核心：database.New

文件：[database/database.go](database/database.go#L32-L109)

这是首位管理员入库的**核心函数**，逐段拆解如下。

### 5.1 函数签名

```go
func New(
    dialect, connection,                 // 数据库类型与连接串
    defaultUser, defaultPass string,     // 从配置传入的默认凭据
    strength int,                         // bcrypt 强度
    createDefaultUserIfNotExist bool,     // 是否允许注入（首启时为 true）
    now func() time.Time,                 // 时间函数（便于测试）
) (*GormDatabase, error)
```

### 5.2 阶段一：SQLite 目录创建

```go
createDirectoryIfSqlite(dialect, connection)
```

[database/database.go#L159-L167](database/database.go#L159-L167)

仅当使用 `sqlite3` 时，确保 `data/gotify.db` 所在的 `data/` 目录存在，否则 `gorm.Open` 会失败。

### 5.3 阶段二：建立数据库连接

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

### 5.4 阶段三：连接池调优

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

### 5.5 阶段四：★ 数据库表结构迁移 (AutoMigrate)

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

### 5.6 阶段五：★ 空库检测与首位管理员注入

这是**最关键的业务逻辑**：

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

[database/database.go#L94-L98](database/database.go#L94-L98)

**幂等性保证**：双重条件 `createDefaultUserIfNotExist == true` 且 `userCount == 0`，确保：
- 数据库已有任何用户时（不管是不是 admin），**绝不会重复注入**
- 即使服务意外重启，也不会产生重复账户

### 5.7 密码加密：password.CreatePassword

文件：[auth/password/password.go](auth/password/password.go#L5-L12)

```go
func CreatePassword(pw string, strength int) []byte {
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(pw), strength)
    if err != nil {
        panic(err)
    }
    return hashedPassword
}
```

- 使用 Go 标准 `golang.org/x/crypto/bcrypt`
- `strength` 即 bcrypt cost，配置默认值为 10（约 100ms 哈希耗时）
- 返回值是 `[]byte`，直接存入 `model.User.Pass` 字段

### 5.8 阶段六：数据补全迁移（兼容旧版本）

```go
db.Transaction(fillMissingSortKeys, &sql.TxOptions{Isolation: sql.LevelSerializable})
db.Transaction(func(tx *gorm.DB) error { return fillMissingCreatedAt(tx, now()) }, ...)
```

这两步与管理员注入无直接关系，但属于初始化流程的一部分，确保旧版本升级后数据完整性。

---

## 六、MySQL/Postgres 多副本首启竞态分析

### 6.1 问题场景

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

### 6.2 代码现状：无显式事务保护

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

### 6.3 实际安全网：唯一索引兜底

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

### 6.4 竞态结论

| 维度 | 判定 |
|------|------|
| **数据正确性** | ✅ 安全。唯一索引保证至多一条 admin 记录，不会产生重复管理员 |
| **启动健壮性** | ✅ 安全。Create 失败的 error 被忽略，不影响后续流程 |
| **逻辑严谨性** | ⚠️ 有瑕疵。Count→Create 非原子操作，理论上是 TOCTOU 反模式；更严谨的做法是将整段包裹在 `SERIALIZABLE` 事务中，或使用 `INSERT ... WHERE NOT EXISTS` 的 upsert 模式 |
| **实际风险** | 极低。多副本同时首启的场景本身罕见，且唯一索引兜底后唯一副作用是日志中出现一条无害的 gorm 写入错误 |

---

## 七、`strength` 与 `createDefaultUserIfNotExist` 形参的设计权衡

### 7.1 `strength` 为什么是形参而非硬编码

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

### 7.2 `createDefaultUserIfNotExist` 为什么是形参而非内部判定

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

## 八、迁移测试验证：migration_test.go

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
    assert.Nil(s.T(), user)
}

// 断言3：原有用户 test_user 保留，且 Admin=true
if user, err := db.GetUserByName("test_user"); assert.NoError(s.T(), err) {
    assert.Equal(s.T(), true, user.Admin)
}
```

这个测试证明：**当数据库已有任何用户时，默认管理员不会被注入**，幂等性得到保障。

---

## 九、路由与服务启动（后续衔接）

### 9.1 router.Create

文件：[router/router.go](router/router.go#L28-L248)

数据库就绪后，路由层将 `db` 注入各 API Handler：

```go
authentication := auth.Auth{DB: db, SecureCookie: conf.Server.SecureCookie}
userHandler := api.UserAPI{DB: db, PasswordStrength: conf.PassStrength, ...}
g.POST("/auth/local/login", sessionHandler.Login)
```

此时数据库中已存在首位管理员，用户可通过 Web UI 或 API 以 `admin/admin` 登录。

### 9.2 runner.Run

文件：[runner/runner.go](runner/runner.go#L20-L61)

启动 HTTP(S) 监听器，将 gin engine 挂载到 `http.Server`，开始对外提供服务。

---

## 十、登录闭环链路（按代码执行顺序逐段拆解）

首启时 `database.New` 把 `admin/admin` 以 bcrypt 密文写进 `users` 表；服务启动后用户通过 `POST /auth/local/login` 登录时，必须经过完全对称的校验链路才能拿到会话 cookie；后续请求再携带 cookie 反向还原身份。以下严格按**代码执行顺序**逐段拆开这条端到端闭环。

---

### 前置 0：登录路由与 SecureCookie 的装配

在进入登录代码之前，先理清 `sessionHandler` 和它的 `SecureCookie` 字段是从哪装配的——这关系到最终签发的 cookie 是否携带 `Secure` 标志。

**代码位置**：[router/router.go#L87-L99](router/router.go#L87-L99) + [router/router.go#L157](router/router.go#L157)

```go
// router.Create 函数内部
authentication := auth.Auth{DB: db, SecureCookie: conf.Server.SecureCookie}   // 中间件用
sessionHandler := api.SessionAPI{
    DB:            db,
    NotifyDeleted: streamHandler.NotifyDeletedClient,
    SecureCookie:  conf.Server.SecureCookie,   // ★ 从 config 透传，默认 false
}
// ...
g.POST("/auth/local/login", sessionHandler.Login)   // 登录端点不挂任何认证中间件（匿名可访问）
```

`conf.Server.SecureCookie` 来自 [config/config.go#L45](config/config.go#L45)：
```go
SecureCookie bool `default:"false"`
```

- 生产环境启用 HTTPS 后，应通过 `GOTIFY_SERVER_SECURECOOKIE=true` 打开
- 此值在第⑧步签发 cookie 时被作为 `secure` 参数传入
- 第⑪步后续请求验证时，此值同样被 `authentication` 用在刷新 cookie 时

---

### 第①步：浏览器请求到达 → SessionAPI.Login

用户在登录框输入 `admin/admin`，Web UI 发起：
```http
POST /auth/local/login HTTP/1.1
Authorization: Basic YWRtaW46YWRtaW4=    ← "admin:admin" 的 base64
Content-Type: application/x-www-form-urlencoded

name=Browser
```

命中路由注册 `g.POST("/auth/local/login", sessionHandler.Login)`，进入：

**代码位置**：[api/session.go#L56-L100](api/session.go#L56-L100)

```go
func (a *SessionAPI) Login(ctx *gin.Context) {
    // ── 第②步在此函数开头 ──
    name, pass, ok := ctx.Request.BasicAuth()
    // ...
}
```

---

### 第②步：BasicAuth 解析，取出用户名/密码明文

**代码位置**：[api/session.go#L57-L61](api/session.go#L57-L61)

```go
name, pass, ok := ctx.Request.BasicAuth()
if !ok {
    ctx.AbortWithError(401, errors.New("basic auth required"))
    return
}
```

- `BasicAuth()` 是 Go 标准库 `net/http` 的方法，从 `Authorization: Basic ...` 头中解码 base64，返回 `(username, password, ok)`
- 解码后得到 `name = "admin"`、`pass = "admin"`，都是**明文**
- 如果请求不带 `Authorization` 头，直接 401 返回

**和入库侧的衔接**：此时的 `name` 明文，与第 10.5 节 `database.New` 中 `db.Create(User{Name: defaultUser, ...})` 的 `defaultUser`，就是即将通过同一个唯一索引对接的同一个字符串。

---

### 第③步：按用户名查询 → 命中 users 表唯一索引

**代码位置**：[api/session.go#L63-L67](api/session.go#L63-L67)

```go
user, err := a.DB.GetUserByName(name)   // name = "admin"
if err != nil {
    ctx.AbortWithError(500, err)
    return
}
```

跳到 `GetUserByName` 实现：[database/user.go#L9-L19](database/user.go#L9-L19)

```go
func (d *GormDatabase) GetUserByName(name string) (*model.User, error) {
    user := new(model.User)
    // 生成 SQL：SELECT * FROM users WHERE name = ?
    err := d.DB.Where("name = ?", name).Find(user).Error
    if err != nil {
        return nil, err
    }
    if user.Name == name {   // 查到了，且 Name 精确匹配（防御脏数据）
        return user, nil
    }
    return nil, nil          // 没查到，返回 (nil, nil)
}
```

**为什么能精确对应入库时写入的那一行？**

看 model 定义 [model/user.go#L8](model/user.go#L8)：

```go
Name string `gorm:"type:varchar(180);uniqueIndex:uix_users_name"`
//                                 ↑↑↑ 唯一索引 uix_users_name
```

和入库时写入时的同一字段、同一索引对接：

| 环节 | 代码 | 产生的 SQL | 用到的索引 |
|---|---|---|---|
| **首启入库** | `db.Create(User{Name:"admin", ...})` | `INSERT INTO users(name, ...) VALUES ('admin', ...)` | `uix_users_name`（唯一性约束检查） |
| **登录查询** | `Where("name = ?", "admin").Find(user)` | `SELECT * FROM users WHERE name = 'admin'` | `uix_users_name`（B 树等值查找） |

**结论**：两者共用同一个唯一索引，这是"写进去的那行一定能被查出来"的数据库级保障。

---

### 第④步：bcrypt 校验 → 与入库时 CreatePassword 对称

**代码位置**：[api/session.go#L68-L71](api/session.go#L68-L71)

```go
if user == nil || !password.ComparePassword(user.Pass, []byte(pass)) {
    // ↑ 两种情况二选一就失败：① 用户不存在  ② 密码比对失败
    ctx.AbortWithError(401, errors.New("invalid credentials"))
    return
}
```

**`ComparePassword` 实现**：[auth/password/password.go#L14-L16](auth/password/password.go#L14-L16)

```go
func ComparePassword(hashedPassword, password []byte) bool {
    return bcrypt.CompareHashAndPassword(hashedPassword, password) == nil
}
```

**与入库侧 `CreatePassword` 的严格对称**：

| | 入库 `CreatePassword` | 登录 `ComparePassword` |
|---|---|---|
| **代码位置** | [password.go#L5-L12](auth/password/password.go#L5-L12) | [password.go#L14-L16](auth/password/password.go#L14-L16) |
| **核心调用** | `bcrypt.GenerateFromPassword([]byte("admin"), 10)` | `bcrypt.CompareHashAndPassword(dbHash, []byte("admin"))` |
| **cost 从哪来** | 第 3 参数 `strength`，来自 `conf.PassStrength`，默认 10 | **不需要传** —— bcrypt 哈希格式 `$2a$10$salt$hash` 中前缀 `$10$` 已编码 cost，Compare 自动提取 |
| **输入明文** | `"admin"`（从配置 defaultPass 来） | `"admin"`（从 Authorization 头解码来） |
| **输出** | `[]byte`（形如 `$2a$10$N9qo...` 的 60 字节） | `bool`（匹配 = true，不匹配 = false） |

**一个容易忽略的设计**：
- 入库时 cost=10，哈希前缀是 `$2a$10$`
- 如果之后把 `GOTIFY_PASSSTRENGTH` 改成 12，重启服务后登录，**旧哈希仍然可以正确比对**
- 因为 `CompareHashAndPassword` 完全不需要知道 cost 参数，全部从哈希前缀中读取
- 这就是为什么 `PassStrength` 只影响**新哈希的生成**，不影响旧哈希的验证

---

### 第⑤步：解析可选的 name 表单字段（会话显示名）

**代码位置**：[api/session.go#L73-L76](api/session.go#L73-L76)

```go
clientParams := ClientParams{}
if err := ctx.Bind(&clientParams); err != nil {
    return
}
```

`ClientParams` 只有一个字段 `Name`，来自请求体 `application/x-www-form-urlencoded` 的 `name=Browser`。这是用来给新创建的 Client 记录起一个人类可读的名字（如"Chrome on Windows"），**不影响身份认证**，失败时也不会中断登录（`return` 只跳绑定，不写响应，流程继续）。

---

### 第⑥步：构造 Client 会话载体 → 生成唯一 Token

**代码位置**：[api/session.go#L78-L85](api/session.go#L78-L85)

```go
elevatedUntil := time.Now().Add(model.DefaultElevationDuration)  // 默认 1 小时
client := model.Client{
    Name:                          clientParams.Name,                 // 显示名
    Token:                         auth.GenerateNotExistingToken(     // ★ 核心：生成会话 token
        generateClientToken, a.clientExists),
    UserID:                        user.ID,                           // ★ 关联到 admin 的 ID
    ElevatedUntil:                 &elevatedUntil,                    // 1 小时内可做管理员操作
    ExpiresAfterInactivitySeconds: auth.CookieMaxAge,                 // 7 天不活动就过期
}
```

**Token 生成逻辑**：[auth/token.go#L43-L45](auth/token.go#L43-L45)

```go
func GenerateClientToken() string {
    return generateRandomToken(clientPrefix)   // clientPrefix = "C"
}
```

最终 token 形如 `C` + 22 位 base64url 随机字符，密钥空间 ≈ 2^132，且 `GenerateNotExistingToken` 会循环直到拿到一个**不与现有 Client 表冲突**的 token。

---

### 第⑦步：CreateClient 写入 clients 表 → 命中 Token 唯一索引

**代码位置**：[api/session.go#L86-L88](api/session.go#L86-L88)

```go
if success := successOrAbort(ctx, 500, a.DB.CreateClient(&client)); !success {
    return
}
```

这步把第⑥步构造的 `Client` 持久化到数据库。对应的 model 定义 [model/client.go#L22](model/client.go#L22) 中 Token 字段也有唯一索引：

```go
Token string `gorm:"type:varchar(180);uniqueIndex:uix_clients_token"`
```

**和后续请求验证的衔接**：
- 现在 clients 表里有了 `(Token: "CAbc123...", UserID: user.ID, ...)` 一行
- 第⑪步后续请求的 cookie 中就携带这个 `CAbc123...`，再通过 `GetClientByToken("CAbc123...")` 反向查回这行，从而还原 `UserID`
- 本质上：`clients` 表是"会话 token → 用户身份"的映射表

---

### 第⑧步：SetCookie 签发 → 写回 Set-Cookie 响应头

**代码位置**：[api/session.go#L90](api/session.go#L90)

```go
auth.SetCookie(ctx.Writer, client.Token, auth.CookieMaxAge, a.SecureCookie)
//           ↑           ↑              ↑                ↑
//         ResponseWriter  会话 token    7 天 = 604800s    前置 0 中 conf.Server.SecureCookie
```

**SetCookie 实现**：[auth/cookie.go#L12-L21](auth/cookie.go#L12-L21)

```go
const CookieName = "gotify-client-token"
const CookieMaxAge = 604800   // 7 天

func SetCookie(w http.ResponseWriter, token string, maxAge int, secure bool) {
    http.SetCookie(w, &http.Cookie{
        Name:     CookieName,            // "gotify-client-token"
        Value:    token,                 // "CAbc123..."
        Path:     "/",
        MaxAge:   maxAge,                // 604800
        Secure:   secure,                // ← 从配置来，HTTPS 时应为 true
        HttpOnly: true,                  // JS 不可读，防 XSS 窃取
        SameSite: http.SameSiteStrictMode, // 严格模式，防 CSRF
    })
}
```

**和入库侧的对称**：
- 入库时写入 `users` 表（永久身份）
- 这里写入的是 cookie 中的 `clients.token`（临时会话凭证）
- 两者通过 `client.UserID = user.ID` 这个外键关系关联，但 cookie 本身**不包含密码、不包含用户名**，只暴露一个不可猜测的随机 token

实际发送的 HTTP 响应头：
```http
HTTP/1.1 200 OK
Set-Cookie: gotify-client-token=CAbc123...; Path=/; Max-Age=604800;
            HttpOnly; SameSite=Strict
            ↑ 如果 SecureCookie=true，还会多一个 Secure 标志
```

---

### 第⑨步：响应 200，登录完成

**代码位置**：[api/session.go#L92-L99](api/session.go#L92-L99)

```go
ctx.JSON(200, &model.CurrentUserExternal{
    ID:            user.ID,
    Name:          user.Name,            // "admin"
    Admin:         user.Admin,           // true（首启注入时设置的）
    CreatedAt:     user.CreatedAt,
    ClientID:      client.ID,            // 新建 Client 的主键
    ElevatedUntil: client.ElevatedUntil, // 1 小时后需要重新提升
})
```

**浏览器端收到的效果**：
1. 解析 JSON，知道当前用户是 `admin` 且是管理员
2. 浏览器自动保存 `Set-Cookie` 头中的 cookie
3. 后续所有对同域的请求，浏览器自动在 `Cookie` 头中附加 `gotify-client-token=CAbc123...`

---

### 第⑩步：后续请求携带 cookie → 路由中间件 RequireClient

用户在浏览器中访问 `/user` 页面，浏览器自动发出：

```http
GET /user HTTP/1.1
Cookie: gotify-client-token=CAbc123...
```

这个端点在 router 中的注册挂了 `authentication.RequireClient` 中间件：

```go
// 来自 router/router.go 的典型注册
g.Group("/user").Use(authentication.RequireClient).GET("", userHandler.GetUsers)
```

中间件 `RequireClient` 实现在 [auth/authentication.go#L52-L54](auth/authentication.go#L52-L54)：

```go
func (a *Auth) RequireClient(ctx *gin.Context) {
    // 按以下顺序尝试认证，全部失败就返回 401
    a.evaluateOr401(ctx,
        a.handleUser(),       // 第一优先级：Basic Auth（用户名+密码）
        a.handleClient())     // 第二优先级：Client Token（cookie / header）
}
```

这个请求没有 BasicAuth 头，所以 `handleUser` 返回 `authStateSkip`，继续下一个 `handleClient`。

---

### 第⑪步：handleClient 验证 cookie → 还原身份

**代码位置**：[auth/authentication.go#L148-L181](auth/authentication.go#L148-L181)

逐段拆开：

**子步骤 A：从请求中读 token**

```go
token, isCookie := a.readTokenFromRequest(ctx)
```

`readTokenFromRequest` 按优先级尝试 4 个来源，本例中命中第 4 个：

```go
// auth/authentication.go#L210-L229
func (a *Auth) readTokenFromRequest(ctx *gin.Context) (string, bool) {
    if token := a.tokenFromQuery(ctx); token != "" { return token, false }       // 1. URL ?token=...
    if token := a.tokenFromXGotifyHeader(ctx); token != "" { return token, false } // 2. X-Gotify-Key: ...
    if token := a.tokenFromAuthorizationHeader(ctx); token != "" { return token, false } // 3. Authorization: Bearer ...
    if token := a.tokenFromCookie(ctx); token != "" { return token, true }        // 4. ★ Cookie 头（本例走这个）
    return "", false
}

func (a *Auth) tokenFromCookie(ctx *gin.Context) string {
    token, err := ctx.Cookie(cookieName)   // cookieName = "gotify-client-token"
    if err != nil { return "" }
    return token   // 返回 "CAbc123..."
}
```

命中后返回 `(token="CAbc123...", isCookie=true)`。

**子步骤 B：查 clients 表 → 命中 Token 唯一索引**

```go
client, err := a.DB.GetClientByToken(token)    // token = "CAbc123..."
if client == nil { return authStateSkip, nil } // token 不存在 → 跳过这个认证方式
```

生成的 SQL 是 `SELECT * FROM clients WHERE token = ?`，使用的索引正是第⑦步 `CreateClient` 时写入的 `uix_clients_token`，精准命中同一行。

**子步骤 C：把 client 注册到 gin context**

```go
RegisterClient(ctx, client)   // 后续 handler 用 auth.GetClient(ctx) 就能拿到
```

**子步骤 D：滑动刷新有效期 + 刷新 cookie**

```go
now := timeNow()
if client.LastUsed == nil || client.LastUsed.Add(5*time.Minute).Before(now) {
    // 距上次使用超 5 分钟：
    //   1) 刷新 DB 中 LastUsed 和 ExpiresAt
    a.DB.UpdateClientTokensLastUsedAndExpiresAt([]string{client.Token}, &now)
    //   2) 如果 token 来自 cookie，重新签发 cookie 刷新 Max-Age
    if isCookie {
        SetCookie(ctx.Writer, client.Token, CookieMaxAge, a.SecureCookie)
        //         ↑ 这里的 a.SecureCookie 就是前置 0 透传的同一值
    }
}
```

**子步骤 E：权限检查（admin / elevated）**

```go
for _, check := range checks {   // checks 是调用方传入的闭包列表
    state, err := check(client)  // 例如 checkClientAdmin → 查 users 表看 Admin 是否为 true
    if state != authStateOk { return state, err }
}
return authStateOk, nil
```

**子步骤 F：中间件放行，执行业务 handler**

`evaluate` 中返回 `authStateOk` → 调用 `ctx.Next()` → 进入 `userHandler.GetUsers`，里面通过 `auth.GetUserID(ctx)` 就能拿到 `client.UserID`，从而知道当前操作的是哪个用户。

---

### 登录闭环的三段对称关系汇总

| 段 | 写入侧（首启或登录时） | 读取侧（登录或后续请求时） | 对称点 |
|---|---|---|---|
| **① 用户身份** | `db.Create(User{Name:"admin", Pass:$2a$10..., Admin:true})` 命中 `uix_users_name` | `GetUserByName("admin")` 同一索引查询；`ComparePassword($2a$10..., plain)` 同一 bcrypt 算法 | 同一个唯一索引、同一 bcrypt 哈希对 |
| **② 会话映射** | `CreateClient(Token:"CAbc123", UserID:user.ID)` 命中 `uix_clients_token` | `GetClientByToken("CAbc123")` 同一索引查询；取出 `UserID` 还原身份 | `clients` 表是 "随机 token → 用户身份" 的可逆映射 |
| **③ cookie 签发** | `SetCookie(token, secure, HttpOnly, SameSiteStrict)` 写 `Set-Cookie` 头 | `tokenFromCookie()` 读 `Cookie` 头；刷新有效期时再次用同一 `secure` 值重发 | `SecureCookie` 配置在签发侧和刷新侧共享；token 值端到端一致 |

**关键理解**：从"用户输入 admin/admin"到"中间件放行请求"，中间走了两次唯一索引、两次 bcrypt/哈希对称、一次外键关联还原。没有任何一步把明文密码或身份写进 cookie，cookie 只是一个指向映射表的随机索引，这是现代 Web 会话管理的标准做法。

---

## 十一、RequireAdmin 不对称：Basic Auth 直过 vs Cookie Session 需 Elevated

首位管理员入库后 Admin=true，但并不意味着拿着 cookie 就能访问所有管理员接口。RequireAdmin 中间件在 Basic Auth 和 Cookie Session 两条路径上采用**完全不同的判定逻辑**。

### 11.1 RequireAdmin 的两条路径对比

**代码位置**：[auth/authentication.go#L45-L48](auth/authentication.go#L45-L48)

`go
func (a *Auth) RequireAdmin(ctx *gin.Context) {
    a.evaluateOr401(ctx,
        a.handleUser(a.checkUserAdmin),
        a.handleClient(a.checkClientAdmin, a.checkClientElevated))
}
`

| 认证方式 | 判定条件 | 代码位置 |
|---|---|---|
| **Basic Auth** | 只需 user.Admin == true | [checkUserAdmin](auth/authentication.go#L270-L275) |
| **Cookie Session** | 需同时满足：<br> user.Admin == true<br> client.ElevatedUntil 未过期 | [checkClientAdmin](auth/authentication.go#L254-L261) + [checkClientElevated](auth/authentication.go#L263-L268) |

**设计意图**：Basic Auth 每次请求都重验密码，可信度高，直接放行；Cookie Session 是长期会话，敏感操作前需额外"提升（elevate）"一步，类似 sudo。

### 11.2 checkUserAdmin：Basic Auth 路径（只看 Admin 标志）

`go
func (a *Auth) checkUserAdmin(user *model.User) (authState, error) {
    if !user.Admin {
        return authStateForbidden, nil
    }
    return authStateOk, nil
}
`

只要 user.Admin == true 就放行，无时间窗口限制，无额外提升步骤。

### 11.3 checkClientAdmin + checkClientElevated：Cookie 路径（双重检查）

**第一关：checkClientAdmin  用户必须是管理员**

`go
func (a *Auth) checkClientAdmin(client *model.Client) (authState, error) {
    user, err := a.DB.GetUserByID(client.UserID)
    if !user.Admin {
        return authStateForbidden, nil
    }
    return authStateOk, nil
}
`

**第二关：checkClientElevated  会话必须处于提升状态**

`go
func (a *Auth) checkClientElevated(client *model.Client) (authState, error) {
    if client.ElevatedUntil == nil || !timeNow().Before(*client.ElevatedUntil) {
        return authStateNotElevated, nil
    }
    return authStateOk, nil
}
`

两关是 AND 关系：都通过才放行。

### 11.4 ElevatedUntil 的来源与掉权机制

**来源 1：登录时自动提升 1 小时**

[api/session.go#L78](api/session.go#L78)：

`go
elevatedUntil := time.Now().Add(model.DefaultElevationDuration)
`

model.DefaultElevationDuration = 1 小时（见 model/elevate.go）。

**来源 2：OIDC 登录时同样自动提升**

[api/oidc.go#L425](api/oidc.go#L425)，同样 1 小时。

**来源 3：手动提升**

调用提升接口传入 durationSeconds 重置 ElevatedUntil。

**掉权条件**：当前时间 >= ElevatedUntil 时立即掉权，访问 RequireAdmin 接口返回 403，错误 "session not elevated, use basic auth or call /client:elevate"。掉权只影响管理员接口，普通接口不受影响。

### 11.5 与首启 admin 的衔接

首启注入的 dmin 用户 Admin=true，所以：
- Basic Auth 访问管理员接口  每次都放行
- Cookie 登录  前 1 小时能访问管理员接口  1 小时后掉权  需重新提升

---

## 十二、OIDC 平行登录入口与 admin 重名场景

本地登录（/auth/local/login）不是唯一入口。conf.OIDC.Enabled = true 时，系统还注册一套 OIDC 登录端点，与本地登录平行共存，共用 users 表和 clients 表。

### 12.1 OIDC 路由注册的条件

**代码位置**：[router/router.go#L119-L127](router/router.go#L119-L127)

`go
if conf.OIDC.Enabled {
    oidcHandler := api.NewOIDC(conf, db, userChangeNotifier)
    oidcGroup := g.Group("/auth/oidc")
    oidcGroup.GET("/login", oidcHandler.LoginHandler())
    oidcGroup.GET("/callback", oidcHandler.CallbackHandler())
    oidcGroup.GET("/elevate", oidcHandler.ElevateHandler)
}
`

- OIDC 路由只在启用时注册
- 本地登录路由始终存在
- 两者共用 users 表和 clients 表

### 12.2 OIDC 登录流程（与本地登录平行）

| 步骤 | 本地登录 | OIDC 登录 |
|---|---|---|
| 入口 | POST /auth/local/login Basic Auth | GET /auth/oidc/login 跳转 Provider |
| 身份验证 | bcrypt 比对本地密码 | OIDC Provider 验证 |
| 用户查找 | GetUserByName(name) | esolveUser(info)  GetUserByName |
| 新用户处理 | 不存在  401 | 不存在且 AutoRegister  自动创建（Admin=false, Pass=nil） |
| 会话创建 | CreateClient | CreateClient（同一套） |
| Cookie 签发 | SetCookie | SetCookie（同一函数） |
| 初始提升 | 1 小时 | 1 小时 |

esolveUser 核心逻辑（[api/oidc.go#L395-L422](api/oidc.go#L395-L422)）：

`go
func (a *OIDCAPI) resolveUser(info *oidc.UserInfo) (*model.User, int, error) {
    username := fmt.Sprint(info.Claims[a.UsernameClaim])
    user, err := a.DB.GetUserByName(username)
    if user == nil {
        if !a.AutoRegister { return nil, http.StatusForbidden, errors.New("user not found") }
        user = &model.User{Name: username, Admin: false, Pass: nil}
        a.DB.CreateUser(user)
    }
    return user, 0, nil
}
`

两个入口都通过 GetUserByName 查 users 表，命中同一个 uix_users_name 唯一索引。

### 12.3 admin 重名场景分析

首启注入了本地 admin 账号（Name: "admin", Admin: true）。如果 OIDC Provider 上也有叫 dmin 的用户：

1. OIDC 认证通过，claims 中 preferred_username = "admin"
2. esolveUser 调 GetUserByName("admin")  命中本地 admin
3. 拿到 Admin=true 的用户对象
4. 创建 Client，1 小时提升，签发 cookie
5. **结果：OIDC 的 admin 用户直接获得 Gotify 管理员权限**

**风险**：若 OIDC Provider 是第三方公共服务，本地恰好有 dmin 用户，可能越权。建议生产环境用 OIDC 时规划好用户名或改名本地 admin。

**反向**：本地 admin 登录不受 OIDC 影响，走 /auth/local/login，两者互相独立。

### 12.4 OIDC 登录后的掉权与提升

OIDC 登录同样 1 小时提升窗口。掉权后走 /auth/oidc/elevate 重新认证刷新 ElevatedUntil，与本地"重新输入密码"平行设计。

---

## 十三、模块衔接时序图

```
  用户命令行                         app.go                       config                        database
──────┬──────────────────────────────┬────────────────────────────┬───────────────────────────────┬──────
      │  ./gotify server             │                            │                               │
      │─────────────────────────────>│                            │                               │
      │                              │  mode.Set(Mode)            │                               │
      │                              │  config.Get()              │                               │
      │                              │───────────────────────────>│                               │
      │                              │                            │ configFiles() 分支            │
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
      │  Authorization: Basic YWRt   │                            │                               │
      │─────────────────────────────>│───────────────────────────────────────────────────────────>│
      │                              │                            │                               │ GetUserByName("admin")
      │                              │                            │                               │ → 命中唯一索引
      │                              │                            │                               │ ComparePassword(hash, "admin")
      │                              │                            │                               │ → true
      │                              │                            │                               │ CreateClient(token, userID)
      │                              │                            │                               │ SetCookie(token, 7d)
      │<─────────────────────────────│<───────────────────────────────────────────────────────────│
      │  200 OK + Set-Cookie         │                            │                               │
      │  gotify-client-token=CAbc... │                            │                               │
```

---

## 十四、关键设计要点总结

| 设计点 | 实现方式 | 代码位置 |
|--------|---------|---------|
| **Mode → 配置文件链路** | ldflags `-X main.Mode=prod` → `mode.Set` → `configFiles()` 分支 | [app.go#L33](app.go#L33)、[config.go#L71-L76](config/config.go#L71-L76) |
| **凭据来源多层级** | 环境变量 > config.yml > struct tag 默认值 | [config.go#L79-L87](config/config.go#L79-L87) |
| **幂等注入** | 双重条件：`createDefaultUserIfNotExist && userCount == 0` | [database.go#L94-L98](database/database.go#L94-L98) |
| **多副本竞态兜底** | `User.Name` 唯一索引 + Create error 被静默忽略 | [user.go#L8](model/user.go#L8)、[database.go#L97](database/database.go#L97) |
| **密码安全存储** | bcrypt 哈希，默认 cost=10，明文不落库 | [password.go#L5-L12](auth/password/password.go#L5-L12) |
| **强度可配置** | `strength` 形参化，测试用 5，生产用 10 | [database.go#L33](database/database.go#L33) |
| **注入开关可控制** | `createDefaultUserIfNotExist` 形参化，避免全局状态耦合 | [database.go#L33](database/database.go#L33) |
| **登录闭环对称** | bcrypt 入库/校验对称 + 唯一索引读写对接 + cookie→Client token 对应 | [session.go#L56-L100](api/session.go#L56-L100) |
| **SecureCookie 签发** | Client token 做 cookie 值，HttpOnly + SameSite=Strict | [cookie.go#L12-L21](auth/cookie.go#L12-L21) |
| **迁移与注入顺序** | AutoMigrate → 计数 → 注入（先建表再写数据） | [database.go#L90-L98](database/database.go#L90-L98) |
| **数据库方言兼容** | MySQL / PostgreSQL / SQLite3 统一入口 | [database.go#L52-L59](database/database.go#L52-L59) |
| **SQLite 并发保护** | `SetMaxOpenConns(1)` 强制串行写 | [database.go#L74-L80](database/database.go#L74-L80) |

---

## 十五、操作指引：自定义首位管理员

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
