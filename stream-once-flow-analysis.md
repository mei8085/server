# 一次性 Stream 响应路径分析报告

## 1. 概述

本报告分析了 Gotify 服务器中 WebSocket Stream 模块的一次性（once）执行机制，重点关注 `once` 结构体在确保连接安全关闭中的作用。

## 2. 核心组件结构

### 2.1 once 结构体

**文件位置**: `api/stream/once.go`

`once` 是对标准库 `sync.Once` 的修改版本，主要区别在于：在执行目标函数之前就释放互斥锁，避免在函数执行期间持有锁。

```go
type once struct {
    m    sync.Mutex
    done uint32
}
```

**关键方法**:
- `Do(f func())`: 确保函数只执行一次
- `mayExecute() bool`: 原子性检查并标记执行状态

### 2.2 client 结构体

**文件位置**: `api/stream/client.go`

每个 WebSocket 连接都对应一个 client 实例，其中嵌入了 `once` 来确保关闭操作的原子性。

```go
type client struct {
    conn    *websocket.Conn
    onClose func(*client)
    write   chan *model.MessageExternal
    userID  uint
    token   string
    once    once  // 关键：确保关闭操作只执行一次
}
```

### 2.3 API 结构体

**文件位置**: `api/stream/stream.go`

管理所有活跃的 WebSocket 客户端连接。

```go
type API struct {
    clients     map[uint][]*client
    lock        sync.RWMutex
    pingPeriod  time.Duration
    pongTimeout time.Duration
    upgrader    *websocket.Upgrader
}
```

## 3. 一次性（Once）执行流程

### 3.1 并发安全关闭机制

#### 3.1.1 Close() 方法

```go
func (c *client) Close() {
    c.once.Do(func() {
        c.conn.Close()
        close(c.write)
    })
}
```

**流程**:
1. 原子检查 `done` 标记（使用 `atomic.LoadUint32`）
2. 如果未执行，获取互斥锁
3. 再次检查 `done` 标记（双重检查锁定模式）
4. 原子性设置 `done = 1`
5. **释放互斥锁**（与标准库 sync.Once 的关键区别）
6. 执行关闭操作：关闭连接和 channel

#### 3.1.2 NotifyClose() 方法

```go
func (c *client) NotifyClose() {
    c.once.Do(func() {
        c.conn.Close()
        close(c.write)
        c.onClose(c)  // 额外：通知 API 移除客户端
    })
}
```

### 3.2 once 执行时序图

```
   Goroutine A                 Goroutine B
       |                           |
       |  Do(f)                    |  Do(f)
       |                           |
       |--> atomic.Load(done)      |--> atomic.Load(done)
       |     == 0                  |     == 0
       |                           |
       |--> mayExecute()           |--> mayExecute()
       |     |                     |     |
       |     |--> Lock()           |     |--> Lock() (阻塞等待)
       |     |                     |
       |     |--> done = 1 (原子)  |
       |     |                     |
       |     |--> Unlock() <--------+---- (A 释放锁，B 获取锁)
       |     |                     |
       |     |<-- return true      |     |--> 检查 done == 1
       |                           |     |--> Unlock()
       |--> f()                    |     |<-- return false
       | (执行关闭逻辑)             |
       |                           | (不执行，直接返回)
       V                           V
```

### 3.3 与标准库 sync.Once 的区别

| 特性 | 本实现 once | 标准库 sync.Once |
|------|------------|----------------|
| **锁持有时间** | 仅在检查和设置 `done` 标记时持有 | 在整个 `f()` 执行期间都持有锁 |
| **并发性能** | 更好，f() 执行时不阻塞其他调用 | 可能导致其他调用者长时间等待 |
| **适用场景** | WebSocket 连接关闭等需要快速释放锁的场景 | 通用的一次性初始化 |

## 4. WebSocket Stream 完整生命周期

### 4.1 连接建立流程 (router.go:215)

```
客户端请求 GET /stream
        |
        V
    身份验证 (RequireClient)
        |
        V
    streamHandler.Handle(ctx)
        |
        |--> 升级 HTTP 为 WebSocket
        |--> 获取 userID 和 token
        |--> 创建 client 实例
        |--> API.register(client)
        |
        +--> 启动读协程: client.startReading()
        |
        +--> 启动写协程: client.startWriteHandler()
```

### 4.2 连接关闭触发点

client 连接可能在以下情况下被关闭：

#### 4.2.1 读协程中发生错误
```go
func (c *client) startReading(pongWait time.Duration) {
    defer c.NotifyClose()  // 确保退出时关闭
    // ... 读取循环
    if _, _, err := c.conn.NextReader(); err != nil {
        return  // 触发 defer NotifyClose()
    }
}
```

#### 4.2.2 写协程中发生错误
```go
func (c *client) startWriteHandler(pingPeriod time.Duration) {
    defer func() {
        c.NotifyClose()  // 确保退出时关闭
        pingTicker.Stop()
    }()
    // ... 写循环
    if err := writeJSON(...); err != nil {
        return  // 触发 defer NotifyClose()
    }
}
```

#### 4.2.3 用户/客户端删除时主动关闭
```go
// stream.go:54 - 用户删除
func (a *API) NotifyDeletedUser(userID uint) error {
    for _, client := range clients {
        client.Close()  // 调用 once 保护的 Close
    }
}

// stream.go:67 - 客户端删除
func (a *API) NotifyDeletedClient(userID uint, token string) {
    client.Close()  // 调用 once 保护的 Close
}
```

#### 4.2.4 服务器关闭时
```go
// stream.go:161
func (a *API) Close() {
    for _, clients := range a.clients {
        for _, client := range clients {
            client.Close()
        }
    }
}
```

## 5. 关键技术分析

### 5.1 双重检查锁定（Double-Checked Locking）

```go
func (o *once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 1 {  // 第一次检查（无锁）
        return
    }
    if o.mayExecute() {  // 第二次检查（有锁）
        f()
    }
}

func (o *once) mayExecute() bool {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        atomic.StoreUint32(&o.done, 1)
        return true
    }
    return false
}
```

**优点**:
- 快速路径：已执行的调用无需获取锁，直接返回
- 线程安全：互斥锁确保并发安全
- 性能优化：锁持有时间极短

### 5.2 避免死锁的设计

**关键改进**: 在 `f()` 执行前就释放互斥锁

**为什么重要**?
- 如果 `f()` 执行时间长（如网络 IO），会导致其他调用者长时间阻塞
- 如果 `f()` 内部尝试获取同一个锁，会导致死锁
- WebSocket 关闭操作可能涉及网络操作，耗时不确定

### 5.3 Channel 关闭安全

```go
c.once.Do(func() {
    c.conn.Close()
    close(c.write)  // channel 只关闭一次
})
```

**保护的问题**:
- Go 中关闭已关闭的 channel 会 panic
- 并发情况下，多个 goroutine 可能同时尝试关闭
- `once` 确保 `close(c.write)` 只执行一次

## 6. 测试验证

**文件位置**: `api/stream/once_test.go`

### 6.1 测试场景

1. **并发调用测试**: 两个 goroutine 同时调用 `Do()`，验证只执行一次
2. **状态持久化测试**: 执行后 `mayExecute()` 应返回 false
3. **后续调用测试**: 执行完成后再次调用，不重复执行

### 6.2 测试代码关键点

```go
go executeOnce.Do(fExecute)  // 并发调用1
go executeOnce.Do(fExecute)  // 并发调用2

// 验证只收到一次执行信号
select {
case <-execution:
    // expected
case <-time.After(100 * time.Millisecond):
    t.Fatal("fExecute should be executed once")
}

// 验证没有第二次执行
select {
case <-execution:
    t.Fatal("should only execute once")
case <-time.After(100 * time.Millisecond):
    // expected
}
```

## 7. 代码优化建议

### 7.1 现有实现的潜在问题

**问题**: `done` 字段使用 `uint32`，但在 32 位系统上 `atomic.StoreUint32` 可能有对齐问题（Go 1.17+ 已大部分解决）

**建议**: 可以考虑使用 `uint64` 或 `sync/atomic.Value`

### 7.2 增强错误处理

当前 `Close()` 方法忽略了 `conn.Close()` 的返回错误：

```go
// 建议：记录关闭错误
func (c *client) Close() {
    c.once.Do(func() {
        if err := c.conn.Close(); err != nil {
            fmt.Println("WebSocket close error:", err)
        }
        close(c.write)
    })
}
```

### 7.3 增加状态查询方法

```go
// 建议：添加方法检查是否已关闭
func (c *client) IsClosed() bool {
    return atomic.LoadUint32(&c.once.done) == 1
}
```

## 8. 总结

### 8.1 核心价值

1. **并发安全**: 确保 WebSocket 连接关闭操作的原子性
2. **防止 panic**: 避免重复关闭 channel 导致的 panic
3. **性能优化**: 锁持有时间短，不阻塞并发操作
4. **资源清理**: 确保连接资源被正确释放

### 8.2 设计亮点

- 对标准库 `sync.Once` 的改进，针对 WebSocket 场景优化
- 双重检查锁定模式的正确应用
- 与 defer 机制结合，确保资源清理
- 模块化设计，once 可复用

### 8.3 适用场景扩展

这种 once 模式适用于：
- 任何需要确保只执行一次且执行时间可能较长的操作
- 网络连接关闭
- 文件句柄关闭
- 资源清理操作
- 避免重复触发的回调函数
