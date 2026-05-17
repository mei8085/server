# Gotify OIDC 登录与 Session 创建机制分析

## 1. 整体架构概述

Gotify 的认证体系采用 **Client Token** 作为核心会话凭证。无论是本地登录还是 OIDC 外部登录，最终都会创建一个 `Client` 记录作为会话载体，其 `Token` 字段通过 Cookie（浏览器场景）或直接返回（原生应用场景）进行持久化。

**重要说明**：OIDC 登录流程中，服务端在登录完成前会维护内存状态。`pendingSessions` 作为临时缓存，承担存储 `state` 参数与待处理会话上下文的功能。只有在登录成功后，最终状态才会持久化到数据库。

## 2. OIDC 登录回调流程详解

### 2.1 登录流程入口

OIDC 登录分为三种场景：

**场景 A：浏览器登录** (`/auth/oidc/login`)
- 接收 `name` 参数作为客户端名称
- 生成随机 `state` 并存储到 `pendingSessions`
- 重定向到 OIDC 提供商授权页面

**场景 B：原生应用授权** (`/auth/oidc/external/authorize`)
- 接收 PKCE `code_challenge`、`redirect_uri`、`name`
- 生成 `state` 并存储 `pendingOIDCSession`（包含 `RedirectURI`）
- 返回授权 URL 给应用，由应用打开浏览器

**场景 C：会话提升** (`/auth/oidc/elevate`)
- 接收 `id`（Client ID）和 `durationSeconds` 参数
- 生成 `state` 并存储 `pendingOIDCSession`（仅包含 `Elevate` 信息）
- 重定向到 OIDC 提供商进行重新认证

### 2.2 State 生成与存储机制

```go
// api/oidc.go:386-392
func (a *OIDCAPI) generateState() (string, error) {
    nonce := make([]byte, 20)
    if _, err := rand.Read(nonce); err != nil {
        return "", err
    }
    return hex.EncodeToString(nonce), nil
}
```

**State 特性：**
- 20 字节随机数 + Hex 编码 = 40 字符字符串
- 存储在 `decaymap.DecayMap` 中，有效期 10 分钟
- 键：state 字符串，值：`pendingOIDCSession` 结构体
- **内存存储**：服务重启后所有 pending 会话丢失，需要重新发起登录

**pendingOIDCSession 结构：**
```go
type pendingOIDCSession struct {
    RedirectURI string    // 原生应用重定向地址
    ClientName  string    // 客户端名称
    CreatedAt   time.Time // 创建时间
    Elevate     *pendingElevation // 会话提升请求（可选）
}
```

**pendingSessions 的核心作用：**
1. **CSRF 防护**：通过 state 参数防止跨站请求伪造
2. **会话上下文缓存**：保存登录发起时的参数（客户端名称、重定向地址等）
3. **会话提升标识**：标记当前流程是普通登录还是已有会话的提升操作

### 2.3 回调处理流程

#### 浏览器回调 (`/auth/oidc/callback`)

```go
// api/oidc.go:202-233
func (a *OIDCAPI) CallbackHandler() gin.HandlerFunc {
    callback := func(w http.ResponseWriter, r *http.Request, tokens *oidc.Tokens[*oidc.IDTokenClaims], state string, provider rp.RelyingParty, info *oidc.UserInfo) {
        // 1. 解析用户信息，自动注册（如果启用）
        user, status, err := a.resolveUser(info)
        
        // 2. State 校验并取出待处理会话（一次性消费）
        session, ok := a.popPendingSession(state)
        
        // 3. 如果是提升会话请求，走特殊分支
        if session.Elevate != nil {
            a.handleElevationCallback(w, session.Elevate, user)
            return
        }
        
        // 4. 创建 Client 记录作为会话载体
        client, err := a.createClient(session.ClientName, user.ID)
        
        // 5. 设置会话 Cookie
        auth.SetCookie(w, client.Token, auth.CookieMaxAge, a.SecureCookie)
        
        // 6. 重定向到首页
        w.Header().Set("Location", "../../")
        w.WriteHeader(http.StatusTemporaryRedirect)
    }
    return gin.WrapF(rp.CodeExchangeHandler(rp.UserinfoCallback(callback), a.Provider))
}
```

**注意**：浏览器回调中，`resolveUser` 在 `popPendingSession` **之前**执行。如果用户解析成功但 state 校验失败，用户可能已创建但 pendingSession 未被消费。

#### 原生应用 Token 交换 (`/auth/oidc/external/token`)

```go
// api/oidc.go:345-384
func (a *OIDCAPI) ExternalTokenHandler(ctx *gin.Context) {
    var req model.OIDCExternalTokenRequest
    
    // 1. State 校验（同样一次性消费）
    session, ok := a.popPendingSession(req.State)
    
    // 2. 使用 code_verifier 进行 PKCE 验证并换取 Token
    tokens, err := rp.CodeExchange[*oidc.IDTokenClaims](ctx.Request.Context(), req.Code, a.Provider, exchangeOpts...)
    
    // 3. 获取用户信息，自动注册
    info, err := rp.Userinfo[*oidc.UserInfo](...)
    user, status, resolveErr := a.resolveUser(info)
    
    // 4. 创建 Client
    client, err := a.createClient(session.ClientName, user.ID)
    
    // 5. 直接返回 Token（不设置 Cookie）
    ctx.JSON(http.StatusOK, &model.OIDCExternalTokenResponse{
        Token: client.Token,
        User:  &model.UserExternal{ID: user.ID, Name: user.Name, Admin: user.Admin},
    })
}
```

**注意**：原生应用流程中，`popPendingSession` 在最开始执行，一旦通过 state 就被立即消费，后续步骤失败不会残留 state。

### 2.4 State 校验核心逻辑

```go
// api/oidc.go:435-441
func (a *OIDCAPI) popPendingSession(key string) (*pendingOIDCSession, bool) {
    session, ok := a.pendingSessions.Pop(key)
    if ok && time.Since(session.CreatedAt) < pendingSessionMaxAge {
        return session, true
    }
    return nil, false
}
```

**关键安全特性：**
1. **一次性消费**：`Pop` 操作取出后立即从 Map 中删除，防止重放攻击
2. **超时校验**：创建时间超过 10 分钟的会话即使存在也无效
3. **双重校验**：DecayMap 自身有过期机制 + 代码层面再次校验时间

## 3. 用户解析与自动注册

```go
// api/oidc.go:394-422
func (a *OIDCAPI) resolveUser(info *oidc.UserInfo) (*model.User, int, error) {
    // 1. 从 OIDC Claims 中提取用户名（可配置 claim 名称）
    usernameRaw, ok := info.Claims[a.UsernameClaim]
    username := fmt.Sprint(usernameRaw)
    
    // 2. 查找本地用户
    user, err := a.DB.GetUserByName(username)
    
    // 3. 用户不存在时自动注册（如果启用）
    if user == nil {
        if !a.AutoRegister {
            return nil, http.StatusForbidden, fmt.Errorf("user does not exist and auto-registration is disabled")
        }
        user = &model.User{Name: username, Admin: false, Pass: nil}
        if err := a.DB.CreateUser(user); err != nil {
            // ...
        }
    }
    return user, 0, nil
}
```

**注意点：**
- OIDC 用户的 `Pass` 字段为 `nil`，无法通过本地密码登录
- 默认创建的用户不是管理员

## 4. Client 会话创建逻辑

```go
// api/oidc.go:424-433
func (a *OIDCAPI) createClient(name string, userID uint) (*model.Client, error) {
    elevatedUntil := time.Now().Add(model.DefaultElevationDuration)
    client := &model.Client{
        Name:          name,
        Token:         auth.GenerateNotExistingToken(generateClientToken, ...),
        UserID:        userID,
        ElevatedUntil: &elevatedUntil,
    }
    return client, a.DB.CreateClient(client)
}
```

**Client 作为会话的核心属性：**
| 字段 | 说明 |
|------|------|
| `Token` | 唯一认证凭证，用于 API 调用 |
| `UserID` | 关联的用户 ID |
| `ElevatedUntil` | 会话提升有效期，默认 1 小时 |
| `LastUsed` | 最后使用时间，用于自动续期 Cookie |

## 5. Cookie 设置与认证流程

```go
// auth/cookie.go:12-22
func SetCookie(w http.ResponseWriter, token string, maxAge int, secure bool) {
    http.SetCookie(w, &http.Cookie{
        Name:     "gotify-client-token",
        Value:    token,
        Path:     "/",
        MaxAge:   maxAge,           // 7 天
        Secure:   secure,           // 依配置而定
        HttpOnly: true,             // 禁止 JS 访问
        SameSite: http.SameSiteStrictMode, // 严格同源策略
    })
}
```

**中间件认证流程** (`auth/authentication.go`):
1. 依次从查询参数、`X-Gotify-Key` 头、`Authorization: Bearer` 头、Cookie 中读取 Token
2. 从数据库查询 `Client` 记录
3. 如果距离上次使用超过 5 分钟，更新 `LastUsed` 并刷新 Cookie 有效期
4. 执行额外检查（权限、会话提升状态等）
5. 将 Client 注册到 Gin Context，供后续 Handler 使用

## 6. 本地登录与 OIDC 登录对比

### 6.1 本地登录流程 (`/auth/local/login`)

```go
// api/session.go:56-98
func (a *SessionAPI) Login(ctx *gin.Context) {
    // 1. 从 Basic Auth 中提取用户名密码
    name, pass, ok := ctx.Request.BasicAuth()
    
    // 2. 查询用户并验证密码哈希
    user, err := a.DB.GetUserByName(name)
    if user == nil || !password.ComparePassword(user.Pass, []byte(pass)) {
        ctx.AbortWithError(401, errors.New("invalid credentials"))
        return
    }
    
    // 3. 创建 Client 记录（与 OIDC 相同）
    elevatedUntil := time.Now().Add(model.DefaultElevationDuration)
    client := model.Client{
        Name:          clientParams.Name,
        Token:         auth.GenerateNotExistingToken(...),
        UserID:        user.ID,
        ElevatedUntil: &elevatedUntil,
    }
    
    // 4. 设置 Cookie
    auth.SetCookie(ctx.Writer, client.Token, auth.CookieMaxAge, a.SecureCookie)
    
    // 5. 返回用户信息（包含 ClientID 和 ElevatedUntil）
    ctx.JSON(200, &model.CurrentUserExternal{...})
}
```

### 6.2 详细差异对比表

| 对比维度 | 本地登录 (`/auth/local/login`) | OIDC 浏览器登录 (`/auth/oidc/callback`) | OIDC 原生应用 (`/auth/oidc/external/token`) |
|---------|-------------------------------|----------------------------------------|--------------------------------------------|
| **认证方式** | HTTP Basic Auth (用户名密码) | OIDC Code Flow + Userinfo | OIDC Code Flow + PKCE + Userinfo |
| **State 机制** | 无 | 有，10 分钟有效期，一次性消费 | 有，10 分钟有效期，一次性消费 |
| **用户存在性** | 必须已存在，验证密码 | 可自动注册（配置项控制） | 可自动注册（配置项控制） |
| **密码字段** | 必须有，验证 bcrypt 哈希 | `Pass = nil`，无法本地登录 | `Pass = nil`，无法本地登录 |
| **Client 创建** | 是，走相同 `CreateClient` 逻辑 | 是，走相同 `CreateClient` 逻辑 | 是，走相同 `CreateClient` 逻辑 |
| **Cookie 设置** | 是，7 天有效期 | 是，7 天有效期 | 否，直接 JSON 返回 Token |
| **响应格式** | `CurrentUserExternal`（含 user + client 信息） | 307 重定向到首页 | `OIDCExternalTokenResponse`（仅 token + 基础用户信息） |
| **会话提升** | 登录即默认提升 1 小时，另有独立提升接口 | 登录即默认提升 1 小时，另有独立 OIDC 提升流程 | 登录即默认提升 1 小时 |
| **CSRF 防护** | 依赖 SameSite Cookie | State 参数 + SameSite Cookie | State 参数 + PKCE |

### 6.3 核心相同点

**三种登录方式最终产生的结果完全一致：**
1. 都在数据库中创建一条 `Client` 记录
2. `Client.Token` 生成算法完全相同
3. `ElevatedUntil` 默认值相同（1 小时）
4. 认证中间件对 Token 的处理逻辑完全一致（不区分登录来源）

### 6.4 失败路径对比分析

#### 本地登录失败场景

| 失败条件 | HTTP 状态码 | 返回方式 | 是否写 Cookie | 其他影响 | 代码依据 |
|---------|-----------|---------|-------------|---------|---------|
| 缺少 Basic Auth 头 | 401 | `ctx.AbortWithError`，JSON 错误 | 否 | 无 | session.go:58-61 |
| 用户不存在 | 401 | `ctx.AbortWithError`，JSON 错误 | 否 | 无 | session.go:63-70 |
| 密码验证失败 | 401 | `ctx.AbortWithError`，JSON 错误 | 否 | 无 | session.go:68-70 |
| 缺少 `name` 表单参数 | 400 | Gin 绑定失败，默认错误 | 否 | 无 | session.go:73-76 |
| Client 创建数据库错误 | 500 | `successOrAbort`，JSON 错误 | 否 | 无 | session.go:85-87 |

**本地登录失败特点：**
- 所有失败均返回 JSON 格式错误信息
- 失败时绝对不会设置 Cookie
- 无任何副作用（不会产生半完成状态）

#### OIDC 浏览器回调失败场景

| 失败条件 | HTTP 状态码 | 返回方式 | 是否写 Cookie | 其他影响 | 代码依据 |
|---------|-----------|---------|-------------|---------|---------|
| OIDC 令牌交换失败（如 code 无效） | 由 OIDC 库决定 | `http.Error`，纯文本错误 | 否 | 执行到回调前失败，`pendingSession` 未消费 | oidc.go:232 |
| 用户信息获取失败 | 500 | `http.Error`，纯文本错误 | 否 | 执行到回调前失败，`pendingSession` 未消费 | oidc.go:232 |
| 用户名 claim 缺失 | 500 | `http.Error`，纯文本错误 | 否 | `resolveUser` 失败，`pendingSession` **未消费** | oidc.go:204-208 |
| 用户不存在且自动注册关闭 | 403 | `http.Error`，纯文本错误 | 否 | `resolveUser` 失败，`pendingSession` **未消费** | oidc.go:204-208 |
| 用户创建数据库错误 | 500 | `http.Error`，纯文本错误 | 否 | `resolveUser` 失败，`pendingSession` **未消费** | oidc.go:204-208 |
| State 无效（Map 中不存在） | 400 | `http.Error`，纯文本错误 | 否 | **用户可能已创建**（resolveUser 在 popPendingSession 之前），`pendingSession` **未消费**（Pop 返回 false） | oidc.go:209-213, 435-441 |
| State 有效但已过期（存在但超时） | 400 | `http.Error`，纯文本错误 | 否 | **用户可能已创建**（resolveUser 在 popPendingSession 之前），`pendingSession` **已消费**（Pop 已从 Map 删除，后因超时而返回失败） | oidc.go:209-213, 435-441 |
| Client 创建数据库错误 | 500 | `http.Error`，纯文本错误 | 否 | 用户已创建，`pendingSession` 已消费 | oidc.go:220-224 |

**OIDC 浏览器回调失败特点：**
- 所有失败均返回纯文本错误信息（非 JSON）
- 失败时不会设置 Cookie
- **重要**：执行顺序 `resolveUser` → `popPendingSession` 导致副作用差异：
  - `resolveUser` 前失败（令牌交换、用户信息获取）：`pendingSession` 未消费
  - `resolveUser` 失败：用户未创建，`pendingSession` **未消费**
  - State 无效：**用户可能已创建**，`pendingSession` **未消费**（Pop 返回 false）
  - State 有效但已过期：**用户可能已创建**，`pendingSession` **已消费**（Pop 已删除但超时）
  - Client 创建失败：用户已创建，`pendingSession` 已消费

#### OIDC 原生应用 Token 交换失败场景

| 失败条件 | HTTP 状态码 | 返回方式 | 是否写 Cookie | 其他影响 | 代码依据 |
|---------|-----------|---------|-------------|---------|---------|
| 请求参数绑定失败 | 400 | `ctx.AbortWithError`，JSON 错误 | 否 | 无 | oidc.go:347-350 |
| State 无效（Map 中不存在） | 400 | `ctx.AbortWithError`，JSON 错误 | 否 | `pendingSession` **未消费**（Pop 返回 false） | oidc.go:351-355, 435-441 |
| State 有效但已过期（存在但超时） | 400 | `ctx.AbortWithError`，JSON 错误 | 否 | `pendingSession` **已消费**（Pop 已从 Map 删除，后因超时而返回失败） | oidc.go:351-355, 435-441 |
| 令牌交换失败（PKCE 验证失败等） | 401 | `ctx.AbortWithError`，JSON 错误 | 否 | `pendingSession` 已消费 | oidc.go:360-364 |
| 用户信息获取失败 | 500 | `ctx.AbortWithError`，JSON 错误 | 否 | `pendingSession` 已消费 | oidc.go:365-369 |
| 用户名 claim 缺失 | 500 | `ctx.AbortWithError`，JSON 错误 | 否 | `pendingSession` 已消费 | oidc.go:370-374 |
| 用户不存在且自动注册关闭 | 403 | `ctx.AbortWithError`，JSON 错误 | 否 | `pendingSession` 已消费 | oidc.go:370-374 |
| 用户创建数据库错误 | 500 | `ctx.AbortWithError`，JSON 错误 | 否 | `pendingSession` 已消费 | oidc.go:370-374 |
| Client 创建数据库错误 | 500 | `ctx.AbortWithError`，JSON 错误 | 否 | `pendingSession` 已消费，用户已创建 | oidc.go:375-379 |

**OIDC 原生应用失败特点：**
- 所有失败均返回 JSON 格式错误
- 从不设置 Cookie（该 API 本身就不写 Cookie）
- **popPendingSession 行为拆分**（oidc.go:435-441）：
  - 先执行 `Pop` 从 Map 中删除
  - 再检查 `time.Since(session.CreatedAt) < pendingSessionMaxAge` 时间窗
  - 因此：无效 state（不存在）→ 不消费；过期 state（存在但超时）→ 已消费
- `popPendingSession` 在最开始执行，除了参数绑定失败和完全无效 state 外，后续失败都会导致 `pendingSession` 已消费
- 用户解析失败时，用户**可能已创建**（如果是新用户且自动注册开启且数据库创建成功）

### 6.5 会话提升分支对比

#### 本地登录的会话提升

本地登录有**两种**提升机制：
1. **登录时自动提升**：每次创建新 Client 时自动设置 `ElevatedUntil` 为当前时间 + 1 小时
2. **独立提升接口**：`POST /client/{id}/elevate` - 可对**已有 Client** 进行提升（需要当前会话已是提升状态）

#### 两种提升接口详细对比

| 对比项 | `POST /client/{id}/elevate` | `GET /auth/oidc/elevate` |
|-------|----------------------------|-------------------------|
| **前置条件** | 需要 `RequireElevatedClient` 中间件（已有提升会话） | 无需已有会话，但需提供 `id`（Client ID）参数 |
| **路由组** | `clientElevated` 组（router.go:220-227） | 无特殊中间件，与普通 OIDC 登录相同 |
| **认证方式** | Basic Auth 或 Client Token（必须已提升） | 完整 OIDC 授权码流程（重新登录） |
| **State 机制** | 无 | 有，完整 state 生命周期 |
| **是否创建新 Client** | 否，仅更新 `ElevatedUntil` | 否，仅更新 `ElevatedUntil` |
| **是否设置 Cookie** | 否 | 否 |
| **是否返回 Token** | 否，返回 204 No Content | 否，返回 HTML 页面提示关闭标签页 |
| **响应格式** | 204 无内容 | 200 HTML 页面 |
| **提升时长** | 自定义 `durationSeconds` | 自定义 `durationSeconds` |
| **典型使用场景** | 已登录用户延长敏感操作的提升有效期 | 会话已过期，通过 OIDC 重新认证来提升 |
| **代码位置** | client.go:287-312 | oidc.go:160-172, 235-266 |

**`/client/{id}/elevate` 核心逻辑**：
```go
// api/client.go:287-312
func (a *ClientAPI) ElevateClient(ctx *gin.Context) {
    withID(ctx, "id", func(id uint) {
        var params model.ElevateRequest
        // 绑定参数...
        
        // 验证 Client 存在且属于当前用户
        client, err := a.DB.GetClientByID(id)
        if client == nil || client.UserID != auth.GetUserID(ctx) {
            ctx.AbortWithError(404, errors.New("client not found"))
            return
        }

        // 仅更新 ElevatedUntil 字段，不创建新 Client
        elevatedUntil := time.Now().Add(time.Duration(params.DurationSeconds) * time.Second)
        a.DB.UpdateClientElevatedUntil(client.ID, &elevatedUntil)

        ctx.Status(204)
    })
}
```

**两种提升方式的本质区别**：
- `/client/{id}/elevate` 是**"用提升会话来续期"** - 鸡生蛋
- `/auth/oidc/elevate` 是**"用重新认证来获得提升"** - 蛋生鸡

### 6.6 会话提升失败路径对比

| 失败条件 | `/client/{id}/elevate` | `/auth/oidc/elevate`（回调阶段） |
|---------|-----------------------|--------------------------------|
| 未认证或认证无效 | 401 中间件拦截 | 401 中间件拦截（OIDC 流程外） |
| 会话未提升 | 403 "session not elevated" | 不适用（走完整 OIDC 流程） |
| Client ID 无效 | 404 "client not found" | 404 "client not found"（oidc.go:241-243） |
| Client 不属于当前用户 | 404 "client not found" | 404 "client not found"（oidc.go:241-243） |
| 数据库查询错误 | 500 | 500 "database error"（oidc.go:237-239） |
| 数据库更新错误 | 500 | 500 "failed to elevate session"（oidc.go:246-248） |
| `pendingSession` 中 `Elevate` 为空 | 不适用 | 不会发生（入口已设置 Elevate 信息） |

## 7. 登出流程

```go
// api/session.go:123-138
func (a *SessionAPI) Logout(ctx *gin.Context) {
    // 1. 清除 Cookie
    auth.SetCookie(ctx.Writer, "", -1, a.SecureCookie)
    
    // 2. 从 Context 获取当前 Client
    client := auth.GetClient(ctx)
    
    // 3. 从数据库删除 Client 记录（物理删除）
    a.NotifyDeleted(client.UserID, client.Token)
    a.DB.DeleteClientByID(client.ID)
    
    ctx.Status(200)
}
```

**关键点：**
- 登出会**物理删除**数据库中的 Client 记录，而非只是标记失效
- 这意味着该 Token 永久失效，无法再使用
- 所有登录方式共享此登出逻辑

## 8. 安全设计总结

1. **会话状态分层管理**：
   - 登录前：`pendingSessions` 内存缓存管理 state 和会话上下文（10 分钟超时）
   - 登录后：数据库 `Client` 表持久化会话，配合 Cookie 机制

2. **一次性 State**：防止 OAuth2 重放攻击

3. **PKCE**：原生应用场景防止授权码劫持

4. **HttpOnly Cookie**：防止 XSS 窃取 Token

5. **SameSite Strict**：防止 CSRF 攻击

6. **Token 随机生成**：确保不可预测性

7. **自动过期机制**：State 10 分钟，Cookie 7 天，提升会话默认 1 小时

8. **会话提升隔离**：提供两种独立的提升机制应对不同场景
   - `/client/{id}/elevate`：已提升会话续期
   - `/auth/oidc/elevate`：通过重新认证获得提升
