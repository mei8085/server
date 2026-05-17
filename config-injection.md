# 服务端配置加载与前端注入完整链路

## 一、配置加载顺序（服务端）

### 1.1 配置合并优先级

服务端使用 `jinzhu/configor` 库加载配置，优先级从高到低为：

```
环境变量 > 配置文件 > 结构体默认值
```

**关键代码**：`config/config.go:79-87`

```go
func Get() *Configuration {
    conf := new(Configuration)
    err := configor.New(&configor.Config{ENVPrefix: "GOTIFY", Silent: true}).Load(conf, configFiles()...)
    // ...
}
```

### 1.2 配置文件搜索路径

根据运行模式不同，配置文件搜索路径有所区别：

| 模式 | 配置文件路径 |
|------|-------------|
| TestDev | `config.yml`（仅当前目录） |
| Dev/Prod | `config.yml`（当前目录） → `/etc/gotify/config.yml`（系统目录） |

**关键代码**：`config/config.go:71-76`

```go
func configFiles() []string {
    if mode.Get() == mode.TestDev {
        return []string{"config.yml"}
    }
    return []string{"config.yml", "/etc/gotify/config.yml"}
}
```

### 1.3 环境变量命名规则

环境变量以 `GOTIFY_` 为前缀，使用全大写，以下划线分隔层级：

| 结构体字段 | 环境变量示例 |
|-----------|-------------|
| `DefaultUser.Name` | `GOTIFY_DEFAULTUSER_NAME` |
| `Server.SSL.Port` | `GOTIFY_SERVER_SSL_PORT` |
| `Server.SSL.LetsEncrypt.Hosts` | `GOTIFY_SERVER_SSL_LETSENCRYPT_HOSTS` |

**示例**：`config/config_test.go:12-42`

```bash
export GOTIFY_DEFAULTUSER_NAME="jmattheis"
export GOTIFY_SERVER_PORT="8080"
export GOTIFY_REGISTRATION="true"
```

### 1.4 配置结构体默认值

在 `Configuration` 结构体中通过 `default` 标签定义默认值：

| 配置项 | 默认值 | 说明 |
|-------|-------|------|
| `Server.Port` | `80` | HTTP 监听端口 |
| `Server.SSL.Enabled` | `false` | SSL 开关 |
| `Registration` | `false` | 用户注册开关 |
| `OIDC.Enabled` | `false` | OIDC 登录开关 |
| `Database.Dialect` | `sqlite3` | 数据库类型 |
| `PluginsDir` | `data/plugins` | 插件目录（空则禁用插件） |

## 二、配置注入前端流程

### 2.1 服务端配置提取与序列化

在路由初始化时，从完整配置中提取前端需要的字段：

**关键代码**：`router/router.go:107`

```go
ui.Register(g, *vInfo, conf.Registration, conf.OIDC.Enabled)
```

提取的字段包括：
- `Version`：版本信息（Version、Commit、BuildDate）
- `Register`：是否允许用户注册
- `OIDC`：是否启用 OIDC 登录

### 2.2 HTML 模板占位符替换

前端配置通过 `index.html` 中的 `%CONFIG%` 占位符注入：

**关键代码**：`ui/serve.go:25-45`

```go
func Register(r *gin.Engine, version model.VersionInfo, register, oidcEnabled bool) {
    uiConfigBytes, _ := json.Marshal(uiConfig{
        Version: version,
        Register: register,
        OIDC: oidcEnabled,
    })
    
    replaceConfig := func(content string) string {
        return strings.Replace(content, "%CONFIG%", string(uiConfigBytes), 1)
    }
    
    ui.GET("/", serveFile("index.html", "text/html", replaceConfig))
}
```

### 2.3 前端配置接收

`index.html` 模板中的占位符位置：

**关键代码**：`ui/index.html:38`

```html
<script>window.config = %CONFIG%;</script>
```

渲染后示例：
```html
<script>window.config = {"register":false,"version":{"version":"2.0.0","commit":"abc123","buildDate":"2024-01-01"},"oidc":false};</script>
```

## 三、前端配置读取与使用

### 3.1 配置合并与封装

前端在 `config.ts` 中定义配置接口，并与 `window.config` 合并：

**关键代码**：`ui/src/config.ts:1-30`

```typescript
export interface IConfig {
    url: string;
    register: boolean;
    version: IVersion;
    oidc: boolean;
}

const config: IConfig = {
    url: 'unset',
    register: false,
    version: {commit: 'unknown', buildDate: 'unknown', version: 'unknown'},
    oidc: false,
    ...window.config,  // 与服务端注入的配置合并
};

export function get<K extends keyof IConfig>(key: K): IConfig[K] {
    return config[key];
}
```

### 3.2 运行时配置与前端路由的真实关系

前端路由在 `Layout.tsx` 中**完全静态声明**，没有任何路由会根据配置动态添加或移除。路由的访问控制和可见性分为以下三类：

#### 3.2.1 静态声明的路由（始终存在于路由表中）

**关键代码**：`ui/src/layout/Layout.tsx:135-162`

```typescript
<Routes>
    <Route path="/login" element={<Login />} />
    <Route path="/" element={authed(<Messages />)} />
    <Route path="/messages/:id" element={authed(<Messages />)} />
    <Route path="/applications" element={authed(<Applications />)} />
    <Route path="/clients" element={authed(<Clients />)} />
    <Route path="/users" element={authed(elevated(<Users />))} />
    <Route path="/plugins" element={authed(<Plugins />)} />
    <Route path="/plugins/:id" element={authed(<Lazy ... />)} />
</Routes>
```

所有路由在编译时就已确定，不存在条件性路由注册。

#### 3.2.2 仅由鉴权或登录态控制的路由

| 路由 | 控制方式 | 说明 |
|------|---------|------|
| `/login` | 无限制 | 始终可访问，已登录用户会被自动重定向到首页 |
| `/`、`/messages/:id` | `RequireAuth` | 需要登录 |
| `/applications` | `RequireAuth` | 需要登录 |
| `/clients` | `RequireAuth` | 需要登录 |
| `/plugins`、`/plugins/:id` | `RequireAuth` | 需要登录 |
| `/users` | `RequireAuth` + `RequireElevation` | 需要登录 + 二次认证 |

**关键代码**：`ui/src/layout/Layout.tsx:189-217`

```typescript
const RequireAuth: React.FC<...> = ({children, authenticating, loggedIn}) => {
    if (authenticating) return <LoadingSpinner />;
    if (!loggedIn) return <Navigate replace={true} to="/login" />;
    return <>{children}</>;
};

export const RequireElevation = observer(({children}) => {
    const {elevateStore} = useStores();
    if (elevateStore.elevated) return <>{children}</>;
    return <ElevationForm />;
});
```

#### 3.2.3 由 register 和 oidc 开关影响的内容（非路由本身）

`register` 和 `oidc` 配置**不影响任何路由的存在或可访问性**，仅控制页面内特定 UI 元素的显示：

| 配置项 | 影响范围 | 关键代码位置 |
|-------|---------|-------------|
| `register` | 登录页面的"Register"按钮显示/隐藏 | `ui/src/user/Login.tsx:25-37` |
| `oidc` | 登录页面的"Login with OIDC"按钮显示/隐藏 | `ui/src/user/Login.tsx:84-101` |
| `oidc` | 二次认证表单的"Elevate via OIDC"按钮显示/隐藏 | `ui/src/common/ElevationForm.tsx:82-94` |

> **重要结论**：没有任何前端路由会因为 `register=false` 或 `oidc=false` 而消失或不可访问。这些配置仅作为特性开关控制页面内元素的渲染。

### 3.3 配置对特性开关的影响

#### 3.3.1 注册功能开关

登录页面根据 `register` 配置决定是否显示注册按钮：

**关键代码**：`ui/src/user/Login.tsx:25-37`

```typescript
const registerButton = () => {
    if (config.get('register'))
        return (
            <Button id="register" variant="contained" color="primary" onClick={() => setRegisterDialog(true)}>
                Register
            </Button>
        );
    else return null;
};
```

#### 3.3.2 OIDC 登录开关

登录页面根据 `oidc` 配置决定是否显示 OIDC 登录按钮：

**关键代码**：`ui/src/user/Login.tsx:84-101`

```typescript
{config.get('oidc') && (
    <>
        <Divider style={{marginTop: 15, marginBottom: 15}}>or</Divider>
        <Button component="a" href={config.get('url') + 'auth/oidc/login?name=...'}>
            Login with OIDC
        </Button>
    </>
)}
```

## 四、对插件系统的影响

### 4.1 插件目录配置与后端功能

`PluginsDir` 配置项控制后端插件系统的加载行为：

**关键代码**：`app.go:33-37`

```go
if conf.PluginsDir != "" {
    if err := os.MkdirAll(conf.PluginsDir, 0o755); err != nil {
        panic(err)
    }
}
```

**关键代码**：`plugin/manager.go:219-222`

```go
func (m *Manager) loadPlugins(directory string) error {
    if directory == "" {
        return nil
    }
    // ... 加载插件逻辑
}
```

- 当 `PluginsDir` 为空字符串时：不创建目录，`loadPlugins` 直接返回，不加载任何插件
- 当 `PluginsDir` 非空时：自动创建目录，扫描并加载目录中的 `.so` 插件文件

**后端 API 路由始终注册**：无论 `PluginsDir` 是否为空，`/plugin` 相关 API 路由都会在 `router/router.go:133-143` 中注册。当插件系统为空时，API 仅返回空列表。

### 4.2 前端插件管理界面的可见性分析

#### 4.2.1 界面可见性：不受配置项直接控制

插件管理界面的可见性**完全不受 `PluginsDir` 配置影响**：

1. **路由始终存在**：`/plugins` 和 `/plugins/:id` 路由在 `Layout.tsx` 中静态声明，始终存在
2. **导航栏链接始终显示**：顶部导航栏中的 "Plugins" 链接在用户登录后始终渲染

**关键代码**：`ui/src/layout/Header.tsx:195-197`

```typescript
<Link className={classes.link} to="/plugins" id="navigate-plugins">
    <ResponsiveButton icon={<Apps />} label="plugins" color="inherit" />
</Link>
```

> **重要结论**：没有任何前端逻辑会根据 `PluginsDir` 配置隐藏插件页面或导航链接。即使用户配置了 `PluginsDir: ""`，登录用户仍然可以看到并访问 `/plugins` 页面。

#### 4.2.2 功能受配置影响的表现

配置对插件功能的影响体现在**数据层面**，而非界面可见性：

- 当 `PluginsDir` 为空时，`/plugin` API 返回空列表，前端页面显示空表格
- 当 `PluginsDir` 非空但目录中没有插件时，同样显示空表格
- 前端无法区分"插件系统被禁用"和"没有安装插件"两种状态

**关键代码**：`ui/src/plugin/Plugins.tsx:18-55`

```typescript
const Plugins = observer(() => {
    const {pluginStore} = useStores();
    React.useEffect(() => void pluginStore.refresh(), []);
    const plugins = pluginStore.getItems();
    return (
        <DefaultPage title="Plugins" maxWidth={1000}>
            {/* 渲染表格，plugins 为空时显示无数据 */}
        </DefaultPage>
    );
});
```

## 五、对客户端管理界面的影响

### 5.1 后端 API 与权限控制

客户端管理 API 路由通过认证中间件保护：

**关键代码**：`router/router.go:201-227`

```go
clientAuth := g.Group("")
clientAuth.Use(authentication.RequireClient)
{
    client := clientAuth.Group("/client")
    {
        client.GET("", clientHandler.GetClients)
        client.POST("", clientHandler.CreateClient)
        client.PUT("/:id", clientHandler.UpdateClient)
    }
}

clientElevated := g.Group("")
clientElevated.Use(authentication.RequireElevatedClient)
{
    clientElevated.POST("/client/:id/elevate", clientHandler.ElevateClient)
    clientElevated.DELETE("/client/:id", clientHandler.DeleteClient)
}
```

- 基础操作（查询、创建、更新）：需要 `RequireClient` 认证
- 敏感操作（删除、提升权限）：需要 `RequireElevatedClient` 二次认证

### 5.2 前端客户端管理界面的可见性分析

#### 5.2.1 界面可见性：不受任何配置项直接控制

客户端管理界面的可见性**完全不受服务端配置影响**：

1. **路由始终存在**：`/clients` 路由在 `Layout.tsx` 中静态声明，始终存在
2. **导航栏链接始终显示**：顶部导航栏中的 "Clients" 链接在用户登录后始终渲染

**关键代码**：`ui/src/layout/Header.tsx:192-194`

```typescript
<Link className={classes.link} to="/clients" id="navigate-clients">
    <ResponsiveButton icon={<DevicesOther />} label="clients" color="inherit" />
</Link>
```

> **重要结论**：没有任何配置项可以控制客户端管理界面的可见性。该页面始终对登录用户可见。

#### 5.2.2 功能控制方式

客户端管理功能的可用性完全由**认证和权限**决定，与配置项无关：

| 操作 | 权限要求 |
|-----|---------|
| 查看客户端列表 | 登录用户 |
| 创建客户端 | 登录用户 |
| 编辑客户端 | 登录用户 |
| 删除客户端 | 登录用户 + 二次认证（RequireElevation） |
| 提升客户端权限 | 登录用户 + 二次认证（RequireElevation） |

**关键代码**：`ui/src/client/Clients.tsx:28-163`

```typescript
const Clients = observer(() => {
    const {clientStore} = useStores();
    // ...
    return (
        <DefaultPage title="Clients" ...>
            {/* 渲染客户端列表 */}
        </DefaultPage>
    );
});
```

> **澄清**：不要将后端 API 的鉴权中间件路径（如 `/client`）误认为是前端界面开关。客户端管理界面本身没有配置开关，其功能限制完全基于用户登录状态和权限等级。

## 六、完整链路图

```
                          ┌─────────────────┐
                          │ 环境变量        │
                          │ GOTIFY_*        │
                          └────────┬────────┘
                                   │
                          ┌────────▼────────┐
                          │ 配置文件        │
                          │ config.yml      │
                          │ /etc/gotify/... │
                          └────────┬────────┘
                                   │
                          ┌────────▼────────┐
                          │ 结构体默认值    │
                          │ default标签     │
                          └────────┬────────┘
                                   │
                          ┌────────▼────────┐
                          │ config.Get()    │
                          └────────┬────────┘
                                   │
┌──────────────────────────────────┼──────────────────────────────────┐
│                                  │                                  │
│          ┌───────────────────────┼───────────────────────┐          │
│          │                       │                       │          │
│  ┌───────▼───────┐     ┌─────────▼─────────┐    ┌────────▼────────┐ │
│  │ 数据库初始化  │     │ 路由配置          │    │ 插件管理器      │ │
│  │ database.New  │     │ router.Create     │    │ plugin.NewManager│ │
│  └───────────────┘     └─────────┬─────────┘    └─────────────────┘ │
│                                  │                                  │
│                          ┌───────▼───────┐                          │
│                          │ ui.Register   │                          │
│                          │ 提取字段:     │                          │
│                          │ - Version     │                          │
│                          │ - Registration│                          │
│                          │ - OIDC.Enabled│                          │
│                          └───────┬───────┘                          │
└──────────────────────────────────┼──────────────────────────────────┘
                                   │
                          ┌────────▼────────┐
                          │ JSON 序列化     │
                          └────────┬────────┘
                                   │
                          ┌────────▼────────┐
                          │ 替换%CONFIG%    │
                          │ index.html      │
                          └────────┬────────┘
                                   │
                          ┌────────▼────────┐
                          │ window.config   │
                          └────────┬────────┘
                                   │
                          ┌────────▼────────┐
                          │ config.ts 合并  │
                          └────────┬────────┘
                                   │
                   ┌───────────────┼───────────────┐
                   │               │               │
           ┌───────▼──────┐ ┌──────▼──────┐ ┌──────▼───────┐
           │ 登录页面     │ │ 路由守卫     │ │ 特性开关     │
           │ - 注册按钮   │ │ RequireAuth │ │ - OIDC按钮   │
           │ - OIDC按钮   │ │ RequireElev │ │              │
           └──────────────┘ └─────────────┘ └──────────────┘
```

## 七、关键文件索引

| 文件路径 | 作用 |
|---------|------|
| `config/config.go` | 配置加载核心逻辑 |
| `config/config_test.go` | 配置加载测试用例 |
| `router/router.go` | 路由初始化与配置传递 |
| `ui/serve.go` | 前端配置注入与模板替换 |
| `ui/index.html` | HTML模板，包含%CONFIG%占位符 |
| `ui/src/config.ts` | 前端配置读取与封装 |
| `ui/src/layout/Layout.tsx` | 前端路由定义 |
| `ui/src/layout/Header.tsx` | 顶部导航栏，包含各页面链接 |
| `ui/src/user/Login.tsx` | 登录页面，使用register和oidc配置 |
| `ui/src/common/ElevationForm.tsx` | 二次认证表单，使用oidc配置 |
| `ui/src/plugin/Plugins.tsx` | 插件管理页面 |
| `ui/src/client/Clients.tsx` | 客户端管理页面 |
| `app.go` | 应用入口，配置加载起点 |
