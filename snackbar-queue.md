# 前端通知与提示条（Snackbar）队列管理全链路分析

## 一、三类通知通道的关系与定位

系统中并存三种互不相同的通知/提示机制，各自职责清晰、相互独立又在特定场景下联动：

| 通道 | 技术栈 | 触发者 | 显示位置 | 典型用途 | 自动消失 |
|------|--------|--------|----------|----------|----------|
| **页面 Snackbar** | notistack + MUI | 前端用户操作 / HTTP 错误 / WS 状态 | 页面右下角（默认） | 操作结果反馈、错误提示、撤销确认 | ✅ 默认自动消失 |
| **桌面浏览器通知** | notifyjs + Web Notification API | WebSocket 新消息到来 | 系统通知栏 | 后台/跨页面提醒新消息到达 | ✅ 5 秒自动关闭 |
| **消息卡片（列表）** | MobX + React 虚拟列表 | WebSocket 新消息 / HTTP 拉取 | 页面主内容区 | 持久化展示所有历史消息 | ❌ 手动删除 |

三者的联动关系：
```
新消息到达 ── WebSocket ──► ① 插入消息列表（持久）
                      ├──► ② 桌面通知（5秒消失）
                      └──► ③ 高优先级播放提示音

用户操作 ──► ① API 请求 ──► ② 操作完成后 Snackbar 提示
          └──► 删除操作 ──► Snackbar Undo 条（5秒窗口）
```

---

## 二、事件来源全景：所有触发入口梳理

### 入口总览

所有页面内 Snackbar 最终都通过两条路径入队：
- **SnackManager.snack()** 统一入口（99% 的场景）
- **直接调用 enqueueSnackbar()** 特殊场景（删除 Undo 条）

### 2.1 全局接口错误处理（apiAuth 拦截器）

**文件**：[apiAuth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/apiAuth.ts)

这是最底层、覆盖最广的 Snackbar 来源——所有 axios 请求的错误拦截。

初始化入口：[index.tsx#L56](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/index.tsx#L56)

```typescript
axios.interceptors.response.use(undefined, (error) => {
    if (!error.response) {
        snack('Gotify server is not reachable, try refreshing the page.');
    }
    const status = error.response.status;
    if (status === 401) {
        currentUser.tryAuthenticate().then(() => snack('Could not complete request.'));
    }
    if (status === 400 || status === 403 || status === 500) {
        snack(error.response.data.error + ': ' + error.response.data.errorDescription);
    }
});
```

触发条件与提示内容：

| HTTP 状态 | 场景 | Snackbar 内容 |
|-----------|------|---------------|
| 无响应（网络错误） | 服务器不可达 / 断网 | `Gotify server is not reachable, try refreshing the page.` |
| 401 Unauthorized | 鉴权失效 | 先尝试重鉴权，失败后 `Could not complete request.` |
| 400 Bad Request | 请求参数错误 | `{error}: {errorDescription}` |
| 403 Forbidden | 权限不足 | `{error}: {errorDescription}` |
| 500 Internal Server Error | 服务端异常 | `{error}: {errorDescription}` |

> ⚠️ 注意：401 场景下重鉴权成功后仍会提示 "Could not complete request."，即原请求不会自动重试，只做状态恢复。

### 2.2 登录、注册与重连流程

**文件**：[CurrentUser.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/CurrentUser.ts)

所有与身份认证相关的操作都集中在 `CurrentUser` 类。

#### (1) 登录 login()
位置：[CurrentUser.ts#L51-L81](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/CurrentUser.ts#L51-L81)

```
成功：snack(`A client named '${name}' was created for your session.`)
失败：snack('Login failed')
```

#### (2) 注册 register()
位置：[CurrentUser.ts#L24-L44](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/CurrentUser.ts#L24-L44)

```
成功：snack('User Created. Logging in...') → 然后自动调用 login
失败：snack('Register failed: {error}: {errorDescription}')
无网络：snack('No network connection or server unavailable.')
```

#### (3) 鉴权与重连 tryAuthenticate / tryReconnect
位置：[CurrentUser.ts#L83-L155](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/CurrentUser.ts#L83-L155)

- `tryAuthenticate()` 失败时：若状态码 ≥ 500，调用 `connectionError()` 进入**指数退避重连**（7.5s → 15s → 30s → ... 最长 120s），此时**不弹 Snackbar**，只显示顶部横幅 `ConnectionErrorBanner`
- `tryReconnect(quiet=false)` 手动触发重连失败：`snack('Reconnect failed')`

#### (4) 修改密码
位置：[CurrentUser.ts#L131-L135](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/CurrentUser.ts#L131-L135)

```
成功：snack('Password changed')
```

### 2.3 资源增删改操作

所有 Store 都继承自 [BaseStore.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/common/BaseStore.ts)，并在构造时注入 `snack: SnackReporter`。

#### 应用管理（AppStore）
**文件**：[AppStore.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/application/AppStore.ts)

| 操作 | 提示内容 | 行号 |
|------|----------|------|
| 创建应用 | `Application created` | [L98](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/application/AppStore.ts#L98) |
| 更新应用 | `Application updated` | [L83](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/application/AppStore.ts#L83) |
| 删除应用 | `Application deleted` | [L25](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/application/AppStore.ts#L25) |
| 上传图标 | `Application image updated` | [L36](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/application/AppStore.ts#L36) |
| 删除图标 | `Application image deleted` | [L43](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/application/AppStore.ts#L43) |

#### 客户端管理（ClientStore）
**文件**：[ClientStore.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/client/ClientStore.ts)

| 操作 | 提示内容 | 行号 |
|------|----------|------|
| 创建客户端 | `Client added` | [L52](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/client/ClientStore.ts#L52) |
| 更新客户端 | `Client updated` | [L33](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/client/ClientStore.ts#L33) |
| 删除客户端 | `Client deleted` | [L19](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/client/ClientStore.ts#L19) |
| 提升权限 | `Client elevated` | [L62](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/client/ClientStore.ts#L62) |
| 取消提升 | `Canceled client elevation` | [L60](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/client/ClientStore.ts#L60) |

> 特殊方法 `createNoNotifcation()`（[L37-L47](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/client/ClientStore.ts#L37-L47)）创建客户端但**不弹 Snackbar**，供登录时自动创建客户端等内部场景使用。

#### 用户管理（UserStore）
**文件**：[UserStore.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/user/UserStore.ts)

| 操作 | 提示内容 | 行号 |
|------|----------|------|
| 创建用户 | `User created` | [L26](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/user/UserStore.ts#L26) |
| 更新用户 | `User updated` | [L33](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/user/UserStore.ts#L33) |
| 删除用户 | `User deleted` | [L19](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/user/UserStore.ts#L19) |

#### 插件管理（PluginStore）
**文件**：[PluginStore.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/plugin/PluginStore.ts)

| 操作 | 提示内容 | 行号 |
|------|----------|------|
| 删除插件 | `Cannot delete plugin`（禁止删除） | [L25](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/plugin/PluginStore.ts#L25) |
| 更新配置 | `Plugin config updated` | [L39](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/plugin/PluginStore.ts#L39) |
| 启用插件 | `Plugin enabled` | [L46](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/plugin/PluginStore.ts#L46) |
| 禁用插件 | `Plugin disabled` | [L46](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/plugin/PluginStore.ts#L46) |

#### 消息管理（MessagesStore）
**文件**：[MessagesStore.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/MessagesStore.ts)

| 操作 | 提示内容 | 行号 |
|------|----------|------|
| 删除全部消息 | `Deleted all messages` / `Deleted all messages from {app}` | [L87/L91](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/MessagesStore.ts#L87) |
| 发送消息 | `Message sent to {app.name}` | [L152](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/MessagesStore.ts#L152) |

### 2.4 WebSocket 连接状态

**文件**：[WebSocketStore.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/WebSocketStore.ts)

| 事件 | 提示内容 | 行号 |
|------|----------|------|
| 连接断开（非登出） | `WebSocket connection closed, trying again in 30 seconds.` | [L40](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/WebSocketStore.ts#L40) |
| 鉴权失败（401） | `Could not authenticate with client token, logging out.` | [L45](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/WebSocketStore.ts#L45) |

> 注意：WebSocket 断开后 30 秒自动重连，由 WS 自身的重连逻辑处理，与 `CurrentUser` 的指数退避重连是**两套独立机制**。

### 2.5 权限提升（ElevateStore）

**文件**：[ElevateStore.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/ElevateStore.ts)

| 场景 | 提示内容 | 行号 |
|------|----------|------|
| OIDC 弹窗被拦截 | `Popup was blocked. Please allow popups for this site and try again.` | [L60](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/ElevateStore.ts#L60) |
| OIDC 提升未完成 | `OIDC elevation was not completed.` | [L85](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/ElevateStore.ts#L85) |

### 2.6 复制密钥到剪贴板

**文件**：[CopyableSecret.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/common/CopyableSecret.tsx)

这是**唯一直接从 React 组件调用 `snackManager.snack()`** 的场景（而非通过 Store）。

位置：[CopyableSecret.tsx#L19-L27](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/common/CopyableSecret.tsx#L19-L27)

```typescript
const copyToClipboard = async () => {
    try {
        await navigator.clipboard.writeText(value);
        snackManager.snack('Copied to clipboard');
    } catch (error) {
        snackManager.snack('Failed to copy to clipboard');
    }
};
```

该组件用于应用 Token、客户端 Token 等密钥的显示与复制。

### 2.7 删除单条消息的 Undo 条（特殊路径）

**文件**：[Messages.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/Messages.tsx)

这是**唯一不经过 SnackManager、直接调用 `enqueueSnackbar()`** 的路径，因为需要捕获 snackbar 的 `key` 并绑定到 pendingDeletes 上。

位置：[Messages.tsx#L34-L52](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/Messages.tsx#L34-L52)

详细的撤销时序见第四章。

---

## 三、Snackbar 队列配置与显示优先级

### 3.1 队列基础设施

#### SnackbarProvider 挂载位置
**文件**：[Layout.tsx#L169](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/layout/Layout.tsx#L169)

```tsx
<SnackbarProvider />
```

使用 notistack v3 的默认配置，未自定义 props。

#### 核心封装 SnackManager
**文件**：[SnackManager.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/snack/SnackManager.ts)

```typescript
export interface SnackReporter {
    (message: string): void;
}

export class SnackManager {
    public snack: SnackReporter = (message: string): void => {
        enqueueSnackbar({message, variant: 'info'});
    };
}
```

所有 Store 都依赖 `SnackReporter` 接口，而非直接依赖 notistack，实现了业务层与 UI 组件库的解耦。

#### 初始化与注入链
[index.tsx#L28-L51](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/index.tsx#L28-L51)：

```
SnackManager 单例
  ├─ appStore
  ├─ userStore
  ├─ messagesStore
  ├─ currentUser
  ├─ elevateStore
  ├─ clientStore
  ├─ wsStore
  └─ pluginStore
```

### 3.2 notistack 默认队列行为

由于 `<SnackbarProvider />` 没有传自定义配置，使用 notistack v3 默认值：

| 配置项 | 默认值 | 含义 |
|--------|--------|------|
| `maxSnack` | 3 | 最多同时显示 3 个，超出的排队等待 |
| `autoHideDuration` | 5000ms | 普通 Snackbar 自动消失时间 |
| `anchorOrigin` | bottom-center | 显示在屏幕底部居中 |
| `variant` | 'default' | 本项目统一设为 'info' |

队列调度策略（notistack 内部实现）：
1. 新 Snackbar 入队时，若当前显示数 < maxSnack，则立即显示
2. 若已达上限，则排队，等前面的消失后按入队顺序依次显示
3. Snackbar 消失时触发 `onExited` 回调，自动从队列中取出下一个

### 3.3 业务优先级（priority 字段）

**重要结论**：消息的 `priority` 字段**不影响 Snackbar 的显示顺序**，Snackbar 严格按入队时间 FIFO。

priority 影响的是另外三个独立通道：

#### (1) 消息卡片左边框颜色
位置：[Message.tsx#L107-L115](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/Message.tsx#L107-L115)

```
priority 范围   左边框颜色
─────────────────────────────────
0 - 3           transparent（无色）
4 - 7           rgba(230,126,34,0.7)  橙色（警告）
8+              #e74c3c               红色（紧急）
```

#### (2) 音频提示（notification.ogg）
位置：[reactions.ts#L22-L32](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/reactions.ts#L22-L32)

触发条件（全部满足）：
- `message.priority >= 4`
- 距离上次播放超过 `AUDIO_REPEAT_DELAY = 1000ms`（防止消息连刷时音频重叠）

#### (3) 桌面通知（notifyjs）
位置：[reactions.ts#L25](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/reactions.ts#L25) + [browserNotification.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/snack/browserNotification.ts)

**所有优先级的新消息都会触发桌面通知**，无过滤。

> 用户可在左侧导航栏点击 "Enable Notifications" 按钮手动授权：[Navigation.tsx#L99-L107](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/layout/Navigation.tsx#L99-L107)

---

## 四、自动消失逻辑与撤销时序

### 4.1 三类自动消失计时器对比

| 通道 | 计时方式 | 时长 | 触发消失后动作 | 定义位置 |
|------|----------|------|----------------|----------|
| 普通 Snackbar | notistack 内置 `autoHideDuration` | 5000ms（默认） | 无回调，仅消失 | notistack 默认 |
| Undo Snackbar | 显式设置 `autoHideDuration` | 5000ms (UndoAutoHideMs) | `onExited → removeSingle()` 真删 | [Messages.tsx#L17](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/Messages.tsx#L17) |
| 桌面浏览器通知 | `setTimeout` 手动计时 | 5000ms | `target.close()` | [browserNotification.ts#L39-L44](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/snack/browserNotification.ts#L39-L44) |
| pendingDeletes 窗口 | 与 Undo Snackbar 同步 | 5000ms | 超时执行真删除 | 依赖 onExited 隐式驱动 |

三处 5 秒计时**相互独立**，来源不同，互不影响。

### 4.2 Undo 删除撤销的完整时序

这是系统中最复杂的通知交互流程，涉及 Snackbar 生命周期、MobX 状态、HTTP 请求、页面关闭兜底的多方协同。

#### 核心数据结构
[MessagesStore.ts#L19-L22](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/MessagesStore.ts#L19-L22)

```typescript
interface PendingDelete {
    key: SnackbarKey;     // notistack 返回的 snackbar 句柄
    message: IMessage;    // 待删消息
}

pendingDeletes: Map<number, PendingDelete>  // key = message.id
```

#### 完整时序图

```
T+0ms  ─ 用户点击删除按钮
         │
         ├─ ① deleteMessage(message)  [Messages.tsx]
         │    ├─ enqueueSnackbar({
         │    │     message: 'Message deleted',
         │    │     action: <Undo Button>,
         │    │     autoHideDuration: 5000,
         │    │     onExited: () => removeSingle(message),
         │    │     disableWindowBlurListener: true,     ← 窗口失焦不暂停计时
         │    │     transitionDuration: {enter:0, exit:0} ← 无动画，立即显示/消失
         │    │  }) → 返回 key
         │    └─ messagesStore.addPendingDelete({message, key})
         │
         ├─ ② 消息从列表中"隐藏"（视觉上立即消失）
         │    messagesStore.visible(id) → !pendingDeletes.has(id)
         │
         └─ ③ Snackbar 显示 "Message deleted [Undo]"

T+0ms ~ T+5000ms  ─ 5 秒撤销窗口
         │
         ├─ 场景 A：用户点 Undo 按钮
         │    ├─ messagesStore.cancelPendingDelete(message)
         │    │    ├─ pendingDeletes.delete(message.id)
         │    │    └─ closeSnackbar(pending.key)  ← 手动关闭 Snackbar
         │    └─ 消息重新出现在列表中
         │
         └─ 场景 B：用户无操作
              └─ 等待 autoHideDuration 倒计时结束

T+5000ms  ─ Snackbar 开始退出动画（动画时长 0ms）
         │
         └─ onExited 回调触发
              └─ messagesStore.removeSingle(message)
                   ├─ 检查 pendingDeletes.has(message.id)  ← 安全闸门
                   ├─ DELETE /message/{id}                    ← keepalive: true
                   ├─ 从 state[AllMessages].messages 移除
                   ├─ 从 state[appid].messages 移除
                   └─ cancelPendingDelete(message)            ← 清 Map
```

关键代码位置：
- 入队 & 加入待删：[Messages.tsx#L34-L52](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/Messages.tsx#L34-L52)
- 撤销：[MessagesStore.ts#L103-L110](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/MessagesStore.ts#L103-L110)
- 实际删除：[MessagesStore.ts#L119-L135](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/MessagesStore.ts#L119-L135)
- 可见性判断：[MessagesStore.ts#L116](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/MessagesStore.ts#L116)

### 4.3 页面关闭时的兜底机制

位置：[reactions.ts#L8-L9](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/reactions.ts#L8-L9)

```typescript
window.addEventListener('pagehide',     stores.messagesStore.executePendingDeletes);
window.addEventListener('beforeunload', stores.messagesStore.executePendingDeletes);
```

`executePendingDeletes()` 会遍历 `pendingDeletes` 中所有待删消息，立即调用 `removeSingle()` 执行真删。

配合 DELETE 请求的 `keepalive: true` 选项（[MessagesStore.ts#L126](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/MessagesStore.ts#L126)），即使页面关闭过程中请求也能可靠送达。

### 4.4 桌面通知的自动消失与交互

位置：[browserNotification.ts#L29-L44](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/snack/browserNotification.ts#L29-L44)

```
notifyShow 事件触发（通知显示时）
  └─ setTimeout(closeAfterTimeout, 5000) → 5 秒后调用 notification.close()

notifyClick 事件触发（用户点击通知时）
  ├─ window.parent.focus() + window.focus()  ← 聚焦主窗口
  ├─ window.location.href = '/'              ← 跳转首页
  └─ target.close()                          ← 立即关闭通知（不等5秒）
```

---

## 五、WebSocket 新消息的完整处理链路

这是连接"WebSocket → 消息列表 → 桌面通知 → 音频 → Snackbar"的中枢。

**入口文件**：[reactions.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/reactions.ts)

```
用户登录状态变为 true
  └─ reaction 触发 loadAll()
       ├─ wsStore.listen(callback)    ← 建立 WebSocket 连接
       └─ appStore.refresh()

WebSocket 收到消息 (ws.onmessage)
  └─ callback(JSON.parse(data.data))  ← 注意：没有做消息类型校验
       │
       ├─ ① messagesStore.publishSingleMessage(message)
       │    ├─ 插入 AllMessages 列表头部
       │    └─ 插入对应 appid 列表头部
       │
       ├─ ② Notifications.notifyNewMessage(message)  ← 桌面通知
       │    └─ 5 秒后自动关闭
       │
       └─ ③ 若 priority >= 4 且间隔 > 1000ms → 播放 notification.ogg
```

关键代码：[reactions.ts#L22-L33](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/reactions.ts#L22-L33)

> ⚠️ 注意：WebSocket 新消息**不会触发页面内 Snackbar**，只会更新消息列表 + 桌面通知 + 音频。页面内 Snackbar 仅用于用户操作反馈和错误提示。

---

## 六、数据流全景图

```
┌──────────────────────────────────────────────────────────────────────┐
│                         后端 (Go Server)                              │
│  /message  REST API    /stream WebSocket    /application /client ... │
└──────────┬───────────────────────┬──────────────────────┬───────────┘
           │                       │                      │
           │ HTTP 请求              │ WS 推送              │ HTTP 请求
           ▼                       ▼                      ▼
┌──────────────────────────┐  ┌────────────────────┐  ┌────────────────┐
│  axios (apiAuth 拦截器)  │  │  WebSocketStore    │  │  axios (主动拉取)│
│  全局错误 → snack()      │  │  /stream           │  │  列表加载等     │
└──────────┬───────────────┘  └─────────┬──────────┘  └────────┬───────┘
           │                             │                        │
           │                             │ reactions.ts           │
           │                             ▼                        │
           │               ┌──────────────────────────┐           │
           │               │  wsStore.listen(cb)      │           │
           │               │  ① publishSingleMessage  │           │
           │               │  ② notifyNewMessage      │           │
           │               │  ③ priority≥4 → 播放音频 │           │
           │               └────────────┬─────────────┘           │
           │                            │                         │
           │                            ▼                         │
           │                    ┌──────────────────┐              │
           │                    │  消息卡片列表     │              │
           │                    │  (持久化展示)     │              │
           │                    └──────────────────┘              │
           │                                                        │
┌──────────▼────────────────────────────────────────────────────────▼──┐
│                                                                      │
│                    所有 Store 中的业务操作                             │
│  MessagesStore / AppStore / ClientStore / UserStore / PluginStore   │
│  ElevateStore / CurrentUser / CopyableSecret                         │
│                                                                      │
│  操作成功 / 操作失败 → 调用 this.snack('...')                         │
│  删除消息 → 直接 enqueueSnackbar (Undo 条)                            │
│                                                                      │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
                                ▼
                    ┌─────────────────────────┐
                    │  SnackManager.snack()   │  ← 统一入口
                    │  (SnackReporter 接口)   │
                    └─────────────┬───────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │  notistack enqueueSnackbar │
                    └─────────────┬───────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │  SnackbarProvider 队列  │
                    │  maxSnack = 3 (默认)    │
                    │  autoHide = 5s (默认)   │
                    │  FIFO 调度              │
                    └─────────────┬───────────┘
                                  │
                    ┌─────────────┴───────────┐
                    ▼                         ▼
           普通 Snackbar               Undo Snackbar
           (默认 5s 消失)              (5s + onExited → 真删)
           无回调                      支持 closeSnackbar(key) 提前关闭
```

---

## 七、关键设计要点总结

### 1. 三层通知的职责分离
- **消息列表**：持久化、可追溯、完整历史 → 对应"数据层"
- **桌面通知**：跨页面/后台提醒新消息到达 → 对应"系统通知层"
- **页面 Snackbar**：即时操作反馈、错误提示、交互确认 → 对应"UI 反馈层"

三层互不干扰，Websocket 新消息只触达前两层，不会弹页面 Snackbar。

### 2. SnackReporter 是解耦边界
所有业务 Store 只依赖 `(message: string) => void` 函数签名，不直接 import notistack。这使得：
- 更换提示组件库时只需改 SnackManager
- 单元测试可注入 mock snack 函数
- 所有提示文案风格统一

### 3. 删除操作的"先软后硬"两阶段模式
`pendingDeletes` Map 是核心设计：
- 立即从视觉上移除（用户体验顺畅）
- 5 秒窗口内可撤销（容错）
- 页面关闭兜底执行（数据一致性）
- `keepalive: true` 保证请求送达（可靠性）

### 4. 两套独立的重连机制
- `CurrentUser.connectionError`：指数退避（7.5s → 120s），显示顶部横幅，不弹 Snackbar
- `WebSocketStore.onclose`：固定 30 秒后重连，弹 Snackbar 提示

两者分别处理"HTTP 鉴权"和"WebSocket 连接"两层连接的断开恢复。

### 5. priority 的语义边界
业务 priority **不参与 Snackbar 队列排序**，它只影响：
- 视觉上的消息卡片边框颜色
- 听觉上的提示音播放与否

Snackbar 队列严格按时间顺序 FIFO，由 notistack 内部控制。
