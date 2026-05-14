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

**重要规则：先命中非空值先使用，值为空则继续回退**。即某一优先级来源存在但值为空（如 `?token=`）时，会继续尝试下一优先级来源；只有提取到非空值时才会锁定该来源进行验证，不会再回退。

代码依据 (`auth/authentication.go:206-213`):
```go
if token := a.tokenFromQuery(ctx); token != "" {  // 只有非空才返回
    return token, false
} else if token := a.tokenFromXGotifyHeader(ctx); token != "" {
    return token, false
}
```

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

## 4. 401 与 403 触发条件按中间件分类

### 4.1 RequireClient 中间件

| 状态码 | 触发条件 |
|--------|----------|
| **401** | 1. Basic Auth 无效或未提供<br>2. Client Token 无效或未提供<br>（二者都失败） |
| **403** | 此中间件不触发 403 |

### 4.2 RequireAdmin 中间件

| 状态码 | 触发条件 |
|--------|----------|
| **401** | 1. Basic Auth 无效或未提供<br>2. Client Token 无效或未提供<br>（二者都失败） |
| **403** | 1. Basic Auth 有效但用户非管理员<br>2. Client Token 有效但关联用户非管理员<br>3. Client Token 有效但会话未提升（not elevated） |

### 4.3 RequireElevatedClient 中间件

| 状态码 | 触发条件 |
|--------|----------|
| **401** | 1. Basic Auth 无效或未提供<br>2. Client Token 无效或未提供<br>（二者都失败） |
| **403** | 1. Client Token 有效但会话未提升（not elevated） |

### 4.4 RequireApplicationToken 中间件

| 状态码 | 触发条件 |
|--------|----------|
| **401** | 1. Application Token 无效或未提供<br>2. 同时 Basic Auth 也无效或未提供 |
| **403** | 1. Application Token 无效或未提供<br>2. **但 Basic Auth 有效**<br>（认证有效但类型错误） |

## 5. 组合判定矩阵

以下矩阵覆盖 cookie、query、header、basic 同时出现时的组合判定场景。

**符号说明**：
- ✓：有效凭据（非空且有效）
- ✗：无效凭据（非空但无效）
- ∅：来源存在但值为空（如 `?token=`）
- -：未提供凭据
- (Q)：Query 参数
- (H)：Header `X-Gotify-Key`
- (A)：Header `Authorization: Bearer`
- (C)：Cookie
- (B)：Basic Auth

### 5.1 RequireClient 组合矩阵

| 序号 | Basic (B) | Query (Q) | Header (H) | Auth (A) | Cookie (C) | 实际使用 | 结果 | 说明 |
|------|-----------|-----------|------------|----------|------------|----------|------|------|
| 1 | - | ✓ | ✓ | ✓ | ✓ | B 无 → Q | 200 | Q 非空有效，直接使用 |
| 2 | - | ✗ | ✓ | ✓ | ✓ | Q | 401 | Q 非空无效，**不回退**到 H/A/C |
| 3 | - | ∅ | ✓ | ✓ | ✓ | Q 空 → H | 200 | Q 为空，回退到 H，H 有效 |
| 4 | - | ∅ | ✗ | ✓ | ✓ | Q 空 → H | 401 | Q 为空回退到 H，H 无效不继续回退 |
| 5 | - | - | ✓ | ✓ | ✓ | H | 200 | H 有效 |
| 6 | - | - | ✗ | ✓ | ✓ | H | 401 | H 无效，不回退到 A/C |
| 7 | - | - | ∅ | ✓ | ✓ | H 空 → A | 200 | H 为空回退到 A |
| 8 | - | - | - | ✓ | ✓ | A | 200 | A 有效 |
| 9 | - | - | - | ✗ | ✓ | A | 401 | A 无效，不回退到 C |
| 10 | - | - | - | ∅ | ✓ | A 空 → C | 200 | A 为空回退到 C |
| 11 | - | - | - | - | ✓ | C | 200 | C 有效 |
| 12 | - | - | - | - | ✗ | C | 401 | C 无效 |
| 13 | ✓ | ✓ | ✓ | ✓ | ✓ | B | 200 | Basic Auth 优先级最高，Token 全部忽略 |
| 14 | ✗ | ✓ | ✓ | ✓ | ✓ | B 失败 → Q | 200 | B 无效，回退到 Token 验证，使用 Q |
| 15 | ✗ | ✗ | ✓ | ✓ | ✓ | B 失败 → Q | 401 | B 无效，Q 也无效不继续回退 |
| 16 | ✗ | ∅ | ✓ | ✓ | ✓ | B 失败 → Q 空 → H | 200 | B 无效，Q 为空回退到 H |
| 17 | - | ✓ | - | - | ✓ | Q | 200 | Q 优先于 C |
| 18 | - | ✗ | - | - | ✓ | Q | 401 | Q 非空无效，**不回退**到 C |
| 19 | - | ∅ | - | - | ✓ | Q 空 → C | 200 | Q 为空，回退到 C |
| 20 | ✓ | ✗ | ✗ | ✗ | ✗ | B | 200 | B 有效，Token 无效不影响 |
| 21 | ✗ | ✗ | ✗ | ✗ | ✗ | B 失败 → Q | 401 | 全部无效 |

### 5.2 RequireApplicationToken 组合矩阵（401 vs 403 差异）

| 序号 | Basic (B) | Query (Q) | 实际使用 | 结果 | 说明 |
|------|-----------|-----------|----------|------|------|
| 1 | - | ✓ (App) | Q | 200 | App Token 有效 |
| 2 | - | ✓ (Client) | Q | 401 | Client Token 不是 App Token，且无有效 B |
| 3 | - | ∅ | Q 空 → 无 Token | 401 | Q 为空，继续检查其他来源，最终无有效 Token |
| 4 | ✓ (用户) | - | Q 无 → B 检查 | 403 | 无 App Token，但 B 有效 |
| 5 | ✗ | - | Q 无 → B 检查 | 401 | 无 App Token，且 B 无效 |
| 6 | ✓ (用户) | ✗ | Q 无效 → B 检查 | 403 | Q 非空无效，但 B 有效 → 返回 403 |
| 7 | ✓ (用户) | ∅ | Q 空 → 其他 → B 检查 | 403 | Q 为空回退，最终无有效 Token，B 有效 → 403 |
| 8 | ✓ (用户) | ✓ (Client) | Q | 403 | Q 是 Client Token 无效，B 有效 → 返回 403 |
| 9 | ✗ | ✓ (Client) | Q | 401 | Q 是 Client Token 无效，B 也无效 → 返回 401 |

**关键更正 1**：高优先级 Token 无效（非空）但 Basic Auth 有效时，返回 **403**，不是 401！

**关键更正 2**：高优先级 Token **为空**值时，会继续回退到下一优先级！

### 5.3 RequireAdmin 组合矩阵（403 场景）

| 序号 | Basic (B) | Query (Q) | 实际使用 | 结果 | 说明 |
|------|-----------|-----------|----------|------|------|
| 1 | ✓ (管理员) | ✓ | B | 200 | 管理员认证成功 |
| 2 | ✓ (普通用户) | ✓ | B | 403 | 认证成功但无权限 |
| 3 | - | ✓ (普通用户) | Q | 403 | Token 有效但用户非管理员 |
| 4 | - | ✓ (管理员但未提升) | Q | 403 | session not elevated |
| 5 | - | ✗ (管理员) | Q | 401 | Token 非空无效 |
| 6 | - | ∅ | Q 空 → 其他 | 401 | Q 为空，无其他有效 Token |
| 7 | ✓ (普通用户) | ✗ | B | 403 | B 有效但非管理员 |
| 8 | ✗ | ✓ (管理员但未提升) | B 失败 → Q | 403 | B 无效，Q 有效但未提升 |
| 9 | ✗ | ∅ | B 失败 → Q 空 → 其他 | 401 | B 无效，Q 为空无其他有效 Token |

## 6. 典型场景分析

### 场景 1：高优先级非空无效，低优先级有效

**请求**：
```
GET /message?token=InvalidToken
Cookie: gotify-client-token=ValidClientToken
```

**访问**：RequireClient 端点

**结果**：
- Query 参数命中 `InvalidToken`（非空）
- 验证失败
- **不回退**到 Cookie
- 返回 401

### 场景 2：高优先级值为空，低优先级有效

**请求**：
```
GET /message?token=
Cookie: gotify-client-token=ValidClientToken
```

**访问**：RequireClient 端点

**结果**：
- Query 参数值为空
- 检测到 `token == ""`，**继续回退**到 Cookie
- Cookie Token 验证成功
- 返回 200

### 场景 3：高优先级 Token 非空无效 + 有效 Basic Auth

**请求**：
```
POST /message?token=InvalidAppToken
Authorization: Basic valid-user-credentials
```

**访问**：RequireApplicationToken 端点

**结果**：
- Application Token 验证失败
- 检查 Basic Auth → 有效用户认证
- 返回 **403**（不是 401）
- 原因：认证有效但不允许用户认证访问应用端点

### 场景 4：高优先级 Token 值为空 + 有效 Basic Auth

**请求**：
```
POST /message?token=
Authorization: Basic valid-user-credentials
```

**访问**：RequireApplicationToken 端点

**结果**：
- Query Token 为空，继续检查其他 Token 来源
- 无其他有效 Token
- 检查 Basic Auth → 有效用户认证
- 返回 **403**

### 场景 5：请求同时包含 Cookie 和 Header Token

**请求**：
```
Cookie: gotify-client-token=CookieToken123
X-Gotify-Key: HeaderToken456
```

**结果**：
- 优先使用 `X-Gotify-Key` 中的 `HeaderToken456`
- Cookie 中的 Token 被忽略

### 场景 6：请求同时包含 Basic Auth 和 Client Token

**请求**：
```
Authorization: Basic dXNlcjE6cGFzc3dvcmQ=
X-Gotify-Key: ClientToken123
```

**结果**（RequireClient 端点）：
- 优先验证 Basic Auth
- 如果用户凭据有效 → 使用用户身份，Token 被忽略
- 如果用户凭据无效 → 回退到 Token 验证

### 场景 7：Cookie 中的 Token 是 Application Token

**请求**：
```
Cookie: gotify-client-token=AppToken_A123
```

**访问**：`POST /message`（RequireApplicationToken）

**结果**：
- 从 Cookie 提取 Token
- 验证为有效 Application Token
- 认证成功

### 场景 8：Cookie 中的 Token 是 Client Token，访问应用端点

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

### 场景 9：同时提供 Query Token（空）和 Cookie Token（有效）

**请求**：
```
GET /message?token=
Cookie: gotify-client-token=ValidCookieToken
```

**结果**：
- Query 参数值为空
- 检测到空值，**继续回退**到 Cookie
- Cookie Token 验证成功
- 返回 200

### 场景 10：有效 Basic Auth + 无效 Token

**请求**：
```
Authorization: Basic valid-user-credentials
X-Gotify-Key: InvalidToken
```

**结果**（RequireClient 端点）：
- Basic Auth 验证成功
- Token 被完全忽略
- 返回 200

### 场景 11：无效 Basic Auth + 有效 Token

**请求**：
```
Authorization: Basic invalid-credentials
X-Gotify-Key: ValidClientToken
```

**结果**（RequireClient 端点）：
- Basic Auth 验证失败
- 回退到 Token 验证
- Token 有效 → 返回 200

### 场景 12：Client Token 有效但非管理员，访问管理员端点

**请求**：
```
X-Gotify-Key: ValidClientTokenOfNormalUser
```

**访问**：RequireAdmin 端点

**结果**：
- Client Token 验证成功
- 检查关联用户是否为管理员 → 否
- 返回 403

## 7. Token 类型区分

系统中有三种 Token 类型，通过前缀区分（`auth/token.go:11-13`）：
- **Application Token**：`A` 前缀
- **Client Token**：`C` 前缀  
- **Plugin Token**：`P` 前缀

**注意**：Cookie 中可以存储任意类型的 Token，系统会根据 Token 内容自动识别类型。

## 8. Cookie 刷新机制

当 Token 通过 Cookie 传递且验证成功时，系统会自动刷新 Cookie：

- 刷新条件：`LastUsed` 为空或超过 5 分钟（`auth/authentication.go:159`）
- 有效期：7 天（`CookieMaxAge = 7 * 24 * 60 * 60`）
- Cookie 属性：`HttpOnly=true`、`SameSite=Strict`、`Secure`（根据配置）

## 9. 认证状态流转

### 9.1 Token 提取流程

```
请求到达
    ↓
┌───────────────────────────────────────────────┐
│  检查 Query 参数 ?token=                        │
└───────────────────────────────────────────────┘
    ↓
    ├────── 非空 ──────┐
    ↓                   ↓
┌─────────────┐   ┌──────────────────────────────┐
│ 验证 Token? │   │ 检查 Header X-Gotify-Key      │
└─────────────┘   └──────────────────────────────┘
    ↓ 成功              ↓
    ↓ 200              ├────── 非空 ──────┐
    ↓                  ↓                   ↓
┌─────────────┐   ┌─────────────┐   ┌──────────────────────────────┐
│ 验证失败?   │   │ 验证 Token? │   │ 检查 Header Authorization     │
│ 返回 401    │   └─────────────┘   └──────────────────────────────┘
└─────────────┘       ↓ 成功              ↓
                      ↓ 200              ├────── 非空 ──────┐
                      ↓                  ↓                   ↓
                   ┌─────────────┐   ┌─────────────┐   ┌──────────────────────┐
                   │ 验证失败?   │   │ 验证 Token? │   │ 检查 Cookie           │
                   │ 返回 401    │   └─────────────┘   └──────────────────────┘
                   └─────────────┘       ↓ 成功              ↓
                                         ↓ 200              ├────── 非空 ──────┐
                                         ↓                  ↓                   ↓
                                      ┌─────────────┐   ┌─────────────┐   ┌────────────┐
                                      │ 验证失败?   │   │ 验证 Token? │   │ 返回空字符串│
                                      │ 返回 401    │   └─────────────┘   └────────────┘
                                      └─────────────┘       ↓ 成功
                                                           ↓ 200
                                                           ↓
                                                        ┌─────────────┐
                                                        │ 验证失败?   │
                                                        │ 返回 401    │
                                                        └─────────────┘
```

### 9.2 RequireClient / RequireAdmin / RequireElevatedClient 流程

```
请求到达
    ↓
┌─────────────────────────────────────────┐
│  检查 Basic Auth                         │
└─────────────────────────────────────────┘
    ↓
    ├─────────────成功─────────────┐
    ↓                               ↓
┌─────────────┐              ┌─────────────────┐
│  验证权限？  │              │  注册用户到上下文  │
└─────────────┘              └─────────────────┘
    ↓ 失败                          ↓
    ↓ 403                       ┌──────┐
    ↓                           │ 200  │
┌─────────────────────────────────┐    └──────┘
│  按优先级提取 Token              │
│  Query → Header → Auth → Cookie │
│  非空锁定，空值回退              │
└─────────────────────────────────┘
    ↓
┌─────────────┐
│ 验证 Token? │
└─────────────┘
    ↓
    ├───────────成功───────────┐
    ↓                           ↓
┌─────────────┐          ┌─────────────────────┐
│ 检查权限？   │          │ 注册客户端到上下文     │
└─────────────┘          └─────────────────────┘
    ↓ 失败                       ↓
    ↓ 403                     ┌──────┐
    ↓                         │ 200  │
┌─────────────┐               └──────┘
│ 返回 401    │
└─────────────┘
```

### 9.3 RequireApplicationToken 流程

```
请求到达
    ↓
┌─────────────────────────────────┐
│  按优先级提取 Token              │
│  Query → Header → Auth → Cookie │
│  非空锁定，空值回退              │
└─────────────────────────────────┘
    ↓
┌───────────────────┐
│ 验证 App Token?   │
└───────────────────┘
    ├──────────成功──────────┐
    ↓                         ↓
┌─────────────────────┐   ┌──────┐
│ 注册应用到上下文      │   │ 200  │
└─────────────────────┘   └──────┘
    ↓ 失败
┌───────────────────┐
│ 检查 Basic Auth?   │
└───────────────────┘
    ├─────────有效─────────┐         ┌───────┐
    ↓                       ↘         │ 403   │
┌─────────────────────┐     ↘        └───────┘
│ 认证有效但类型错误   │
└─────────────────────┘
    ↓ 无效
┌─────────────┐
│ 返回 401    │
└─────────────┘
```

## 10. 关键代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| Token 提取优先级（空值判断） | `auth/authentication.go` | 205-216 |
| 认证评估逻辑 | `auth/authentication.go` | 84-112 |
| RequireClient 逻辑 | `auth/authentication.go` | 52-54 |
| RequireAdmin 逻辑 | `auth/authentication.go` | 46-48 |
| RequireApplicationToken 逻辑 | `auth/authentication.go` | 62-76 |
| 403 触发逻辑（Application） | `auth/authentication.go` | 70-73 |
| Cookie 设置 | `auth/cookie.go` | 12-22 |
| Token 生成规则 | `auth/token.go` | 37-55 |
