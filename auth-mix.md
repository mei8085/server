# Cookie 与 Token 混合认证规则说明

## 1. 认证方式概述

服务支持多种认证方式，包括：
- **Basic Auth**：通过 `Authorization: Basic` 头进行用户认证
- **Token**：通过多种方式传递的客户端/应用令牌
- **Cookie**：通过 `gotify-client-token` Cookie 传递令牌

## 2. Token 提取优先级

在 `readTokenFromRequest` 函数（`auth/authentication.go:205-216`）中，Token 按以下优先级顺序从请求中提取：

| 优先级 | 来源 | 说明 | 示例 |
|--------|------|------|------|
| 1 | Query 参数 | URL 查询参数 `token` | `?token=xxxxxx` |
| 2 | Header `X-Gotify-Key` | 自定义请求头 | `X-Gotify-Key: xxxxxx` |
| 3 | Header `Authorization` | Bearer 认证 | `Authorization: Bearer xxxxxx` |
| 4 | Cookie | Cookie `gotify-client-token` | 浏览器自动携带 |

**重要**：只要找到一个有效的 Token 源，就会使用该值，后续来源不再检查。

## 3. 认证方法组合与优先级

不同 API 端点使用不同的认证中间件，各中间件的认证尝试顺序如下：

### 3.1 RequireClient（客户端接口）

**适用端点**：`/application`、`/client`、`/message`、`/stream` 等

**认证顺序**（`auth/authentication.go:52-54`）：
1. **Basic Auth**（用户凭据）
   - 成功 → 注册用户，继续请求
   - 失败/未提供 → 进入下一步
2. **Client Token**（任意 Token 来源）
   - 成功 → 注册客户端，继续请求
   - 失败 → 返回 401

### 3.2 RequireAdmin（管理员接口）

**适用端点**：`/user`（管理用户列表、删除用户等）

**认证顺序**（`auth/authentication.go:46-48`）：
1. **Basic Auth** + 管理员检查
   - 成功 → 注册用户，继续请求
   - 非管理员 → 返回 403
   - 失败/未提供 → 进入下一步
2. **Client Token** + 管理员检查 + 提升权限检查
   - 成功 → 注册客户端，继续请求
   - 非管理员 → 返回 403
   - 未提升 → 返回 403（session not elevated）
   - 失败 → 返回 401

### 3.3 RequireElevatedClient（提升权限接口）

**适用端点**：删除应用、删除客户端、修改密码等

**认证顺序**（`auth/authentication.go:56-59`）：
1. **Basic Auth**
   - 成功 → 注册用户，继续请求
   - 失败/未提供 → 进入下一步
2. **Client Token** + 提升权限检查
   - 已提升 → 注册客户端，继续请求
   - 未提升 → 返回 403（session not elevated）
   - 失败 → 返回 401

### 3.4 RequireApplicationToken（消息推送接口）

**适用端点**：`POST /message`（应用推送消息）

**认证顺序**（`auth/authentication.go:62-76`）：
1. **Application Token**（任意 Token 来源）
   - 成功 → 注册应用，继续请求
   - 失败 → 进入下一步
2. **Basic Auth** 检查（仅用于区分认证失败原因）
   - 有效用户认证 → 返回 403（不允许用户认证访问应用端点）
   - 无效 → 返回 401

### 3.5 Optional（可选认证）

**适用端点**：`POST /user`（用户注册）

**行为**：无论认证成功与否，都继续请求。认证成功时，注册用户/客户端信息到上下文。

## 4. 典型场景分析

### 场景 1：请求同时包含 Cookie 和 Header Token

**请求**：
```
Cookie: gotify-client-token=CookieToken123
X-Gotify-Key: HeaderToken456
```

**结果**：
- 优先使用 `X-Gotify-Key` 中的 `HeaderToken456`
- Cookie 中的 Token 被忽略

### 场景 2：请求同时包含 Basic Auth 和 Client Token

**请求**：
```
Authorization: Basic dXNlcjE6cGFzc3dvcmQ=
X-Gotify-Key: ClientToken123
```

**结果**（RequireClient 端点）：
- 优先验证 Basic Auth
- 如果用户凭据有效 → 使用用户身份，Token 被忽略
- 如果用户凭据无效 → 回退到 Token 验证

### 场景 3：Cookie 中的 Token 是 Application Token

**请求**：
```
Cookie: gotify-client-token=AppToken_A123
```

**访问**：`POST /message`（RequireApplicationToken）

**结果**：
- 从 Cookie 提取 Token
- 验证为有效 Application Token
- 认证成功

### 场景 4：Cookie 中的 Token 是 Client Token，访问应用端点

**请求**：
```
Cookie: gotify-client-token=ClientToken_C123
```

**访问**：`POST /message`（RequireApplicationToken）

**结果**：
- 从 Cookie 提取 Token
- Application Token 验证失败（因为是 Client Token）
- 检查是否有 Basic Auth → 无
- 返回 401

### 场景 5：同时提供 Query Token 和 Cookie Token

**请求**：
```
GET /message?token=QueryToken123
Cookie: gotify-client-token=CookieToken456
```

**结果**：
- 优先使用 Query 参数中的 `QueryToken123`
- Cookie 中的 Token 被忽略

## 5. Token 类型区分

系统中有三种 Token 类型，通过前缀区分（`auth/token.go:11-13`）：
- **Application Token**：`A` 前缀
- **Client Token**：`C` 前缀  
- **Plugin Token**：`P` 前缀

**注意**：Cookie 中可以存储任意类型的 Token，系统会根据 Token 内容自动识别类型。

## 6. Cookie 刷新机制

当 Token 通过 Cookie 传递且验证成功时，系统会自动刷新 Cookie：

- 刷新条件：`LastUsed` 为空或超过 5 分钟（`auth/authentication.go:159`）
- 有效期：7 天（`CookieMaxAge = 7 * 24 * 60 * 60`）
- Cookie 属性：`HttpOnly=true`、`SameSite=Strict`、`Secure`（根据配置）

## 7. 认证状态流转

```
请求到达
    ↓
┌─────────────────────────────────┐
│  按优先级提取 Token              │
│  Query → Header → Auth → Cookie │
└─────────────────────────────────┘
    ↓
    ├─────────────────────────────────────────┐
    ↓                                         ↓
┌─────────────┐     成功     ┌─────────────────────────┐
│ Basic Auth? │ ───────────> │ 注册用户到上下文         │
└─────────────┘              └─────────────────────────┘
    ↓ 失败/未提供                          ↓
┌─────────────┐     成功     ┌─────────────────────────┐
│ Token 验证? │ ───────────> │ 注册客户端/应用到上下文  │
└─────────────┘              └─────────────────────────┘
    ↓ 失败
┌─────────────┐
│ 返回 401    │
└─────────────┘
```

## 8. 关键代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| Token 提取优先级 | `auth/authentication.go` | 205-216 |
| 认证评估逻辑 | `auth/authentication.go` | 84-112 |
| RequireClient 逻辑 | `auth/authentication.go` | 52-54 |
| RequireAdmin 逻辑 | `auth/authentication.go` | 46-48 |
| Cookie 设置 | `auth/cookie.go` | 12-22 |
| Token 生成规则 | `auth/token.go` | 37-55 |
