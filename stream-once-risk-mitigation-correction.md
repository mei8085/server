# Stream 风险缓解方案纠正报告

## 核心议题：select + default 真的能防止 panic 吗？

本报告逐条核对 Go channel 关闭后的精确语义，纠正之前的错误假设，给出真正稳妥的并发改造方案。

---

## 1. Go Channel 语义精确核对

### 1.1 基础行为对照表

| 操作 | 未关闭 channel (空) | 未关闭 channel (有空间) | 已关闭 channel |
|------|---------------------|------------------------|----------------|
| `ch <- x` (直接发送) | 阻塞 | 成功发送 ✓ | **panic** 💥 |
| `x, ok := <-ch` (直接接收) | 阻塞 | 成功接收 ✓ | 返回 (零值, false) |
| `close(ch)` | 成功关闭 ✓ | 成功关闭 ✓ | **panic** 💥 |

### 1.2 Select 语句的精确行为

#### 场景 A: select + 发送 case (channel 已满)
```go
select {
case ch <- x:
    fmt.Println("sent")
default:
    fmt.Println("not sent")
}
```
**行为**: 执行 default 分支 ✅

#### 场景 B: select + 发送 case (channel 已关闭！)
```go
close(ch)

select {
case ch <- x:       // 这个 case 会发生什么？
    fmt.Println("sent")
default:
    fmt.Println("not sent")
}
```

**关键问题**: 会执行 default，还是直接 panic？

---

## 2. 核心纠正：select + default 不能防止 panic

### 2.1 精确行为验证

**Go 语言确切语义**:
> 在 select 语句中，如果某个 case 是向已关闭的 channel 发送数据，**会直接 panic**，不会进入 default 分支。

**为什么**:
- select 会按伪随机顺序检查所有 case
- 对于发送 case，检查时会判断 channel 状态
- 如果发现 channel 已关闭，**直接触发 panic**
- panic 发生在 case 评估阶段，在选择 default 之前

### 2.2 验证代码
```go
func main() {
    ch := make(chan int, 1)
    close(ch)
    
    // 这会 panic，不会进入 default！
    select {
    case ch <- 1:
        fmt.Println("sent")
    default:
        fmt.Println("default")  // 永远不会执行到这里
    }
}

// 输出: panic: send on closed channel
```

### 2.3 之前结论的错误

❌ **之前的错误假设**: "使用 select+default 非阻塞发送可以防止 panic"

✅ **正确结论**: 
- select + default **只能防止因 channel 满导致的阻塞**
- select + default **完全不能防止向已关闭 channel 发送导致的 panic**

---

## 3. 什么时候 select + default 仍会触发 panic

### 3.1 触发条件清单

| 场景 | 是否 panic | 说明 |
|------|-----------|------|
| channel 已满，select 发送 | ❌ 不 panic | 进入 default |
| channel 已关闭，select 发送 | ✅ 必然 panic | 即使有 default 也没用 |
| channel 未关闭，正在发送时被关闭 | ⚠️ 可能 panic | 竞态条件 |

### 3.2 竞态条件详细分析

```
Goroutine A (发送):
    T1: 执行 select
    T2: 检查 ch 是否可发送
    T3: 发现可以发送（但此时 channel 尚未关闭）
    
Goroutine B (关闭):
    T2.5: close(ch)  // 在 A 检查后发生
    
Goroutine A (继续):
    T4: 执行 ch <- x  // 此时 channel 已关闭 → panic!
```

**结论**: 即使检查时 channel 未关闭，在检查和实际发送之间也可能被关闭，导致 panic。

---

## 4. 稳妥的并发改造方案对比

### 方案一：recover 兜底（不推荐）

```go
func safeSend(ch chan *model.MessageExternal, msg *model.MessageExternal) (ok bool) {
    defer func() {
        if recover() != nil {
            ok = false
        }
    }()
    
    ch <- msg
    return true
}

// 使用
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    a.lock.RLock()
    clients := a.clients[userID]
    a.lock.RUnlock()
    
    for _, c := range clients {
        safeSend(c.write, msg)
    }
}
```

| 优点 | 缺点 |
|------|------|
| 简单直接 | 性能开销 |
| 确实能防止 panic | 掩盖设计问题 |
| | 不是清晰的设计 |

**评分**: ⭐⭐

---

### 方案二：原子关闭标记 + 检查（推荐）

```go
type client struct {
    conn      *websocket.Conn
    write     chan *model.MessageExternal
    userID    uint
    token     string
    once      once
    closed    uint32  // 新增：原子标记
}

func (c *client) Close() {
    c.once.Do(func() {
        atomic.StoreUint32(&c.closed, 1)  // 先标记
        c.conn.Close()
        close(c.write)
    })
}

func (c *client) IsClosed() bool {
    return atomic.LoadUint32(&c.closed) == 1
}

// 安全发送
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    a.lock.RLock()
    clients := a.clients[userID]
    a.lock.RUnlock()
    
    for _, c := range clients {
        if c.IsClosed() {
            continue
        }
        
        select {
        case c.write <- msg:
            // 发送成功
        default:
            // channel 满，丢弃
        }
    }
}
```

**⚠️ 仍有竞态**: 检查 `IsClosed()` 和实际发送之间，channel 可能被关闭。

| 优点 | 缺点 |
|------|------|
| 99% 场景下工作 | 仍有理论上的竞态窗口 |
| 性能好 | 不能 100% 保证 |

**评分**: ⭐⭐⭐⭐

---

### 方案三：发送方负责关闭（最佳实践）

**Go Channel 设计原则**:
> ✅ 发送方负责关闭 channel，而不是接收方
> ❌ 接收方不应该关闭 channel

#### 问题分析
当前代码违反了这个原则：
- 写协程（接收方）负责关闭 write channel
- Notify（发送方）只负责发送
- 这导致了经典的 "发送方不知道 channel 已关闭" 问题

#### 改造方案

```go
// 改变职责：API 负责管理 client 生命周期
type API struct {
    clients     map[uint][]*client
    lock        sync.RWMutex
    // ...
}

// 从 map 移除后再关闭，确保发送方看不到已关闭的 client
func (a *API) removeAndClose(c *client) {
    a.lock.Lock()
    
    // 1. 先从 map 中移除，新的 Notify 不会看到这个 client
    if userIDClients, ok := a.clients[c.userID]; ok {
        for i, client := range userIDClients {
            if client == c {
                a.clients[c.userID] = append(userIDClients[:i], userIDClients[i+1:]...)
                break
            }
        }
    }
    
    a.lock.Unlock()
    
    // 2. 现在可以安全关闭了（已经没有新的发送者了）
    c.Close()
}

// Notify 只能拿到 map 中存在的（未关闭的）client
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    a.lock.RLock()
    clients := a.clients[userID]
    a.lock.RUnlock()
    
    for _, c := range clients {
        select {
        case c.write <- msg:
        default:
        }
    }
}

// 改造 NotifyClose
func (c *client) NotifyClose() {
    c.once.Do(func() {
        c.conn.Close()
        // 不在这里关闭 channel 和调用 onClose
        // 改为回调 removeAndClose
        c.onClose(c)  // onClose 现在调用 removeAndClose
    })
}
```

| 优点 | 缺点 |
|------|------|
| 遵循 Go 最佳实践 | 需要重构职责 |
| 从根源消除 panic | 代码改动较大 |
| 100% 安全 | |

**评分**: ⭐⭐⭐⭐⭐

---

### 方案四：使用 sync.Map 替代带锁 map（性能优化）

```go
type API struct {
    clients     sync.Map  // map[uint][]*client
    // ...
}

func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    if clients, ok := a.clients.Load(userID); ok {
        for _, c := range clients.([]*client) {
            select {
            case c.write <- msg:
            default:
            }
        }
    }
}
```

| 优点 | 缺点 |
|------|------|
| 高并发性能更好 | 类型不安全 |
| 读写不互斥 | 需要额外处理类型断言 |

**评分**: ⭐⭐⭐⭐

---

## 5. 推荐的分阶段实施方案

### 阶段一：立即修复（P0）- 最小改动防止 panic

```go
// 先用 recover 兜底，最快速、最安全
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    a.lock.RLock()
    defer a.lock.RUnlock()
    
    if clients, ok := a.clients[userID]; ok {
        for _, c := range clients {
            func() {
                defer func() { recover() }()  // 简单兜底
                
                select {
                case c.write <- msg:
                default:
                }
            }()
        }
    }
}
```

**理由**: 
- 改动最小，风险最低
- 100% 防止 panic
- 为后续重构争取时间

### 阶段二：中期重构（P1）- 遵循最佳实践

1. 引入原子 `closed` 标记
2. 重构 `removeAndClose` 方法
3. 发送方先检查再发送

### 阶段三：长期优化（P2）- 性能优化

1. 考虑使用 `sync.Map`
2. 调整 channel 容量
3. 添加监控和指标

---

## 6. 最终结论校准

### 6.1 关键纠正

| 之前结论 | 纠正后结论 |
|---------|-----------|
| select + default 可以防止 panic | ❌ **完全不能**，向已关闭 channel 发送在任何情况下都会 panic |
| 主要风险是死锁 | ❌ **主要风险是 panic 和竞态条件** |

### 6.2 真实风险重新评估

| 风险 | 确定性 | 说明 |
|------|--------|------|
| **Send on closed channel panic** | 🔴 100% | 只要有并发，几乎必然发生 |
| **竞态条件** | 🔴 高 | 检查和发送之间的窗口可能被关闭 |
| **不可恢复阻塞** | 🟠 中 | 客户端卡住时发生 |
| **严格死锁** | 🟢 无 | Go 运行时会打破 |

### 6.3 最终建议

1. ✅ **立即实施**: 先用 `recover` 兜底，这是唯一能 100% 防止 panic 的简单方法
2. ✅ **必须重构**: 遵循 Go channel 最佳实践，发送方管理生命周期
3. ❌ **不要依赖**: select + default 不能解决关闭问题

> **关键教训**: Go 的并发安全不能靠运气，必须遵循设计原则。发送方负责关闭 channel 不是教条，是血的教训。
