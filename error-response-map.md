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

## 2. 错误处理链路

### 2.1 整体架构

```
控制器抛错 → Gin Error 队列 → 错误处理中间件 → 统一响应输出
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
- 调用示例：`successOrAbort(ctx, 500, a.DB.CreateMessage(msg))`

**方式二：直接调用 `ctx.AbortWithError()`**

```go
ctx.AbortWithError(404, errors.New("application does not exist"))
```

- 用于业务逻辑错误，如资源不存在、权限不足等
- 会被错误中间件捕获并转为标准 Error 模型

**方式三：参数绑定错误**

- 通过 `ctx.Bind()` / `ctx.MustBindWith()` 触发
- 由 Gin 自动添加到错误队列，类型为 `gin.ErrorTypeBind`

**方式四：绕过中间件的直接响应**（⚠️ 非标准路径）

- `ctx.JSON()`：直接写入 JSON 响应，不经过错误中间件
- `http.Error()`：返回纯文本错误，用于 OIDC 浏览器流程

#### 2.2.2 错误捕获中间件

定义在 `error/handler.go:14-64`，核心逻辑：

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

**注意**：错误中间件在所有路由之前注册，但仍有接口绕过了它（见第 4 章）。

---

## 3. 控制器抛错分析

### 3.1 消息接口 (MessageAPI)

| 接口 | 错误场景 | 状态码 | 错误信息 | 响应模型 |
|------|----------|--------|----------|----------|
| `GET /message` | 数据库查询失败 | 500 | 数据库错误原文 | ✅ Error |
| `GET /application/{id}/message` | 应用不存在 | 404 | `"application does not exist"` | ✅ Error |
| `DELETE /message/{id}` | 消息不存在 | 404 | `"message does not exist"` | ✅ Error |
| `POST /message` | 数据库插入失败 | 500 | 数据库错误原文 | ✅ Error |
| 参数绑定失败 | 400 | 校验错误详情 | ✅ Error |

### 3.2 应用接口 (ApplicationAPI)

| 接口 | 错误场景 | 状态码 | 错误信息 | 响应模型 |
|------|----------|--------|----------|----------|
| `POST /application` | 排序键重复 | 400 | `"sort key is not unique"` | ✅ Error |
| `DELETE /application/{id}` | 删除内部应用 | 400 | `"cannot delete internal application"` | ✅ Error |
| `DELETE /application/{id}` | 应用不存在 | 404 | `"app with id %d doesn't exists"` | ✅ Error |
| `POST /{id}/image` | 缺少文件 | 400 | `"file with key 'file' must be present"` | ✅ Error |
| `POST /{id}/image` | 非图片文件 | 400 | `"file must be an image"` | ✅ Error |
| `POST /{id}/image` | 无效扩展名 | 400 | `"invalid file extension"` | ✅ Error |
| `DELETE /{id}/image` | 无自定义图片 | 400 | `"app with id %d does not have a customized image"` | ✅ Error |

### 3.3 客户端接口 (ClientAPI)

| 接口 | 错误场景 | 状态码 | 错误信息 | 响应模型 |
|------|----------|--------|----------|----------|
| `PUT /client/{id}` | 客户端不存在 | 404 | `"client with id %d doesn't exists"` | ✅ Error |
| `DELETE /client/{id}` | 客户端不存在 | 404 | `"client with id %d doesn't exists"` | ✅ Error |
| `POST /{id}/elevate` | 客户端不存在 | 404 | `"client not found"` | ✅ Error |

### 3.4 用户接口 (UserAPI)

| 接口 | 错误场景 | 状态码 | 错误信息 | 响应模型 |
|------|----------|--------|----------|----------|
| `POST /user` | 用户名已存在 | 400 | `"username already exists"` | ✅ Error |
| `POST /user` | 注册关闭时非管理员创建 | 401/403 | `"you are not allowed to access this api"` | ✅ Error |
| `POST /user` | 非管理员创建管理员用户 | 401/403 | `"you are not allowed to create an admin user"` | ✅ Error |
| `GET /user/{id}` | 用户不存在 | 404 | `"user does not exist"` | ✅ Error |
| `DELETE /user/{id}` | 删除最后一个管理员 | 400 | `"cannot delete last admin"` | ✅ Error |
| `POST /user/{id}` | 降级最后一个管理员 | 400 | `"cannot delete last admin"` | ✅ Error |

### 3.5 认证中间件 (auth/authentication.go)

| 错误场景 | 状态码 | 错误信息 | 响应模型 |
|----------|--------|----------|----------|
| 未提供有效认证 | 401 | `"you need to provide a valid access token or user credentials to access this api"` | ✅ Error |
| 权限不足 | 403 | `"you are not allowed to access this api"` | ✅ Error |
| 会话未提升 | 403 | `"session not elevated, use basic auth or call /client:elevate"` | ✅ Error |

### 3.6 会话接口 (SessionAPI)

| 接口 | 错误场景 | 状态码 | 错误信息 | 响应模型 |
|------|----------|--------|----------|----------|
| `POST /auth/local/login` | 缺少 Basic Auth | 401 | `"basic auth required"` | ✅ Error |
| `POST /auth/local/login` | 凭证无效 | 401 | `"invalid credentials"` | ✅ Error |
| `POST /auth/logout` | 无客户端认证 | 403 | `"no client auth provided"` | ✅ Error |

### 3.7 插件接口 (PluginAPI)

| 接口 | 错误场景 | 状态码 | 错误信息 | 响应模型 |
|------|----------|--------|----------|----------|
| 插件不存在/无权限 | 404 | `"unknown plugin"` | ✅ Error |
| 插件实例不存在 | 404 | `"plugin instance not found"` | ✅ Error |
| 重复启用/禁用 | 400 | 插件错误原文 | ✅ Error |
| 插件不支持能力 | 400 | `"plugin does not support %s"` | ✅ Error |
| YAML 配置解析失败 | 400 | 解析错误原文 | ✅ Error |
| 配置验证失败 | 400 | 验证错误原文 | ✅ Error |

### 3.8 健康检查接口 (HealthAPI) ⚠️ 非标准

| 接口 | 错误场景 | 状态码 | 响应内容 | 响应模型 |
|------|----------|--------|----------|----------|
| `GET /health` | 数据库连接失败 | 500 | `{"health": "orange", "database": "red"}` | ❌ Health 模型 |
| `GET /health` | 数据库正常 | 200 | `{"health": "green", "database": "green"}` | ✅ Health 模型 |

**关键差异**：
- 健康检查接口**从不返回 Error 模型**
- 即使数据库失败（500 状态码），仍返回 `model.Health` 结构
- 直接使用 `ctx.JSON()` 写入响应，绕过错误中间件
- 健康检查路由在 `router/router.go:119` 注册，位于错误中间件之后

### 3.9 OIDC 接口

#### 3.9.1 OIDC 外部接口（Native App）✅ 标准

| 接口 | 错误场景 | 状态码 | 错误信息 | 响应模型 |
|------|----------|--------|----------|----------|
| `POST /auth/oidc/external/authorize` | 参数绑定失败 | 400 | 绑定错误详情 | ✅ Error |
| `POST /auth/oidc/external/authorize` | 状态生成失败 | 500 | 错误原文 | ✅ Error |
| `POST /auth/oidc/external/token` | 参数绑定失败 | 400 | 绑定错误详情 | ✅ Error |
| `POST /auth/oidc/external/token` | state 无效/过期 | 400 | `"unknown or expired state"` | ✅ Error |
| `POST /auth/oidc/external/token` | token 交换失败 | 401 | `"token exchange failed: ..."` | ✅ Error |
| `POST /auth/oidc/external/token` | 用户信息获取失败 | 500 | `"failed to get user info: ..."` | ✅ Error |
| `POST /auth/oidc/external/token` | 用户不存在/自动注册关闭 | 403 | `"user does not exist and auto-registration is disabled"` | ✅ Error |
| `GET /auth/oidc/elevate` | 参数绑定失败 | 400 | 绑定错误详情 | ✅ Error |
| `GET /auth/oidc/elevate` | 状态生成失败 | 500 | 错误原文 | ✅ Error |

#### 3.9.2 OIDC 浏览器接口 ⚠️ 非标准

| 接口 | 错误场景 | 状态码 | 响应内容 | 响应模型 |
|------|----------|--------|----------|----------|
| `GET /auth/oidc/login` | 缺少 client name | 400 | `"invalid client name"`（纯文本） | ❌ text/plain |
| `GET /auth/oidc/login` | 状态生成失败 | 500 | `"failed to generate state: ..."`（纯文本） | ❌ text/plain |
| `GET /auth/oidc/callback` | 用户解析失败 | 403/500 | 错误详情（纯文本） | ❌ text/plain |
| `GET /auth/oidc/callback` | state 无效/过期 | 400 | `"unknown or expired state"`（纯文本） | ❌ text/plain |
| `GET /auth/oidc/callback` | 客户端创建失败 | 500 | `"failed to create client: ..."`（纯文本） | ❌ text/plain |
| `GET /auth/oidc/callback`（提升流程）| 数据库错误 | 500 | `"database error: ..."`（纯文本） | ❌ text/plain |
| `GET /auth/oidc/callback`（提升流程）| 客户端不存在 | 404 | `"client not found"`（纯文本） | ❌ text/plain |
| `GET /auth/oidc/callback`（提升流程）| 提升失败 | 500 | `"failed to elevate session: ..."`（纯文本） | ❌ text/plain |

**关键差异**：
- OIDC 浏览器流程使用 `gin.WrapF()` 包装标准 `http.HandlerFunc`
- 错误时调用 `http.Error()`，返回 `Content-Type: text/plain; charset=utf-8`
- 纯文本格式，不是 JSON，也不包含 Error 模型字段
- 绕过 Gin 错误中间件，直接写入 HTTP 响应

### 3.10 通用工具错误

| 场景 | 状态码 | 错误信息 | 响应模型 |
|------|--------|----------|----------|
| ID 路径参数解析失败 | 400 | `"invalid id"` | ✅ Error |
| 路由不匹配 | 404 | `"page not found"` | ✅ Error |

---

## 4. 错误统一性分析

### 4.1 统一程度评估（修正后）

| 维度 | 统一程度 | 说明 |
|------|----------|------|
| **响应结构** | ⚠️ 大部分统一 | 绝大多数 API 返回 Error 模型，但健康检查和 OIDC 浏览器接口例外 |
| **HTTP 状态码** | ✅ 基本统一 | 同类错误使用相同状态码（404 资源不存在、400 参数错误等） |
| **错误描述风格** | ⚠️ 部分不统一 | 相同语义的错误信息有多种表述 |
| **错误抛出方式** | ⚠️ 三种路径 | `successOrAbort` / `ctx.AbortWithError` / 直接写入响应 |

### 4.2 三类错误响应路径

```
┌─────────────────────────────────────────────────────────────┐
│                     错误响应路径分类                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  路径 A：标准路径（95% API）                                 │
│  ───────────────────────────────────────────────────────    │
│  控制器 → ctx.AbortWithError() → Gin 错误队列 → 中间件       │
│         → writeError() → model.Error JSON                   │
│                                                             │
│  路径 B：健康检查（/health）                                 │
│  ───────────────────────────────────────────────────────    │
│  控制器 → ctx.JSON() → model.Health JSON（即使 500）        │
│         （绕过错误中间件）                                   │
│                                                             │
│  路径 C：OIDC 浏览器流程                                     │
│  ───────────────────────────────────────────────────────    │
│  控制器 → http.Error() → text/plain 纯文本                  │
│         （绕过 Gin 错误队列和中间件）                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 存在的不一致问题

#### 4.3.1 响应模型不统一（最严重）

| 接口类别 | 错误模型 | Content-Type |
|----------|----------|-------------|
| 消息/应用/客户端/用户/插件/会话 API | ✅ `model.Error` | `application/json` |
| 健康检查 API | ❌ `model.Health` | `application/json` |
| OIDC 外部 API | ✅ `model.Error` | `application/json` |
| OIDC 浏览器 API | ❌ 纯文本 | `text/plain` |

**影响**：客户端需要处理三种不同的错误响应格式。

#### 4.3.2 资源不存在的错误信息不统一

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

#### 4.3.3 相同语义不同表述

| 语义 | 多种表述 |
|------|---------|
| 资源不存在 | 至少 6 种不同表述 |
| 权限不足 | `"you are not allowed to access this api"` / `"you are not allowed to create an admin user"` |

#### 4.3.4 数据库错误直接暴露

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

---

## 5. 对客户端 UI 的影响（重新评估）

### 5.1 UI 错误处理机制

在 `ui/src/apiAuth.ts:6-23` 中定义了 axios 响应拦截器：

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

### 5.2 三类接口的实际表现

| 接口类别 | 状态码 | UI 表现 | 问题 |
|----------|--------|---------|------|
| 标准 API (路径 A) | 400/403/404/500 | ✅ 正常显示错误信息 | 无 |
| 健康检查 (路径 B) | 500 | ❌ 显示 `undefined: undefined` | `data.error` 和 `data.errorDescription` 不存在 |
| OIDC 浏览器 (路径 C) | 400/403/500 | ❌ 显示 `undefined: undefined` | `data` 是字符串，不是对象 |

**健康检查 500 时的实际响应**：
```json
{"health": "orange", "database": "red"}
```
UI 访问 `data.error` → `undefined`，`data.errorDescription` → `undefined`

**OIDC 浏览器错误的实际响应**：
```
invalid client name
```
UI 访问 `data.error` → `undefined`（因为 data 是字符串）

### 5.3 UI 处理逻辑（修正后）

| 状态码 | UI 行为 | 适用接口 |
|--------|---------|----------|
| 无响应（网络错误） | 显示 "Gotify server is not reachable, try refreshing the page." | 所有 |
| 401 | 尝试重新认证，显示 "Could not complete request." | 标准 API |
| 400 / 403 / 500 | 显示拼接的错误信息：`error + ': ' + errorDescription` | 标准 API |
| 健康检查 500 | 显示 `undefined: undefined` | 健康检查 |
| OIDC 浏览器错误 | 显示 `undefined: undefined` | OIDC 浏览器流程 |
| 其他 | 静默拒绝 Promise，上层自行处理 | - |

### 5.4 UI 类型定义

在 `ui/src/types.ts` 中**没有**定义 Error 接口类型，UI 直接访问 `error.response.data.error` 和 `error.response.data.errorDescription`。

### 5.5 影响评估（重新评估）

| 影响点 | 说明 | 严重程度 |
|--------|------|----------|
| **用户体验** | 健康检查和 OIDC 流程出错时显示 `undefined: undefined`，用户完全无法理解 | 🔴 高 |
| **健壮性** | UI 假设所有错误响应都是 Error 模型，但实际有三种格式 | 🔴 高 |
| **国际化** | 硬编码的英文错误信息难以进行多语言支持 | 🟡 中 |
| **错误分类** | UI 仅按 HTTP 状态码粗略分类，无法做精细化处理 | 🟡 中 |
| **调试友好** | `errorDescription` 详细信息有助于开发调试（标准 API） | ✅ 好 |

---

## 6. 对第三方接入方的影响（重新评估）

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
- OIDC 浏览器接口文档声明错误返回 `Error` 模型，但实际返回纯文本（文档错误）

### 6.2 接入方处理建议

**修正后的 Python 客户端错误处理**：

```python
import requests
import json

def handle_error(response):
    if response.status_code >= 400:
        content_type = response.headers.get('Content-Type', '')
        
        # 路径 C: OIDC 浏览器流程 - text/plain
        if 'text/plain' in content_type:
            error_desc = response.text
            raise ApiError(
                error_code=response.status_code,
                error=http_status_text(response.status_code),
                error_description=error_desc
            )
        
        # 路径 A/B: JSON 响应
        try:
            data = response.json()
        except json.JSONDecodeError:
            raise ServerError(f"Invalid JSON response: {response.text}")
        
        # 路径 B: 健康检查 - Health 模型
        if 'health' in data and 'database' in data:
            raise HealthCheckError(
                health=data['health'],
                database=data['database']
            )
        
        # 路径 A: 标准 Error 模型
        if 'errorCode' in data:
            error_code = data['errorCode']
            error_msg = data['error']
            error_desc = data['errorDescription']
            
            if error_code == 401:
                raise AuthenticationError(error_desc)
            elif error_code == 403:
                raise PermissionDenied(error_desc)
            elif error_code == 404:
                raise ResourceNotFound(error_desc)
            elif error_code == 400:
                raise BadRequest(error_desc)
            else:
                raise ServerError(error_desc)
        
        # 未知格式
        raise ServerError(f"Unknown error format: {data}")
```

### 6.3 影响评估（重新评估）

| 影响点 | 说明 | 严重程度 |
|--------|------|----------|
| **集成复杂度** | 需要处理三种不同的错误响应格式，增加了集成代码的复杂度 | 🔴 高 |
| **文档一致性** | OIDC 浏览器接口的 Swagger 文档与实际响应不符 | 🔴 高 |
| **错误恢复** | 401 可触发刷新 token，400 需要检查请求参数，500 需重试（标准 API） | 🟡 中 |
| **版本兼容性** | 错误结构稳定，但错误信息文本可能随版本变化，依赖字符串匹配较脆弱 | 🟡 中 |
| **监控告警** | 可基于 `errorCode` 进行错误分类统计（标准 API） | ✅ 好 |

---

## 7. 改进建议

### 7.1 统一错误响应模型（最高优先级）

**问题**：健康检查和 OIDC 浏览器流程绕过错误中间件，返回非标准格式

**建议**：

#### 方案 A：统一使用 Error 模型

```go
// 健康检查失败时返回 Error 模型
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

#### 方案 B：OIDC 浏览器流程改用 Gin 原生 handler

```go
// 不使用 gin.WrapF，改用原生 gin handler
func (a *OIDCAPI) LoginHandler() gin.HandlerFunc {
    return func(ctx *gin.Context) {
        clientName := ctx.Query("name")
        if clientName == "" {
            ctx.AbortWithError(http.StatusBadRequest, errors.New("invalid client name"))
            return
        }
        // ... 其余逻辑
    }
}
```

### 7.2 错误信息标准化

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

### 7.3 增加业务错误码

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

### 7.4 敏感错误包装

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

### 7.5 UI 端改进

**建议**：增加响应格式检查，处理非标准错误响应

```typescript
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
        snack(data); // OIDC 纯文本错误
    } else {
        snack('An unexpected error occurred');
    }
}
```

---

## 8. 总结

### 8.1 优点

1. **结构基本统一**：绝大多数 API 错误响应使用相同的 JSON 结构
2. **链路清晰**：标准路径下控制器 → 中间件 → 响应，流程明确
3. **HTTP 友好**：错误码与 HTTP 语义一致
4. **调试便利**：`errorDescription` 提供详细上下文（标准 API）

### 8.2 不足（按严重程度排序）

1. 🔴 **响应模型不统一**：健康检查和 OIDC 浏览器流程返回非标准格式，破坏客户端兼容性
2. 🔴 **文档与实现不一致**：OIDC 浏览器接口文档声明返回 Error 模型，但实际返回纯文本
3. 🟡 **错误信息不统一**：相同类型错误的描述文本不一致
4. 🟡 **缺少业务错误码**：第三方接入方难以精确判断错误类型
5. 🟡 **敏感信息暴露**：500 错误直接返回内部错误详情
6. 🟡 **国际化困难**：硬编码英文错误信息

### 8.3 关键文件速查

| 文件 | 职责 |
|------|------|
| `model/error.go` | 标准错误响应模型定义 |
| `model/health.go` | 健康检查模型定义（非错误） |
| `error/handler.go` | 全局错误处理中间件 |
| `api/errorHandling.go` | 控制器错误抛出工具 |
| `api/health.go` | 健康检查接口（绕过错误中间件） |
| `api/oidc.go` | OIDC 接口（浏览器流程绕过错误中间件） |
| `router/router.go` | 中间件注册、路由定义 |
| `auth/authentication.go` | 认证相关错误抛出 |
| `ui/src/apiAuth.ts` | 前端 axios 错误拦截器 |
