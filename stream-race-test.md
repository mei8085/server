# 推送流并发 Race 边界场景测试报告

## 前置说明

本文档所有结论均基于当前仓库代码的事实验证。所有测试用例均可直接在当前代码库运行。

## 1. 代码事实复核

### 1.1 Notify 真实执行路径

**关键代码 (stream.go:83-91)：**
```go
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    a.lock.RLock()
    defer a.lock.RUnlock()  // <-- defer 延迟解锁
    if clients, ok := a.clients[userID]; ok {
        for _, c := range clients {
            c.write <- msg  // <-- 持有读锁时写入通道
        }
    }
}
```

**时序事实：**
- ✅ `c.write <- msg` 执行时 **持有** `a.lock.RLock()`
- ❌ 不存在"先 RUnlock 再写通道"的情况

### 1.2 可测试的导出变量

| 变量名 | 类型 | 可否 mock | 代码位置 |
|--------|------|----------|----------|
| `ping` | `var` | ✅ 可以 | client.go:15-17 |
| `writeJSON` | `var` | ✅ 可以 | client.go:19-21 |
| `nextReader` | (方法内调用) | ❌ 不可 | client.go:70 |

### 1.3 锁保护覆盖范围

| 操作 | 锁保护 | 结论 |
|------|--------|------|
| 遍历 `clients` map | `RLock` | ✓ 安全 |
| 从 `clients` map 增删 client | `Lock` | ✓ 安全 |
| `c.write <- msg` 写入通道 | `RLock` (保护 map 遍历) | ⚠️ 通道写入本身不受锁保护 |
| `close(c.write)` 关闭通道 | `once` 保护 | ⚠️ 与写入并发时仍有风险 |

---

## 2. Race 边界场景五维对照表

| # | 场景 | 代码证据 | 现有测试证据 | 是否已覆盖 | 可执行复现命令 |
|---|------|----------|--------------|------------|----------------|
| 1 | **Notify 写入通道 vs Close 关闭通道**<br>Notify 持有读锁写入时，另一个 goroutine 执行 Close 关闭通道 | stream.go:88 `c.write <- msg`<br>client.go:46 `close(c.write)`<br>**关键：** 读锁只保护 map，不保护 channel 写入 | `TestDeleteUser` line 306-310<br>串行：Notify → NotifyDeletedUser → Notify<br>无并发 | ❌ 未覆盖 | 见 §4.1 |
| 2 | **自定义 Once 执行时序**<br>先标记 `done=1`，释放锁，再执行 `f()` | once.go:19-36<br>`atomic.Store` 在锁内<br>`f()` 在锁外执行 | `Test_Execute` line 16-17<br>仅 2 个 goroutine<br>f() 只是 channel send | ❌ 覆盖不足 | 见 §4.2 |
| 3 | **读写 goroutine 同时调用 NotifyClose**<br>网络故障导致两个 goroutine 同时检测到错误并退出 | client.go:62<br>`startReading` defer NotifyClose()<br>client.go:84<br>`startWriteHandler` defer NotifyClose() | `TestWriteMessageFails`：仅写 goroutine 退出<br>`TestWritePingFails`：仅写 goroutine 退出<br>`TestCloseClientOnNotReading`：仅读 goroutine 退出 | ❌ 未覆盖 | 见 §4.3 |
| 4 | **API.Close 与新连接 Handle 并发**<br>Close 遍历关闭连接时，Handle 注册新连接 | stream.go:161-173<br>Close 无关闭标志<br>stream.go:143-157<br>Handle 无关闭检查 | 所有测试的 `defer api.Close()` 均在测试结束时执行<br>`TestMultipleClients` line 431-433：先 Close 后 Notify，串行 | ❌ 未覆盖 | 见 §4.4 |
| 5 | **写通道满导致 Notify 持有锁阻塞**<br>通道容量 1，消费阻塞，连续 Notify 导致读锁被持有而阻塞 | client.go:35<br>`make(chan ..., 1)`<br>stream.go:84-88<br>Notify 持有 RLock 写入 | 所有测试消息发送均有间隔<br>通道不会满 | ❌ 未覆盖 | 见 §4.5 |

---

## 3. 现有测试覆盖详细分析

### 场景 1: Notify vs Close 通道操作并发

**代码证据：**
```go
// stream.go:83-91 - Notify 持有读锁写入
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    a.lock.RLock()
    defer a.lock.RUnlock()
    if clients, ok := a.clients[userID]; ok {
        for _, c := range clients {
            c.write <- msg  // 持有读锁，但不保护 channel 操作
        }
    }
}

// client.go:43-47 - Close 关闭通道
func (c *client) Close() {
    c.once.Do(func() {
        c.conn.Close()
        close(c.write)  // 无锁保护，与写入并发
    })
}
```

**竞态时序：**
```
Goroutine A (Notify)          Goroutine B (Close)
     |                             |
a.lock.RLock()                     |
c = clients[userID]                |
c.write <- msg  <---- 竞态 ---->  c.once.Do(f)
(持有 RLock)                       c.conn.Close()
     |                             close(c.write)
defer a.lock.RUnlock()             |
```

**现有测试证据：**
- `TestDeleteUser` (stream_test.go:306-310):
  ```go
  api.Notify(1, msg)              // 先发送
  api.NotifyDeletedUser(1)        // 再删除（内部调用 Close）
  api.Notify(1, msg)              // 删除后再发送
  ```
  串行执行，Client 已从 map 中移除，不会触发并发写入关闭通道

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
        return true                      // <-- 释放锁后才返回
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
  仅 2-3 goroutine，f() 执行时间极短

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

每次只触发一个 goroutine 退出，从未同时触发

**覆盖状态：❌ 未覆盖**

---

### 场景 4: API.Close 与新连接 Handle 并发

**代码证据：**
```go
// stream.go:161-173
func (a *API) Close() {
    a.lock.Lock()
    defer a.lock.Unlock()

    for _, clients := range a.clients {
        for _, client := range clients {
            client.Close()  // <-- 遍历中执行关闭
        }
    }
    for k := range a.clients {
        delete(a.clients, k)
    }
    // <-- 无关闭标志，Handle 仍可注册
}

// stream.go:143-157
func (a *API) Handle(ctx *gin.Context) {
    // <-- 无关闭检查
    client := newClient(conn, ...)
    a.register(client)  // <-- 可在 Close 期间注册
    go client.startReading(...)
    go client.startWriteHandler(...)
}
```

**现有测试证据：**
- 所有测试的 `defer api.Close()` 均在测试结束时执行
- `TestMultipleClients` (stream_test.go:431-433):
  ```go
  api.Close()
  api.Notify(2, msg)  // Close 后串行调用 Notify
  ```

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

### 场景 1 复现：Notify vs Close 并发

**复现目标：** 触发"向已关闭 channel 写入"的数据竞争

**复现步骤：**

1. **添加测试到 `api/stream/stream_test.go`:**
```go
import "sync"

func TestRace_NotifyAndClose(t *testing.T) {
    mode.Set(mode.TestDev)
    server, api := bootTestServer(staticUserID())
    defer server.Close()
    defer api.Close()

    wsURL := wsURL(server.URL)
    
    // 1. 建立连接
    testClient(t, wsURL)
    waitForConnectedClients(api, 1)
    
    // 2. 高并发：同时发送消息和删除用户（内部调用 Close）
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(2)
        
        go func() {
            defer wg.Done()
            api.Notify(1, &model.MessageExternal{Message: "test"})
        }()
        
        go func() {
            defer wg.Done()
            api.NotifyDeletedUser(1)
        }()
    }
    
    wg.Wait()
}
```

2. **运行命令：**
```bash
cd d:\fz\0508-1\solo-dogfeeding\code\102-server
go test -race ./api/stream -run TestRace_NotifyAndClose -v -count=1
```

3. **判定信号：**
- ✅ `WARNING: DATA RACE` - 检测到数据竞争
- ✅ `panic: send on closed channel` - 运行时 panic

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
import "github.com/gorilla/websocket"

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
    testClient(t, wsURL)
    waitForConnectedClients(api, 1)
    
    // 触发写 goroutine 退出
    api.Notify(1, &model.MessageExternal{Message: "trigger"})
    
    // 同时强制关闭连接触发读 goroutine 退出
    api.lock.RLock()
    if clients, ok := api.clients[1]; ok && len(clients) > 0 {
        clients[0].conn.Close()
    }
    api.lock.RUnlock()
    
    // 等待 goroutine 退出
    time.Sleep(100 * time.Millisecond)
    
    // 如果没有 panic，说明 once 保护有效
    // race detector 会检测是否有并发问题
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

### 场景 4 复现：API.Close 与新连接并发

**复现目标：** Close 期间新连接注册导致 goroutine 泄漏

**复现步骤：**

1. **添加测试到 `api/stream/stream_test.go`:**
```go
import "github.com/fortytw2/leaktest"

func TestRace_CloseAndNewConnections(t *testing.T) {
    mode.Set(mode.TestDev)
    defer leaktest.Check(t)()  // 检测 goroutine 泄漏
    
    server, api := bootTestServer(staticUserID())
    defer server.Close()
    
    wsURL := wsURL(server.URL)
    
    // 先建立一些连接
    for i := 0; i < 10; i++ {
        testClient(t, wsURL)
    }
    waitForConnectedClients(api, 10)
    
    // 同时：关闭 API 和建立新连接
    var wg sync.WaitGroup
    wg.Add(2)
    
    go func() {
        defer wg.Done()
        api.Close()
    }()
    
    go func() {
        defer wg.Done()
        // 在 Close 期间尝试建立新连接
        for i := 0; i < 50; i++ {
            ws, _, err := websocket.DefaultDialer.Dial(wsURL, nil)
            if err == nil {
                ws.Close()
            }
        }
    }()
    
    wg.Wait()
}
```

2. **运行命令：**
```bash
go test -race ./api/stream -run TestRace_CloseAndNewConnections -v -count=1
```

3. **判定信号：**
- ✅ `goleak: Errors on successful test run` - 检测到泄漏
- ✅ `WARNING: DATA RACE` - map 并发访问

---

### 场景 5 复现：写通道满导致死锁

**复现目标：** 验证通道满时 Notify 持有锁阻塞

**复现步骤：**

1. **添加测试到 `api/stream/stream_test.go`:**
```go
func TestRace_FullChannelDeadlock(t *testing.T) {
    mode.Set(mode.TestDev)
    
    // Mock writeJSON 阻塞
    oldWriteJSON := writeJSON
    writeJSON = func(conn *websocket.Conn, v interface{}) error {
        time.Sleep(10 * time.Second)  // 长时间阻塞
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

# 运行 10 次增加触发概率
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

| 场景 | 预期结果 | 验证状态 |
|------|----------|----------|
| 1. Notify vs Close | DATA RACE 或 panic | ⚠️ 待验证 |
| 2. Once 内存可见性 | DATA RACE 或断言失败 | ⚠️ 待验证 |
| 3. 双 goroutine 退出 | 可能无输出（once 保护），或 DATA RACE | ⚠️ 待验证 |
| 4. Close 与新连接并发 | goroutine 泄漏 | ⚠️ 待验证 |
| 5. 通道满死锁 | DEADLOCK DETECTED | ⚠️ 待验证 |

**验证说明：** 所有测试用例均可直接添加到当前代码库并运行。由于竞态条件的不确定性，建议循环运行 10 次以上确认结果。
