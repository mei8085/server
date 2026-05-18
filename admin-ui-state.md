# Gotify Admin UI 状态管理分析

本文档分析 Gotify 管理界面中 **浅深色主题**、**登录态** 和 **侧边导航选中态** 的管理方式，以及它们与后端鉴权、路由守卫、配置接口之间的关联。

---

## 一、整体架构概览

Gotify UI 使用 **MobX** 作为状态管理库，采用 **Context + Hooks** 的方式在组件间共享 store。

### 1.1 Store 初始化与注入

**文件**: `ui/src/index.tsx:28-71`

```typescript
const initStores = (): StoreMapping => {
    const snackManager = new SnackManager();
    const appStore = new AppStore(snackManager.snack);
    const userStore = new UserStore(snackManager.snack);
    const messagesStore = new MessagesStore(appStore, snackManager.snack);
    const currentUser = new CurrentUser(snackManager.snack);
    const elevateStore = new ElevateStore(snackManager.snack, currentUser);
    const clientStore = new ClientStore(snackManager.snack);
    const wsStore = new WebSocketStore(snackManager.snack, currentUser);
    const pluginStore = new PluginStore(snackManager.snack);
    // ...
};
```

**Store 上下文**: `ui/src/stores.tsx:24-30`

```typescript
export const StoreContext = React.createContext<StoreMapping | undefined>(undefined);
export const useStores = (): StoreMapping => {
    const mapping = React.useContext(StoreContext);
    if (!mapping) throw new Error('uninitialized');
    return mapping;
};
```

### 1.2 配置注入机制

**后端**: `ui/serve.go:25-37`

Go 后端在服务 index.html 时，将配置信息注入到 `window.config` 中：

```go
uiConfigBytes, _ := json.Marshal(uiConfig{
    Version:  version,
    Register: register,
    OIDC:     oidcEnabled,
})
replaceConfig := func(content string) string {
    return strings.Replace(content, "%CONFIG%", string(uiConfigBytes), 1)
}
```

**前端**: `ui/index.html:38`

```html
<script>window.config = %CONFIG%;</script>
```

**前端读取**: `ui/src/config.ts:16-22`

```typescript
const config: IConfig = {
    url: 'unset',
    register: false,
    version: {commit: 'unknown', buildDate: 'unknown', version: 'unknown'},
    oidc: false,
    ...window.config,
};
```

---

## 二、浅深色主题管理

### 2.1 主题类型定义

**文件**: `ui/src/layout/theme.ts:1-4`

```typescript
export type ThemeKey = 'dark' | 'light' | 'system';
export const isThemeKey = (value: string | null): value is ThemeKey =>
    value === 'light' || value === 'dark' || value === 'system';
```

### 2.2 主题状态管理

**文件**: `ui/src/layout/Layout.tsx:47-90`

主题状态完全在 **Layout 组件内部** 通过 `useState` 管理，不涉及 MobX store：

```typescript
const localStorageThemeKey = 'gotify-theme';

const Layout = observer(() => {
    const [currentTheme, setCurrentTheme] = React.useState<ThemeKey>(() => {
        const stored = window.localStorage.getItem(localStorageThemeKey);
        return isThemeKey(stored) ? stored : 'system';
    });
    const prefersDark = useMediaQuery('(prefers-color-scheme: dark)');
    const paletteMode = currentTheme === 'system' ? (prefersDark ? 'dark' : 'light') : currentTheme;
    const theme = React.useMemo(
        () => createTheme({ palette: { mode: paletteMode } }),
        [paletteMode]
    );
    
    const toggleTheme = () => {
        const nextMap: Record<ThemeKey, ThemeKey> = {
            dark: 'light',
            light: 'system',
            system: 'dark',
        };
        const next = nextMap[currentTheme];
        setCurrentTheme(next);
        localStorage.setItem(localStorageThemeKey, next);
    };
    // ...
});
```

### 2.3 主题切换流程

1. **初始化**: 从 `localStorage` 读取 `gotify-theme`，默认为 `'system'`
2. **系统检测**: 使用 `useMediaQuery('(prefers-color-scheme: dark)')` 检测系统偏好
3. **切换逻辑**: 点击按钮按 `dark → light → system → dark` 循环切换
4. **持久化**: 每次切换后同步到 `localStorage`
5. **应用**: 通过 MUI 的 `ThemeProvider` 注入到整个应用

### 2.4 主题切换按钮

**文件**: `ui/src/layout/Header.tsx:136-143`

```typescript
<IconButton onClick={toggleTheme} color="inherit" size="large" title={themeLabel} aria-label={themeLabel}>
    {themeIcon}
</IconButton>
```

---

## 三、登录态管理

### 3.1 核心 Store: CurrentUser

**文件**: `ui/src/CurrentUser.ts:8-151`

这是登录态管理的核心类，使用 MobX observable 管理状态：

```typescript
export class CurrentUser {
    @observable accessor loggedIn = false;           // 是否登录
    @observable accessor refreshKey = 0;              // 用于强制刷新组件
    @observable accessor authenticating = true;       // 是否正在认证中
    @observable accessor user: ICurrentUser = {       // 当前用户信息
        name: 'unknown',
        admin: false,
        id: -1
    };
    @observable accessor connectionErrorMessage: string | null = null; // 连接错误信息
}
```

### 3.2 登录流程

#### 前端登录

**文件**: `ui/src/user/Login.tsx:38-41`

```typescript
const login = (e: React.MouseEvent<HTMLButtonElement>) => {
    e.preventDefault();
    currentUser.login(username, password);
};
```

**CurrentUser.login**: `ui/src/CurrentUser.ts:46-76`

```typescript
public login = async (username: string, password: string) => {
    runInAction(() => {
        this.loggedIn = false;
        this.authenticating = true;
    });
    const name = this.createClientName(); // 浏览器信息，如 "Chrome 120.0.0"
    axios.create().request({
        url: config.get('url') + 'auth/local/login',
        method: 'POST',
        data: {name},
        headers: {Authorization: 'Basic ' + btoa(username + ':' + password)},
    }).then(action((resp: AxiosResponse<ICurrentUser>) => {
        this.user = resp.data;
        this.loggedIn = true;
        this.authenticating = false;
        this.connectionErrorMessage = null;
        this.reconnectTime = 7500;
    }));
};
```

#### 后端登录处理

**文件**: `api/session.go:56-98`

```go
func (a *SessionAPI) Login(ctx *gin.Context) {
    // 1. 解析 Basic Auth
    name, pass, ok := ctx.Request.BasicAuth()
    
    // 2. 验证用户密码
    user, err := a.DB.GetUserByName(name)
    if user == nil || !password.ComparePassword(user.Pass, []byte(pass)) {
        ctx.AbortWithError(401, errors.New("invalid credentials"))
        return
    }
    
    // 3. 创建 Client Token（会话）
    elevatedUntil := time.Now().Add(model.DefaultElevationDuration)
    client := model.Client{
        Name:          clientParams.Name,
        Token:         auth.GenerateNotExistingToken(generateClientToken, a.clientExists),
        UserID:        user.ID,
        ElevatedUntil: &elevatedUntil,
    }
    a.DB.CreateClient(&client)
    
    // 4. 设置 Cookie（HttpOnly, Secure, SameSite=Strict）
    auth.SetCookie(ctx.Writer, client.Token, auth.CookieMaxAge, a.SecureCookie)
    
    // 5. 返回用户信息
    ctx.JSON(200, &model.CurrentUserExternal{...})
}
```

**Cookie 设置**: `auth/cookie.go:12-21`

```go
func SetCookie(w http.ResponseWriter, token string, maxAge int, secure bool) {
    http.SetCookie(w, &http.Cookie{
        Name:     "gotify-client-token",
        Value:    token,
        Path:     "/",
        MaxAge:   maxAge,    // 7天
        Secure:   secure,
        HttpOnly: true,      // 禁止 JS 访问
        SameSite: http.SameSiteStrictMode,
    })
}
```

### 3.3 会话认证流程

#### 前端自动认证

**文件**: `ui/src/index.tsx:60`

应用启动时自动尝试认证：

```typescript
stores.currentUser.tryAuthenticate().catch(() => {});
```

**CurrentUser.tryAuthenticate**: `ui/src/CurrentUser.ts:78-115`

```typescript
public tryAuthenticate = async (): Promise<AxiosResponse<ICurrentUser>> => {
    return axios.create().get(config.get('url') + 'current/user')
        .then(action((passThrough) => {
            this.user = passThrough.data;
            this.loggedIn = true;
            this.authenticating = false;
            this.connectionErrorMessage = null;
            return passThrough;
        }))
        .catch(action((error: AxiosError) => {
            this.authenticating = false;
            if (!error || !error.response) {
                this.connectionError('No network connection or server unavailable.');
                return Promise.reject(error);
            }
            if (error.response.status >= 400 && error.response.status < 500) {
                this.logout(); // 4xx 错误，清除登录态
            }
            return Promise.reject(error);
        }));
};
```

#### 后端鉴权中间件

**文件**: `auth/authentication.go:52-54`

```go
func (a *Auth) RequireClient(ctx *gin.Context) {
    a.evaluateOr401(ctx, a.handleUser(), a.handleClient())
}
```

**Token 读取顺序**: `auth/authentication.go:205-216`

```go
func (a *Auth) readTokenFromRequest(ctx *gin.Context) (string, bool) {
    if token := a.tokenFromQuery(ctx); token != "" {           // 1. Query 参数 ?token=xxx
        return token, false
    } else if token := a.tokenFromXGotifyHeader(ctx); token != "" { // 2. X-Gotify-Key 头
        return token, false
    } else if token := a.tokenFromAuthorizationHeader(ctx); token != "" { // 3. Authorization: Bearer xxx
        return token, false
    } else if token := a.tokenFromCookie(ctx); token != "" {    // 4. Cookie
        return token, true
    }
    return "", false
}
```

### 3.4 登出流程

**文件**: `ui/src/CurrentUser.ts:117-124`

```typescript
public logout = async () => {
    if (this.loggedIn) {
        runInAction(() => {
            this.loggedIn = false;
        });
        await axios.post(config.get('url') + 'auth/logout').catch(() => Promise.resolve());
    }
};
```

**后端登出**: `api/session.go:123-138`

```go
func (a *SessionAPI) Logout(ctx *gin.Context) {
    auth.SetCookie(ctx.Writer, "", -1, a.SecureCookie) // 清除 Cookie
    client := auth.GetClient(ctx)
    a.DB.DeleteClientByID(client.ID) // 删除 Client
    ctx.Status(200)
}
```

### 3.5 登录态变化的副作用

**文件**: `ui/src/reactions.ts:37-46`

使用 MobX reaction 监听登录态变化：

```typescript
reaction(
    () => stores.currentUser.loggedIn,
    (loggedIn) => {
        if (loggedIn) {
            loadAll(); // 加载应用列表、建立 WebSocket 连接
        } else {
            clearAll(); // 清除所有数据、关闭 WebSocket
        }
    }
);
```

---

## 四、权限提升（Elevation）管理

### 4.1 为什么需要权限提升？

Gotify 采用 **双因子认证** 模式：
- **普通登录**: 获取 Client Token，可访问大部分 API
- **权限提升**: 重新输入密码，获取临时 elevated 状态（默认 1 小时），可执行敏感操作（如删除用户、删除应用）

### 4.2 ElevateStore

**文件**: `ui/src/ElevateStore.ts:7-101`

```typescript
export class ElevateStore {
    @observable accessor elevated = false;            // 是否已提升
    @observable accessor oidcElevatePending = false;  // OIDC 提升是否进行中
    
    // 检查 elevatedUntil 时间戳
    @action
    public refreshElevated = (): number => {
        const elevatedUntil = this.currentUser.user.elevatedUntil;
        if (!elevatedUntil) {
            this.elevated = false;
            return 0;
        }
        const ms = new Date(elevatedUntil).getTime() - 30_000 - Date.now();
        if (ms <= 0) {
            this.elevated = false;
            return 0;
        }
        this.elevated = true;
        return ms;
    };
}
```

### 4.3 权限提升流程

**本地提升**: `ui/src/ElevateStore.ts:34-45`

```typescript
public localElevate = async (password: string, durationSeconds: number): Promise<void> => {
    await axios.create().request({
        url: `${config.get('url')}client/${this.currentUser.user.clientId}/elevate`,
        method: 'POST',
        data: {durationSeconds},
        headers: {
            Authorization: 'Basic ' + btoa(this.currentUser.user.name + ':' + password),
        },
    });
    await this.currentUser.tryAuthenticate(); // 刷新用户信息
};
```

**后端提升**: `api/client.go`（`ElevateClient` 方法）

更新 Client 的 `ElevatedUntil` 字段。

### 4.4 自动刷新 Elevation 状态

**文件**: `ui/src/reactions.ts:48-62`

```typescript
reaction(
    () => stores.currentUser.user.elevatedUntil,
    () => {
        window.clearTimeout(elevationTimerId);
        const disableAfter = stores.elevateStore.refreshElevated();
        if (disableAfter > 0) {
            elevationTimerId = window.setTimeout(
                () => stores.elevateStore.refreshElevated(),
                disableAfter
            );
        }
    },
    {fireImmediately: true}
);
```

---

## 五、路由守卫

### 5.1 前端路由配置

**文件**: `ui/src/layout/Layout.tsx:135-162`

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

### 5.2 登录守卫

**文件**: `ui/src/layout/Layout.tsx:189-199`

```typescript
const RequireAuth: React.FC<
    React.PropsWithChildren<{loggedIn: boolean; authenticating: boolean}>
> = ({children, authenticating, loggedIn}) => {
    if (authenticating) {
        return <LoadingSpinner />;
    }
    if (!loggedIn) {
        return <Navigate replace={true} to="/login" />;
    }
    return <>{children}</>;
};

const authed = (children: React.ReactNode) => (
    <RequireAuth loggedIn={loggedIn} authenticating={authenticating}>
        {children}
    </RequireAuth>
);
```

### 5.3 权限提升守卫

**文件**: `ui/src/layout/Layout.tsx:201-217`

```typescript
export const RequireElevation = observer(({children}: React.PropsWithChildren) => {
    const {elevateStore} = useStores();
    if (elevateStore.elevated) {
        return <>{children}</>;
    }
    return (
        <DefaultPage title="Authentication Required" maxWidth={400}>
            <Paper elevation={6}>
                <Box sx={{padding: 2}}>
                    <ElevationForm />
                </Box>
            </Paper>
        </DefaultPage>
    );
});

const elevated = (children: React.ReactNode) => <RequireElevation>{children}</RequireElevation>;
```

### 5.4 后端路由权限

**文件**: `router/router.go:183-236`

```go
// 需要普通 Client Token
clientAuth := g.Group("")
clientAuth.Use(authentication.RequireClient)
clientAuth.GET("/application", applicationHandler.GetApplications)
clientAuth.GET("/current/user", userHandler.GetCurrentUser)

// 需要 Elevated Client Token
clientElevated := g.Group("")
clientElevated.Use(authentication.RequireElevatedClient)
clientElevated.POST("/client/:id/elevate", clientHandler.ElevateClient)
clientElevated.DELETE("/application/:id", applicationHandler.DeleteApplication)
clientElevated.POST("/current/user/password", userHandler.ChangePassword)

// 需要 Admin 权限
authAdmin := g.Group("/user")
authAdmin.Use(authentication.RequireAdmin)
authAdmin.GET("", userHandler.GetUsers)
authAdmin.DELETE("/:id", userHandler.DeleteUserByID)
```

---

## 六、侧边导航选中态（深度分析）

### 6.1 核心结论：当前代码**没有实现**选中态高亮

经过全面代码审计，确认 Gotify UI 的侧边导航**没有任何选中态高亮逻辑**。这是让人困惑的根本原因——期望有但实际没有。

**证据链**:
- ✗ 未使用 `react-router-dom` 的 `NavLink` 组件（会自动添加 `active` 类）
- ✗ 未调用 `useLocation()` / `useMatch()` 等路由钩子
- ✗ 未给 `ListItemButton` 传递 `selected` 属性
- ✗ 项目中无任何 CSS/SCSS 文件定义选中样式
- ✗ 测试用例中无任何选中态断言
- ✗ 全局搜索 `selected` / `isSelected` / `aria-selected` 无匹配结果

---

### 6.2 导航组件完整结构

**文件**: `ui/src/layout/Navigation.tsx:1-133`

```typescript
const Navigation = observer(({loggedIn, navOpen, setNavOpen}: IProps) => {
    const [showRequestNotification, setShowRequestNotification] =
        React.useState(mayAllowPermission);
    const {classes} = useStyles();
    const {appStore} = useStores();
    const apps = appStore.getItems();

    const userApps =
        apps.length === 0
            ? null
            : apps.map((app) => (
                  <Link
                      onClick={() => setNavOpen(false)}
                      className={`${classes.link} item`}
                      to={'/messages/' + app.id}
                      key={app.id}>
                      <ListItemButton>
                          <ListItemAvatar style={{minWidth: 42}}>
                              <Avatar style={{width: 32, height: 32}} src={app.image} variant="square" />
                          </ListItemAvatar>
                          <ListItemText primary={app.name} />
                      </ListItemButton>
                  </Link>
              ));

    return (
        <ResponsiveDrawer navOpen={navOpen} setNavOpen={setNavOpen} id="message-navigation">
            <div className={classes.toolbar} />
            <Link className={classes.link} to="/" onClick={() => setNavOpen(false)}>
                <ListItemButton disabled={!loggedIn} className="all">
                    <ListItemText primary="All Messages" />
                </ListItemButton>
            </Link>
            <Divider />
            <div>{loggedIn ? userApps : placeholderItems}</div>
            <Divider />
            {/* 通知按钮... */}
        </ResponsiveDrawer>
    );
});
```

**关键观察**:
- 使用普通 `Link` 而非 `NavLink`
- `ListItemButton` 未接收 `selected` prop
- 仅通过 `className="item"` 和 `className="all"` 标识，无动态选中类

---

### 6.3 路由切换的实际流程

虽然没有选中态高亮，但路由切换本身是完整的：

```
用户点击导航项
    ↓
<Link to="/messages/1"> 触发路由变化
    ↓
React Router 更新 URL 为 /messages/1
    ↓
<Routes> 匹配到 /messages/:id
    ↓
<Messages> 组件通过 useParams() 获取 id=1
    ↓
messagesStore.loadMore(1) 加载对应应用的消息
    ↓
页面内容更新，但导航项无视觉变化
```

**Messages 组件读取路由参数**: `ui/src/message/Messages.tsx:19-27`

```typescript
const Messages = observer(() => {
    const {id} = useParams<{id: string}>();
    const appId = id == null ? -1 : parseInt(id as string, 10);
    
    const messages = messagesStore.get(appId);
    const name = appStore.getName(appId);
    // ...
});
```

---

### 6.4 导航状态与登录态的关联

导航项的显示完全依赖登录态：

| 登录状态 | 导航内容 |
|---------|---------|
| `loggedIn = false` | 显示 placeholderItems（"Some Server", "A Raspberry PI"），"All Messages" 按钮 disabled |
| `loggedIn = true` | 显示真实的用户应用列表（从 appStore 获取） |

**关联代码**: `ui/src/layout/Navigation.tsx:96`

```typescript
<div>{loggedIn ? userApps : placeholderItems}</div>
```

appStore 的数据加载由登录态触发：`ui/src/reactions.ts:37-46`

```typescript
reaction(
    () => stores.currentUser.loggedIn,
    (loggedIn) => {
        if (loggedIn) {
            stores.appStore.refresh(); // 登录成功后加载应用列表
        } else {
            stores.appStore.clear();   // 登出后清空应用列表
        }
    }
);
```

---

### 6.5 导航状态与后端鉴权的关联

导航项本身不直接参与鉴权，但导航到的页面受后端鉴权保护：

1. **导航可见性**: 未登录时只显示占位项，真实应用列表仅登录后可见
2. **路由守卫**: 所有导航目标路径（`/`, `/messages/:id`, `/applications` 等）都被 `RequireAuth` 包裹
3. **API 鉴权**: 页面加载数据时调用的 API（如 `GET /message`, `GET /application`）都需要 `RequireClient` 中间件验证

**后端路由配置**: `router/router.go:185-198`

```go
clientAuth.Use(authentication.RequireClient)
clientAuth.GET("/application", applicationHandler.GetApplications)
clientAuth.GET("/message", messageHandler.GetMessages)
```

---

### 6.6 导航状态与配置注入的关联

配置注入不直接影响导航选中态，但影响导航相关的功能：

1. **OIDC 配置**: 如果 `config.oidc = true`，登录页会显示 OIDC 登录按钮，但不影响导航本身
2. **注册配置**: `config.register` 控制是否显示注册按钮，不影响导航
3. **版本信息**: 显示在 Header 中，与导航无关

---

### 6.7 为什么没有选中态？（设计推测）

可能的原因：

1. **简约设计**: Gotify 是消息推送服务，导航不是核心交互，用户更关注消息内容
2. **标题替代**: Messages 页面的标题（`DefaultPage title={name}`）已经表明当前位置
3. **优先级低**: 功能优先级低于消息推送、插件系统等核心功能
4. **技术债务**: 可能是待实现的功能，目前仅有基础的路由跳转

---

### 6.8 如何添加选中态高亮（实现方案）

如果需要添加选中态高亮，有两种主流方案：

#### 方案 A：使用 NavLink（推荐）

```tsx
// 修改 Navigation.tsx
import {NavLink} from 'react-router-dom';

// 对于 "All Messages"
<NavLink to="/" className={({isActive}) => `${classes.link} ${isActive ? 'active' : ''}`}>
    <ListItemButton selected={location.pathname === '/'}>
        <ListItemText primary="All Messages" />
    </ListItemButton>
</NavLink>

// 对于应用列表
{apps.map((app) => (
    <NavLink
        key={app.id}
        to={`/messages/${app.id}`}
        className={({isActive}) => `${classes.link} item ${isActive ? 'active' : ''}`}>
        <ListItemButton selected={location.pathname === `/messages/${app.id}`}>
            {/* ... */}
        </ListItemButton>
    </NavLink>
))}
```

#### 方案 B：使用 useLocation + 手动判断

```tsx
import {useLocation} from 'react-router-dom';

const Navigation = observer(({loggedIn, navOpen, setNavOpen}: IProps) => {
    const location = useLocation();
    const {appStore} = useStores();
    const apps = appStore.getItems();
    
    const isAppActive = (appId: number) => location.pathname === `/messages/${appId}`;
    const isAllActive = () => location.pathname === '/';
    
    return (
        <>
            <Link to="/">
                <ListItemButton selected={isAllActive()}>
                    <ListItemText primary="All Messages" />
                </ListItemButton>
            </Link>
            {apps.map((app) => (
                <Link key={app.id} to={`/messages/${app.id}`}>
                    <ListItemButton selected={isAppActive(app.id)}>
                        {/* ... */}
                    </ListItemButton>
                </Link>
            ))}
        </>
    );
});
```

---

### 6.9 当前选中态问题总结

| 方面 | 状态 |
|-----|------|
| 路由跳转 | ✅ 正常工作 |
| 视觉高亮 | ❌ 未实现 |
| 无障碍 (aria-selected) | ❌ 未设置 |
| 键盘导航焦点 | ⚠️ 依赖浏览器默认行为 |
| 与登录态关联 | ✅ 登录后才显示真实导航项 |
| 与后端鉴权关联 | ✅ 导航目标受路由守卫保护 |

---

## 七、全局错误处理与重认证

### 7.1 Axios 拦截器

**文件**: `ui/src/apiAuth.ts:5-24`

```typescript
export const initAxios = (currentUser: CurrentUser, snack: SnackReporter) => {
    axios.interceptors.response.use(undefined, (error) => {
        if (!error.response) {
            snack('Gotify server is not reachable, try refreshing the page.');
            return Promise.reject(error);
        }
        const status = error.response.status;
        if (status === 401) {
            currentUser.tryAuthenticate().then(() => snack('Could not complete request.'));
        }
        if (status === 400 || status === 403 || status === 500) {
            snack(error.response.data.error + ': ' + error.response.data.errorDescription);
        }
        return Promise.reject(error);
    });
};
```

### 7.2 连接断开自动重连

**文件**: `ui/src/CurrentUser.ts:140-150`

```typescript
private readonly connectionError = (message: string) => {
    this.connectionErrorMessage = message;
    if (this.reconnectTimeoutId !== null) {
        window.clearTimeout(this.reconnectTimeoutId);
    }
    this.reconnectTimeoutId = window.setTimeout(
        () => this.tryReconnect(true),
        this.reconnectTime
    );
    this.reconnectTime = Math.min(this.reconnectTime * 2, 120000); // 指数退避，最大 2 分钟
};
```

---

## 八、状态关联总览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              前端 UI 层                                 │
├─────────────────┬─────────────────┬─────────────────┬───────────────────┤
│   主题管理      │   登录态管理    │  权限提升管理   │  导航状态         │
│  (localStorage) │  (MobX Store)   │  (MobX Store)   │  (React Router)   │
└────────┬────────┴────────┬────────┴────────┬────────┴────────┬──────────┘
         │                 │                 │                 │
         ▼                 ▼                 ▼                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              React 组件层                                │
│  Layout (路由守卫) ── Header ── Navigation ── 各业务页面                 │
└────────┬────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              HTTP 层                                    │
│  Axios (Cookie 自动携带) ── 拦截器 (401 重认证)                         │
└────────┬────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              后端 API 层                                │
│  Gin 中间件 ── Auth 鉴权 ── Session 管理 ── 数据库操作                   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 九、关键交互流程

### 9.1 页面初始化流程

```
1. 加载 index.html → 注入 window.config
2. 初始化所有 Stores
3. 注册 Axios 拦截器
4. 调用 currentUser.tryAuthenticate()
   ├─ 发送 GET /current/user（自动携带 Cookie）
   ├─ 成功 → loggedIn = true → 加载数据、连接 WebSocket
   └─ 失败 → loggedIn = false → 路由守卫跳转到 /login
```

### 9.2 登录流程

```
用户输入账号密码
    ↓
currentUser.login() 发送 POST /auth/local/login (Basic Auth)
    ↓
后端验证 → 创建 Client → 设置 Cookie → 返回用户信息
    ↓
前端 loggedIn = true → 路由守卫放行 → 加载数据
```

### 9.3 访问敏感页面（如 /users）

```
用户访问 /users
    ↓
RequireAuth 检查 loggedIn → 未登录跳 /login
    ↓
RequireElevation 检查 elevated → 未提升显示 ElevationForm
    ↓
用户输入密码提升权限
    ↓
elevated = true → 显示用户管理页面
```

---

## 十、总结

| 状态类型 | 存储方式 | 管理位置 | 后端关联 |
|---------|---------|---------|---------|
| 主题 | localStorage | Layout 组件内部 | 无 |
| 登录态 | MobX (CurrentUser) + HttpOnly Cookie | CurrentUser Store | Session API、Auth 中间件 |
| 权限提升 | MobX (ElevateStore) + 数据库字段 | ElevateStore | Elevate API、RequireElevatedClient 中间件 |
| 导航选中 | 无（当前未实现） | - | - |

### 设计特点

1. **安全性**: 使用 HttpOnly Cookie 存储 Token，防止 XSS 攻击
2. **分层鉴权**: 普通登录 + 权限提升的双因子模式，保护敏感操作
3. **响应式**: MobX observable + reaction 实现状态变化的自动响应
4. **容错机制**: 连接断开自动重连（指数退避），401 自动尝试重认证
5. **配置注入**: 后端动态注入配置，无需前端重新构建
