# 反向代理场景下跨域请求处理策略分析报告（最终版 - 含令牌来源闭环）

## 目录
1. [关键代码证据索引（经复核）](#1-关键代码证据索引经复核)
2. [跨域规则的配置来源与加载路径](#2-跨域规则的配置来源与加载路径)
3. [WebSocket 浏览器侧认证限制分析](#3-websocket-浏览器侧认证限制分析)
4. [跨域凭据链路深度分析](#4-跨域凭据链路深度分析)
5. [令牌来源闭环分析](#5-令牌来源闭环分析)
6. [完全跨域场景下可执行的令牌生命周期](#6-完全跨域场景下可执行的令牌生命周期)
7. [最终方案矩阵（HTTP API / WebSocket 分开）](#7-最终方案矩阵http-api--websocket-分开)
8. [可落地部署方案与风险评估](#8-可落地部署方案与风险评估)

---

## 1. 关键代码证据索引（经复核）

| 结论 | 代码文件 | 精确行号 | 验证状态 |
|------|---------|---------|---------|
| WebSocket Origin 只匹配 hostname | `api/stream/stream.go` | 第 203 行 | ✅ 已复核 |
| WebSocket Upgrade 调用 | `api/stream/stream.go` | 第 144 行 | ✅ 已复核 |
| 前端自动构造同源 URL | `ui/src/index.tsx` | 第 20-26 行 | ✅ 已复核 |
| WebSocket URL 从同源 URL 派生 | `ui/src/message/WebSocketStore.ts` | 第 22 行 | ✅ 已复核 |
| Cookie 使用 SameSite=Strict | `auth/cookie.go` | 第 20 行 | ✅ 已复核 |
| CurrentUserExternal 不含 Token | `model/user.go` | 第 88-113 行 | ✅ 已复核 |
| `/auth/local/login` 只通过 Cookie 发 Token | `api/session.go` | 第 89-97 行 | ✅ 已复核 |
| `POST /client` 返回包含 Token 的 Client | `api/client.go` | 第 154 行 | ✅ 已复核 |
| CORS 中间件全局挂载 | `router/router.go` | 第 131 行 | ✅ 已复核 |
| CORS 未配置 AllowCredentials | `auth/cors.go` | 第 15-18 行 | ✅ 已复核 |
| Token 读取优先级 | `auth/authentication.go` | 第 205-216 行 | ✅ 已复核 |
| axios 未设置 withCredentials | `ui/src/CurrentUser.ts` | 第 52-53 行 | ✅ 已复核 |

---

## 2. 跨域规则的配置来源与加载路径

### 2.1 配置来源层级

#### 2.1.1 配置文件
- **主配置文件**：`config.yml`（当前工作目录）
- **系统级配置**：`/etc/gotify/config.yml`（Linux 系统）
- **配置格式**：YAML 格式

**代码证据**：`config/config.go:71-76`

#### 2.1.2 环境变量
- **前缀**：`GOTIFY_`
- **配置加载库**：`github.com/jinzhu/configor`
- **优先级**：环境变量 > 配置文件

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
```

**代码证据**：`api/stream/stream.go:187-209`

**匹配示例（可直接生效）**：

| Origin 请求头值 | 配置的正则 | 匹配的 hostname | 结果 |
|----------------|-----------|----------------|------|
| `https://app.example.com` | `^app\.example\.com$` | `app.example.com` | ✅ 匹配 |
| `https://app.example.com:8443` | `^app\.example\.com$` | `app.example.com` | ✅ 匹配（端口被忽略） |
| `http://localhost:3000` | `^localhost$` | `localhost` | ✅ 匹配 |

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

### 3.2 Gotify WebSocket 认证机制

**服务端认证流程**：
**代码证据**：`api/stream/stream.go:143-158`
```go
func (a *API) Handle(ctx *gin.Context) {
    // 第 144 行：先做 Upgrade，此时已应用 CORS 检查
    conn, err := a.upgrader.Upgrade(ctx.Writer, ctx.Request, nil)
    // ...
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
> ✅ **完全跨域场景下，X-Gotify-Key 仅适用于 HTTP API，WebSocket 必须依赖 Cookie 或 Query Token**

---

## 4. 跨域凭据链路深度分析

### 4.1 前端 axios withCredentials 配置核查

#### 4.1.1 当前配置状态
**核查结论**：前端代码中**未显式设置** `withCredentials`，使用 axios 默认值 `false`

**代码证据**：`ui/src/CurrentUser.ts:52-53` - axios.create() 无参数
```typescript
axios
    .create()  // ⚠️ 第 53 行：使用默认配置，withCredentials = false
    .request({...})
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

## 5. 令牌来源闭环分析

### 5.1 登录接口返回结构核查

⚠️ **核心发现（关键证据）**：`/auth/local/login` 响应体**只包含 CurrentUserExternal，不返回 client token**

**CurrentUserExternal 结构定义**：
**代码证据**：`model/user.go:88-113`
```go
type CurrentUserExternal struct {
    ID             uint        `json:"id"`              // 用户 ID
    Name           string      `json:"name"`            // 用户名
    Admin          bool        `json:"admin"`           // 是否管理员
    ClientID       uint        `json:"clientId,omitempty"`  // ⚠️ 只有 client ID，没有 token！
    ElevatedUntil *time.Time `json:"elevatedUntil,omitempty"`
}
```

**登录接口实际返回流程**：
**代码证据**：`api/session.go:89-97`
```go
auth.SetCookie(ctx.Writer, client.Token, auth.CookieMaxAge, a.SecureCookie)

ctx.JSON(200, &model.CurrentUserExternal{
    ID:            user.ID,
    Name:          user.Name,
    Admin:         user.Admin,
    ClientID:      client.ID,  // ⚠️ 只返回 ID，不返回 token！
    ElevatedUntil: client.ElevatedUntil,
})
```

**关键结论**：
> Token 只通过 `Set-Cookie` 响应头返回给浏览器，**永远不会出现在响应体中**
> 完全跨域场景下，由于 Cookie 被浏览器拦截，前端**无法获取到任何 token**

### 5.2 对 "Header+Query 混合认证" 方案的直接影响

| 影响项 | 分析结果 |
|-------|---------|
| **令牌来源缺失** | ❌ 登录接口不返回 token，前端无法获取用于 Header 认证的 token |
| **引导问题** | ❌ 首次登录后，前端没有 token 可以设置到 X-Gotify-Key 请求头 |
| **方案可行性** | ❌ 「Header+Query 混合认证」在当前代码下**无法直接落地**，必须修改 |
| **根本原因** | 认证体系设计上假设了同域 Cookie 总是可用，未考虑完全跨域场景 |

---

## 6. 完全跨域场景下可执行的令牌生命周期

### 6.1 令牌获取路径（3 种可行方案）

#### 方案 A：修改登录接口返回 Token（需要后端改动）

**路径**：`POST /auth/local/login` → 返回体新增 token 字段

**前置条件**：
- 配置 `server.cors.alloworigins` 包含前端域名
- 配置 `server.cors.allowheaders` 包含 `Authorization`

**安全风险**：
- ⚠️ Token 暴露在响应体中，增加 XSS 窃取风险
- ⚠️ 需要考虑响应加密或只返回一次

**落地状态**：🔧 需要改动后可落地

---

#### 方案 B：通过 Basic Auth 调用 POST /client 创建新客户端（当前代码可复用）

**代码证据**：`api/client.go:154` - CreateClient 返回完整 Client 对象（包含 Token）

**路径**：
1. 前端使用 Basic Auth 调用 `POST /client` 创建新客户端
2. 响应体中包含完整 Client 对象，其中有 Token 字段
3. 将 Token 存储到 localStorage

**前置条件**：
- 配置 `server.cors.alloworigins` 包含前端域名
- 配置 `server.cors.allowheaders` 包含 `Authorization`
- 用户知道自己的账号密码（首次登录时）

**安全风险**：
- ⚠️ 需要让用户输入密码并存储在前端内存中
- ⚠️ 每次创建新客户端都会产生新 token
- ⚠️ 会产生大量客户端记录

**落地状态**：✅ 当前代码可直接落地（但体验较差）

---

#### 方案 C：启用跨域 Cookie（需要前后端 3 处改动）

**路径**：
1. 前端设置 `axios.defaults.withCredentials = true`
2. 后端设置 CORS `AllowCredentials = true`
3. 后端设置 Cookie `SameSite = None`（配合 Secure=true）
4. 登录后 Token 通过 Cookie 自动传输

**前置条件**：
- 必须 HTTPS（SameSite=None 要求 Secure=true）
- 配置 `server.cors.alloworigins`（不能用通配符）
- 配置 `server.securecookie: true`

**安全风险**：
- ⚠️ SameSite=None 降低 CSRF 防护
- ⚠️ 依赖浏览器第三方 Cookie 政策（可能被用户禁用）
- ⚠️ 需要精确配置 Origin，不能用 `*`

**落地状态**：🔧 需要改动后可落地

---

### 6.2 令牌续期路径

#### 当前续期机制（基于 Cookie）
服务端在收到有效 token 后会自动刷新 Cookie 有效期

**代码证据**：`auth/authentication.go:158-166`
```go
now := timeNow()
if client.LastUsed == nil || client.LastUsed.Add(5*time.Minute).Before(now) {
    if err := a.DB.UpdateClientTokensLastUsed([]string{client.Token}, &now); err != nil {
        // ...
    }
    if isCookie {
        SetCookie(ctx.Writer, client.Token, CookieMaxAge, a.SecureCookie)
    }
}
```

**跨域场景续期方案**：

| 认证方式 | 续期机制 | 可行性 |
|---------|---------|-------|
| **Cookie 方案 C** | Cookie 自动续期 | ✅ 可用（需 withCredentials=true） |
| **Header/Query 方案 A/B** | 需要前端主动调用续期接口 | 🔧 需要新增接口 |

### 6.3 令牌撤销路径

**当前撤销接口**：
- `POST /auth/logout` - 删除当前客户端（基于 Cookie 认证）
  **代码证据**：`api/session.go:123-138`
- `DELETE /client/{id}` - 删除指定客户端（需 elevated 权限）
  **代码证据**：`api/client.go:197-243`

**跨域场景下的撤销**：

| 认证方式 | 可用接口 | 注意事项 |
|---------|---------|---------|
| **Cookie 方案 C** | `POST /auth/logout` | ✅ 正常工作 |
| **Header/Query 方案 A/B** | `DELETE /client/{id}` | ⚠️ 需要额外管理 client ID |

---

## 7. 最终方案矩阵（HTTP API / WebSocket 分开）

### 7.1 三种部署场景定义

| 场景代号 | 部署方式 | 域名示例 | 浏览器同源判定 |
|---------|---------|---------|--------------|
| A | 同域部署 | UI + API + WS: `https://gotify.example.com/` | ✅ 同源 |
| B | 反代同域 | UI: `https://app.example.com/`<br>通过 Nginx 转发 API/WS 到内网 Gotify | ✅ 浏览器认为同源 |
| C | 完全跨域 | UI: `https://my-dashboard.com/`<br>API: `https://gotify.other.com/` | ❌ 完全跨域 |

### 7.2 HTTP API 认证方案矩阵

| 认证方案 | 场景 A（同域） | 场景 B（反代同域） | 场景 C（完全跨域） | 落地状态 | 需修改 |
|---------|-------------|-----------------|-----------------|---------|-------|
| **Cookie（默认）** | ✅ 原生支持 | ✅ 原生支持 | ❌ 不可用 | ✅ 场景 A/B<br>❌ 场景 C | 场景 C 需 3 处修改 |
| **X-Gotify-Key Header** | ⚠️ 技术可但无 token 来源 | ⚠️ 技术可但无 token 来源 | ⚠️ 需先解决 token 来源 | 🔧 需解决 token 获取 | 需新增/修改登录接口 |
| **Authorization Bearer** | ⚠️ 技术可但无 token 来源 | ⚠️ 技术可但无 token 来源 | ⚠️ 需先解决 token 来源 | 🔧 需解决 token 获取 | 需新增/修改登录接口 |
| **Query Token** | ✅ 支持 | ✅ 支持 | ✅ 支持（需 token 获取） | 🔧 需解决 token 获取 | 前端需拼接 URL |
| **Basic Auth** | ✅ 支持（仅首次） | ✅ 支持（仅首次） | ✅ 支持（仅首次） | ✅ 当前代码可直接落地（仅首次认证） | 用户需每次输入密码 |

### 7.3 WebSocket 认证方案矩阵

| 认证方案 | 场景 A（同域） | 场景 B（反代同域） | 场景 C（完全跨域） | 落地状态 | 需修改 |
|---------|-------------|-----------------|-----------------|---------|-------|
| **Cookie（默认）** | ✅ 原生支持 | ✅ 原生支持 | ❌ 不可用 | ✅ 场景 A/B<br>❌ 场景 C | 场景 C 需 3 处修改 |
| **X-Gotify-Key Header** | ❌ 不可用 | ❌ 不可用 | ❌ 不可用 | ❌ 不可落地 | 浏览器不支持 |
| **Authorization Bearer** | ❌ 不可用 | ❌ 不可用 | ❌ 不可用 | ❌ 不可落地 | 浏览器不支持 |
| **Query Token (`?token=xxx`)** | ✅ 支持 | ✅ 支持 | ✅ 唯一可行方案 | 🔧 需解决 token 获取 | 前端需拼接 URL |
| **Basic Auth** | ❌ 不可用 | ❌ 不可用 | ❌ 不可用 | ❌ 不可落地 | WebSocket 构造函数不支持 |

### 7.4 完全跨域场景推荐组合方案

#### 推荐组合：Basic Auth 获取 Token + Header + Query 混合

| 阶段 | 操作 | 认证方式 | 说明 |
|------|------|---------|------|
| **首次登录** | `POST /client` + Basic Auth | Basic Auth | 获取 token 存入 localStorage |
| **HTTP API** | 所有请求加 `X-Gotify-Key` Header | Header 认证 | 所有 API 调用 |
| **WebSocket** | `new WebSocket(url + '?token=xxx')` | Query Token | 消息流连接 |

**落地状态**：✅ 当前代码可直接落地（但需前端大量修改）

---

## 8. 可落地部署方案与风险评估

### 8.1 方案优先级推荐

| 优先级 | 方案 | 适用场景 | HTTP API 落地状态 | WebSocket 落地状态 | 总改动量 |
|-------|------|---------|-----------------|-------------------|---------|
| 1 | **同域部署** | 新项目、域名可控 | ✅ 当前代码可直接落地 | ✅ 当前代码可直接落地 | 0 |
| 2 | **反代同域** | 已有域名、可配置 Nginx | ✅ 当前代码可直接落地 | ✅ 当前代码可直接落地 | 仅运维配置 |
| 3 | **启用跨域 Cookie** | 必须完全跨域、不想改认证逻辑 | 🔧 需要改动后可落地 | 🔧 需要改动后可落地 | 前端 + 后端 3 处 |
| 4 | **Basic Auth 获取 + Header + Query 混合** | 必须完全跨域、不能改 Cookie | 🔧 需要改动后可落地 | 🔧 需要改动后可落地 | 前端大量修改 |
| 5 | **修改登录接口返回 Token** | 必须完全跨域、追求最佳体验 | 🔧 需要改动后可落地 | 🔧 需要改动后可落地 | 前端 + 后端 |

### 8.2 完全跨域场景完整配置参考

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

### 8.3 风险评估矩阵

| 风险点 | 同域部署 | 反代同域 | 跨域 Cookie | Header+Query 混合 |
|--------|---------|---------|------------|-----------------|
| CSRF 攻击 | ✅ 低（SameSite=Strict） | ✅ 低 | ❌ 高（SameSite=None） | ⚠️ 中（无 SameSite 保护） |
| XSS 攻击 | ✅ 低（HttpOnly Cookie） | ✅ 低 | ✅ 低 | ⚠️ 中（localStorage） |
| 令牌泄露 | ✅ 低 | ✅ 低 | ✅ 低 | ⚠️ 中（URL 日志） |
| 浏览器兼容性 | ✅ 高 | ✅ 高 | ⚠️ 中（SameSite=None 支持） | ✅ 高 |
| 维护成本 | ✅ 低 | ✅ 低 | ❌ 高 | ❌ 高（大量前端逻辑） |
| 首次登录体验 | ✅ 好 | ✅ 好 | ✅ 好 | ⚠️ 差（需额外 API 调用） |

---

## 总结（最终版）

### 核心发现（经代码复核）
1. **WebSocket Hostname 匹配**：`server.stream.allowedorigins` 正则**只匹配 Origin 的 hostname 部分**（`api/stream/stream.go:203`）
2. **前端同源策略**：UI 基于 `window.location` 自动构造同源 URL，默认不会跨域调用（`ui/src/index.tsx:20-26`）
3. **WebSocket Header 限制**：浏览器标准 WebSocket API **不支持自定义请求头**，因此 X-Gotify-Key 仅适用于 HTTP API
4. **Token 来源闭环问题**：`/auth/local/login` 只通过 Cookie 返回 Token，响应体中**不包含 Token 字段**（`api/session.go:89-97`，`model/user.go:88-113`）
5. **混合认证不可直接落地**：由于登录接口不返回 Token，「Header+Query 混合认证」在**当前代码下无法直接落地**，必须修改后端接口或使用 Basic Auth 迂回
6. **Cookie 三重阻碍**：完全跨域场景下，`axios.withCredentials=false` + `cors.AllowCredentials=false` + `SameSite=Strict` 三重阻碍导致 Cookie 认证完全失效

### 最佳实践
1. **同域部署优先**：利用前端自动构造同源 URL 的特性，避免跨域复杂性
2. **反向代理其次**：通过 Nginx 反代实现浏览器认为的同域，无需修改代码
3. **完全跨域场景**：
   - 如能修改后端：推荐方案 C（启用跨域 Cookie）改动量最小
   - 如不能修改后端：使用「Basic Auth 创建客户端获取 Token + Header + Query Token」组合可勉强落地
4. **WebSocket 正则简化**：只写 hostname 匹配，如 `^my-app\.com$`
5. **明确配置 `trustedproxies`**：不使用 `0.0.0.0/0`，明确列出可信代理

### 部署决策树
```
开始
  ↓
是否可以同域部署？
  ├─ 是 → 场景 A：直接部署 ✅（零配置，零风险，推荐首选）
  └─ 否
      ↓
是否可以通过反向代理实现浏览器认为的同域？
      ├─ 是 → 场景 B：Nginx 反代 ✅（零代码修改，低风险）
      └─ 否 → 场景 C：完全跨域
          ↓
          选择方案：
          ├─ 能修改后端？
          │   ├─ 是 → 方案 C：启用跨域 Cookie（3 处改动，体验好）
          │   └─ 否 → 方案 B：Basic Auth 创建客户端（仅前端修改，体验差）
          └─
              └─ 追求最佳体验 → 修改登录接口返回 Token + Header+Query 混合
```
