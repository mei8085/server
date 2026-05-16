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
| `ElevatedUntil` | 会话提升有效期，默认 24 小时 |
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
| **认证方式** | HTTP Basic Auth (用户名密码) | OIDC Code Flow + UserInfo | OIDC Code Flow + PKCE + UserInfo |
| **State 机制** | 无 | 有，10 分钟有效期，一次性消费 | 有，10 分钟有效期，一次性消费 |
| **用户存在性** | 必须已存在，验证密码 | 可自动注册（配置项控制） | 可自动注册（配置项控制） |
| **密码字段** | 必须有，验证 bcrypt 哈希 | `Pass = nil`，无法本地登录 | `Pass = nil`，无法本地登录 |
| **Client 创建** | 是，走相同 `CreateClient` 逻辑 | 是，走相同 `CreateClient` 逻辑 | 是，走相同 `CreateClient` 逻辑 |
| **Cookie 设置** | 是，7 天有效期 | 是，7 天有效期 | 否，直接 JSON 返回 Token |
| **响应格式** | `CurrentUserExternal`（含 user + client 信息） | 307 重定向到首页 | `OIDCExternalTokenResponse`（仅 token + 基础用户信息） |
| **会话提升** | 登录即默认提升 24 小时 | 登录即默认提升 24 小时，另有独立提升流程 | 登录即默认提升 24 小时 |
| **CSRF 防护** | 依赖 SameSite Cookie | State 参数 + SameSite Cookie | State 参数 + PKCE |

### 6.3 核心相同点

**三种登录方式最终产生的结果完全一致：**
1. 都在数据库中创建一条 `Client` 记录
2. `Client.Token` 生成算法完全相同
3. `ElevatedUntil` 默认值相同（24 小时）
4. 认证中间件对 Token 的处理逻辑完全一致（不区分登录来源）

### 6.4 失败路径对比分析

#### 本地登录失败场景

| 失败条件 | HTTP 状态码 | 返回方式 | 是否写 Cookie | 其他影响 |
|---------|-----------|---------|-------------|---------|
| 缺少 Basic Auth 头 | 401 | `ctx.AbortWithError`，JSON 错误 | 否 | 无 |
| 用户不存在 | 401 | `ctx.AbortWithError`，JSON 错误 | 否 | 无 |
| 密码验证失败 | 401 | `ctx.AbortWithError`，JSON 错误 | 否 | 无 |
| 缺少 `name` 表单参数 | 400 | Gin 绑定失败，默认错误 | 否 | 无 |
| Client 创建数据库错误 | 500 | `successOrAbort`，JSON 错误 | 否 | 无 |

**本地登录失败特点：**
- 所有失败均返回 JSON 格式错误信息
- 失败时绝对不会设置 Cookie
- 无任何副作用（不会产生半完成状态）

#### OIDC 浏览器回调失败场景

| 失败条件 | HTTP 状态码 | 返回方式 | 是否写 Cookie | 其他影响 |
|---------|-----------|---------|-------------|---------|
| OIDC 令牌交换失败（如 code 无效） | 由 OIDC 库决定 | `http.Error`，纯文本错误 | 否 | `state` 已被 OIDC 库消费（若有） |
| 用户信息获取失败 | 500 | `http.Error`，纯文本错误 | 否 | `state` 已消费，`pendingSession` 已删除 |
| 用户名 claim 缺失 | 500 | `http.Error`，纯文本错误 | 否 | 同上 |
| 用户不存在且自动注册关闭 | 403 | `http.Error`，纯文本错误 | 否 | 同上 |
| 用户创建数据库错误 | 500 | `http.Error`，纯文本错误 | 否 | 同上 |
| State 无效或已过期 | 400 | `http.Error`，纯文本错误 | 否 | 无 |
| Client 创建数据库错误 | 500 | `http.Error`，纯文本错误 | 否 | 用户可能已创建（事务边界问题） |

**OIDC 浏览器回调失败特点：**
- 所有失败均返回纯文本错误信息（非 JSON）
- 失败时不会设置 Cookie
- **重要**：失败发生在不同阶段产生不同副作用：
  - State 校验失败：无任何副作用
  - 用户解析失败：`pendingSession` 已删除，需重新发起登录
  - Client 创建失败：用户可能已创建，`pendingSession` 已删除

#### OIDC 原生应用 Token 交换失败场景

| 失败条件 | HTTP 状态码 | 返回方式 | 是否写 Cookie | 其他影响 |
|---------|-----------|---------|-------------|---------|
| 请求参数绑定失败 | 400 | `ctx.AbortWithError`，JSON 错误 | 否 | 无 |
| State 无效或已过期 | 400 | `ctx.AbortWithError`，JSON 错误 | 否 | 无 |
| 令牌交换失败（PKCE 验证失败等） | 401 | `ctx.AbortWithError`，JSON 错误 | 否 | `pendingSession` 已删除 |
| 用户信息获取失败 | 500 | `ctx.AbortWithError`，JSON 错误 | 否 | `pendingSession` 已删除 |
| 用户解析失败（同浏览器） | 403/500 | `ctx.AbortWithError`，JSON 错误 | 否 | `pendingSession` 已删除 |
| Client 创建失败 | 500 | `ctx.AbortWithError`，JSON 错误 | 否 | 用户可能已创建 |

**OIDC 原生应用失败特点：**
- 所有失败均返回 JSON 格式错误
- 从不设置 Cookie（该 API 本身就不写 Cookie）
- 副作用与浏览器回调类似，但响应格式对应用更友好

### 6.5 会话提升分支对比

#### 本地登录的会话提升

本地登录**没有独立的提升流程**。会话提升是登录成功后的自然结果：
- 每次创建新 Client 时自动设置 `ElevatedUntil` 为当前时间 + 24 小时
- 无法对已有 Client 进行提升（必须重新登录创建新 Client）

#### OIDC 的会话提升流程

**提升入口** (`/auth/oidc/elevate`)：
```go
// api/oidc.go:160-172
func (a *OIDCAPI) ElevateHandler(ctx *gin.Context) {
    var elevate pendingElevation
    if err := ctx.BindQuery(&elevate); err != nil {
        return
    }
    state, err := a.generateState()
    // ...
    a.pendingSessions.Set(time.Now(), state, &pendingOIDCSession{
        CreatedAt: time.Now(), 
        Elevate: &elevate  // 标记为提升会话
    })
    rp.AuthURLHandler(func() string { return state }, a.Provider)(ctx.Writer, ctx.Request)
}
```

**提升回调处理** (`handleElevationCallback`)：
```go
// api/oidc.go:235-266
func (a *OIDCAPI) handleElevationCallback(w http.ResponseWriter, elevate *pendingElevation, user *model.User) {
    // 1. 验证 Client 存在且属于当前用户
    client, err := a.DB.GetClientByID(elevate.ClientID)
    if client == nil || client.UserID != user.ID {
        http.Error(w, "client not found", http.StatusNotFound)
        return
    }
    
    // 2. 更新提升过期时间（不创建新 Client）
    elevatedUntil := time.Now().Add(time.Duration(elevate.DurationSeconds) * time.Second)
    if err := a.DB.UpdateClientElevatedUntil(client.ID, &elevatedUntil); err != nil {
        http.Error(w, fmt.Sprintf("failed to elevate session: %v", err), http.StatusInternalServerError)
        return
    }
    
    // 3. 返回成功页面（不设置 Cookie，不返回 Token）
    w.WriteHeader(http.StatusOK)
    w.Header().Add("content-type", "text/html")
    io.WriteString(w, `... 提升成功页面 ...`)
}
```

#### 会话提升对比表

| 对比项 | 本地登录提升 | OIDC 提升流程 |
|-------|------------|--------------|
| **流程性质** | 登录副作用，无独立流程 | 独立完整的 OIDC 授权流程 |
| **是否创建新 Client** | 是（每次登录都创建） | 否（仅更新已有 Client 的 `ElevatedUntil`） |
| **是否设置 Cookie** | 是（新 Client Token） | 否（复用已有 Cookie） |
| **是否返回 Token** | 是（在 JSON 响应中） | 否（仅返回 HTML 提示页面） |
| **提升时长** | 固定 24 小时 | 可自定义 `durationSeconds` |
| **前置条件** | 用户名密码 | 已有有效 Client + OIDC 重新认证 |
| **State 机制** | 无 | 有（完整的 state 生命周期） |

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

7. **自动过期机制**：State 10 分钟，Cookie 7 天，提升会话 24 小时

8. **会话提升隔离**：提升流程独立于登录流程，仅更新权限不创建新会话
