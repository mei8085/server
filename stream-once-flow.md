# 一次性事件流响应链路分析报告

## 概述

本报告详细梳理了从事件流请求进来到连接关闭的完整链路，包括涉及的各个环节、资源分配与清理动作。系统采用 WebSocket 协议实现实时消息推送，通过自定义的 `once` 机制确保资源清理的原子性和幂等性。

---

## 核心组件

### 1. `once` 结构体 (`api/stream/once.go`)

这是整个链路中确保一次性执行的关键组件，基于标准库 `sync.Once` 修改：

- **特点**：解锁互斥锁的时间更早，在执行函数 `f()` 时不会持有锁
- **作用**：确保连接关闭和资源清理操作只执行一次

```go
type once struct {
    m    sync.Mutex
    done uint32
}
```

### 2. `client` 结构体 (`api/stream/client.go`)

代表单个客户端连接，包含：

- `conn`: WebSocket 连接对象
- `onClose`: 连接关闭时的回调函数（用于从 API 的客户端列表中移除）
- `write`: 消息发送通道（带缓冲，容量为 1）
- `userID`: 用户 ID
- `token`: 客户端 token
- `once`: 确保关闭操作只执行一次的机制

### 3. `API` 结构体 (`api/stream/stream.go`)

管理所有客户端连接的核心结构：

- `clients`: 按用户 ID 分组的客户端列表映射
- `lock`: 保护客户端映射的读写锁
- `pingPeriod`: 心跳发送间隔
- `pongTimeout`: 心跳响应超时时间

---

## 完整链路分析

### 阶段 1：请求进入与协议升级

**入口点**：`router/router.go:215` - `GET /stream` 路由

1. **路由匹配与认证**
   - 请求通过 `authentication.RequireClient` 中间件进行客户端认证
   - 认证成功后调用 `streamHandler.Handle(ctx *gin.Context)`

2. **WebSocket 协议升级**
   - `stream.go:144`: `a.upgrader.Upgrade(ctx.Writer, ctx.Request, nil)`
   - 将 HTTP 连接升级为 WebSocket 协议
   - 检查 Origin 头（生产环境下）

3. **客户端创建与注册**
   - `stream.go:150-153`: 获取客户端 token（如果存在）
   - `stream.go:154`: 创建新的 `client` 对象
     ```go
     client := newClient(conn, auth.GetUserID(ctx), token, a.remove)
     ```
   - `stream.go:155`: 注册到 API 的客户端映射中
     ```go
     a.register(client)
     ```

4. **启动处理 goroutine**
   - `stream.go:156`: 启动读取 goroutine
     ```go
     go client.startReading(a.pongTimeout)
     ```
   - `stream.go:157`: 启动写入 goroutine
     ```go
     go client.startWriteHandler(a.pingPeriod)
     ```

### 阶段 2：消息传输与心跳维持

#### 2.1 消息接收与连接保活 (`startReading`)

**文件**：`api/stream/client.go:61-75`

```go
func (c *client) startReading(pongWait time.Duration) {
    defer c.NotifyClose()
    c.conn.SetReadLimit(64)
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

**主要职责**：
1. 设置读取限制（64 字节，防止大消息攻击）
2. 设置读取超时（pongWait）
3. 注册 Pong 处理器，收到 Pong 时重置读取超时
4. 循环读取客户端消息（实际忽略内容，只检测连接状态）
5. **关键**：`defer c.NotifyClose()` 确保无论何种方式退出都会触发关闭

#### 2.2 消息发送与心跳 (`startWriteHandler`)

**文件**：`api/stream/client.go:81-108`

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

**主要职责**：
1. 创建心跳定时器（`pingTicker`）
2. 使用 `select` 同时监听：
   - `write` 通道：发送消息给客户端
   - `pingTicker.C`：定时发送心跳
3. 发送消息时设置 2 秒写入超时
4. **关键**：`defer` 确保退出时关闭连接和停止定时器

### 阶段 3：消息通知机制

**入口点**：`stream.go:83-91` - `Notify` 方法

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

**调用链**：
1. 消息创建时（`api/message.go`），调用 `messageHandler.Notifier.Notify(userID, msg)`
2. 遍历该用户的所有连接客户端
3. 将消息发送到每个客户端的 `write` 通道
4. 由 `startWriteHandler` goroutine 实际发送

### 阶段 4：连接关闭与资源清理

#### 4.1 关闭触发方式

连接关闭可以由以下几种方式触发：

1. **客户端主动断开**
   - `startReading` 中 `conn.NextReader()` 返回错误
   - 触发 `defer c.NotifyClose()`

2. **心跳超时**
   - 客户端未在 `pongTimeout` 内响应 Pong
   - `startReading` 中读取超时，`NextReader()` 返回错误

3. **写入错误**
   - `startWriteHandler` 中发送消息或心跳失败
   - 函数返回，触发 `defer c.NotifyClose()`

4. **主动关闭**
   - 用户删除时：`stream.go:54-64` - `NotifyDeletedUser`
   - 客户端删除时：`stream.go:67-80` - `NotifyDeletedClient`
   - 服务关闭时：`stream.go:161-173` - `Close`

#### 4.2 关闭流程详解

##### 4.2.1 `NotifyClose()` 方法

**文件**：`api/stream/client.go:51-57`

```go
func (c *client) NotifyClose() {
    c.once.Do(func() {
        c.conn.Close()
        close(c.write)
        c.onClose(c)
    })
}
```

**执行步骤**：
1. **`once.Do()` 检查**：确保只执行一次
   - 原子读取 `done` 标志，若已为 1 则直接返回
   - 否则获取互斥锁，再次检查，设置 `done = 1`
2. **关闭 WebSocket 连接**：`c.conn.Close()`
3. **关闭消息通道**：`close(c.write)`
   - 通知 `startWriteHandler` 退出循环
4. **从 API 移除客户端**：`c.onClose(c)`
   - 调用 `a.remove(client)` 方法

##### 4.2.2 从 API 移除客户端

**文件**：`api/stream/stream.go:93-104`

```go
func (a *API) remove(remove *client) {
    a.lock.Lock()
    defer a.lock.Unlock()
    if userIDClients, ok := a.clients[remove.userID]; ok {
        for i, client := range userIDClients {
            if client == remove {
                a.clients[remove.userID] = append(userIDClients[:i], userIDClients[i+1:]...)
                break
            }
        }
    }
}
```

**执行步骤**：
1. 获取写锁保护客户端映射
2. 查找该用户的客户端列表
3. 遍历找到目标客户端
4. 从切片中移除该客户端
5. 释放锁

##### 4.2.3 goroutine 退出流程

当 `NotifyClose()` 被调用后：

**`startWriteHandler` goroutine**：
- `write` 通道被关闭
- `select` 中 `message, ok := <-c.write` 的 `ok` 为 `false`
- 函数返回
- 执行 `defer`：
  - 再次调用 `c.NotifyClose()`（但 `once` 确保只执行一次）
  - 停止 `pingTicker`

**`startReading` goroutine**：
- 若连接已关闭，`conn.NextReader()` 返回错误
- 函数返回
- 执行 `defer c.NotifyClose()`（同样被 `once` 过滤）

#### 4.3 资源清理清单

| 资源类型 | 分配位置 | 清理位置 | 清理方式 |
|---------|---------|---------|---------|
| WebSocket 连接 | `stream.go:144` Upgrade | `client.go:45`/`53` | `conn.Close()` |
| 消息通道 `write` | `client.go:35` | `client.go:46`/`54` | `close(write)` |
| 心跳定时器 | `client.go:82` | `client.go:85` | `ticker.Stop()` |
| API 客户端映射项 | `stream.go:155` register | `stream.go:93-104` remove | 从切片移除 |
| 读取 goroutine | `stream.go:156` | `client.go:62` defer | 自然退出 |
| 写入 goroutine | `stream.go:157` | `client.go:84` defer | 自然退出 |

---

## 一次性机制详解

### `once.Do()` 执行流程

```go
func (o *once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 1 {
        return  // 快速路径：原子读取，已执行则直接返回
    }
    if o.mayExecute() {  // 慢速路径：需要获取锁确认
        f()
    }
}

func (o *once) mayExecute() bool {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        atomic.StoreUint32(&o.done, 1)  // 原子设置，防止并发
        return true
    }
    return false
}
```

### 关键设计点

1. **双重检查锁定**：
   - 第一次：原子读取（无锁，快速路径）
   - 第二次：持有锁后读取（确保正确性）

2. **提前解锁**：
   - 与标准库 `sync.Once` 不同，此版本在设置 `done=1` 后立即解锁
   - 执行 `f()` 时不持有锁，避免潜在的死锁

3. **幂等性保证**：
   - 无论调用多少次 `Close()` 或 `NotifyClose()`
   - 无论从哪个 goroutine 调用
   - 实际清理逻辑只执行一次

### 并发场景示例

假设有以下并发情况：
- Goroutine A：读取超时，调用 `NotifyClose()`
- Goroutine B：写入失败，调用 `NotifyClose()`
- Goroutine C：用户被删除，调用 `client.Close()`

执行顺序可能如下：
1. A 调用 `NotifyClose()` → `once.Do()` → `done=0` → 获取锁 → 设置 `done=1` → 执行清理
2. B 调用 `NotifyClose()` → `once.Do()` → 原子读取 `done=1` → 直接返回
3. C 调用 `Close()` → `once.Do()` → 原子读取 `done=1` → 直接返回

**结果**：清理逻辑只执行一次，所有后续调用都被安全忽略。

---

## 特殊场景处理

### 1. 用户删除时的连接关闭

**文件**：`stream.go:54-64`

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

**流程**：
1. 获取写锁
2. 获取该用户的所有客户端
3. 对每个客户端调用 `Close()`
   - 注意：`Close()` 只关闭连接和通道，不调用 `onClose`
4. 直接从映射中删除整个用户条目
5. 释放锁

**为什么用 `Close()` 而不是 `NotifyClose()`？**
- 因为后面会直接 `delete(a.clients, userID)`
- 不需要每个客户端都回调 `onClose` 来逐个移除
- 更高效

### 2. 客户端 token 删除时的连接关闭

**文件**：`stream.go:67-80`

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

**流程**：
1. 获取写锁
2. 反向遍历用户的客户端列表（避免删除索引问题）
3. 匹配 token 相同的客户端
4. 调用 `client.Close()` 关闭连接
5. 从切片中移除
6. 更新映射
7. 释放锁

### 3. 服务关闭时的清理

**文件**：`stream.go:161-173`

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

**返回值**：此函数作为 `router.Create()` 的第二个返回值（清理函数）

---

## 链路时序图

```
客户端                    服务端
   |                         |
   |---- GET /stream ------>|
   |                         | 1. 认证中间件验证
   |                         | 2. WebSocket 升级
   |                         | 3. 创建 client 对象
   |                         | 4. 注册到 clients 映射
   |                         | 5. 启动 startReading
   |                         | 6. 启动 startWriteHandler
   |<---- 101 Switching -----|
   |     Protocols           |
   |                         |
   |                         |-- 定时 Ping -->|
   |<---- Ping --------------|
   |---- Pong -------------->|
   |                         |
   |                         | 消息到达
   |                         | Notify(userID, msg)
   |                         | 写入 write 通道
   |<---- Message -----------|
   |                         |
   |  (连接断开/超时/主动关闭)  |
   |                         |
   |                         | NotifyClose() 触发
   |                         | ├─ once.Do() 检查
   |                         | ├─ conn.Close()
   |                         | ├─ close(write)
   |                         | └─ onClose() 从映射移除
   |                         |
   |                         | startWriteHandler 退出
   |                         | └─ ticker.Stop()
   |                         |
   |                         | startReading 退出
   |                         |
   |<---- 连接关闭 ----------|
```

---

## 总结

### 关键设计原则

1. **一次性执行保证**：通过自定义 `once` 机制确保资源清理的原子性和幂等性
2. **goroutine 协作**：读取和写入 goroutine 通过 `defer` 和通道关闭协同退出
3. **资源追踪**：API 结构体维护所有客户端连接，支持批量操作
4. **异常安全**：所有可能的退出路径都通过 `defer` 确保清理执行
5. **并发安全**：使用读写锁保护客户端映射，使用互斥锁保护 `once` 状态

### 涉及的关键文件

- `api/stream/once.go` - 一次性执行机制
- `api/stream/client.go` - 单客户端连接管理
- `api/stream/stream.go` - 整体流 API 管理
- `router/router.go` - 路由注册与初始化
- `api/message.go` - 消息通知触发点

### 资源清理的核心保障

| 保障机制 | 实现位置 | 作用 |
|---------|---------|------|
| `once.Do()` | `client.go:44`/`52` | 确保清理只执行一次 |
| `defer` | `client.go:62`/`83` | 确保异常路径也能执行清理 |
| 通道关闭 | `client.go:46`/`54` | 通知写入 goroutine 退出 |
| 读写锁 | `stream.go:21` | 保护客户端映射的并发访问 |
| 原子操作 | `once.go:20`/`32` | 无锁快速路径检查 |

这种设计确保了无论连接以何种方式关闭，所有资源都能被正确、完整地清理，且不会发生重复释放的问题。
