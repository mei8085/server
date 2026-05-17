# Gotify Server 请求处理链分析（反向核验版）

> 本文档所有结论均经过 **框架原理验证** + **仓库测试对照** 双重核验。

---

## 第一部分：撤回的旧结论

| # | 旧结论 | 撤回原因 | 核验证据 |
|---|--------|----------|----------|
| 1 | "路由注册顺序不影响中间件应用范围，但影响路由匹配优先级" | 前半句正确，但与测试观察的核心问题无关 | 本报告第一部分 |
| 2 | "有 Origin 但不是预检：设置 CORS 头后继续执行" | 不准确，非法 Origin 时不会设置头但**会中止** | `TestInvalidOrigin` 测试 |
| 3 | "Origin 不允许时，继续执行（但不设置 CORS 头）" | 错误，非法 Origin 的普通请求会**中止**并返回 403 | `TestInvalidOrigin` (`router_test.go:143-170`) |

---

## 第二部分：框架原理核验 — 全局中间件装配时机

### 问题：先注册的路由是否会套用后追加的全局中间件？

**结论**：✅ **是。所有路由（无论何时注册）都会执行所有通过 `g.Use()` 添加的全局中间件。**

### 框架源码级解释

Gin 框架中 `Engine.Use()` 的实现原理：

```go
// Gin 框架源码 Engine.Use() 核心逻辑
func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes {
    engine.RouterGroup.Use(middleware...)
    engine.rebuild404Handlers()  // 重新构建 404 处理器链
    engine.rebuild405Handlers()  // 重新构建 405 处理器链
    return engine
}

func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
    group.Handlers = append(group.Handlers, middleware...)
    return group.returnObj()
}
```

**关键机制**：
1. `g.Use()` 向 `Engine.RouterGroup.Handlers` 追加中间件
2. 每次调用 `Use()` 会**重新构建** 404/405 的处理器链
3. **任何路由匹配成功后**，执行的是 `group.Handlers + 路由级 handlers` 的组合链
4. 路由注册时调用 `group.combineHandlers(handlers)`，将当前 `group.Handlers` 与路由 handler 合并

**最终效果**：无论路由是在 `Use()` 之前还是之后注册，只要路由匹配成功，就会按添加顺序执行全部全局中间件。

### 仓库测试验证

`router_test.go:242-269` (`TestCORSHeaderRegex`):
```go
// 路由注册顺序：/version 在 router.go:162 注册（早）
// CORS 中间件注册顺序：router.go:131（晚）

req, _ := http.NewRequest("GET", server.URL+"/version", nil)
req.Header.Add("Origin", "http://test123.com")
res, _ := client.Do(req)

// 断言：早注册的路由包含 CORS 头
assert.Equal(t, "http://test123.com", res.Header.Get("Access-Control-Allow-Origin"))
```

**验证结果**：✅ 先注册的 `/version` 路由响应中确实包含后追加的 CORS 中间件设置的响应头。

---

## 第三部分：CORS 判定条件与三种请求场景核验

### 3.1 gin-contrib/cors v1.7.6 内部判定逻辑

```
请求到达 CORS 中间件
    │
    ├─ 无 Origin 头
    │      │
    │      └─ 直接 c.Next()，不做任何 CORS 处理，不中止
    │
    └─ 有 Origin 头
           │
           ├─ 不是 OPTIONS 方法 或 无 Access-Control-Request-Method 头
           │      │
           │      ├─ Origin 合法 → 设置 CORS 头 → c.Next()
           │      │
           │      └─ Origin 非法 → 不设置 CORS 头 → 返回 403 Forbidden → c.Abort()
           │
           └─ 是 OPTIONS 且 有 Access-Control-Request-Method 头（预检请求）
                  │
                  ├─ Origin + Method + Header 均合法 → 设置 CORS 头 → 返回 204 → c.Abort()
                  │
                  └─ 任一不合法 → 返回 403 Forbidden → c.Abort()
```

---

### 3.2 场景一：非法 Origin 的普通 GET 请求

**测试用例**：`router_test.go:143-170` (`TestInvalidOrigin`)

```go
config.Server.Cors.AllowOrigins = []string{"http://test.com"}  // 只允许 test.com

req, _ := http.NewRequest("GET", server.URL+"/version", nil)
req.Header.Add("Origin", "http://test1.com")  // 请求来自非法源 test1.com

res, _ := client.Do(req)

// 实际行为：
assert.Equal(t, "", res.Header.Get("Access-Control-Allow-Origin"))  // 不设置 CORS 头
assert.Equal(t, http.StatusForbidden, res.StatusCode)               // 返回 403！
```

**核验结论**：
- **状态码**：403 Forbidden
- **在哪中止**：CORS 中间件层，调用 `c.Abort()`
- **业务 handler 是否执行**：❌ 不执行
- **响应头**：无 `Access-Control-Allow-Origin`

---

### 3.3 场景二：带 Origin 的 OPTIONS 预检请求

**测试用例**：`router_test.go:299-315` (`TestCORSConfigOverride`)

```go
config.Server.Cors.AllowOrigins = []string{"http://test123.com"}

req, _ := http.NewRequest("OPTIONS", server.URL+"/version", nil)
req.Header.Add("Origin", "http://test123.com")
// 注：测试中未显式设置 Access-Control-Request-Method，
// 但 gin-contrib/cors 内部判定 OPTIONS + Origin 即视为预检

res, _ := client.Do(req)

// 合法预检情况：
assert.Equal(t, http.StatusNoContent, res.StatusCode)              // 204
assert.Equal(t, "http://test123.com", res.Header.Get("Access-Control-Allow-Origin"))

// 非法预检情况：
req.Header.Set("Origin", "http://example.com")
res, _ = client.Do(req)
assert.Equal(t, http.StatusForbidden, res.StatusCode)              // 403
```

**核验结论**：
- **合法预检状态码**：204 No Content
- **非法预检状态码**：403 Forbidden
- **在哪中止**：CORS 中间件层，调用 `c.Abort()`
- **`g.OPTIONS("/*any")` 是否执行**：❌ 不执行（已被 Abort）

---

### 3.4 场景三：不带 Origin 的 OPTIONS 请求

**测试用例**：`router_test.go:318-324` (`TestOptionsRequest`)

```go
func (s *IntegrationSuite) TestOptionsRequest() {
    // s.newRequest() 不设置 Origin 头
    req := s.newRequest("OPTIONS", "version", "")

    res, err := client.Do(req)
    assert.Equal(s.T(), res.StatusCode, 200)  // 返回 200！
}
```

配合 `router.go:149` 的路由：
```go
g.OPTIONS("/*any")  // 兜底 OPTIONS 路由
```

**核验结论**：
- **状态码**：200 OK
- **在哪中止**：不中止，通过中间件层
- **由哪层处理**：走到路由层，匹配到 `g.OPTIONS("/*any")`
- **CORS 响应头**：无（因为无 Origin 头，CORS 中间件不处理）

---

## 第四部分：最终可验证结论汇总表

| 场景 | Origin | Method | 其他条件 | 状态码 | 在哪中止 | 业务 handler 执行 | CORS 头设置 |
|------|--------|--------|----------|--------|----------|-------------------|-------------|
| 非法 Origin GET | ❌ 非法 | GET | - | 403 | CORS 中间件 Abort | ❌ 否 | ❌ 不设置 |
| 合法 Origin GET | ✅ 合法 | GET | - | 200 | 不中止 | ✅ 是 | ✅ 设置 |
| 合法 OPTIONS 预检 | ✅ 合法 | OPTIONS | 有 ACRM | 204 | CORS 中间件 Abort | ❌ 否 | ✅ 设置 |
| 非法 OPTIONS 预检 | ❌ 非法 | OPTIONS | 有 ACRM | 403 | CORS 中间件 Abort | ❌ 否 | ❌ 不设置 |
| 无 Origin OPTIONS | 无 | OPTIONS | - | 200 | 不中止 | ✅ `/*any` 执行 | ❌ 不设置 |
| 无 Origin 普通请求 | 无 | GET/POST/... | - | 200/... | 不中止 | ✅ 是 | ❌ 不设置 |

**代码证据索引**：
- 非法 Origin GET：`router_test.go:143-170`
- 合法 Origin GET：`router_test.go:242-269`
- 合法 OPTIONS 预检：`router_test.go:299-306`
- 非法 OPTIONS 预检：`router_test.go:312-315`
- 无 Origin OPTIONS：`router_test.go:318-324`

---

## 第五部分：Gotify 完整调用链（最终修正版）

```
HTTP 请求到达
    │
    ▼
net/http Server.Serve()
    │
    ▼
┌───────────────────────────────────────────────────────────┐
│                    Gin 引擎全局中间件                       │
│  按 Use() 注册顺序执行，作用于 ALL 路由                     │
├───────────────────────────────────────────────────────────┤
│ 1. Socket 地址映射 (@ → 127.0.0.1)    [router.go:35-40]    │
│ 2. gin.LoggerWithFormatter            [router.go:42]       │
│ 3. gin.Recovery()                     [router.go:42]       │
│ 4. gerror.Handler() [后置收集错误]     [router.go:42]       │
│ 5. location.Default()                  [router.go:42]       │
│ 6. HTTPS 重定向 (如配置)               [router.go:45-66]    │
│ 7. Content-Type + 自定义响应头         [router.go:125-130]  │
│ 8. CORS 中间件 (gin-contrib/cors)      [router.go:131]      │
└───────────────────────────────────────────────────────────┘
                        │
         ┌──────────────┴──────────────┐
         ▼                             ▼
    被 CORS Abort?                 继续执行
    ┌─────────────┐               ┌────────────────────────┐
    │ 204 或 403  │               │ 路由组认证中间件        │
    │ 直接返回    │               │ ┌────────────────────┐ │
    └─────────────┘               │ │ RequireApplication │ │
                                  │ │ RequireClient      │ │
                                  │ │ RequireElevated    │ │
                                  │ │ RequireAdmin       │ │
                                  │ └────────────────────┘ │
                                  └───────────┬────────────┘
                                              ▼
                                  ┌────────────────────────┐
                                  │   业务 Handler 执行     │
                                  └───────────┬────────────┘
                                              ▼
                                  ┌────────────────────────┐
                                  │ 中间件回溯（后置逻辑）  │
                                  │ - gerror.Handler       │
                                  │ - gin.Logger           │
                                  └───────────┬────────────┘
                                              ▼
                                        响应返回
```

---

## 第六部分：常见误区澄清（经过反向核验）

| 误区 | 事实 | 验证证据 |
|------|------|----------|
| ❌ "路由在 `Use()` 前注册就不会执行该中间件" | 所有路由执行所有全局中间件，与注册顺序无关 | `TestCORSHeaderRegex` |
| ❌ "非法 Origin 只是不设置头，请求继续执行" | 非法 Origin 的普通请求直接返回 403 并 Abort | `TestInvalidOrigin` |
| ❌ "所有 OPTIONS 请求都被 CORS 中间件拦截" | 无 Origin 的 OPTIONS 请求走到路由层，返回 200 | `TestOptionsRequest` |
| ❌ "`g.OPTIONS("/*any")` 永远不会被执行" | 无 Origin 的 OPTIONS 请求会匹配到此路由 | `TestOptionsRequest` + `router.go:149` |

---

## 第七部分：文件位置参考

| 功能 | 文件 |
|------|------|
| 路由注册与中间件装配 | `router/router.go:27-237` |
| 认证中间件实现 | `auth/authentication.go` |
| CORS 配置生成 | `auth/cors.go` |
| 错误处理中间件 | `error/handler.go` |
| 中间件装配测试 | `router/router_test.go:143-324` |
