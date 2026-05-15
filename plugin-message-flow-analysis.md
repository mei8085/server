# 插件消息通道完整分析报告

## 一、系统架构概览

插件消息通道涉及三个核心层次：

| 层次 | 组件 | 职责 |
|------|------|------|
| 插件层 | `plugin/compat/` | 插件接口适配、消息发送 |
| 管理层 | `plugin/manager.go` | 插件生命周期、消息路由 |
| 分发层 | `api/stream/` | WebSocket 连接管理、实时推送 |

---

## 二、鉴权上下文存储与读取机制

### 2.1 存储结构设计

**核心数据结构**（`auth/util.go:10-14`）：

```go
type authentication struct {
    client *model.Client
    app    *model.Application
    user   *model.User
}
```

所有认证信息共享同一个存储键 `authKey = "auth"`，三个字段互斥，一次只能存储一种类型。

### 2.2 注册函数机制

**注册函数互斥性**：

| 函数 | 设置字段 | 覆盖行为 |
|------|----------|----------|
| `RegisterUser(ctx, user)` | `info.user` | 覆盖整个 auth 对象 |
| `RegisterClient(ctx, client)` | `info.client` | 覆盖整个 auth 对象 |
| `RegisterApplication(ctx, app)` | `info.app` | 覆盖整个 auth 对象 |

**代码实现**（`auth/util.go:17-29`）：

```go
func RegisterUser(ctx *gin.Context, user *model.User) {
    ctx.Set(authKey, &authentication{user: user})
}

func RegisterClient(ctx *gin.Context, client *model.Client) {
    ctx.Set(authKey, &authentication{client: client})
}

func RegisterApplication(ctx *gin.Context, app *model.Application) {
    ctx.Set(authKey, &authentication{app: app})
}
```

### 2.3 读取机制与优先级

**GetUserID 获取优先级**（`auth/util.go:48-60`）：

```go
func TryGetUserID(ctx *gin.Context) *uint {
    info := getInfo(ctx)
    switch {
    case info.user != nil:
        return &info.user.ID          // 优先级1：用户直接登录
    case info.client != nil:
        return &info.client.UserID    // 优先级2：客户端 Token
    case info.app != nil:
        return &info.app.UserID       // 优先级3：应用 Token
    default:
        return nil
    }
}
```

**优先级关系图**：

```
用户登录 (Basic Auth)
        │
        ▼
RegisterUser(info.user) ──┐
                          │
客户端登录 (Client Token) │
        │                 │
        ▼                 ▼
RegisterClient(info.client) ──▶ GetUserID() ──▶ 返回 UserID
                          │                 │
应用登录 (App Token)      │                 │
        │                 │                 ▼
        ▼                 └───────────▶ panic("token and user may not be null")
RegisterApplication(info.app)
```

### 2.4 认证流程中的调用时机

**认证中间件调用链**（`auth/authentication.go`）：

| 认证类型 | 触发条件 | 注册函数调用 |
|----------|----------|--------------|
| 用户认证 | Basic Auth 成功 | `RegisterUser(ctx, user)` |
| 客户端认证 | Client Token 有效 | `RegisterClient(ctx, client)` |
| 应用认证 | Application Token 有效 | `RegisterApplication(ctx, app)` |

---

## 三、完整消息路径匹配逻辑

### 3.1 路径匹配全景图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        插件消息完整路径匹配                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌─────────────┐    ┌──────────────────┐    ┌──────────────────┐              │
│  │  Plugin     │    │  redirectToCh    │    │    Manager      │              │
│  │  Instance   │───▶│  (Handler)       │───▶│  (Channel)      │              │
│  │             │    │                  │    │                 │              │
│  │  Bind:      │    │  Bind:           │    │  Bind:          │              │
│  │  - UserID   │    │  - UserID        │    │  - messages     │              │
│  │  - AppID    │    │  - ApplicationID │    │    channel      │              │
│  └─────────────┘    └──────────────────┘    └────────┬─────────┘              │
│                                                       │                       │
│                       ┌───────────────────────────────┼───────────────────┐    │
│                       ▼                               ▼                   ▼    │
│                ┌───────────────┐            ┌────────────────┐   ┌──────────┐ │
│                │   Database    │            │   Stream API   │   │  Client  │ │
│                │  CreateMsg    │            │  Notify()      │   │  Reconnect│ │
│                └───────┬───────┘            └───────┬────────┘   └────┬─────┘ │
│                       │                            │                 │        │
│                       ▼                            ▼                 ▼        │
│                ┌───────────────┐            ┌────────────────┐   ┌──────────┐ │
│                │ MessageStore  │◀───────────│  clients[UID]  │◀──│ GET /msg │ │
│                │               │            │  按UserID广播   │   │ ?since= │ │
│                └───────────────┘            └────────────────┘   └──────────┘ │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 阶段一：插件实例绑定

**代码路径**：`plugin/manager.go:315-341`

```go
func (m *Manager) initializeSingleUserPlugin(userCtx compat.UserContext, p compat.Plugin) error {
    info := p.PluginInfo()
    instance := p.NewPluginInstance(userCtx)  // 注入用户上下文
    userID := userCtx.ID

    pluginConf, err := m.db.GetPluginConfByUserAndPath(userID, info.ModulePath)
    if pluginConf == nil {
        pluginConf, err = m.createPluginConf(instance, info, userID)
    }

    m.instances[pluginConf.ID] = instance  // 注册实例

    if compat.HasSupport(instance, compat.Messenger) {
        instance.SetMessageHandler(redirectToChannel{
            ApplicationID: pluginConf.ApplicationID,  // 绑定应用ID
            UserID:        pluginConf.UserID,         // 绑定用户ID
            Messages:      m.messages,                // 绑定消息通道
        })
    }
    ...
}
```

**绑定的数据结构**：

```go
type redirectToChannel struct {
    ApplicationID uint                        // 自动创建的内部应用ID
    UserID        uint                        // 插件所属用户ID
    Messages      chan MessageWithUserID      // 全局消息通道引用
}
```

### 3.3 阶段二：消息封装与通道分发

**代码路径**：`plugin/messagehandler.go:23-34`

```go
func (c redirectToChannel) SendMessage(msg compat.Message) error {
    c.Messages <- MessageWithUserID{
        Message: model.MessageExternal{
            ApplicationID: c.ApplicationID,  // 传递应用ID
            Message:       msg.Message,
            Title:         msg.Title,
            Priority:      &msg.Priority,
            Date:          time.Now(),
            Extras:        msg.Extras,
        },
        UserID: c.UserID,  // 传递用户ID（关键路由信息）
    }
    return nil
}
```

**封装后的消息结构**：

```go
type MessageWithUserID struct {
    Message model.MessageExternal  // 消息内容
    UserID  uint                   // 用户标识（用于路由）
}
```

### 3.4 阶段三：Manager 处理与分发

**代码路径**：`plugin/manager.go:67-84`

```go
go func() {
    for {
        message := <-manager.messages  // 从通道接收
        
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
        
        db.CreateMessage(internalMsg)           // 步骤1：持久化
        message.Message.ID = internalMsg.ID
        notifier.Notify(message.UserID, &message.Message)  // 步骤2：按UserID通知
    }
}()
```

### 3.5 阶段四：Stream 按 UserID 广播

**代码路径**：`api/stream/stream.go:83-91`

```go
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    a.lock.RLock()
    defer a.lock.RUnlock()
    
    if clients, ok := a.clients[userID]; ok {  // 按UserID查找客户端列表
        for _, c := range clients {
            c.write <- msg  // 广播到该用户的所有连接
        }
    }
}
```

**客户端存储结构**：

```go
type API struct {
    clients map[uint][]*client  // key: UserID, value: 该用户的所有WebSocket连接
    ...
}
```

### 3.6 阶段五：断线重连与 Since 补拉

**代码路径**：`api/message.go:88-98`

```go
func (a *MessageAPI) GetMessages(ctx *gin.Context) {
    userID := auth.GetUserID(ctx)  // 获取当前用户ID
    
    withPaging(ctx, func(params *pagingParams) {
        // since参数：返回ID大于since的消息
        messages, err := a.DB.GetMessagesByUserSince(userID, params.Limit+1, params.Since)
        if success := successOrAbort(ctx, 500, err); !success {
            return
        }
        ctx.JSON(200, buildWithPaging(ctx, params, messages))
    })
}
```

**补拉流程**：

```
客户端断线
     │
     ▼
记录最后收到的消息ID (lastMsgID)
     │
     ▼
重新连接 WebSocket
     │
     ▼
调用 GET /message?since=lastMsgID
     │
     ▼
服务端返回 ID > lastMsgID 的消息
     │
     ▼
客户端恢复消息序列
```

---

## 四、完整时序详解

### 4.1 时序流程图

```
Plugin Instance          redirectToChannel        Manager             Stream API           WebSocket Client
       │                        │                     │                     │                      │
       │ SendMessage(msg)       │                     │                     │                      │
       │───────────────────────▶│                     │                     │                      │
       │                        │                     │                     │                      │
       │                        │ 写入 messages chan  │                     │                      │
       │                        │────────────────────▶│                     │                      │
       │                        │                     │                     │                      │
       │                        │                     │ 接收消息             │                      │
       │                        │                     │                     │                      │
       │                        │                     │ 创建 internalMsg     │                      │
       │                        │                     │                     │                      │
       │                        │                     │ db.CreateMessage()   │                      │
       │                        │                     │─────────────────────▶│                      │
       │                        │                     │                     │                      │
       │                        │                     │ notifier.Notify(UID)│                      │
       │                        │                     │─────────────────────▶│                      │
       │                        │                     │                     │                      │
       │                        │                     │                     │ 遍历 clients[UID]    │
       │                        │                     │                     │                      │
       │                        │                     │                     │ 写入 c.write chan    │
       │                        │                     │                     │─────────────────────▶│
       │                        │                     │                     │                      │
       │                        │                     │                     │                      │ writeJSON(conn, msg)
       │                        │                     │                     │                      │
```

### 4.2 时序步骤详解

| 步骤 | 组件 | 操作 | 文件位置 | 关键数据 |
|------|------|------|----------|----------|
| 1 | Plugin Instance | 调用 `SendMessage(msg)` | `plugin/compat/v1.go:151` | `Message{Title, Message, Priority}` |
| 2 | redirectToChannel | 封装为 `MessageWithUserID` | `plugin/messagehandler.go:24` | `UserID, ApplicationID` |
| 3 | redirectToChannel | 发送到 `manager.messages` channel | `plugin/messagehandler.go:24` | channel 写入 |
| 4 | Manager | 从 channel 接收消息 | `plugin/manager.go:69` | `MessageWithUserID` |
| 5 | Manager | 创建 `model.Message` 内部对象 | `plugin/manager.go:70-79` | 转换消息格式 |
| 6 | Database | `CreateMessage(internalMsg)` | `plugin/manager.go:80` | 持久化到 DB |
| 7 | Manager | 调用 `notifier.Notify(UserID, msg)` | `plugin/manager.go:82` | 传递 UserID |
| 8 | Stream API | 查找 `clients[UserID]` | `api/stream/stream.go:86` | 按用户路由 |
| 9 | Stream API | 遍历客户端并写入 `c.write` | `api/stream/stream.go:88` | 广播消息 |
| 10 | WebSocket Client | `writeJSON(conn, msg)` | `api/stream/client.go:95` | 发送到客户端 |

---

## 五、失败边界说明

### 5.1 鉴权失败边界

| 边界场景 | 触发条件 | 处理方式 | 代码位置 |
|----------|----------|----------|----------|
| Token 不存在 | 请求未携带有效 Token | 返回 401 Unauthorized | `auth/authentication.go:115` |
| Token 无效 | Token 格式正确但不存在于 DB | 返回 401 Unauthorized | `auth/authentication.go:128/155/188` |
| 用户不存在 | 用户被删除但 Token 仍有效 | `GetUserID()` panic | `auth/util.go:41-44` |
| 权限不足 | 非管理员访问管理员接口 | 返回 403 Forbidden | `auth/authentication.go:120` |

### 5.2 消息发送失败边界

| 边界场景 | 触发条件 | 处理方式 | 代码位置 |
|----------|----------|----------|----------|
| 消息通道阻塞 | channel 满且无消费者 | `SendMessage()` 阻塞 | `plugin/messagehandler.go:24` |
| 数据库写入失败 | DB 连接异常 | 消息丢失（无重试） | `plugin/manager.go:80` |
| 插件未启用 | `pluginConf.Enabled = false` | 消息被丢弃 | `plugin/manager.go:353-362` |
| ApplicationID 无效 | 内部应用被删除 | 消息写入失败 | `plugin/messagehandler.go:26` |

### 5.3 消息分发失败边界

| 边界场景 | 触发条件 | 处理方式 | 代码位置 |
|----------|----------|----------|----------|
| 用户无在线客户端 | `clients[userID]` 为空 | 消息仅持久化，不推送 | `api/stream/stream.go:86` |
| WebSocket 写入失败 | 连接断开或超时 | 调用 `NotifyClose()` | `api/stream/client.go:96-98` |
| 客户端 channel 阻塞 | `c.write` 满 | 发送 goroutine 阻塞 | `api/stream/client.go:88-99` |

### 5.4 断线重连失败边界

| 边界场景 | 触发条件 | 处理方式 | 代码位置 |
|----------|----------|----------|----------|
| 消息 ID 溢出 | `since` 参数超过最大值 | 返回空列表 | `api/message.go:89` |
| 网络分区 | 客户端无法连接 | 持续重试（前端逻辑） | 前端实现 |
| 消息被删除 | `since` 对应消息已删除 | 返回后续消息 | `api/message.go:89` |

### 5.5 资源竞争边界

| 边界场景 | 触发条件 | 保护机制 | 代码位置 |
|----------|----------|----------|----------|
| 并发注册插件 | 多用户同时初始化 | `m.mutex.Lock()` | `plugin/manager.go:268` |
| 并发修改客户端列表 | 同时注册/注销连接 | `a.lock.RLock/RUnlock()` | `api/stream/stream.go:84` |
| 并发消息写入 | 多个插件同时发消息 | channel 串行化 | `plugin/manager.go:69` |

---

## 六、关键数据结构关系

### 6.1 核心数据结构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        数据结构关系图                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Plugin Instance                                                        │
│  ├── UserContext { ID, Name, Admin }                                   │
│  └── MessageHandler: redirectToChannel                                  │
│        ├── ApplicationID (uint)                                        │
│        ├── UserID (uint)                                               │
│        └── Messages (chan MessageWithUserID)                           │
│                                                                         │
│  MessageWithUserID                                                      │
│  ├── Message: MessageExternal                                          │
│  │     ├── ID, ApplicationID, Title, Message, Priority, Date, Extras    │
│  │     └── (来自插件或应用)                                             │
│  └── UserID (uint) ──────┐                                             │
│                          │                                              │
│                          ▼                                              │
│  Stream API.clients                                                      │
│       └── map[uint][]*client  ←── 按 UserID 路由                        │
│              └── *client { conn, write, userID, token }                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 数据流转表格

| 数据项 | 来源 | 传递路径 | 用途 |
|--------|------|----------|------|
| `UserID` | 插件初始化时注入 | `UserContext` → `redirectToChannel` → `MessageWithUserID` | 消息路由、权限验证 |
| `ApplicationID` | 自动创建内部应用 | `PluginConf` → `redirectToChannel` → `Message` | 消息归属、统计 |
| `Message` | 插件调用 `SendMessage()` | `compat.Message` → `MessageExternal` → `Message` | 消息内容 |
| `Token` | 客户端/WebSocket连接 | `auth.GetClient()` → `client.token` | 连接标识、权限验证 |

---

## 七、安全机制

### 7.1 Token 隔离机制

| Token 类型 | 前缀 | 用途 | 存储位置 |
|------------|------|------|----------|
| Application Token | `A` | 消息发送 API | `model.Application.Token` |
| Client Token | `C` | 客户端 API 访问 | `model.Client.Token` |
| Plugin Token | `P` | Webhook 鉴权 | `model.PluginConf.Token` |

### 7.2 权限检查链

```
HTTP Request
     │
     ▼
Auth Middleware (RequireClient/RequireApplicationToken)
     │
     ▼
Token 解析 → DB 查询 → Register* → GetUserID()
     │
     ▼
Handler 业务逻辑
     │
     ▼
权限验证 (isPluginOwner, app.UserID == currentUserID)
```

**代码示例**（`api/plugin.go:406-408`）：

```go
func isPluginOwner(ctx *gin.Context, conf *model.PluginConf) bool {
    return conf.UserID == auth.GetUserID(ctx)
}
```

---

## 八、总结

### 8.1 核心设计原则

| 原则 | 实现方式 |
|------|----------|
| **解耦性** | 插件通过 channel 与主服务通信，无直接依赖 |
| **可靠性** | 消息先持久化再推送，支持断线重连恢复 |
| **安全性** | 多层鉴权、用户隔离、Token 防护 |
| **可扩展性** | 插件能力通过 Capability 接口扩展 |
| **一致性** | 插件消息与应用消息统一处理路径 |

### 8.2 消息流转核心路径

```
插件调用 SendMessage(msg)
    ↓
redirectToChannel 封装 (UserID + ApplicationID + Message)
    ↓
manager.messages channel (goroutine 串行处理)
    ↓
Database.CreateMessage() (持久化保障)
    ↓
notifier.Notify(UserID, msg) (按用户路由)
    ↓
Stream API.clients[UserID] (广播到所有连接)
    ↓
WebSocket 客户端写入
    ↓
断线重连: GET /message?since=lastID (补拉遗漏消息)
```

### 8.3 关键技术点

1. **鉴权上下文共享**：`RegisterUser/Client/Application` 互斥，`GetUserID` 按优先级返回
2. **消息路由键**：`UserID` 是贯穿整个消息路径的核心路由标识
3. **持久化优先**：消息先写入数据库再推送，保证不丢失
4. **增量拉取**：`since` 参数支持断线后高效恢复消息序列