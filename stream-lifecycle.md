# 长连推送通道生命周期分析报告

## 1. 概述

本报告基于代码和现有测试，分析长连推送通道（WebSocket）从握手建立到客户端断开的完整生命周期。所有结论均标注证据来源。

核心代码位置：
- `api/stream/stream.go` - WebSocket 连接管理器（API）
- `api/stream/client.go` - 单个客户端连接管理
- `api/stream/once.go` - 原子关闭工具
- `api/stream/stream_test.go` - 测试用例

## 2. 术语定义

为避免歧义，本文档使用以下术语：

| 术语 | 代码对应 | 说明 |
|------|---------|------|
| 注册表 | `API.clients` | `map[uint][]*client`，按 userID 分组存储活跃连接 |
| 消息通道 | `client.write` | `chan *model.MessageExternal`，容量为1 |
| `Close()` | `client.go:43` | 关闭方法一 |
| `NotifyClose()` | `client.go:51` | 关闭方法二 |
| 回调 | `client.onClose` | 创建客户端时传入的 `API.remove` 函数 |

## 3. 生命周期流程概览

```
客户端握手建立连接 (API.Handle)
    ↓
连接注册到注册表 (API.register)
    ↓
启动读协程 (client.startReading) + 写协程 (client.startWriteHandler)
    ↓
心跳维护 (ping/pong) + 消息派发 (API.Notify)
    ↓
连接关闭（三条路径之一）
    ↓
资源释放
```

## 4. 可证实事实：代码层面

本节所有结论均可通过阅读源代码直接验证。

### 4.1 once 的执行顺序

**代码证据** (`once.go:19-35`)：

```go
func (o *once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 1 {
        return  // 事实F1：done=1 时直接返回
    }
    if o.mayExecute() {
        f()  // 事实F3：mayExecute 返回 true 后才执行 f()
    }
}

func (o *once) mayExecute() bool {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        atomic.StoreUint32(&o.done, 1)  // 事实F2：在返回前设置 done=1
        return true
    }
    return false
}
```

**已证实事实**：

| ID | 事实 | 证据位置 |
|----|------|---------|
| F1 | `once.Do()` 开始时检查 `done == 1`，为 true 则直接返回，不执行任何操作 | `once.go:20-22` |
| F2 | `mayExecute()` 在返回 `true` 之前，先执行 `atomic.StoreUint32(&o.done, 1)` | `once.go:32` |
| F3 | `once.Do()` 仅在 `mayExecute()` 返回 `true` 后才执行 `f()` | `once.go:23-25` |

**推论（基于事实的逻辑推导）**：

| ID | 推论 | 推导依据 |
|----|------|---------|
| F4 | 第一个调用 `once.Do()` 的 goroutine 会：(1) 设置 `done=1`，(2) 执行 `f()` | F2 + F3 |
| F5 | 后续调用 `once.Do()` 的 goroutine 会看到 `done == 1`，直接返回，**不执行**传入的函数 | F1 |
| F6 | 如果 goroutine A 调用 `once.Do(f)`，goroutine B 调用 `once.Do(g)`，且 A 先进入 `mayExecute()`，则 `f()` 会被执行，`g()` 不会被执行 | F4 + F5 |

### 4.2 两种关闭方法的区别

**代码证据** (`client.go:43-57`)：

```go
// Close closes the connection.
func (c *client) Close() {
    c.once.Do(func() {
        c.conn.Close()      // 执行A1
        close(c.write)      // 执行A2
        // 注意：不调用 c.onClose(c)
    })
}

// NotifyClose closes the connection and notifies that the connection was closed.
func (c *client) NotifyClose() {
    c.once.Do(func() {
        c.conn.Close()      // 执行B1
        close(c.write)      // 执行B2
        c.onClose(c)        // 执行B3：调用回调
    })
}
```

**已证实事实**：

| ID | 事实 | 证据位置 |
|----|------|---------|
| F7 | `Close()` 传入的函数执行：`{ conn.Close(), close(write) }` | `client.go:44-47` |
| F8 | `Close()` 传入的函数**不包含** `c.onClose(c)` | `client.go:44-47`（代码中不存在这一行） |
| F9 | `NotifyClose()` 传入的函数执行：`{ conn.Close(), close(write), onClose(c) }` | `client.go:52-56` |
| F10 | `NotifyClose()` 传入的函数**包含** `c.onClose(c)` | `client.go:55` |

**对比表**：

| 操作 | `Close()` 执行 | `NotifyClose()` 执行 |
|------|----------------|---------------------|
| `conn.Close()` | ✓ | ✓ |
| `close(write)` | ✓ | ✓ |
| `onClose(c)` | ✗ | ✓ |

### 4.3 回调函数的作用

**代码证据** (`stream.go:93-104`)：

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

**已证实事实**：

| ID | 事实 | 证据位置 |
|----|------|---------|
| F11 | `API.remove()` 获取写锁 `a.lock.Lock()` | `stream.go:94` |
| F12 | `API.remove()` 从 `a.clients[userID]` 切片中移除匹配的 client | `stream.go:96-101` |
| F13 | 客户端创建时传入的 `onClose` 是 `a.remove` | `stream.go:154`: `newClient(..., a.remove)` |

**推论**：

| ID | 推论 | 推导依据 |
|----|------|---------|
| F14 | 调用 `onClose(c)` 等价于调用 `API.remove(c)`，即从注册表移除该客户端 | F10 + F12 + F13 |

### 4.4 协程的 defer 语句

**代码证据** (`client.go:61-86`)：

```go
func (c *client) startReading(pongWait time.Duration) {
    defer c.NotifyClose()  // 事实F15
    // ...
    for {
        if _, _, err := c.conn.NextReader(); err != nil {
            // ...
            return  // 函数返回时触发 defer
        }
    }
}

func (c *client) startWriteHandler(pingPeriod time.Duration) {
    defer func() {
        c.NotifyClose()  // 事实F16
        pingTicker.Stop()
    }()
    // ...
}
```

**已证实事实**：

| ID | 事实 | 证据位置 |
|----|------|---------|
| F15 | 读协程 `startReading` 使用 `defer c.NotifyClose()` | `client.go:62` |
| F16 | 写协程 `startWriteHandler` 使用 `defer c.NotifyClose()` | `client.go:84` |
| F17 | 读协程在 `NextReader()` 返回错误时 `return` | `client.go:70-73` |
| F18 | 写协程在 `writeJSON` 或 `ping` 返回错误时 `return` | `client.go:96-99`, `102-105` |
| F19 | 写协程在 `c.write` 通道关闭时（`ok == false`）`return` | `client.go:90-93` |

### 4.5 三条关闭路径的代码实现

#### 路径一：被动断开的触发点

**代码证据**：协程退出时触发 `defer NotifyClose()`（见 F15-F19）

#### 路径二：主动删除的实现

**代码证据** (`stream.go:54-80`)：

```go
func (a *API) NotifyDeletedUser(userID uint) error {
    a.lock.Lock()
    defer a.lock.Unlock()
    if clients, ok := a.clients[userID]; ok {
        for _, client := range clients {
            client.Close()  // 事实F20：调用 Close()
        }
        delete(a.clients, userID)  // 事实F21：手动从注册表删除
    }
    return nil
}

func (a *API) NotifyDeletedClient(userID uint, token string) {
    a.lock.Lock()
    defer a.lock.Unlock()
    if clients, ok := a.clients[userID]; ok {
        for i := len(clients) - 1; i >= 0; i-- {
            client := clients[i]
            if client.token == token {
                client.Close()  // 事实F22：调用 Close()
                clients = append(clients[:i], clients[i+1:]...)  // 事实F23：手动从注册表删除
            }
        }
        a.clients[userID] = clients
    }
}
```

**已证实事实**：

| ID | 事实 | 证据位置 |
|----|------|---------|
| F20 | `NotifyDeletedUser` 对匹配的客户端调用 `client.Close()` | `stream.go:59` |
| F21 | `NotifyDeletedUser` 调用 `delete(a.clients, userID)` 手动从注册表删除 | `stream.go:61` |
| F22 | `NotifyDeletedClient` 对匹配的客户端调用 `client.Close()` | `stream.go:74` |
| F23 | `NotifyDeletedClient` 通过切片操作手动从注册表删除 | `stream.go:75` |
| F24 | `NotifyDeletedUser` 和 `NotifyDeletedClient` 都在持有 `a.lock` 时执行上述操作 | `stream.go:55`, `68` |

#### 路径三：服务关闭的实现

**代码证据** (`stream.go:161-173`)：

```go
func (a *API) Close() {
    a.lock.Lock()
    defer a.lock.Unlock()

    for _, clients := range a.clients {
        for _, client := range clients {
            client.Close()  // 事实F25：调用 Close()
        }
    }
    for k := range a.clients {
        delete(a.clients, k)  // 事实F26：手动清空注册表
    }
}
```

**已证实事实**：

| ID | 事实 | 证据位置 |
|----|------|---------|
| F25 | `API.Close()` 对所有客户端调用 `client.Close()` | `stream.go:167` |
| F26 | `API.Close()` 调用 `delete(a.clients, k)` 手动清空注册表 | `stream.go:171` |
| F27 | `API.Close()` 在持有 `a.lock` 时执行上述操作 | `stream.go:162` |

### 4.6 锁的使用

**代码证据**：

| 函数 | 锁类型 | 操作 |
|------|--------|------|
| `API.register` | `Lock` | 写入 `clients` map |
| `API.remove` | `Lock` | 修改 `clients` map 切片 |
| `API.Notify` | `RLock` | 读取 `clients` map |
| `NotifyDeletedUser` | `Lock` | 读取 + 修改 `clients` map |
| `NotifyDeletedClient` | `Lock` | 读取 + 修改 `clients` map |
| `API.Close` | `Lock` | 读取 + 清空 `clients` map |

**已证实事实**：

| ID | 事实 | 证据位置 |
|----|------|---------|
| F28 | `API.Notify` 使用 `RLock`，写入 `c.write` 通道在 `RLock` 保护下执行 | `stream.go:84-91` |
| F29 | `API.remove`、`NotifyDeletedUser`、`NotifyDeletedClient`、`API.Close` 都使用 `Lock` | 各处代码 |
| F30 | `sync.RWMutex` 的 `RLock` 和 `Lock` 互斥（Go 标准库语义） | Go 语言规范 |

**推论**：

| ID | 推论 | 推导依据 |
|----|------|---------|
| F31 | `API.Notify` 执行期间，`API.remove`、`NotifyDeletedUser`、`NotifyDeletedClient`、`API.Close` 会被阻塞 | F28 + F29 + F30 |
| F32 | `close(c.write)` 不会与 `c.write <- msg` 并发执行（前者需要 `Lock`，后者在 `RLock` 下） | F7/F9 + F28 + F31 |

## 5. 可证实事实：测试层面

本节所有结论均可通过运行测试用例直接验证。

### 5.1 路径一验证：被动断开

**测试用例** (`stream_test.go:138-158`)：

```go
func TestCloseClientOnNotReading(t *testing.T) {
    server, api := bootTestServer(staticUserID())
    defer server.Close()
    defer api.Close()

    wsURL := wsURL(server.URL)
    ws, _, err := websocket.DefaultDialer.Dial(wsURL, nil)  // 建立连接
    assert.Nil(t, err)
    defer ws.Close()

    waitForConnectedClients(api, 1)
    assert.NotEmpty(t, clients(api, 1))  // 断言T1：连接已注册

    time.Sleep(api.pingPeriod + api.pongTimeout)  // 等待心跳超时

    assert.Empty(t, clients(api, 1))  // 断言T2：连接已从注册表移除
}
```

**测试证实的事实**：

| ID | 事实 | 验证方式 |
|----|------|---------|
| T1 | 连接建立后会被注册到注册表 | `assert.NotEmpty(t, clients(api, 1))` |
| T2 | 心跳超时后，客户端会从注册表移除 | `assert.Empty(t, clients(api, 1))` |

**结合代码事实的推导**：

心跳超时 → 读协程 `NextReader()` 返回错误 → `return` → `defer NotifyClose()` → 执行 `onClose(c)` → 调用 `API.remove()` → 从注册表移除

### 5.2 路径二验证：主动删除

**测试用例** (`stream_test.go:178-199`)：

```go
func TestDeleteClientShouldCloseConnection(t *testing.T) {
    server, api := bootTestServer(staticUserID())
    defer server.Close()
    defer api.Close()

    wsURL := wsURL(server.URL)
    user := testClient(t, wsURL)  // 建立连接
    defer user.conn.Close()

    waitForConnectedClients(api, 1)

    api.Notify(1, &model.MessageExternal{Message: "msg"})
    user.expectMessage(...)  // 断言T3：能收到消息

    api.NotifyDeletedClient(1, "customtoken")  // 主动删除

    api.Notify(1, &model.MessageExternal{Message: "msg"})
    user.expectNoMessage()  // 断言T4：不再收到消息
}
```

**测试证实的事实**：

| ID | 事实 | 验证方式 |
|----|------|---------|
| T3 | 正常连接时能收到消息 | `user.expectMessage(...)` |
| T4 | `NotifyDeletedClient` 后，客户端不再收到消息 | `user.expectNoMessage()` |

**结合代码事实的推导**：

`NotifyDeletedClient` → 调用 `Close()`（关闭连接和通道）→ 手动从注册表移除 → `API.Notify` 找不到该客户端 → 不再派发消息

### 5.3 多客户端场景验证

**测试用例** (`stream_test.go:201-262` - `TestDeleteMultipleClients`)：

验证了同一 token 多个连接会被全部关闭，不同 token 的连接不受影响。

## 6. 三条关闭路径的完整分析

### 6.1 统一分析框架

每条路径按以下格式分析：

```
触发条件 → 调用序列 → 状态变化 → 注册表变化
```

### 6.2 路径一：被动断开

**触发条件**（可证实事实 F15-F19）：
- 客户端主动关闭 WebSocket 连接
- 心跳超时（`NextReader` 返回超时错误）
- 网络中断导致读写错误

**调用序列**（基于事实 F15-F19、F9、F14）：

```
步骤1：协程检测到错误
  ├─ 读协程：NextReader() 返回错误 → return
  └─ 写协程：writeJSON/ping 返回错误 或 通道关闭 → return

步骤2：defer 触发
  └─ defer c.NotifyClose()

步骤3：NotifyClose 执行
  ├─ once.Do() 检查 done
  │   └─ 如果 done == 0 → 设置 done = 1（事实F2）
  ├─ 执行函数体（事实F9）：
  │   ├─ c.conn.Close()
  │   ├─ close(c.write)
  │   └─ c.onClose(c) → API.remove(c)（事实F14）
  └─ API.remove 执行（事实F11-F12）：
      ├─ a.lock.Lock()
      ├─ 从注册表移除
      └─ a.lock.Unlock()
```

**状态变化表**（基于事实 F9、F12）：

| 阶段 | done 标志 | 连接状态 | 消息通道状态 | 注册表状态 |
|------|----------|---------|-------------|-----------|
| 正常运行 | 0 | 打开 | 打开 | 客户端在注册表中 |
| 协程 return | 0 | 可能已断开 | 打开 | 客户端在注册表中 |
| NotifyClose 开始执行 | 0 → 1 | 打开 → 关闭 | 打开 → 关闭 | 客户端在注册表中 |
| onClose 执行完成 | 1 | 关闭 | 关闭 | 客户端已从注册表移除 |

**注册表变化**：由 `onClose` 回调自动移除（事实 F14）

---

### 6.3 路径二：主动删除

**触发条件**（可证实事实 F20-F24）：
- 调用 `NotifyDeletedUser(userID)`
- 调用 `NotifyDeletedClient(userID, token)`

**调用序列**（基于事实 F20-F24、F7、F15-F16）：

```
步骤1：调用者获取锁
  └─ a.lock.Lock()（事实F24）

步骤2：查找并关闭客户端
  ├─ 遍历注册表找到匹配的客户端
  ├─ 调用 client.Close()（事实F20、F22）
  │   ├─ once.Do() 检查 done
  │   │   └─ done == 0 → 设置 done = 1（事实F2）
  │   └─ 执行函数体（事实F7）：
  │       ├─ c.conn.Close()
  │       └─ close(c.write)
  │       注意：不执行 onClose（事实F8）
  └─ 手动从注册表移除（事实F21、F23）

步骤3：释放锁
  └─ a.lock.Unlock()

步骤4：协程检测到连接关闭（并发执行）
  ├─ 读协程：NextReader() 返回错误 → return
  ├─ 写协程：writeJSON 错误 或 通道关闭 → return
  └─ defer c.NotifyClose()（事实F15、F16）

步骤5：NotifyClose 检查 once
  └─ once.Do() 看到 done == 1 → 直接返回（事实F1）
     什么都不执行！
```

**状态变化表**（基于事实 F7、F21/F23、F1）：

| 阶段 | done 标志 | 连接状态 | 消息通道状态 | 注册表状态 |
|------|----------|---------|-------------|-----------|
| 正常运行 | 0 | 打开 | 打开 | 客户端在注册表中 |
| 获取锁 | 0 | 打开 | 打开 | 客户端在注册表中 |
| 执行 Close | 0 → 1 | 打开 → 关闭 | 打开 → 关闭 | 客户端在注册表中 |
| 手动移除 | 1 | 关闭 | 关闭 | 客户端已从注册表移除 |
| 释放锁 | 1 | 关闭 | 关闭 | 客户端已从注册表移除 |
| 协程 NotifyClose | 1（不变） | 关闭（不变） | 关闭（不变） | 已移除（不变） |

**注册表变化**：手动移除（事实 F21、F23），`onClose` 回调**不会**被调用（因为 `Close()` 不调用它，且后续 `NotifyClose()` 被 `once` 阻止）

---

### 6.4 路径三：服务关闭

**触发条件**（可证实事实 F25-F27）：
- 调用 `API.Close()`

**调用序列**（基于事实 F25-F27、F7、F15-F16）：

```
步骤1：调用者获取锁
  └─ a.lock.Lock()（事实F27）

步骤2：关闭所有客户端
  ├─ 遍历所有客户端
  ├─ 对每个调用 client.Close()（事实F25）
  │   └─ [同路径二步骤2] 设置 done=1，关闭连接和通道，不调用 onClose
  └─ 手动清空注册表（事实F26）

步骤3：释放锁
  └─ a.lock.Unlock()

步骤4：所有协程退出
  └─ [同路径二步骤4-5] 协程调用 NotifyClose，但 done=1，直接返回
```

**状态变化表**：与路径二相同，区别是所有客户端都被关闭。

**注册表变化**：手动清空（事实 F26）

---

### 6.5 三条路径对比

| 维度 | 路径一：被动断开 | 路径二：主动删除 | 路径三：服务关闭 |
|------|-----------------|-----------------|-----------------|
| **触发方式** | 协程检测到错误后 return | 外部调用 `NotifyDeletedUser/Client` | 外部调用 `API.Close` |
| **调用的关闭方法** | `NotifyClose()` | `Close()` | `Close()` |
| **onClose 是否执行** | 是（事实F9） | 否（事实F8） | 否（事实F8） |
| **注册表移除方式** | 回调自动移除（F14） | 手动移除（F21/F23） | 手动清空（F26） |
| **协程后续 NotifyClose** | (是它触发的) | done=1，直接返回（F1） | done=1，直接返回（F1） |
| **是否在锁内执行关闭** | 否 | 是（F24） | 是（F27） |

## 7. 无法从当前证据确认的内容

以下内容无法从现有代码和测试直接证实，需要更多证据（如设计文档、原作者说明、额外测试用例）：

| 编号 | 内容 | 说明 |
|------|------|------|
| U1 | 为什么需要两种关闭方法（`Close` vs `NotifyClose`） | 代码显示了区别，但没有注释说明设计意图 |
| U2 | 如果路径二使用 `NotifyClose` 会发生什么 | 没有测试用例验证此场景，理论分析可能出错 |
| U3 | 自定义 `once` 与标准 `sync.Once` 的行为差异在本项目中是否必要 | 代码使用了自定义版本，但没有测试验证标准版本会出问题 |
| U4 | 自定义 `once` "提前解锁避免死锁" 的具体场景 | 注释提到了这个目的，但没有测试或代码注释说明具体死锁场景 |
| U5 | 消息通道容量为 1 的设计原因 | 代码设置了容量 1，但没有说明原因 |

## 8. 总结

### 8.1 已证实的核心结论

1. **once 的行为**：第一个调用者在执行函数前设置 `done=1`，后续调用者直接返回（F1-F6）

2. **两种关闭方法的本质区别**：
   - `NotifyClose()`：关闭连接 + 关闭通道 + **调用回调**（从注册表移除）
   - `Close()`：关闭连接 + 关闭通道 + **不调用回调**

3. **三条路径的注册表变化机制**：
   - **路径一（被动断开）**：协程调用 `NotifyClose()` → 回调自动移除
   - **路径二/三（主动删除/服务关闭）**：调用 `Close()` → 手动移除 → 协程的 `NotifyClose()` 被 `once` 阻止

4. **并发安全**：
   - 派发消息时的 `RLock` 与修改注册表时的 `Lock` 互斥（F30-F32）
   - 不会发生"向已关闭通道写入"的 panic

### 8.2 三条路径的统一表述

每条关闭路径都遵循以下模式：

```
[触发] → [执行 Close 或 NotifyClose] → [关闭资源] → [注册表移除]
                                    ↑
                              once 确保只执行一次
```

区别仅在于：
- 谁触发关闭（协程 vs 外部调用）
- 注册表由谁移除（回调 vs 手动）
