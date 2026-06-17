# 前端通知与提示条（Snackbar）队列管理分析

## 一、整体架构概览

通知系统由两条并行的显示通道构成：

| 通道 | 技术栈 | 用途 |
|------|--------|------|
| 页内 Snackbar | `notistack` 库 + `SnackbarProvider` | 用户操作反馈、操作结果提示、删除撤销确认 |
| 桌面浏览器通知 | `notifyjs` 库 + Web Notification API | 新消息到达的跨页面/后台通知 |

两者通过 `SnackManager`、`MessagesStore`、`reactions` 串联，由 MobX 的响应式系统驱动。

---

## 二、事件来源（三条链路）

### 链路 1：WebSocket 实时消息推送（新消息到达）

**入口**：[WebSocketStore.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/WebSocketStore.ts)

```
WebSocket (/stream)
  └─ ws.onmessage → callback(JSON.parse(data.data))
       └─ reactions.ts: stores.wsStore.listen(callback)
            ├─ ① messagesStore.publishSingleMessage(message)   → 插入消息列表
            ├─ ② Notifications.notifyNewMessage(message)       → 桌面通知
            └─ ③ 若 priority ≥ 4 且间隔 > 1s → 播放音频提示
```

关键代码位置：
- WebSocket 连接建立：[WebSocketStore.ts#L16-L51](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/WebSocketStore.ts#L16-L51)
- 收到消息后的分流处理：[reactions.ts#L23-L33](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/reactions.ts#L23-L33)

### 链路 2：用户操作的结果提示

统一通过 `SnackReporter` 接口（本质是 `SnackManager.snack`）触发，所有 Store 在构造时注入该回调。

典型触发点：

| 操作 | 调用位置 | Snack 内容 |
|------|----------|------------|
| 删除全部消息 | [MessagesStore.ts#L87](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/MessagesStore.ts#L87) | `Deleted all messages` / `Deleted all messages from {app}` |
| 发送消息成功 | [MessagesStore.ts#L152](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/MessagesStore.ts#L152) | `Message sent to {app.name}` |
| WebSocket 断开 | [WebSocketStore.ts#L40](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/WebSocketStore.ts#L40) | `WebSocket connection closed, trying again in 30 seconds.` |
| 鉴权失败 | [WebSocketStore.ts#L45](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/WebSocketStore.ts#L45) | `Could not authenticate with client token, logging out.` |

核心封装：[SnackManager.ts#L7-L10](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/snack/SnackManager.ts#L7-L10)

```typescript
public snack: SnackReporter = (message: string): void => {
    enqueueSnackbar({message, variant: 'info'});
};
```

### 链路 3：删除单条消息的 Undo 提示条

这是**唯一带交互**的 Snackbar，直接在 React 组件中调用 `enqueueSnackbar`。

位置：[Messages.tsx#L34-L52](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/Messages.tsx#L34-L52)

```
用户点击删除按钮
  └─ deleteMessage(message)
       ├─ enqueueSnackbar({
       │     message: 'Message deleted',
       │     action: Undo 按钮,
       │     autoHideDuration: 5000,
       │     onExited: removeSingle(message)  ← 实际执行删除
       │  }) → 返回 key
       └─ messagesStore.addPendingDelete({message, key})
```

---

## 三、显示优先级机制

### 3.1 Snackbar 队列本身的显示顺序

**由 `notistack` 库内部管理**，应用层不做自定义排序。`SnackbarProvider` 挂载在：
[Layout.tsx#L169](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/layout/Layout.tsx#L169)

notistack 默认行为：
- 新 Snackbar 入队时显示在最上方（或按 anchorOrigin 堆叠）
- 达到 `maxStack` 限制时，较早的会被临时隐藏，等前面的消失后再显示
- 应用层通过 `enqueueSnackbar` 的 `variant` 区分样式：info / success / warning / error（本项目只用 info）

### 3.2 消息业务优先级（priority 字段）

定义在 [types.ts#L38-L47](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/types.ts#L38-L47) 的 `IMessage.priority: number`。

**该优先级不影响 Snackbar 的显示顺序**，而是影响三个独立的视觉/听觉通道：

#### (1) 消息卡片左边框颜色
位置：[Message.tsx#L107-L115](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/Message.tsx#L107-L115)

```
priority 范围   左边框颜色
─────────────────────────────────
0 - 3           transparent（无色）
4 - 7           rgba(230,126,34,0.7)  橙色
8+              #e74c3c               红色
```

应用位置：[Message.tsx#L171-L173](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/Message.tsx#L171-L173)

#### (2) 音频提示（notification.ogg）
位置：[reactions.ts#L26-L32](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/reactions.ts#L26-L32)

触发条件：
- `message.priority >= 4`
- 且距离上次播放超过 `AUDIO_REPEAT_DELAY = 1000ms`（防止连刷）

```typescript
if (message.priority >= 4 && Date.now() > lastAudio + AUDIO_REPEAT_DELAY) {
    audio ??= new Audio('static/notification.ogg');
    audio.play();
}
```

#### (3) 浏览器桌面通知（notifyjs）
位置：[browserNotification.ts#L18-L27](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/snack/browserNotification.ts#L18-L27)

**所有优先级的新消息都会触发**，没有按 priority 过滤。

---

## 四、自动消失逻辑与衔接

### 4.1 页面内 Snackbar 的自动消失

#### 普通提示条（链路 1、链路 2）
由 `SnackManager` 统一 enqueue，`autoHideDuration` 使用 notistack 默认值（通常 5000ms，由 SnackbarProvider 的默认配置决定）。

#### Undo 删除确认条（链路 3）
明确指定 `autoHideDuration: UndoAutoHideMs = 5000`
位置：[Messages.tsx#L17](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/Messages.tsx#L17)、[Messages.tsx#L48](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/Messages.tsx#L48)

**关键衔接——onExited 钩子**：
```
onExited 触发（Snackbar 退出动画结束）
  └─ messagesStore.removeSingle(message)   // 真正调 API 删除
       └─ 还会自动 cancelPendingDelete
```

位置：[Messages.tsx#L49](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/message/Messages.tsx#L49)

### 4.2 浏览器桌面通知的自动消失
位置：[browserNotification.ts#L39-L44](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/snack/browserNotification.ts#L39-L44)

```typescript
function closeAfterTimeout(event: Event) {
    setTimeout(() => {
        const target = event.target as Notification;
        target.close();
    }, 5000);  // 固定 5 秒
}
```

**与 onClick 的配合**：用户点击通知时
- 聚焦主窗口 → 跳转 `/`
- 立即关闭通知（不等 5 秒）

位置：[browserNotification.ts#L29-L37](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/snack/browserNotification.ts#L29-L37)

### 4.3 Undo 撤销流程的完整时序

```
T+0s   用户点删除按钮
       ├─ enqueueSnackbar('Message deleted', 5s)
       └─ pendingDeletes.set(message.id, {key, message})  ← 消息在列表中隐藏
       
T+0s ~ T+5s
       用户可点 Undo：
         ├─ pendingDeletes.delete(message.id)
         ├─ closeSnackbar(key)          ← 提前关闭 Snackbar
         └─ 消息重新可见（visible() 返回 true）

T+5s   Snackbar 超时 onExited
       └─ removeSingle(message)
            ├─ DELETE /message/{id}     ← keepalive 请求，页面关闭也能发
            ├─ 从 state[AllMessages].messages 中移除
            ├─ 从 state[appid].messages 中移除
            └─ cancelPendingDelete (清 Map)
```

**页面关闭时的兜底**：
位置：[reactions.ts#L8-L9](file:///d:/fz/0601-2/solo-dogfeeding/code/24-server/ui/src/reactions.ts#L8-L9)

```typescript
window.addEventListener('pagehide',     stores.messagesStore.executePendingDeletes);
window.addEventListener('beforeunload', stores.messagesStore.executePendingDeletes);
```
即用户关页时如果还有待删除的消息，**全部立即执行**（不等 5 秒），配合 DELETE 请求的 `keepalive: true` 选项确保送达。

---

## 五、数据流全景图

```
                          ┌──────────────────────┐
                          │   后端 HTTP API      │
                          │  POST /message       │
                          │  DELETE /message/:id │
                          └─────────┬────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
          ┌─────────▼─────┐  ┌─────▼──────┐  ┌────▼────────┐
          │  WebSocket    │  │  axios     │  │  axios      │
          │  /stream      │  │  (主动拉取) │  │  (用户操作) │
          └───────┬───────┘  └─────┬──────┘  └────┬────────┘
                  │                │               │
  reactions.ts    ▼                │               │
  ┌───────────────┴─────────────┐  │               │
  │ wsStore.listen(callback)    │  │               │
  │                             │  │               │
  │ ① publishSingleMessage ─────┼──┼──► MessagesStore.state[].messages
  │                             │  │               │
  │ ② notifyNewMessage ─────────┼──┼──► notifyjs 桌面通知 (5s自关)
  │                             │  │               │
  │ ③ priority≥4 播放音频 ◄─────┘  │               │
  └───────────────────────────────┘               │
                                                  │
                页面用户操作                        │
                (删除单条/删除全部/发送)             │
                        │                           │
                        ▼                           │
          MessagesStore.sendMessage / removeByApp / removeSingle
                        │                           │
                        └───────── SnackReporter ───┘
                                    │
                                    ▼
                          SnackManager.snack()
                                    │
                                    ▼
                      notistack enqueueSnackbar()
                                    │
                                    ▼
                        SnackbarProvider 管理队列
                         (默认 autoHideDuration)
                                    │
                                    ▼
                           Snackbar 渲染显示
                                    │
                  ┌─────────────────┴──────────────────┐
                  ▼                                     ▼
       普通提示（默认超时消失）              Undo 提示（5秒 onExited→真删除）
                                                │
                                        用户点 Undo? ──是──► closeSnackbar + 恢复
                                                │否
                                                ▼
                                        removeSingle API
```

---

## 六、关键设计要点总结

1. **Snackbar 的显示排序完全交给 notistack**：应用层只负责 `enqueueSnackbar`，不关心队列调度。业务 priority 影响的是卡片边框色 + 音频，不是 Snackbar 本身的先后。

2. **删除操作采用"先标记后确认"模式**：`pendingDeletes` Map 是核心数据结构，承担"5秒窗口内可撤销 + 页面关闭兜底执行"双职责。

3. **两处 5 秒定时互相独立**：
   - Snackbar 的 `autoHideDuration: 5000` 控制 Undo 条可见时间
   - 浏览器通知的 `setTimeout(5000)` 控制桌面通知停留
   两者来源不同、互不影响。

4. **SnackReporter 是业务层到 UI 层的解耦边界**：所有 Store 只依赖这个 `(string) => void` 函数签名，不直接 import notistack，便于替换和测试。
