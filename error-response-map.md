# 消息推送服务错误响应模型分析报告

## 1. 错误模型定义

### 1.1 标准错误模型

标准错误响应模型定义在 `model/error.go:8-24`，绝大多数 API 错误使用此结构：

```go
type Error struct {
    Error            string `json:"error"`              // 通用错误消息（如 "Unauthorized"）
    ErrorCode        int    `json:"errorCode"`          // HTTP 状态码
    ErrorDescription string `json:"errorDescription"`   // 详细错误描述
}
```

### 1.2 字段说明

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `error` | string | HTTP 状态文本，与 `errorCode` 对应 | `"Unauthorized"` |
| `errorCode` | int | HTTP 状态码 | `401` |
| `errorDescription` | string | 具体错误详情，面向开发者 | `"you need to provide a valid access token..."` |

### 1.3 设计特点

- **与 HTTP 语义绑定**：`error` 和 `errorCode` 严格对应 HTTP 状态码文本和数值
- **两层错误信息**：通用层（`error`）+ 详情层（`errorDescription`）
- **无业务错误码**：仅有 HTTP 状态码，无额外业务细分错误码

---

## 2. 调用链路与错误传递机制

### 2.1 链路分类

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    调用链路与错误传递分类                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  🔗 链路 A：Axios API 调用链路（95% 业务接口）                            │
│  ─────────────────────────────────────────────────────────────────      │
│  调用方式：axios.request() / axios.get() / axios.post()                  │
│  适用接口：/message, /application, /client, /user, /plugin 等           │
│  错误传递：响应 → axios 拦截器 → 解析 Error JSON → 显示给用户             │
│  错误格式：✅ 标准 model.Error JSON                                      │
│                                                                         │
│  🔗 链路 B：浏览器跳转/弹窗链路（OIDC 浏览器流程）                         │
│  ─────────────────────────────────────────────────────────────────      │
│  调用方式：<a href="..."> 页面跳转 / window.open() 打开弹窗              │
│  适用接口：/auth/oidc/login, /auth/oidc/callback                        │
│  错误传递：错误直接渲染在浏览器页面（跳转）或弹窗中（window.open）        │
│  错误格式：❌ text/plain 纯文本（http.Error()）                          │
│                                                                         │
│  🔗 链路 C：特殊接口（健康检查）                                          │
│  ─────────────────────────────────────────────────────────────────      │
│  调用方式：可通过 axios 或直接浏览器访问                                  │
│  适用接口：/health                                                       │
│  错误传递：如果 axios 调用 → 拦截器尝试解析 Error 但失败                  │
│  错误格式：❌ model.Health JSON（即使 500）                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件

#### 2.2.1 错误抛出方式

**方式一：`successOrAbort` 工具函数** (`api/errorHandling.go:5-10`)

```go
func successOrAbort(ctx *gin.Context, code int, err error) (success bool) {
    if err != nil {
        ctx.AbortWithError(code, err)
    }
    return err == nil
}
```

- 主要用于数据库操作等通用错误
- 仅适用于链路 A（Axios API）

**方式二：直接调用 `ctx.AbortWithError()`**

```go
ctx.AbortWithError(404, errors.New("application does not exist"))
```

- 用于业务逻辑错误，如资源不存在、权限不足等
- 会被错误中间件捕获并转为标准 Error 模型
- 适用于链路 A（Axios API）和 OIDC 提升入口

**方式三：参数绑定错误**

- 通过 `ctx.Bind()` / `ctx.MustBindWith()` 触发
- 由 Gin 自动添加到错误队列，类型为 `gin.ErrorTypeBind`
- 适用于链路 A（Axios API）和 OIDC 提升入口

**方式四：`http.Error()` 纯文本响应**

```go
http.Error(w, "invalid client name", http.StatusBadRequest)
```

- 用于 OIDC 浏览器流程的回调阶段
- 返回 `Content-Type: text/plain; charset=utf-8`
- 绕过 Gin 错误队列和中间件
- 适用于链路 B（浏览器跳转/弹窗的回调阶段）

**方式五：直接 `ctx.JSON()` 响应**

```go
ctx.JSON(500, model.Health{Health: "orange", Database: "red"})
```

- 用于健康检查接口
- 即使是错误状态码也返回业务模型
- 绕过错误中间件
- 适用于链路 C（健康检查）

#### 2.2.2 错误捕获中间件

定义在 `error/handler.go:14-64`，**对链路 A 和 OIDC 提升入口生效**：

```go
func Handler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next() // 执行业务逻辑
        
        if len(c.Errors) > 0 {
            for _, e := range c.Errors {
                switch e.Type {
                case gin.ErrorTypeBind:
                    // 参数校验错误，转换为友好文本
                    errs, _ := e.Err.(validator.ValidationErrors)
                    var stringErrors []string
                    for _, err := range errs {
                        stringErrors = append(stringErrors, validationErrorToText(err))
                    }
                    writeError(c, strings.Join(stringErrors, "; "))
                default:
                    // 其他错误直接使用错误信息
                    writeError(c, e.Err.Error())
                }
            }
        }
    }
}
```

#### 2.2.3 错误响应写入

```go
func writeError(ctx *gin.Context, errString string) {
    status := http.StatusBadRequest
    if ctx.Writer.Status() != http.StatusOK {
        status = ctx.Writer.Status()  // 使用 AbortWithError 设置的状态码
    }
    ctx.JSON(status, &model.Error{
        Error:            http.StatusText(status),
        ErrorCode:        status,
        ErrorDescription: errString,
    })
}
```

### 2.3 中间件注册位置

在 `router/router.go:42` 注册：

```go
g.Use(gin.LoggerWithFormatter(logFormatter), gin.Recovery(), gerror.Handler(), location.Default())
```

**关键注意**：
- 错误中间件在所有路由之前注册
- 但 OIDC 回调使用 `gin.WrapF()` 包装 `http.HandlerFunc`，绕过了 Gin 的错误机制
- 健康检查直接写入响应，不调用 `AbortWithError`

---

## 3. 控制器抛错分析

### 3.1 消息接口 (MessageAPI) - 链路 A

| 接口 | 错误场景 | 状态码 | 错误信息 | 响应模型 |
|------|----------|--------|----------|----------|
| `GET /message` | 数据库查询失败 | 500 | 数据库错误原文 | ✅ Error |
| `GET /application/{id}/message` | 应用不存在 | 404 | `"application does not exist"` | ✅ Error |
| `DELETE /message/{id}` | 消息不存在 | 404 | `"message does not exist"` | ✅ Error |
| `POST /message` | 数据库插入失败 | 500 | 数据库错误原文 | ✅ Error |
| 参数绑定失败 | 400 | 校验错误详情 | ✅ Error |

### 3.2 应用接口 (ApplicationAPI) - 链路 A

| 接口 | 错误场景 | 状态码 | 错误信息 | 响应模型 |
|------|----------|--------|----------|----------|
| `POST /application` | 排序键重复 | 400 | `"sort key is not unique"` | ✅ Error |
| `DELETE /application/{id}` | 删除内部应用 | 400 | `"cannot delete internal application"` | ✅ Error |
| `DELETE /application/{id}` | 应用不存在 | 404 | `"app with id %d doesn't exists"` | ✅ Error |
| `POST /{id}/image` | 缺少文件 | 400 | `"file with key 'file' must be present"` | ✅ Error |
| `POST /{id}/image` | 非图片文件 | 400 | `"file must be an image"` | ✅ Error |
| `POST /{id}/image` | 无效扩展名 | 400 | `"invalid file extension"` | ✅ Error |
| `DELETE /{id}/image` | 无自定义图片 | 400 | `"app with id %d does not have a customized image"` | ✅ Error |

### 3.3 客户端接口 (ClientAPI) - 链路 A

| 接口 | 错误场景 | 状态码 | 错误信息 | 响应模型 |
|------|----------|--------|----------|----------|
| `PUT /client/{id}` | 客户端不存在 | 404 | `"client with id %d doesn't exists"` | ✅ Error |
| `DELETE /client/{id}` | 客户端不存在 | 404 | `"client with id %d doesn't exists"` | ✅ Error |
| `POST /{id}/elevate` | 客户端不存在 | 404 | `"client not found"` | ✅ Error |

### 3.4 用户接口 (UserAPI) - 链路 A

| 接口 | 错误场景 | 状态码 | 错误信息 | 响应模型 |
|------|----------|--------|----------|----------|
| `POST /user` | 用户名已存在 | 400 | `"username already exists"` | ✅ Error |
| `POST /user` | 注册关闭时非管理员创建 | 401/403 | `"you are not allowed to access this api"` | ✅ Error |
| `POST /user` | 非管理员创建管理员用户 | 401/403 | `"you are not allowed to create an admin user"` | ✅ Error |
| `GET /user/{id}` | 用户不存在 | 404 | `"user does not exist"` | ✅ Error |
| `DELETE /user/{id}` | 删除最后一个管理员 | 400 | `"cannot delete last admin"` | ✅ Error |
| `POST /user/{id}` | 降级最后一个管理员 | 400 | `"cannot delete last admin"` | ✅ Error |

### 3.5 认证中间件 (auth/authentication.go) - 链路 A

| 错误场景 | 状态码 | 错误信息 | 响应模型 |
|----------|--------|----------|----------|
| 未提供有效认证 | 401 | `"you need to provide a valid access token or user credentials to access this api"` | ✅ Error |
| 权限不足 | 403 | `"you are not allowed to access this api"` | ✅ Error |
| 会话未提升 | 403 | `"session not elevated, use basic auth or call /client:elevate"` | ✅ Error |

### 3.6 会话接口 (SessionAPI) - 链路 A

| 接口 | 错误场景 | 状态码 | 错误信息 | 响应模型 |
|------|----------|--------|----------|----------|
| `POST /auth/local/login` | 缺少 Basic Auth | 401 | `"basic auth required"` | ✅ Error |
| `POST /auth/local/login` | 凭证无效 | 401 | `"invalid credentials"` | ✅ Error |
| `POST /auth/logout` | 无客户端认证 | 403 | `"no client auth provided"` | ✅ Error |

### 3.7 插件接口 (PluginAPI) - 链路 A

| 接口 | 错误场景 | 状态码 | 错误信息 | 响应模型 |
|------|----------|--------|----------|----------|
| 插件不存在/无权限 | 404 | `"unknown plugin"` | ✅ Error |
| 插件实例不存在 | 404 | `"plugin instance not found"` | ✅ Error |
| 重复启用/禁用 | 400 | 插件错误原文 | ✅ Error |
| 插件不支持能力 | 400 | `"plugin does not support %s"` | ✅ Error |
| YAML 配置解析失败 | 400 | 解析错误原文 | ✅ Error |
| 配置验证失败 | 400 | 验证错误原文 | ✅ Error |

### 3.8 健康检查接口 (HealthAPI) - 链路 C ⚠️

| 接口 | 错误场景 | 状态码 | 响应内容 | 响应模型 |
|------|----------|--------|----------|----------|
| `GET /health` | 数据库连接失败 | 500 | `{"health": "orange", "database": "red"}` | ❌ Health 模型 |
| `GET /health` | 数据库正常 | 200 | `{"health": "green", "database": "green"}` | ✅ Health 模型 |

**关键差异**：
- 健康检查接口**从不返回 Error 模型**
- 即使数据库失败（500 状态码），仍返回 `model.Health` 结构
- 直接使用 `ctx.JSON()` 写入响应，绕过错误中间件

### 3.9 OIDC 接口

#### 3.9.1 OIDC 外部接口（Native App）- 链路 A ✅

| 接口 | 错误场景 | 状态码 | 错误信息 | 响应模型 |
|------|----------|--------|----------|----------|
| `POST /auth/oidc/external/authorize` | 参数绑定失败 | 400 | 绑定错误详情 | ✅ Error |
| `POST /auth/oidc/external/authorize` | 状态生成失败 | 500 | 错误原文 | ✅ Error |
| `POST /auth/oidc/external/token` | 参数绑定失败 | 400 | 绑定错误详情 | ✅ Error |
| `POST /auth/oidc/external/token` | state 无效/过期 | 400 | `"unknown or expired state"` | ✅ Error |
| `POST /auth/oidc/external/token` | token 交换失败 | 401 | `"token exchange failed: ..."` | ✅ Error |
| `POST /auth/oidc/external/token` | 用户信息获取失败 | 500 | `"failed to get user info: ..."` | ✅ Error |
| `POST /auth/oidc/external/token` | 用户不存在/自动注册关闭 | 403 | `"user does not exist and auto-registration is disabled"` | ✅ Error |

#### 3.9.2 OIDC 提升流程（两段链路拆分）

**阶段一：提升入口 `/auth/oidc/elevate` - 链路 A ✅**

`api/oidc.go:160-172` 是**原生 Gin handler**，不是 `gin.WrapF` 包装的：

```go
func (a *OIDCAPI) ElevateHandler(ctx *gin.Context) {
    var elevate pendingElevation
    if err := ctx.BindQuery(&elevate); err != nil {
        return  // Gin 自动处理，转为 Error 模型
    }
    state, err := a.generateState()
    if err != nil {
        ctx.AbortWithError(http.StatusInternalServerError, err)  // 标准 Error 模型
        return
    }
    // ... 重定向到 OIDC provider
}
```

| 错误场景 | 状态码 | 错误信息 | 响应模型 |
|----------|--------|----------|----------|
| 参数绑定失败（缺少 `id` 或 `durationSeconds`） | 400 | 校验错误详情（如 "Field 'id' is required"） | ✅ Error |
| 状态生成失败 | 500 | 错误原文 | ✅ Error |

**✅ 阶段一结论**：提升入口的错误返回标准 `model.Error` JSON，经过错误中间件处理。

---

**阶段二：提升回调 `/auth/oidc/callback` - 链路 B ⚠️**

`api/oidc.go:202-233` 使用 `gin.WrapF` 包装，所有错误使用 `http.Error()` 返回纯文本：

```go
func (a *OIDCAPI) CallbackHandler() gin.HandlerFunc {
    callback := func(w http.ResponseWriter, r *http.Request, ...) {
        // 用户解析失败
        user, status, err := a.resolveUser(info)
        if err != nil {
            http.Error(w, err.Error(), status)  // 纯文本
            return
        }
        // state 无效/过期
        session, ok := a.popPendingSession(state)
        if !ok {
            http.Error(w, "unknown or expired state", http.StatusBadRequest)  // 纯文本
            return
        }
        // 提升回调
        if session.Elevate != nil {
            a.handleElevationCallback(w, session.Elevate, user)  // 纯文本错误
            return
        }
        // ...
    }
    return gin.WrapF(rp.CodeExchangeHandler(rp.UserinfoCallback(callback), a.Provider))
}
```

`handleElevationCallback` (`api/oidc.go:235-266`) 中的错误：

| 错误场景 | 状态码 | 响应内容 | 响应模型 |
|----------|--------|----------|----------|
| 数据库查询错误 | 500 | `"database error: ..."`（纯文本） | ❌ text/plain |
| 客户端不存在或不属于当前用户 | 404 | `"client not found"`（纯文本） | ❌ text/plain |
| 提升会话更新失败 | 500 | `"failed to elevate session: ..."`（纯文本） | ❌ text/plain |

**⚠️ 阶段二结论**：提升回调的所有错误均为 `text/plain` 纯文本，绕过错误中间件。

#### 3.9.3 OIDC 登录流程 - 链路 B ⚠️

**阶段一：登录入口 `/auth/oidc/login` - 链路 B ⚠️**

`api/oidc.go:115-130` 使用 `gin.WrapF` 包装：

| 错误场景 | 状态码 | 响应内容 | 响应模型 |
|----------|--------|----------|----------|
| 缺少 client name | 400 | `"invalid client name"`（纯文本） | ❌ text/plain |
| 状态生成失败 | 500 | `"failed to generate state: ..."`（纯文本） | ❌ text/plain |

**阶段二：登录回调 `/auth/oidc/callback` - 链路 B ⚠️**

与提升回调共用同一个 handler，错误也为纯文本：

| 错误场景 | 状态码 | 响应内容 | 响应模型 |
|----------|--------|----------|----------|
| 用户解析失败 | 403/500 | 错误详情（纯文本） | ❌ text/plain |
| state 无效/过期 | 400 | `"unknown or expired state"`（纯文本） | ❌ text/plain |
| 客户端创建失败 | 500 | `"failed to create client: ..."`（纯文本） | ❌ text/plain |

#### 3.9.4 OIDC 各接口错误格式汇总

| 接口 | 阶段 | 错误格式 | 经过中间件 |
|------|------|----------|-----------|
| `GET /auth/oidc/login` | 入口 | ❌ text/plain | 否 |
| `GET /auth/oidc/callback`（登录） | 回调 | ❌ text/plain | 否 |
| `GET /auth/oidc/elevate` | 入口 | ✅ model.Error JSON | 是 |
| `GET /auth/oidc/callback`（提升） | 回调 | ❌ text/plain | 否 |
| `POST /auth/oidc/external/authorize` | 全部 | ✅ model.Error JSON | 是 |
| `POST /auth/oidc/external/token` | 全部 | ✅ model.Error JSON | 是 |

**关键修正**：OIDC 提升流程**不是全程 text/plain**，入口阶段（`/auth/oidc/elevate`）返回标准 Error JSON，只有回调阶段（`/auth/oidc/callback`）才返回纯文本。

### 3.10 通用工具错误 - 链路 A

| 场景 | 状态码 | 错误信息 | 响应模型 |
|------|--------|----------|----------|
| ID 路径参数解析失败 | 400 | `"invalid id"` | ✅ Error |
| 路由不匹配 | 404 | `"page not found"` | ✅ Error |

---

## 4. 错误统一性分析

### 4.1 统一程度评估

| 维度 | 统一程度 | 说明 |
|------|----------|------|
| **响应结构（链路 A）** | ✅ 完全统一 | 所有 Axios API 和 OIDC 提升入口错误均使用 `model.Error` 结构 |
| **响应结构（整体）** | ⚠️ 大部分统一 | OIDC 登录入口和所有回调阶段返回纯文本，健康检查返回 Health 模型 |
| **HTTP 状态码** | ✅ 基本统一 | 同类错误使用相同状态码（404 资源不存在、400 参数错误等） |
| **错误描述风格** | ⚠️ 部分不统一 | 相同语义的错误信息有多种表述 |
| **调用链路** | ❌ 不统一 | 三种不同的调用和错误传递机制 |

### 4.2 三类错误响应路径

```
┌─────────────────────────────────────────────────────────────┐
│                     错误响应路径分类                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  路径 A：标准路径（链路 A，95% API + OIDC 提升入口）          │
│  ───────────────────────────────────────────────────────    │
│  控制器 → ctx.AbortWithError() → Gin 错误队列 → 中间件       │
│         → writeError() → model.Error JSON                   │
│                                                             │
│  路径 B：健康检查（链路 C）                                   │
│  ───────────────────────────────────────────────────────    │
│  控制器 → ctx.JSON() → model.Health JSON（即使 500）        │
│         （绕过错误中间件）                                   │
│                                                             │
│  路径 C：OIDC 浏览器流程（链路 B 的回调阶段）                 │
│  ───────────────────────────────────────────────────────    │
│  控制器 → http.Error() → text/plain 纯文本                  │
│         （绕过 Gin 错误队列和中间件）                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 存在的不一致问题

#### 4.3.1 调用链路与错误传递不统一（最根本）

| 链路 | 调用方式 | 错误格式 | 经过 axios 拦截器 |
|------|----------|----------|-----------------|
| 链路 A（Axios API） | `axios.get()` / `axios.post()` | `model.Error` JSON | ✅ 是 |
| 链路 A（OIDC 提升入口） | `window.open()` 弹窗加载 | `model.Error` JSON | ❌ 否，显示在弹窗中 |
| 链路 B（OIDC 登录入口） | `<a href="...">` 页面跳转 | `text/plain` | ❌ 否，直接显示在页面 |
| 链路 B（OIDC 回调） | 浏览器/弹窗跳转 | `text/plain` | ❌ 否，显示在浏览器/弹窗中 |
| 链路 C（健康检查） | 可通过 axios 或直接访问 | `model.Health` JSON | ⚠️ 如果 axios 调用则经过，但格式不匹配 |

#### 4.3.2 响应模型不统一

| 接口类别 | 错误模型 | Content-Type |
|----------|----------|-------------|
| 消息/应用/客户端/用户/插件/会话 API | ✅ `model.Error` | `application/json` |
| OIDC 提升入口 | ✅ `model.Error` | `application/json` |
| OIDC 外部 API | ✅ `model.Error` | `application/json` |
| OIDC 登录入口 | ❌ 纯文本 | `text/plain` |
| OIDC 回调（登录+提升） | ❌ 纯文本 | `text/plain` |
| 健康检查 API | ❌ `model.Health` | `application/json` |

#### 4.3.3 资源不存在的错误信息不统一

| 接口 | 404 错误信息 |
|------|-------------|
| 应用 | `"application does not exist"` / `"app with id %d doesn't exists"` |
| 客户端 | `"client with id %d doesn't exists"` / `"client not found"` |
| 消息 | `"message does not exist"` |
| 用户 | `"user does not exist"` |
| 插件 | `"unknown plugin"` / `"plugin instance not found"` |
| OIDC 提升回调 | `"client not found"`（纯文本） |

**问题**：
- 动词时态不一致：`exist` vs `exists`
- 格式不统一：有的带 ID，有的不带
- 表述差异：`does not exist` vs `doesn't exists` vs `not found` vs `unknown`

#### 4.3.4 相同语义不同表述

| 语义 | 多种表述 |
|------|---------|
| 资源不存在 | 至少 6 种不同表述 |
| 权限不足 | `"you are not allowed to access this api"` / `"you are not allowed to create an admin user"` |

#### 4.3.5 数据库错误直接暴露

- 500 错误直接返回数据库错误原文，可能暴露内部实现细节
- 缺少对敏感错误信息的包装

### 4.4 参数校验错误

参数校验错误通过 `validator` 库进行，错误信息转换规则在 `error/handler.go:43-56`：

| 校验规则 | 错误信息格式 |
|----------|-------------|
| `required` | `"Field '%s' is required"` |
| `max` | `"Field '%s' must be less or equal to %s"` |
| `min` | `"Field '%s' must be more or equal to %s"` |
| 其他 | `"Field '%s' is not valid"` |

- ✅ 统一的格式化输出
- ✅ 字段名自动转为小写开头（camelCase）
- ✅ 适用于所有使用 `ctx.Bind` 的接口，包括 OIDC 提升入口

---

## 5. 对客户端 UI 的影响（深度校正）

### 5.1 前端调用方式汇总

| 功能 | 调用方式 | 链路类型 | 错误格式 |
|------|----------|----------|----------|
| 获取消息列表 | `axios.get('/message')` | 链路 A | ✅ Error JSON |
| 创建应用 | `axios.post('/application')` | 链路 A | ✅ Error JSON |
| 删除客户端 | `axios.delete('/client/{id}')` | 链路 A | ✅ Error JSON |
| 本地登录 | `axios.post('/auth/local/login')` | 链路 A | ✅ Error JSON |
| 本地提升权限 | `axios.post('/client/{id}/elevate')` | 链路 A | ✅ Error JSON |
| **OIDC 登录** | **`<a href="/auth/oidc/login">` 页面跳转** | **链路 B** | **❌ 纯文本** |
| **OIDC 提升权限（入口）** | **`window.open('/auth/oidc/elevate')` 弹窗** | **链路 A** | **✅ Error JSON（显示在弹窗）** |
| **OIDC 提升权限（回调）** | **弹窗内跳转** | **链路 B** | **❌ 纯文本（显示在弹窗）** |
| 健康检查 | 未在 UI 中直接调用 | 链路 C | ❌ Health JSON |

### 5.2 Axios 拦截器处理逻辑（链路 A）

在 `ui/src/apiAuth.ts:6-23` 中定义：

```typescript
axios.interceptors.response.use(undefined, (error) => {
    if (!error.response) {
        snack('Gotify server is not reachable, try refreshing the page.');
        return Promise.reject(error);
    }

    const status = error.response.status;

    if (status === 401) {
        currentUser.tryAuthenticate().then(() => snack('Could not complete request.'));
    }

    if (status === 400 || status === 403 || status === 500) {
        snack(error.response.data.error + ': ' + error.response.data.errorDescription);
    }

    return Promise.reject(error);
});
```

**适用范围**：仅对通过 axios 发起的请求生效。**不适用**于通过 `<a href>` 跳转或 `window.open()` 打开的页面。

### 5.3 各链路错误在 UI 中的实际表现

#### 5.3.1 链路 A（Axios API）- ✅ 正常

| 状态码 | UI 表现 |
|--------|---------|
| 无响应（网络错误） | 显示 "Gotify server is not reachable, try refreshing the page." |
| 401 | 尝试重新认证，显示 "Could not complete request." |
| 400 / 403 / 500 | 显示拼接的错误信息：`error + ': ' + errorDescription` |

**示例**：
```
Bad Request: Field 'name' is required
```

#### 5.3.2 OIDC 登录流程 - 🔴 纯文本错误页

**OIDC 登录流程**（`ui/src/user/Login.tsx:84-100`）：
```tsx
<Button
    component="a"
    href={config.get('url') + 'auth/oidc/login?name=' + encodeURIComponent(clientName)}
    ...
>
    Login with OIDC
</Button>
```

- 点击按钮后，**整页跳转**到 `/auth/oidc/login`
- 如果 `/auth/oidc/login` 出错（如缺少 name 参数）：
  - 浏览器显示 Go 默认错误页面（白底黑字的纯文本）
  - **不会经过 axios 拦截器**
  - 用户看到的是：
    ```
    invalid client name
    ```
- 如果 OIDC 回调 `/auth/oidc/callback` 出错：
  - 同样显示纯文本错误页面
  - 用户无法回到应用，只能手动返回

#### 5.3.3 OIDC 提升流程 - 🟡 分阶段处理

**OIDC 提升流程**（`ui/src/ElevateStore.ts:47-101`）：

```typescript
public oidcElevate = (durationSeconds: number): void => {
    const url = config.get('url') + 'auth/oidc/elevate?id=' + ...;
    this.oidcPopup = window.open(url, 'gotify-oidc-elevate', 'width=600,height=700');
    // ...
    this.oidcPollIntervalId = window.setInterval(this.checkOidcPopup, 500);
};

private checkOidcPopup = async () => {
    if (this.oidcPopup && !this.oidcPopup.closed) {
        return; // 等待弹窗关闭
    }
    // 弹窗已关闭，检查是否提升成功
    try {
        await this.currentUser.tryAuthenticate();
    } catch {
        // errors handled in tryAuthenticate
    }
    if (!this.elevated) {
        this.snack('OIDC elevation was not completed.');
    }
    this.cleanupOidcElevate();
};
```

**阶段一：提升入口错误（`/auth/oidc/elevate`）**

- 用户点击 "Elevate via OIDC" 按钮，`window.open()` 打开弹窗
- 如果入口参数错误或状态生成失败：
  - **弹窗内显示标准 Error JSON**（因为入口是原生 Gin handler）
  - 弹窗内容示例：
    ```json
    {"error":"Bad Request","errorCode":400,"errorDescription":"Field 'id' is required"}
    ```
  - 这是原始 JSON，没有格式化，用户体验差
  - **当前实现中主窗口未尝试读取弹窗内容**

**阶段二：提升回调错误（`/auth/oidc/callback`）**

- 用户在 OIDC provider 认证后，弹窗跳转到回调接口
- 如果回调出错：
  - **弹窗内显示纯文本错误**
  - 示例：
    ```
    client not found
    ```
  - **当前实现中主窗口未尝试读取弹窗内容**

**阶段三：弹窗关闭后的主窗口处理**

- 主窗口轮询检查弹窗是否关闭
- **当前实现**：无论弹窗中发生了什么错误，主窗口都不会尝试从弹窗读取错误信息
- 如果弹窗关闭但未提升成功，主窗口只显示通用错误：
  ```
  OIDC elevation was not completed.
  ```

**📌 技术可行性校正**：
- ❌ **不是技术上不可行**：同源弹窗的 DOM 和 `window` 对象在技术上是可以访问的（`window.opener` 可用于通信）
- ⚠️ **当前未实现回传机制**：当前代码没有建立任何通信机制（如 `postMessage`、轮询弹窗 URL、或注入脚本主动上报）
- ⚠️ **存在现实约束**：
  - OIDC provider 页面是跨域的，无法访问其内容
  - 回调成功时会自动执行 `window.close()`，主窗口可能来不及读取
  - 弹窗错误页面是后端直接返回的静态内容，没有与主窗口通信的脚本

**✅ 校正结论**：OIDC 提升流程不是全程纯文本。入口错误显示 JSON（用户体验差），回调错误显示纯文本。当前实现未建立回传机制，但技术上（同源场景下）存在改进空间。

#### 5.3.4 链路 C（健康检查）- ⚠️ 格式不匹配

健康检查未在 UI 中直接调用，但如果通过 axios 调用：

```typescript
// 假设 UI 调用健康检查
axios.get('/health')
```

- 数据库正常（200）：✅ 正常解析 `model.Health`
- 数据库失败（500）：❌ 拦截器尝试访问 `data.error` 和 `data.errorDescription`
  - 实际响应：`{"health": "orange", "database": "red"}`
  - 显示：`undefined: undefined`

### 5.4 UI 类型定义

在 `ui/src/types.ts` 中**没有**定义 Error 接口类型，UI 直接访问 `error.response.data.error` 和 `error.response.data.errorDescription`。

### 5.5 影响评估（校正后）

| 影响点 | 说明 | 严重程度 |
|--------|------|----------|
| **OIDC 登录错误体验** | 出错时显示浏览器默认纯文本页面，用户体验差，无法返回应用 | 🔴 高 |
| **OIDC 提升入口错误** | 参数错误时弹窗内显示原始 JSON，用户无法理解 | 🔴 高 |
| **OIDC 提升回调错误** | 弹窗内显示纯文本错误，用户体验差 | 🟡 中 |
| **OIDC 提升错误不透明** | 当前实现中主窗口不读取弹窗错误，只能显示通用提示 | 🟡 中 |
| **健壮性** | 多种错误处理路径，增加了维护复杂度 | 🟡 中 |
| **健康检查兼容性** | 如果 UI 将来调用健康检查，500 时会显示 `undefined: undefined` | 🟡 中 |
| **国际化** | 硬编码的英文错误信息难以进行多语言支持 | 🟡 中 |
| **调试友好** | `errorDescription` 详细信息有助于开发调试（链路 A） | ✅ 好 |

---

## 6. 对第三方接入方的影响（校正后）

### 6.1 API 文档

Swagger 文档中每个接口都声明了可能的错误响应，例如：

```yaml
responses:
  400:
    description: Bad Request
    schema:
      $ref: "#/definitions/Error"
  401:
    description: Unauthorized
    schema:
      $ref: "#/definitions/Error"
```

**文档与实际不符**：
- 健康检查接口文档声明 500 返回 `Health` 模型（文档正确）
- OIDC 登录入口文档声明错误返回 `Error` 模型，但实际返回纯文本（文档错误）
- OIDC 提升入口文档声明错误返回 `Error` 模型，**实际也返回 Error 模型**（文档正确 ✅）
- OIDC 回调文档声明错误返回 `Error` 模型，但实际返回纯文本（文档错误）

### 6.2 接入方分类与影响

#### 6.2.1 原生应用（Native App）- 使用 OIDC 外部接口

使用 `POST /auth/oidc/external/authorize` 和 `POST /auth/oidc/external/token`：
- ✅ 所有错误均返回标准 `model.Error` JSON
- ✅ 可通过 `errorCode` 和 `errorDescription` 精确处理
- ✅ 文档与实现一致

#### 6.2.2 Web 应用（Browser）- 使用 OIDC 浏览器流程

使用 `GET /auth/oidc/login`、`GET /auth/oidc/elevate` 和 `GET /auth/oidc/callback`：

| 接口 | 文档声明 | 实际返回 | 是否一致 |
|------|----------|----------|---------|
| `GET /auth/oidc/login` | Error 模型 | 纯文本 | ❌ 不一致 |
| `GET /auth/oidc/elevate` | Error 模型 | Error 模型 | ✅ 一致 |
| `GET /auth/oidc/callback` | Error 模型 | 纯文本 | ❌ 不一致 |

- ⚠️ 需要处理两种错误格式：提升入口是 JSON，登录入口和回调是纯文本
- ⚠️ 回调错误显示在浏览器/弹窗中
- 📌 **技术可行性校正**：
  - ❌ **不是绝对无法通过 JS 获取**：技术上有多种可能的方案
  - ⚠️ **但各有约束**：
    1. **弹窗轮询 URL 方案**：如果回调在同源下，可以通过 `popup.location.href` 读取 URL 中的错误参数（但当前错误不在 URL 中）
    2. **postMessage 方案**：需要后端在错误页面注入脚本主动向 opener 发送消息（当前未实现）
    3. **服务端轮询方案**：可以创建一个临时的提升会话，主窗口通过 API 轮询状态（当前未设计）
    4. **重定向携带参数方案**：后端错误时重定向到前端页面并在 query 中携带错误（当前直接返回纯文本）
- ❌ 文档与实现部分不一致

#### 6.2.3 系统集成（System Integration）- 调用业务 API

使用 `/message`、`/application` 等接口：
- ✅ 所有错误均返回标准 `model.Error` JSON
- ✅ 可通过 `errorCode` 进行错误分类和告警

#### 6.2.4 监控系统（Monitoring）- 调用健康检查

使用 `GET /health`：
- ⚠️ 即使失败（500）也返回 `model.Health` JSON
- ✅ 可通过 `health` 和 `database` 字段判断状态
- ⚠️ 不要依赖 HTTP 状态码判断健康状态，必须检查响应体

### 6.3 接入方处理建议

**校正后的 Python 客户端错误处理**：

```python
import requests
import json
from typing import Optional, Dict, Any

class GotifyApiError(Exception):
    """标准 API 错误"""
    def __init__(self, error_code: int, error: str, error_description: str):
        self.error_code = error_code
        self.error = error
        self.error_description = error_description
        super().__init__(f"{error}: {error_description}")

class GotifyHealthError(Exception):
    """健康检查错误"""
    def __init__(self, health: str, database: str):
        self.health = health
        self.database = database
        super().__init__(f"Health check failed: health={health}, database={database}")

class GotifyOIDCError(Exception):
    """OIDC 浏览器流程纯文本错误"""
    def __init__(self, status_code: int, error_text: str):
        self.status_code = status_code
        self.error_text = error_text
        super().__init__(f"OIDC error ({status_code}): {error_text}")

def handle_response(response: requests.Response) -> Dict[str, Any]:
    """统一处理 API 响应"""
    content_type = response.headers.get('Content-Type', '')
    
    # 非错误响应直接返回 JSON
    if response.status_code < 400:
        if 'application/json' in content_type:
            return response.json()
        return {}
    
    # 错误响应处理
    # 1. OIDC 浏览器流程回调阶段: text/plain
    if 'text/plain' in content_type:
        raise GotifyOIDCError(response.status_code, response.text)
    
    # 2. JSON 响应
    try:
        data = response.json()
    except json.JSONDecodeError:
        raise GotifyApiError(
            error_code=response.status_code,
            error="Invalid Response",
            error_description=f"Invalid JSON: {response.text[:200]}"
        )
    
    # 3. 健康检查: model.Health
    if 'health' in data and 'database' in data:
        raise GotifyHealthError(
            health=data['health'],
            database=data['database']
        )
    
    # 4. 标准 API 错误 / OIDC 提升入口: model.Error
    if 'errorCode' in data:
        raise GotifyApiError(
            error_code=data['errorCode'],
            error=data.get('error', 'Unknown Error'),
            error_description=data.get('errorDescription', '')
        )
    
    # 5. 未知格式
    raise GotifyApiError(
        error_code=response.status_code,
        error="Unknown Error Format",
        error_description=f"Response: {json.dumps(data)[:200]}"
    )

# OIDC 提升流程示例（Web 应用集成）
def integrate_oidc_elevation_web(client_id: int, duration_seconds: int):
    """
    Web 应用集成 OIDC 提升流程的说明：
    
    当前实现限制：
    1. 提升入口 /auth/oidc/elevate 返回标准 Error JSON（可通过 API 调用获取错误）
    2. 回调 /auth/oidc/callback 返回纯文本错误（显示在弹窗中）
    3. 回调错误无法通过 API 直接获取，因为回调是浏览器跳转
    
    技术上可行的改进方向（需要后端配合）：
    A. 后端在错误时重定向到前端页面并携带 error 参数
    B. 后端在错误页面注入 postMessage 脚本向主窗口发送错误
    C. 后端创建临时错误会话，前端通过 API 轮询获取
    
    当前推荐的集成方式：
    - 先通过 API 调用提升入口验证参数正确性
    - 如果参数验证失败，直接显示错误给用户
    - 如果参数验证成功，打开弹窗让用户完成 OIDC 认证
    - 弹窗关闭后，通过 API 检查是否提升成功
    - 如果未成功，提示用户检查弹窗中显示的错误信息
    """
    base_url = "https://gotify.example.com"
    
    # 阶段一：先通过 API 验证参数（可选，提前捕获参数错误）
    try:
        # 注意：这会消耗一个 state，如果直接打开弹窗可能导致 state 不一致
        # 实际集成时建议直接打开弹窗，让参数错误在弹窗中显示
        response = requests.get(
            f"{base_url}/auth/oidc/elevate",
            params={"id": client_id, "durationSeconds": duration_seconds},
            allow_redirects=False  # 不跟随重定向
        )
        # 如果是 302 重定向说明参数验证通过
        if response.status_code == 302:
            authorize_url = response.headers['Location']
            print(f"请在浏览器中访问: {authorize_url}")
        else:
            # 非 302 说明有错误，尝试解析 Error 模型
            handle_response(response)
    except GotifyApiError as e:
        print(f"参数错误: {e.error_code} - {e.error}")
        print(f"详情: {e.error_description}")
        return False
    
    # 阶段二：用户在浏览器/弹窗中完成认证
    # 回调错误会显示在浏览器中，应用无法通过 API 获取
    # 建议在 UI 中提示用户：如果发生错误，错误信息会显示在弹窗中
    
    # 阶段三：验证是否提升成功
    # 通过用户信息 API 检查是否已提升
    
    return True

# 使用示例
try:
    response = requests.get('https://gotify.example.com/message')
    data = handle_response(response)
    print("Messages:", data)
except GotifyApiError as e:
    print(f"API Error: {e.error_code} - {e.error}")
    print(f"Description: {e.error_description}")
except GotifyHealthError as e:
    print(f"Health Check Failed: {e.health}")
    print(f"Database Status: {e.database}")
except GotifyOIDCError as e:
    print(f"OIDC Callback Error: {e.status_code}")
    print(f"Error: {e.error_text}")
```

### 6.4 影响评估（校正后）

| 影响点 | 说明 | 严重程度 |
|--------|------|----------|
| **集成复杂度** | 需要处理三种不同的错误响应格式（Error JSON / Health JSON / text/plain） | 🔴 高 |
| **文档一致性** | OIDC 登录入口和回调的 Swagger 文档与实际响应不符，误导接入方 | 🔴 高 |
| **OIDC 提升入口** | 文档与实现一致，返回标准 Error JSON，处理简单 | ✅ 无问题 |
| **回调错误回传** | 当前实现未建立回传机制，但技术上存在改进空间（需要后端配合） | 🟡 中 |
| **错误恢复** | 链路 A：401 可触发刷新 token，400 需检查参数，500 需重试 | 🟡 中 |
| **版本兼容性** | 错误结构稳定，但错误信息文本可能随版本变化，依赖字符串匹配较脆弱 | 🟡 中 |
| **监控告警** | 链路 A：可基于 `errorCode` 分类统计；健康检查需检查响应体字段 | ✅ 好 |

---

## 7. 改进建议

### 7.1 统一 OIDC 错误响应模型（最高优先级）

**问题**：OIDC 登录入口和回调阶段返回纯文本，与文档和其他接口不一致

**建议**：

#### 方案 A：OIDC 浏览器流程改用 Gin 原生 handler（推荐）

```go
// api/oidc.go - LoginHandler 改为原生 gin handler
func (a *OIDCAPI) LoginHandler() gin.HandlerFunc {
    return func(ctx *gin.Context) {
        clientName := ctx.Query("name")
        if clientName == "" {
            ctx.AbortWithError(http.StatusBadRequest, errors.New("invalid client name"))
            return
        }
        state, err := a.generateState()
        if err != nil {
            ctx.AbortWithError(http.StatusInternalServerError, err)
            return
        }
        a.pendingSessions.Set(time.Now(), state, &pendingOIDCSession{
            ClientName: clientName, 
            CreatedAt: time.Now(),
        })
        rp.AuthURLHandler(func() string { return state }, a.Provider)(
            ctx.Writer, ctx.Request,
        )
    }
}

// CallbackHandler 同理改为原生 gin handler
```

**优点**：
- 统一使用 Error 模型
- 错误经过中间件处理
- 文档与实现一致

#### 方案 B：OIDC 浏览器流程错误时重定向回 UI 并携带错误参数

```go
// 出错时不直接返回错误，而是重定向回 UI 并在 query 中携带错误信息
func callbackError(w http.ResponseWriter, r *http.Request, errMsg string, status int) {
    redirectURL := fmt.Sprintf("../../?error=%s&status=%d", 
        url.QueryEscape(errMsg), status)
    http.Redirect(w, r, redirectURL, http.StatusTemporaryRedirect)
}
```

**优点**：
- 用户不会看到纯文本错误页面或原始 JSON
- UI 可以统一处理错误显示
- 前端可以通过 URL 参数获取错误信息

#### 方案 C：OIDC 弹窗错误页面注入 postMessage 脚本

如果后端不便改为 JSON，可以在纯文本错误页面中注入脚本：

```go
func callbackError(w http.ResponseWriter, errMsg string, status int) {
    // 返回 HTML 页面而不是纯文本
    w.Header().Set("Content-Type", "text/html; charset=utf-8")
    w.WriteHeader(status)
    fmt.Fprintf(w, `
<!DOCTYPE html>
<html>
<head><title>Error</title></head>
<body>
    <h1>%s</h1>
    <p>%s</p>
    <script>
        // 向主窗口发送错误信息
        if (window.opener) {
            window.opener.postMessage({
                type: 'oidc-error',
                status: %d,
                message: %q
            }, '*');
        }
    </script>
</body>
</html>`, http.StatusText(status), errMsg, status, errMsg)
}
```

**前端配合**：

```typescript
// 在主窗口监听错误消息
window.addEventListener('message', (event) => {
    if (event.data && event.data.type === 'oidc-error') {
        snack(`OIDC Error: ${event.data.message}`);
        // 关闭弹窗
        if (this.oidcPopup) {
            this.oidcPopup.close();
        }
    }
});
```

**优点**：
- 前端可以主动获取错误信息
- 用户体验更好
- 不需要大幅修改后端架构

### 7.2 前端 OIDC 提升错误提示改进

在后端未统一之前，前端可以改进 OIDC 提升流程的错误提示：

```typescript
// ui/src/ElevateStore.ts
private checkOidcPopup = async () => {
    if (this.oidcPopup && !this.oidcPopup.closed) {
        return;
    }
    window.clearInterval(this.oidcPollIntervalId);
    this.oidcPollIntervalId = undefined;

    try {
        await this.currentUser.tryAuthenticate();
    } catch {
        // errors handled in tryAuthenticate
    }

    if (!this.elevated) {
        // 改进错误提示，引导用户查看弹窗
        this.snack('OIDC elevation was not completed. ' +
                  'If an error occurred, details were shown in the popup window.');
    }
    this.cleanupOidcElevate();
};
```

### 7.3 健康检查接口改进

**方案 A：健康检查失败时也返回 Error 模型**

```go
// api/health.go
func (a *HealthAPI) Health(ctx *gin.Context) {
    if err := a.DB.Ping(); err != nil {
        ctx.AbortWithError(500, errors.New("database connection failed"))
        return
    }
    ctx.JSON(200, model.Health{
        Health:   model.StatusGreen,
        Database: model.StatusGreen,
    })
}
```

**注意**：这会改变健康检查的语义，需要评估对现有监控系统的影响。

**方案 B：保持现状但在文档中明确说明**

如果为了兼容现有监控系统，可以保持现状，但在文档中明确说明健康检查的错误格式。

### 7.4 错误信息标准化

**问题**：相同语义的错误信息表述不统一

**建议**：定义统一的错误信息常量

```go
package errors

const (
    ErrApplicationNotFound = "application with id %d does not exist"
    ErrClientNotFound      = "client with id %d does not exist"
    ErrMessageNotFound     = "message with id %d does not exist"
    ErrUserNotFound        = "user with id %d does not exist"
    ErrPluginNotFound      = "plugin with id %d does not exist"
)
```

### 7.5 增加业务错误码

**问题**：仅靠 HTTP 状态码和错误描述无法精确定位错误类型

**建议**：扩展 Error 模型，增加业务错误码

```go
type Error struct {
    Error            string `json:"error"`
    ErrorCode        int    `json:"errorCode"`        // HTTP 状态码
    ErrorDescription string `json:"errorDescription"`
    BusinessCode     string `json:"businessCode"`     // 业务错误码，如 "APP_NOT_FOUND"
}
```

### 7.6 敏感错误包装

**问题**：500 错误直接暴露数据库错误详情

**建议**：对内部错误进行包装

```go
func writeError(ctx *gin.Context, errString string) {
    status := http.StatusBadRequest
    if ctx.Writer.Status() != http.StatusOK {
        status = ctx.Writer.Status()
    }
    
    description := errString
    if status == http.StatusInternalServerError {
        description = "internal server error"
        // 记录原始错误到日志
        log.Error(errString)
    }
    
    ctx.JSON(status, &model.Error{
        Error:            http.StatusText(status),
        ErrorCode:        status,
        ErrorDescription: description,
    })
}
```

### 7.7 UI 端健壮性改进

**建议**：增加响应格式检查，处理非标准错误响应

```typescript
// ui/src/apiAuth.ts
interface IApiError {
    error: string;
    errorCode: number;
    errorDescription: string;
}

interface IHealthStatus {
    health: string;
    database: string;
}

function isApiError(data: any): data is IApiError {
    return data && typeof data === 'object' && 'errorCode' in data;
}

function isHealthStatus(data: any): data is IHealthStatus {
    return data && typeof data === 'object' && 'health' in data;
}

// 在 axios 拦截器中
if (status === 400 || status === 403 || status === 500) {
    const data = error.response.data;
    if (isApiError(data)) {
        snack(data.error + ': ' + data.errorDescription);
    } else if (isHealthStatus(data)) {
        snack(`Health check failed: database=${data.database}`);
    } else if (typeof data === 'string') {
        snack(data); // 纯文本错误
    } else {
        snack('An unexpected error occurred');
    }
}
```

---

## 8. 总结

### 8.1 OIDC 提升流程的两段链路校正

| 阶段 | 接口 | handler 类型 | 错误格式 | 经过中间件 |
|------|------|-------------|----------|-----------|
| 入口 | `GET /auth/oidc/elevate` | 原生 Gin handler | ✅ `model.Error` JSON | 是 |
| 回调 | `GET /auth/oidc/callback` | `gin.WrapF` 包装 | ❌ `text/plain` 纯文本 | 否 |

**✅ 重要校正**：OIDC 提升流程**不是全程 text/plain**。入口阶段返回标准 Error JSON，只有回调阶段才返回纯文本。

### 8.2 OIDC 弹窗错误传递校正

| 维度 | 说明 |
|------|------|
| **当前实现** | 主窗口仅轮询检查弹窗是否关闭，不尝试读取弹窗中的任何错误信息 |
| **绝对判断修正** | 不是"无法通过 JS 获取"，而是"当前未建立回传机制" |
| **技术可行性** | 同源场景下技术上可行（可通过 `postMessage`、URL 参数、服务端轮询等方式实现） |
| **现实约束** | 1. OIDC provider 页面跨域无法访问；2. 成功时弹窗会自动关闭；3. 当前错误页面没有通信脚本 |
| **改进方向** | 需要后端配合，在错误时重定向到前端或注入通信脚本 |

### 8.3 两类调用链路的本质区别

| 维度 | Axios API 链路（链路 A） | 浏览器跳转/弹窗链路（链路 B） |
|------|-------------------------|-----------------------------|
| **调用方式** | `axios.get/post()` | `<a href>` / `window.open()` |
| **错误格式** | `model.Error` JSON | 登录入口+回调: text/plain；提升入口: Error JSON |
| **经过拦截器** | ✅ 是 | ❌ 否 |
| **错误处理** | 前端统一拦截、格式化显示 | 浏览器直接渲染 / 当前未建立回传机制 |
| **用户体验** | 错误以 snackbar 形式友好展示 | 跳转时显示纯文本/JSON，弹窗错误当前不透明 |
| **占比** | ~95% 接口 + OIDC 提升入口 | OIDC 登录入口 + 所有回调 |

### 8.4 优点

1. **Axios API 结构统一**：绝大多数 API 错误响应使用相同的 JSON 结构
2. **OIDC 提升入口规范**：提升入口返回标准 Error 模型，与文档一致
3. **链路清晰**：标准路径下控制器 → 中间件 → 响应，流程明确
4. **HTTP 友好**：错误码与 HTTP 语义一致
5. **调试便利**：`errorDescription` 提供详细上下文（标准 API）

### 8.5 不足（按严重程度排序）

1. 🔴 **OIDC 登录入口错误体验差**：出错时显示纯文本页面
2. 🔴 **OIDC 提升入口错误显示原始 JSON**：用户无法理解
3. 🔴 **OIDC 回调错误纯文本**：用户体验差
4. 🔴 **文档与实现部分不一致**：OIDC 登录入口和回调的 Swagger 文档错误
5. 🟡 **OIDC 提升错误不透明**：当前未建立回传机制，但技术上可行
6. 🟡 **响应模型不统一**：三种不同的错误响应格式，增加客户端集成复杂度
7. 🟡 **错误信息不统一**：相同类型错误的描述文本不一致
8. 🟡 **缺少业务错误码**：第三方接入方难以精确判断错误类型
9. 🟡 **敏感信息暴露**：500 错误直接返回内部错误详情
10. 🟡 **国际化困难**：硬编码英文错误信息

### 8.6 关键文件速查

| 文件 | 职责 |
|------|------|
| `model/error.go` | 标准错误响应模型定义 |
| `model/health.go` | 健康检查模型定义（非错误） |
| `error/handler.go` | 全局错误处理中间件（链路 A + OIDC 提升入口） |
| `api/errorHandling.go` | 控制器错误抛出工具 |
| `api/health.go` | 健康检查接口（绕过错误中间件） |
| `api/oidc.go` | OIDC 接口（提升入口为原生 Gin handler，登录入口和回调为 gin.WrapF） |
| `router/router.go` | 中间件注册、路由定义 |
| `auth/authentication.go` | 认证相关错误抛出 |
| `ui/src/apiAuth.ts` | 前端 axios 错误拦截器（仅 axios 请求） |
| `ui/src/ElevateStore.ts` | OIDC 提升流程的弹窗管理（当前仅轮询关闭状态，不读取错误） |
| `ui/src/user/Login.tsx` | OIDC 登录按钮 |
