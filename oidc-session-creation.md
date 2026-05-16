# Gotify OIDC 登录与 Session 创建机制分析

## 1. 整体架构概述

Gotify 的认证体系采用 **Client Token** 作为核心会话凭证。无论是本地登录还是 OIDC 外部登录，最终都会创建一个 `Client` 记录作为会话载体，其 `Token` 字段通过 Cookie（浏览器场景）或直接返回（原生应用场景）进行持久化。

## 2. OIDC 登录回调流程详解

### 2.1 登录流程入口

OIDC 登录分为两种场景：

**场景 A：浏览器登录** (`/auth/oidc/login`)
- 接收 `name` 参数作为客户端名称
- 生成随机 `state` 并存储到 `pendingSessions`
- 重定向到 OIDC 提供商授权页面

**场景 B：原生应用授权** (`/auth/oidc/external/authorize`)
- 接收 PKCE `code_challenge`、`redirect_uri`、`name`
- 生成 `state` 并存储 `pendingOIDCSession`（包含 `RedirectURI`）
- 返回授权 URL 给应用，由应用打开浏览器

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

**pendingOIDCSession 结构：**
```go
type pendingOIDCSession struct {
    RedirectURI string    // 原生应用重定向地址
    ClientName  string    // 客户端名称
    CreatedAt   time.Time // 创建时间
    Elevate     *pendingElevation // 会话提升请求（可选）
}
```

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
| **会话提升** | 登录即默认提升 24 小时 | 登录即默认提升 24 小时 | 登录即默认提升 24 小时 |
| **CSRF 防护** | 依赖 SameSite Cookie | State 参数 + SameSite Cookie | State 参数 + PKCE |

### 6.3 核心相同点

**三种登录方式最终产生的结果完全一致：**
1. 都在数据库中创建一条 `Client` 记录
2. `Client.Token` 生成算法完全相同
3. `ElevatedUntil` 默认值相同（24 小时）
4. 认证中间件对 Token 的处理逻辑完全一致（不区分登录来源）

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

1. **无状态设计**：服务端不维护内存会话，所有状态都在数据库中
2. **一次性 State**：防止 OAuth2 重放攻击
3. **PKCE**：原生应用场景防止授权码劫持
4. **HttpOnly Cookie**：防止 XSS 窃取 Token
5. **SameSite Strict**：防止 CSRF 攻击
6. **Token 随机生成**：确保不可预测性
7. **自动过期机制**：State 10 分钟，Cookie 7 天，提升会话 24 小时
