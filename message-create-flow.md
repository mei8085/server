# 消息发送链路分析

## 概述

本文档详细分析了从管理界面点击"Push Message"按钮发送测试消息，到消息通过 WebSocket 实时推送到前端的完整技术链路。包含主路径分析、安全机制、失败路径处理和背压语义说明。

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

### 2.5 发送链路与接收链路的双认证体系

**核心设计：发送和接收使用两套独立的认证体系，通过 `UserID` 汇合。**

#### 发送链路（/message POST）
```
POST /message → RequireApplicationToken 中间件
    ↓
使用 Application Token 认证
    ↓
认证成功后，Application.UserID 标识消息归属用户
    ↓
消息写入数据库时关联 ApplicationID（间接关联 UserID）
```

#### 接收链路（/stream WebSocket）
```
GET /stream → RequireClient 中间件
    ↓
使用 Client Token 认证
    ↓
认证成功后，Client.UserID 标识接收者
    ↓
WebSocket 客户端按 UserID 分组存储在 stream.API.clients 中
```

#### 汇合点：通过 UserID 关联推送目标

```go
// Application 模型（model/application.go:23）
UserID uint   `gorm:"index;..." json:"-"`  // 应用所属用户

// Client 模型
UserID uint   // 客户端所属用户

// 推送时（api/message.go:382）
a.Notifier.Notify(auth.GetUserID(ctx), toExternalMessage(msgInternal))
// 这里的 UserID 来自 Application.UserID

// 分发时（api/stream/stream.go:92-99）
if clients, ok := a.clients[userID]; ok {
    for _, c := range clients {
        c.write <- msg  // 推送给该 UserID 下所有 Client 连接
    }
}
```

**设计意图：**
- **隔离**：发送方（应用）和接收方（客户端）使用不同 token，防止权限混淆
- **关联**：通过 UserID 将发送和接收关联到同一用户，确保只有消息所属用户能收到
- **灵活**：一个用户可以有多个应用发送、多个客户端接收，全部通过 UserID 汇聚

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
2. 从认证上下文中获取应用信息，**强制覆盖** `ApplicationID`
3. **标题兜底：** 如果标题为空，使用应用名称
4. **优先级兜底：** 如果未设置优先级，使用应用默认优先级
5. 设置当前时间为消息日期
6. **强制重置** `ID = 0`，由数据库自增生成
7. 转换为内部模型 `Message` 并持久化到数据库
8. 通过 `Notifier` 通知所有已连接的客户端（WebSocket 推送）
9. 返回创建成功的消息

### 3.2 防伪造保护机制

CreateMessage 中有两处关键的防伪造保护：

```go
// 保护 1：强制覆盖 ApplicationID，忽略请求体中的 appid
message.ApplicationID = application.ID

// 保护 2：强制重置 ID，防止客户端指定
message.ID = 0
```

**安全含义详解：**

| 保护措施 | 防止的攻击 | 安全意义 |
|---------|-----------|---------|
| `ApplicationID = application.ID` | 越权发送：恶意用户在请求体中传入其他应用的 appid，试图伪造其他应用发送消息 | 确保消息的 ApplicationID 只能来自认证上下文，即只能用当前 token 对应的应用发送 |
| `ID = 0` | ID 注入：恶意用户指定一个已存在的 ID，可能导致覆盖或冲突；或指定一个未来的 ID 破坏自增序列 | 确保 ID 始终由数据库自增生成，维护数据一致性 |

> **重要**：虽然 `MessageExternal` 结构体定义了 `ID` 和 `ApplicationID` 字段（用于返回响应），但在创建消息时这两个字段会被强制覆盖，客户端传入的值完全被忽略。

### 3.3 模型转换

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

### 3.4 数据库持久化 - database/message.go

```go
// database/message.go:21-24
func (d *GormDatabase) CreateMessage(message *model.Message) error {
    return d.DB.Create(message).Error
}
```

使用 GORM ORM 直接插入数据库，主键 ID 由数据库自动生成。

---

## 第四部分：失败路径与状态语义

### 4.1 认证失败路径

#### 场景 1：Token 缺失
- **触发条件**：请求中没有携带任何 token（没有 X-Gotify-Key 头、没有 token query 参数等）
- **处理流程**：`handleApplication` 返回 `authStateSkip` → `handleUser` 也返回 `authStateSkip` → `abort401`
- **响应**：
  ```json
  {
    "error": "Unauthorized",
    "errorCode": 401,
    "errorDescription": "you need to provide a valid access token or user credentials to access this api"
  }
  ```

#### 场景 2：Token 非法/无效
- **触发条件**：携带了 token，但数据库中不存在对应的 Application
- **处理流程**：`GetApplicationByToken(token)` 返回 `nil` → `handleApplication` 返回 `authStateSkip` → 同场景 1
- **响应**：同场景 1，401 Unauthorized

#### 场景 3：使用用户凭证（Basic Auth）
- **触发条件**：请求携带了有效的 Basic Auth 用户凭证
- **处理流程**：
  ```go
  // RequireApplicationToken 中
  if a.evaluate(ctx, a.handleApplication) {
      return  // Application Token 验证失败，继续
  }
  state, err := a.handleUser()(ctx)
  if state != authStateSkip {
      // 用户凭证有效，但不允许用于应用接口
      a.abort403(ctx)  // 返回 403 Forbidden
      return
  }
  ```
- **响应**：
  ```json
  {
    "error": "Forbidden",
    "errorCode": 403,
    "errorDescription": "you are not allowed to access this api"
  }
  ```

### 4.2 Bind 失败路径

#### 场景 4：请求体绑定失败
- **触发条件**：JSON 格式错误、`message` 字段为空等
- **处理流程**：
  ```go
  // CreateMessage 中
  if err := ctx.Bind(&message); err == nil {
      // 正常处理
  }
  // Bind 失败时，函数直接返回，没有显式处理
  // 错误由全局错误中间件处理
  ```

- **错误处理中间件**（error/handler.go）：
  ```go
  func Handler() gin.HandlerFunc {
      return func(c *gin.Context) {
          c.Next()
          if len(c.Errors) > 0 {
              for _, e := range c.Errors {
                  switch e.Type {
                  case gin.ErrorTypeBind:
                      // 处理验证错误，如 "Field 'message' is required"
                      writeError(c, strings.Join(stringErrors, "; "))
                  }
              }
          }
      }
  }
  ```

- **响应（以 message 为空为例）**：
  ```json
  {
    "error": "Bad Request",
    "errorCode": 400,
    "errorDescription": "Field 'message' is required"
  }
  ```

### 4.3 数据库写入失败

#### 场景 5：数据库错误
- **触发条件**：数据库连接失败、约束冲突等
- **处理流程**：
  ```go
  if success := successOrAbort(ctx, 500, a.DB.CreateMessage(msgInternal)); !success {
      return
  }
  ```
- **响应**：
  ```json
  {
    "error": "Internal Server Error",
    "errorCode": 500,
    "errorDescription": "<具体数据库错误信息>"
  }
  ```

---

## 第五部分：WebSocket 实时推送

### 5.1 Notifier 接口

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

### 5.2 Stream API 通知机制 - api/stream/stream.go

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

### 5.3 WebSocket 客户端连接管理

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

### 5.4 背压语义详解

**核心设计：`write` channel 容量 = 1**

```go
// stream/client.go:35
write:   make(chan *model.MessageExternal, 1),
```

#### 背压机制分析

| 场景 | 行为 | 影响 |
|------|------|------|
| 正常流速 | 消息写入 channel，立即被读取发送 | 无阻塞，实时推送 |
| 客户端网络慢 | channel 满时，`c.write <- msg` 会阻塞 | 阻塞当前 `Notify` 调用，进而阻塞对应的 HTTP 请求处理 |
| 客户端已断开 | select 会检测到 write 错误，退出循环 | 连接关闭，从 clients map 中移除 |

**并发语义澄清：RLock 不阻塞其他 Notify**

```go
// stream.go:82-91
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    a.lock.RLock()       // 读锁，允许多个 goroutine 同时持有
    defer a.lock.RUnlock()
    if clients, ok := a.clients[userID]; ok {
        for _, c := range clients {
            c.write <- msg  // 真正可能阻塞的地方
        }
    }
}
```

**关键纠正**：`RLock` 是共享锁，**多个 Notify 可以并发持锁**，不会互相阻塞。真正的阻塞点是 `c.write <- msg` 这个 channel 写入操作。

**慢连接对系统的影响链：**
```
[新消息到达]
    ↓
Notify() 获取 RLock（可并发）
    ↓
遍历该 userID 下的所有客户端
    ↓
某客户端网络慢 → channel 已满 → c.write <- msg 阻塞
    ↓
当前 Notify() 被阻塞在 channel 写入上（仍持有 RLock）
    ↓
其他 Notify() 可以正常获取 RLock 并处理其他用户的消息
    ↓
只有被阻塞的 Notify() 对应的 HTTP 请求超时
    ↓
其他用户的推送不受影响
```

**设计权衡：**
- 容量 1 意味着**最多缓冲 1 条消息**，保证消息的实时性
- 牺牲了对慢客户端的容忍度，但避免了内存无限增长
- 慢客户端会被自动断开（写入超时后连接关闭）
- 影响范围是**单用户级**：某个用户的慢客户端只会阻塞该用户的 Notify，不影响其他用户

### 5.5 c.write 关闭与并发发送风险

#### 关闭路径分析

`c.write` channel 的关闭有两条路径：

```go
// 路径 1：主动关闭（client.go:43-48）
func (c *client) Close() {
    c.once.Do(func() {
        c.conn.Close()
        close(c.write)  // 关闭 channel
    })
}

// 路径 2：读写循环异常退出（client.go:50-57）
func (c *client) NotifyClose() {
    c.once.Do(func() {
        c.conn.Close()
        close(c.write)  // 关闭 channel
        c.onClose(c)    // 从 clients map 中移除
    })
}
```

`NotifyClose` 被以下场景调用：
- `startReading` defer：读取超时或客户端断开
- `startWriteHandler` defer：写入超时或错误

#### once 机制保护边界

项目使用了一个自定义的 `once` 实现（once.go）：

```go
func (o *once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 1 {
        return  // 快速路径：已执行，直接返回
    }
    if o.mayExecute() {
        f()  // 慢速路径：执行 f()
    }
}

func (o *once) mayExecute() bool {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        atomic.StoreUint32(&o.done, 1)  // 先标记 done=1
        return true                     // 再返回执行 f()
    }
    return false
}
```

**关键特性**：`done` 标记在 `f()` 执行**之前**就被设置为 1。

#### 并发发送与关闭的竞态窗口

考虑以下时序：

```
Goroutine A (Notify 发送路径):
  T1: a.lock.RLock()
  T2: if atomic.LoadUint32(&c.once.done) == 0  // 还没关闭
  T3: c.write <- msg                           // 准备写入

Goroutine B (关闭路径):
  T0: c.NotifyClose()
  T1: atomic.LoadUint32(&o.done) == 0
  T2: o.mayExecute() → 设置 done=1，返回 true
  T3: close(c.write)                            // 关闭 channel
```

**竞态窗口**：
- 如果 T3(A) 在 T3(B) **之前**完成：写入成功，无问题
- 如果 T3(B) 在 T3(A) **之前**完成：`c.write <- msg` 会触发 **panic: send on closed channel**

**once 无法覆盖这个窗口**：因为 `done` 标记只在 Do 入口检查，而 `c.write <- msg` 不在 Do 保护范围内。

#### 风险等级与可观测后果

| 风险场景 | 触发条件 | 后果 | 概率 |
|---------|---------|------|------|
| 写入已关闭的 channel | 关闭和发送在极短时间内并发 | panic: send on closed channel | 低（需精确时序命中）|
| 读取已关闭的 channel | startWriteHandler 中 `<-c.write` | 返回 ok=false，正常退出循环 | 预期行为 |
| 重复关闭 channel | 多次调用 Close/NotifyClose | once 保证只执行一次 close | 已保护 |

**可观测后果**：
- 服务端日志无特殊记录（panic 会被 Go runtime 捕获）
- 该次消息推送失败，但不会影响其他客户端
- 发送方的 HTTP 请求可能超时或收到 500 错误
- 客户端重连后可以通过拉取 API 获取丢失的消息

### 5.5 消息写入 WebSocket

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

**写入超时**：`writeWait = 2 * time.Second`，如果 2 秒内无法写入 WebSocket，连接会被关闭。

### 5.6 WebSocket 连接建立

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

## 第六部分：前端 WebSocket 接收与状态更新

### 6.1 WebSocket Store - WebSocketStore.ts

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

### 6.2 消息分发 - reactions.ts

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

### 6.3 本地状态更新 - MessagesStore.ts

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

### 6.4 前端一致性机制

**关键设计：sendMessage 不直接更新本地列表，依赖 WebSocket 回流。**

```typescript
// sendMessage 只发送请求，不修改本地状态
public sendMessage = async (appId: number, message: string, title: string, priority: number) => {
    // ... 发送 POST 请求
    await axios.post(`${config.get('url')}message`, payload, {
        headers: {'X-Gotify-Key': app.token},
    });
    // 注意：这里没有调用 publishSingleMessage！
    this.snack(`Message sent to ${app.name}`);
};

// 列表更新完全依赖 WebSocket 推送
// reactions.ts 中监听 WebSocket，收到消息后调用 publishSingleMessage
```

**一致性保证：**
1. **单一数据源**：数据库是唯一真相源，前端状态是数据库的投影
2. **时序一致性**：WebSocket 推送顺序与数据库写入顺序一致（因为 Notify 在 DB.CreateMessage 之后同步调用）
3. **失败透明**：如果发送成功但 WebSocket 未收到，用户刷新页面时会从 API 拉取到消息
4. **乐观提示**：通过 snackbar 提示"Message sent"，用户知道操作已提交

**潜在时序问题：**
- 如果发送请求返回 200，但 WebSocket 推送丢失，用户会看到提示但列表不更新
- 解决方式：用户可以手动刷新，或下一次消息推送时重新建立连接

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
    ├─ Token 缺失 → 401
    ├─ Token 无效 → 401
    └─ 用户凭证命中 → 403
        ↓
[MessageAPI.CreateMessage 处理]
    ├─ Bind 失败 → 400
    ├─ 强制覆盖 ApplicationID（防伪造）
    ├─ 强制重置 ID = 0（防伪造）
    ├─ 标题/优先级兜底
    ├─ 转换为内部模型
    ├─ 写入数据库 → 失败返回 500
    └─ 调用 Notifier.Notify
        ↓
[Stream.API.Notify 查找用户的 WebSocket 客户端]
        ↓
[消息写入每个客户端的 write channel]
    └─ 慢客户端阻塞 → 可能拖慢系统
        ↓
[客户端 startWriteHandler 读取 channel 并发送 WebSocket]
    └─ 写入超时 → 连接关闭
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

## 主路径 + 失败路径对照小结

### 主路径（成功场景）

| 步骤 | 组件 | 行为 | 成功标志 |
|------|------|------|---------|
| 1 | PushMessageDialog | 收集 title/message/priority | message 非空 |
| 2 | MessagesStore.sendMessage | 组装 payload，携带应用 token 发送 POST | HTTP 200 |
| 3 | RequireApplicationToken | 验证 Application Token | 认证通过 |
| 4 | CreateMessage | 绑定请求体，强制覆盖 ApplicationID 和 ID | Bind 成功 |
| 5 | CreateMessage | 写入数据库 | DB.CreateMessage 成功 |
| 6 | Stream.API.Notify | 写入各客户端 write channel | 无阻塞 |
| 7 | client.startWriteHandler | WebSocket 发送消息 | 写入成功 |
| 8 | WebSocketStore.onmessage | 解析消息 | JSON 解析成功 |
| 9 | publishSingleMessage | 更新本地状态 | MobX 触发 UI 更新 |

### 失败路径对照

| 失败场景 | 触发点 | HTTP 状态 | 错误信息示例 | 处理方 |
|---------|--------|-----------|-------------|--------|
| Token 缺失 | RequireApplicationToken | 401 | you need to provide a valid access token... | 认证中间件 |
| Token 无效 | RequireApplicationToken | 401 | you need to provide a valid access token... | 认证中间件 |
| 使用用户凭证 | RequireApplicationToken | 403 | you are not allowed to access this api | 认证中间件 |
| message 为空 | CreateMessage.Bind | 400 | Field 'message' is required | 全局错误中间件 |
| JSON 格式错误 | CreateMessage.Bind | 400 | invalid character... | 全局错误中间件 |
| 数据库错误 | DB.CreateMessage | 500 | <数据库错误详情> | successOrAbort |
| WebSocket 写入超时 | startWriteHandler | -（连接关闭）| WriteError: ... | 客户端 goroutine |

### 关键安全检查点

| 检查点 | 位置 | 保护目的 |
|--------|------|---------|
| ApplicationID 强制覆盖 | CreateMessage:367 | 防止越权发送到其他应用 |
| ID 强制重置为 0 | CreateMessage:377 | 防止 ID 注入和自增序列破坏 |
| 仅 Application Token 可发送 | RequireApplicationToken | 权限隔离，防止客户端/用户 token 滥用 |
| 推送按 UserID 分组 | Stream.API.Notify | 确保只有消息所属用户能收到 |

---

## 关键设计要点

1. **双认证体系**：发送用 Application Token，接收用 Client Token，通过 UserID 关联
2. **应用 Token 隔离**：每个应用有独立的 token，发送消息时必须使用对应应用的 token
3. **防伪造保护**：强制覆盖 ApplicationID、重置 ID，确保数据完整性
4. **双重数据模型**：`MessageExternal` 用于 API 交互，`Message` 用于数据库存储，通过转换函数解耦
5. **Channel 背压**：WebSocket 发送通过容量为 1 的 channel 异步解耦，慢客户端会被自动断开
6. **按用户分组**：WebSocket 客户端按 userID 分组管理，通知时只需遍历目标用户的连接
7. **前端最终一致性**：sendMessage 不直接更新列表，依赖 WebSocket 回流，保证数据来源单一
8. **MobX 响应式**：前端使用 MobX 管理状态，消息到达后自动更新 UI
9. **优雅降级**：标题和优先级都有兜底逻辑，确保消息完整性

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
| 后端 错误处理 | `error/handler.go` | 全局错误处理，Bind 错误格式化 |
| 后端 数据库 | `database/message.go` | 消息持久化 |
| 后端 模型 | `model/message.go` | 消息数据结构定义 |
| 后端 模型 | `model/application.go` | 应用数据结构，含 UserID |
| 后端 路由 | `router/router.go` | 路由注册与中间件配置 |
