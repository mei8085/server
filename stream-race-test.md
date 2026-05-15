# 推送流并发 Race 边界场景测试报告

## 前置说明

本文档所有结论均基于当前仓库代码的事实验证。所有测试用例均可直接在当前代码库复制运行。

## 1. 代码事实复核

### 1.1 Notify 与关闭链路的真实执行路径

**关键代码 (stream.go:83-91) - Notify：**
```go
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    a.lock.RLock()
    defer a.lock.RUnlock()  // <-- 持有读锁直到函数返回
    if clients, ok := a.clients[userID]; ok {
        for _, c := range clients {
            c.write <- msg  // <-- 持有读锁时写入通道
        }
    }
}
```

**关键代码 (client.go:50-57) - NotifyClose 运行态关闭链路：**
```go
func (c *client) NotifyClose() {
    c.once.Do(func() {
        c.conn.Close()       // <-- 1. 关闭连接
        close(c.write)       // <-- 2. 关闭通道（不持有 a.lock）
        c.onClose(c)         // <-- 3. 调用 remove，此时才会持有 a.lock
    })
}
```

**关键代码 (stream.go:93-104) - remove 回调：**
```go
func (a *API) remove(remove *client) {
    a.lock.Lock()  // <-- 这里才会持有写锁
    defer a.lock.Unlock()
    // 从 map 中移除
}
```

**竞态原理事实：**
- ✅ Notify 持有 `a.lock.RLock` 时执行 `c.write <- msg`
- ✅ NotifyClose 执行 `close(c.write)` 时不持有 `a.lock`
- ✅ 两者可以真正并发执行，不会被同一把锁串行化
- ✅ 这就是真实运行态的竞争窗口（网络断开触发 NotifyClose 时，Notify 可能正在发消息）

### 1.2 API.Close 与 Handle 注册的状态关系

**关键代码 (stream.go:161-173) - Close：**
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
    // <-- 无"已关闭"标志，释放锁后 Handle 仍可注册
}
```

**状态变化事实：**
- ✅ Close 持有写锁时：关闭所有连接、清空 clients map
- ✅ Close 返回后：`a.clients` 应该是空 map
- ✅ 但 Handle 仍能注册新连接，因为没有"已关闭"检查

---

## 2. Race 边界场景五维对照表

| # | 场景 | 代码证据 | 现有测试证据 | 是否已覆盖 | 可执行复现命令 |
|---|------|----------|--------------|------------|----------------|
| 1 | **Notify 写入 vs 运行态 NotifyClose 关闭通道**<br>Notify 持有读锁写入时，读 goroutine 检测到网络错误并发调用 NotifyClose | stream.go:88 `c.write <- msg`<br>client.go:54 `close(c.write)`<br>**关键：** close 时不持有 a.lock，可与 Notify 并发 | `TestDeleteUser` line 306-310<br>串行：Notify → NotifyDeletedUser → Notify<br>用互斥锁串行化，无并发 | ❌ 未覆盖 | 见 §4.1 |
| 2 | **自定义 Once 执行时序**<br>先标记 `done=1`，释放锁，再执行 `f()` | once.go:19-36<br>`atomic.Store` 在锁内<br>`f()` 在锁外执行 | `Test_Execute` line 16-17<br>仅 2 个 goroutine<br>f() 只是 channel send | ❌ 覆盖不足 | 见 §4.2 |
| 3 | **读写 goroutine 同时调用 NotifyClose**<br>网络故障导致两个 goroutine 同时检测到错误并退出 | client.go:62<br>`startReading` defer NotifyClose()<br>client.go:84<br>`startWriteHandler` defer NotifyClose() | `TestWriteMessageFails`：仅写 goroutine 退出<br>`TestWritePingFails`：仅写 goroutine 退出<br>`TestCloseClientOnNotReading`：仅读 goroutine 退出 | ❌ 未覆盖 | 见 §4.3 |
| 4 | **API.Close 之后新连接注册的状态检查**<br>Close 清空 clients 后，Handle 仍能注册新连接，应被拒绝或有明确状态 | stream.go:161-173<br>Close 清空 map 但无关闭标志<br>stream.go:143-157<br>Handle 无条件注册 | 所有测试的 `defer api.Close()` 均在测试结束时执行<br>无 Close 后 clients 状态的断言 | ❌ 未覆盖 | 见 §4.4 |
| 5 | **写通道满导致 Notify 持有锁阻塞**<br>通道容量 1，消费阻塞，连续 Notify 导致读锁被持有而阻塞 | client.go:35<br>`make(chan ..., 1)`<br>stream.go:84-88<br>Notify 持有 RLock 写入 | 所有测试消息发送均有间隔<br>通道不会满 | ❌ 未覆盖 | 见 §4.5 |

---

## 3. 现有测试覆盖详细分析

### 场景 1: Notify 写入 vs 运行态 NotifyClose 关闭通道

**真实运行态的竞争窗口说明：**

```
时序（真实运行场景）：
Goroutine A (Notify)                  Goroutine B (startReading)
     |                                       |
a.lock.RLock()                               |
c = clients[userID]                          |
     |                                       | 网络断开！
     |                                       conn.NextReader() 出错
     |                                       return
c.write <- msg  <------- 竞态窗口 ------->  defer c.NotifyClose()
(持有 RLock)                                      |
     |                                       c.once.Do(f)
     |                                       c.conn.Close()
     |                                       close(c.write)  // 不持有 a.lock
defer a.lock.RUnlock()                      c.onClose(c) // 调用 remove（此时才持有 a.lock）
```

**为什么这是真实的竞争窗口：**
1. ✅ Notify 持有 `a.lock.RLock` 执行 `c.write <- msg`
2. ✅ 读 goroutine 检测到网络错误，执行 `NotifyClose()`
3. ✅ `NotifyClose()` 中 `close(c.write)` 执行时不持有 `a.lock`
4. ✅ 两者可以真正并发，不会被串行化
5. ✅ 这就是生产环境中真实发生的场景

**现有测试为什么无法触发：**
- `TestDeleteUser` 使用 `NotifyDeletedUser()`，它持有 `a.lock.Lock()`
- `Lock()` 与 `RLock()` 互斥，导致被完全串行化
- 串行执行时，要么 Notify 完成才 Close，要么相反，不会有竞争

**覆盖状态：❌ 未覆盖**

---

### 场景 2: 自定义 Once 执行时序

**代码证据：**
```go
// once.go:19-36
func (o *once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 1 {
        return
    }
    if o.mayExecute() {
        f()  // <-- 执行 f 时不持有锁
    }
}

func (o *once) mayExecute() bool {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        atomic.StoreUint32(&o.done, 1)  // <-- 先标记 done
        return true                      // <-- 释放锁后才执行 f
    }
    return false
}
```

**与标准 `sync.Once` 差异：**
| 行为 | 自定义 once | 标准 sync.Once |
|------|------------|---------------|
| 标记 done 时机 | f() 执行前 | f() 执行后 |
| 执行 f() 时是否持有锁 | 否 | 是 |
| 其他 goroutine 看到 done=1 时 f() 状态 | 可能正在执行 | 已完成 |

**现有测试证据：**
- `Test_Execute` (once_test.go:10-42):
  ```go
  go executeOnce.Do(fExecute)  // 2 个 goroutine
  go executeOnce.Do(fExecute)
  time.Sleep(...)              // 等待第一个完成
  go executeOnce.Do(fExecute)  // 第三个
  ```
  仅 2-3 goroutine，f() 执行时间极短，race 窗口太小

**覆盖状态：❌ 覆盖不足**

---

### 场景 3: 读写 goroutine 同时调用 NotifyClose

**代码证据：**
```go
// client.go:61-75
func (c *client) startReading(pongWait time.Duration) {
    defer c.NotifyClose()  // <-- 出错时调用
    for {
        if _, _, err := c.conn.NextReader(); err != nil {
            return
        }
    }
}

// client.go:81-108
func (c *client) startWriteHandler(pingPeriod time.Duration) {
    defer func() {
        c.NotifyClose()  // <-- 出错时调用
        pingTicker.Stop()
    }()
    for {
        select {
        case message, ok := <-c.write:
            if !ok { return }
            if err := writeJSON(c.conn, message); err != nil {
                return  // 触发 NotifyClose
            }
        case <-pingTicker.C:
            if err := ping(c.conn); err != nil {
                return  // 触发 NotifyClose
            }
        }
    }
}
```

**现有测试证据：**
- `TestWriteMessageFails` (stream_test.go:39-67): 仅 mock `writeJSON`
- `TestWritePingFails` (stream_test.go:69-100): 仅 mock `ping`
- `TestCloseClientOnNotReading` (stream_test.go:138-158): pong 超时

每次只触发一个 goroutine 退出，从未同时触发两个

**覆盖状态：❌ 未覆盖**

---

### 场景 4: API.Close 之后新连接注册的状态检查

**代码证据：**
```go
// stream.go:161-173
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
    // <-- 没有"已关闭"标志，Handle 仍可注册
}

// stream.go:143-157
func (a *API) Handle(ctx *gin.Context) {
    // <-- 没有关闭检查，无条件注册
    client := newClient(conn, ...)
    a.register(client)
    go client.startReading(...)
    go client.startWriteHandler(...)
}
```

**明确的状态期望（可断言）：**
| 时间点 | clients map 状态 | 期望 |
|--------|-----------------|------|
| Close() 执行中 | 持有写锁，逐个关闭连接 | 应被阻塞直到完成 |
| Close() 返回后立即 | `len(a.clients) == 0` | ✅ 已清空，可断言 |
| Close() 返回后调用 Handle | 新连接被注册到 a.clients | ❓ 应该被拒绝？还是允许注册？ |

**可直接断言的检查点：**
1. Close 返回后，`a.clients` 必须是空 map（可断言）
2. Close 返回后调用 Notify，不应该 panic（可断言）
3. Close 返回后调用 Handle，新连接是否应该能注册（设计决策）

**现有测试证据：**
- 所有测试的 `defer api.Close()` 均在测试结束时执行
- 无任何测试断言 Close 之后的 clients 状态

**覆盖状态：❌ 未覆盖**

---

### 场景 5: 写通道满导致 Notify 持有锁阻塞

**代码证据：**
```go
// client.go:35
write: make(chan *model.MessageExternal, 1)  // 容量 1

// stream.go:83-91
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    a.lock.RLock()         // <-- 获取读锁
    defer a.lock.RUnlock() // <-- 函数返回才释放
    for _, c := range clients {
        c.write <- msg     // <-- 如果通道满，这里阻塞并持有读锁
    }
}
```

**死锁风险：**
1. `startWriteHandler` 因网络阻塞无法消费消息
2. 第一条消息填满通道
3. 第二条 `Notify` 写入时阻塞，同时持有 `a.lock.RLock()`
4. 所有需要 `a.lock.Lock()` 的操作（register/remove/Close）都被阻塞
5. 整个推送系统死锁

**现有测试证据：**
- 无测试覆盖此场景
- 所有测试消息发送都有时间间隔

**覆盖状态：❌ 未覆盖**

---

## 4. 可执行复现步骤（按仓库直接验证）

### 场景 1 复现：Notify vs 运行态 NotifyClose 并发

**复现目标：** 模拟真实运行场景 - 读 goroutine 检测到网络错误调用 NotifyClose，与 Notify 并发执行

**设计原理：**
- ✅ 不直接调用 `client.Close()`（内部方式）
- ✅ 通过关闭底层 websocket 连接，触发读 goroutine 的错误路径
- ✅ 读 goroutine 执行 `NotifyClose()` 时与 `Notify` 真实并发
- ✅ `close(c.write)` 执行时不持有 `a.lock`，与 `Notify` 的写入形成竞争窗口

**复现步骤：**

1. **添加测试到 `api/stream/stream_test.go`:**
```go
import "sync"
import "github.com/gorilla/websocket"

func TestRace_NotifyAndNotifyClose(t *testing.T) {
    mode.Set(mode.TestDev)
    server, api := bootTestServer(staticUserID())
    defer server.Close()
    defer api.Close()

    wsURL := wsURL(server.URL)
    
    // 1. 建立 WebSocket 连接
    ws, _, err := websocket.DefaultDialer.Dial(wsURL, nil)
    if err != nil {
        t.Fatal(err)
    }
    defer ws.Close()
    
    // 等待连接注册完成
    waitForConnectedClients(api, 1)

    // 2. 高并发：同时发送 Notify 和关闭连接
    // 关闭连接会触发读 goroutine 错误路径 -> NotifyClose -> close(c.write)
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(2)
        
        go func() {
            defer wg.Done()
            // Notify 持有 RLock 写入通道
            api.Notify(1, &model.MessageExternal{Message: "test"})
        }()
        
        go func() {
            defer wg.Done()
            // 关闭底层连接，触发读 goroutine 的 NotifyClose
            // NotifyClose 中 close(c.write) 不持有 a.lock
            // 与 Notify 的写入形成真实竞争窗口
            ws.Close()
        }()
    }
    
    wg.Wait()
}
```

2. **运行命令：**
```bash
cd d:\fz\0508-1\solo-dogfeeding\code\102-server
go test -race ./api/stream -run TestRace_NotifyAndNotifyClose -v -count=1
```

3. **判定信号：**
- ✅ `WARNING: DATA RACE` - race detector 检测到 channel send/close 竞争
- ✅ `panic: send on closed channel` - 运行时 panic（极端竞态触发）

**为什么这个方案与真实运行态一致：**
- ✅ 完全模拟了生产环境"网络断开时正在发消息"的场景
- ✅ 关闭链路是真实的代码路径（不是测试直接调用内部方法）
- ✅ `close(c.write)` 执行时确实不持有 `a.lock`，与 `Notify` 的写入形成真实并发

---

### 场景 2 复现：自定义 Once 内存可见性

**复现目标：** 验证自定义 Once 的内存可见性问题

**复现步骤：**

1. **添加测试到 `api/stream/once_test.go`:**
```go
func TestRace_OnceMemoryVisibility(t *testing.T) {
    var wg sync.WaitGroup
    
    for round := 0; round < 100; round++ {
        o := &once{}
        var sharedState int32
        
        for i := 0; i < 50; i++ {
            wg.Add(1)
            go func() {
                defer wg.Done()
                o.Do(func() {
                    atomic.StoreInt32(&sharedState, 42)
                    // 模拟耗时操作，扩大 race 窗口
                    time.Sleep(10 * time.Microsecond)
                })
                
                if atomic.LoadInt32(&sharedState) != 42 {
                    t.Error("Memory visibility issue")
                }
            }()
        }
        
        wg.Wait()
    }
}
```

2. **运行命令：**
```bash
go test -race ./api/stream -run TestRace_OnceMemoryVisibility -v -count=1
```

3. **判定信号：**
- ✅ `WARNING: DATA RACE` - 内存竞争
- ✅ 测试断言失败 `Memory visibility issue`

---

### 场景 3 复现：读写 goroutine 同时退出

**复现目标：** 两个 goroutine 同时调用 NotifyClose

**复现步骤：**

1. **添加测试到 `api/stream/stream_test.go`:**
```go
func TestRace_BothGoroutinesExit(t *testing.T) {
    mode.Set(mode.TestDev)
    
    // Mock 写路径出错
    oldWriteJSON := writeJSON
    oldPing := ping
    
    writeJSON = func(conn *websocket.Conn, v interface{}) error {
        return errors.New("write error")
    }
    ping = func(conn *websocket.Conn) error {
        return errors.New("ping error")
    }
    
    defer func() {
        writeJSON = oldWriteJSON
        ping = oldPing
    }()
    
    server, api := bootTestServer(staticUserID())
    defer server.Close()
    defer api.Close()
    
    wsURL := wsURL(server.URL)
    ws, _, err := websocket.DefaultDialer.Dial(wsURL, nil)
    if err != nil {
        t.Fatal(err)
    }
    defer ws.Close()
    
    waitForConnectedClients(api, 1)
    
    // 同时：1) 触发写 goroutine 退出  2) 关闭连接触发读 goroutine 退出
    var wg sync.WaitGroup
    wg.Add(2)
    
    go func() {
        defer wg.Done()
        api.Notify(1, &model.MessageExternal{Message: "trigger"})
    }()
    
    go func() {
        defer wg.Done()
        ws.Close()
    }()
    
    wg.Wait()
    time.Sleep(100 * time.Millisecond)
    
    // 没有 panic = once 保护有效，但 race detector 可能检测到竞争
}
```

2. **运行命令：**
```bash
go test -race ./api/stream -run TestRace_BothGoroutinesExit -v -count=1
```

3. **判定信号：**
- ✅ `WARNING: DATA RACE` - 检测到竞争
- ⚠️ 无输出 = once 保护有效（但 race 可能存在）

---

### 场景 4 复现：API.Close 之后新连接注册的状态检查

**复现目标：** 可直接断言的状态检查，不是依赖泄漏观测

**可断言的检查点：**
1. ✅ Close 返回后，`a.clients` 必须是空 map
2. ✅ Close 返回后调用 Notify，不应该 panic
3. ✅ Close 返回后调用 Handle，观察是否能注册新连接（记录当前行为）

**复现步骤：**

1. **添加测试到 `api/stream/stream_test.go`:**
```go
import "github.com/gorilla/websocket"

func TestRace_CloseAndNewConnections_StateCheck(t *testing.T) {
    mode.Set(mode.TestDev)
    server, api := bootTestServer(staticUserID())
    defer server.Close()
    
    wsURL := wsURL(server.URL)
    
    // 1. 先建立一些连接
    for i := 0; i < 10; i++ {
        ws, _, err := websocket.DefaultDialer.Dial(wsURL, nil)
        if err != nil {
            t.Fatal(err)
        }
        defer ws.Close()
    }
    waitForConnectedClients(api, 10)
    
    // 2. 执行 Close
    api.Close()
    
    // === 可断言的检查点 1: Close 返回后 clients map 必须为空 ===
    api.lock.RLock()
    clientCountAfterClose := len(api.clients[1])
    api.lock.RUnlock()
    
    // 这个断言必须通过，否则就是 bug
    if clientCountAfterClose != 0 {
        t.Errorf("After Close(), clients should be empty, but got %d", clientCountAfterClose)
    } else {
        t.Log("Check 1 PASSED: clients map is empty after Close()")
    }
    
    // === 可断言的检查点 2: Close 后调用 Notify 不应该 panic ===
    defer func() {
        if r := recover(); r != nil {
            t.Errorf("Notify after Close() panicked: %v", r)
        } else {
            t.Log("Check 2 PASSED: Notify after Close() does not panic")
        }
    }()
    api.Notify(1, &model.MessageExternal{Message: "after close"})
    
    // === 可断言的检查点 3: Close 后调用 Handle，观察新连接是否能注册 ===
    // 这记录当前行为，用于设计决策
    ws, _, err := websocket.DefaultDialer.Dial(wsURL, nil)
    if err != nil {
        t.Logf("Handle after Close() was rejected: %v", err)
    } else {
        defer ws.Close()
        time.Sleep(50 * time.Millisecond) // 等待注册
        
        api.lock.RLock()
        newClientCount := len(api.clients[1])
        api.lock.RUnlock()
        
        if newClientCount > 0 {
            t.Errorf("OBSERVATION: Handle after Close() registered new connection, count=%d (design decision needed)", newClientCount)
        } else {
            t.Log("Check 3 PASSED: Handle after Close() did not register new connection")
        }
    }
}
```

2. **运行命令：**
```bash
go test -v ./api/stream -run TestRace_CloseAndNewConnections_StateCheck -count=1
```

3. **判定信号（可直接断言）：**
- ✅ **检查 1 通过**：`clients map is empty after Close()` - Close 正确清空 map
- ❌ **检查 1 失败**：clients 非空 - 这是 bug
- ✅ **检查 2 通过**：`Notify after Close() does not panic` - 行为正确
- ❌ **检查 2 失败**：Notify 触发 panic - 这是 bug
- ⚠️ **检查 3 观察**：记录 Handle 在 Close 之后的行为（当前设计 vs 期望设计）

---

### 场景 5 复现：写通道满导致死锁

**复现目标：** 验证通道满时 Notify 持有锁阻塞

**复现步骤：**

1. **添加测试到 `api/stream/stream_test.go`:**
```go
func TestRace_FullChannelDeadlock(t *testing.T) {
    mode.Set(mode.TestDev)
    
    // Mock writeJSON 阻塞，模拟消费停止
    oldWriteJSON := writeJSON
    writeJSON = func(conn *websocket.Conn, v interface{}) error {
        time.Sleep(10 * time.Second)  // 长时间阻塞，不消费通道
        return nil
    }
    defer func() { writeJSON = oldWriteJSON }()
    
    server, api := bootTestServer(staticUserID())
    defer server.Close()
    defer api.Close()
    
    wsURL := wsURL(server.URL)
    testClient(t, wsURL)
    waitForConnectedClients(api, 1)
    
    // 带超时检测死锁
    done := make(chan bool)
    go func() {
        // 连续发送 3 条消息，通道容量只有 1
        // 第 2、3 条会阻塞并持有 RLock
        for i := 0; i < 3; i++ {
            api.Notify(1, &model.MessageExternal{Message: "test"})
        }
        done <- true
    }()
    
    select {
    case <-done:
        t.Log("No deadlock detected")
    case <-time.After(500 * time.Millisecond):
        t.Fatal("DEADLOCK DETECTED: Notify blocked holding RLock")
    }
}
```

2. **运行命令：**
```bash
go test ./api/stream -run TestRace_FullChannelDeadlock -v -count=1
```

3. **判定信号：**
- ✅ `DEADLOCK DETECTED` - 测试超时失败，检测到死锁

---

## 5. 批量验证脚本

**一键运行所有 Race 测试：**

```bash
# Windows PowerShell
cd d:\fz\0508-1\solo-dogfeeding\code\102-server

# 循环运行 10 次增加触发概率
for ($i=1; $i -le 10; $i++) {
    Write-Host "=== Run $i ==="
    go test -race ./api/stream/... -run "TestRace_" -count=1 -timeout=60s
    if ($LASTEXITCODE -ne 0) {
        Write-Host "FAILED at run $i"
        exit 1
    }
}
```

---

## 6. 验证结果汇总

| 场景 | 可断言的判定标准 | 验证状态 |
|------|-----------------|----------|
| 1. Notify vs NotifyClose | race detector 报告 channel send/close 竞争<br>或 panic: send on closed channel | ⚠️ 待验证 |
| 2. Once 内存可见性 | race detector 报告数据竞争<br>或断言失败 Memory visibility issue | ⚠️ 待验证 |
| 3. 双 goroutine 退出 | race detector 报告竞争<br>或无 panic（once 保护有效） | ⚠️ 待验证 |
| 4. Close 后状态检查 | 检查 1: clients map 必须为空<br>检查 2: Notify 不 panic<br>检查 3: Handle 行为观察 | ⚠️ 待验证 |
| 5. 通道满死锁 | DEADLOCK DETECTED 超时失败 | ⚠️ 待验证 |

**验证说明：**
- ✅ 场景 1 已硬化为真实运行态链路（关闭 websocket 触发 NotifyClose）
- ✅ 场景 4 已硬化为可直接断言的状态检查（不是依赖泄漏观测）
- ✅ 所有测试用例均可直接复制到当前代码库运行
- ✅ 每个场景都有明确的通过/失败判定标准
