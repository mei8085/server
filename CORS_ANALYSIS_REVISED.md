# 反向代理场景下跨域请求处理策略分析报告（修订版）

## 目录
1. [跨域规则的配置来源与加载路径](#1-跨域规则的配置来源与加载路径)
2. [插件模块的请求与跨域策略约束](#2-插件模块的请求与跨域策略约束)
3. [应用管理界面在不同代理边界下的鉴权行为差异](#3-应用管理界面在不同代理边界下的鉴权行为差异)
4. [关键代码证据索引](#4-关键代码证据索引)

---

## 1. 跨域规则的配置来源与加载路径

### 1.1 配置来源层级

#### 1.1.1 配置文件
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

#### 1.1.2 环境变量
- **前缀**：`GOTIFY_`
- **配置加载库**：`github.com/jinzhu/configor`
- **优先级**：环境变量 > 配置文件

**代码证据**：`config/config.go:78-87`
```go
func Get() *Configuration {
    conf := new(Configuration)
    err := configor.New(&configor.Config{
        ENVPrefix: "GOTIFY", 
        Silent: true
    }).Load(conf, configFiles()...)
}
```

### 1.2 CORS 相关配置项

#### 1.2.1 通用 HTTP 请求 CORS 配置
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

#### 1.2.2 WebSocket 专用 CORS 配置（修订）

⚠️ **重要修正**：`server.stream.allowedorigins` 实际只匹配 Origin 的 **hostname**，不包含协议、端口和路径。

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
        // ⚠️ 关键：只匹配 u.Hostname()，不是完整 Origin URL
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

#### 1.2.3 响应头全局配置
```yaml
server:
  responseheaders:     # 全局响应头配置
    X-Custom-Header: "custom value"
```

### 1.3 配置加载流程

#### 1.3.1 CORS 配置应用流程
**代码证据**：`auth/cors.go:14-44`

```
配置读取 → 模式判断 → 编译正则 → 配置 CORS 中间件
     ↓
  ┌───────────────────────────────────────────┐
  │ 开发模式 (mode.IsDev())                   │
  │ • AllowAllOrigins = true                  │
  │ • 预定义方法: GET, POST, DELETE, OPTIONS, PUT │
  │ • 预定义头: X-Gotify-Key, Authorization 等  │
  └───────────────────────────────────────────┘
                    ↓ 否
  ┌───────────────────────────────────────────┐
  │ 生产模式                                  │
  │ • 编译 alloworigins 为正则表达式           │
  │ • 使用 AllowOriginFunc 动态校验           │
  │ • 使用配置的 allowmethods 和 allowheaders  │
  │ • 兜底：检查 ResponseHeaders 中的 ACAO 头  │
  └───────────────────────────────────────────┘
```

#### 1.3.2 WebSocket 跨域校验流程（修订）
**代码证据**：`api/stream/stream.go:187-209`

```
WebSocket 握手请求
     ↓
1. 无 Origin 头 → 直接允许 ✅
2. 同源（Host 头匹配） → 直接允许 ✅
3. 提取 u.Hostname() → 与配置正则匹配
4. 匹配成功 → 允许 ✅
5. 匹配失败 → 拒绝 ❌
```

### 1.4 关键实现细节

#### 1.4.1 开发模式与生产模式差异
| 特性 | 开发模式 | 生产模式 |
|------|----------|----------|
| Origin 校验 | 允许所有源 | 正则匹配校验 |
| 允许方法 | 固定 5 种 | 可配置 |
| 允许头 | 固定头列表 | 可配置 |

**代码证据**：`auth/cors.go:19-41`

#### 1.4.2 正则表达式编译
- **位置**：`auth/cors.go:55-62` 和 `api/stream/stream.go:225-232`
- **特性**：启动时预编译，提升运行时性能
- **注意**：`MustCompile` 会在正则无效时 panic

---

## 2. 插件模块的请求与跨域策略约束

### 2.1 插件路由结构

#### 2.1.1 插件路由挂载点
**代码证据**：`router/router.go:93-96`
```go
pluginManager, err := plugin.NewManager(
    db, 
    conf.PluginsDir, 
    g.Group("/plugin/:id/custom/"),  // 插件自定义路由前缀
    streamHandler
)
```

#### 2.1.2 管理 API 路由
**代码证据**：`router/router.go:133-142`
```go
g.GET("/plugin", authentication.RequireClient, pluginHandler.GetPlugins)
pluginRoute := g.Group("/plugin/", authentication.RequireClient)
{
    pluginRoute.GET("/:id/config", pluginHandler.GetConfig)
    pluginRoute.POST("/:id/config", pluginHandler.UpdateConfig)
    pluginRoute.GET("/:id/display", pluginHandler.GetDisplay)
    pluginRoute.POST("/:id/enable", pluginHandler.EnablePlugin)
    pluginRoute.POST("/:id/disable", pluginHandler.DisablePlugin)
}
```

### 2.2 插件自定义 Webhook 注册
**代码证据**：`plugin/manager.go:348-352`
```go
if compat.HasSupport(instance, compat.Webhooker) {
    id := pluginConf.ID
    g := m.mux.Group(pluginConf.Token+"/", requirePluginEnabled(id, m.db))
    instance.RegisterWebhook(
        strings.Replace(g.BasePath(), ":id", strconv.Itoa(int(id)), 1), 
        g
    )
}
```

#### 2.2.1 自定义路由完整路径
```
/plugin/{plugin_id}/custom/{plugin_token}/{plugin_defined_path}
```

### 2.3 插件请求的跨域策略约束

#### 2.3.1 继承全局 CORS 策略
- **中间件挂载位置**：`router/router.go:131`（在插件路由定义之前）
- **作用范围**：所有插件相关路由继承全局 CORS 配置
- **无特殊例外**：插件路由没有独立的 CORS 设置

**代码证据**：`router/router.go:131`
```go
g.Use(cors.New(auth.CorsConfig(conf)))  // 全局挂载
// ... 后续定义的插件路由全部继承此 CORS 配置
```

#### 2.3.2 两层认证机制

| 层级 | 中间件 | 作用 |
|------|--------|------|
| 1 | `cors.New(auth.CorsConfig(conf))` | 全局跨域校验 |
| 2 | `authentication.RequireClient` | 客户端令牌认证（仅管理 API） |
| 3 | `requirePluginEnabled` | 插件启用状态校验（仅自定义路由） |

#### 2.3.3 插件 Token 的作用
- 插件自定义路由使用 Token 作为路径的一部分
- 跨域请求需要显式携带 Token
- **注意**：Token 暴露在 URL 中，需确保 HTTPS

### 2.4 插件跨域访问限制矩阵

| 访问场景 | 需要配置 CORS 吗？ | 额外条件 |
|----------|-------------------|----------|
| 插件管理 API（同域） | 否 | Cookie 认证 |
| 插件管理 API（跨域） | 是 | 需配置 `server.cors.alloworigins` |
| 插件自定义 Webhook（同域） | 否 | 依赖插件实现 |
| 插件自定义 Webhook（跨域） | 是 | 需配置 `server.cors.alloworigins` + 插件 Token |

---

## 3. 应用管理界面在不同代理边界下的鉴权行为差异（修订）

### 3.1 管理界面的鉴权机制

#### 3.1.1 前端 URL 构造逻辑（核心修正）

⚠️ **重要发现**：前端管理界面基于 `window.location` **自动构造同源 URL**，不依赖硬编码配置。

**代码证据**：`ui/src/index.tsx:20-26`
```typescript
const {port, hostname, protocol, pathname} = window.location;
const slashes = protocol.concat('//');
const path = pathname.endsWith('/') ? pathname : pathname.substring(0, pathname.lastIndexOf('/'));
const url = slashes.concat(port ? hostname.concat(':', port) : hostname) + path;
const urlWithSlash = url.endsWith('/') ? url : url.concat('/');
const prodUrl = urlWithSlash;  // 所有 API 和 WebSocket 都使用此 URL
```

**构造规则详解**：
| URL 组成部分 | 来源 | 示例值 |
|------------|------|-------|
| `protocol` | `window.location.protocol` | `https:` |
| `hostname` | `window.location.hostname` | `app.example.com` |
| `port` | `window.location.port` | `8443`（空则省略） |
| `path` | `window.location.pathname` 的目录部分 | `/gotify/` |

**WebSocket URL 派生**：
**代码证据**：`ui/src/message/WebSocketStore.ts:22`
```typescript
const wsUrl = config.get('url').replace('http', 'ws').replace('https', 'wss');
const ws = new WebSocket(wsUrl + 'stream');  // 例如：wss://app.example.com/gotify/stream
```

#### 3.1.2 服务端认证中间件
**代码证据**：`auth/authentication.go:45-76`

| 认证方式 | 优先级 | 跨域兼容性 |
|---------|--------|------------|
| Cookie (`gotify-client-token`) | 4 | 低（受 SameSite 限制） |
| `X-Gotify-Key` Header | 2 | 高 |
| `Authorization: Bearer` | 3 | 高 |
| Query `token` 参数 | 1 | 高 |
| Basic Auth | - | 中（需 CORS 允许头部） |

**代码证据**：`auth/authentication.go:205-216`
```go
func (a *Auth) readTokenFromRequest(ctx *gin.Context) (string, bool) {
    if token := a.tokenFromQuery(ctx); token != "" {        // 优先级 1
        return token, false
    } else if token := a.tokenFromXGotifyHeader(ctx); token != "" {  // 优先级 2
        return token, false
    } else if token := a.tokenFromAuthorizationHeader(ctx); token != "" {  // 优先级 3
        return token, false
    } else if token := a.tokenFromCookie(ctx); token != "" {  // 优先级 4
        return token, true
    }
    return "", false
}
```

#### 3.1.3 前端认证流程
**代码证据**：`ui/src/CurrentUser.ts:78-114`
```typescript
public tryAuthenticate = async (): Promise<AxiosResponse<ICurrentUser>> => {
    return axios
        .create()
        .get(config.get('url') + 'current/user')  // 发送到同源 URL，携带 Cookie
        .then(/* 登录成功 */)
        .catch(/* 401 则登出 */)
}
```

### 3.2 不同代理边界下的鉴权行为（修订）

#### 3.2.1 场景一：同域部署（无反向代理边界）

```
用户浏览器 → https://gotify.example.com/
                ↓ 直接访问
            Gotify 服务器（UI + API）
```

**真实鉴权路径**：
1. **URL 构造**：`prodUrl = "https://gotify.example.com/"`（同源）
   **代码证据**：`ui/src/index.tsx:20-26`
2. **API 请求**：`axios.get("https://gotify.example.com/current/user")`
   - 浏览器自动携带 `gotify-client-token` Cookie
   - **代码证据**：`ui/src/CurrentUser.ts:81`
3. **WebSocket 连接**：`new WebSocket("wss://gotify.example.com/stream")`
   - 无跨域问题，直接连接
   - **代码证据**：`ui/src/message/WebSocketStore.ts:22-23`

**鉴权限制**：
- ✅ Cookie 认证工作正常（`SameSite=Strict` 在同域生效）
  **代码证据**：`auth/cookie.go:305`
- ✅ WebSocket 连接无跨域问题
- ✅ 会话自动续期（Cookie 刷新）
- ✅ 无需配置任何 CORS

#### 3.2.2 场景二：子域名反向代理（同根域，UI 和 API 分离）

```
用户浏览器 → https://app.example.com/ (前端 UI)
                ↓ 跨域请求
            https://api.example.com/ (Gotify API)
```

**真实鉴权路径**：
1. **URL 构造**：`prodUrl = "https://app.example.com/"`（前端所在域）
2. **问题**：前端尝试请求 `https://app.example.com/current/user`，但 Gotify 在 `api.example.com`
3. **解决方案**：需要在 `app.example.com` 配置反向代理
   - `app.example.com/api/*` → 转发到 `api.example.com/*`
   - 或修改前端代码硬编码 API URL

**如果通过反向代理实现同源**：
```
用户浏览器 → https://app.example.com/
                ↓ 同域请求
            Nginx 反向代理
                ↓ 转发
            Gotify 服务器（内网）
```

**此时鉴权行为**：
1. **URL 构造**：`prodUrl = "https://app.example.com/"`
2. **API 请求**：`axios.get("https://app.example.com/current/user")`
   - 请求到 Nginx，Nginx 转发到 Gotify
   - Cookie 正常发送（同域）
3. **WebSocket**：`wss://app.example.com/stream`
   - 通过 Nginx 转发升级

**鉴权限制**：
- ✅ Cookie 正常（只要浏览器认为是同域）
- ✅ 无需配置 CORS
- ⚠️ 需要正确配置 `trustedproxies` 获取真实 IP
  **代码证据**：`router/router.go:31-32`

#### 3.2.3 场景三：完全跨域部署（UI 和 API 在完全不同的域）

```
用户浏览器 → https://my-dashboard.com/ (前端 UI)
                ↓ 完全跨域
            https://gotify.other.com/ (Gotify API)
```

**真实鉴权路径**：
1. **URL 构造**：`prodUrl = "https://my-dashboard.com/"`
   **代码证据**：`ui/src/index.tsx:20-26`
2. **默认问题**：前端请求 `https://my-dashboard.com/current/user` → 404（Gotify 不在此域）
3. **必须修改**：前端 `window.config.url` 硬编码为 `https://gotify.other.com/`

**修改后的鉴权流程**：
1. **登录请求**：Basic Auth 到 `https://gotify.other.com/auth/local/login`
   - **代码证据**：`ui/src/CurrentUser.ts:46-76`
   - 服务器返回 `Set-Cookie: gotify-client-token=xxx; SameSite=Strict; HttpOnly`
     **代码证据**：`auth/cookie.go:297-307`
2. **后续请求**：`axios.get("https://gotify.other.com/current/user")`
   - ❌ **浏览器不发送 Cookie**！（`SameSite=Strict` + 完全跨域）
     **代码证据**：`auth/cookie.go:305`
   - 服务端返回 401 Unauthorized
   - 前端触发登出逻辑
     **代码证据**：`ui/src/CurrentUser.ts:109-110`

**鉴权限制**：
- ❌ **Cookie 认证完全失效**：`SameSite=StrictMode` 导致跨域 Cookie 不发送
  **代码证据**：`auth/cookie.go:305`
- ✅ 必须配置 `server.cors.alloworigins` 包含前端域名
  **代码证据**：`auth/cors.go:27-37`
- ✅ WebSocket 必须配置 `server.stream.allowedorigins`（只匹配 hostname）
  **代码证据**：`api/stream/stream.go:203`
- ❌ 会话自动续期失效（依赖 Cookie 刷新）
- ⚠️ 前端需要修改认证逻辑：改用 `X-Gotify-Key` 或 `Authorization` 头

### 3.3 关键配置项对鉴权的影响

#### 3.3.1 `server.trustedproxies`
**代码证据**：`router/router.go:31-32`
```go
g.RemoteIPHeaders = []string{"X-Forwarded-For"}
g.SetTrustedProxies(conf.Server.TrustedProxies)
```
- **影响**：正确获取客户端真实 IP
- **风险**：未正确配置可能导致安全问题
- **建议**：明确列出可信的反向代理 IP

#### 3.3.2 `server.securecookie`
**代码证据**：`auth/authentication.go:42`
```go
type Auth struct {
    DB           Database
    SecureCookie bool  // Cookie Secure 标志
}
```
- **影响**：HTTPS 场景下必须启用
- **跨域影响**：Secure + SameSite=None 才能在跨域场景使用（但当前代码是 Strict）
- **浏览器行为**：现代浏览器默认 SameSite=Lax

#### 3.3.3 Cookie 属性设置
**代码证据**：`auth/cookie.go:297-308`
```go
func SetCookie(w http.ResponseWriter, token string, maxAge int, secure bool) {
    http.SetCookie(w, &http.Cookie{
        Name:     cookieName,
        Value:    token,
        Path:     "/",
        MaxAge:   maxAge,
        Secure:   secure,
        HttpOnly: true,
        SameSite: http.SameSiteStrictMode,  // ⚠️ 关键：Strict 模式！
    })
}
```

**关键发现**：`SameSite=StrictMode` 导致跨域场景 Cookie 完全无法发送。

### 3.4 跨域鉴权的解决方案对比

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **Cookie (默认)** | 自动续期、HttpOnly | SameSite 限制跨域 | 同域部署 |
| **X-Gotify-Key Header** | 不受 SameSite 限制 | 需前端管理 Token | 完全跨域 |
| **Authorization Bearer** | 标准方案 | 需前端管理 Token | 完全跨域 |
| **Query token** | 简单 | 暴露在日志中 | WebSocket |

### 3.5 反向代理配置建议

#### 3.5.1 Nginx 配置示例（同域路径前缀）
```nginx
location /gotify/ {
    proxy_pass http://gotify-internal/;
    
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
```

#### 3.5.2 Gotify 配置（完全跨域场景）
```yaml
server:
  trustedproxies:
    - 192.168.1.0/24  # 反向代理所在网段
  securecookie: true    # HTTPS 环境必须启用
  cors:
    alloworigins:
      - '^https?://my-dashboard\.com$'      # 注意：HTTP CORS 匹配完整 URL
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
      - "^my-dashboard\\.com$"              # ⚠️ WebSocket 只匹配 hostname！
      - "^.+\\.my-dashboard\\.com$"
```

---

## 4. 关键代码证据索引

| 结论 | 代码位置 | 关键行 |
|------|---------|-------|
| WebSocket Origin 只匹配 hostname | `api/stream/stream.go` | 第 203 行 |
| 前端自动构造同源 URL | `ui/src/index.tsx` | 第 20-26 行 |
| WebSocket URL 从同源 URL 派生 | `ui/src/message/WebSocketStore.ts` | 第 22 行 |
| Cookie 使用 SameSite=Strict | `auth/cookie.go` | 第 305 行 |
| CORS 中间件全局挂载 | `router/router.go` | 第 131 行 |
| Token 读取优先级 | `auth/authentication.go` | 第 205-216 行 |
| 配置文件路径 | `config/config.go` | 第 71-76 行 |

---

## 总结（修订版）

### 核心发现
1. **WebSocket Hostname 匹配**：`server.stream.allowedorigins` 正则**只匹配 Origin 的 hostname 部分**，不要包含协议或端口
2. **前端同源策略**：UI 基于 `window.location` 自动构造同源 URL，默认不会跨域调用
3. **Cookie 硬限制**：`SameSite=StrictMode` 导致完全跨域场景下 Cookie 认证完全失效
4. **双轨 CORS**：HTTP API 匹配完整 Origin URL，WebSocket 只匹配 hostname
5. **开发 vs 生产**：开发模式完全放开 CORS，生产模式严格正则匹配

### 最佳实践
1. **同域部署优先**：利用前端自动构造同源 URL 的特性，避免跨域复杂性
2. **WebSocket 正则简化**：只写 hostname 匹配，如 `^my-app\.com$`
3. **完全跨域场景**：必须修改前端使用 Header 认证，不能依赖 Cookie
4. **明确配置 `trustedproxies`**：不使用 `0.0.0.0/0`，明确列出可信代理
5. **生产环境启用 `securecookie`**：配合 HTTPS 使用
6. **CORS 正则精确**：避免过度放宽，使用 `^` 和 `$` 锚定边界
