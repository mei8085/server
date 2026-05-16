# 反向代理场景下跨域请求处理策略分析报告

## 目录
1. [跨域规则的配置来源与加载路径](#1-跨域规则的配置来源与加载路径)
2. [插件模块的请求与跨域策略约束](#2-插件模块的请求与跨域策略约束)
3. [应用管理界面在不同代理边界下的鉴权行为差异](#3-应用管理界面在不同代理边界下的鉴权行为差异)

---

## 1. 跨域规则的配置来源与加载路径

### 1.1 配置来源层级

#### 1.1.1 配置文件
- **主配置文件**：`config.yml`（当前工作目录）
- **系统级配置**：`/etc/gotify/config.yml`（Linux 系统）
- **配置格式**：YAML 格式

#### 1.1.2 环境变量
- **前缀**：`GOTIFY_`
- **配置加载库**：`github.com/jinzhu/configor`
- **优先级**：环境变量 > 配置文件

### 1.2 CORS 相关配置项

#### 1.2.1 通用 HTTP 请求 CORS 配置 (`auth/cors.go:14-44`)
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

#### 1.2.2 WebSocket 专用 CORS 配置 (`api/stream/stream.go:31-38`)
```yaml
server:
  stream:
    allowedorigins:     # WebSocket 专用允许源列表（正则表达式）
      - ".+.example.com"
      - "otherdomain.com"
```

#### 1.2.3 响应头全局配置
```yaml
server:
  responseheaders:     # 全局响应头配置（可包含 Access-Control-* 头）
    X-Custom-Header: "custom value"
```

### 1.3 配置加载流程

#### 1.3.1 加载入口 (`config/config.go:78-87`)
```go
func Get() *Configuration {
    conf := new(Configuration)
    err := configor.New(&configor.Config{
        ENVPrefix: "GOTIFY", 
        Silent: true
    }).Load(conf, configFiles()...)
    // ...
}
```

#### 1.3.2 CORS 配置应用流程 (`auth/cors.go:14-44`)

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

#### 1.3.3 WebSocket 跨域校验流程 (`api/stream/stream.go:187-209`)
```go
func isAllowedOrigin(r *http.Request, allowedOrigins []*regexp.Regexp) bool {
    // 1. 无 Origin 头 → 允许
    // 2. 同源（Host 匹配） → 允许
    // 3. 匹配配置的正则 → 允许
    // 4. 否则 → 拒绝
}
```

### 1.4 关键实现细节

#### 1.4.1 开发模式与生产模式差异
| 特性 | 开发模式 | 生产模式 |
|------|----------|----------|
| Origin 校验 | 允许所有源 | 正则匹配校验 |
| 允许方法 | 固定 5 种 | 可配置 |
| 允许头 | 固定头列表 | 可配置 |

#### 1.4.2 正则表达式编译
- **位置**：`auth/cors.go:55-62` 和 `api/stream/stream.go:225-232`
- **特性**：启动时预编译，提升运行时性能
- **注意**：`MustCompile` 会在正则无效时 panic

---

## 2. 插件模块的请求与跨域策略约束

### 2.1 插件路由结构 (`router/router.go:93-101`)

#### 2.1.1 插件路由挂载点
```go
// 插件自定义路由前缀
g.Group("/plugin/:id/custom/")
```

#### 2.1.2 管理 API 路由
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

### 2.2 插件自定义 Webhook 注册 (`plugin/manager.go:348-352`)

```go
if compat.HasSupport(instance, compat.Webhooker) {
    id := pluginConf.ID
    g := m.mux.Group(pluginConf.Token+"/", requirePluginEnabled(id, m.db))
    instance.RegisterWebhook(strings.Replace(g.BasePath(), ":id", strconv.Itoa(int(id)), 1), g)
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
| 插件管理 API（跨域） | 是 | 需配置 alloworigins |
| 插件自定义 Webhook（同域） | 否 | 依赖插件实现 |
| 插件自定义 Webhook（跨域） | 是 | 需配置 alloworigins + 插件 Token |

---

## 3. 应用管理界面在不同代理边界下的鉴权行为差异

### 3.1 管理界面的鉴权机制

#### 3.1.1 服务端认证中间件 (`auth/authentication.go:45-76`)

| 认证方式 | 优先级 | 跨域兼容性 |
|---------|--------|------------|
| Cookie (`gotify-client-token`) | 4 | 低（受 SameSite 限制） |
| `X-Gotify-Key` Header | 2 | 高 |
| `Authorization: Bearer` | 3 | 高 |
| Query `token` 参数 | 1 | 高 |
| Basic Auth | - | 中（需 CORS 允许头部） |

#### 3.1.2 前端认证实现 (`ui/src/apiAuth.ts`)
```typescript
// Axios 拦截器处理 401 错误
axios.interceptors.response.use(undefined, (error) => {
    if (error.response?.status === 401) {
        currentUser.tryAuthenticate()
            .then(() => snack('Could not complete request.'));
    }
});
```

### 3.2 不同代理边界下的鉴权行为

#### 3.2.1 场景一：同域部署（无反向代理边界）

```
用户浏览器 → https://gotify.example.com/
                ↓ 直接访问
            Gotify 服务器
```

**鉴权行为**：
- ✅ Cookie 认证工作正常（SameSite 规则满足）
- ✅ WebSocket 连接无跨域问题
- ✅ 所有 API 调用正常
- ✅ 会话自动续期（Cookie 更新）

#### 3.2.2 场景二：子域名反向代理（同根域）

```
用户浏览器 → https://app.example.com/ (UI)
                ↓ 反向代理
            https://api.example.com/ (Gotify)
```

**鉴权行为**：
- ⚠️ Cookie 需配置 `Domain` 属性
- ⚠️ `SecureCookie` 需启用（生产环境 HTTPS）
- ✅ 需配置 `server.cors.alloworigins` 包含 UI 域名
- ✅ WebSocket 需配置 `server.stream.allowedorigins`

#### 3.2.3 场景三：完全跨域反向代理（不同根域）

```
用户浏览器 → https://my-app.com/ (UI)
                ↓ 跨域请求
            https://gotify.other.com/ (Gotify)
```

**鉴权行为**：
- ❌ Cookie 认证失效（第三方 Cookie 被浏览器阻止）
- ✅ 必须使用 `X-Gotify-Key` 或 `Authorization` 头
- ✅ 必须正确配置 CORS `alloworigins`
- ✅ WebSocket 必须配置 `stream.allowedorigins`
- ❌ 会话自动续期失效（依赖 Cookie）

#### 3.2.4 场景四：路径前缀反向代理（同域）

```
用户浏览器 → https://example.com/gotify/
                ↓ 路径重写
            Gotify 服务器（内部路径 /）
```

**鉴权行为**：
- ✅ Cookie 正常（同域）
- ⚠️ 需配置 `Cookie Path` 为 `/gotify/`
- ✅ 无跨域问题（CORS 可不配置）
- ⚠️ WebSocket 路径需正确代理

### 3.3 关键配置项对鉴权的影响

#### 3.3.1 `server.trustedproxies` (`router/router.go:31-32`)
```go
g.RemoteIPHeaders = []string{"X-Forwarded-For"}
g.SetTrustedProxies(conf.Server.TrustedProxies)
```
- **影响**：正确获取客户端真实 IP
- **风险**：未正确配置可能导致安全问题
- **建议**：明确列出可信的反向代理 IP

#### 3.3.2 `server.securecookie` (`auth/authentication.go:42`)
```go
type Auth struct {
    DB           Database
    SecureCookie bool  // Cookie Secure 标志
}
```
- **影响**：HTTPS 场景下必须启用
- **跨域影响**：Secure + SameSite=None 才能在跨域场景使用
- **浏览器行为**：现代浏览器默认 SameSite=Lax

#### 3.3.3 Cookie 属性设置 (`auth/cookie.go`)
```go
const CookieMaxAge = 7 * 24 * 60 * 60 // 7 天

func SetCookie(w http.ResponseWriter, token string, maxAge int, secure bool) {
    http.SetCookie(w, &http.Cookie{
        Name:     cookieName,
        Value:    token,
        Path:     "/",
        MaxAge:   maxAge,
        Secure:   secure,
        HttpOnly: true,
        SameSite: http.SameSiteStrictMode,  // Strict 模式！
    })
}
```

**关键发现**：`SameSite=StrictMode` 导致跨域场景 Cookie 完全无法发送。

### 3.4 跨域鉴权的解决方案对比

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **Cookie (默认)** | 自动续期、HttpOnly | SameSite 限制跨域 | 同域部署 |
| **X-Gotify-Key Header** | 不受 SameSite 限制 | 需前端管理 | 完全跨域 |
| **Authorization Bearer** | 标准方案 | 需前端管理 | 完全跨域 |
| **Query token** | 简单 | 暴露在日志中 | WebSocket |

### 3.5 反向代理配置建议

#### 3.5.1 Nginx 配置示例（跨域场景）
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
    
    # 建议：不在代理层处理 CORS，让 Gotify 自己处理
    # proxy_hide_header Access-Control-Allow-Origin;
}
```

#### 3.5.2 Gotify 配置（跨域场景）
```yaml
server:
  trustedproxies:
    - 192.168.1.0/24  # 反向代理所在网段
  securecookie: true    # HTTPS 环境必须启用
  cors:
    alloworigins:
      - '^https?://my-app\.com$'
      - '^https?://.*\.my-app\.com$'
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
      - "^https?://my-app\\.com$"
```

---

## 总结

### 核心发现
1. **双轨 CORS**：HTTP API 和 WebSocket 使用独立的 CORS 配置
2. **开发 vs 生产**：开发模式完全放开 CORS，生产模式严格正则匹配
3. **插件继承**：插件路由继承全局 CORS 策略，无特殊例外
4. **Cookie 限制**：`SameSite=Strict` 导致跨域场景必须改用 Header 认证
5. **代理信任**：`trustedproxies` 配置直接影响客户端 IP 获取和安全性

### 最佳实践
1. 同域部署优先，避免跨域复杂性
2. 跨域场景使用 Header 认证而非 Cookie
3. 明确配置 `trustedproxies`，不使用 `0.0.0.0/0`
4. 生产环境必须启用 `securecookie`
5. CORS 正则表达式要精确，避免过度放宽
