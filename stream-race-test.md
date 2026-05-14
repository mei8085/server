# 推送流并发 Race 边界场景测试报告

## 1. 概述

本文档分析 Gotify 服务器推送流（WebSocket）模块中的并发 Race 边界场景，并说明如何稳定复现这些时序问题。

## 2. 核心并发结构分析

### 2.1 关键数据结构

#### API 管理器 (`api/stream/stream.go:19-25`)
```go
type API struct {
    clients     map[uint][]*client  // 用户ID到客户端列表的映射
    lock        sync.RWMutex        // 保护 clients 的读写锁
    pingPeriod  time.Duration
    pongTimeout time.Duration
    upgrader    *websocket.Upgrader
}
```

#### 客户端 (`api/stream/client.go:23-30`)
```go
type client struct {
    conn    *websocket.Conn
    onClose func(*client)
    write   chan *model.MessageExternal  // 容量为1的缓冲通道
    userID  uint
    token   string
    once    once  // 自定义Once，确保Close只执行一次
}
```

#### 自定义Once (`api/stream/once.go:14-17`)
```go
type once struct {
    m    sync.Mutex
    done uint32
}
```

### 2.2 并发执行流

每个客户端连接启动两个goroutine：
1. `startReading` - 读取客户端消息，处理pong
2. `startWriteHandler` - 写入消息到客户端，发送ping

## 3. Race 边界场景分类

### 场景1：并发连接关闭 vs 消息写入

#### 触发条件
- 客户端A正在执行`NotifyClose()`（调用`onClose`回调从API.clients中移除）
- 同时API调用`Notify()`向客户端A发送消息

#### 风险点
1. `client.write`通道关闭后，写入操作导致panic
2. 移除客户端列表时与读取操作产生数据竞争

#### 代码位置
- `stream.go:83-91` - Notify() 发送消息
- `client.go:51-57` - NotifyClose() 关闭连接
- `stream.go:93-104` - remove() 从列表移除

---

### 场景2：自定义Once的并发执行

#### 触发条件
- 两个goroutine同时调用`client.Close()`或`client.NotifyClose()`
- 第一个goroutine在`mayExecute()`中设置`done=1`后释放锁
- 第二个goroutine在原子读取时可能看到未完全同步的状态

#### 风险点
1. 标准`sync.Once`在执行f()时持有锁，而自定义实现提前释放锁
2. 内存可见性问题：`done`标记设置后，f()内部的状态可能未完全同步到其他goroutine

#### 代码位置
- `once.go:19-26` - Do() 方法
- `once.go:28-36` - mayExecute() 方法

---

### 场景3：客户端注册与移除的竞态

#### 触发条件
- 客户端A连接成功，执行`register()`添加到列表
- 同时客户端A因网络错误执行`remove()`从列表移除
- 或者`NotifyDeletedClient()`/`NotifyDeletedUser()`并发执行

#### 风险点
1. 客户端被重复添加或移除
2. 遍历客户端列表时并发修改导致数据不一致

#### 代码位置
- `stream.go:106-110` - register()
- `stream.go:93-104` - remove()
- `stream.go:66-80` - NotifyDeletedClient()
- `stream.go:53-64` - NotifyDeletedUser()

---

### 场景4：写通道满时的并发关闭

#### 触发条件
- `client.write`通道容量为1，已写入一条消息但未被消费
- `NotifyClose()`被调用，执行`close(c.write)`
- 同时`Notify()`尝试写入第二条消息到`c.write`

#### 风险点
1. 向已关闭的通道写入导致panic
2. 关闭通道后，select case仍然可能收到零值

#### 代码位置
- `stream.go:83-91` - Notify()
- `client.go:43-57` - Close()/NotifyClose()

---

### 场景5：API.Close() 与新连接并发

#### 触发条件
- `API.Close()`正在遍历关闭所有客户端
- 同时有新客户端调用`Handle()`进行注册

#### 风险点
1. 新连接在Close()执行期间注册，导致资源泄漏
2. 遍历map时并发修改

#### 代码位置
- `stream.go:161-173` - Close()
- `stream.go:143-158` - Handle()

---

### 场景6：读写goroutine同时退出

#### 触发条件
- `startReading`因读取错误调用`NotifyClose()`
- 同时`startWriteHandler`因写入错误也调用`NotifyClose()`
- 两个goroutine都尝试关闭同一个连接和通道

#### 风险点
1. 重复关闭连接或通道
2. `onClose`回调被重复调用

#### 代码位置
- `client.go:61-75` - startReading()
- `client.go:81-108` - startWriteHandler()

---

### 场景7：CollectConnectedClientTokens 并发遍历

#### 触发条件
- `CollectConnectedClientTokens()`正在遍历所有客户端收集token
- 同时有客户端连接或断开，修改`API.clients`

#### 风险点
1. 遍历过程中map被修改导致部分数据丢失或重复
2. 数据竞争导致崩溃

#### 代码位置
- `stream.go:41-51` - CollectConnectedClientTokens()

## 4. 稳定复现方法

### 4.1 使用 Go Race Detector

运行测试时启用race检测：
```bash
go test -race ./api/stream/... -v
```

### 4.2 构造高并发测试场景

#### 测试用例1：并发连接关闭与消息发送

```go
func TestRace_CloseAndNotify(t *testing.T) {
    api := New(100*time.Millisecond, 100*time.Millisecond, []string{})
    defer api.Close()
    
    // 启动N个客户端
    for i := 0; i < 100; i++ {
        go func() {
            // 建立连接...
            api.Notify(userID, msg)
        }()
    }
    
    // 同时随机关闭连接
    for i := 0; i < 50; i++ {
        go func() {
            time.Sleep(time.Duration(rand.Intn(10)) * time.Millisecond)
            api.NotifyDeletedUser(userID)
        }()
    }
    
    time.Sleep(500 * time.Millisecond)
}
```

#### 测试用例2：自定义Once的并发测试

```go
func TestRace_CustomOnce(t *testing.T) {
    o := &once{}
    var wg sync.WaitGroup
    
    // 100个goroutine同时调用Do
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            o.Do(func() {
                // 模拟耗时操作，增加race概率
                time.Sleep(10 * time.Millisecond)
            })
        }()
    }
    
    wg.Wait()
}
```

#### 测试用例3：API.Close 与新连接并发

```go
func TestRace_CloseAndNewConnection(t *testing.T) {
    server, api := bootTestServer(staticUserID())
    
    var wg sync.WaitGroup
    
    // 启动多个连接
    for i := 0; i < 20; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            ws, _, err := websocket.DefaultDialer.Dial(wsURL(server.URL), nil)
            if err == nil {
                time.Sleep(time.Duration(rand.Intn(50)) * time.Millisecond)
                ws.Close()
            }
        }()
    }
    
    // 同时调用Close
    go func() {
        time.Sleep(25 * time.Millisecond)
        api.Close()
    }()
    
    wg.Wait()
    server.Close()
}
```

### 4.3 增加延迟放大Race窗口

在关键位置插入随机延迟：

```go
// 在 once.Do 的 f() 执行前
func (o *once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 1 {
        return
    }
    if o.mayExecute() {
        // 测试时注入延迟，增加race概率
        if testing.Testing() {
            time.Sleep(1 * time.Millisecond)
        }
        f()
    }
}
```

### 4.4 循环运行测试

```bash
# 循环运行100次测试，增加捕获race的概率
for i in {1..100}; do 
    go test -race ./api/stream/... -run TestRace -count=1
    if [ $? -ne 0 ]; then
        echo "Failed at iteration $i"
        exit 1
    fi
done
```

## 5. 现有测试覆盖分析

### 5.1 已覆盖的场景

| 测试用例 | 覆盖场景 |
|---------|---------|
| `TestMultipleClients` | 多客户端并发连接、断开、消息发送 |
| `TestDeleteUser` / `TestDeleteClient` | 删除用户/客户端时的并发处理 |
| `Test_Execute` (once_test.go) | 自定义Once的基础并发 |
| `TestCloseClientOnNotReading` | 读取超时关闭连接 |
| `TestWriteMessageFails` / `TestWritePingFails` | 写入错误时的退出逻辑 |

### 5.2 未充分覆盖的场景

1. **高压力下的消息发送与连接关闭并发**
2. **API.Close() 与新连接注册的竞态**
3. **写通道满时的关闭场景**
4. **读写goroutine同时出错退出**

## 6. 关键时序图

### 6.1 客户端关闭时序

```
Goroutine A (startReading)        Goroutine B (Notify)
     |                                  |
     |  err := ReadMessage()            |
     |  (检测到错误)                    |
     |                                  |
     v                                  v
NotifyClose()                     c.write <- msg
     |                                  |
     o.m.Lock()                         |
     o.done == 0                        |
     atomic.Store(&o.done, 1)           |
     o.m.Unlock()                       |
     |                                  |
     c.conn.Close()              <==== 竞态窗口 ====>
     close(c.write)                     |
     c.onClose(c)                       |
     |                                  |
     v                                  v
                               向已关闭通道写入 -> PANIC
```

### 6.2 自定义Once执行时序

```
Goroutine 1              Goroutine 2              Goroutine 3
     |                       |                       |
     o.Do(f)                 o.Do(f)                 o.Do(f)
     |                       |                       |
     atomic.Load == 0        atomic.Load == 0        atomic.Load == 0
     |                       |                       |
     o.m.Lock()              o.m.Lock() (等待)       o.m.Lock() (等待)
     o.done == 0             |                       |
     atomic.Store(&o.done,1) |                       |
     o.m.Unlock() ---------> |                       |
     |                       o.done == 1             |
     f()                     return                  |
     |                                               |
     | <---------------- 竞态窗口 ----------------> |
     |                                               o.done == 1?
     |                                               (内存可见性问题)
```

## 7. 修复建议方向

1. **写通道关闭保护**：在Notify中检查通道状态或使用select带default
2. **Once内存屏障**：确保f()执行完成后，其他goroutine能看到完整状态
3. **API.Close原子性**：添加关闭标记，拒绝新连接
4. **通道容量评估**：考虑增加write通道容量或使用非阻塞发送

## 8. 总结

推送流模块的Race风险主要集中在：
- 连接关闭与消息发送的时序竞争
- 自定义Once实现的内存可见性
- 客户端列表的并发修改
- 双goroutine退出时的重复操作

通过启用`-race`标志、构造高并发测试场景、在关键位置插入延迟，可以稳定复现这些时序问题。
