# 健康检查与版本信息接口整体图景

## 一、接口概览

| 接口 | 路径 | 方法 | 鉴权要求 | 定义位置 |
|------|------|------|----------|----------|
| 健康检查 | `/health` | GET/HEAD | 无需鉴权 | `router/router.go:119` |
| 版本信息 | `/version` | GET | 无需鉴权 | `router/router.go:162-164` |

---

## 二、健康检查接口 (`/health`)

### 2.1 核心实现

**处理函数** (`api/health.go:34-46`):

```go
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

### 2.2 读取的进程内状态

健康检查接口**不读取进程内状态**，仅通过数据库 Ping 操作判断外部依赖的可用性。

### 2.3 依赖的底层组件

```
HealthAPI
    └── HealthDatabase (接口)
            └── GormDatabase (database/ping.go:4-9)
                    └── *gorm.DB
                            └── sql.DB.Ping()
```

- **`database/ping.go:4-9`**: 调用底层 `sql.DB.Ping()` 验证数据库连接
- 检查点：数据库连接池是否可用、网络是否通畅

### 2.4 状态模型 (`model/health.go`)

```go
type Health struct {
    Health   string `json:"health"`   // 整体健康状态
    Database string `json:"database"` // 数据库健康状态
}
```

**状态等级**:
- `green`: 正常
- `orange`: 部分异常（数据库异常时整体状态）
- `red`: 严重异常

### 2.5 与系统启动顺序的关联

**启动流程** (`app.go:26-54`):

```
1. 初始化版本信息
2. 加载配置
3. 创建目录
4. 初始化数据库连接 (database.New) ←── 必须成功
5. 创建路由 (router.Create)
    ├── 注册 /health 路由 (router.go:119)
    └── 注入 DB 实例到 HealthAPI
6. 启动服务器 (runner.Run)
```

- `/health` 接口在**数据库初始化完成后**才会注册
- 服务器启动后即可调用，但数据库连接失败会在步骤 4 `panic`

### 2.6 鉴权分析

- **无需鉴权**：路由注册在认证中间件之前 (`router/router.go:119` 位于 `authentication.Require*` 之前)
- 访问方式：匿名访问
- 日志优化：本地健康检查请求不记录日志 (`router/router.go:243-244`)

### 2.7 特殊场景响应行为

#### 2.7.1 数据库初始化失败场景

**核心代码** (`app.go:42-45`):
```go
db, err := database.New(conf.Database.Dialect, conf.Database.Connection, ...)
if err != nil {
    panic(err)  // 进程直接崩溃退出
}
```

**可达性分析**：
- `database.New()` 失败 → 进程 `panic` → `router.Create()` 永不执行 → HTTP 服务器永不启动
- **结论**：`/health` 接口完全不可达，TCP 连接被拒绝

#### 2.7.2 HTTPS 重定向场景

**重定向中间件** (`router/router.go:45-66`) — 注册在所有路由之前：
```go
if conf.Server.SSL.Enabled && conf.Server.SSL.RedirectToHTTPS {
    g.Use(func(ctx *gin.Context) {
        if ctx.Request.TLS != nil {
            ctx.Next()  // 已是 HTTPS，继续处理
            return
        }
        if ctx.Request.Method != http.MethodGet && ctx.Request.Method != http.MethodHead {
            ctx.Data(http.StatusBadRequest, "text/plain; charset=utf-8", []byte("Use HTTPS"))
            ctx.Abort()
            return
        }
        // GET/HEAD 请求 302 重定向到 HTTPS
        ctx.Redirect(http.StatusFound, fmt.Sprintf("https://%s%s", host, ctx.Request.RequestURI))
        ctx.Abort()
    })
}
```

**响应行为**（中间件在路由匹配前执行）：

| 请求方式 | 协议 | 响应行为 |
|---------|------|----------|
| GET/HEAD | HTTP | 302 Found，重定向到 HTTPS 同路径 |
| GET/HEAD | HTTPS | 200 OK，正常返回健康状态 |
| POST/PUT/DELETE 等 | HTTP | 400 Bad Request，响应体 "Use HTTPS" |
| POST/PUT/DELETE 等 | HTTPS | 404 Not Found（路由不匹配） |

#### 2.7.3 非 GET/HEAD 请求场景

**路由定义** (`router/router.go:119`):
```go
g.Match([]string{"GET", "HEAD"}, "/health", healthHandler.Health)
```

**响应行为**（无 HTTPS 重定向时）：
- POST/PUT/DELETE/PATCH 等方法 → 404 Not Found（由 `NoRoute` 处理）
- 只有 GET 和 HEAD 方法能命中路由

---

## 三、版本信息接口 (`/version`)

### 3.1 核心实现

**处理函数** (`router/router.go:162-164`):

```go
g.GET("version", func(ctx *gin.Context) {
    ctx.JSON(200, vInfo)
})
```

### 3.2 读取的进程内状态

版本信息在**编译时注入**，进程启动后为只读状态：

```go
// app.go:15-24
var (
    Version   = "unknown"   // 版本号
    Commit    = "unknown"   // Git Commit Hash
    BuildDate = "unknown"   // 构建时间
    Mode      = mode.Dev    // 构建模式
)
```

### 3.3 依赖的底层组件

```
VersionInfo (model/version.go)
    ├── Version string   ← 编译时注入
    ├── Commit string    ← 编译时注入
    └── BuildDate string ← 编译时注入
```

- **无运行时依赖**：不依赖数据库、网络或任何外部组件
- 数据来源：编译时通过 `-ldflags` 注入

### 3.4 与系统启动顺序的关联

**启动流程** (`app.go:26-54`):

```
1. 初始化版本信息 ←── 最早执行，编译期已确定
    vInfo := &model.VersionInfo{
        Version: Version,
        Commit: Commit,
        BuildDate: BuildDate
    }
2. 加载配置
3. 创建目录
4. 初始化数据库连接 (database.New) ←── 失败则 panic
5. 创建路由 (router.Create)
    └── 注册 /version 路由 (router.go:162)
        └── 注入 vInfo 指针
6. 启动服务器 (runner.Run)
```

- `/version` 接口返回的数据在**程序启动第一时间**就已确定
- 虽然 `vInfo` 对象在内存中存在，但 HTTP 服务器启动依赖数据库初始化成功

### 3.5 鉴权分析

- **无需鉴权**：路由注册在认证中间件之前 (`router/router.go:162` 位于 `authentication.Require*` 之前)
- 访问方式：匿名访问

### 3.6 特殊场景响应行为

#### 3.6.1 数据库初始化失败场景 — 核心可达性核对

**关键问题**：版本数据在内存中存在，但数据库初始化失败时，接口是否还能访问？

**执行流程精确分析** (`app.go:26-54`):
```go
1. vInfo := &model.VersionInfo{...}  // ✅ 版本信息已创建，在内存中
2. conf := config.Get()              // ✅ 配置加载完成
3. 创建目录 (PluginsDir, ImagesDir)  // ✅ 目录创建完成
4. db, err := database.New(...)      // ❌ 失败，panic(err)
   → 进程崩溃退出（无 recover）
   → router.Create() 永不执行
   → runner.Run() 永不执行
   → HTTP 监听端口永不打开
```

**结论（经过代码核对）**：
- ❌ **`/version` 接口完全不可达**，TCP 连接被拒绝
- 虽然 `vInfo` 对象在内存中逻辑存在，但 HTTP 服务器未启动，无法通过网络访问
- 这是一个常见的认知误区：**不要以为版本接口不依赖数据库就一定能访问**，它同样依赖完整的启动流程

#### 3.6.2 HTTPS 重定向场景

**重定向中间件** (`router/router.go:45-66`) 注册在 `/version` 路由之前，因此同样适用：

| 请求方式 | 协议 | 响应行为 |
|---------|------|----------|
| GET | HTTP | 302 Found，重定向到 HTTPS 同路径 |
| GET | HTTPS | 200 OK，正常返回版本信息 |
| HEAD | HTTP | 302 Found，重定向到 HTTPS 同路径 |
| HEAD | HTTPS | 404 Not Found（路由不匹配，只注册了 GET） |
| POST/PUT/DELETE 等 | HTTP | 400 Bad Request，响应体 "Use HTTPS" |
| POST/PUT/DELETE 等 | HTTPS | 404 Not Found（路由不匹配） |

#### 3.6.3 非 GET 请求场景

**路由定义** (`router/router.go:162`):
```go
g.GET("version", func(ctx *gin.Context) {  // 只匹配 GET 方法
    ctx.JSON(200, vInfo)
})
```

**响应行为**（无 HTTPS 重定向时）：
- HEAD → 404 Not Found（与 `/health` 不同，`/version` 只注册了 GET）
- POST/PUT/DELETE/PATCH → 404 Not Found
- 只有 GET 方法能命中路由

> **与 `/health` 的差异**：`/health` 支持 GET 和 HEAD，`/version` 仅支持 GET

---

## 四、两类接口运维场景差异定位

| 维度 | 健康检查 (`/health`) | 版本信息 (`/version`) |
|------|---------------------|----------------------|
| **核心用途** | 探测服务**运行时可用性** | 标识服务**静态版本属性** |
| **调用频率** | 高频（负载均衡、K8s liveness/readiness 探针，通常 1-10s/次） | 低频（部署验证、问题排查、版本追溯） |
| **返回值变化** | 动态变化（随服务状态变化） | 静态不变（进程生命周期内恒定） |
| **失败影响** | 失败意味着服务不可用，触发告警/摘除 | 几乎不会失败（除非进程完全崩溃） |
| **依赖组件** | 数据库连接 | 无（编译时数据） |
| **性能开销** | 有（数据库 Ping 操作） | 极低（直接返回内存对象） |
| **支持方法** | GET + HEAD | 仅 GET |
| **适用场景** | - 负载均衡健康检查<br>- Kubernetes 存活/就绪探针<br>- 监控告警 | - 部署后版本验证<br>- 故障时版本追溯<br>- 自动化部署流水线校验 |
| **启动时机** | 数据库就绪后可用 | 数据库就绪后可用（相同） |

### 4.1 特殊场景响应行为对比

| 场景 | `/health` | `/version` |
|------|-----------|------------|
| **数据库初始化失败** | ❌ TCP 连接被拒绝（进程 panic） | ❌ TCP 连接被拒绝（进程 panic，完全相同） |
| **数据库运行时断开** | 500 {health: "orange", database: "red"} | 200 正常返回（无数据库依赖） |
| **HTTP GET + HTTPS 重定向开** | 302 重定向到 HTTPS | 302 重定向到 HTTPS（相同） |
| **HTTP POST + HTTPS 重定向开** | 400 "Use HTTPS" | 400 "Use HTTPS"（相同） |
| **HTTPS GET** | 200 健康状态 | 200 版本信息 |
| **HTTPS HEAD** | 200 空响应体 | 404 Not Found（差异！） |
| **HTTPS POST** | 404 Not Found | 404 Not Found（相同） |

> **关键发现**：
> 1. 数据库初始化失败时，两个接口**行为完全一致**——都不可达
> 2. 数据库运行时断开时，行为出现差异——`/health` 报错，`/version` 正常
> 3. HTTPS HEAD 请求行为不同——`/health` 返回 200，`/version` 返回 404

---

## 五、路由注册顺序与鉴权边界

```
router/router.go 路由注册顺序:

1. 全局中间件 (Logger, Recovery, ErrorHandler 等)
2. HTTPS 重定向中间件 (如果启用) ←── 作用于所有后续路由
3. /health ←────────── 公开，无鉴权，支持 GET+HEAD
4. /swagger
5. /image
6. /docs
7. JSON Header 中间件
8. CORS 中间件
9. /plugin (需要 RequireClient)
10. /user (部分公开)
11. /auth/local/login
12. /version ←──────── 公开，无鉴权，仅支持 GET
13. /gotifyinfo ←───── 公开，无鉴权
14. /message (需要 RequireApplicationToken)
15. ... 其他需要鉴权的接口
```

**关键观察**:
- `/health` 和 `/version` 均注册在**认证中间件链之前**
- 这是运维接口的标准设计：确保即使认证系统出现问题，运维人员仍能获取基础状态信息
- HTTPS 重定向中间件作用于两个接口

---

## 六、架构设计思考

### 6.1 健康检查的局限性

当前实现仅检查**数据库连接**，未覆盖：
- 插件系统状态
- 消息队列积压
- 流式连接（WebSocket）健康度
- 磁盘空间

### 6.2 版本信息的设计优点

- **编译时注入**：无需读取外部文件，性能最优
- **不可变性**：进程生命周期内不变化，可安全缓存
- **完整标识**：Version + Commit + BuildDate 三元组唯一标识构建产物

### 6.3 运维友好性设计

1. **无鉴权**：便于监控系统集成
2. **标准 HTTP 状态码**：200 = 健康，500 = 不健康
3. **结构化 JSON 响应**：便于自动化解析
4. **本地日志静默**：`127.0.0.1` 的健康检查请求不打日志，避免日志泛滥 (`router/router.go:243-244`)

---

## 七、运维最佳实践

### 7.1 启动探针配置建议

**不要使用 `/version` 作为启动探针**：
- ❌ 错误做法：用 `/version` 判断服务是否启动成功（数据库初始化失败时同样不可达）
- ✅ 正确做法：
  - Kubernetes `startupProbe` → 使用 `/health`（验证数据库就绪）
  - Kubernetes `livenessProbe` → 使用 `/health`
  - Kubernetes `readinessProbe` → 使用 `/health`

### 7.2 HTTPS 环境下的监控配置

开启 HTTPS 重定向时：
- 监控系统应直接访问 HTTPS 端口，避免 302 重定向开销
- 如果必须访问 HTTP，确保监控客户端支持自动跟随 302 重定向
- 健康检查使用 GET 方法（最稳妥），HEAD 方法对于 `/version` 会返回 404

### 7.3 故障排查流程

```
服务不可达 → 先检查端口是否监听
    │
    ├─ 端口未监听 → 进程未启动 → 查看启动日志（数据库连接失败？配置错误？）
    │
    └─ 端口监听 → 调用 /health
            │
            ├─ 200 → 服务正常，问题在其他地方（业务逻辑、网络等）
            ├─ 500 → 数据库连接问题，检查数据库状态
            └─ 其他 → 检查 HTTPS 配置、负载均衡、中间件
```

### 7.4 版本验证最佳实践

部署验证流程：
1. 等待服务启动完成（`/health` 返回 200）
2. 调用 `/version` 获取版本信息
3. 与预期版本对比，确认部署成功

> 不要跳过第一步直接调用 `/version`，因为数据库初始化失败时两个接口都不可达，无法区分"进程未启动"和"版本不匹配"。

### 7.5 HEAD 方法使用注意事项

- `/health` 支持 HEAD，可用于轻量级连通性检查（无响应体）
- `/version` 不支持 HEAD，会返回 404，切勿用 HEAD 检查版本接口
- 自动化脚本中统一使用 GET 方法最稳妥
