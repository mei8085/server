# 反向代理场景下跨域请求处理策略分析报告（最终版）

## 目录
1. [跨域规则的配置来源与加载路径](#1-跨域规则的配置来源与加载路径)
2. [插件模块的请求与跨域策略约束](#2-插件模块的请求与跨域策略约束)
3. [应用管理界面在不同代理边界下的鉴权行为差异](#3-应用管理界面在不同代理边界下的鉴权行为差异)
4. [跨域凭据链路深度分析](#4-跨域凭据链路深度分析)
5. [可落地鉴权方案矩阵](#5-可落地鉴权方案矩阵)
6. [关键代码证据索引](#6-关键代码证据索引)

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

#### 1.2.2 WebSocket 专用 CORS 配置

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

#### 1.3.2 WebSocket 跨域校验流程
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

## 3. 应用管理界面在不同代理边界下的鉴权行为差异

### 3.1 管理界面的鉴权机制

#### 3.1.1 前端 URL 构造逻辑

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

### 3.2 不同代理边界下的鉴权行为

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

---

## 4. 跨域凭据链路深度分析

### 4.1 前端 axios withCredentials 配置核查

#### 4.1.1 当前配置状态
**核查结论**：前端代码中**未显式设置** `withCredentials`，使用 axios 默认值 `false`

**代码证据**：全局搜索 `ui/` 目录无 `withCredentials` 匹配项

**axios 默认行为说明**：
- `withCredentials: false`（默认）：跨域请求时**不发送** Cookie，也**不接受**响应中的 `Set-Cookie`
- `withCredentials: true`：跨域请求时**发送** Cookie，并**接受**响应中的 `Set-Cookie`

**代码证据**：`ui/src/CurrentUser.ts:52-59`（登录请求示例）
```typescript
axios
    .create()  // ⚠️ 使用默认配置，withCredentials = false
    .request({
        url: config.get('url') + 'auth/local/login',
        method: 'POST',
        data: {name},
        headers: {Authorization: 'Basic ' + btoa(username + ':' + password)},
    })
```

#### 4.1.2 完全跨域场景下 Cookie 发送前提

要让 Cookie 在完全跨域场景下正常工作，必须同时满足以下 **4 个条件**：

| 序号 | 条件 | 当前状态 | 是否满足 |
|------|------|---------|---------|
| 1 | 前端 `axios.withCredentials = true` | ❌ 默认 `false` | 否 |
| 2 | 服务端 CORS `AllowCredentials = true` | ❌ 未设置 | 否 |
| 3 | Cookie `SameSite` 属性必须为 `None` | ❌ `Strict` | 否 |
| 4 | Cookie `Secure` 属性必须为 `true`（HTTPS） | ⚠️ 可配置 | 部分 |

**链路分析**：
```
前端发送请求 (withCredentials=false)
      ↓
浏览器拦截：不发送 Cookie
      ↓
服务端接收请求：无 Cookie → 401 Unauthorized
      ↓
即使登录成功返回 Set-Cookie
      ↓
浏览器拦截：不保存跨域 Cookie
      ↓
后续请求继续 401
```

### 4.2 服务端 CORS AllowCredentials 配置核查

#### 4.2.1 当前配置状态
**核查结论**：服务端 CORS 配置中**未显式开启** `AllowCredentials`，使用 `gin-contrib/cors` 默认值 `false`

**代码证据**：`auth/cors.go:14-43`
```go
func CorsConfig(conf *config.Configuration) cors.Config {
    corsConf := cors.Config{
        MaxAge:                 12 * time.Hour,
        AllowBrowserExtensions: true,
        // ⚠️ 关键缺失：未设置 AllowCredentials = true
    }
    // ... 开发模式配置
    // ... 生产模式配置
    return corsConf
}
```

#### 4.2.2 跨域登录后持续 401 的根因分析

**完整链路（完全跨域场景）**：

**阶段 1：登录请求**
```
1. 前端 → POST https://gotify.other.com/auth/local/login
   withCredentials: false（默认）
   + Basic Auth 头

2. 服务端验证 Basic Auth 成功
   → 生成 Client Token
   → 返回 Set-Cookie: gotify-client-token=xxx
      SameSite=Strict; HttpOnly
   **代码证据**：`auth/cookie.go:297-307`

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
   **代码证据**：`auth/authentication.go:205-216`

6. 前端 401 处理
   → 触发登出逻辑
   → loggedIn = false
   **代码证据**：`ui/src/CurrentUser.ts:109-110`

7. 循环：用户再次登录 → 再次 401 → 再次登出...
```

**三重阻碍导致无法跨域使用 Cookie**：

| 阻碍点 | 位置 | 影响 |
|--------|------|------|
| 1 | `axios.withCredentials = false` | 不发送也不接受跨域 Cookie |
| 2 | `cors.AllowCredentials = false` | 服务端拒绝带凭据的跨域请求 |
| 3 | `SameSite = StrictMode` | 浏览器根本不保存跨域 Cookie |

**代码证据汇总**：
- 阻碍 1：axios 默认行为（ui 目录无 withCredentials 配置）
- 阻碍 2：`auth/cors.go:14-18` 未配置 AllowCredentials
- 阻碍 3：`auth/cookie.go:305` SameSite=StrictMode

---

## 5. 可落地鉴权方案矩阵

### 5.1 三种部署场景定义

| 场景代号 | 部署方式 | 域名示例 | 浏览器同源判定 |
|---------|---------|---------|--------------|
| A | 同域部署 | UI + API: `https://gotify.example.com/` | ✅ 同源 |
| B | 反代同域 | UI: `https://app.example.com/`<br>通过 Nginx 转发 API 到内网 Gotify | ✅ 浏览器认为同源 |
| C | 完全跨域 | UI: `https://my-dashboard.com/`<br>API: `https://gotify.other.com/` | ❌ 完全跨域 |

### 5.2 方案可行性矩阵

| 认证方案 | 场景 A（同域） | 场景 B（反代同域） | 场景 C（完全跨域） | 需修改代码 |
|---------|-------------|-----------------|-----------------|-----------|
| **Cookie（默认）** | ✅ **原生支持** | ✅ **原生支持** | ❌ **完全不可用** | 否 |
| **X-Gotify-Key Header** | ✅ 支持 | ✅ 支持 | ✅ **推荐方案** | ✅ 需修改前端 |
| **Authorization Bearer** | ✅ 支持 | ✅ 支持 | ✅ **推荐方案** | ✅ 需修改前端 |

### 5.3 各场景详细落地方案

#### 5.3.1 场景 A：同域部署（推荐默认方案）

**配置清单**：
```yaml
# config.yml - 无需额外 CORS 配置
server:
  securecookie: true  # HTTPS 环境必须启用
  # cors.alloworigins - 无需配置
  # stream.allowedorigins - 无需配置
```

**工作原理**：
1. ✅ 前端 URL 自动构造为同源
   **代码证据**：`ui/src/index.tsx:20-26`
2. ✅ Cookie SameSite=Strict 在同域正常工作
   **代码证据**：`auth/cookie.go:305`
3. ✅ axios.withCredentials=false 在同域不影响 Cookie 发送
4. ✅ 无需 CORS 配置

**验证步骤**：
1. 访问 `https://gotify.example.com/`
2. 登录检查 Cookie：`Application → Cookies → gotify-client-token`
3. 刷新页面，确认保持登录状态

---

#### 5.3.2 场景 B：反代同域部署（生产环境推荐）

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

**前端部署**：
- 前端构建产物放在 `https://app.example.com/gotify/` 路径下
- URL 自动构造为 `https://app.example.com/gotify/`（同源）
  **代码证据**：`ui/src/index.tsx:20-26`

**工作原理**：
1. ✅ 浏览器认为是同域（相同 protocol://host:port）
2. ✅ Cookie 正常发送（SameSite=Strict 在同域生效）
   **代码证据**：`auth/cookie.go:305`
3. ✅ 无需 CORS 配置
4. ✅ 无需修改前端代码

---

#### 5.3.3 场景 C：完全跨域部署（需代码修改）

⚠️ **必须同时做以下 3 处修改**

**修改 1/3：前端 axios 配置（必须）**

**文件**：`ui/src/index.tsx` 或全局初始化位置
```typescript
// 添加全局 axios 默认配置
axios.defaults.withCredentials = true;
// 或在每次 create 时指定
// const instance = axios.create({ withCredentials: true });
```

**修改 2/3：服务端 CORS 配置（必须）**

**文件**：`auth/cors.go`
```go
func CorsConfig(conf *config.Configuration) cors.Config {
    corsConf := cors.Config{
        MaxAge:                 12 * time.Hour,
        AllowBrowserExtensions: true,
        AllowCredentials:       true,  // ✅ 新增：允许跨域凭据
    }
    // ... 其余保持不变
}
```

**修改 3/3：Cookie SameSite 属性（必须）**

**文件**：`auth/cookie.go`
```go
func SetCookie(w http.ResponseWriter, token string, maxAge int, secure bool) {
    sameSite := http.SameSiteStrictMode
    // 新增：跨域场景需要 SameSite=None
    // 注意：SameSite=None 必须配合 Secure=true
    if len(conf.Server.Cors.AllowOrigins) > 0 {
        sameSite = http.SameSiteNoneMode
    }
    
    http.SetCookie(w, &http.Cookie{
        Name:     cookieName,
        Value:    token,
        Path:     "/",
        MaxAge:   maxAge,
        Secure:   secure,
        HttpOnly: true,
        SameSite: sameSite,  // ✅ 根据场景动态设置
    })
}
```

**配置文件**：
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
```

**可选替代方案：Header 认证（推荐，无需修改 Cookie）**

如果不想修改 Cookie 的 SameSite 策略，可以改用 Header 认证：

**前端修改**：
```typescript
// 登录成功后保存 token 到 localStorage
// 后续请求添加 X-Gotify-Key 头
axios.interceptors.request.use(config => {
    const token = localStorage.getItem('gotify-token');
    if (token) {
        config.headers['X-Gotify-Key'] = token;
    }
    return config;
});
```

**优点**：
- ✅ 不受 SameSite 限制
- ✅ 无需修改 Cookie 配置
- ✅ 标准 RESTful 认证方式

**缺点**：
- ❌ 需要修改前端代码
- ❌ Token 存储在 localStorage（XSS 风险，虽然后端还有 HttpOnly）
- ❌ 会话续期需要手动处理

---

## 6. 关键代码证据索引

| 结论 | 代码位置 | 关键行 |
|------|---------|-------|
| WebSocket Origin 只匹配 hostname | `api/stream/stream.go` | 第 203 行 |
| 前端自动构造同源 URL | `ui/src/index.tsx` | 第 20-26 行 |
| WebSocket URL 从同源 URL 派生 | `ui/src/message/WebSocketStore.ts` | 第 22 行 |
| Cookie 使用 SameSite=Strict | `auth/cookie.go` | 第 305 行 |
| CORS 中间件全局挂载 | `router/router.go` | 第 131 行 |
| Token 读取优先级 | `auth/authentication.go` | 第 205-216 行 |
| 配置文件路径 | `config/config.go` | 第 71-76 行 |
| axios 未设置 withCredentials | `ui/` 目录全局搜索 | 无匹配项 |
| CORS 未开启 AllowCredentials | `auth/cors.go` | 第 14-18 行 |
| 前端登出逻辑（401 触发） | `ui/src/CurrentUser.ts` | 第 109-110 行 |

---

## 总结（最终版）

### 核心发现
1. **WebSocket Hostname 匹配**：`server.stream.allowedorigins` 正则**只匹配 Origin 的 hostname 部分**，不要包含协议或端口
2. **前端同源策略**：UI 基于 `window.location` 自动构造同源 URL，默认不会跨域调用
3. **Cookie 三重阻碍**：完全跨域场景下，`axios.withCredentials=false` + `cors.AllowCredentials=false` + `SameSite=Strict` 三重阻碍导致 Cookie 认证完全失效
4. **双轨 CORS**：HTTP API 匹配完整 Origin URL，WebSocket 只匹配 hostname
5. **开发 vs 生产**：开发模式完全放开 CORS，生产模式严格正则匹配

### 最佳实践
1. **同域部署优先**：利用前端自动构造同源 URL 的特性，避免跨域复杂性
2. **反向代理其次**：通过 Nginx 反代实现浏览器认为的同域，无需修改代码
3. **WebSocket 正则简化**：只写 hostname 匹配，如 `^my-app\.com$`
4. **完全跨域场景**：
   - 必须开启 `axios.withCredentials = true`
   - 必须开启服务端 `AllowCredentials = true`
   - 必须将 Cookie `SameSite` 改为 `None`（配合 `Secure=true`）
   - **或**改用 Header 认证（推荐，更简单）
5. **明确配置 `trustedproxies`**：不使用 `0.0.0.0/0`，明确列出可信代理
6. **生产环境启用 `securecookie`**：配合 HTTPS 使用，跨域场景必须
7. **CORS 正则精确**：避免过度放宽，使用 `^` 和 `$` 锚定边界

### 部署决策树
```
开始
  ↓
是否可以同域部署？
  ├─ 是 → 场景 A：直接部署 ✅（零配置）
  └─ 否
      ↓
是否可以通过反向代理实现浏览器认为的同域？
      ├─ 是 → 场景 B：Nginx 反代 ✅（零代码修改）
      └─ 否 → 场景 C：完全跨域
          ↓
          选择方案：
          ├─ 方案 1：修改三处代码启用跨域 Cookie
          └─ 方案 2：改用 Header 认证（推荐）
```
