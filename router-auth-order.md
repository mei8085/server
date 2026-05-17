# Gotify Server 请求处理链分析

## 概述

本文档详细分析 Gotify Server 中 HTTP 请求的完整处理流程，包括路由装配顺序、中间件执行链、CORS 处理、认证机制和业务 handler 的先后关系。

## 1. 服务启动入口 (`app.go:26-54`)

请求首先通过 `net/http` 标准库的 `Server.Serve()` 进入 Gin 引擎。

```go
engine, closeable := router.Create(db, vInfo, conf)
// ...
s := &http.Server{Handler: router}
s.Serve(httpListener)  // 或 s.ServeTLS()
```

## 2. Gin 引擎初始化与全局中间件 (`router/router.go:27-66`)

### 2.1 引擎基础配置 (`router.go:28-40`)

```go
g := gin.New()
g.RemoveExtraSlash = true
g.RemoteIPHeaders = []string{"X-Forwarded-For"}
g.SetTrustedProxies(conf.Server.TrustedProxies)
g.ForwardedByClientIP = true
```

### 2.2 Socket 地址映射中间件 (`router.go:35-40`)

**位置**: 第一个中间件
**作用**: 将 Unix Socket 的 "@" 地址映射为 127.0.0.1

```go
g.Use(func(ctx *gin.Context) {
    if ctx.Request.RemoteAddr == "@" {
        ctx.Request.RemoteAddr = "127.0.0.1:65535"
    }
})
```

### 2.3 核心全局中间件链 (`router.go:42`)

**执行顺序**:
1. `gin.LoggerWithFormatter(logFormatter)` - 请求日志记录
2. `gin.Recovery()` - panic 恢复
3. `gerror.Handler()` - 错误处理 (`error/handler.go:15-41`)
4. `location.Default()` - 地理位置解析

```go
g.Use(gin.LoggerWithFormatter(logFormatter), gin.Recovery(), gerror.Handler(), location.Default())
```

**错误处理中间件特点** (`error/handler.go`):
- 在 `c.Next()` 后执行（后置中间件）
- 收集 `c.Errors` 中的所有错误
- 统一格式化为 JSON 错误响应

### 2.4 404 路由处理 (`router.go:43`)

```go
g.NoRoute(gerror.NotFound())
```

### 2.5 HTTPS 重定向中间件 (`router.go:45-66`)

**条件**: 仅当 `SSL.Enabled` 且 `SSL.RedirectToHTTPS` 为 true 时启用
**位置**: CORS 中间件之前

```go
g.Use(func(ctx *gin.Context) {
    if ctx.Request.TLS != nil {
        ctx.Next()  // 已 HTTPS，继续处理
        return
    }
    // 非 GET/HEAD 方法返回 400 Bad Request
    // GET/HEAD 方法重定向到 HTTPS
})
```

## 3. 早期无认证路由注册 (`router/router.go:107-124`)

**在 CORS 和认证中间件之前注册的路由**:

| 路径 | 方法 | 处理器 | 说明 |
|------|------|--------|------|
| `/ui/*` | 多种 | `ui.Register()` | UI 静态资源 |
| `/auth/oidc/*` | GET/POST | OIDC 相关 handler | OIDC 认证流程 |
| `/health` | GET/HEAD | `healthHandler.Health` | 健康检查 |
| `/swagger` | GET | `docs.Serve` | Swagger JSON |
| `/image/*` | GET | `StaticFS` | 图片资源 |
| `/docs` | GET | `docs.UI` | Swagger UI |

**重要**: 这些路由在 CORS 中间件之前注册，但仍会经过 CORS 处理（因为 CORS 是全局中间件）。

## 4. Content-Type 和响应头设置 (`router.go:125-130`)

**位置**: CORS 中间件之前
**作用**: 为后续 API 响应设置默认 Content-Type

```go
g.Use(func(ctx *gin.Context) {
    ctx.Header("Content-Type", "application/json")
    for header, value := range conf.Server.ResponseHeaders {
        ctx.Header(header, value)
    }
})
```

## 5. CORS 中间件 (`router.go:131`, `auth/cors.go`)

### 5.1 注册位置

```go
g.Use(cors.New(auth.CorsConfig(conf)))
```

**关键点**:
- 使用 `github.com/gin-contrib/cors` 库
- 是**全局中间件**，作用于所有在其之后注册的路由
- 对之前注册的路由（如 UI、健康检查）也会生效（因为 Gin 中间件是按添加顺序执行的）

### 5.2 CORS 配置 (`auth/cors.go:14-44`)

**开发模式** (`mode.IsDev()`):
- `AllowAllOrigins = true` - 允许所有源
- 允许方法: GET, POST, DELETE, OPTIONS, PUT
- 允许头: X-Gotify-Key, Authorization, Content-Type, Upgrade, Origin, Connection, Accept-Encoding, Accept-Language, Host

**生产模式**:
- 基于配置的正则匹配源 (`AllowOrigins`)
- 基于配置的允许方法和头
- `MaxAge = 12 * time.Hour`

### 5.3 gin-contrib/cors 预检请求处理

`gin-contrib/cors` 中间件内部处理流程：

1. **检测预检请求**: 检查请求方法是否为 OPTIONS 且包含 Origin 和 Access-Control-Request-Method 头
2. **验证 Origin**: 检查请求源是否在允许列表中
3. **验证 Method**: 检查请求方法是否被允许
4. **设置 CORS 响应头**:
   - Access-Control-Allow-Origin
   - Access-Control-Allow-Methods
   - Access-Control-Allow-Headers
   - Access-Control-Max-Age
5. **中止请求**: 对于预检请求，直接返回 204 No Content 并 `ctx.Abort()`

**注意**: 这意味着 `g.OPTIONS("/*any")` 注册的路由实际上**可能永远不会被执行**，因为 CORS 中间件已经处理并中止了预检请求。

## 6. OPTIONS 通配路由 (`router.go:149`)

```go
g.OPTIONS("/*any")
```

**作用**: 兜底处理所有未被其他路由匹配的 OPTIONS 请求
**实际情况**: 由于 gin-contrib/cors 已经在中间件层处理并中止了预检请求，此路由通常不会执行

## 7. 公开路由与可选认证 (`router.go:133-164`)

### 7.1 Plugin 路由 (`router.go:133-143`)

```go
g.GET("/plugin", authentication.RequireClient, pluginHandler.GetPlugins)
pluginRoute := g.Group("/plugin/", authentication.RequireClient)
```

### 7.2 用户注册路由 (`router.go:145`)

```go
g.Group("/user").Use(authentication.Optional).POST("", userHandler.CreateUser)
```

**`authentication.Optional` 特点** (`auth/authentication.go:78-82`):
- 尝试认证但不强制
- 认证失败不中止请求，继续执行
- 认证成功则设置用户上下文

### 7.3 登录路由 (`router.go:147`)

```go
g.POST("/auth/local/login", sessionHandler.Login)
```

**无认证**，用于获取会话 token

### 7.4 公开信息路由 (`router.go:162-179`)

```go
g.GET("version", ...)
g.GET("gotifyinfo", ...)
```

**无认证**，公开版本和服务信息

## 8. 应用消息推送路由 (`router.go:181`)

```go
g.Group("/").Use(authentication.RequireApplicationToken).POST("/message", messageHandler.CreateMessage)
```

**认证中间件**: `RequireApplicationToken`
- 仅接受应用 token（不接受用户/客户端 token）
- 用户认证即使成功也会返回 403 Forbidden

## 9. 客户端认证路由组 (`router.go:183-218`)

```go
clientAuth := g.Group("")
clientAuth.Use(authentication.RequireClient)
```

### 9.1 包含的路由

| 路径前缀 | 说明 |
|----------|------|
| `/application/*` | 应用管理 |
| `/client/*` | 客户端管理 |
| `/message/*` | 消息管理 |
| `/stream` | WebSocket 流 |
| `/current/user` | 当前用户信息 |
| `/auth/logout` | 登出 |

### 9.2 `RequireClient` 认证流程 (`auth/authentication.go:52-54`)

```go
func (a *Auth) RequireClient(ctx *gin.Context) {
    a.evaluateOr401(ctx, a.handleUser(), a.handleClient())
}
```

**认证尝试顺序**:
1. **用户认证** (`handleUser`): Basic Auth
2. **客户端认证** (`handleClient`): token 认证

**token 读取优先级** (`readTokenFromRequest`, `auth/authentication.go:205-216`):
1. Query 参数: `?token=`
2. Header: `X-Gotify-Key`
3. Header: `Authorization: Bearer `
4. Cookie: `gotify-client-token`

## 10. 提升权限客户端路由组 (`router.go:220-227`)

```go
clientElevated := g.Group("")
clientElevated.Use(authentication.RequireElevatedClient)
```

**路由**:
- `POST /client/:id/elevate` - 提升客户端
- `DELETE /application/:id` - 删除应用
- `DELETE /client/:id` - 删除客户端
- `POST /current/user/password` - 修改密码

**`RequireElevatedClient` 特点**:
- 要求 `client.ElevatedUntil` 在当前时间之后
- 否则返回 403 Forbidden: "session not elevated"

## 11. 管理员路由组 (`router.go:229-236`)

```go
authAdmin := g.Group("/user")
authAdmin.Use(authentication.RequireAdmin)
```

**路由**:
- `GET /user` - 获取所有用户
- `DELETE /user/:id` - 删除用户
- `GET /user/:id` - 获取单个用户
- `POST /user/:id` - 更新用户

**`RequireAdmin` 特点**:
- 同时支持用户认证（Basic Auth）和客户端认证
- 要求用户具有 `Admin = true` 属性
- 客户端认证时会查询关联用户检查 admin 权限

## 12. 完整请求处理流程图

```
HTTP 请求到达
    │
    ▼
net/http Server.Serve()
    │
    ▼
┌─────────────────────────────────────────┐
│            Gin 引擎处理                  │
├─────────────────────────────────────────┤
│ 1. Socket 地址映射 (@ → 127.0.0.1)       │
│ 2. gin.LoggerWithFormatter               │
│ 3. gin.Recovery()                        │
│ 4. gerror.Handler() [等待 Next() 后执行]  │
│ 5. location.Default()                    │
│ 6. HTTPS 重定向 (如配置)                 │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│         路由匹配 (按注册顺序)            │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│    Content-Type + 自定义响应头设置       │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│         CORS 中间件处理                  │
│  ┌───────────────────────────────────┐  │
│  │ 是 OPTIONS 预检请求?              │  │
│  │     ├── 是 → 验证 Origin/Method   │  │
│  │     │     ├── 合法 → 设置CORS头    │  │
│  │     │     │        返回 204 → 中止 │  │
│  │     │     └── 非法 → 返回 403     │  │
│  │     │              → 中止          │  │
│  │     └── 否 → 设置 CORS 头          │  │
│  │           继续执行 → ctx.Next()    │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
    │ (非预检请求继续)
    ▼
┌─────────────────────────────────────────┐
│         路由组特定中间件                 │
│  ┌───────────────────────────────────┐  │
│  │ 可选认证 Optional                  │  │
│  │ 应用认证 RequireApplicationToken   │  │
│  │ 客户端认证 RequireClient           │  │
│  │ 提升权限认证 RequireElevatedClient │  │
│  │ 管理员认证 RequireAdmin             │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
    │ (认证通过)
    ▼
┌─────────────────────────────────────────┐
│         业务 Handler 执行                │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│    中间件回溯 (按逆序执行后置逻辑)       │
│  - gerror.Handler() 处理收集的错误       │
│  - gin.LoggerWithFormatter 记录日志      │
└─────────────────────────────────────────┘
    │
    ▼
响应返回给客户端
```

## 13. 关键注意事项

### 13.1 中间件执行顺序

Gin 中间件是**按添加顺序**执行的，路由匹配成功后，该路由路径上所有中间件按注册顺序执行。

### 13.2 CORS 与 OPTIONS 路由

由于 `gin-contrib/cors` 在中间件层就处理了 OPTIONS 预检请求并调用 `ctx.Abort()`，因此 `g.OPTIONS("/*any")` 注册的路由处理器**通常不会被执行**。

### 13.3 错误处理时机

`gerror.Handler()` 是一个"后置中间件"，它在 `c.Next()` 后执行，意味着：
- 先执行业务 handler 和其他中间件
- handler 执行完后，错误处理中间件才收集并处理 `c.Errors` 中的错误

### 13.4 认证优先级

1. 对于 `/message` POST：**仅**应用 token 有效，用户 token 无效
2. 对于其他 API：用户认证（Basic Auth）优先于客户端 token 认证
3. token 来源优先级：Query > X-Gotify-Key Header > Bearer Header > Cookie

### 13.5 无认证路由

以下路由**完全不需要认证**：
- `/health` - 健康检查
- `/version` - 版本信息
- `/gotifyinfo` - 服务信息
- `/auth/local/login` - 登录
- `/user` POST - 用户注册（启用注册时）
- `/swagger`, `/docs` - API 文档
- `/image/*` - 图片资源
- UI 静态资源
- OIDC 相关回调路由

## 14. 文件位置参考

| 功能 | 文件 |
|------|------|
| 路由配置 | `router/router.go` |
| 认证中间件 | `auth/authentication.go` |
| CORS 配置 | `auth/cors.go` |
| 错误处理 | `error/handler.go` |
| 服务启动 | `app.go`, `runner/runner.go` |
