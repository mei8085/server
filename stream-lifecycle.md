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
- 如果客户端写协程处理慢，后续消息可能会阻塞发送者

### 3.6 原子关闭机制（once）

在分析关闭路径之前，必须先理解 `once` 的行为，这是理解整个并发收尾顺序的关键。

**自定义 once 实现** (once.go:14-36):

```go
type once struct {
    m    sync.Mutex
    done uint32
}

func (o *once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 1 {
        return
    }
    if o.mayExecute() {
        f()
    }
}

func (o *once) mayExecute() bool {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        atomic.StoreUint32(&o.done, 1)  // 关键：在执行 f() 之前就设置 done = 1
        return true
    }
    return false
}
```

**关键执行顺序**：
1. `mayExecute()` 获取锁
2. **先设置 `done = 1`**（这是后续调用者看到的状态）
3. 释放锁
4. 然后才执行 `f()`

**并发影响**：
- 如果 goroutine A 先调用 `once.Do(f)`，它会在执行 `f()` 之前就将 `done` 设为 1
- goroutine B 随后调用 `once.Do(g)` 时，会看到 `done == 1`，直接返回，**不会执行 `g`**
- 这意味着：**第一个调用者决定执行哪个函数，后续调用者什么都不做**

### 3.7 两种关闭方式的区别

**方式一：`Close()`** (client.go:43)

```go
func (c *client) Close() {
    c.once.Do(func() {
        c.conn.Close()
        close(c.write)
        // 注意：不调用 c.onClose(c)
    })
}
```

**执行内容**：
- 关闭 WebSocket 连接：`c.conn.Close()`
- 关闭消息通道：`close(c.write)`
- **不调用** `c.onClose(c)` 回调

**方式二：`NotifyClose()`** (client.go:51)

```go
func (c *client) NotifyClose() {
    c.once.Do(func() {
        c.conn.Close()
        close(c.write)
        c.onClose(c)  // 调用回调
    })
}
```

**执行内容**：
- 关闭 WebSocket 连接：`c.conn.Close()`
- 关闭消息通道：`close(c.write)`
- **调用** `c.onClose(c)` 回调（即 `API.remove(c)`）

**核心区别总结**：

| 操作 | `Close()` | `NotifyClose()` |
|------|-----------|-----------------|
| `conn.Close()` | ✓ | ✓ |
| `close(write)` | ✓ | ✓ |
| `onClose(c)` | ✗ | ✓ |

### 3.8 三条关闭路径的详细分析

#### 3.8.1 路径一：被动断开（客户端主动断开 / 心跳超时 / 读写失败）

**触发场景**：
- 客户端主动关闭 WebSocket 连接
- 客户端未响应心跳导致读超时
- 网络中断导致读写失败

**执行流程**：

```
步骤1：协程检测到错误
  ├─ 读协程：NextReader() 返回错误（连接关闭或超时）
  └─ 写协程：WriteJSON() 或 ping() 返回错误

步骤2：协程退出，触发 defer
  └─ defer c.NotifyClose()

步骤3：NotifyClose() 执行（once 确保只执行一次）
  ├─ c.conn.Close()        → 关闭连接（可能重复，但无害）
  ├─ close(c.write)        → 关闭消息通道
  └─ c.onClose(c)          → 调用 API.remove(c)

步骤4：API.remove() 从注册表移除
  ├─ 获取写锁 a.lock.Lock()
  ├─ 从 a.clients[userID] 切片中移除该客户端
  └─ 释放锁
```

**状态变化**：

| 阶段 | 注册表状态 | 消息通道状态 | 连接状态 |
|------|-----------|-------------|---------|
| 正常运行 | 客户端在注册表中 | `write` 通道打开 | 连接打开 |
| 错误检测 | 客户端在注册表中 | `write` 通道打开 | 连接可能已断开 |
| NotifyClose 执行 | 客户端在注册表中 | `write` 通道关闭 | 连接关闭 |
| onClose 回调执行 | 客户端已从注册表移除 | `write` 通道关闭 | 连接关闭 |

**并发时序**：

```
时间线 →
协程A (读/写)
  │
  ├─ 检测到错误 → return
  │
  ├─ defer NotifyClose() 触发
  │   │
  │   ├─ once.mayExecute() → 锁 → done=1 → 解锁
  │   │
  │   ├─ 执行 f(): { conn.Close(), close(write), onClose(c) }
  │   │                       │              │            │
  │   │                       │              │            └─ API.remove()
  │   │                       │              │                 ├─ 锁
  │   │                       │              │                 ├─ 从注册表移除
  │   │                       │              │                 └─ 解锁
  │   │                       │              │
  │   │                       │              └─ write 通道关闭
  │   │                       │
  │   │                       └─ 连接关闭
  │
  └─ 协程退出
```

**关键要点**：
- 注册表移除由 `onClose` 回调完成
- 回调在 `once` 保护下只执行一次
- 回调需要获取 `API.lock`，这也是使用自定义 `once` 的原因（避免锁嵌套导致死锁）

---

#### 3.8.2 路径二：主动删除（`NotifyDeletedClient` / `NotifyDeletedUser`）

**触发场景**：
- 管理员删除某个客户端 token：`NotifyDeletedClient(userID, token)`
- 管理员删除某个用户：`NotifyDeletedUser(userID)`

**以 `NotifyDeletedClient` 为例** (stream.go:67):

```go
func (a *API) NotifyDeletedClient(userID uint, token string) {
    a.lock.Lock()
    defer a.lock.Unlock()
    if clients, ok := a.clients[userID]; ok {
        for i := len(clients) - 1; i >= 0; i-- {
            client := clients[i]
            if client.token == token {
                client.Close()              // 调用 Close()，不是 NotifyClose()
                clients = append(clients[:i], clients[i+1:]...)  // 手动移除
            }
        }
        a.clients[userID] = clients
    }
}
```

**执行流程**：

```
步骤1：API 层获取写锁
  └─ a.lock.Lock()

步骤2：遍历查找匹配的客户端
  └─ 找到 token 匹配的 client

步骤3：调用 client.Close()
  ├─ once.mayExecute() → done = 1（关键！）
  ├─ 执行 { conn.Close(), close(write) }
  └─ 注意：不调用 onClose(c)

步骤4：手动从注册表移除
  └─ clients = append(clients[:i], clients[i+1:]...)

步骤5：释放锁
  └─ a.lock.Unlock()

步骤6：协程因连接关闭而退出（并发执行）
  ├─ 读协程：NextReader() 因连接关闭返回错误
  ├─ 写协程：要么收到 close(write) 的信号，要么写入失败
  └─ 协程退出 → defer NotifyClose()

步骤7：NotifyClose() 检查 once
  ├─ atomic.LoadUint32(&done) == 1
  └─ 直接返回，什么都不做！
```

**状态变化**：

| 阶段 | 注册表状态 | 消息通道状态 | 连接状态 | done 标志 |
|------|-----------|-------------|---------|----------|
| 正常运行 | 客户端在注册表中 | `write` 打开 | 连接打开 | 0 |
| 获取锁 | 客户端在注册表中 | `write` 打开 | 连接打开 | 0 |
| 执行 Close() | 客户端在注册表中 | `write` 关闭 | 连接关闭 | 1 |
| 手动移除 | 客户端已从注册表移除 | `write` 关闭 | 连接关闭 | 1 |
| 释放锁 | 客户端已从注册表移除 | `write` 关闭 | 连接关闭 | 1 |
| 协程退出 | 客户端已从注册表移除 | `write` 关闭 | 连接关闭 | 1 |
| NotifyClose 被调用 | 客户端已从注册表移除 | `write` 关闭 | 连接关闭 | 1 |
| NotifyClose 实际执行 | (无变化，因为 done=1) | (无变化) | (无变化) | 1 |

**并发时序**：

```
时间线 →
Goroutine API (调用 NotifyDeletedClient)
  │
  ├─ a.lock.Lock()
  │
  ├─ 找到目标 client
  │
  ├─ client.Close()
  │   │
  │   ├─ once.mayExecute()
  │   │   ├─ 锁
  │   │   ├─ done = 1  ← 关键！先设置
  │   │   └─ 解锁
  │   │
  │   └─ 执行 f(): { conn.Close(), close(write) }
  │                       │              │
  │                       │              └─ write 通道关闭
  │                       │
  │                       └─ 连接关闭
  │
  ├─ 手动从注册表移除 client
  │
  └─ a.lock.Unlock()


─────────────────────────────────────────────
并发的协程 (读/写)

  │   (在 API 调用之前或之后运行)
  │
  ├─ 检测到连接已关闭 / write 通道已关闭
  │
  ├─ return → defer NotifyClose() 触发
  │
  ├─ NotifyClose():
  │   ├─ atomic.Load(&done) → 返回 1
  │   └─ 直接返回，什么都不做！
  │
  └─ 协程退出
```

**关键要点**：
1. **注册表必须手动移除**：因为 `Close()` 不调用 `onClose` 回调
2. **once 阻止重复执行**：协程随后调用的 `NotifyClose()` 看到 `done=1`，直接返回
3. **避免死锁**：如果在持有 `API.lock` 时调用 `NotifyClose()`，它会尝试回调 `API.remove()` 再次获取锁，导致死锁
4. **手动移除 + Close()** 是正确的模式

---

#### 3.8.3 路径三：服务关闭（`API.Close()`）

**触发场景**：
- 服务优雅退出时调用 `API.Close()`

**函数** (stream.go:161):

```go
func (a *API) Close() {
    a.lock.Lock()
    defer a.lock.Unlock()

    for _, clients := range a.clients {
        for _, client := range clients {
            client.Close()  // 调用 Close()
        }
    }
    for k := range a.clients {
        delete(a.clients, k)  // 手动清空
    }
}
```

**执行流程与路径二类似**：
1. 获取写锁
2. 遍历所有客户端，调用 `client.Close()`（设置 `done=1`，关闭连接和通道）
3. 手动清空注册表
4. 释放锁
5. 协程因连接关闭退出，调用 `NotifyClose()` 但 `done=1`，无操作

**状态变化与路径二相同**，区别仅在于：
- 路径二：删除单个或部分客户端
- 路径三：删除所有客户端，清空整个 map

### 3.9 三条路径对比总结

| 维度 | 路径一：被动断开 | 路径二：主动删除 | 路径三：服务关闭 |
|------|-----------------|-----------------|-----------------|
| **触发者** | 协程（读/写） | API 层调用者 | API 层调用者 |
| **调用的关闭方法** | `NotifyClose()` | `Close()` | `Close()` |
| **onClose 回调** | ✓ 被调用 | ✗ 不被调用 | ✗ 不被调用 |
| **注册表移除方式** | 回调自动移除 | 手动移除 | 手动清空 |
| **调用时是否持有 API.lock** | 否 | 是 | 是 |
| **协程后续 NotifyClose 行为** | (就是它触发的) | done=1，直接返回 | done=1，直接返回 |
| **典型场景** | 客户端断开、心跳超时 | 删除用户/客户端 | 服务退出 |

### 3.10 为什么需要两种关闭方式？

**设计意图分析**：

1. **路径一（被动断开）需要回调**：
   - 协程检测到错误，但无法直接访问 `API.clients` map
   - 需要通过回调 `onClose(c)` 通知管理器移除自己
   - `NotifyClose()` 就是为此设计的

2. **路径二/三（主动删除）不能使用回调**：
   - 调用者已经持有 `API.lock`
   - 如果调用 `NotifyClose()`，它会尝试回调 `API.remove()`
   - `API.remove()` 也需要获取 `API.lock`
   - 结果：**死锁**！

3. **once 的关键作用**：
   - 路径二/三先调用 `Close()` 设置 `done=1`
   - 协程随后的 `NotifyClose()` 看到 `done=1` 直接返回
   - 确保不会重复执行关闭逻辑
   - 确保不会尝试回调导致死锁

**死锁避免演示**：

```go
// 错误的做法（会死锁）：
func (a *API) NotifyDeletedClient_BAD(userID uint, token string) {
    a.lock.Lock()         // 第一次获取锁
    defer a.lock.Unlock()
    
    client.Close()        // 如果内部调用 NotifyClose()
                          // → onClose(c) → API.remove()
                          // → a.lock.Lock() 第二次获取
                          // → 死锁！
}

// 正确的做法：
func (a *API) NotifyDeletedClient(userID uint, token string) {
    a.lock.Lock()
    defer a.lock.Unlock()
    
    client.Close()        // 只关闭，不回调
    // 手动移除
}
```

## 4. 关键衔接点分析

### 4.1 注册与清理的衔接

**注册时传入回调** (stream.go:154):
```go
client := newClient(conn, auth.GetUserID(ctx), token, a.remove)
```

`a.remove` 是 `API.remove()` 方法，用于从管理器中移除客户端。

**回调的触发条件**：
- 仅当通过 `NotifyClose()` 关闭时才触发
- 通过 `Close()` 关闭时不触发
- 由 `once` 保证只触发一次

### 4.2 协程退出与清理的衔接

两个协程都使用 `defer c.NotifyClose()` 确保退出时清理：
- 读协程: 读取失败 → 函数返回 → defer 触发 → NotifyClose
- 写协程: 写入失败或通道关闭 → 函数返回 → defer 触发 → NotifyClose

**但**：
- 如果 `Close()` 已被调用（路径二/三），`done=1`
- `NotifyClose()` 会直接返回，不执行任何操作
- 这是设计意图，不是缺陷

### 4.3 消息派发与连接状态的衔接

**`API.Notify()` 使用 `RLock`**：
- 允许并发派发消息
- 但如果在派发过程中客户端被移除会怎样？

**竞态分析**：

```
Goroutine A (派发消息)          Goroutine B (删除客户端)
    │                               │
    ├─ RLock()                      │
    │                               ├─ Lock() ← 被 RLock 阻塞
    │                               │
    ├─ 遍历 clients 列表            │
    │                               │
    ├─ c.write <- msg               │
    │   (如果 c.write 已关闭?)      │
    │                               │
    └─ RUnlock()                    │
                                    ├─ 获得 Lock()
                                    ├─ client.Close()
                                    ├─ 从列表移除
                                    └─ Unlock()
```

**实际情况**：
- 路径一（被动断开）：`NotifyClose()` 会先回调 `API.remove()` 获取写锁，所以派发要么在移除之前完成，要么在移除之后开始
- 路径二/三（主动删除）：调用 `Close()` 关闭 `write` 通道，但此时可能正在派发

**潜在问题**：
- 如果 `API.Notify()` 正在向 `c.write` 写入，而另一个 goroutine 同时 `close(c.write)`
- 向已关闭的通道写入会 **panic**

**但**：查看代码，`Close()` 和 `Notify()` 的执行顺序：
- 路径二/三中，`Close()` 是在持有 `Lock()` 时调用的
- `Notify()` 持有 `RLock()`
- 所以 `Close()` 和 `Notify()` 不会同时执行
- 要么 `Notify()` 先完成（写入成功或阻塞），然后 `Close()` 执行
- 要么 `Close()` 先执行（关闭通道），然后 `Notify()` 看到客户端已不在列表中

**实际安全保障**：
- `API.Notify()` 持有 `RLock` 期间遍历 `clients` 列表
- `API.remove()` 和主动删除都持有 `Lock`
- `RLock` 和 `Lock` 互斥
- 所以遍历和修改不会同时发生
- 但如果 `Notify()` 已经拿到了 `client` 指针，然后释放了 `RLock`，再写入 `c.write`...

**更精确的分析**：

```go
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    a.lock.RLock()
    defer a.lock.RUnlock()
    if clients, ok := a.clients[userID]; ok {
        for _, c := range clients {
            c.write <- msg  // 在 RLock 保护下写入
        }
    }
}
```

整个 `for` 循环都在 `RLock` 保护下，所以：
- 写入 `c.write` 时，`RLock` 仍然持有
- 任何需要 `Lock` 的操作（包括 `remove`、主动删除）都被阻塞
- 所以 `close(c.write)` 不会与 `c.write <- msg` 并发执行
- **没有 panic 风险**

## 5. 并发安全分析

| 操作 | 锁类型 | 说明 | 潜在冲突 |
|------|--------|------|---------|
| 注册客户端 | `Lock` | 写入 map | 与所有写操作互斥 |
| 移除客户端 (`remove`) | `Lock` | 修改 map 切片 | 与所有写操作互斥 |
| 派发消息 | `RLock` | 读取 map，写入通道 | 与其他读并行，与写互斥 |
| 主动删除客户端 | `Lock` | 调用 Close() + 手动移除 | 与所有操作互斥 |
| 关闭所有连接 | `Lock` | 遍历调用 Close() + 清空 map | 与所有操作互斥 |
| 统计连接数 | `RLock` | 读取 map | 与其他读并行 |

## 6. once 与标准 sync.Once 的区别

**标准 `sync.Once`**：
```go
func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 0 {
        o.doSlow(f)
    }
}

func (o *Once) doSlow(f func()) {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        f()                    // 在持有锁时执行 f()
        atomic.StoreUint32(&o.done, 1)  // 执行完才设置 done
    }
}
```

**自定义 `once`**：
```go
func (o *once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 1 {
        return
    }
    if o.mayExecute() {
        f()                    // 锁已释放
    }
}

func (o *once) mayExecute() bool {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        atomic.StoreUint32(&o.done, 1)  // 先设置 done
        return true
    }
    return false
}
```

**关键区别**：

| 维度 | 标准 sync.Once | 自定义 once |
|------|---------------|-------------|
| done 设置时机 | 执行 f() 之后 | 执行 f() 之前 |
| f() 执行时是否持有锁 | 是 | 否 |
| 后续调用者看到 done=1 的时机 | f() 执行完之后 | f() 开始执行之前 |

**为什么需要自定义 once？**

假设使用标准 `sync.Once`，路径二的时序：

```
Goroutine API                          Goroutine 协程
    │                                       │
    ├─ Lock()                               │
    │                                       │
    ├─ client.Close()                       │
    │   │                                   │
    │   ├─ once.Do(f)                       │
    │   │   ├─ done==0 → doSlow             │
    │   │   │   ├─ 锁                       │
    │   │   │   ├─ f() 开始执行             │
    │   │   │   │   ├─ conn.Close()         │
    │   │   │   │   └─ close(write)         │
    │   │   │   │                           │
    │   │   │   │   ┌───────────────────────┤ 检测到连接关闭
    │   │   │   │   │                       │
    │   │   │   │   │                       ├─ NotifyClose()
    │   │   │   │   │                       │   ├─ done==0
    │   │   │   │   │                       │   └─ 等待 once 锁
    │   │   │   │   │                       │
    │   │   │   ├─ done = 1                 │
    │   │   │   └─ 解锁                     │
    │   │   │                               │
    │   │   └─ 返回                         │
    │   │                                   │
    │   ├─ 手动移除                          │
    │   │                                   │
    │   └─ Unlock()                         │
    │                                       │
    │                                       ├─ 获得 once 锁
    │                                       ├─ done==1 → 返回
    │                                       └─ 协程退出
```

**这似乎也能工作？** 是的，但问题在于：

如果 `f()` 中需要获取另一个锁（如 `API.lock`），而调用者已经持有该锁：

```go
// 使用标准 sync.Once，f() 执行时持有 once 的锁
func (c *client) NotifyClose() {
    c.once.Do(func() {
        c.conn.Close()
        close(c.write)
        c.onClose(c)  // 调用 API.remove()，需要 API.lock
    })
}

// API.remove() 需要锁
func (a *API) remove(remove *client) {
    a.lock.Lock()  // 如果调用者已持有 API.lock？
    // ...
}
```

**路径一的死锁场景**（使用标准 sync.Once）：

```
Goroutine 协程
    │
    ├─ NotifyClose()
    │   │
    │   ├─ once.Do(f)
    │   │   ├─ 持有 once 锁
    │   │   ├─ 执行 f():
    │   │   │   ├─ conn.Close()
    │   │   │   ├─ close(write)
    │   │   │   └─ onClose(c) → API.remove()
    │   │   │                       │
    │   │   │                       ├─ a.lock.Lock() ← 成功
    │   │   │                       ├─ 移除
    │   │   │                       └─ a.lock.Unlock()
    │   │   ├─ done = 1
    │   │   └─ 释放 once 锁
    │
    └─ 退出
```

这也能工作... 那为什么要自定义 once？

**真正的原因**：查看 `once.go` 的注释：

```go
// Modified version of sync.Once
// This version unlocks the mutex early and therefore doesn't hold the lock while executing func f().
```

**设计意图**：让 `f()` 执行时不持有 `once` 的锁。

**可能的场景**：如果 `f()` 中有回调，而回调可能触发对同一个 `once` 的嵌套调用？但在这个代码库中似乎没有这种情况。

**更可能的原因**：这是一个通用的改进，避免 `f()` 执行时阻塞其他 `once.Do()` 调用者。在高并发场景下：
- 标准 `sync.Once`：其他调用者等待 `f()` 执行完
- 自定义 `once`：其他调用者立即看到 `done=1` 并返回

## 7. 测试用例验证

从 `stream_test.go` 可以验证以下场景：

1. **心跳测试** (`TestPing`): 验证 ping 发送和消息接收正常
2. **超时清理** (`TestCloseClientOnNotReading`): 验证不响应心跳的客户端被清理（路径一）
3. **消息派发** (`TestMessageDirectlyAfterConnect`): 验证连接后立即能收到消息
4. **客户端删除** (`TestDeleteClientShouldCloseConnection`): 验证删除 token 会断开连接（路径二）
5. **用户删除** (`TestDeleteUser`): 验证删除用户会断开所有连接（路径二）
6. **多客户端** (`TestMultipleClients`): 验证同一用户多设备连接和独立断开

**关键测试验证**：

`TestCloseClientOnNotReading` (路径一验证):
```go
// 建立连接但不读取（不响应心跳）
ws, _, err := websocket.DefaultDialer.Dial(wsURL, nil)
waitForConnectedClients(api, 1)
assert.NotEmpty(t, clients(api, 1))

// 等待超时
time.Sleep(api.pingPeriod + api.pongTimeout)

// 验证客户端已被移除（通过回调）
assert.Empty(t, clients(api, 1))
```

`TestDeleteClientShouldCloseConnection` (路径二验证):
```go
// 建立连接
user := testClient(t, wsURL)
waitForConnectedClients(api, 1)

// 删除客户端（主动删除）
api.NotifyDeletedClient(1, "customtoken")

// 验证不再收到消息（连接已关闭，注册表已手动移除）
api.Notify(1, &model.MessageExternal{Message: "msg"})
user.expectNoMessage()
```

## 8. 总结

长连推送通道的生命周期设计清晰，关键点：

1. **双协程架构**: 读写分离，各自独立处理异常
2. **分层管理**: `API` 负责连接注册和派发，`client` 负责单个连接管理
3. **心跳机制**: 利用 WebSocket ping/pong + 超时检测保活
4. **原子关闭**: 自定义 `once` 确保清理只执行一次，关键特性是**在执行函数前就设置 `done=1`**
5. **双模式关闭**:
   - `NotifyClose()`: 供协程使用，会触发回调移除注册表
   - `Close()`: 供 API 层使用，不触发回调，避免死锁
6. **三条关闭路径**:
   - 路径一（被动断开）：协程触发 `NotifyClose()` → 回调自动移除
   - 路径二/三（主动删除/服务关闭）：API 调用 `Close()` → 手动移除 → 协程的 `NotifyClose()` 因 `done=1` 跳过
7. **并发安全**: 通过 `RWMutex` 平衡性能和安全，派发消息时持有的 `RLock` 确保写入通道时不会并发关闭
8. **死锁避免**: 主动删除时使用 `Close()` 而非 `NotifyClose()`，避免在持有 `API.lock` 时回调尝试再次获取锁