# 反向代理场景下跨域请求处理策略分析报告（最终版）

## 目录
1. [关键代码证据索引（经复核）](#1-关键代码证据索引经复核)
2. [跨域规则的配置来源与加载路径](#2-跨域规则的配置来源与加载路径)
3. [WebSocket 浏览器侧认证限制分析](#3-websocket-浏览器侧认证限制分析)
4. [跨域凭据链路深度分析](#4-跨域凭据链路深度分析)
5. [三场景鉴权矩阵（HTTP vs WebSocket）](#5-三场景鉴权矩阵http-vs-websocket)
6. [可落地部署方案与风险评估](#6-可落地部署方案与风险评估)

---

## 1. 关键代码证据索引（经复核）

| 结论 | 代码文件 | 精确行号 | 验证状态 |
|------|---------|---------|---------|
| WebSocket Origin 只匹配 hostname | `api/stream/stream.go` | 第 203 行 | ✅ 已复核 |
| WebSocket Upgrade 调用 | `api/stream/stream.go` | 第 144 行 | ✅ 已复核 |
| 前端自动构造同源 URL | `ui/src/index.tsx` | 第 20-26 行 | ✅ 已复核 |
| WebSocket URL 从同源 URL 派生 | `ui/src/message/WebSocketStore.ts` | 第 22 行 | ✅ 已复核 |
| Cookie 使用 SameSite=Strict | `auth/cookie.go` | 第 20 行 | ✅ 已复核 |
| Cookie 名称常量 | `auth/cookie.go` | 第 10 行 | ✅ 已复核 |
| CORS 中间件全局挂载 | `router/router.go` | 第 131 行 | ✅ 已复核 |
| CORS 配置未设置 AllowCredentials | `auth/cors.go` | 第 15-18 行 | ✅ 已复核 |
| Token 读取优先级 | `auth/authentication.go` | 第 205-216 行 | ✅ 已复核 |
| axios 未设置 withCredentials | `ui/src/CurrentUser.ts` | 第 52-53 行 | ✅ 已复核 |
| 前端登出逻辑（401 触发） | `ui/src/CurrentUser.ts` | 第 109-110 行 | ✅ 已复核 |
| 配置文件路径 | `config/config.go` | 第 71-76 行 | ✅ 已复核 |

---

## 2. 跨域规则的配置来源与加载路径

### 2.1 配置来源层级

#### 2.1.1 配置文件
- **主配置文件**：`config.yml`（当前工作目录）
- **系统级配置**：`/etc/gotify/config.yml`（Linux 系统）
- **配置格式**：YAML 格式

**代码证据**：`config/config.go:71-76`
```go
func configFiles() []string {
    if mode.Get() == mode.TestDev {
        return []string{"config.yml"}
    }
    return []string{"config.yml", "/etc/gotify/config.yml"}
}
```

#### 2.1.2 环境变量
- **前缀**：`GOTIFY_`
- **配置加载库**：`github.com/jinzhu/configor`
- **优先级**：环境变量 > 配置文件

**代码证据**：`config/config.go:78-87`

### 2.2 CORS 相关配置项

#### 2.2.1 通用 HTTP 请求 CORS 配置
```yaml
server:
  cors:
    alloworigins:       # 允许的源站列表（正则表达式）
      - '.+.example.com'
      - 'otherdomain.com'
    allowmethods:       # 允许的 HTTP 方法
      - "GET"
      - "POST"
    allowheaders:       # 允许的请求头
      - "Authorization"
      - "content-type"
```

**代码证据**：`auth/cors.go:14-44`

#### 2.2.2 WebSocket 专用 CORS 配置

⚠️ **重要结论（经代码复核）**：`server.stream.allowedorigins` 实际只匹配 Origin 的 **hostname**，不包含协议、端口和路径。

```yaml
server:
  stream:
    allowedorigins:     # ⚠️ 只匹配 hostname，正则不要包含协议或端口
      - "^my-app\\.com$"           # 匹配精确域名
      - "^.+\\.my-app\\.com$"      # 匹配所有子域名
      - "^192\\.168\\.1\\.100$"    # 匹配精确 IP 地址
```

**代码证据**：`api/stream/stream.go:187-209`
```go
func isAllowedOrigin(r *http.Request, allowedOrigins []*regexp.Regexp) bool {
    origin := r.Header.Get("origin")
    // ...
    u, err := url.Parse(origin)
    // ...
    for _, allowedOrigin := range allowedOrigins {
        // ⚠️ 关键：第 203 行，只匹配 u.Hostname()，不是完整 Origin URL
        if allowedOrigin.MatchString(strings.ToLower(u.Hostname())) {
            return true
        }
    }
    return false
}
```

**匹配示例（可直接生效）**：

| Origin 请求头值 | 配置的正则 | 匹配的 hostname | 结果 |
|----------------|-----------|----------------|------|
| `https://app.example.com` | `^app\.example\.com$` | `app.example.com` | ✅ 匹配 |
| `https://app.example.com:8443` | `^app\.example\.com$` | `app.example.com` | ✅ 匹配（端口被忽略） |
| `https://sub.app.example.com` | `^.+\.app\.example\.com$` | `sub.app.example.com` | ✅ 匹配 |
| `http://localhost:3000` | `^localhost$` | `localhost` | ✅ 匹配 |

**常见错误配置（不生效）**：
- ❌ `^https?://my-app\.com$` → 包含协议，永远不匹配
- ❌ `^my-app\.com:8443$` → 包含端口，永远不匹配

#### 2.2.3 WebSocket CheckOrigin 调用链
**代码证据**：`api/stream/stream.go:211-223`
```go
func newUpgrader(allowedWebSocketOrigins []string) *websocket.Upgrader {
    compiledAllowedOrigins := compileAllowedWebSocketOrigins(allowedWebSocketOrigins)
    return &websocket.Upgrader{
        ReadBufferSize:  1024,
        WriteBufferSize: 1024,
        CheckOrigin: func(r *http.Request) bool {
            if mode.IsDev() {
                return true
            }
            return isAllowedOrigin(r, compiledAllowedOrigins)
        },
    }
}
```

---

## 3. WebSocket 浏览器侧认证限制分析

### 3.1 WebSocket API 原生限制

⚠️ **核心限制（浏览器行为）**：**浏览器标准 WebSocket API 不支持自定义请求头**

**限制说明**：
```javascript
// 浏览器标准 WebSocket 构造函数
const ws = new WebSocket(url, protocols);
// ❌ 没有第三个参数用于设置 headers
// ❌ 不能设置 X-Gotify-Key、Authorization 等自定义请求头
```

**代码证据**：`ui/src/message/WebSocketStore.ts:22` - 前端未也无法设置 WebSocket 请求头
```typescript
const wsUrl = config.get('url').replace('http', 'ws').replace('https', 'wss');
const ws = new WebSocket(wsUrl + 'stream');  // ⚠️ 无法设置自定义请求头
```

### 3.2 Gotify WebSocket 认证机制

**服务端认证流程**：
**代码证据**：`api/stream/stream.go:143-158`
```go
func (a *API) Handle(ctx *gin.Context) {
    // 第 144 行：先做 Upgrade，此时已应用 CORS 检查
    conn, err := a.upgrader.Upgrade(ctx.Writer, ctx.Request, nil)
    if err != nil {
        ctx.Error(err)
        return
    }

    var token string
    // 第 151-153 行：从 Gin Context 获取已认证的客户端（来自中间件）
    if c := auth.GetClient(ctx); c != nil {
        token = c.Token
    }
    // ...
}
```

### 3.3 WebSocket 认证途径（按优先级）

**代码证据**：`auth/authentication.go:205-216`

| 认证方式 | HTTP API 可用 | WebSocket 可用 | 跨域可行性 | 说明 |
|---------|-------------|---------------|-----------|------|
| **Cookie** | ✅ 是 | ✅ 是 | ❌ 受 SameSite 限制 | 优先级第 4 |
| **Query Token (`?token=xxx`)** | ✅ 是 | ✅ 是 | ✅ 完全可行 | 优先级第 1 |
| **`X-Gotify-Key` Header** | ✅ 是 | ❌ 否 | - | 优先级第 2，WebSocket 不可用 |
| **`Authorization: Bearer`** | ✅ 是 | ❌ 否 | - | 优先级第 3，WebSocket 不可用 |
| **Basic Auth** | ✅ 是 | ⚠️ 极少用 | ⚠️ 需 URL 编码 | 浏览器不支持直接设置 |

**关键结论**：
> ✅ **完全跨域场景下，X-Gotify-Key 仅适用于 HTTP API，WebSocket 必须依赖 Cookie 或 query token**

---

## 4. 跨域凭据链路深度分析

### 4.1 前端 axios withCredentials 配置核查

#### 4.1.1 当前配置状态
**核查结论**：前端代码中**未显式设置** `withCredentials`，使用 axios 默认值 `false`

**代码证据**：`ui/src/CurrentUser.ts:52-53` - axios.create() 无参数
```typescript
axios
    .create()  // ⚠️ 第 53 行：使用默认配置，withCredentials = false
    .request({
        url: config.get('url') + 'auth/local/login',
        method: 'POST',
        data: {name},
        headers: {Authorization: 'Basic ' + btoa(username + ':' + password)},
    })
```

**axios 默认行为说明**：
- `withCredentials: false`（默认）：跨域请求时**不发送** Cookie，也**不接受**响应中的 `Set-Cookie`
- `withCredentials: true`：跨域请求时**发送** Cookie，并**接受**响应中的 `Set-Cookie`

#### 4.1.2 完全跨域场景下 Cookie 发送前提

要让 Cookie 在完全跨域场景下正常工作，必须同时满足以下 **4 个条件**：

| 序号 | 条件 | 当前状态 | 是否满足 |
|------|------|---------|---------|
| 1 | 前端 `axios.withCredentials = true` | ❌ 默认 `false` | 否 |
| 2 | 服务端 CORS `AllowCredentials = true` | ❌ 未设置 | 否 |
| 3 | Cookie `SameSite` 属性必须为 `None` | ❌ `Strict`（第 20 行） | 否 |
| 4 | Cookie `Secure` 属性必须为 `true`（HTTPS） | ⚠️ 可配置 | 部分 |

### 4.2 服务端 CORS AllowCredentials 配置核查

#### 4.2.1 当前配置状态
**核查结论**：服务端 CORS 配置中**未显式开启** `AllowCredentials`，使用 `gin-contrib/cors` 默认值 `false`

**代码证据**：`auth/cors.go:14-18`
```go
func CorsConfig(conf *config.Configuration) cors.Config {
    corsConf := cors.Config{
        MaxAge:                 12 * time.Hour,
        AllowBrowserExtensions: true,
        // ⚠️ 第 15-18 行：关键缺失，未设置 AllowCredentials = true
    }
    // ...
}
```

### 4.3 跨域登录后持续 401 的根因分析

**完整链路（完全跨域场景）**：

**阶段 1：登录请求**
```
1. 前端 → POST https://gotify.other.com/auth/local/login
   withCredentials: false（默认）
   + Basic Auth 头
   代码证据：ui/src/CurrentUser.ts:52-59

2. 服务端验证 Basic Auth 成功
   → 生成 Client Token
   → 返回 Set-Cookie: gotify-client-token=xxx; SameSite=Strict; HttpOnly
   代码证据：auth/cookie.go:12-21（第 20 行 SameSite=Strict）

3. 浏览器响应处理
   → 检测到跨域响应 Set-Cookie
   → 检测到 SameSite=Strict
   → ❌ 拒绝保存 Cookie！
   → 浏览器中无任何 Cookie 保存
```

**阶段 2：后续请求（如 tryAuthenticate）**
```
4. 前端 → GET https://gotify.other.com/current/user
   withCredentials: false（默认）
   → ❌ 浏览器无 Cookie 可发送
   → 请求头中无 Cookie

5. 服务端认证中间件
   → 读取 Cookie：空
   → 读取 X-Gotify-Key：空
   → 读取 Authorization Bearer：空
   → 读取 Query token：空
   → ❌ 返回 401 Unauthorized
   代码证据：auth/authentication.go:205-216

6. 前端 401 处理
   → 触发登出逻辑
   → loggedIn = false
   代码证据：ui/src/CurrentUser.ts:109-110

7. 循环：用户再次登录 → 再次 401 → 再次登出...
```

### 4.4 三重阻碍导致无法跨域使用 Cookie

| 阻碍点 | 代码位置 | 影响 |
|--------|---------|------|
| 1 | `axios.withCredentials = false`（默认） | 不发送也不接受跨域 Cookie |
| 2 | `auth/cors.go:15-18` 未配置 AllowCredentials | 服务端拒绝带凭据的跨域请求 |
| 3 | `auth/cookie.go:20` SameSite=StrictMode | 浏览器根本不保存跨域 Cookie |

---

## 5. 三场景鉴权矩阵（HTTP vs WebSocket）

### 5.1 三种部署场景定义

| 场景代号 | 部署方式 | 域名示例 | 浏览器同源判定 |
|---------|---------|---------|--------------|
| A | 同域部署 | UI + API + WS: `https://gotify.example.com/` | ✅ 同源 |
| B | 反代同域 | UI: `https://app.example.com/`<br>通过 Nginx 转发 API/Ws 到内网 Gotify | ✅ 浏览器认为同源 |
| C | 完全跨域 | UI: `https://my-dashboard.com/`<br>API: `https://gotify.other.com/` | ❌ 完全跨域 |

### 5.2 HTTP API 鉴权矩阵

| 认证方案 | 场景 A（同域） | 场景 B（反代同域） | 场景 C（完全跨域） | 需修改代码 |
|---------|-------------|-----------------|-----------------|-----------|
| **Cookie（默认）** | ✅ **原生支持** | ✅ **原生支持** | ❌ **完全不可用** | 否 |
| **X-Gotify-Key Header** | ✅ 支持 | ✅ 支持 | ✅ **推荐方案** | ✅ 前端需拦截器 |
| **Authorization Bearer** | ✅ 支持 | ✅ 支持 | ✅ **推荐方案** | ✅ 前端需拦截器 |
| **Query Token** | ✅ 支持 | ✅ 支持 | ⚠️ 支持但不推荐 | ✅ 前端需处理 |

### 5.3 WebSocket 鉴权矩阵

| 认证方案 | 场景 A（同域） | 场景 B（反代同域） | 场景 C（完全跨域） | 需修改代码 |
|---------|-------------|-----------------|-----------------|-----------|
| **Cookie（默认）** | ✅ **原生支持** | ✅ **原生支持** | ❌ **完全不可用** | 否 |
| **X-Gotify-Key Header** | ❌ 不可用 | ❌ 不可用 | ❌ 不可用 | - |
| **Authorization Bearer** | ❌ 不可用 | ❌ 不可用 | ❌ 不可用 | - |
| **Query Token (`?token=xxx`)** | ✅ 支持 | ✅ 支持 | ✅ **唯一可行方案** | ✅ 前端需拼接 URL |

### 5.4 各场景详细鉴权链路

#### 5.4.1 场景 A：同域部署（推荐默认方案）

**HTTP API 链路**：
1. ✅ URL 自动构造为同源（第 20-26 行）
2. ✅ Cookie SameSite=Strict 在同域正常工作（第 20 行）
3. ✅ axios.withCredentials=false 在同域不影响 Cookie 发送
4. ✅ 无需 CORS 配置

**WebSocket 链路**：
1. ✅ 同源请求，CheckOrigin 直接通过（第 198-200 行）
2. ✅ Cookie 正常发送，服务端认证成功
3. ✅ 无需配置 stream.allowedorigins

**配置清单**：
```yaml
# config.yml - 无需额外 CORS 配置
server:
  securecookie: true  # HTTPS 环境必须启用
  # cors.alloworigins - 无需配置
  # stream.allowedorigins - 无需配置
```

**验证步骤**：
1. 访问 `https://gotify.example.com/`
2. 登录检查 Cookie：`Application → Cookies → gotify-client-token`
3. 刷新页面，确认保持登录状态
4. 检查 Network → WS → Headers，确认连接建立成功

---

#### 5.4.2 场景 B：反代同域部署（生产环境推荐）

**核心原理**：通过 Nginx 反向代理让浏览器认为 UI 和 API 同域

**Nginx 反向代理配置**：
```nginx
server {
    listen 443 ssl;
    server_name app.example.com;

    # 前端静态资源（或另一个 upstream）
    location / {
        root /var/www/dashboard;
        try_files $uri $uri/ /index.html;
    }

    # Gotify API 转发 - 保持同域路径
    location /gotify/ {
        proxy_pass http://gotify-internal:8080/;
        
        # 保留原始 Host
        proxy_set_header Host $host;
        
        # 转发真实 IP
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket 支持
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

**Gotify 配置**：
```yaml
server:
  trustedproxies:
    - 192.168.1.10/32  # Nginx 服务器 IP
  securecookie: true
  # CORS 无需配置 - 浏览器认为是同域
```

**HTTP API 链路**：
1. ✅ 浏览器认为是同域（相同 protocol://host:port）
2. ✅ Cookie 正常发送（SameSite=Strict 在同域生效）
3. ✅ 无需 CORS 配置
4. ✅ 无需修改前端代码

**WebSocket 链路**：
1. ✅ 浏览器认为是同源，CheckOrigin 直接通过
2. ✅ Cookie 正常发送，服务端认证成功
3. ✅ 无需配置 stream.allowedorigins

---

#### 5.4.3 场景 C：完全跨域部署（需代码修改）

⚠️ **重要**：此场景下 WebSocket 无法使用 Header 认证，必须用 Cookie 或 Query Token

**方案 1：启用跨域 Cookie（需 3 处修改 + 1 处配置）**

**修改 1/4：前端 axios 配置**
```typescript
// ui/src/index.tsx 或全局初始化位置
axios.defaults.withCredentials = true;
```

**修改 2/4：服务端 CORS 配置**
```go
// auth/cors.go:14-18
func CorsConfig(conf *config.Configuration) cors.Config {
    corsConf := cors.Config{
        MaxAge:                 12 * time.Hour,
        AllowBrowserExtensions: true,
        AllowCredentials:       true,  // ✅ 新增：允许跨域凭据
    }
    // ...
}
```

**修改 3/4：Cookie SameSite 属性**
```go
// auth/cookie.go:12-21
func SetCookie(w http.ResponseWriter, token string, maxAge int, secure bool) {
    // 跨域场景需要 SameSite=None
    // 注意：SameSite=None 必须配合 Secure=true
    sameSite := http.SameSiteNoneMode
    
    http.SetCookie(w, &http.Cookie{
        Name:     CookieName,
        Value:    token,
        Path:     "/",
        MaxAge:   maxAge,
        Secure:   secure,  // 必须为 true
        HttpOnly: true,
        SameSite: sameSite,
    })
}
```

**修改 4/4：WebSocket Origin 白名单配置**
```yaml
server:
  stream:
    allowedorigins:
      - "^my-dashboard\\.com$"  # ⚠️ 只匹配 hostname
```

**方案 2：Header + Query Token 混合认证（推荐，改动最小）**

**前端修改（HTTP API 拦截器）**：
```typescript
// 添加 axios 请求拦截器
axios.interceptors.request.use(config => {
    const token = localStorage.getItem('gotify-token');
    if (token) {
        config.headers['X-Gotify-Key'] = token;  // ✅ HTTP API 使用 Header
    }
    return config;
});

// WebSocketStore 中修改 WebSocket 连接
const ws = new WebSocket(wsUrl + 'stream?token=' + token);  // ✅ WebSocket 使用 Query Token
```

**优点**：
- ✅ 不受 SameSite 限制
- ✅ 无需修改 Cookie 配置
- ✅ 无需修改服务端 CORS AllowCredentials（但仍需配置 allowOrigins）
- ✅ 改动集中在前端

**缺点**：
- ❌ Token 暴露在 WebSocket URL 中（可能出现在日志）
- ❌ Token 存储在 localStorage（XSS 风险，虽然后端还有 HttpOnly）
- ❌ 会话续期需要手动处理

---

## 6. 可落地部署方案与风险评估

### 6.1 方案优先级推荐

| 优先级 | 方案 | 适用场景 | 改动量 | 风险 |
|-------|------|---------|-------|------|
| 1 | **同域部署** | 新项目、域名可控 | 无 | 最低 |
| 2 | **反代同域** | 已有域名、可配置 Nginx | 仅运维配置 | 低 |
| 3 | **Header+Query 混合认证** | 必须完全跨域 | 前端修改 | 中 |
| 4 | **跨域 Cookie** | 必须完全跨域、不想改前端逻辑 | 前端 + 后端 | 高 |

### 6.2 完全跨域场景完整配置参考

```yaml
server:
  securecookie: true  # SameSite=None 必须配合 HTTPS
  
  cors:
    alloworigins:
      - '^https?://my-dashboard\.com$'
      - '^https?://.+\.my-dashboard\.com$'
    allowmethods:
      - "GET"
      - "POST"
      - "DELETE"
      - "OPTIONS"
      - "PUT"
    allowheaders:
      - "X-Gotify-Key"
      - "Authorization"
      - "Content-Type"
  
  stream:
    allowedorigins:
      - "^my-dashboard\\.com$"  # ⚠️ 只匹配 hostname
      - "^.+\\.my-dashboard\\.com$"
  
  trustedproxies:
    - 192.168.1.10/32  # 明确列出可信代理
```

### 6.3 风险评估矩阵

| 风险点 | 同域部署 | 反代同域 | 混合认证 | 跨域 Cookie |
|--------|---------|---------|---------|-----------|
| CSRF 攻击 | ✅ 低（SameSite=Strict） | ✅ 低 | ⚠️ 中（无 SameSite 保护） | ❌ 高（SameSite=None） |
| XSS 攻击 | ✅ 低（HttpOnly Cookie） | ✅ 低 | ⚠️ 中（localStorage） | ✅ 低 |
| 令牌泄露 | ✅ 低 | ✅ 低 | ⚠️ 中（URL 日志） | ✅ 低 |
| 浏览器兼容性 | ✅ 高 | ✅ 高 | ✅ 高 | ⚠️ 中（SameSite=None 支持） |
| 维护成本 | ✅ 低 | ✅ 低 | ⚠️ 中 | ❌ 高 |

---

## 总结

### 核心发现（经代码复核）
1. **WebSocket Hostname 匹配**：`server.stream.allowedorigins` 正则**只匹配 Origin 的 hostname 部分**（`api/stream/stream.go:203`），不要包含协议或端口
2. **前端同源策略**：UI 基于 `window.location` 自动构造同源 URL（`ui/src/index.tsx:20-26`），默认不会跨域调用
3. **WebSocket Header 限制**：浏览器标准 WebSocket API **不支持自定义请求头**，因此 X-Gotify-Key 仅适用于 HTTP API
4. **Cookie 三重阻碍**：完全跨域场景下，`axios.withCredentials=false` + `cors.AllowCredentials=false` + `SameSite=Strict` 三重阻碍导致 Cookie 认证完全失效
5. **WebSocket 认证依赖**：跨域场景下 WebSocket 必须依赖 Cookie 或 Query Token，Header 认证不可用

### 最佳实践
1. **同域部署优先**：利用前端自动构造同源 URL 的特性，避免跨域复杂性
2. **反向代理其次**：通过 Nginx 反代实现浏览器认为的同域，无需修改代码
3. **WebSocket 正则简化**：只写 hostname 匹配，如 `^my-app\.com$`
4. **完全跨域场景推荐混合认证**：
   - HTTP API 使用 X-Gotify-Key Header
   - WebSocket 使用 ?token=xxx Query 参数
   - 无需修改 Cookie 和 CORS AllowCredentials
5. **明确配置 `trustedproxies`**：不使用 `0.0.0.0/0`，明确列出可信代理
6. **生产环境启用 `securecookie`**：配合 HTTPS 使用，跨域场景必须

### 部署决策树
```
开始
  ↓
是否可以同域部署？
  ├─ 是 → 场景 A：直接部署 ✅（零配置，零风险）
  └─ 否
      ↓
是否可以通过反向代理实现浏览器认为的同域？
      ├─ 是 → 场景 B：Nginx 反代 ✅（零代码修改，低风险）
      └─ 否 → 场景 C：完全跨域
          ↓
          选择方案：
          ├─ 方案 1：Header+Query 混合认证 ✅（推荐，改动小，风险中）
          └─ 方案 2：启用跨域 Cookie（改动大，风险高）
```
