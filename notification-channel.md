# 管理界面通知通道协作机制分析报告

## 一、整体架构概览

Gotify 管理界面的通知系统采用「WebSocket 统一推送 + 多通道分发」的架构设计。服务端通过 WebSocket 实时推送消息，前端接收后并行分发到三类展示通道：

```
服务端消息创建
       ↓
WebSocket 推送 (/stream)
       ↓
前端统一接收 (reactions.ts:23-33)
       ├───────────────┬───────────────┐
       ↓               ↓               ↓
消息列表存储    浏览器原生通知    高优先级音效
(MessagesStore) (browserNotification)  (notification.ogg)
```

## 二、服务端推送到前端的分流机制

### 2.1 服务端推送流程

1. **消息创建入口**：`api/message.go:363-385` 的 `CreateMessage` 方法
   - 应用通过 Token 认证后提交消息
   - 消息先持久化到数据库
   - 然后调用 `Notifier.Notify()` 推送给在线客户端

2. **WebSocket 分发**：`api/stream/stream.go:82-91` 的 `Notify` 方法
   - 维护 `map[uint][]*client` 用户连接映射表
   - 消息通过 `write` channel 发送给该用户的所有在线连接
   - 支持多端同时在线（浏览器、移动端等）

### 2.2 前端接收与分流

前端在 `ui/src/reactions.ts:22-33` 中注册 WebSocket 回调：

```typescript
stores.wsStore.listen((message) => {
    stores.messagesStore.publishSingleMessage(message);  // 1. 存入消息列表
    Notifications.notifyNewMessage(message);              // 2. 浏览器原生通知
    if (message.priority >= 4) {                          // 3. 高优先级音效
        audio.play();
    }
});
```

**关键分流点**：三条通道并行执行，互不阻塞。

## 三、两类提示通道详解

### 3.1 浏览器原生通知通道

**文件**：`ui/src/snack/browserNotification.ts`

#### 核心特性：
- **技术实现**：基于 `notifyjs` 库封装，使用浏览器 `Notification` API
- **展示形式**：系统级通知弹窗（出现在操作系统通知区域）
- **生命周期**：5 秒后自动关闭 (`closeAfterTimeout`)
- **交互行为**：点击通知聚焦窗口并跳转到首页 (`closeAndFocus`)
- **静默模式**：`silent: true`，不播放系统提示音（音效由前端独立控制）

#### 权限管理：
- 权限检测：`mayAllowPermission()` - 检查浏览器支持且未被拒绝
- 权限请求：用户在侧边栏点击 "Enable Notifications" 按钮触发 (`Navigation.tsx:100-106`)
- 权限状态持久化：浏览器记住用户选择

### 3.2 界面下方提示队列 (Snackbar)

**文件**：`ui/src/snack/SnackManager.ts`

#### 核心特性：
- **技术实现**：基于 `notistack` 库，使用 Material-UI 的 Snackbar 组件
- **展示位置**：界面底部居中（由 `SnackbarProvider` 在 `Layout.tsx:169` 注入）
- **使用场景**：**不用于新消息通知**，仅用于操作反馈：
  - 删除消息/应用成功提示
  - WebSocket 连接断开提示
  - 认证失败提示
  - 消息发送成功提示

#### 队列管理：
- 由 `notistack` 内部管理队列，支持多条 Snackbar 堆叠
- 删除操作支持 "Undo" 操作（5 秒超时，见 `Messages.tsx:34-52`）

## 四、优先级规则

### 4.1 优先级定义

**来源**：`model/message.go` 和 `model/application.go`

- 每条消息携带 `priority` 字段（整数）
- 未指定时使用应用的 `DefaultPriority`（默认为 0）

### 4.2 前端分级处理

| 优先级范围 | 触发行为 | 视觉表现 |
|-----------|---------|---------|
| priority >= 4 | 播放通知音效 (notification.ogg) | 消息左侧橙色边框 (`rgba(230, 126, 34, 0.7)`) |
| priority > 7 | 同上 + 更高视觉权重 | 消息左侧红色边框 (`#e74c3c`) |
| priority < 4 | 仅消息列表和浏览器通知 | 无色边框 |

**音效防抖动**：`AUDIO_REPEAT_DELAY = 1000ms`，1 秒内最多播放一次音效。

## 五、去重规则

### 5.1 服务端去重
- **无显式去重机制**
- 每条消息拥有唯一自增 ID，由数据库保证唯一性
- 重复推送依赖 WebSocket 连接可靠性

### 5.2 前端去重
- **无显式去重逻辑**
- `MessagesStore.publishSingleMessage()` 直接 `unshift` 到列表头部
- 依赖服务端保证不重复推送

> **设计考量**：Gotify 定位为通知中转服务，消息丢失比重复推送更严重，因此采用「至少一次」语义。

## 六、与消息列表和未读计数的关系

### 6.1 消息列表同步

**文件**：`ui/src/message/MessagesStore.ts:74-81`

```typescript
publishSingleMessage(message: IMessage) {
    if (this.exists(AllMessages)) {
        this.stateOf(AllMessages).messages.unshift(message);  // 全部消息列表
    }
    if (this.exists(message.appid)) {
        this.stateOf(message.appid).messages.unshift(message); // 应用专属列表
    }
}
```

- 新消息实时插入到列表顶部
- 同时更新「全部消息」和「对应应用」两个视图
- 不需要用户手动刷新

### 6.2 未读计数

**当前设计**：**无未读计数机制**

- 所有消息默认为「已读」状态展示
- 没有「标记为已读」功能
- 没有未读消息红点/角标提示

> **设计考量**：Gotify 更偏向消息历史记录而非即时通讯，未读状态非核心需求。

## 七、异常场景兜底行为

### 7.1 通知权限受限

| 权限状态 | 浏览器原生通知 | 消息列表 | Snackbar 提示 | 音效 |
|---------|--------------|---------|--------------|------|
| granted（已授权） | ✓ 正常显示 | ✓ 正常更新 | ✓ 正常显示 | ✓ 正常播放 |
| prompt（未决定） | ✗ 不显示（需用户授权） | ✓ 正常更新 | ✗ 无提示（仅侧边栏显示按钮） | ✓ 正常播放 |
| denied（已拒绝） | ✗ 不显示 | ✓ 正常更新 | ✗ 无提示 | ✓ 正常播放 |

**兜底策略**：通知权限缺失仅影响原生通知通道，消息列表和音效不受影响，确保用户不会错过重要信息。

### 7.2 网络异常处理

**文件**：`ui/src/message/WebSocketStore.ts:25-48`

1. **连接错误** (`onerror`)：
   - 标记 `wsActive = false`
   - 控制台打印错误，**不显示用户可见提示**

2. **连接关闭** (`onclose`)：
   - 尝试验证用户身份（防止 Token 过期）
   - 通过 Snackbar 提示：「WebSocket connection closed, trying again in 30 seconds.」
   - 30 秒后自动重试连接

3. **重连保护**：
   - 重试前检查用户登录状态
   - 401 未授权时提示登出

### 7.3 页面卸载兜底

**文件**：`ui/src/reactions.ts:8-9`

```typescript
window.addEventListener('pagehide', stores.messagesStore.executePendingDeletes);
window.addEventListener('beforeunload', stores.messagesStore.executePendingDeletes);
```

- 页面关闭前执行待删除的消息（使用 `keepalive: true` 保证请求发出）
- WebSocket 连接主动关闭

## 八、关键代码路径汇总

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| WebSocket 接收回调 | `ui/src/reactions.ts` | 22-33 |
| 浏览器通知实现 | `ui/src/snack/browserNotification.ts` | 18-27 |
| Snackbar 管理 | `ui/src/snack/SnackManager.ts` | 7-11 |
| 消息存储与发布 | `ui/src/message/MessagesStore.ts` | 74-81 |
| WebSocket 重连逻辑 | `ui/src/message/WebSocketStore.ts` | 32-48 |
| 服务端消息推送 | `api/stream/stream.go` | 82-91 |
| 消息创建与通知 | `api/message.go` | 363-385 |
| 优先级视觉映射 | `ui/src/message/Message.tsx` | 107-115 |

## 十、边界场景深度分析

### 10.1 多标签页并发场景

#### 服务端分发机制
**证据链**：`api/stream/stream.go:82-91`

```go
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    a.lock.RLock()
    defer a.lock.RUnlock()
    if clients, ok := a.clients[userID]; ok {
        for _, c := range clients {
            c.write <- msg
        }
    }
}
```

- 服务端维护 `map[uint][]*client`，同一用户的所有连接并列存储
- 每个标签页建立独立 WebSocket 连接（`stream.go:154` 中 `register(client)`）
- 消息遍历所有连接发送，**无标签页去重逻辑**

#### 前端多实例隔离
**证据链**：`ui/src/index.tsx:28-51`

```typescript
const initStores = (): StoreMapping => {
    const snackManager = new SnackManager();
    const wsStore = new WebSocketStore(snackManager.snack, currentUser);
    // ... 每个标签页创建独立的 Store 实例
};
```

- 每个标签页初始化独立的 Store 实例树
- 无 `BroadcastChannel`、`localStorage` 跨标签页同步机制
- 各标签页独立调用 `wsStore.listen()` 注册回调

#### 两类通道行为对比

| 通道类型 | 重复触发情况 | 互斥/优先级关系 | 证据链 |
|---------|-------------|----------------|--------|
| **浏览器原生通知** | ✗ **会重复触发**。N 个标签页收到同一条消息，会显示 N 条系统通知 | 无统一优先级/互斥。各标签页独立显示，由操作系统通知中心管理 | `reactions.ts:25` → `browserNotification.ts:18-27` → 浏览器 Notification API |
| **Snackbar 队列** | ✗ **会重复触发**（仅操作反馈场景）。每个标签页独立显示操作结果 | 无统一优先级/互斥。各标签页 Snackbar 栈独立 | `SnackManager.ts:7-11` → `notistack` 内部队列 |
| **消息列表** | ✓ 各标签页独立维护，数据库层面保证消息唯一 | 无跨标签页同步 | `MessagesStore.ts:74-81` → MobX observable 状态 |

> **关键发现**：浏览器原生通知和 Snackbar 队列在多标签页场景下完全隔离，无任何跨页协调机制。同一用户打开 N 个标签页，将收到 N 条重复的浏览器通知。

---

### 10.2 通知权限在 granted/prompt/denied 间切换

#### 权限检测的静态性
**证据链**：`ui/src/layout/Navigation.tsx:46-47`

```typescript
const [showRequestNotification, setShowRequestNotification] =
    React.useState(mayAllowPermission);
```

- `mayAllowPermission()` **仅在组件初始化时调用一次**
- 未监听 `Notification.permission` 的 `change` 事件
- 权限状态切换后，UI 不会自动响应

#### 三种切换路径的行为分析

##### 路径 1：prompt → granted（用户主动授权）
**证据链**：`Navigation.tsx:100-106` → `browserNotification.ts:9-16`

```typescript
<Button
    onClick={() => {
        requestPermission();
        setShowRequestNotification(false);
    }}>
    Enable Notifications
</Button>
```

1. 用户点击侧边栏 "Enable Notifications" 按钮
2. 调用 `Notify.requestPermission()` 触发浏览器权限弹窗
3. 用户授权后，`Notification.permission` 变为 `granted`
4. 按钮消失（`setShowRequestNotification(false)`）
5. **后续消息正常触发浏览器通知**
6. Snackbar 队列不受影响，继续正常工作

##### 路径 2：granted → denied（用户在浏览器设置中撤销权限）
**证据链**：`browserNotification.ts:18-27`

```typescript
export function notifyNewMessage(msg: IMessage) {
    const notify = new Notify(msg.title, { ... });
    notify.show();  // notifyjs 内部检查权限
}
```

1. 用户在浏览器设置中拒绝通知权限
2. 前端无感知，继续调用 `notifyNewMessage()`
3. `notifyjs` 内部检测到权限为 `denied`，静默失败
4. **浏览器通知不显示，但无错误抛出**
5. 消息列表、音效、Snackbar 均不受影响

##### 路径 3：denied → granted（用户在浏览器设置中重新授权）
**证据链**：`Navigation.tsx:47`（仅初始化时调用一次）

1. 用户在浏览器设置中重新授予权限
2. 由于 `showRequestNotification` 已设为 `false`，侧边栏不会重新显示授权按钮
3. 但实际上 `notifyNewMessage()` 已可正常工作
4. **用户需刷新页面才能重新检测到权限状态**

#### 两类通道在权限切换时的关系

| 权限切换方向 | 浏览器原生通知 | Snackbar 队列 | 统一优先级/互斥 |
|-------------|--------------|--------------|----------------|
| prompt → granted | 从无到有，后续消息正常显示 | 始终正常 | 无。Snackbar 不需要权限，两者独立 |
| granted → denied | 后续消息静默不显示 | 始终正常 | 无。权限仅影响浏览器通知通道 |
| denied → granted | 需刷新页面后恢复 | 始终正常 | 无。状态同步需要页面刷新 |

> **关键发现**：浏览器通知与 Snackbar 队列之间无任何权限关联或互斥逻辑。Snackbar 队列完全不依赖浏览器通知权限，任何权限状态下均可正常工作。

---

### 10.3 重连期间重复消息

#### WebSocket 重连机制
**证据链**：`ui/src/message/WebSocketStore.ts:32-48`

```typescript
ws.onclose = () => {
    this.wsActive = false;
    this.currentUser.tryAuthenticate()
        .then(() => {
            this.snack('WebSocket connection closed, trying again in 30 seconds.');
            setTimeout(() => this.listen(callback), 30000);
        });
};
```

- 连接断开后，30 秒后自动重连
- 重连时重新调用 `listen(callback)`，创建新的 WebSocket 连接
- **服务端无消息补发机制**：WebSocket 握手不携带 `since` 或 `lastMessageId` 参数

#### 消息丢失与重复的可能性分析

##### 场景 A：连接断开瞬间的消息丢失
**证据链**：`api/stream/client.go:95-99`

```go
c.conn.SetWriteDeadline(time.Now().Add(writeWait))
if err := writeJSON(c.conn, message); err != nil {
    printWebSocketError("WriteError", err)
    return  // 写入失败直接返回，消息丢弃
}
```

- 服务端写入 WebSocket 失败时直接丢弃消息
- 无重试队列、无持久化未确认消息
- 重连后**不会补发**丢失的消息
- 浏览器通知和 Snackbar 均不会收到这些消息

##### 场景 B：应用层重试导致的重复消息
**证据链**：`api/message.go:363-385`

```go
func (a *MessageAPI) CreateMessage(ctx *gin.Context) {
    // ... 每次调用 CreateMessage 生成新的消息 ID
    msgInternal := toInternalMessage(&message)
    a.DB.CreateMessage(msgInternal)  // 写入数据库，生成唯一 ID
    a.Notifier.Notify(auth.GetUserID(ctx), toExternalMessage(msgInternal))
}
```

- 如果发送方（应用）因网络超时重试，会创建**多条不同 ID 的消息**
- 每条消息都独立触发浏览器通知和消息列表更新
- 从用户视角看是重复通知，但系统认为是独立消息
- **无应用层去重机制**（如基于消息内容哈希去重）

##### 场景 C：前端重连后的消息去重检查
**证据链**：`ui/src/message/MessagesStore.ts:74-81`

```typescript
publishSingleMessage(message: IMessage) {
    if (this.exists(AllMessages)) {
        this.stateOf(AllMessages).messages.unshift(message);  // 直接插入，不检查 ID
    }
}
```

- 直接 `unshift` 到列表头部，**不检查消息 ID 是否已存在**
- 浏览器通知也直接调用，无去重逻辑
- 如果因网络异常导致服务端重复推送同 ID 消息（理论上不会发生），前端会重复显示

#### 两类通道在重连期间的行为

| 通道类型 | 重连期间重复触发可能性 | 互斥/优先级关系 | 证据链 |
|---------|----------------------|----------------|--------|
| **浏览器原生通知** | ✓ 可能重复。应用层重试会导致多条通知 | 无统一优先级。每条通知独立显示 | `reactions.ts:25` → `browserNotification.ts:18-27` |
| **Snackbar 队列** | ✗ 不会因消息重复触发。仅在连接断开时显示一次「重连中」提示 | 无。Snackbar 与消息推送是独立事件流 | `WebSocketStore.ts:32-48` → `SnackManager.ts:7-11` |
| **消息列表** | ✓ 可能重复。应用层重试会导致多条消息记录 | 无。按时间倒序排列 | `MessagesStore.ts:74-81` → 数据库独立 ID |

> **关键发现**：重连期间的重复风险主要来自应用层重试，而非 WebSocket 本身。浏览器通知和消息列表都会呈现为多条独立消息，Snackbar 队列则不受消息重复影响，仅在连接状态变化时触发。

## 十一、总结

Gotify 管理界面的通知系统采用「分层降级」设计：

1. **最高优先级**：消息列表持久化存储，确保不丢失
2. **次高优先级**：高优先级消息音效提醒，确保及时感知
3. **便利性功能**：浏览器原生通知和 Snackbar 提供即时视觉反馈，但不作为可靠通知保证

### 边界场景核心结论

| 边界场景 | 浏览器原生通知行为 | Snackbar 队列行为 | 统一协调机制 |
|---------|------------------|------------------|-------------|
| 多标签页并发 | 每个标签页独立触发，N 页显示 N 条 | 每个标签页独立显示操作反馈 | 无。完全隔离 |
| 权限状态切换 | 受权限影响，静默失败无提示 | 完全不受权限影响 | 无。两者独立 |
| 重连期间 | 应用层重试会导致重复通知 | 仅连接状态变化时触发 | 无。独立事件流 |

这种设计在保障核心功能可靠性的同时，充分利用浏览器能力提供良好的用户体验，异常场景下有清晰的降级路径，但在多标签页协调和权限动态响应方面存在优化空间。
