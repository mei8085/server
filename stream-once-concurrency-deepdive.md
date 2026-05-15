# Stream 并发语义深挖报告

## 1. 核心问题概述

本报告深入分析 Gotify Stream 模块中**消息通知持锁发送**与**连接关闭流程**之间的并发交互，以及可能存在的**死锁风险**和**阻塞场景**。

## 2. 关键数据结构与锁关系

### 2.1 锁层次结构

```
API.lock (sync.RWMutex)
    ├── RLock(): Notify() 发送消息时持有
    ├── Lock(): register() 注册客户端时持有
    ├── Lock(): remove() 移除客户端时持有
    ├── Lock(): NotifyDeletedUser() 删除用户时持有
    └── Lock(): NotifyDeletedClient() 删除客户端时持有

client.once (sync.Mutex + atomic)
    └── Do(): Close() / NotifyClose() 时持有
```

### 2.2 Channel 配置

**关键发现**: write channel 容量为 1

```go
// client.go:35
write: make(chan *model.MessageExternal, 1),  // 容量 = 1
```

## 3. 消息通知持锁发送流程

### 3.1 Notify 执行路径

```go
// stream.go:83-91
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    a.lock.RLock()         // <-- 获取读锁
    defer a.lock.RUnlock() // <-- 函数退出时释放
    
    if clients, ok := a.clients[userID]; ok {
        for _, c := range clients {
            c.write <- msg  // <-- 持锁发送！
        }
    }
}
```

### 3.2 执行时序分析

```
时间线:
  T1: a.lock.RLock()          ✓ 获取读锁
  T2: c.write <- msg          尝试发送消息
    - 如果 channel 为空: 立即成功 ✓
    - 如果 channel 已满: 阻塞等待 ⚠️
  T3: a.lock.RUnlock()        只有发送完成后才释放
```

## 4. 连接关闭流程分析

### 4.1 NotifyClose 执行路径

```go
// client.go:51-57
func (c *client) NotifyClose() {
    c.once.Do(func() {
        c.conn.Close()      // 1. 关闭网络连接
        close(c.write)      // 2. 关闭 channel
        c.onClose(c)        // 3. 回调 API.remove
    })
}

// stream.go:93-104
func (a *API) remove(remove *client) {
    a.lock.Lock()           // <-- 需要获取写锁！
    defer a.lock.Unlock()
    // 从 clients map 中移除客户端
}
```

### 4.2 关闭触发场景

| 触发方式 | 调用链 | 锁需求 |
|---------|--------|--------|
| **读错误** | `startReading` → `defer NotifyClose()` → `remove()` | 需要写锁 |
| **写错误** | `startWriteHandler` → `defer NotifyClose()` → `remove()` | 需要写锁 |
| **用户删除** | `NotifyDeletedUser()` → `client.Close()` | 已持有写锁 |
| **客户端删除** | `NotifyDeletedClient()` → `client.Close()` | 已持有写锁 |
| **服务器关闭** | `Close()` → `client.Close()` | 已持有写锁 |

## 5. 临界阻塞场景深度分析

### 5.1 场景一：Channel 满导致读锁无法释放

**触发条件**:
1. 某个客户端消费缓慢，`write` channel 已满（容量1已占满）
2. 此时调用 `Notify()` 发送新消息

```
Goroutine A (消息发送)
    |
    |--> a.lock.RLock() ✓
    |
    |--> c.write <- msg  [阻塞等待]
    |     (因为 channel 已满)
    |
    [读锁被持有，无法释放]

Goroutine B (任意需要写锁的操作)
    |
    |--> a.lock.Lock() [阻塞等待]
    |     (读锁未释放，无法获取写锁)
    |
    [永久阻塞]
```

**风险等级**: ⚠️ **高** - 可能导致整个系统消息处理卡住

### 5.2 场景二：关闭回调死锁

**触发条件**:
1. `Notify()` 持读锁调用 `c.write <- msg`
2. 同时写协程出错，触发 `NotifyClose()`

```
Goroutine A (Notify)
    |
    T1: a.lock.RLock() ✓
    T2: c.write <- msg  [正在发送]
    |     (持有读锁)
    |
Goroutine B (写协程出错)
    |
    T3: 网络写入错误 → return
    T4: defer NotifyClose() 触发
    T5: c.once.Do() 执行
    T6: c.conn.Close() ✓
    T7: close(c.write) ✓
    T8: c.onClose(c) → 调用 a.remove()
    T9: a.lock.Lock() [阻塞！]
    |     (因为 Goroutine A 还持有读锁)
    |
结果: Goroutine B 阻塞在获取写锁
      Goroutine A 可能发现 channel 已关闭而 panic
```

**风险等级**: ⚠️ **中高** - 可能导致 panic 或 goroutine 泄漏

### 5.3 场景三：关闭操作的并发竞态

```
Goroutine A (NotifyDeletedUser)          Goroutine B (Notify)
    |                                       |
    |--> a.lock.Lock() ✓                   |--> a.lock.RLock() [等待]
    |                                       |
    |--> for _, c := range clients         |   (被 A 的写锁阻塞)
    |       client.Close()                 |
    |       (once 确保只执行一次)           |
    |                                       |
    |--> delete(a.clients, userID)         |
    |                                       |
    |--> a.lock.Unlock()                   |--> a.lock.RLock() ✓
                                            |
                                            |--> clients 已经为空
                                            |     不会发送任何消息
```

**风险等级**: ✅ **低** - 一旦写锁释放，读锁能正常获取，clients 已被清空

## 6. 死锁可能性形式化分析

### 6.1 锁获取顺序矩阵

| 操作 | 先获取 | 后获取 | 是否可能死锁 |
|------|--------|--------|-------------|
| Notify → NotifyClose | API.RLock | API.Lock | **是** - 读锁升级写锁 |
| Notify → Notify | API.RLock | API.RLock | 否 - 读锁可共享 |
| NotifyClose → Notify | API.Lock | API.RLock | 否 - 写锁释放后读锁才能获取 |
| NotifyClose → NotifyClose | API.Lock | API.Lock | 否 - 互斥 |

### 6.2 关键死锁路径

```
路径: Notify → channel 满阻塞 → NotifyClose 需要写锁

锁依赖图:
Notify (持有 RLock)
    ↓
需要 write channel 空位
    ↓
需要写协程消费
    ↓
写协程出错 → NotifyClose
    ↓
需要 API.Lock (写锁)
    ↓
被 Notify 的 RLock 阻塞 ←──────┘

结果: 循环等待 → 死锁
```

## 7. Once 机制的并发保护边界

### 7.1 once 保护的范围

```go
c.once.Do(func() {
    c.conn.Close()      // 受保护 ✓
    close(c.write)      // 受保护 ✓
    c.onClose(c)        // 受保护 ✓ - 但此回调会获取新锁！
})
```

### 7.2 once 的设计局限

**优点**:
- 确保 `Close()` 只执行一次
- 防止 `close(c.write)` 重复调用导致 panic
- 锁释放早，不阻塞外部调用者

**设计盲点**:
- 没有保护 `onClose` 回调可能引发的锁获取
- 回调执行时外部可能仍持有 API 锁

## 8. 代码优化建议

### 8.1 优化一：非阻塞发送消息

**问题**: 持锁发送可能阻塞导致锁无法释放

```go
// 优化前
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    a.lock.RLock()
    defer a.lock.RUnlock()
    if clients, ok := a.clients[userID]; ok {
        for _, c := range clients {
            c.write <- msg  // 可能阻塞！
        }
    }
}

// 优化后
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    a.lock.RLock()
    clients := a.clients[userID]
    a.lock.RUnlock()  // 提前释放锁！
    
    for _, c := range clients {
        select {
        case c.write <- msg:
            // 发送成功
        default:
            // channel 满，丢弃消息或异步处理
            log.Printf("client %d channel full, dropping message", c.userID)
        }
    }
}
```

### 8.2 优化二：异步移除客户端

**问题**: NotifyClose 回调同步获取写锁

```go
// 优化前
func (c *client) NotifyClose() {
    c.once.Do(func() {
        c.conn.Close()
        close(c.write)
        c.onClose(c)  // 同步调用，可能阻塞
    })
}

// 优化后
func (c *client) NotifyClose() {
    c.once.Do(func() {
        c.conn.Close()
        close(c.write)
        go c.onClose(c)  // 异步执行，不阻塞 once
    })
}
```

### 8.3 优化三：增加 Channel 容量

**问题**: 容量为 1 极易满

```go
// 优化前
write: make(chan *model.MessageExternal, 1),

// 优化后
write: make(chan *model.MessageExternal, 16),  // 更大容量
```

### 8.4 优化四：超时保护

```go
// 带超时的发送
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    a.lock.RLock()
    clients := make([]*client, 0)
    if cs, ok := a.clients[userID]; ok {
        clients = append(clients, cs...)
    }
    a.lock.RUnlock()
    
    for _, c := range clients {
        select {
        case c.write <- msg:
        case <-time.After(10 * time.Millisecond):
            log.Printf("client %d send timeout", c.userID)
        }
    }
}
```

## 9. 并发安全性验证清单

- [x] **双重检查锁定**: once 实现正确
- [x] **Channel 关闭保护**: once 防止重复关闭
- [ ] **持锁操作超时**: ⚠️ 缺少保护
- [ ] **锁持有时间最小化**: ⚠️ Notify 持锁发送
- [x] **读/写锁正确使用**: RWMutex 读写分离
- [ ] **锁获取顺序一致性**: ⚠️ Notify → NotifyClose 路径可能倒置
- [ ] **回调锁风险**: ⚠️ onClose 回调可能引发新锁

## 10. 总结

### 10.1 核心发现

1. **持锁发送风险**: `Notify()` 持有读锁时发送消息，若 channel 已满会导致读锁无法释放
2. **锁升级死锁**: 持读锁时触发关闭流程需要写锁，形成循环等待
3. **容量瓶颈**: write channel 容量仅为 1，极易满
4. **回调风险**: `onClose` 回调在 once 保护内执行，可能获取新锁

### 10.2 风险等级评估

| 风险项 | 概率 | 影响 | 等级 |
|--------|------|------|------|
| 持锁发送阻塞 | 中 | 高 | **高** |
| 关闭流程死锁 | 低 | 极高 | **高** |
| 消息丢失（channel 满） | 中 | 中 | **中** |
| goroutine 泄漏 | 低 | 低 | **低** |

### 10.3 建议优先级

1. **P0**: 重构 `Notify()`，先复制客户端列表，释放锁后再发送
2. **P1**: 增加 write channel 容量（建议 8-16）
3. **P1**: `NotifyClose()` 中异步执行 `onClose` 回调
4. **P2**: 发送消息增加超时机制
5. **P2**: 监控 channel 使用率，及时发现慢客户端

---

**结论**: 当前实现在高并发或客户端消费缓慢的场景下，存在**真实的死锁风险**。建议优先实施"释放锁再发送"的优化，这是成本最低但收益最大的改进。
