# Gotify UI Store 连接与错误处理机制

## 1. 概述

Gotify 前端采用 MobX 进行状态管理，通过多个 Store 协同工作来处理 API 调用失败后的错误传播和状态恢复。本文档按 Store 维度梳理失败分流机制，明确错误状态生命周期、提示条显示条件，以及连接恢复后的状态清理范围。

**排障核心结论**：
- 普通 API 500 错误 ≠ 触发连接错误 Banner
- 只有 `tryAuthenticate()` 认证请求的 500/网络错误才会触发连接错误状态
- Retry 成功 ≠ 立即清理数据，需 `connectionErrorMessage` 从非 null → null 才触发
- ⚠️ 已知问题 1：手动 Retry 成功后，已挂起的自动重连定时器不会被清理
- ⚠️ 已知问题 2：连接恢复时，`pendingDeletes`（待删除消息）不会被清理，可能导致消息"消失"

## 2. Store 架构与错误处理分流

### 2.1 核心 Store 错误处理矩阵

| Store 名称 | API 请求方式 | 错误处理分流 |
|-----------|------------|------------|
| **CurrentUser** | `axios.create()` 独立实例 | ✅ 触发连接错误状态 + 重连 |
| **AppStore** | 共享 axios 实例 | ❌ 仅 Snackbar，不触发连接错误 |
| **ClientStore** | 共享 axios 实例 | ❌ 仅 Snackbar，不触发连接错误 |
| **MessagesStore** | 共享 axios 实例 | ❌ 仅 Snackbar，不触发连接错误 |
| **UserStore** | 共享 axios 实例 | ❌ 仅 Snackbar，不触发连接错误 |
| **PluginStore** | 共享 axios 实例 | ❌ 仅 Snackbar，不触发连接错误 |
| **ElevateStore** | `axios.create()` 独立实例 | ❌ 错误通过 tryAuthenticate 传导 |
| **WebSocketStore** | WebSocket 协议 | ✅ 连接关闭触发 tryAuthenticate |
| **SnackManager** | 纯 UI 组件 | ✅ 统一显示所有消息提示 |

### 2.2 关键区分：两个 axios 实例

```
┌─────────────────────────────────────────────────────────────┐
│                    共享 axios 实例                            │
│  ├─ AppStore、ClientStore、MessagesStore、UserStore、PluginStore │
│  └─ 错误处理：apiAuth.ts 拦截器 → 仅 Snackbar                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│               CurrentUser 独立 axios 实例                      │
│  ├─ tryAuthenticate()、login()、register() 使用 axios.create() │
│  └─ 错误处理：直接触发 connectionError() → 连接错误 Banner + 重连 │
└─────────────────────────────────────────────────────────────┘
```

## 3. 错误触发分流详解

### 3.1 Axios 全局拦截器分流逻辑 (`apiAuth.ts:1-24`)

**仅作用于共享 axios 实例的所有 Store**

```javascript
axios.interceptors.response.use(undefined, (error) => {
    // ── 分支 1：纯网络错误（无 response） ──
    if (!error.response) {
        snack('Gotify server is not reachable, try refreshing the page.');
        return Promise.reject(error);
        // ↑ 仅 Snackbar，不触发连接错误状态
    }

    const status = error.response.status;

    // ── 分支 2：401 未授权 ──
    if (status === 401) {
        currentUser.tryAuthenticate().then(() => snack('Could not complete request.'));
        // ↑ 间接触发：调用 tryAuthenticate() 可能进入连接错误流程
    }

    // ── 分支 3：400/403/500 业务错误 ──
    if (status === 400 || status === 403 || status === 500) {
        snack(error.response.data.error + ': ' + error.response.data.errorDescription);
        // ↑ 仅 Snackbar，不触发连接错误状态
        // ⚠️  关键：此处 500 错误不会触发 connectionErrorMessage！
    }

    return Promise.reject(error);
});
```

### 3.2 CurrentUser 认证请求分流 (`CurrentUser.ts:78-115`)

**唯一能触发连接错误状态的入口**

```javascript
public tryAuthenticate = async (): Promise<AxiosResponse<ICurrentUser>> => {
    return axios
        .create()  // ⚠️  独立实例，不经过全局拦截器
        .get(config.get('url') + 'current/user')
        .then(
            action((passThrough) => {
                // ── 成功分支 ──
                this.user = passThrough.data;
                this.loggedIn = true;
                this.authenticating = false;
                this.connectionErrorMessage = null;  // 清除错误状态
                this.reconnectTime = 7500;
                return passThrough;
            })
        )
        .catch(
            action((error: AxiosError) => {
                this.authenticating = false;

                // ── 分支 A：触发连接错误（显示 Banner + 重连） ──
                if (!error || !error.response) {
                    // 网络错误
                    this.connectionError('No network connection or server unavailable.');
                    return Promise.reject(error);
                }
                if (error.response.status >= 500) {
                    // 服务器错误
                    this.connectionError(
                        `${error.response.statusText} (code: ${error.response.status}).`
                    );
                    return Promise.reject(error);
                }

                // ── 分支 B：清除连接错误状态 ──
                this.connectionErrorMessage = null;

                // ── 分支 C：4xx 客户端错误 → 登出 ──
                if (error.response.status >= 400 && error.response.status < 500) {
                    this.logout();
                }
                return Promise.reject(error);
            })
        );
};
```

### 3.3 错误分流决策树

```
任意 API 请求失败
    │
    ├─→ 请求来自哪个 axios 实例？
    │    │
    │    ├─→ 共享实例（AppStore/ClientStore/等）
    │    │    │
    │    │    ├─→ 无 response（网络错误）→ Snackbar：服务器不可达
    │    │    ├─→ 401 → 调用 tryAuthenticate() → 可能进入连接错误
    │    │    ├─→ 400/403/500 → Snackbar：具体错误信息
    │    │    └─→ 其他 → 直接 reject
    │    │
    │    └─→ CurrentUser 独立实例（tryAuthenticate/login/register）
    │         │
    │         ├─→ login/register → 自行 catch → Snackbar
    │         │
    │         └─→ tryAuthenticate →
    │              │
    │              ├─→ 网络错误 / 5xx → ✅ connectionError() → Banner + 重连
    │              ├─→ 4xx → connectionErrorMessage = null + logout
    │              └─→ 成功 → connectionErrorMessage = null
    │
    └─→ WebSocket 连接关闭
         │
         └─→ 调用 tryAuthenticate() → 同上分流
```

### 3.4 连接错误状态设置 (`CurrentUser.ts:140-150`)

```javascript
private readonly connectionError = (message: string) => {
    // 1. 设置错误消息 → 触发 Banner 显示
    this.connectionErrorMessage = message;

    // 2. 清除之前的重连定时器
    if (this.reconnectTimeoutId !== null) {
        window.clearTimeout(this.reconnectTimeoutId);
    }

    // 3. 设置新的重连定时器（指数退避）
    this.reconnectTimeoutId = window.setTimeout(
        () => this.tryReconnect(true),  // quiet 模式，失败不弹 Snackbar
        this.reconnectTime
    );

    // 4. 增加下次重连间隔（7.5s → 15s → 30s → 60s → 120s）
    this.reconnectTime = Math.min(this.reconnectTime * 2, 120000);
};
```

### 3.5 重连定时器清理的已知问题

**关键发现**：`tryAuthenticate()` 成功时**不会清理已挂起的重连定时器**，`logout()` 也不会清理。

```javascript
// tryAuthenticate 成功分支（CurrentUser.ts:83-90）
action((passThrough) => {
    this.user = passThrough.data;
    this.loggedIn = true;
    this.authenticating = false;
    this.connectionErrorMessage = null;
    this.reconnectTime = 7500;
    // ⚠️  缺少：window.clearTimeout(this.reconnectTimeoutId)
    // ⚠️  缺少：this.reconnectTimeoutId = null
    return passThrough;
})

// logout 实现（CurrentUser.ts:117-124）
public logout = async () => {
    if (this.loggedIn) {
        runInAction(() => {
            this.loggedIn = false;
        });
        await axios.post(config.get('url') + 'auth/logout').catch(() => Promise.resolve());
        // ⚠️  缺少：window.clearTimeout(this.reconnectTimeoutId)
        // ⚠️  缺少：this.reconnectTimeoutId = null
    }
};
```

#### 场景 A：手动 Retry 成功后定时器残留

1. 网络断开 → `connectionError()` 被调用 → 设置 15 秒后自动重连的定时器
2. 第 5 秒时网络恢复 → 用户手动点击 Retry 按钮 → `tryReconnect(false)` → `tryAuthenticate()` 成功
3. `connectionErrorMessage = null` → Reaction 触发 `clearAll()` + `loadAll()` → 数据恢复正常
4. **第 15 秒时**：之前挂起的定时器触发 → 再次调用 `tryReconnect(true)` → 再次发送 `/current/user` 请求

**影响**：
- ✅ 不会导致数据重复清理：因为 `connectionErrorMessage` 已经是 null，再次设置为 null 不会触发 MobX reaction
- ❌ 会产生一次多余的 API 请求（`GET /current/user`）
- ❌ 如果此时网络再次波动，可能意外触发连接错误状态

**排障提示**：如果在网络恢复后观察到"多余的认证请求"，这是预期行为，不是 Bug，但可以优化。

#### 场景 B：已登出但旧重连定时器还在（边界场景）

**场景复现**：
1. 网络断开 → `connectionError()` 被调用 → 设置 15 秒后自动重连的定时器
2. 第 5 秒时，用户手动点击登出按钮（或 4xx 错误触发自动登出）
3. `logout()` 被调用 → `loggedIn = false` → Reaction 触发 `clearAll()`
4. ⚠️ **关键**：`reconnectTimeoutId` 没有被清理，定时器仍然挂起
5. **第 15 秒时**：定时器触发 → 调用 `tryReconnect(true)` → 调用 `tryAuthenticate()`

**后续流程分支**：

```
tryAuthenticate() 发送 GET /current/user
    ↓
    ├─→ 服务器正常，返回 401 Unauthorized（因为已登出）
    │    ↓
    │    connectionErrorMessage = null
    │    ↓
    │    调用 logout()（但已经是登出状态，无额外影响）
    │    ↓
    │    ✅ 不会显示连接错误 Banner
    │    ✅ 不会影响登录页
    │
    ├─→ 网络错误（无 response）或 5xx 服务器错误
    │    ↓
    │    调用 connectionError("错误信息")
    │    ↓
    │    connectionErrorMessage = "错误信息" → 非空
    │    ↓
    │    ❗ 登录页顶部显示红色连接错误 Banner
    │    ↓
    │    设置新的重连定时器，继续指数退避重试
    │
    └─→ 其他 4xx 错误
         ↓
         connectionErrorMessage = null
         ↓
         调用 logout()
         ↓
         ✅ 无不良影响
```

**对登录页的影响**：
- ✅ 如果服务器正常（返回 401）：登录页不受影响，用户正常登录
- ❌ 如果网络断开或服务器 5xx：登录页顶部显示连接错误 Banner，可能误导用户以为登录服务不可用
- ❌ 自动重连会持续在后台进行，产生多余的 API 请求

**排障判断点**：
1. 观察 Network 面板：登出后是否仍有 `/current/user` 请求发出
2. 检查 `connectionErrorMessage` 的值：登出状态下是否意外变为非空
3. 确认登出前是否有连接错误状态（连接错误 Banner 曾显示过）

**临时解决方案**：登出后刷新页面（F5），清除所有定时器和状态

### 3.6 完整错误状态生命周期

```
┌─────────────────────────────────────────────────────────────────┐
│                      初始状态                                     │
│  connectionErrorMessage = null                                   │
│  reconnectTimeoutId = null                                       │
│  reconnectTime = 7500ms                                          │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                tryAuthenticate() 失败（网络/5xx）                  │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                   connectionError() 被调用                        │
│  ├─ connectionErrorMessage = "错误信息"                           │
│  ├─ 清除已有 reconnectTimeoutId（如果有）                          │
│  ├─ 设置新的 reconnectTimeoutId（T 秒后触发）                      │
│  └─ reconnectTime = min(reconnectTime * 2, 120000)               │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                      等待重连                                      │
│  连接错误 Banner 显示                                              │
│  倒计时中... 用户可以点击 Retry 手动触发                            │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
              ┌──────────────┴──────────────┐
              ↓                             ↓
┌───────────────────────────┐   ┌───────────────────────────┐
│   定时器到点自动触发       │   │   用户点击 Retry 手动触发   │
│   tryReconnect(true)      │   │   tryReconnect(false)     │
└─────────────┬─────────────┘   └─────────────┬─────────────┘
              ↓                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                     tryAuthenticate()                            │
│  ├─ 成功 → connectionErrorMessage = null                         │
│  │    ├─ 如果之前是错误状态 → Reaction 触发清理+加载              │
│  │    └─ ⚠️  reconnectTimeoutId 未清理（可能后续再触发一次）       │
│  │
│  ├─ 网络/5xx 失败 → connectionError()                            │
│  │    └─ 重置定时器，继续退避重试                                 │
│  │
│  └─ 4xx 失败 → connectionErrorMessage = null + logout()          │
│       └─ 如果之前是错误状态 → Reaction 触发清理                   │
└─────────────────────────────────────────────────────────────────┘
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

**✅ 触发条件（必须同时满足）**：
1. `CurrentUser.connectionErrorMessage !== null`
2. 错误源自 `tryAuthenticate()` 调用

**❌ 不会触发的场景**：
- 普通业务 API（AppStore/ClientStore 等）的 500 错误
- 普通业务 API 的网络错误
- 这些场景只弹 Snackbar，不显示顶部 Banner

**清除条件**：
- `tryAuthenticate()` 成功时设置 `connectionErrorMessage = null`
- `tryAuthenticate()` 遇到 4xx 错误时设置 `connectionErrorMessage = null`
- 用户点击 "Retry" 按钮调用 `tryReconnect()` → 内部调用 `tryAuthenticate()`

### 4.2 Snackbar（底部消息提示）

**管理组件**: `SnackManager.ts:1-11`

**所有 Store 的所有错误/成功反馈都走这个通道**

| 来源 Store | 场景 | 消息内容 |
|-----------|------|---------|
| **CurrentUser** | 注册成功 | "User Created. Logging in..." |
| **CurrentUser** | 注册失败 - 网络错误 | "No network connection or server unavailable." |
| **CurrentUser** | 注册失败 - 业务错误 | "Register failed: {error}: {errorDescription}" |
| **CurrentUser** | 客户端创建成功 | "A client named '{name}' was created for your session." |
| **CurrentUser** | 登录失败 | "Login failed" |
| **CurrentUser** | 手动重连失败 | "Reconnect failed" |
| **CurrentUser** | 密码修改成功 | "Password changed" |
| **AppStore** | 应用删除/更新/创建成功 | "Application {deleted/updated/created}" |
| **AppStore** | 应用图片更新/删除成功 | "Application image {updated/deleted}" |
| **ClientStore** | 客户端删除/更新/添加成功 | "Client {deleted/updated/added}" |
| **ClientStore** | 客户端权限提升 | "{Canceled client elevation / Client elevated}" |
| **MessagesStore** | 删除消息 | "Deleted {all messages / all messages from app}" |
| **MessagesStore** | 消息发送成功 | "Message sent to {appName}" |
| **PluginStore** | 插件配置更新 | "Plugin config updated" |
| **PluginStore** | 插件启用/禁用 | "Plugin {enabled/disabled}" |
| **UserStore** | 用户操作成功 | "User {deleted/created/updated}" |
| **WebSocketStore** | 连接关闭提示 | "WebSocket connection closed, trying again in 30 seconds." |
| **WebSocketStore** | 认证失败登出 | "Could not authenticate with client token, logging out." |
| **apiAuth 拦截器** | 网络错误 | "Gotify server is not reachable, try refreshing the page." |
| **apiAuth 拦截器** | 401 重试失败 | "Could not complete request." |
| **apiAuth 拦截器** | 业务错误 | "{error}: {errorDescription}" |
| **ElevateStore** | Popup 被阻止 | "Popup was blocked. Please allow popups..." |
| **ElevateStore** | OIDC 提升未完成 | "OIDC elevation was not completed." |

## 5. 连接恢复路径详解

### 5.1 恢复触发的精确条件

**MobX Reaction 定义** (`reactions.ts:64-73`)

```javascript
reaction(
    () => stores.currentUser.connectionErrorMessage,  // 监听的 observable
    (connectionErrorMessage) => {                      // 新值
        if (!connectionErrorMessage) {
            // ⚠️  核心前提：connectionErrorMessage 从「有值」→「null」
            // 不是所有 null 都会触发，必须是从非 null 变化到 null
            clearAll();
            loadAll();
            stores.currentUser.refreshKey++;
        }
    }
);
```

**✅ 会触发恢复的场景**：
1. `connectionErrorMessage = "错误信息"` → `tryAuthenticate()` 成功 → `connectionErrorMessage = null`
2. `connectionErrorMessage = "错误信息"` → `tryAuthenticate()` 遇到 4xx → `connectionErrorMessage = null`

**❌ 不会触发恢复的场景**：
1. 初始状态 `connectionErrorMessage = undefined/null` → 保持 null → 不触发
2. `connectionErrorMessage = null` → 直接设置 null → 不触发
3. `connectionErrorMessage = "A 错误"` → 变为 `"B 错误"` → 不触发（因为都不是 null）

### 5.2 Retry 按钮完整流程

```
用户点击 Retry 按钮
    ↓
调用 tryReconnect(quiet = false)
    ↓
调用 tryAuthenticate()
    ↓
    ├─→ 认证成功
    │    ↓
    │    connectionErrorMessage = null
    │    ↓
    │    MobX Reaction 检测到：非 null → null
    │    ↓
    │    ├─→ clearAll() → 清理数据
    │    ├─→ loadAll() → 重新加载
    │    └─→ refreshKey++ → 强制重渲染
    │
    └─→ 认证失败
         ↓
         connectionErrorMessage 保持原值（或更新）
         ↓
         quiet=false → 弹出 Snackbar："Reconnect failed"
         ↓
         ❌ 不执行 clearAll/loadAll
```

### 5.3 状态清理范围详解

#### ✅ 会被清理的状态 (`clearAll()` - `reactions.ts:11-17`)

```javascript
const clearAll = () => {
    // 1. MessagesStore：清空消息状态
    stores.messagesStore.clearAll();
    //    - this.state = {} （所有应用的消息缓存）
    //    - 重新创建空的应用状态
    //    - ⚠️  注意：pendingDeletes 未被清理

    // 2. AppStore：清空应用列表
    stores.appStore.clear();
    //    - this.items = []

    // 3. ClientStore：清空客户端列表
    stores.clientStore.clear();
    //    - this.items = []

    // 4. UserStore：清空用户列表
    stores.userStore.clear();
    //    - this.items = []

    // 5. WebSocketStore：关闭连接
    stores.wsStore.close();
    //    - ws.close(1000, 'WebSocketStore#close')
};
```

#### ❌ 不会被清理的状态

| Store | 未清理内容 | 原因 / 影响 |
|-------|-----------|------------|
| **CurrentUser** | `user` 对象、`loggedIn`、`authenticating` | 认证成功时已更新，无需额外清理 |
| **ElevateStore** | `elevated`、`oidcElevatePending` | 权限状态独立，与连接恢复无关 |
| **SnackManager** | 已显示的 Snackbar | Snackbar 自行管理生命周期 |
| **PluginStore** | `items`（插件列表） | ❗ 潜在遗漏：clearAll 中未调用，但 refreshKey++ 后页面会重新加载 |
| **MessagesStore** | `pendingDeletes`（待删除消息） | ❗ 重要 Bug：会导致消息"消失"，详见 5.4 |

### 5.4 MessagesStore pendingDeletes 清理问题详解

**关键发现**：`MessagesStore.clearAll()` 只清空了 `this.state`，但**没有清理 `pendingDeletes`**。

```javascript
// MessagesStore.clearAll() 实现（MessagesStore.ts:157-160）
@action
public clearAll = () => {
    this.state = {};  // ✅ 清理消息缓存
    this.createEmptyStatesForApps(this.appStore.getItems());
    // ⚠️  缺少：this.pendingDeletes.clear();
};
```

**pendingDeletes 的作用**：
- 存储用户点击删除但还在 Undo 倒计时内的消息
- `visible(messageId)` 方法：`return !this.pendingDeletes.has(messageId)`
- 消息列表渲染时会过滤掉 `pendingDeletes` 中的消息：
  ```javascript
  // MessagesStore.ts:204
  .messages.filter((message) => !this.pendingDeletes.has(message.id))
  ```

**场景复现**：
1. 用户删除一条消息 → 消息进入 `pendingDeletes` → 列表中隐藏（等待 Undo 或 5 秒后自动删除）
2. 在 Undo 倒计时内（< 5 秒），网络断开 → 连接错误 Banner 出现
3. 网络恢复 → 连接恢复 → `clearAll()` 被调用 → `this.state = {}`，但 `pendingDeletes` 保留
4. `refreshKey++` → 页面重新挂载 → 重新调用 `loadMore()` 从服务器加载消息
5. **问题**：服务器返回的消息列表中包含刚才"删除"的消息（因为服务器还没收到删除请求），但由于 `pendingDeletes` 中还有这条消息的 ID，`get()` 方法会过滤掉它
6. **用户可见影响**：这条消息在列表中"消失"了，用户以为被删除了，但实际上服务器上还存在

**其他连锁影响**：
- 如果 Snackbar Undo 按钮还在显示，用户点击 Undo → `cancelPendingDelete()` 会从 `pendingDeletes` 中移除 → 消息重新出现（表现为"消失又回来"）
- 如果 5 秒倒计时结束，`removeSingle()` 会发送删除请求到服务器 → 消息真正被删除 → 下次刷新时消失
- 如果页面被刷新（F5），`pendingDeletes` 丢失 → 消息重新出现（因为服务器上还存在）

**排障提示**：
- 如果用户反馈"消息消失了，但刷新页面又出现"，检查是否在删除消息后立即发生了连接恢复
- 查看 Network 面板：连接恢复后重新加载的消息列表中是否包含该消息 ID
- 确认 `pendingDeletes` Map 的内容（可通过 MobX DevTools 查看）

#### 场景 B：登出再登录跨会话残留（边界场景）

**代码验证结论**：
- ✅ `pendingDeletes` 是 MessagesStore 的类成员，在应用生命周期内只初始化一次（构造函数中）
- ✅ 登出时调用 `clearAll()` → 调用 `messagesStore.clearAll()` → 只清空 `this.state`，**不清理 `pendingDeletes`**
- ✅ 登录时调用 `loadAll()` → **没有任何清理 `pendingDeletes` 的逻辑**
- ✅ `get()` 方法中明确过滤：`.filter((message) => !this.pendingDeletes.has(message.id))`
- ⚠️ **结论**：`pendingDeletes` 会跨会话保留，并继续参与新会话的消息列表过滤

**跨会话可复现实验**：

| 项目 | 内容 |
|------|------|
| **前置条件** | 1. 服务器有 2 个测试用户（用户 A、用户 B）<br>2. 用户 A 有一条消息 ID=123<br>3. 用户 B 有一条消息 ID=123（或服务器重启后 ID 重置，新消息 ID 从 1 开始）<br>4. 浏览器已安装 MobX DevTools（可选） |
| **操作顺序** | 1. 以用户 A 登录，进入消息列表<br>2. 找到 ID=123 的消息，点击删除<br>3. **在 5 秒 Undo 倒计时内**，点击用户头像 → 登出<br>4. 立即以用户 B 登录（或同一用户 A 重新登录）<br>5. 进入消息列表，查看消息 |
| **预期行为** | 用户 B 的消息列表应显示所有消息，包括 ID=123 的消息（与用户 A 的删除操作无关） |
| **实际行为** | 用户 B 的消息列表中 ID=123 的消息被过滤，无法看到<br>刷新页面（F5）后，ID=123 的消息重新出现 |
| **验证方法** | 1. 打开 Network 面板：确认 `/message` 接口返回数据中包含 ID=123<br>2. 打开 MobX DevTools：确认 `pendingDeletes` Map 中仍有 ID=123 的条目<br>3. 在 Console 执行：`stores.messagesStore.pendingDeletes.has(123)` → 返回 `true` |

**用户可见影响**：
- ❌ 新会话中 ID=123 的消息"消失"，用户看不到这条消息
- ❌ 如果是不同用户登录，前一用户的待删除状态会影响当前用户的消息可见性（会话隔离被破坏）
- ❌ 刷新页面（F5）后 `pendingDeletes` 丢失，消息重新出现
- ✅ 如果 Undo 倒计时结束，`removeSingle()` 会尝试发送删除请求（但可能因会话已切换而 401 失败）

**风险概率说明**：
- 消息 ID 是数据库自增 ID，正常情况下不同会话中 ID 不会立即重复
- 但如果服务器重启、数据库重置或 ID 回绕，ID 可能重复
- 更常见的场景：同一用户快速登出再登录，前一删除操作的 ID 恰好是新会话中某条消息的 ID

**排障判断点**：
1. 观察 MobX DevTools 中 `pendingDeletes` 的内容：登出后是否非空
2. 复现步骤：删除消息 → 5 秒内登出 → 立即登录 → 检查对应 ID 的消息是否可见
3. 查看 Console：是否有 `removeSingle()` 失败的错误（跨会话删除请求）
4. 对比刷新页面前后的消息列表差异
5. 在 Console 执行 `stores.messagesStore.pendingDeletes.size` 确认残留数量

**临时解决方案**：登出后刷新页面（F5），清除所有内存状态

#### 场景 C：与登出后残留重连定时器的组合影响

当 `pendingDeletes` 跨会话残留**和**登出后重连定时器残留**同时存在时，会出现以下复合现象：

```
用户 A 操作：
  1. 删除消息（ID=123）→ pendingDeletes 新增 ID=123
  2. 断开网络 → tryAuthenticate 失败 → connectionError()
     → 设置 15 秒后重连的定时器
  3. 在 5 秒内登出 → loggedIn = false
     → clearAll() → this.state = {}，但 pendingDeletes 保留
     → ⚠️ reconnectTimeoutId 未清理，定时器仍挂起

用户 B 登录（第 10 秒时）：
  4. 登录成功 → loggedIn = true → loadAll()
     → 加载消息列表，但 pendingDeletes 仍有 ID=123
     → ID=123 的消息被过滤，用户 B 看不到

第 15 秒时（组合效应触发）：
  5. 旧定时器触发 → tryReconnect(true) → tryAuthenticate()
     → 发送 GET /current/user（携带用户 B 的凭证）
     → 认证成功（用户 B 已登录）
     → connectionErrorMessage = null（已是 null，无变化）
     → ⚠️ 由于 connectionErrorMessage 没有"从非 null 变为 null"，
        Reaction 不会触发 clearAll/loadAll
     → pendingDeletes 继续保留，ID=123 的消息继续被隐藏

用户可见的复合现象：
  - 用户 B 的消息列表中 ID=123 的消息消失
  - 第 15 秒左右 Network 面板出现一次多余的 /current/user 请求
  - 刷新页面后，ID=123 的消息重新出现，多余请求停止
```

**排障判断点（复合场景）**：
1. 消息列表中某些消息意外消失 + Network 面板有多余的认证请求
2. 登出前同时存在：未完成的删除操作 + 连接错误状态
3. 刷新页面后两个问题同时消失

### 5.5 重新加载操作 (`loadAll()` - `reactions.ts:22-35`)

```javascript
const loadAll = () => {
    // 1. 重建 WebSocket 连接
    stores.wsStore.listen((message) => {
        stores.messagesStore.publishSingleMessage(message);
        Notifications.notifyNewMessage(message);
        // ... 高优先级消息播放声音
    });

    // 2. 重新加载应用列表
    stores.appStore.refresh();
    //    - 调用 requestItems() → GET /application
    //    - 成功后更新 this.items
};
```

### 5.6 组件强制刷新机制

通过 `refreshKey` 变化触发整个应用的重新挂载：

**Layout.tsx:106**:
```javascript
<div key={refreshKey}>
    {/* 所有子组件都会因为 key 变化而重新挂载 */}
    {/* 包括：Header、Navigation、所有页面 Routes */}
</div>
```

**效果**：
- 所有页面组件的 `useEffect` 重新执行
- MessagesPage 重新调用 `messagesStore.loadMore()` 加载消息
- ClientsPage 重新调用 `clientStore.refresh()` 加载客户端
- UsersPage 重新调用 `userStore.refresh()` 加载用户
- PluginsPage 重新调用 `pluginStore.refresh()` 加载插件

## 6. 登出时的状态清理

**Reaction 定义**: `reactions.ts:37-46`

```javascript
reaction(
    () => stores.currentUser.loggedIn,
    (loggedIn) => {
        if (loggedIn) {
            loadAll();    // 登录成功 - 加载数据
        } else {
            clearAll();   // 登出 - 清理所有状态（同连接恢复）
        }
    }
);
```

**触发场景**：
- 用户主动点击登出按钮
- `tryAuthenticate()` 遇到 4xx 错误 → 调用 `logout()`

## 7. 排障检查清单

### 7.1 连接错误 Banner 不显示？
- ✅ 检查是不是 `tryAuthenticate()` 触发的错误
- ✅ 普通 API 500 只会弹 Snackbar，不会显示 Banner
- ✅ 检查 `connectionErrorMessage` 的当前值

### 7.2 Retry 成功但数据没刷新？
- ✅ 检查 `connectionErrorMessage` 是否从 **非 null** 变为 **null**
- ✅ 初始 null → 成功 null 不会触发 reaction
- ✅ 检查 reaction 是否正常注册（`registerReactions()` 被调用）

### 7.3 网络恢复后出现多余的认证请求？
- ✅ 这是**预期行为**：`tryAuthenticate()` 成功时未清理重连定时器
- ✅ 之前挂起的定时器会在原定时间再次触发 `tryReconnect(true)`
- ✅ 不会导致数据重复清理，但会产生一次多余的 API 请求
- ✅ 可优化点：在 `tryAuthenticate()` 成功分支中添加 `clearTimeout` 逻辑

### 7.4 登出后登录页仍显示连接错误 Banner？
- ✅ 检查登出前是否有连接错误状态（连接错误 Banner 曾显示
- ✅ 观察 Network 面板：登出后是否仍有 `/current/user` 请求发出
- ✅ 验证：登出后网络断开/服务器 5xx → 旧定时器触发 → `connectionErrorMessage` 被设置为非空
- ✅ 临时解决：登出后刷新页面（F5）
- ✅ 修复方案：在 `logout()` 中添加 `clearTimeout(this.reconnectTimeoutId)`

### 7.5 删除消息后立即断网恢复，消息"消失"？
- ✅ 这是**Bug**：`MessagesStore.clearAll()` 未清理 `pendingDeletes`
- ✅ 复现路径：删除消息 → 5秒 Undo 窗口内网断网恢复 → 消息被过滤
- ✅ 验证方法：F5 刷新页面，消息会重新出现（服务器上还存在）
- ✅ 临时解决：让用户等待 5 秒后自动执行删除，或点击 Undo 再重新删除
- ✅ 修复方案：在 `clearAll()` 中添加 `this.pendingDeletes.clear()`

### 7.6 新登录用户看不到某些消息？
- ✅ 检查前一用户登出前是否有未完成的删除操作（5 秒 Undo 窗口内）
- ✅ 复现路径：用户 A 删除消息 → 5 秒内登出 → 用户 B 立即登录 → 相同 ID 消息被过滤
- ✅ 验证方法：F5 刷新页面，消息重新出现
- ✅ 临时解决：登出后刷新页面
- ✅ 修复方案：在 `clearAll()` 中添加 `this.pendingDeletes.clear()`

### 7.7 数据清理不完整？
- ✅ 检查 clearAll() 调用了哪些 Store 的 clear()
- ✅ PluginStore 可能遗漏清理（refreshKey++ 后页面会重新加载）
- ✅ ElevateStore 状态不会被清理
- ✅ MessagesStore.pendingDeletes 不会被清理（见 7.5、7.6）
- ✅ CurrentUser.reconnectTimeoutId 不会被清理（见 7.3、7.4）

### 7.8 重连间隔异常？
- ✅ 初始间隔 7500ms，每次失败翻倍
- ✅ 最大间隔 120000ms（2分钟）
- ✅ 成功后重置为 7500ms

## 8. 关键设计要点总结

### 8.1 连接错误单一入口
- 只有 `tryAuthenticate()` 能触发连接错误状态
- 避免多个 Store 独立管理连接状态的复杂性
- 普通业务错误不影响全局连接状态

### 8.2 错误分层处理
| 错误类型 | 显示方式 | 处理逻辑 |
|---------|---------|---------|
| 认证网络错误/5xx | 顶部红色 Banner | 自动重连（指数退避） |
| 普通 API 错误 | 底部 Snackbar | 仅提示，不重连 |
| 操作成功反馈 | 底部 Snackbar | 仅提示 |

### 8.3 恢复的原子性
- 连接恢复后强制清理所有 Store 状态
- 通过 `refreshKey` 强制组件重新挂载
- 确保 UI 显示的是最新的服务器数据，无陈旧缓存

### 8.4 Reaction 的精确性
- 只在 `connectionErrorMessage` 从 **错误状态 → 正常状态** 时触发
- 避免无意义的重复清理和加载
- 保证状态转换的可预测性

## 9. 完整数据流图

```
┌─────────────────────────────────────────────────────────────────┐
│                      普通 API 请求失败                             │
│  (AppStore/ClientStore/MessagesStore/PluginStore/UserStore)      │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Axios 全局拦截器                               │
│  ├─ 无 response → Snackbar：服务器不可达                          │
│  ├─ 401 → 调用 tryAuthenticate() → 进入下方认证流程               │
│  └─ 400/403/500 → Snackbar：具体错误信息                          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   tryAuthenticate 认证请求                        │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                      认证成功 ✅                                   │
│  ├─ connectionErrorMessage = null                                │
│  ├─ reconnectTime = 7500ms                                       │
│  ├─ loggedIn = true                                              │
│  └─ 若是从错误状态恢复 → Reaction 触发 clearAll + loadAll          │
└─────────────────────────────────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                      认证失败 ❌                                   │
│  ├─ 网络错误 / 5xx                                               │
│  │   ├─ connectionErrorMessage = "错误信息" → Banner 显示        │
│  │   ├─ 启动指数退避定时器（7.5s → 15s → ... → 120s）            │
│  │   └─ 自动调用 tryReconnect(quiet=true)                        │
│  │
│  └─ 4xx 客户端错误                                               │
│      ├─ connectionErrorMessage = null                            │
│      ├─ 调用 logout() → loggedIn = false                         │
│      └─ Reaction 触发 clearAll（如果之前是错误状态）               │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   用户点击 Retry 按钮                              │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│  tryReconnect(quiet=false) → 调用 tryAuthenticate()              │
│  ├─ 成功 → connectionErrorMessage=null → Reaction 触发清理+加载   │
│  └─ 失败 → connectionErrorMessage 不变 → Snackbar 重连失败提示     │
└─────────────────────────────────────────────────────────────────┘
```
