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

## 九、总结

Gotify 管理界面的通知系统采用「分层降级」设计：

1. **最高优先级**：消息列表持久化存储，确保不丢失
2. **次高优先级**：高优先级消息音效提醒，确保及时感知
3. **便利性功能**：浏览器原生通知和 Snackbar 提供即时视觉反馈，但不作为可靠通知保证

这种设计在保障核心功能可靠性的同时，充分利用浏览器能力提供良好的用户体验，异常场景下有清晰的降级路径。
