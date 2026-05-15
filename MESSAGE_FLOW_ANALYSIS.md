# 实时消息推送系统架构分析报告

## 1. 系统架构概览

本系统采用**三层架构**实现实时消息推送：

| 层级 | 组件 | 职责 |
|------|------|------|
| **服务端** | `api/stream/` + `api/message.go` | WebSocket连接管理、消息广播、REST API |
| **前端状态层** | `WebSocketStore` + `MessagesStore` | WebSocket管理、消息存储与分发 |
| **UI展示层** | `Messages.tsx` | 消息列表渲染、交互控制 |

---

## 2. WebSocket通道机制

### 2.1 服务端连接管理

**核心组件**：`api/stream/stream.go`

```go
type API struct {
    clients     map[uint][]*client  // userID -> []client
    lock        sync.RWMutex
    pingPeriod  time.Duration
    pongTimeout time.Duration
    upgrader    *websocket.Upgrader
}
```

**连接管理流程**：

```
客户端请求 /stream
        ↓
    协议升级 (HTTP → WebSocket)
        ↓
    创建 client 实例
        ↓
    注册到 clients[userID]
        ↓
    启动读循环 + 写循环
```

**关键设计**：
- 使用 `sync.RWMutex` 保护并发访问
- 按 `userID` 分组管理客户端，支持多端登录
- 心跳机制：`pingPeriod` 定时发送ping，`pongTimeout` 超时断开

### 2.2 客户端读写循环

**核心组件**：`api/stream/client.go`

```go
type client struct {
    conn    *websocket.Conn
    write   chan *model.MessageExternal  // 消息写入通道
    userID  uint
    token   string
}
```

**写循环逻辑**：
```go
for {
    select {
    case message, ok := <-c.write:  // 接收待推送消息
        writeJSON(c.conn, message)
    case <-pingTicker.C:             // 定时心跳
        ping(c.conn)
    }
}
```

**读循环逻辑**：
- 设置读超时 `pongTimeout`
- 监听pong响应，刷新超时时间
- 忽略客户端发来的消息（单向推送）

### 2.3 前端WebSocket管理

**核心组件**：`ui/src/message/WebSocketStore.ts`

```typescript
public listen = (callback: (msg: IMessage) => void) => {
    const wsUrl = config.get('url').replace('http', 'ws').replace('https', 'wss');
    const ws = new WebSocket(wsUrl + 'stream');
    
    ws.onmessage = (data) => callback(JSON.parse(data.data));
    
    ws.onclose = () => {
        this.currentUser.tryAuthenticate().then(() => {
            setTimeout(() => this.listen(callback), 30000);  // 30秒后重连
        });
    };
};
```

**重连机制**：
1. 连接关闭时尝试重新认证
2. 认证成功后延迟30秒重新连接
3. 认证失败（401）时登出用户

---

## 3. 消息存储与分类机制

### 3.1 前端消息状态管理

**核心组件**：`ui/src/message/MessagesStore.ts`

**状态结构**：
```typescript
interface MessagesState {
    messages: IObservableArray<IMessage>;  // 消息数组
    hasMore: boolean;                      // 是否有更多消息
    nextSince: number;                     // 分页游标
    loaded: boolean;                       // 是否已加载
}

state: Record<string, MessagesState>  // appId -> MessagesState
```

**按应用分类存储**：

| AppID | 含义 |
|-------|------|
| `-1` | 全部消息视图 |
| `appId` | 特定应用的消息视图 |

### 3.2 消息推送与分发

**消息到达处理**：

```typescript
@action
public publishSingleMessage = (message: IMessage) => {
    // 更新全部消息视图
    if (this.exists(AllMessages)) {
        this.stateOf(AllMessages).messages.unshift(message);
    }
    // 更新对应应用视图
    if (this.exists(message.appid)) {
        this.stateOf(message.appid).messages.unshift(message);
    }
};
```

**关键特性**：
- 使用 `unshift` 将新消息插入数组头部（最新消息在前）
- 同时更新全局视图和应用视图，保持数据一致性

---

## 4. 消息去重机制

### 4.1 当前实现分析

**现有机制**：
1. **ID唯一性保障**：服务端数据库主键自增，每条消息ID唯一
2. **实时推送无重复**：WebSocket推送基于服务端事件触发，不会重复推送
3. **分页加载防重复**：使用 `since` 参数作为游标，按ID降序加载

**潜在问题**：离线重连时，WebSocket重连后可能收到已存在的消息

**当前代码中未实现的去重逻辑**：
```typescript
// MessagesStore.ts 中缺少对重复消息的检测
publishSingleMessage = (message: IMessage) => {
    // 缺少：检查消息是否已存在
    // 缺少：去重逻辑
    this.stateOf(appId).messages.unshift(message);
};
```

### 4.2 建议的去重方案

```typescript
@action
public publishSingleMessage = (message: IMessage) => {
    const allState = this.stateOf(AllMessages);
    // 检查是否已存在
    const existsInAll = allState.messages.some(m => m.id === message.id);
    
    if (!existsInAll && this.exists(AllMessages)) {
        allState.messages.unshift(message);
    }
    
    const appState = this.stateOf(message.appid);
    const existsInApp = appState.messages.some(m => m.id === message.id);
    
    if (!existsInApp && this.exists(message.appid)) {
        appState.messages.unshift(message);
    }
};
```

---

## 5. 离线回填机制

### 5.1 分页加载策略

**REST API接口**：

| 接口 | 用途 |
|------|------|
| `GET /message?since={id}` | 获取用户所有消息 |
| `GET /application/{id}/message?since={id}` | 获取指定应用消息 |

**分页参数**：

```typescript
interface IPaging {
    next?: string;   // 下一页URL
    since?: number;  // 下一页起始ID
    size: number;    // 当前页大小
    limit: number;   // 请求限制
}
```

### 5.2 加载流程

**初始化加载**：
```typescript
React.useEffect(() => {
    if (!messagesStore.loaded(appId)) {
        messagesStore.loadMore(appId);  // since=0
    }
}, [appId]);
```

**滚动加载更多**：
```typescript
const checkIfLoadMore = () => {
    if (messagesStore.canLoadMore(appId)) {
        messagesStore.loadMore(appId);  // since=nextSince
    }
};
```

**加载逻辑**：
```typescript
@action
public loadMore = async (appId: number) => {
    const state = this.stateOf(appId);
    const pagedResult = await this.fetchMessages(appId, state.nextSince);
    
    runInAction(() => {
        state.messages.replace([...state.messages, ...pagedResult.messages]);
        state.nextSince = pagedResult.paging.since ?? 0;
        state.hasMore = 'next' in pagedResult.paging;
        state.loaded = true;
    });
};
```

### 5.3 离线重连回填流程

#### 5.3.1 当前已实现的行为链路

**链路描述**：

1. **连接断开检测**：WebSocket 连接断开时触发 `ws.onclose` 回调
2. **认证状态检查**：若用户未登录则直接返回，不进行重连
3. **重新认证尝试**：调用 `currentUser.tryAuthenticate()` 验证身份有效性
4. **延迟重连**：认证成功后等待30秒，重新调用 `listen()` 建立连接
5. **认证失败处理**：若认证返回401则登出用户

**代码证据**：

1. **WebSocket断开与重连** (`ui/src/message/WebSocketStore.ts:25-48`)：
```typescript
ws.onclose = () => {
    this.wsActive = false;
    if (!this.currentUser.loggedIn) {
        return;
    }
    this.currentUser
        .tryAuthenticate()
        .then(() => {
            this.snack('WebSocket connection closed, trying again in 30 seconds.');
            setTimeout(() => this.listen(callback), 30000);
        })
        .catch((error: AxiosError) => {
            if (error?.response?.status === 401) {
                this.snack('Could not authenticate with client token, logging out.');
            }
        });
};
```

2. **分页加载机制** (`ui/src/message/MessagesStore.ts:48-71`)：
```typescript
@action
public loadMore = async (appId: number) => {
    const state = this.stateOf(appId);
    if (!state.hasMore || this.loading) {
        return Promise.resolve();
    }
    this.loading = true;
    try {
        const pagedResult = await this.fetchMessages(appId, state.nextSince).then(
            (resp) => resp.data
        );
        runInAction(() => {
            state.messages.replace([...state.messages, ...pagedResult.messages]);
            state.nextSince = pagedResult.paging.since ?? 0;
            state.hasMore = 'next' in pagedResult.paging;
            state.loaded = true;
        });
    } finally {
        this.loading = false;
    }
    return Promise.resolve();
};
```

3. **服务端分页查询** (`api/message.go:88-98`)：
```go
func (a *MessageAPI) GetMessages(ctx *gin.Context) {
    userID := auth.GetUserID(ctx)
    withPaging(ctx, func(params *pagingParams) {
        messages, err := a.DB.GetMessagesByUserSince(userID, params.Limit+1, params.Since)
        if success := successOrAbort(ctx, 500, err); !success {
            return
        }
        ctx.JSON(200, buildWithPaging(ctx, params, messages))
    })
}
```

**关键说明**：
- **无主动回填**：重连成功后不会主动拉取离线期间的消息
- **`since` 参数**：`loadMore` 使用 `state.nextSince`（上次加载的最后一条消息ID），而非本地最新消息ID
- **离线消息获取方式**：用户需手动点击"Refresh"按钮调用 `refreshByApp()`，或通过滚动加载更多历史消息

---

#### 5.3.2 可选改进

**当前限制**：

| 限制 | 说明 |
|------|------|
| 无主动回填 | 离线期间消息不会自动出现在列表中 |
| 无消息ID追踪 | 无法精确确定离线期间产生了哪些消息 |
| 无去重保护 | 重连后可能收到已存在的消息 |

**改进方案**：

1. **WebSocketStore 追踪最新消息ID**：
```typescript
export class WebSocketStore {
    private lastMessageId = 0;

    public listen = (callback: (msg: IMessage) => void, fetchMissed: (sinceId: number) => void) => {
        const ws = new WebSocket(wsUrl + 'stream');
        
        ws.onopen = () => {
            if (this.lastMessageId > 0) {
                fetchMissed(this.lastMessageId);
            }
        };

        ws.onmessage = (data) => {
            const msg = JSON.parse(data.data) as IMessage;
            this.lastMessageId = Math.max(this.lastMessageId, msg.id);
            callback(msg);
        };
        // ... 其余逻辑不变
    };
}
```

2. **MessagesStore 添加离线消息回填方法**：
```typescript
@action
public fetchMissedMessages = async (sinceId: number) => {
    if (this.exists(AllMessages)) {
        await this.fetchMissedForApp(AllMessages, sinceId);
    }
    Object.keys(this.state).forEach(appIdStr => {
        const appId = parseInt(appIdStr, 10);
        if (this.exists(appId) && appId !== AllMessages) {
            this.fetchMissedForApp(appId, sinceId);
        }
    });
};

private fetchMissedForApp = async (appId: number, sinceId: number) => {
    const state = this.stateOf(appId);
    const pagedResult = await this.fetchMessages(appId, sinceId).then(r => r.data);
    
    runInAction(() => {
        const existingIds = new Set(state.messages.map(m => m.id));
        const newMessages = pagedResult.messages.filter(m => !existingIds.has(m.id));
        state.messages.replace([...newMessages, ...state.messages]);
    });
};
```

**改进收益**：

| 改进项 | 收益 |
|--------|------|
| 主动回填 | 重连后自动获取离线消息，无需手动刷新 |
| 精确拉取 | 根据本地最新ID确定拉取范围 |
| 消息去重 | 避免重复消息 |

---

## 6. UI状态层协作

### 6.1 组件关系图

```
┌─────────────────────────────────────────────────────────────┐
│                        UI Layer                            │
│                  Messages.tsx (消息列表)                     │
└────────────────────────────┬────────────────────────────────┘
                             │ 使用
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                    State Layer                              │
│  ┌─────────────────┐        ┌───────────────────────────┐   │
│  │ WebSocketStore  │───────▶│    MessagesStore          │   │
│  │ - 连接管理       │        │ - 按appId存储消息         │   │
│  │ - 自动重连       │        │ - 分页加载               │   │
│  │ - 消息回调       │        │ - 待删除队列管理         │   │
│  └─────────────────┘        └───────────────────────────┘   │
└────────────────────────────┬────────────────────────────────┘
                             │ 监听
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                    API Layer                               │
│              WebSocket /stream + REST /message             │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 数据流

| 方向 | 数据 | 传输方式 |
|------|------|----------|
| 服务端 → 前端 | 新消息 | WebSocket |
| 前端 → 服务端 | 加载历史 | REST GET |
| 前端 → 服务端 | 删除消息 | REST DELETE |
| 前端内部 | 状态更新 | MobX observable |

---

## 7. 刷新功能实现

### 7.1 按应用刷新

```typescript
@action
public refreshByApp = async (appId: number) => {
    this.clearAll();      // 清空所有状态
    this.loadMore(appId); // 重新加载
};
```

### 7.2 时间维度刷新

通过分页机制隐式支持：
- `since=0` 加载最新消息
- 滚动自动加载更早的历史消息

### 7.3 UI触发点

```tsx
<Button onClick={() => messagesStore.refreshByApp(appId)}>
    Refresh
</Button>
```

### 7.4 未读维度刷新

**当前实现状态**：**未实现**

**代码证据**：

1. **服务端模型缺失** (`model/message.go`)：
```go
type Message struct {
    ID            uint
    ApplicationID uint
    Message       string
    Title         string
    Priority      int
    Extras        []byte
    Date          time.Time
    // 缺失：IsRead bool
    // 缺失：ReadAt time.Time
}
```

2. **前端类型缺失** (`ui/src/types.ts`)：
```typescript
interface IMessage {
    id: number;
    appid: number;
    message: string;
    title: string;
    priority: number;
    date: string;
    image?: string;
    extras?: IMessageExtras;
    // 缺失：isRead?: boolean;
}
```

3. **MessagesStore无未读状态管理** (`ui/src/message/MessagesStore.ts`)：
```typescript
interface MessagesState {
    messages: IObservableArray<IMessage>;
    hasMore: boolean;
    nextSince: number;
    loaded: boolean;
    // 缺失：unreadCount: number;
}
```

**功能缺口分析**：

| 维度 | 缺口描述 | 影响 |
|------|----------|------|
| **数据模型** | 缺少 `isRead` / `readAt` 字段 | 无法追踪消息阅读状态 |
| **状态管理** | 缺少未读计数统计 | 无法展示未读消息数量 |
| **API支持** | 缺少按未读筛选接口 | 无法实现未读消息过滤 |
| **UI交互** | 缺少标记已读功能 | 用户无法清除未读状态 |

**业务影响**：
- 用户无法快速定位未读消息
- 无法实现"未读消息数量"徽章提示
- 无法支持"只看未读"筛选模式
- 多端登录时无法同步已读状态

---

## 8. 代码优化建议

### 8.1 缺失的消息去重

**问题**：`publishSingleMessage` 未检查重复消息

**优化方案**：见 4.2 节

### 8.2 离线消息回填完整性

**问题**：WebSocket重连后缺少主动回填机制

**优化方案**：
```typescript
class WebSocketStore {
    private lastMessageId = 0;
    
    public listen = (callback: (msg: IMessage) => void) => {
        const ws = new WebSocket(wsUrl + 'stream');
        
        ws.onopen = () => {
            // 重连后主动拉取离线消息
            this.fetchMissedMessages(this.lastMessageId);
        };
        
        ws.onmessage = (data) => {
            const msg = JSON.parse(data.data);
            this.lastMessageId = Math.max(this.lastMessageId, msg.id);
            callback(msg);
        };
    };
}
```

### 8.3 批量删除优化

**当前实现**：
```typescript
executePendingDeletes = () =>
    Array.from(this.pendingDeletes.values())
        .forEach(({message}) => this.removeSingle(message));
```

**优化方案**：批量API减少请求
```typescript
executePendingDeletes = async () => {
    const ids = Array.from(this.pendingDeletes.keys());
    await axios.delete(config.get('url') + 'message/batch', {
        data: {ids}
    });
    // 本地删除
};
```

---

## 9. 总结

| 维度 | 当前状态 | 评价 |
|------|----------|------|
| **WebSocket通道** | 稳定，支持心跳和重连 | 良好 |
| **消息分类** | 按appId分组存储 | 良好 |
| **消息去重** | 依赖服务端ID唯一性 | 需增强 |
| **离线回填** | 依赖分页加载 | 需增强主动回填 |
| **UI刷新** | 支持按应用刷新 | 良好 |

**核心亮点**：
- MobX响应式状态管理，UI自动更新
- 服务端按用户分组广播，安全性高
- 分页加载支持大数据量
- 自动重连机制保证可靠性