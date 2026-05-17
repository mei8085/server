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

### 3.2 配置对路由的影响

前端路由在 `Layout.tsx` 中定义，认证状态影响路由访问：

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
</Routes>
```

`authed` 高阶组件通过 `RequireAuth` 检查登录状态，未登录则重定向到 `/login`。

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

### 4.1 插件目录配置

`PluginsDir` 配置项控制插件功能是否启用：

**关键代码**：`app.go:33-37`

```go
if conf.PluginsDir != "" {
    if err := os.MkdirAll(conf.PluginsDir, 0o755); err != nil {
        panic(err)
    }
}
```

**关键代码**：`router/router.go:93`

```go
pluginManager, err := plugin.NewManager(db, conf.PluginsDir, g.Group("/plugin/:id/custom/"), streamHandler)
```

- 当 `PluginsDir` 为空字符串时，插件系统被禁用
- 当 `PluginsDir` 非空时，自动创建目录并初始化插件管理器

### 4.2 前端插件管理界面

插件管理页面通过 API 获取插件列表，与配置间接相关：

**关键代码**：`ui/src/plugin/Plugins.tsx:18-55`

- 插件页面路径：`/plugins`
- 插件详情页面路径：`/plugins/:id`
- 需要登录认证（通过 `authed` 高阶组件保护）

## 五、对客户端管理界面的影响

### 5.1 认证与权限控制

客户端管理界面通过认证中间件保护：

**关键代码**：`router/router.go:201-206`

```go
client := clientAuth.Group("/client")
{
    client.GET("", clientHandler.GetClients)
    client.POST("", clientHandler.CreateClient)
    client.PUT("/:id", clientHandler.UpdateClient)
}
```

`clientAuth` 路由组使用 `authentication.RequireClient` 中间件，需要有效的客户端 Token 或登录会话。

### 5.2 前端客户端管理

客户端管理页面展示用户创建的客户端：

**关键代码**：`ui/src/client/Clients.tsx:28-113`

- 页面路径：`/clients`
- 功能：创建、编辑、删除、提升客户端权限
- 删除和提升权限操作需要 `RequireElevation` 二次认证

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
| `ui/src/user/Login.tsx` | 登录页面，使用register和oidc配置 |
| `app.go` | 应用入口，配置加载起点 |
