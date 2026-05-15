# 插件消息通道完整时序分析

## 一、系统架构概览

插件消息通道涉及三个核心层次：

| 层次 | 组件 | 职责 |
|------|------|------|
| 插件层 | `plugin/compat/` | 插件接口适配、消息发送 |
| 管理层 | `plugin/manager.go` | 插件生命周期、消息路由 |
| 分发层 | `api/stream/` | WebSocket 连接管理、实时推送 |

---

## 二、三道关卡详解

### 关卡一：鉴权

#### 2.1 插件注册时的鉴权上下文建立

```
用户认证 → Token提取 → 用户信息关联 → 插件实例绑定
```

**代码路径**：`plugin/manager.go:281-292`

```go
func (m *Manager) initializeForUser(user model.User) error {
    userCtx := compat.UserContext{
        ID:    user.ID,
        Name:  user.Name,
        Admin: user.Admin,
    }
    
    for _, p := range m.plugins {
        if err := m.initializeSingleUserPlugin(userCtx, p); err != nil {
            return err
        }
    }
    ...
}
```

**鉴权上下文传递链**：

1. **用户登录** → `auth/authentication.go` 验证凭证
2. **Token 解析** → `auth.GetUserID(ctx)` 从 Gin Context 提取用户 ID
3. **插件实例化** → 将 `UserContext` 注入插件构造函数

#### 2.2 WebSocket 连接鉴权

**代码路径**：`api/stream/stream.go:143-158`

```go
func (a *API) Handle(ctx *gin.Context) {
    conn, err := a.upgrader.Upgrade(ctx.Writer, ctx.Request, nil)
    if err != nil { ... }
    
    var token string
    if c := auth.GetClient(ctx); c != nil {
        token = c.Token
    }
    client := newClient(conn, auth.GetUserID(ctx), token, a.remove)
    a.register(client)
    ...
}
```

**鉴权方式**：支持多种认证方式
- `X-Gotify-Key` Header
- `Authorization: Bearer <token>`
- URL Query `?token=`
- Cookie `gotify-client-token`

---

### 关卡二：注册

#### 2.1 插件消息处理器注册

**代码路径**：`plugin/manager.go:335-341`

```go
if compat.HasSupport(instance, compat.Messenger) {
    instance.SetMessageHandler(redirectToChannel{
        ApplicationID: pluginConf.ApplicationID,  // 自动创建的内部应用ID
        UserID:        pluginConf.UserID,         // 所属用户ID
        Messages:      m.messages,                // 全局消息通道
    })
}
```

**关键设计**：每个支持 Messenger 能力的插件实例，都会被注入一个 `redirectToChannel` 处理器，该处理器持有：
- **ApplicationID**：插件专属的内部应用标识
- **UserID**：插件所属用户
- **Messages Channel**：指向 Manager 的全局消息队列

#### 2.2 内部应用自动创建

**代码路径**：`plugin/manager.go:408-420`

```go
if compat.HasSupport(instance, compat.Messenger) {
    app := &model.Application{
        Token:       auth.GenerateNotExistingToken(...),
        Name:        info.String(),
        UserID:      userID,
        Internal:    true,           // 标记为内部应用
        Description: fmt.Sprintf("auto generated application for %s", info.ModulePath),
    }
    if err := m.db.CreateApplication(app); err != nil {
        return nil, err
    }
    pluginConf.ApplicationID = app.ID
}
```

**设计意图**：插件消息与普通应用消息统一处理，复用现有的消息存储和分发机制。

---

### 关卡三：运行时分发

#### 3.1 消息从插件到 Manager

**代码路径**：`plugin/messagehandler.go:23-34`

```go
func (c redirectToChannel) SendMessage(msg compat.Message) error {
    c.Messages <- MessageWithUserID{
        Message: model.MessageExternal{
            ApplicationID: c.ApplicationID,
            Message:       msg.Message,
            Title:         msg.Title,
            Priority:      &msg.Priority,
            Date:          time.Now(),
            Extras:        msg.Extras,
        },
        UserID: c.UserID,
    }
    return nil
}
```

**消息封装**：插件消息被包装为 `MessageWithUserID`，携带用户上下文信息。

#### 3.2 Manager 消息处理 Goroutine

**代码路径**：`plugin/manager.go:67-84`

```go
go func() {
    for {
        message := <-manager.messages
        internalMsg := &model.Message{
            ApplicationID: message.Message.ApplicationID,
            Title:         message.Message.Title,
            Priority:      *message.Message.Priority,
            Date:          message.Message.Date,
            Message:       message.Message.Message,
        }
        if message.Message.Extras != nil {
            internalMsg.Extras, _ = json.Marshal(message.Message.Extras)
        }
        db.CreateMessage(internalMsg)           // 持久化到数据库
        message.Message.ID = internalMsg.ID
        notifier.Notify(message.UserID, &message.Message)  // 通知所有在线客户端
    }
}()
```

**处理流程**：
1. 从 channel 接收消息
2. 转换为内部消息格式
3. 持久化到数据库
4. 通过 Notifier 推送至客户端

#### 3.3 WebSocket 消息分发

**代码路径**：`api/stream/stream.go:83-91`

```go
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    a.lock.RLock()
    defer a.lock.RUnlock()
    if clients, ok := a.clients[userID]; ok {
        for _, c := range clients {
            c.write <- msg  // 发送到每个客户端的写入 channel
        }
    }
}
```

**分发策略**：
- **按用户分组**：`clients[userID]` 保存同一用户的所有连接
- **广播模式**：消息推送给用户的所有在线客户端

---

## 三、完整时序图

```
┌─────────────┐     ┌────────────────┐     ┌─────────────┐     ┌──────────────────┐     ┌─────────────┐
│   Plugin    │     │ redirectToCh  │     │   Manager   │     │     Notifier     │     │  WebSocket  │
│  Instance   │────▶│   (Handler)   │────▶│  (Channel)  │────▶│   (Stream API)   │────▶│   Clients   │
│             │     │               │     │             │     │                  │     │             │
└─────────────┘     └────────────────┘     └─────────────┘     └──────────────────┘     └─────────────┘
                                                                         │
                                                                         ▼
                                                              ┌──────────────────┐
                                                              │    Database      │
                                                              │ (Message Store)  │
                                                              └──────────────────┘
```

**时序步骤详解**：

| 步骤 | 组件 | 操作 | 文件位置 |
|------|------|------|----------|
| 1 | Plugin Instance | 调用 `SendMessage(msg)` | `plugin/compat/v1.go:151` |
| 2 | redirectToChannel | 封装消息并发送到 channel | `plugin/messagehandler.go:24` |
| 3 | Manager | 从 channel 接收消息 | `plugin/manager.go:69` |
| 4 | Manager | 创建内部消息对象 | `plugin/manager.go:70-79` |
| 5 | Database | 持久化消息 | `plugin/manager.go:80` |
| 6 | Notifier | 调用 `Notify(userID, msg)` | `plugin/manager.go:82` |
| 7 | Stream API | 遍历用户所有客户端 | `api/stream/stream.go:86` |
| 8 | WebSocket Client | 写入消息到客户端 channel | `api/stream/stream.go:88` |
| 9 | Client goroutine | 发送消息到 WebSocket | `api/stream/client.go:95` |

---

## 四、鉴权上下文传递机制

### 4.1 上下文存储位置

**Gin Context 键值对**：

| Key | 存储内容 | 设置位置 |
|-----|----------|----------|
| `userIDKey` | 用户 ID (uint) | `auth/authentication.go:128` |
| `clientKey` | Client 对象 | `auth/authentication.go:156` |
| `applicationKey` | Application 对象 | `auth/authentication.go:190` |

### 4.2 获取方法

```go
// auth/token.go
func GetUserID(ctx *gin.Context) uint
func GetClient(ctx *gin.Context) *model.Client  
func GetApplication(ctx *gin.Context) *model.Application
```

### 4.3 上下文传递链路

```
HTTP Request
     │
     ▼
┌─────────────────────┐
│ Auth Middleware     │  ← 解析 Token，验证身份
│ (RequireClient)     │     ↓
└─────────────────────┘     ↓
     │                      ↓
     │              RegisterUser(ctx, user)
     │              RegisterClient(ctx, client)
     │                      ↓
     ▼                      ↓
┌─────────────────────┐     ↓
│ Handler Function    │ ←───┘
│  (GetUserID(ctx))   │  获取用户 ID
└─────────────────────┘
```

---

## 五、消息处理路径匹配

### 5.1 路径匹配规则

| 场景 | 匹配条件 | 目标用户 |
|------|----------|----------|
| 插件消息 | `MessageWithUserID.UserID` | 插件所属用户 |
| 应用消息 | `Application.UserID` | 应用所属用户 |
| 用户查询 | `auth.GetUserID(ctx)` | 当前认证用户 |

### 5.2 数据库级隔离

**代码路径**：`api/message.go:89`

```go
func (a *MessageAPI) GetMessages(ctx *gin.Context) {
    userID := auth.GetUserID(ctx)
    messages, err := a.DB.GetMessagesByUserSince(userID, params.Limit+1, params.Since)
    ...
}
```

**隔离原则**：所有消息查询和操作都基于当前认证用户的 ID 进行过滤，确保数据隔离。

---

## 六、断线重连消息不丢失机制

### 6.1 持久化保障

**存储位置**：`database/message.go`

消息在发送到客户端之前，首先被持久化到数据库：

```go
db.CreateMessage(internalMsg)           // 先保存
message.Message.ID = internalMsg.ID
notifier.Notify(message.UserID, &message.Message)  // 再推送
```

### 6.2 客户端重连策略

**轮询 API**：`api/message.go:88-98`

```go
func (a *MessageAPI) GetMessages(ctx *gin.Context) {
    userID := auth.GetUserID(ctx)
    withPaging(ctx, func(params *pagingParams) {
        messages, err := a.DB.GetMessagesByUserSince(userID, params.Limit+1, params.Since)
        ...
        ctx.JSON(200, buildWithPaging(ctx, params, messages))
    })
}
```

**重连流程**：

```
客户端断线
     │
     ▼
用户重新连接 WebSocket
     │
     ▼
调用 GET /message?since=<lastMessageID>
     │
     ▼
服务端返回遗漏的消息
     │
     ▼
客户端恢复消息序列
```

### 6.3 WebSocket 心跳机制

**代码路径**：`api/stream/client.go:81-108`

```go
func (c *client) startWriteHandler(pingPeriod time.Duration) {
    pingTicker := time.NewTicker(pingPeriod)
    defer func() {
        c.NotifyClose()
        pingTicker.Stop()
    }()

    for {
        select {
        case message, ok := <-c.write:
            // 发送消息
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            writeJSON(c.conn, message)
        case <-pingTicker.C:
            // 发送心跳
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            ping(c.conn)
        }
    }
}
```

**心跳参数**（由配置决定）：
- `pingPeriod`：服务器发送 ping 的间隔
- `pongTimeout`：等待 pong 响应的超时时间

**断线检测**：
- 读端：`startReading` 设置 `SetReadDeadline`，超时未收到 pong 则断开
- 写端：`startWriteHandler` 发送 ping，失败则断开

### 6.4 消息不丢失保障机制

| 机制 | 作用 | 实现位置 |
|------|------|----------|
| 数据库持久化 | 消息落地存储 | `plugin/manager.go:80` |
| Since 参数查询 | 支持增量拉取 | `api/message.go:88` |
| WebSocket 心跳 | 快速检测断线 | `api/stream/client.go:65` |
| 客户端缓存 | 记录最后消息 ID | 前端实现 |

---

## 七、关键数据结构

### 7.1 MessageWithUserID

```go
type MessageWithUserID struct {
    Message model.MessageExternal
    UserID  uint
}
```

**作用**：在插件发送消息时携带用户上下文，确保消息能够正确路由到目标用户。

### 7.2 redirectToChannel

```go
type redirectToChannel struct {
    ApplicationID uint
    UserID        uint
    Messages      chan MessageWithUserID
}
```

**作用**：作为插件的消息处理器，封装了目标应用、用户和消息通道。

### 7.3 Manager 结构

```go
type Manager struct {
    mutex     *sync.RWMutex
    instances map[uint]compat.PluginInstance  // 插件实例映射
    plugins   map[string]compat.Plugin        // 插件定义映射
    messages  chan MessageWithUserID          // 全局消息通道
    db        Database
    mux       *gin.RouterGroup
}
```

**核心组件**：
- `messages` channel：所有插件消息的汇聚点
- `instances`：按用户/插件配置 ID 管理实例

---

## 八、安全机制

### 8.1 Token 隔离

- **Plugin Token**：用于 Webhook 鉴权（`pluginConf.Token`）
- **Application Token**：用于消息发送鉴权
- **Client Token**：用于 API 访问鉴权

### 8.2 权限检查

**代码路径**：`api/plugin.go:406-408`

```go
func isPluginOwner(ctx *gin.Context, conf *model.PluginConf) bool {
    return conf.UserID == auth.GetUserID(ctx)
}
```

**验证原则**：用户只能访问和管理自己的插件配置。

### 8.3 Webhook 安全

**代码路径**：`plugin/manager.go:348-352`

```go
if compat.HasSupport(instance, compat.Webhooker) {
    id := pluginConf.ID
    g := m.mux.Group(pluginConf.Token+"/", requirePluginEnabled(id, m.db))
    instance.RegisterWebhook(strings.Replace(g.BasePath(), ":id", strconv.Itoa(int(id)), 1), g)
}
```

**安全措施**：
- Webhook 路径包含随机 Token
- 需要插件处于启用状态

---

## 九、总结

插件消息通道的完整流程体现了以下设计原则：

| 原则 | 实现方式 |
|------|----------|
| **解耦性** | 插件通过 channel 与主服务通信，无直接依赖 |
| **可靠性** | 消息先持久化再推送，支持断线重连恢复 |
| **安全性** | 多层鉴权、用户隔离、Token 防护 |
| **可扩展性** | 插件能力通过 Capability 接口扩展 |

**消息流转核心路径**：

```
插件 SendMessage
    ↓
redirectToChannel (封装上下文)
    ↓
manager.messages channel
    ↓
Manager goroutine (持久化 + 分发)
    ↓
Stream API.Notify (按用户广播)
    ↓
WebSocket 客户端
```