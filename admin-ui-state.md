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

## 六