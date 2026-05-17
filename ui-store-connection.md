# Gotify UI Store 连接与错误处理机制

## 1. 概述

Gotify 前端采用 MobX 进行状态管理，通过多个 Store 协同工作来处理 API 调用失败后的错误传播和状态恢复。本文档详细说明错误状态的生命周期、提示条的显示条件，以及连接恢复后的状态清理机制。

## 2. Store 架构

### 2.1 核心 Store 列表

| Store 名称 | 职责 | 错误处理方式 |
|-----------|------|-------------|
| `CurrentUser` | 管理用户认证状态、连接错误 | 集中管理连接错误状态，触发重连逻辑 |
| `AppStore` | 应用管理 | 继承 BaseStore，通过 axios 拦截器处理错误 |
| `ClientStore` | 客户端管理 | 继承 BaseStore，通过 axios 拦截器处理错误 |
| `MessagesStore` | 消息管理 | 独立状态管理，通过 axios 拦截器处理错误 |
| `UserStore` | 用户管理 | 继承 BaseStore，通过 axios 拦截器处理错误 |
| `PluginStore` | 插件管理 | 继承 BaseStore，通过 axios 拦截器处理错误 |
| `ElevateStore` | 权限提升管理 | 直接调用 API，错误通过 tryAuthenticate 处理 |
| `WebSocketStore` | WebSocket 连接管理 | 连接失败后通过 CurrentUser.tryAuthenticate 触发重连 |
| `SnackManager` | 消息提示管理 | 统一显示操作结果和错误信息 |

### 2.2 Store 初始化流程 (`index.tsx:28-51`)

```javascript
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

## 3. 错误状态生命周期

### 3.1 错误触发点

#### 3.1.1 Axios 全局拦截器 (`apiAuth.ts:1-24`)

所有 Store 的 API 请求都通过 axios 发出，全局拦截器统一处理错误：

```javascript
axios.interceptors.response.use(undefined, (error) => {
    if (!error.response) {
        // 网络错误 - 显示 Snackbar 提示
        snack('Gotify server is not reachable, try refreshing the page.');
        return Promise.reject(error);
    }

    const status = error.response.status;

    if (status === 401) {
        // 未授权 - 尝试重新认证
        currentUser.tryAuthenticate().then(() => snack('Could not complete request.'));
    }

    if (status === 400 || status === 403 || status === 500) {
        // 业务错误 - 显示具体错误信息
        snack(error.response.data.error + ': ' + error.response.data.errorDescription);
    }

    return Promise.reject(error);
});
```

#### 3.1.2 CurrentUser 认证失败 (`CurrentUser.ts:78-115`)

`tryAuthenticate` 方法是连接错误的主要入口：

```javascript
public tryAuthenticate = async (): Promise<AxiosResponse<ICurrentUser>> => {
    return axios
        .create()
        .get(config.get('url') + 'current/user')
        .then(
            action((passThrough) => {
                // 认证成功 - 清除错误状态
                this.user = passThrough.data;
                this.loggedIn = true;
                this.authenticating = false;
                this.connectionErrorMessage = null;
                this.reconnectTime = 7500;
                return passThrough;
            })
        )
        .catch(
            action((error: AxiosError) => {
                this.authenticating = false;
                if (!error || !error.response) {
                    // 网络错误 - 设置连接错误消息
                    this.connectionError('No network connection or server unavailable.');
                    return Promise.reject(error);
                }

                if (error.response.status >= 500) {
                    // 服务器错误 - 设置连接错误消息
                    this.connectionError(
                        `${error.response.statusText} (code: ${error.response.status}).`
                    );
                    return Promise.reject(error);
                }

                // 4xx 错误 - 清除连接错误消息
                this.connectionErrorMessage = null;

                if (error.response.status >= 400 && error.response.status < 500) {
                    // 认证失败 - 登出用户
                    this.logout();
                }
                return Promise.reject(error);
            })
        );
};
```

#### 3.1.3 WebSocket 连接失败 (`WebSocketStore.ts:32-48`)

```javascript
ws.onclose = () => {
    this.wsActive = false;
    if (!this.currentUser.loggedIn) {
        return;
    }
    this.currentUser
        .tryAuthenticate()
        .then(() => {
            this.snack('WebSocket connection closed, trying again in 30 seconds.');
            setTimeout(() => this.listen(callback), 30000);
        })
        .catch((error: AxiosError) => {
            if (error?.response?.status === 401) {
                this.snack('Could not authenticate with client token, logging out.');
            }
        });
};
```

### 3.2 错误状态设置 (`CurrentUser.ts:140-150`)

```javascript
private readonly connectionError = (message: string) => {
    // 1. 设置错误消息
    this.connectionErrorMessage = message;
    
    // 2. 清除之前的重连定时器
    if (this.reconnectTimeoutId !== null) {
        window.clearTimeout(this.reconnectTimeoutId);
    }
    
    // 3. 设置新的重连定时器（指数退避）
    this.reconnectTimeoutId = window.setTimeout(
        () => this.tryReconnect(true),
        this.reconnectTime
    );
    
    // 4. 增加下次重连间隔（最大 120 秒）
    this.reconnectTime = Math.min(this.reconnectTime * 2, 120000);
};
```

### 3.3 错误状态流转图

```
API 请求失败
    ↓
axios 拦截器捕获
    ↓
    ├─→ 401 → tryAuthenticate() → 触发 connectionError()
    ├─→ 500 → connectionError()
    ├─→ 400/403 → Snackbar 显示错误
    └─→ 网络错误 → Snackbar 显示错误
    ↓
connectionErrorMessage 设置为非 null
    ↓
Layout 组件显示 ConnectionErrorBanner
    ↓
启动重连定时器（7.5s → 15s → 30s → ... → 120s）
    ↓
tryReconnect() 调用 tryAuthenticate()
    ↓
认证成功 → connectionErrorMessage = null
    ↓
MobX reaction 触发 → 清理所有 Store 状态 → 重新加载数据
```

## 4. 提示条出现条件

### 4.1 ConnectionErrorBanner（顶部红色错误条）

**显示位置**: `Layout.tsx:107-113`

```javascript
{!connectionErrorMessage ? null : (
    <ConnectionErrorBanner
        height={64}
        retry={() => tryReconnect()}
        message={connectionErrorMessage}
    />
)}
```

**触发条件**：
- `CurrentUser.connectionErrorMessage !== null`
- 主要由以下情况触发：
  1. `tryAuthenticate()` 遇到网络错误（无 response）
  2. `tryAuthenticate()` 遇到 5xx 服务器错误
  3. 初始认证失败

**清除条件**：
- `tryAuthenticate()` 成功时设置 `connectionErrorMessage = null`
- 用户点击 "Retry" 按钮调用 `tryReconnect()`

### 4.2 Snackbar（底部消息提示）

**管理组件**: `SnackManager.ts:1-11`

```javascript
export class SnackManager {
    public snack: SnackReporter = (message: string): void => {
        enqueueSnackbar({message, variant: 'info'});
    };
}
```

**显示场景**：

| 场景 | 消息内容 | 来源文件 |
|------|---------|---------|
| 注册成功 | "User Created. Logging in..." | `CurrentUser.ts:24` |
| 注册失败 - 网络错误 | "No network connection or server unavailable." | `CurrentUser.ts:30` |
| 注册失败 - 业务错误 | "Register failed: {error}: {errorDescription}" | `CurrentUser.ts:35-37` |
| 客户端创建成功 | "A client named '{name}' was created for your session." | `CurrentUser.ts:62` |
| 登录失败 | "Login failed" | `CurrentUser.ts:73` |
| 重连失败 | "Reconnect failed" | `CurrentUser.ts:135` |
| 密码修改成功 | "Password changed" | `CurrentUser.ts:129` |
| 应用删除成功 | "Application deleted" | `AppStore.ts:25` |
| 应用图片更新 | "Application image updated" | `AppStore.ts:36` |
| 应用图片删除 | "Application image deleted" | `AppStore.ts:43` |
| 应用更新成功 | "Application updated" | `AppStore.ts:83` |
| 应用创建成功 | "Application created" | `AppStore.ts:98` |
| 客户端删除成功 | "Client deleted" | `ClientStore.ts:19` |
| 客户端更新成功 | "Client updated" | `ClientStore.ts:26` |
| 客户端添加成功 | "Client added" | `ClientStore.ts:39` |
| 取消客户端提升 | "Canceled client elevation" | `ClientStore.ts:47` |
| 客户端提升成功 | "Client elevated" | `ClientStore.ts:49` |
| 删除所有消息 | "Deleted all messages" | `MessagesStore.ts:87` |
| 删除应用所有消息 | "Deleted all messages from {appName}" | `MessagesStore.ts:91` |
| 消息发送成功 | "Message sent to {appName}" | `MessagesStore.ts:153` |
| 插件配置更新 | "Plugin config updated" | `PluginStore.ts:39` |
| 插件启用/禁用 | "Plugin {enabled/disabled}" | `PluginStore.ts:46` |
| 用户删除成功 | "User deleted" | `UserStore.ts:19` |
| 用户创建成功 | "User created" | `UserStore.ts:26` |
| 用户更新成功 | "User updated" | `UserStore.ts:33` |
| WebSocket 关闭提示 | "WebSocket connection closed, trying again in 30 seconds." | `WebSocketStore.ts:40` |
| WebSocket 认证失败 | "Could not authenticate with client token, logging out." | `WebSocketStore.ts:45` |
| 服务器不可达 | "Gotify server is not reachable, try refreshing the page." | `apiAuth.ts:8` |
| 请求完成失败 | "Could not complete request." | `apiAuth.ts:15` |
| API 业务错误 | "{error}: {errorDescription}" | `apiAuth.ts:19` |
| Popup 被阻止 | "Popup was blocked. Please allow popups for this site and try again." | `ElevateStore.ts:60` |
| OIDC 提升未完成 | "OIDC elevation was not completed." | `ElevateStore.ts:85` |

## 5. 连接恢复后的状态清理

### 5.1 触发恢复的条件

当 `connectionErrorMessage` 从非 null 变为 null 时，MobX reaction 被触发：

**Reaction 定义**: `reactions.ts:64-73`

```javascript
reaction(
    () => stores.currentUser.connectionErrorMessage,
    (connectionErrorMessage) => {
        if (!connectionErrorMessage) {
            // 错误已清除 - 清理并重新加载所有数据
            clearAll();
            loadAll();
            stores.currentUser.refreshKey++;
        }
    }
);
```

### 5.2 清理操作 (`reactions.ts:11-17`)

```javascript
const clearAll = () => {
    stores.messagesStore.clearAll();      // 清空所有消息状态
    stores.appStore.clear();              // 清空应用列表
    stores.clientStore.clear();           // 清空客户端列表
    stores.userStore.clear();             // 清空用户列表
    stores.wsStore.close();               // 关闭 WebSocket 连接
};
```

### 5.3 重新加载操作 (`reactions.ts:22-35`)

```javascript
const loadAll = () => {
    stores.wsStore.listen((message) => {  // 重新建立 WebSocket 连接
        stores.messagesStore.publishSingleMessage(message);
        Notifications.notifyNewMessage(message);
        // ... 播放通知声音
    });
    stores.appStore.refresh();            // 重新加载应用列表
};
```

### 5.4 组件强制刷新

通过 `refreshKey` 变化触发整个应用的重新渲染：

**Layout.tsx:106**:
```javascript
<div key={refreshKey}>
    {/* 所有子组件都会因为 key 变化而重新挂载 */}
</div>
```

这会导致：
- 所有页面组件的 `useEffect` 重新执行
- 各页面根据需要重新调用 `loadMore` 等方法加载数据

### 5.5 各 Store 的 clear() 实现

#### BaseStore.ts:55-58（基类）
```javascript
@action
public clear = (): void => {
    this.items = [];  // 清空数据数组
};
```

#### MessagesStore.ts:157-160
```javascript
@action
public clearAll = () => {
    this.state = {};  // 清空所有应用的消息状态
    this.createEmptyStatesForApps(this.appStore.getItems());
};
```

#### WebSocketStore.ts:53
```javascript
public close = () => this.ws?.close(1000, 'WebSocketStore#close');
```

## 6. 登出时的状态清理

除了连接恢复时的清理，用户登出时也会触发完整的状态清理：

**Reaction 定义**: `reactions.ts:37-46`

```javascript
reaction(
    () => stores.currentUser.loggedIn,
    (loggedIn) => {
        if (loggedIn) {
            loadAll();    // 登录成功 - 加载数据
        } else {
            clearAll();   // 登出 - 清理所有状态
        }
    }
);
```

## 7. 关键设计要点

### 7.1 单一错误源
- 连接错误状态集中在 `CurrentUser.connectionErrorMessage`
- 避免了多个 Store 独立管理连接状态的复杂性

### 7.2 指数退避重连
- 初始重连间隔：7.5 秒
- 每次失败后间隔翻倍
- 最大间隔：120 秒
- 避免对服务器造成过大压力

### 7.3 状态一致性保证
- 连接恢复后强制清理所有 Store 状态
- 通过 `refreshKey` 强制组件重新挂载
- 确保 UI 显示的是最新的服务器数据

### 7.4 错误分层处理
1. **连接错误**：顶部红色 Banner 显示 + 自动重连
2. **业务错误**：底部 Snackbar 显示具体信息
3. **操作反馈**：底部 Snackbar 显示操作结果

## 8. 数据流总结

```
┌─────────────────────────────────────────────────────────────────┐
│                        API 请求失败                               │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Axios 全局拦截器                               │
│  ├─ 401 → tryAuthenticate()                                     │
│  ├─ 500/网络错误 → connectionError()                             │
│  └─ 400/403 → Snackbar 显示错误                                  │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                CurrentUser.connectionErrorMessage                │
│  ├─ 设置错误消息                                                 │
│  ├─ 启动指数退避重连定时器                                       │
│  └─ 触发 Layout 显示 ConnectionErrorBanner                       │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                      重连成功 (tryAuthenticate)                   │
│  ├─ connectionErrorMessage = null                                │
│  └─ reconnectTime 重置为 7500ms                                  │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                   MobX Reaction 触发                              │
│  ├─ clearAll() → 清理所有 Store 数据                             │
│  ├─ loadAll() → 重新加载应用 + 重建 WebSocket                    │
│  └─ refreshKey++ → 强制组件重新挂载                               │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                        应用恢复正常                               │
└─────────────────────────────────────────────────────────────────┘
```
