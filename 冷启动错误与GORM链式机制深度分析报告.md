# 冷启动错误丢弃 · GORM 链式机制 · successOrAbort 全统计 · 驱动翻译一致性 — 深度分析报告

---

## 一、默认用户冷启动：那两行到底丢了几次错误？

### 1.1 代码原文

**文件**：[database/database.go:88-92](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/database/database.go#L88-L92)

```go
userCount := int64(0)
db.Find(new(model.User)).Count(&userCount)    // ← 第 1 处：整条链的返回值被丢弃

if createDefaultUserIfNotExist && userCount == 0 {
    db.Create(&model.User{...})                // ← 第 2 处：返回值被丢弃
}
```

### 1.2 答案：**2 次独立的错误丢弃**

| 语句 | 操作 | 返回值类型 | `.Error` 是否被读取 | 错误丢弃？ |
|-----|------|-----------|:------------------:|:---------:|
| `db.Find(new(model.User)).Count(&userCount)` | 先 SELECT \* 再 SELECT COUNT(*) | `*gorm.DB` | ❌ 从未读取 | ✅ 丢弃 1 次 |
| `db.Create(&model.User{...})` | INSERT 一条默认用户 | `*gorm.DB` | ❌ 从未读取 | ✅ 丢弃 2 次 |

注意：第一行是 **一条链**（Find → Count），但它只构成 **一次丢弃**——因为整条链的最终 `*gorm.DB` 返回值被整体丢弃，其 `.Error` 字段从未被访问。

---

## 二、GORM 链式 Session 机制详解

### 2.1 GORM v1 链式调用的核心原理

GORM 的所有方法都返回 `*gorm.DB`，但这 **不是同一个对象**。每次调用都在内部 **克隆 Statement**，形成一条调用链：

```
db  (原始根 *gorm.DB，Statement = 空/默认)
 │
 ├─ .Model(&User{})    → 克隆 Statement，设置 Model = User
 │   │
 │   ├─ .Where("id=?", 1)  → 再克隆，追加 Where 条件
 │   │   │
 │   │   └─ .First(&user)  → 克隆 + 执行 finisher → 返回新 *gorm.DB（含 .Error）
 │   │
 │   └─ .Count(&count)     → 克隆 + 执行 finisher → 返回另一个 *gorm.DB（含 .Error）
 │
 └─ .Create(&user)         → 克隆 + 执行 finisher → 又一个 *gorm.DB
```

**关键点**：
- 每个 finisher（Find / First / Count / Create / Save / Delete 等）都会：
  1. 先检查 `db.Error` 是否已有值 → 有则跳过执行，直接返回
  2. 构建 SQL 并执行
  3. 把执行结果（数据 + error）写入返回的新 `*gorm.DB`
- **错误不会"回流"到原始 `db` 对象**，只存在于 finisher 返回的那个链对象上
- 如果不接住 finisher 的返回值，error 就随对象一起被 GC 了

### 2.2 `db.Find(new(model.User)).Count(&userCount)` 的实际执行流程

这是一个 **双 finisher 链**，会执行 **两条独立的 SQL**：

```
时序：
  T0: userCount = 0 (int64 零值)
  
  T1: db.Find(new(model.User))
        ├─ 克隆 Statement
        ├─ 构建 SQL: SELECT * FROM "users"
        ├─ 执行 → 返回所有行
        ├─ 把结果扫描进 new(model.User) 这个临时对象
        │   （这个临时对象没有被赋值给任何变量，马上被 GC）
        ├─ 如果出错，错误放在返回的 *gorm.DB 的 .Error 里
        └─ 返回 *gorm.DB（链对象 #1）
  
  T2: 链对象 #1.Count(&userCount)
        ├─ 先检查链对象 #1 的 .Error
        │   ├─ 如果 Find 已经出错 → 直接返回同一 error，不执行新 SQL
        │   └─ 如果 Find 成功 → 继续
        ├─ 重新构建 SQL: SELECT count(*) FROM "users"
        │   （Count 会改写 SELECT 子句，忽略之前 Find 的字段）
        ├─ 执行 → 返回单一数值
        ├─ 把 count 值写入 &userCount（用户传入的指针）
        ├─ 如果出错，错误放在返回的链对象 #2 的 .Error 里
        └─ 返回 *gorm.DB（链对象 #2） ← 这个返回值被丢弃！
  
  T3: userCount 变量
        ├─ 如果 Count 成功 → 被写入实际行数（如 0 / 3 / 100）
        └─ 如果 Count 失败 → 保持初始值 0 不变 ★
```

**执行了几条 SQL？两条**：
1. `SELECT * FROM users`（Find）
2. `SELECT count(*) FROM users`（Count）

> ⚠️ `Find` 其实在这里完全多余。它只是为 Count 提供"表名 / Model 类型"的来源，等价于 `db.Model(&model.User{}).Count(&userCount)`，但多执行了一次全表 SELECT *，浪费性能。

### 2.3 Find 失败时 userCount 是不是还是 0？

**答案：是的，保持零值 0 不变。**

因为：
1. `userCount := int64(0)` 初始化为 0
2. `Count(&userCount)` 只有在 **成功执行** 时才会修改 `userCount` 的值
3. 如果 Count 之前就有错误（比如 Find 失败），Count finisher 直接返回 error，**不会写入 `userCount`**
4. 如果 Count 自身执行失败（比如数据库连接断在执行时），也 **不会写入 `userCount`**

→ 所以 **任何一步失败，`userCount` 都保持初始值 `0`**

### 2.4 首启误判的后果（最隐蔽的 Bug）

**场景**：首次启动时数据库连接有问题（密码错、权限 revoked、磁盘只读等）

```
时序推演：
  T0: database.New() 开始执行
  T1: gorm.Open(...) → 成功（只是建立连接池，不一定真能执行 SQL）
       （很多数据库驱动的 Open 是懒连接，第一次真实查询才会报错）
  T2: db.AutoMigrate(...) → 可能成功也可能失败
       （如果失败了，New 会 return error，app.go 里 panic，我们先看 AutoMigrate 成功但后续查询失败的情况）
  T3: db.Find(new(model.User)).Count(&userCount)
       ├─ Find 执行 → 失败（DB 密码错误 / 表不存在 / 连接断了）
       │   └─ .Error 被写入链对象 #1
       ├─ Count 发现链对象 #1 已有 Error → 直接返回，不执行
       │   └─ .Error 同样在链对象 #2 中
       └─ 整条链的返回值（链对象 #2）被丢弃 → 错误无人知晓
  T4: userCount == 0 为 true（因为没被修改过）
  T5: 进入 if → 执行 db.Create(&model.User{...})
       ├─ INSERT 也失败（同一个连接问题）
       ├─ .Error 写入返回值
       └─ 返回值被丢弃 → 错误还是无人知晓
  T6: 继续执行 fillMissingSortKeys / fillMissingCreatedAt
       （这些也可能失败，但它们有 err 检查吗？看代码：有，而且会 return error）
```

等等，让我重新看一下 [database/database.go:94-99](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/database/database.go#L94-L99)：

```go
if err := db.Transaction(fillMissingSortKeys, ...); err != nil {
    return nil, err      // ← 这里会返回错误！
}

if err := db.Transaction(func(tx *gorm.DB) error { return fillMissingCreatedAt(tx, now()) }, ...); err != nil {
    return nil, err      // ← 这里也会返回错误！
}
```

**修正：如果数据库有真实连接问题，fillMissingSortKeys 也会失败，错误会被正常返回，New() 会 return error，app.go 里会 panic**，整个进程起不来。

那"userCount 误判"在什么场景下才会真的造成问题？

**答案：一种非常特定的场景 —— 读权限有了但写权限没有，或者 SELECT 能走缓存/副本但 INSERT 要走主库且主库挂了。**

更常见的真实场景是：
- **表 users 被意外 DROP 了**：Find 会报"表不存在"，Count 也会报错 → userCount 还是 0 → 尝试 Create → 也报错 → 返回值丢弃 → fillMissingSortKeys 操作 applications 表（存在）→ 成功 → fillMissingCreatedAt 也成功 → New() 返回 nil error → 进程启动成功但没有默认用户也没法登录

不，等一下，如果 users 表不存在，fillMissingCreatedAt 也会操作 users 表（[database/database.go:105-117](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/database/database.go#L105-L117)）：

```go
func fillMissingCreatedAt(db *gorm.DB, now time.Time) error {
    models := []any{
        new(model.User),       // ← 第一个就是 User！
        new(model.Application),
        ...
    }
    for _, m := range models {
        if err := db.Model(m).Where("created_at IS NULL").UpdateColumn(...).Error; err != nil {
            return err          // ← 会返回错误
        }
    }
    return nil
}
```

**所以最终结论**：大多数数据库故障都会在后续的迁移步骤中被捕获并导致 New() 失败。`db.Find().Count()` 和 `db.Create()` 的错误丢弃虽然是代码味道，但在常规故障模式下不会造成"静默启动成功但功能不可用"。

**但有一种场景确实会被漏过去**：
- `users` 表存在且可读
- `applications` 表也没问题（迁移都过了）
- 但 **只有 `users` 表的 INSERT 权限被 revoke 了**（非常罕见的运维配置错误）
- → Count 成功，userCount=0（真的是 0）
- → Create 失败（没权限）
- → 错误被丢弃
- → 迁移也都成功（只操作 applications 等表或者只读 users）
- → New() 返回 nil error
- → 进程起来了，但默认用户不存在，admin/admin 登录不上

这个场景虽然极端，但确实存在——而且排错极难，因为启动日志里**完全没有任何错误信息**。

---

## 三、NewManager 崩溃：panic 还是 log.Fatal？Gin Recovery 救得了吗？

### 3.1 代码定位

**文件**：[router/router.go:103-106](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/router/router.go#L103-L106)

```go
pluginManager, err := plugin.NewManager(db, conf.PluginsDir, g.Group("/plugin/:id/custom/"), streamHandler)
if err != nil {
    panic(err)    // ← 是 panic，不是 log.Fatal
}
```

**不是** `log.Fatal`，是 `panic(err)`。

### 3.2 panic vs log.Fatal 的区别

| 特性 | `panic(err)` | `log.Fatal(err)` |
|-----|:-----------:|:----------------:|
| 输出到 stderr | ✅（Go 运行时输出 panic 堆栈） | ✅（log 包输出 + os.Exit） |
| 能否被 recover 捕获 | ✅ 可以（在同一个 goroutine 中） | ❌ 不行（内部调 os.Exit，直接退出） |
| defer 是否执行 | ✅ 会执行（在 recover 前/栈展开时） | ❌ 不执行（os.Exit 直接退出） |
| 退出码 | 2（Go 默认 panic 退出码） | 1（log.Fatal 内部的 os.Exit(1)） |

### 3.3 Gin Recovery 能救吗？

**答案：完全救不了。**

原因有两层：

#### 原因 1：执行时机不同

```
启动时序：
  main()
    ├─ database.New()        ← 初始化 DB，如果失败直接 panic
    ├─ router.Create()       ← 这里面创建 pluginManager
    │   ├─ gin.New()         ← 创建 gin Engine
    │   ├─ g.Use(gin.Recovery(), gerror.Handler(), ...)  ← 注册中间件
    │   │                     （中间件只是注册到 g 里，还没开始处理请求）
    │   ├─ ... 各种 handler 初始化 ...
    │   └─ plugin.NewManager(...) ← 这里 panic！
    │                            发生在 router.Create 函数内部
    │                            在 main goroutine 中
    ├─ runner.Run(engine)    ← 启动 HTTP 服务（如果到得了这一步的话）
```

`gin.Recovery()` 是一个 **HTTP 中间件**，它只会在处理 HTTP 请求的 goroutine 中、在 `c.Next()` 的 recover 里捕获 panic。

而 `plugin.NewManager()` 是在 **main goroutine** 中、**服务启动之前** 同步调用的。它的 panic 没有任何 recover 兜底，会直接导致整个进程崩溃。

#### 原因 2：goroutine 不同

Gin 处理每个请求时会有自己的 goroutine（net/http 提供），Recovery 中间件的 recover 只在那个 goroutine 里生效。main goroutine 中的 panic 是另一条完全独立的调用栈，Recovery 根本接触不到。

### 3.4 崩溃时用户能看到什么？

**运维视角（容器/K8s）**：
- Pod 状态：`CrashLoopBackOff`
- 退出码：`2`（Go panic 默认退出码）
- 日志：stderr 中有完整的 panic stack trace

**代码视角**：
- `app.go` 中 `router.Create(db, vInfo, conf)` 的调用没有 recover
- panic 会一路冒泡到 main 函数顶部
- Go 运行时输出 panic 信息后终止进程

### 3.5 其他启动期 panic 位置汇总

| 位置 | 触发条件 |
|-----|---------|
| [app.go:36](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/app.go#L36) | PluginsDir 目录创建失败 |
| [app.go:40](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/app.go#L40) | UploadedImagesDir 目录创建失败 |
| [app.go:45](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/app.go#L45) | database.New() 失败 |
| [router/router.go:105](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/router/router.go#L105) | plugin.NewManager() 失败 |
| [database/database.go:157](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/database/database.go#L157) | SQLite 数据目录创建失败 |
| [ui/serve.go:28/42/54](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/ui/serve.go) | UI 静态资源嵌入失败 |
| [auth/token.go:22](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/auth/token.go#L22) | 加密随机源不可用 |
| [auth/password/password.go:9](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/auth/password/password.go#L9) | bcrypt 初始化失败 |
| [config/config.go:83](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/config/config.go#L83) | 配置文件加载失败 |
| [mode/mode.go:41](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/mode/mode.go#L41) | 运行模式未知 |

全部都是启动期的、main goroutine 中的 panic，没有一个能被 gin.Recovery 接住。

---

## 四、successOrAbort 全量统计与分类

### 4.1 总数

Grep 全项目得到 **48 处** `successOrAbort(` 匹配，但其中：
- 1 处是 **函数定义**（[api/errorHandling.go:5](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/api/errorHandling.go#L5)）
- 1 处是 **测试调用**（[api/errorHandling_test.go:15](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/api/errorHandling_test.go#L15)）

→ **生产代码中共 46 处调用**

### 4.2 按文件分类

| 文件 | 调用次数 | 主要用途 |
|-----|:-------:|---------|
| [api/user.go](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/api/user.go) | 13 处 | 用户 CRUD、管理员计数检查、密码修改等 |
| [api/message.go](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/api/message.go) | 10 处 | 消息 CRUD、批量删除、按应用/用户删除 |
| [api/plugin.go](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/api/plugin.go) | 8 处 | 插件配置读写、启用禁用、显示获取等 |
| [api/application.go](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/api/application.go) | 8 处 | 应用 CRUD、图片上传、排序更新等 |
| [api/client.go](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/api/client.go) | 6 处 | 客户端 CRUD、更新等 |
| [api/session.go](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/api/session.go) | 2 处 | 登录创建 client、登出删除 client |

### 4.3 按状态码参数分类

所有 46 处调用中，**传入的状态码全部是 500**：

```
successOrAbort(ctx, 500, ...)
```

没有任何一处传 400、401、403、404 等其他状态码。

**为什么都是 500？**

因为 `successOrAbort` 被设计为 **"数据库操作失败的兜底处理"**——数据库操作失败被视为"服务器内部错误"。但这个设计有问题：数据库操作失败不全是 500，唯一索引冲突、外键约束失败等**用户可理解、可操作的错误**应该返回 4xx。

### 4.4 按调用方式分类

| 调用方式 | 数量 | 示例 |
|---------|:----:|------|
| `if success := ... ; !success { return }` | 大多数（约 38 处） | `if success := successOrAbort(ctx, 500, err); !success { return }` |
| 直接调用，不接收返回值 | 约 8 处 | `successOrAbort(ctx, 500, a.DB.DeleteMessagesByUser(userID))` |

后者（直接调用不接收返回值）通常出现在函数末尾，调用完之后自然 return，不需要显式 return。

### 4.5 为什么说 successOrAbort 是"万金油但不治病"

它的实现极其简单（[api/errorHandling.go:5-9](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/api/errorHandling.go#L5-L9)）：

```go
func successOrAbort(ctx *gin.Context, code int, err error) (success bool) {
    if err != nil {
        ctx.AbortWithError(code, err)    // 无脑塞进去
    }
    return err == nil
}
```

**问题**：
1. **不分析错误类型**：是用户输入错误（4xx）还是服务器故障（5xx），完全不管
2. **不封装友好消息**：原生数据库错误直接暴露给用户（含表名、列名、索引名等内部信息）
3. **不做日志分级**：所有错误都推到 gin 的 errors 列表，由 gerror.Handler 统一处理

对比 `handleApplicationError` 就是"专科医生"——知道识别 `ErrDuplicatedKey` 并转 400。
而 `successOrAbort` 就是"全科急诊"——啥都接，但啥都不治，一律推去 500。

---

## 五、gorm.ErrDuplicatedKey 三种驱动翻译一致性

### 5.1 翻译开关：TranslateError

**配置**：[database/database.go:39](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/database/database.go#L39)

```go
gormConfig := &gorm.Config{
    TranslateError: true,  // ← 全局开启
}
```

GORM v1.31.1 引入了这个配置，作用是将各数据库驱动的**方言特定错误**翻译为 GORM 定义的**通用错误类型**。

### 5.2 三种驱动的翻译实现原理

| 驱动 | 版本 | 底层原生错误 | 翻译为 gorm.ErrDuplicatedKey 的条件 |
|-----|------|-------------|-----------------------------------|
| **MySQL** | v1.6.0（[gorm.io/driver/mysql v1.6.0](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/go.mod#L20)） | `*mysql.MySQLError` 错误号 `1062` (ER_DUP_ENTRY) | Number == 1062 → 翻译 |
| **PostgreSQL** | v1.6.0（[gorm.io/driver/postgres v1.6.0](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/go.mod#L21)） | `*pgconn.PgError` SQLSTATE `23505` (unique_violation) | Code == "23505" → 翻译 |
| **SQLite** | v1.6.0（[gorm.io/driver/sqlite v1.6.0](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/go.mod#L22)） | `sqlite3.Error` 扩展错误码 `SQLITE_CONSTRAINT_UNIQUE` | Code == sqlite3.ErrConstraint 且是唯一约束冲突 → 翻译 |

### 5.3 一致性分析

**✅ 翻译目标一致**：三种驱动最终都产出 `gorm.ErrDuplicatedKey` 这个 sentinel error，可以通过 `errors.Is(err, gorm.ErrDuplicatedKey)` 统一判断。

**✅ 错误包装一致**：GORM 的翻译实现是把原始驱动错误包装在内部，然后返回 `ErrDuplicatedKey`，使用 `fmt.Errorf("%w: %w", gorm.ErrDuplicatedKey, originalErr)` 或类似方式包装，保持错误链完整。

**⚠️ 以下边缘情况可能不一致**：

| 场景 | MySQL | PostgreSQL | SQLite | 一致性？ |
|-----|:-----:|:----------:|:------:|:-------:|
| 主键冲突（PRIMARY KEY） | ✅ 翻译 | ✅ 翻译 | ✅ 翻译 | ✅ 一致 |
| 唯一索引冲突（UNIQUE INDEX） | ✅ 翻译 | ✅ 翻译 | ✅ 翻译 | ✅ 一致 |
| 联合唯一索引冲突 | ✅ 翻译 | ✅ 翻译 | ✅ 翻译 | ✅ 一致 |
| `INSERT ... ON CONFLICT DO NOTHING` | N/A | ❌ 不返回错误 | N/A | 不同数据库语义不同 |
| 外键约束冲突 | ❌ 不翻译（不是这个错误） | ❌ 不翻译 | ❌ 不翻译 | ✅ 一致（都不翻译） |
| `ErrRecordNotFound`（另一个常用翻译） | ✅ 翻译 | ✅ 翻译 | ✅ 翻译 | ✅ 一致 |

**结论**：对于 `gorm.ErrDuplicatedKey` 这个特定错误类型，三种驱动的翻译是**一致的**。项目中 `handleApplicationError` 的判断方式 `errors.Is(err, gorm.ErrDuplicatedKey)` 在三种数据库下都能正常工作。

### 5.4 项目中实际捕获 ErrDuplicatedKey 的位置

全项目只有 **1 处** 显式捕获了 `gorm.ErrDuplicatedKey`：

- [api/application.go:486](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/api/application.go#L486) — `handleApplicationError` 函数

```go
if errors.Is(err, gorm.ErrDuplicatedKey) {
    ctx.AbortWithError(400, errors.New("sort key is not unique"))
}
```

其余所有数据库写入（CreateUser / CreateClient / CreatePluginConf / 各种 Update / 各种 Delete）遇到唯一索引冲突时，都会被 `successOrAbort` 或直接 `AbortWithError(500, err)` 推成 500 错误。

---

## 六、核心代码速查

| 关注点 | 文件 | 行号 |
|-------|------|-----|
| 默认用户冷启动：Count + Create 错误丢弃 | [database/database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/database/database.go) | 88-92 |
| GORM TranslateError 开关 | [database/database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/database/database.go) | 36-41 |
| successOrAbort 函数定义 | [api/errorHandling.go](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/api/errorHandling.go) | 5-9 |
| handleApplicationError（唯一捕获 ErrDuplicatedKey） | [api/application.go](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/api/application.go) | 485-491 |
| NewManager 失败 panic | [router/router.go](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/router/router.go) | 103-106 |
| gin.Recovery 中间件注册 | [router/router.go](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/router/router.go) | 43 |
| app.go 启动期其他 panic | [app.go](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/app.go) | 36, 40, 45 |
| fillMissingSortKeys 迁移（有 error 检查） | [database/database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/database/database.go) | 94-96, 120-151 |
| fillMissingCreatedAt 迁移（有 error 检查） | [database/database.go](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/database/database.go) | 98-100, 105-117 |
| GORM 驱动版本（三种） | [go.mod](file:///d:/fz/0601-1/solo-dogfeeding/code/76-server/go.mod) | 20-23 |

---

## 七、一句话结论汇总

| 问题 | 答案 |
|-----|------|
| 冷启动那几行有几次错误丢弃？ | **2 次**：Count 链 1 次 + Create 1 次 |
| Find 失败时 userCount 是不是 0？ | **是**，保持零值不变 |
| 首启误判有什么后果？ | 极端场景下（users 表只读不可写）进程照常启动但没有默认用户，登录失败且无报错日志 |
| NewManager 崩是 panic 还是 log.Fatal？ | **panic** |
| Gin Recovery 能救吗？ | **不能**，因为发生在 main goroutine、服务启动前，不是 HTTP 请求链路内 |
| successOrAbort 生产代码有几处？ | **46 处**，全传 500 |
| 三种驱动 ErrDuplicatedKey 翻译一致吗？ | **一致**，都可以用 `errors.Is(err, gorm.ErrDuplicatedKey)` 判断 |
| 项目中几处显式捕获了 ErrDuplicatedKey？ | **1 处**：只有 `handleApplicationError` |
