# 长连推送通道生命周期分析报告

## 1. 概述

本报告详细分析了项目中长连推送通道（WebSocket）从握手建立到客户端断开期间的完整生命周期，重点关注连接注册、心跳维护、消息派发与异常清理之间的衔接关系。

核心代码位于 `api/stream/` 目录下：
- `stream.go` - WebSocket 连接管理器（API）
- `client.go` - 单个客户端连接的管理
- `once.go` - 确保关闭操作只执行一次的原子工具

## 2. 生命周期流程概览

```
客户端握手建立连接
    ↓
连接注册到管理器
    ↓
启动两个协程：读协程 + 写协程
    ↓
心跳维护（ping/pong 机制）
    ↓
消息派发（从应用到客户端）
    ↓
异常清理（多种触发方式）
    ↓
资源释放与连接移除
```

## 3. 详细阶段分析

### 3.1 握手建立阶段

**入口函数**: `API.Handle()` (stream.go:143)

```go
func (a *API) Handle(ctx *gin.Context) {
    conn, err := a.upgrader.Upgrade(ctx.Writer, ctx.Request, nil)
    if err != nil {
        ctx.Error(err)
        return
    }
    // ... 后续处理
}
```

**关键步骤**:
1. **协议升级**: 使用 `websocket.Upgrader.Upgrade()` 将 HTTP 请求升级为 WebSocket 连接
2. **源验证**: 通过 `newUpgrader()` 创建的 upgrader 会验证 Origin 头（生产环境）
   - 开发模式允许所有来源
   - 生产模式需要检查 `allowedWebSocketOrigins` 配置
3. **认证信息提取**:
   - 从上下文获取用户ID: `auth.GetUserID(ctx)`
   - 从上下文获取客户端 token: `auth.GetClient(ctx).Token`

### 3.2 连接注册阶段

**关键函数**: `API.register()` (stream.go:106)

```go
func (a *API) register(client *client) {
    a.lock.Lock()
    defer a.lock.Unlock()
    a.clients[client.userID] = append(a.clients[client.userID], client)
}
```

**注册流程**:
1. **创建客户端实例**: `newClient(conn, userID, token, a.remove)`
   - 传入连接、用户ID、token
   - 传入 `a.remove` 回调用于后续清理
   - 初始化消息通道: `write: make(chan *model.MessageExternal, 1)`（容量为1）
2. **注册到管理器**: `a.register(client)`
   - 使用 `sync.RWMutex` 保证线程安全
   - 按 userID 分组存储在 `map[uint][]*client` 中

**数据结构**:
```go
type API struct {
    clients     map[uint][]*client  // 用户ID -> 客户端列表
    lock        sync.RWMutex
    pingPeriod  time.Duration
    pongTimeout time.Duration
    upgrader    *websocket.Upgrader
}
```

### 3.3 协程启动阶段

**关键代码**: (stream.go:156-157)

```go
go client.startReading(a.pongTimeout)
go client.startWriteHandler(a.pingPeriod)
```

**两个协程的职责**:

#### 3.3.1 读协程 (`startReading`)

**函数**: `client.startReading()` (client.go:61)

```go
func (c *client) startReading(pongWait time.Duration) {
    defer c.NotifyClose()
    c.conn.SetReadLimit(64)  // 限制读取大小，防止攻击
    c.conn.SetReadDeadline(time.Now().Add(pongWait))
    c.conn.SetPongHandler(func(appData string) error {
        c.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })
    for {
        if _, _, err := c.conn.NextReader(); err != nil {
            printWebSocketError("ReadError", err)
            return
        }
    }
}
```

**职责**:
1. **设置读取限制**: 最多读取 64 字节（客户端不发送业务数据，只响应心跳）
2. **设置读取超时**: 初始设置为 `pingPeriod + pongTimeout`
3. **Pong 处理器**: 收到客户端 pong 响应后，重置读取超时时间
4. **循环读取**: 持续读取，忽略实际内容（因为客户端不发送业务消息）
5. **异常退出**: 读取出错时触发 `NotifyClose()` 进行清理

#### 3.3.2 写协程 (`startWriteHandler`)

**函数**: `client.startWriteHandler()` (client.go:81)

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
            if !ok {
                return
            }
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := writeJSON(c.conn, message); err != nil {
                printWebSocketError("WriteError", err)
                return
            }
        case <-pingTicker.C:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := ping(c.conn); err != nil {
                printWebSocketError("PingError", err)
                return
            }
        }
    }
}
```

**职责**:
1. **心跳定时器**: 按 `pingPeriod` 间隔发送 ping
2. **消息发送**: 从 `c.write` 通道接收消息并发送给客户端
3. **写入超时**: 每次写入设置 2 秒超时 (`writeWait = 2 * time.Second`)
4. **异常退出**: 写入失败或 ping 失败时触发 `NotifyClose()`

### 3.4 心跳维护阶段

**心跳机制**: 使用 WebSocket 标准的 ping/pong 帧

**服务器端**:
- 写协程按 `pingPeriod` 定时发送 ping 帧
- 写超时为 2 秒
- 发送失败则认为连接已断开

**客户端响应**:
- 收到 ping 后自动回复 pong（WebSocket 协议层面）
- 读协程的 PongHandler 会重置读取超时

**超时判定**:
- 读协程设置 `SetReadDeadline`
- 若在 `pingPeriod + pongTimeout` 内未收到任何数据（包括 pong）
- `NextReader()` 会返回超时错误
- 触发连接清理

**关键参数配置** (stream.go:31-38):
```go
func New(pingPeriod, pongTimeout time.Duration, allowedWebSocketOrigins []string) *API {
    return &API{
        pingPeriod:  pingPeriod,
        pongTimeout: pingPeriod + pongTimeout,  // 注意：这里相加了
        // ...
    }
}
```

### 3.5 消息派发阶段

#### 3.5.1 消息来源

**消息创建入口**: `MessageAPI.CreateMessage()` (api/message.go:363)

```go
func (a *MessageAPI) CreateMessage(ctx *gin.Context) {
    // ... 验证和创建消息
    a.Notifier.Notify(auth.GetUserID(ctx), toExternalMessage(msgInternal))
    ctx.JSON(200, toExternalMessage(msgInternal))
}
```

**Notifier 接口** (api/message.go:32-34):
```go
type Notifier interface {
    Notify(userID uint, message *model.MessageExternal)
}
```

`stream.API` 实现了这个接口。

#### 3.5.2 消息广播

**函数**: `API.Notify()` (stream.go:83)

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

**派发流程**:
1. 获取读锁（允许多个读，不阻塞其他读）
2. 根据 userID 查找所有连接的客户端
3. 将消息写入每个客户端的 `write` 通道
4. 写协程从通道取出并发送

**关键设计**:
- 使用 `RLock` 而非 `Lock`，提高并发读性能
- 消息通道容量为 1（`make(chan *model.MessageExternal, 1)`）
- 如果客户端写协程处理慢，后续消息可能会阻塞发送者（测试中使用了 `time.Sleep` 等待）

### 3.6 异常清理阶段

**清理触发方式有多种**：

#### 3.6.1 客户端主动断开

- 读协程 `NextReader()` 收到关闭错误
- 触发 `NotifyClose()`

#### 3.6.2 心跳超时

- 客户端未在规定时间内响应 pong
- 读超时触发 `NextReader()` 返回错误
- 触发 `NotifyClose()`

#### 3.6.3 写入失败

- 写协程发送消息或 ping 失败
- 触发 `NotifyClose()`

#### 3.6.4 客户端被删除

**函数**: `API.NotifyDeletedClient()` (stream.go:67)

```go
func (a *API) NotifyDeletedClient(userID uint, token string) {
    a.lock.Lock()
    defer a.lock.Unlock()
    if clients, ok := a.clients[userID]; ok {
        for i := len(clients) - 1; i >= 0; i-- {
            client := clients[i]
            if client.token == token {
                client.Close()
                clients = append(clients[:i], clients[i+1:]...)
            }
        }
        a.clients[userID] = clients
    }
}
```

**流程**:
1. 获取写锁
2. 遍历该用户的所有客户端
3. 匹配 token，调用 `client.Close()`
4. 从列表中移除

#### 3.6.5 用户被删除

**函数**: `API.NotifyDeletedUser()` (stream.go:54)

```go
func (a *API) NotifyDeletedUser(userID uint) error {
    a.lock.Lock()
    defer a.lock.Unlock()
    if clients, ok := a.clients[userID]; ok {
        for _, client := range clients {
            client.Close()
        }
        delete(a.clients, userID)
    }
    return nil
}
```

**流程**:
1. 获取写锁
2. 关闭该用户所有连接
3. 从 map 中删除整个用户条目

#### 3.6.6 服务关闭

**函数**: `API.Close()` (stream.go:161)

```go
func (a *API) Close() {
    a.lock.Lock()
    defer a.lock.Unlock()

    for _, clients := range a.clients {
        for _, client := range clients {
            client.Close()
        }
    }
    for k := range a.clients {
        delete(a.clients, k)
    }
}
```

## 4. 关键衔接点分析

### 4.1 注册与清理的衔接

**注册时传入回调**:
```go
client := newClient(conn, auth.GetUserID(ctx), token, a.remove)
```

`a.remove` 是 `API.remove()` 方法，用于从管理器中移除客户端。

**清理时调用回调**:
```go
func (c *client) NotifyClose() {
    c.once.Do(func() {
        c.conn.Close()
        close(c.write)
        c.onClose(c)  // 调用 a.remove(c)
    })
}
```

### 4.2 协程退出与清理的衔接

两个协程都使用 `defer c.NotifyClose()` 确保退出时清理：
- 读协程: 读取失败 → 函数返回 → defer 触发 → NotifyClose
- 写协程: 写入失败或通道关闭 → 函数返回 → defer 触发 → NotifyClose

### 4.3 原子关闭保障

使用自定义的 `once` 结构体（once.go）确保关闭操作只执行一次：

```go
func (c *client) NotifyClose() {
    c.once.Do(func() {
        c.conn.Close()
        close(c.write)
        c.onClose(c)
    })
}
```

**与标准 `sync.Once` 的区别**:
- 标准 `sync.Once` 在执行函数时持有锁
- 自定义版本提前解锁，避免死锁（因为 `c.onClose(c)` 会调用 `API.remove()` 需要获取锁）

### 4.4 消息派发与连接状态的衔接

`API.Notify()` 使用 `RLock`：
- 允许并发派发消息
- 但如果在派发过程中客户端被移除，可能会向已关闭的通道写入？

**实际安全保障**:
- `client.Close()` 只关闭连接，不关闭通道
- `client.NotifyClose()` 才会关闭 `write` 通道
- 但 `Notify()` 写入时如果通道已关闭会 panic？

查看测试用例 `TestWriteMessageFails`，实际是通过写协程处理的：
- `Notify()` 写入 `c.write` 通道（如果通道已关闭会 panic，但 once 机制保证不会重复关闭）
- 写协程从通道读取并发送
- 发送失败时触发清理

## 5. 并发安全分析

| 操作 | 锁类型 | 说明 |
|------|--------|------|
| 注册客户端 | `Lock` | 写入 map |
| 移除客户端 | `Lock` | 修改 map 切片 |
| 派发消息 | `RLock` | 读取 map，允许并发 |
| 关闭所有连接 | `Lock` | 修改 map |
| 统计连接数 | `RLock` | 读取 map |

## 6. 测试用例验证

从 `stream_test.go` 可以验证以下场景：

1. **心跳测试** (`TestPing`): 验证 ping 发送和消息接收正常
2. **超时清理** (`TestCloseClientOnNotReading`): 验证不响应心跳的客户端被清理
3. **消息派发** (`TestMessageDirectlyAfterConnect`): 验证连接后立即能收到消息
4. **客户端删除** (`TestDeleteClientShouldCloseConnection`): 验证删除 token 会断开连接
5. **用户删除** (`TestDeleteUser`): 验证删除用户会断开所有连接
6. **多客户端** (`TestMultipleClients`): 验证同一用户多设备连接和独立断开

## 7. 总结

长连推送通道的生命周期设计清晰，关键点：

1. **双协程架构**: 读写分离，各自独立处理异常
2. **分层管理**: `API` 负责连接注册和派发，`client` 负责单个连接管理
3. **心跳机制**: 利用 WebSocket ping/pong + 超时检测保活
4. **原子关闭**: 自定义 `once` 确保清理只执行一次且避免死锁
5. **多维度清理**: 支持客户端断开、超时、主动删除等多种触发方式
6. **并发安全**: 通过 `RWMutex` 平衡性能和安全
