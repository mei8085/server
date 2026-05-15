# Stream 并发风险结论校准报告

## 校准说明

本报告对之前的并发风险分析进行逐条复核，基于 Go 语言确切的并发语义，明确区分：
- ✅ **可恢复阻塞**: 暂时性阻塞，条件满足后自动恢复
- ❌ **触发 Panic**: 确定会导致程序崩溃
- 🔒 **严格死锁**: 循环等待，永久阻塞无法恢复

---

## 场景一：持锁发送消息阻塞

### 代码路径
```go
// stream.go:83-91
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    a.lock.RLock()         // T1: 获取读锁
    defer a.lock.RUnlock() // T4: 函数退出时释放
    if clients, ok := a.clients[userID]; ok {
        for _, c := range clients {
            c.write <- msg  // T2: 持锁发送！可能阻塞
        }                      // T3: 发送完成
    }
}
```

### 精确时序分析

```
前提条件: client.write channel 已满 (容量=1 已占满)

Goroutine A (调用 Notify):
    T1: a.lock.RLock() ✓          获取读锁成功
    T2: c.write <- msg             尝试发送，channel 已满 → **阻塞**
    [永久等待 channel 空位...]

其他 Goroutine 行为:
    - 其他 Notify(): 可以同时获取读锁 ✓ (RWMutex 读锁共享)
    - register/remove/Close(): 需要写锁 → **全部阻塞**
```

### 风险等级校准

| 评估项 | 结论 | 依据 |
|--------|------|------|
| **是否可恢复** | ❌ **不可恢复** | 只有写协程消费消息才能空位，但写协程若被卡住则永远无法恢复 |
| **是否触发 Panic** | ❌ 不直接触发 | 只是 goroutine 阻塞 |
| **是否严格死锁** | ⚠️ **条件性死锁** | 当客户端消费卡住时形成死锁；否则只是暂时阻塞 |
| **系统影响** | **严重** | 所有需要写锁的操作永久阻塞 |

### 校准结论
**不是严格意义上的死锁，但是「不可恢复阻塞」**

理由：
- 不是循环等待，而是单方阻塞
- 如果客户端恢复消费，阻塞会自动解除
- 但如果客户端真正卡住（网络问题等），则永远无法恢复

---

## 场景二：关闭回调与读锁的交互

### 调用链
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
    a.lock.Lock()           // 需要获取写锁
    defer a.lock.Unlock()
    // 移除操作
}
```

### 临界场景分析

```
前提条件: Goroutine A 正在 Notify() 中持有读锁

Goroutine A (Notify):
    T1: a.lock.RLock() ✓
    T2: c.write <- msg  [正在发送，持有读锁]

Goroutine B (写协程出错):
    T3: 网络写入错误 → return
    T4: defer NotifyClose() 触发
    T5: c.once.Do() 执行
    T6: c.conn.Close() ✓
    T7: close(c.write) ✓
    T8: c.onClose(c) → 调用 a.remove()
    T9: a.lock.Lock() [阻塞等待写锁]
        (因为 Goroutine A 持有读锁)
```

### 此时 Goroutine A 的行为

**关键问题**: 当 `close(c.write)` 被调用后，正在阻塞的 `c.write <- msg` 会发生什么？

**Go 语言确切语义**:
> 向已关闭的 channel 发送数据会 **立即 panic**

所以实际时序是：
```
Goroutine A:
    T2: c.write <- msg  [阻塞等待]
    T7: [channel 被关闭]
    T7.1: c.write <- msg **panic**!
    T7.2: defer a.lock.RUnlock() 触发 ✓
    T7.3: 读锁释放

Goroutine B:
    T9: a.lock.Lock() 现在可以获取写锁了 ✓
```

### 风险等级校准

| 评估项 | 结论 | 依据 |
|--------|------|------|
| **是否可恢复** | ❌ **不可恢复** | Panic 发生 |
| **是否触发 Panic** | ✅ **确定触发** | 向已关闭的 channel 发送会 panic |
| **是否严格死锁** | ❌ **不会死锁** | Panic 会释放锁，打破循环 |
| **系统影响** | **极严重** | 消息发送 goroutine panic |

### 校准结论
**不是死锁，而是确定的 Panic**

关键纠正：
- ❌ 之前认为会死锁是错误的
- ✅ **Panic 会打破死锁循环**，因为 defer 会释放读锁
- ✅ 但代价是发送 goroutine 崩溃

---

## 场景三：NotifyDeletedUser 持锁关闭连接

### 代码路径
```go
// stream.go:54-63
func (a *API) NotifyDeletedUser(userID uint) error {
    a.lock.Lock()           // T1: 获取写锁
    defer a.lock.Unlock()   // T5: 释放
    if clients, ok := a.clients[userID]; ok {
        for _, client := range clients {
            client.Close()  // T2: 关闭连接
        }                      // T3: 关闭完成
        delete(a.clients, userID)  // T4: 清理 map
    }
    return nil
}

// client.go:43-48
func (c *client) Close() {
    c.once.Do(func() {
        c.conn.Close()      // 不回调 onClose
        close(c.write)      // 只关闭 channel
    })
}
```

### 并发分析

```
Goroutine A (NotifyDeletedUser):
    T1: a.lock.Lock() ✓           获取写锁
    T2: client.Close()
        T2.1: c.conn.Close() ✓
        T2.2: close(c.write) ✓   不调用 onClose！
    T3: delete(a.clients, userID)
    T4: a.lock.Unlock() ✓

其他并发行为:
    - 正在阻塞的 Notify(): 发现 channel 关闭 → panic
    - 新的 Notify(): 写锁释放后可以获取读锁，但 clients 已被删除
```

### 风险等级校准

| 评估项 | 结论 | 依据 |
|--------|------|------|
| **是否可恢复** | ❌ **不可恢复** | 正在发送的 goroutine panic |
| **是否触发 Panic** | ✅ **高概率触发** | 如果有 Notify 正在发送，会 panic |
| **是否严格死锁** | ❌ **不会死锁** | Close 不回调 remove，没有锁循环 |
| **系统影响** | **中等** | 可能导致 panic，但窗口较小 |

### 校准结论
**不会死锁，但可能导致并发 Panic**

关键点：
- `Close()` 不调用 `onClose`，没有锁循环
- 但如果此时有 goroutine 正在 `c.write <- msg` 阻塞，会 panic
- 窗口很小，只有持有写锁的短暂时间内

---

## 场景四：读协程错误触发 NotifyClose

### 调用链
```go
// client.go:61-75
func (c *client) startReading(pongWait time.Duration) {
    defer c.NotifyClose()  // 退出时触发
    for {
        if _, _, err := c.conn.NextReader(); err != nil {
            return  // 触发 defer
        }
    }
}
```

### 并发分析

```
正常情况 (没有并发 Notify):
    读错误 → NotifyClose → onClose → remove(Lock)
    此时没有读锁持有 → 获取写锁成功 ✓

并发情况 (Notify 正在执行):
    Goroutine A: Notify() 持有读锁，正在发送
    Goroutine B: 读错误 → NotifyClose → remove 需要写锁
    Goroutine B 阻塞等待写锁
    
    此时:
    - 如果 A 发送完成，释放读锁 → B 获取写锁成功 ✓
    - 如果 A 阻塞在发送，B 关闭 channel → A panic 释放锁 ✓
```

### 风险等级校准

| 评估项 | 结论 | 依据 |
|--------|------|------|
| **是否可恢复** | ✅ **可恢复** | 要么发送完成，要么 panic 释放锁 |
| **是否触发 Panic** | ⚠️ **可能触发** | 取决于是否有并发发送阻塞 |
| **是否严格死锁** | ❌ **不会死锁** | 总有一条路径能释放锁 |
| **系统影响** | **低-中** | 最坏情况是一个 goroutine panic |

### 校准结论
**不会死锁，最坏情况是 Panic**

---

## 场景五：写协程错误触发 NotifyClose

### 与场景四对比
与读协程错误的行为完全一致，都是：
1. 写错误触发 return
2. defer NotifyClose 执行
3. 需要获取写锁进行 remove

### 风险等级校准
与场景四完全相同：
- ❌ 不会死锁
- ⚠️ 可能触发 panic（有并发发送时）
- ✅ 最终锁总能释放

---

## 严格死锁条件复核

### Go 死锁四要素
死锁必须同时满足以下四个条件：

| 条件 | 是否满足 | 说明 |
|------|----------|------|
| 1. **互斥** | ✅ | 锁是互斥的 |
| 2. **持有并等待** | ⚠️ | Notify 持有读锁，等待 channel |
| 3. **不可剥夺** | ✅ | 锁不能被强制释放 |
| 4. **循环等待** | ❌ **不满足** | 没有形成闭环 |

### 为什么没有循环等待？

```
声称的循环:
    Notify(持有RLock) → 等待channel → NotifyClose → 等待Lock → 被Notify阻塞

实际打破点:
    NotifyClose 执行 close(c.write) → Notify 的发送 panic → defer 释放 RLock
    ↑ 这个链条被 Panic 打破了！
```

**校准结论**: 当前代码 **不存在严格意义上的死锁**

---

## 最终风险矩阵校准

| 场景 | 可恢复阻塞 | 触发 Panic | 严格死锁 | 实际风险 |
|------|-----------|------------|----------|----------|
| **持锁发送阻塞** | ❌ 不可恢复 (客户端卡住时) | ❌ 不直接 | ⚠️ 条件性 | 高 - 系统部分卡住 |
| **NotifyClose 回调锁** | ✅ 最终可恢复 | ✅ 确定触发 (有并发时) | ❌ 否 | 高 - goroutine panic |
| **NotifyDeletedUser 持锁关闭** | ✅ 很快恢复 | ⚠️ 可能 | ❌ 否 | 中 |
| **读协程错误关闭** | ✅ 可恢复 | ⚠️ 可能 | ❌ 否 | 低-中 |
| **写协程错误关闭** | ✅ 可恢复 | ⚠️ 可能 | ❌ 否 | 低-中 |

---

## 核心结论校准

### ✅ 纠正的错误结论

1. **之前认为有死锁是错误的**
   - 实际不存在严格死锁
   - Panic 机制会打破潜在的锁循环

2. **主要风险不是死锁，而是 Panic**
   - 并发场景下向已关闭 channel 发送会直接 panic
   - 这是比死锁更严重的问题（程序崩溃）

3. **持锁发送的主要问题是"不可恢复阻塞"**
   - 不是死锁，但当客户端卡住时同样会导致系统不可用

### ⚠️ 确认的真实风险

| 风险类型 | 确定性 | 影响 |
|----------|--------|------|
| **Goroutine Panic** | 高 (并发场景下几乎必然) | 消息发送 goroutine 崩溃 |
| **系统局部卡住** | 中 (客户端卡住时) | 单个用户的写锁操作无法进行 |
| **消息丢失** | 高 | channel 满时新消息被阻塞 |

### 🛠️ 优化建议优先级调整

| 优先级 | 优化项 | 解决的问题 |
|--------|--------|-----------|
| **P0** | Notify() 使用 select+default 非阻塞发送 | 防止 panic，防止阻塞 |
| **P1** | Notify() 释放锁后再发送 | 最小化锁持有时间 |
| **P1** | 增加 channel 容量 (8-16) | 降低阻塞概率 |
| **P2** | onClose 异步执行 | 代码整洁性优化，不是必须 |

---

## 最终校准结论

**当前代码不存在严格意义上的死锁，但存在严重的并发安全性问题：**

1. 🚫 **Panic 风险**: 并发情况下几乎必然发生 `send on closed channel` panic
2. 🚫 **不可恢复阻塞**: 客户端消费缓慢时会导致系统局部卡住
3. ✅ **无严格死锁**: Go 的 panic 机制会打破潜在的锁循环

**最优先修复**: 使用 `select` + `default` 进行非阻塞发送，这同时解决 panic 和阻塞问题。
