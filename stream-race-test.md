# 推送流并发 Race 边界场景测试报告

## 1. 锁保护机制复核

### 1.1 已确认安全（受锁保护，无Race）

以下操作均在 `a.lock` 保护下执行，不存在数据竞争：

| 操作 | 锁类型 | 代码位置 | 结论 |
|------|--------|----------|------|
| `CollectConnectedClientTokens` | `RLock` | stream.go:41-51 | ✓ 安全，遍历 clients map 时受读锁保护 |
| `NotifyDeletedUser` | `Lock` | stream.go:53-64 | ✓ 安全，修改 clients map 时受写锁保护 |
| `NotifyDeletedClient` | `Lock` | stream.go:66-80 | ✓ 安全，修改 clients map 时受写锁保护 |
| `register` | `Lock` | stream.go:106-110 | ✓ 安全，添加客户端受写锁保护 |
| `remove` | `Lock` | stream.go:93-104 | ✓ 安全，移除客户端受写锁保护 |
| `Notify` 遍历 clients | `RLock` | stream.go:82-91 | ✓ 安全，遍历客户端列表受读锁保护 |

### 1.2 真正的 Race 边界（锁未覆盖）

以下操作序列存在竞态风险：

---

## 2. Race 边界场景清单（逐项对照）

### 场景 1: Notify 通道写入 vs Close 通道关闭

**触发时序：**
```
Goroutine A (Notify)          Goroutine B (Close)
     |                             |
a.lock.RLock()                     |
c = clients[userID]                |
a.lock.RUnlock()                   |
     |                             c.once.Do(f)
     |                                |
     |                             c.conn.Close()
     |                             close(c.write)   <---+
     |                                                | 竞态窗口
c.write <- msg   <------------------------------------+
```

**风险点：**
- `a.lock.RUnlock()` 释放后，`c.write <- msg` 不再受锁保护
- 此时 `Close()` 可能在另一个 goroutine 中执行 `close(c.write)`
- 向已关闭的 channel 写入会触发 panic

**代码位置：**
- stream.go:88 - `c.write <- msg` (锁已释放)
- client.go:43-47 - `Close()` 中的 `close(c.write)`

**现有测试证据：**
- `TestDeleteUser`: 先 `Notify`，再 `NotifyDeletedUser`，再 `Notify` - 串行执行，无并发
- `TestDeleteClient`: 同上，串行
- `TestMultipleClients`: 先关闭连接，`time.Sleep(500ms)` 后再 `Notify` - 有延迟保护

**测试覆盖状态：❌ 未覆盖**
- 现有测试均为串行执行，未模拟"Notify 和 Close 同时发生"的场景
- 即使启用 `-race`，现有测试也无法触发此 race

---

### 场景 2: 自定义 Once 内存可见性问题

**关键代码 (once.go:19-36)：**
```go
func (o *once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 1 {  // 快速路径
        return
    }
    if o.mayExecute() {  // 慢速路径
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

**触发时序：**
```
Goroutine 1              Goroutine 2
     |                       |
o.Do(f)                     o.Do(f)
     |                       |
atomic.Load == 0            atomic.Load == 0
     |                       |
o.m.Lock()                  o.m.Lock() (等待)
o.done == 0                 |
atomic.Store(&o.done, 1)    |
o.m.Unlock() -------------> |
     |                       o.done == 1
f()                          return (看到 done=1)
     |
     +-- 竞态窗口 --+
                   |
                f() 中的操作（如 close(c.write)）可能尚未对 Goroutine 2 完全可见
```

**与标准 `sync.Once` 的差异：**
- 标准库：在执行 `f` 期间持有锁，`f` 完成后才标记 done
- 自定义实现：先标记 done，释放锁，再执行 `f`
- 问题：其他 goroutine 看到 `done=1` 时，`f` 可能还在执行中

**代码位置：**
- once.go:19-36 - `Do()` 和 `mayExecute()`

**现有测试证据：**
- `Test_Execute` (once_test.go:10-42):
  ```go
  go executeOnce.Do(fExecute)  // 启动 2 个 goroutine
  go executeOnce.Do(fExecute)
  // 等待第一个完成，再启动第三个
  go executeOnce.Do(fExecute)
  ```

**测试覆盖状态：❌ 部分覆盖**
- 只有 2-3 个 goroutine，并发压力低
- `f()` 只是向 channel 发送，执行时间极短，race 窗口小
- 未验证 `f()` 内部状态的内存可见性

---

### 场景 3: 读写 goroutine 同时调用 NotifyClose

**触发时序：**
```
Goroutine (startReading)    Goroutine (startWriteHandler)
     |                             |
conn.NextReader() 出错        conn.WriteJSON() 出错
     |                             |
c.NotifyClose()                c.NotifyClose()
     |                             |
     +---- 同时进入 once.Do ------+
```

**风险点：**
- `startReading` 和 `startWriteHandler` 是独立 goroutine
- 网络故障可能导致两者同时检测到错误
- 两者都调用 `NotifyClose()`，依赖 `once` 保证只执行一次

**代码位置：**
- client.go:62 - `startReading` defer `NotifyClose()`
- client.go:84 - `startWriteHandler` defer `NotifyClose()`

**现有测试证据：**
- `TestWriteMessageFails`: mock `writeJSON` 出错，只有写 goroutine 退出
- `TestWritePingFails`: mock `ping` 出错，只有写 goroutine 退出
- `TestCloseClientOnNotReading`: pong 超时，只有读 goroutine 退出

**测试覆盖状态：❌ 未覆盖**
- 现有测试每次只让一个 goroutine 出错
- 未模拟"两个 goroutine 同时检测到错误并退出"的场景

---

### 场景 4: API.Close 与新连接 Handle 并发

**触发时序：**
```
Goroutine A (API.Close)     Goroutine B (Handle)
     |                             |
a.lock.Lock()                      |
for each client:                   |
    client.Close()                 |
delete(a.clients, ...)             |
a.lock.Unlock()                    |
                                   a.lock.Lock()
                                   a.clients[userID] = append(...)
                                   a.lock.Unlock()
                                   启动读写 goroutine
```

**风险点：**
- `API.Close()` 没有设置"关闭中"标志
- `Handle()` 没有检查 API 是否已关闭
- 关闭过程中可能有新连接注册，导致 goroutine 泄漏

**代码位置：**
- stream.go:161-173 - `Close()` 遍历关闭所有客户端
- stream.go:154-157 - `Handle()` 注册客户端并启动 goroutine

**现有测试证据：**
- `TestMultipleClients`: line 431-433 - 先 `api.Close()`，再 `api.Notify()` - 串行
- 所有测试的 `defer api.Close()` 都在测试结束时执行，此时已无新连接

**测试覆盖状态：❌ 未覆盖**
- 无测试模拟"Close 执行期间有新连接进来"

---

### 场景 5: 写通道满时的 Notify 阻塞

**触发条件：**
- `client.write` 容量为 1
- `startWriteHandler` 因网络阻塞未能及时消费消息
- `Notify` 连续发送多条消息

**风险点：**
- 第二条 `c.write <- msg` 会阻塞
- 由于 `Notify` 持有 `a.lock.RLock()`，阻塞会导致所有读操作被阻塞
- 整个推送系统可能因此死锁

**代码位置：**
- client.go:35 - `make(chan *model.MessageExternal, 1)`
- stream.go:84-85 - Notify 持有 RLock 时写入通道

**现有测试证据：**
- 无测试覆盖此场景
- 所有测试中消息发送都有间隔，通道不会满

**测试覆盖状态：❌ 未覆盖**

---

## 3. 边界场景 vs 测试覆盖 对照表

| 场景 | 描述 | 现有测试 | 是否已覆盖 |
|------|------|----------|------------|
| 1 | Notify 写入 vs Close 关闭通道 | `TestDeleteUser`, `TestDeleteClient`, `TestMultipleClients` | ❌ 串行执行，无并发 |
| 2 | 自定义 Once 内存可见性 | `Test_Execute` | ❌ 仅 2-3 goroutine，压力不足 |
| 3 | 读写 goroutine 同时调用 NotifyClose | `TestWriteMessageFails`, `TestWritePingFails` | ❌ 每次只触发一个 goroutine 退出 |
| 4 | API.Close 与新连接 Handle 并发 | （无对应测试） | ❌ 未覆盖 |
| 5 | 写通道满导致 Notify 阻塞死锁 | （无对应测试） | ❌ 未覆盖 |

---

## 4. 稳定复现最小步骤

### 场景 1 复现：Notify vs Close 并发

**目标：** 触发"向已关闭 channel 写入"的 panic 或 race detection

**最小步骤：**

```go
// 添加到 stream_test.go
func TestRace_NotifyAndClose(t *testing.T) {
    mode.Set(mode.TestDev)
    server, api := bootTestServer(staticUserID())
    defer server.Close()
    defer api.Close()

    wsURL := wsURL(server.URL)
    
    // 1. 建立连接
    client := testClient(t, wsURL)
    waitForConnectedClients(api, 1)
    
    // 2. 获取内部 client 指针（通过辅助函数）
    api.lock.RLock()
    internalClient := api.clients[1][0]
    api.lock.RUnlock()

    // 3. 高并发：同时发送消息和关闭连接
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(2)
        
        go func() {
            defer wg.Done()
            api.Notify(1, &model.MessageExternal{Message: "test"})
        }()
        
        go func() {
            defer wg.Done()
            internalClient.Close()
        }()
    }
    
    wg.Wait()
}
```

**触发信号：**
- 运行 `go test -race ./api/stream -run TestRace_NotifyAndClose`
- 预期：race detector 报告 "write on closed channel" 或数据竞争

---

### 场景 2 复现：自定义 Once 内存可见性

**最小步骤：**

```go
// 添加到 once_test.go
func TestRace_OnceMemoryVisibility(t *testing.T) {
    var wg sync.WaitGroup
    
    for round := 0; round < 100; round++ {
        o := &once{}
        var sharedState int32
        
        // 启动 50 个 goroutine 同时调用 Do
        for i := 0; i < 50; i++ {
            wg.Add(1)
            go func() {
                defer wg.Done()
                o.Do(func() {
                    // f() 中修改共享状态
                    atomic.StoreInt32(&sharedState, 42)
                    // 模拟耗时操作，扩大 race 窗口
                    time.Sleep(10 * time.Microsecond)
                })
                
                // Do 返回后读取 sharedState
                // 如果内存可见性有问题，这里可能读到 0 而不是 42
                if atomic.LoadInt32(&sharedState) != 42 {
                    t.Error("Memory visibility issue: sharedState is not 42")
                }
            }()
        }
        
        wg.Wait()
    }
}
```

**触发信号：**
- 运行 `go test -race ./api/stream -run TestRace_OnceMemoryVisibility`
- 预期：要么测试失败（sharedState != 42），要么 race detector 报告数据竞争

---

### 场景 3 复现：读写 goroutine 同时退出

**最小步骤：**

```go
// 添加到 stream_test.go
func TestRace_BothGoroutinesExit(t *testing.T) {
    mode.Set(mode.TestDev)
    
    // 同时 mock 两个出错路径
    oldWriteJSON := writeJSON
    oldPing := ping
    oldNextReader := nextReader
    
    writeJSON = func(conn *websocket.Conn, v interface{}) error {
        return errors.New("write error")
    }
    ping = func(conn *websocket.Conn) error {
        return errors.New("ping error")
    }
    // 注意：需要导出 NextReader 或通过其他方式 mock
    
    defer func() {
        writeJSON = oldWriteJSON
        ping = oldPing
        nextReader = oldNextReader
    }()
    
    server, api := bootTestServer(staticUserID())
    defer server.Close()
    defer api.Close()
    
    wsURL := wsURL(server.URL)
    testClient(t, wsURL)
    waitForConnectedClients(api, 1)
    
    // 发送消息触发写错误，同时让读也出错
    api.Notify(1, &model.MessageExternal{Message: "trigger"})
    
    // 等待 goroutine 退出
    time.Sleep(100 * time.Millisecond)
    
    // 如果没有 panic 或 race，则 once 保护有效
    // 但 race detector 应该能检测到是否有并发问题
}
```

**触发信号：**
- 运行 `go test -race ./api/stream -run TestRace_BothGoroutinesExit`
- 预期：检查是否有重复关闭的 race

---

### 场景 4 复现：API.Close 与新连接并发

**最小步骤：**

```go
// 添加到 stream_test.go
func TestRace_CloseAndNewConnections(t *testing.T) {
    mode.Set(mode.TestDev)
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
    
    // 检查是否有 goroutine 泄漏（leaktest 会检测）
}
```

**触发信号：**
- 运行 `go test -race ./api/stream -run TestRace_CloseAndNewConnections`
- 预期：leaktest 检测到 goroutine 泄漏，或 race detector 报告 map 并发访问

---

### 场景 5 复现：写通道满导致死锁

**最小步骤：**

```go
// 添加到 stream_test.go
func TestRace_FullChannelDeadlock(t *testing.T) {
    mode.Set(mode.TestDev)
    
    // mock writeJSON 永不返回，模拟网络阻塞
    oldWriteJSON := writeJSON
    writeJSON = func(conn *websocket.Conn, v interface{}) error {
        time.Sleep(10 * time.Second) // 长时间阻塞
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
        // 正常完成
    case <-time.After(500 * time.Millisecond):
        t.Fatal("Deadlock detected: Notify blocked on full channel")
    }
}
```

**触发信号：**
- 运行 `go test ./api/stream -run TestRace_FullChannelDeadlock`
- 预期：测试超时失败，检测到死锁

---

## 5. 通用复现策略

### 5.1 运行命令

```bash
# 单次运行带 race 检测
go test -race ./api/stream/... -v -count=1

# 循环运行 100 次增加触发概率
for i in {1..100}; do
    echo "=== Run $i ==="
    go test -race ./api/stream/... -count=1 -timeout=30s
    if [ $? -ne 0 ]; then
        echo "Failed at run $i"
        exit 1
    fi
done
```

### 5.2 判定成功标准

| 判定信号 | 来源 | 说明 |
|---------|------|------|
| `panic: send on closed channel` | 运行时 | 场景 1 被触发 |
| `WARNING: DATA RACE` | race detector | 任何数据竞争被检测到 |
| 测试超时 | go test | 场景 5 死锁被触发 |
| `goleak: Errors on successful test run` | leaktest | 场景 4 goroutine 泄漏 |
| 测试断言失败 | `t.Error()` / `t.Fatal()` | 场景 2 内存可见性问题 |

---

## 6. 结论

### 6.1 现有测试的局限性

1. **串行为主**：所有测试用例均为串行执行，未模拟真实高并发场景
2. **Race 窗口过小**：并发 goroutine 数量少（2-3 个），操作耗时短
3. **延迟保护**：`time.Sleep(500ms)` 等延迟掩盖了潜在并发问题
4. **Mock 单一**：每次只 mock 一个出错路径，未模拟双 goroutine 同时出错

### 6.2 真正的高风险场景

| 风险等级 | 场景 | 后果 |
|---------|------|------|
| 🔴 高 | 场景 1: Notify vs Close | 进程 panic 崩溃 |
| 🟠 中 | 场景 5: 通道满死锁 | 整个推送系统阻塞 |
| 🟠 中 | 场景 4: Close 期间新连接 | goroutine 泄漏 |
| 🟡 低 | 场景 2: Once 内存可见性 | 状态不一致（概率低） |
| 🟡 低 | 场景 3: 双 goroutine 退出 | 重复调用（once 已防护） |
