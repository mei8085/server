# 消息发送链路分析

## 概述

本文档详细分析了从管理界面点击"Push Message"按钮发送测试消息，到消息通过 WebSocket 实时推送到前端的完整技术链路。

---

## 第一部分：前端表单状态组装

### 1.1 触发入口 - Messages.tsx

在消息列表页面 `ui/src/message/Messages.tsx` 中，当用户在某个具体应用下点击"Push Message"按钮时：

```typescript
// Messages.tsx:120-129
{app && (
    <Button
        id="push-message"
        variant="contained"
        color="primary"
        onClick={() => setPushMessageOpen(true)}
        style={{marginRight: 5}}>
        Push Message
    </Button>
)}
```

点击后打开 `PushMessageDialog` 对话框，并传入应用信息：

```typescript
// Messages.tsx:160-168
{pushMessageOpen && app && (
    <PushMessageDialog
        appName={app.name}
        defaultPriority={app.defaultPriority}
        fClose={() => setPushMessageOpen(false)}
        fOnSubmit={(message, title, priority) =>
            messagesStore.sendMessage(app.id, message, title, priority)
        }
    />
)}
```

### 1.2 表单组件 - PushMessageDialog.tsx

`PushMessageDialog` 组件负责收集用户输入的消息数据：

```typescript
// PushMessageDialog.tsx:19-28
export const PushMessageDialog = ({appName, defaultPriority, fClose, fOnSubmit}: IProps) => {
    const [title, setTitle] = useState('');
    const [message, setMessage] = useState('');
    const [priority, setPriority] = useState(defaultPriority);

    const submitEnabled = message.trim().length !== 0;
    const submitAndClose = async () => {
        await fOnSubmit(message, title, priority);
        fClose();
    };
    // ...
};
```

**表单字段说明：**
- `title`: 消息标题（可选，为空时后端会用应用名填充）
- `message`: 消息内容（必填）
- `priority`: 消息优先级（默认使用应用的 defaultPriority）

### 1.3 数据提交 - MessagesStore.ts

表单提交后调用 `MessagesStore.sendMessage` 方法：

```typescript
// MessagesStore.ts:137-154
public sendMessage = async (
    appId: number,
    message: string,
    title: string,
    priority: number
): Promise<void> => {
    const app = this.appStore.getByID(appId);
    const payload: Pick<IMessage, 'title' | 'message' | 'priority'> = {
        message,
        priority,
        title,
    };

    await axios.post(`${config.get('url')}message`, payload, {
        headers: {'X-Gotify-Key': app.token},
    });
    this.snack(`Message sent to ${app.name}`);
};
```

**关键点：**
1. 从 `appStore` 获取应用的 `token`
2. 组装请求 payload，只包含 `message`、`title`、`priority`
3. 使用应用 token 作为 `X-Gotify-Key` 请求头发送 POST 请求到 `/message` 接口

---

## 第二部分：应用与权限关系

### 2.1 路由配置 - router.go

消息创建接口使用 `RequireApplicationToken` 中间件：

```go
// router.go:181
g.Group("/").Use(authentication.RequireApplicationToken).POST("/message", messageHandler.CreateMessage)
```

### 2.2 认证中间件 - authentication.go

`RequireApplicationToken` 中间件验证逻辑：

```go
// authentication.go:62-76
func (a *Auth) RequireApplicationToken(ctx *gin.Context) {
    if a.evaluate(ctx, a.handleApplication) {
        return
    }
    state, err := a.handleUser()(ctx)
    if err != nil {
        ctx.AbortWithError(500, err)
    }
    if state != authStateSkip {
        // Return to the user that it's valid authentication, but we don't allow user auth for application endpoints.
        a.abort403(ctx)
        return
    }
    a.abort401(ctx)
}
```

**处理流程：**
1. 首先尝试用 `handleApplication` 验证应用 token
2. 如果失败，尝试用用户凭证验证，但用户凭证即使有效也会返回 403
3. 都失败则返回 401

### 2.3 应用 Token 验证

```go
// authentication.go:178-203
func (a *Auth) handleApplication(ctx *gin.Context) (authState, error) {
    token, isCookie := a.readTokenFromRequest(ctx)
    if token == "" {
        return authStateSkip, nil
    }
    app, err := a.DB.GetApplicationByToken(token)
    if err != nil {
        return authStateSkip, err
    }
    if app == nil {
        return authStateSkip, nil
    }
    RegisterApplication(ctx, app)
    // 更新 LastUsed 时间...
    return authStateOk, nil
}
```

**Token 读取顺序（readTokenFromRequest）：**
1. Query 参数 `token`
2. Header `X-Gotify-Key`
3. Authorization Header `Bearer <token>`
4. Cookie `gotify-client-token`

### 2.4 权限模型总结

| 认证方式 | 能否调用 /message POST | 说明 |
|---------|-----------------------|------|
| Application Token | ✅ 可以 | 专门用于发送消息 |
| Client Token | ❌ 不可以 | 客户端用于接收消息 |
| User Basic Auth | ❌ 不可以 | 用户用于管理操作 |

---

## 第三部分：后端消息处理与写入

### 3.1 API 层处理 - api/message.go

```go
// message.go:363-385
func (a *MessageAPI) CreateMessage(ctx *gin.Context) {
    message := model.MessageExternal{}
    if err := ctx.Bind(&message); err == nil {
        application := auth.GetApplication(ctx)
        message.ApplicationID = application.ID
        if strings.TrimSpace(message.Title) == "" {
            message.Title = application.Name
        }

        if message.Priority == nil {
            message.Priority = &application.DefaultPriority
        }

        message.Date = timeNow()
        message.ID = 0
        msgInternal := toInternalMessage(&message)
        if success := successOrAbort(ctx, 500, a.DB.CreateMessage(msgInternal)); !success {
            return
        }
        a.Notifier.Notify(auth.GetUserID(ctx), toExternalMessage(msgInternal))
        ctx.JSON(200, toExternalMessage(msgInternal))
    }
}
```

**处理步骤：**
1. 绑定请求体到 `MessageExternal` 结构体
2. 从认证上下文中获取应用信息，设置 `ApplicationID`
3. **标题兜底：** 如果标题为空，使用应用名称
4. **优先级兜底：** 如果未设置优先级，使用应用默认优先级
5. 设置当前时间为消息日期
6. 转换为内部模型 `Message` 并持久化到数据库
7. 通过 `Notifier` 通知所有已连接的客户端（WebSocket 推送）
8. 返回创建成功的消息

### 3.2 模型转换

**外部模型（API 交互）：**
```go
// model/message.go:23-66
type MessageExternal struct {
    ID            uint                   `json:"id"`
    ApplicationID uint                   `json:"appid"`
    Message       string                 `json:"message" binding:"required"`
    Title         string                 `json:"title"`
    Priority      *int                   `json:"priority"`
    Extras        map[string]interface{} `json:"extras,omitempty"`
    Date          time.Time              `json:"date"`
}
```

**内部模型（数据库存储）：**
```go
// model/message.go:8-16
type Message struct {
    ID            uint      `gorm:"autoIncrement;primaryKey;index"`
    ApplicationID uint
    Message       string    `gorm:"type:text"`
    Title         string    `gorm:"type:text"`
    Priority      int
    Extras        []byte
    Date          time.Time
}
```

**转换函数：**
- `toInternalMessage`: 外部 → 内部，`Extras` 序列化为 JSON 字节
- `toExternalMessage`: 内部 → 外部，`Extras` 反序列化为 map

### 3.3 数据库持久化 - database/message.go

```go
// database/message.go:21-24
func (d *GormDatabase) CreateMessage(message *model.Message) error {
    return d.DB.Create(message).Error
}
```

使用 GORM ORM 直接插入数据库，主键 ID 由数据库自动生成。

---

## 第四部分：WebSocket 实时推送

### 4.1 Notifier 接口

在 `api/message.go` 中定义了 `Notifier` 接口：

```go
// message.go:32-34
type Notifier interface {
    Notify(userID uint, message *model.MessageExternal)
}
```

`MessageAPI` 持有 `Notifier` 实例，在 `router.go` 中注入的是 `stream.API`：

```go
// router.go:78
messageHandler := api.MessageAPI{Notifier: streamHandler, DB: db}
```

### 4.2 Stream API 通知机制 - api/stream/stream.go

```go
// stream.go:82-91
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

**通知机制：**
1. 根据 `userID` 查找所有已连接的客户端
2. 遍历每个客户端，将消息写入客户端的 `write` channel
3. 使用读写锁保证并发安全

### 4.3 WebSocket 客户端连接管理

客户端注册：
```go
// stream.go:106-110
func (a *API) register(client *client) {
    a.lock.Lock()
    defer a.lock.Unlock()
    a.clients[client.userID] = append(a.clients[client.userID], client)
}
```

客户端结构：
```go
// stream/client.go:23-30
type client struct {
    conn    *websocket.Conn
    onClose func(*client)
    write   chan *model.MessageExternal  // 消息写入通道（缓冲大小 1）
    userID  uint
    token   string
    once    once
}
```

### 4.4 消息写入 WebSocket

每个客户端有独立的写入 goroutine：

```go
// stream/client.go:81-107
func (c *client) startWriteHandler(pingPeriod time.Duration) {
    pingTicker := time.NewTicker(pingPeriod)
    defer func() {
        c.NotifyClose()
        pingTicker.Stop()
    }()

    for {
        select {
        case message, ok := <-c.write:
            if !ok {
                return
            }
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := writeJSON(c.conn, message); err != nil {
                printWebSocketError("WriteError", err)
                return
            }
        case <-pingTicker.C:
            // 发送 ping 保持连接...
        }
    }
}
```

### 4.5 WebSocket 连接建立

客户端通过 `/stream` 端点建立连接：

```go
// router.go:215
clientAuth.GET("/stream", streamHandler.Handle)
```

`Handle` 方法升级 HTTP 连接为 WebSocket：

```go
// stream.go:143-158
func (a *API) Handle(ctx *gin.Context) {
    conn, err := a.upgrader.Upgrade(ctx.Writer, ctx.Request, nil)
    if err != nil {
        ctx.Error(err)
        return
    }

    var token string
    if c := auth.GetClient(ctx); c != nil {
        token = c.Token
    }
    client := newClient(conn, auth.GetUserID(ctx), token, a.remove)
    a.register(client)
    go client.startReading(a.pongTimeout)
    go client.startWriteHandler(a.pingPeriod)
}
```

---

## 第五部分：前端 WebSocket 接收与状态更新

### 5.1 WebSocket Store - WebSocketStore.ts

```typescript
// WebSocketStore.ts:16-51
public listen = (callback: (msg: IMessage) => void) => {
    if (!this.currentUser.loggedIn || this.wsActive) {
        return;
    }
    this.wsActive = true;

    const wsUrl = config.get('url').replace('http', 'ws').replace('https', 'wss');
    const ws = new WebSocket(wsUrl + 'stream');

    ws.onmessage = (data) => callback(JSON.parse(data.data));

    ws.onclose = () => {
        // 自动重连逻辑，30秒后重试
    };

    this.ws = ws;
};
```

### 5.2 消息分发 - reactions.ts

在应用初始化时注册 WebSocket 监听回调：

```typescript
// reactions.ts:22-35
const loadAll = () => {
    stores.wsStore.listen((message) => {
        stores.messagesStore.publishSingleMessage(message);
        Notifications.notifyNewMessage(message);
        // 高优先级消息播放提示音
        if (message.priority >= 4 && Date.now() > lastAudio + AUDIO_REPEAT_DELAY) {
            audio ??= new Audio('static/notification.ogg');
            audio.currentTime = 0;
            audio.play();
        }
    });
    stores.appStore.refresh();
};
```

### 5.3 本地状态更新 - MessagesStore.ts

收到新消息后更新本地状态：

```typescript
// MessagesStore.ts:73-81
@action
public publishSingleMessage = (message: IMessage) => {
    if (this.exists(AllMessages)) {
        this.stateOf(AllMessages).messages.unshift(message);
    }
    if (this.exists(message.appid)) {
        this.stateOf(message.appid).messages.unshift(message);
    }
};
```

**更新逻辑：**
1. 同时更新"全部消息"列表（AllMessages = -1）
2. 更新对应应用的消息列表
3. 使用 `unshift` 将新消息插入列表头部
4. MobX 响应式更新会自动触发 UI 重渲染

---

## 完整链路时序图

```
[用户点击 Push Message]
        ↓
[PushMessageDialog 收集表单数据]
        ↓
[MessagesStore.sendMessage 组装 payload]
        ↓
[HTTP POST /message 携带 X-Gotify-Key 头]
        ↓
[后端 RequireApplicationToken 中间件验证]
        ↓
[MessageAPI.CreateMessage 处理]
    ├─ 绑定请求体
    ├─ 设置 ApplicationID
    ├─ 标题/优先级兜底
    ├─ 转换为内部模型
    ├─ 写入数据库
    └─ 调用 Notifier.Notify
        ↓
[Stream.API.Notify 查找用户的 WebSocket 客户端]
        ↓
[消息写入每个客户端的 write channel]
        ↓
[客户端 startWriteHandler 读取 channel 并发送 WebSocket]
        ↓
[前端 WebSocketStore 接收消息]
        ↓
[reactions 调用 publishSingleMessage]
        ↓
[MessagesStore 更新本地状态]
        ↓
[MobX 响应式更新 UI]
```

---

## 关键设计要点

1. **应用 Token 隔离**：每个应用有独立的 token，发送消息时必须使用对应应用的 token
2. **双重数据模型**：`MessageExternal` 用于 API 交互，`Message` 用于数据库存储，通过转换函数解耦
3. **Channel 解耦**：WebSocket 发送通过 channel 异步解耦，不阻塞 HTTP 请求
4. **按用户分组**：WebSocket 客户端按 userID 分组管理，通知时只需遍历目标用户的连接
5. **MobX 响应式**：前端使用 MobX 管理状态，消息到达后自动更新 UI
6. **优雅降级**：标题和优先级都有兜底逻辑，确保消息完整性

---

## 涉及文件清单

| 层级 | 文件路径 | 职责 |
|------|---------|------|
| 前端 UI | `ui/src/message/Messages.tsx` | 消息列表页面，触发推送对话框 |
| 前端 UI | `ui/src/message/PushMessageDialog.tsx` | 推送消息表单组件 |
| 前端 Store | `ui/src/message/MessagesStore.ts` | 消息状态管理，API 调用 |
| 前端 Store | `ui/src/message/WebSocketStore.ts` | WebSocket 连接管理 |
| 前端 核心 | `ui/src/reactions.ts` | 注册 WebSocket 监听回调 |
| 后端 API | `api/message.go` | 消息 API 处理，调用 Notifier |
| 后端 API | `api/stream/stream.go` | WebSocket 服务端，消息通知 |
| 后端 API | `api/stream/client.go` | WebSocket 客户端读写循环 |
| 后端 认证 | `auth/authentication.go` | Token 验证中间件 |
| 后端 数据库 | `database/message.go` | 消息持久化 |
| 后端 模型 | `model/message.go` | 消息数据结构定义 |
| 后端 路由 | `router/router.go` | 路由注册与中间件配置 |
