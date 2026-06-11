# Gotify Server DTO 响应序列化裁剪规则分析

## 1. 架构总览

本项目采用 **Internal-External 双模型** 架构，数据在响应序列化阶段经历以下流程：

```
数据库层 (model.Xxx)  →  API 层手动转换  →  外部 DTO (model.XxxExternal)  →  Gin JSON 序列化  →  调用方
```

核心设计原则：**内部模型（Internal Model）** 包含完整的数据库字段（含敏感信息），**外部模型（External Model / DTO）** 仅暴露安全的、面向调用方的子集。两者之间通过 API 层的手写转换函数完成映射。

---

## 2. 裁剪规则分类

### 规则一：字段剔除（Field Exclusion）—— 敏感字段根本不进入 DTO

最核心的裁剪手段：内部模型中的敏感字段在外部 DTO 中**完全不存在**，从结构体层面杜绝泄露。

| 内部模型 | 被剔除的敏感字段 | 外部 DTO | 说明 |
|---|---|---|---|
| `model.User` | `Pass []byte` | `model.UserExternal` | 密码哈希永不返回 |
| `model.User` | `Applications []Application` | `model.UserExternal` | 用户关联的应用列表不随用户返回 |
| `model.User` | `Clients []Client` | `model.UserExternal` | 用户关联的客户端列表不随用户返回 |
| `model.User` | `Plugins []PluginConf` | `model.UserExternal` | 用户关联的插件配置不随用户返回 |
| `model.Application` | `UserID uint` (json:"-") | `model.Application`（直接用作 DTO） | 用户 ID 在 JSON 输出中被 `json:"-"` 标签隐藏 |
| `model.Application` | `Messages []MessageExternal` (json:"-") | `model.Application` | 关联消息不随应用返回 |
| `model.Client` | `UserID uint` (json:"-") | `model.Client`（直接用作 DTO） | 用户 ID 在 JSON 输出中被 `json:"-"` 标签隐藏 |
| `model.PluginConf` | `UserID uint` | `model.PluginConfExternal` | 用户 ID 不暴露 |
| `model.PluginConf` | `Config []byte` | `model.PluginConfExternal` | 插件内部配置不随列表返回（需单独 API 获取） |
| `model.PluginConf` | `Storage []byte` | `model.PluginConfExternal` | 插件内部存储不暴露 |
| `model.PluginConf` | `ApplicationID uint` | `model.PluginConfExternal` | 关联应用 ID 不暴露 |
| `model.Message` | `Extras []byte` (原始字节) | `model.MessageExternal` | 原始 JSON 字节被转为 `map[string]interface{}`，仅非空时输出 |

**关键代码位置：**

- User 转换函数 [toExternalUser](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/user.go#L484-L491)：只映射 `Name`, `Admin`, `ID`, `CreatedAt`，`Pass` 等敏感字段被完全忽略
- Application 的 `UserID` 和 `Messages` 字段使用 `json:"-"` 标签：[model/application.go](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/model/application.go#L23-L46)
- Client 的 `UserID` 字段使用 `json:"-"` 标签：[model/client.go](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/model/client.go#L23)
- PluginConf 到 PluginConfExternal 的映射：[api/plugin.go](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/plugin.go#L73-L84)，`Config`, `Storage`, `UserID`, `ApplicationID` 均不传递

### 规则二：`json:"-"` 标签 —— 序列化时跳过

部分内部模型直接被用作响应 DTO（如 `Application`、`Client`），此时通过 Go 的 `json:"-"` 结构体标签在序列化阶段剔除特定字段：

```go
// model/application.go
UserID uint `gorm:"index;..." json:"-"`       // 序列化时跳过
Messages []MessageExternal `gorm:"-" json:"-"` // 序列化时跳过

// model/client.go
UserID uint `gorm:"index" json:"-"`            // 序列化时跳过
```

### 规则三：`omitempty` 标签 —— 零值字段省略

Go 的 `json:"...,omitempty"` 标签使得字段在为零值（nil 指针、空字符串、零值、空 map/slice）时不输出到 JSON：

| DTO | 字段 | 类型 | omitempty 行为 |
|---|---|---|---|
| `UserExternal` | `Pass` | `string` | `pass,omitempty`：创建/更新请求中密码为空时不输出 |
| `CurrentUserExternal` | `ClientID` | `uint` | `clientId,omitempty`：非客户端认证时为零值，不输出 |
| `CurrentUserExternal` | `ElevatedUntil` | `*time.Time` | `elevatedUntil,omitempty`：未提权或提权过期时为 nil，不输出 |
| `Client` | `ElevatedUntil` | `*time.Time` | `elevatedUntil,omitempty`：未提权时不输出 |
| `Client` | `ExpiresAt` | `*time.Time` | `expiresAt,omitempty`：永不过期的客户端不输出 |
| `MessageExternal` | `Extras` | `map[string]interface{}` | `extras,omitempty`：无额外数据时不输出 |
| `Paging` | `Next` | `string` | `next,omitempty`：无下一页时为空字符串，不输出 |
| `PluginConfExternal` | `Author` | `string` | `author,omitempty`：插件未提供作者时不输出 |
| `PluginConfExternal` | `Website` | `string` | `website,omitempty`：插件未提供网站时不输出 |
| `PluginConfExternal` | `License` | `string` | `license,omitempty`：插件未提供许可证时不输出 |

### 规则四：类型转换 —— 内部格式转为外部友好格式

部分字段在内部模型与外部 DTO 之间做了类型变换，在转换过程中自然裁剪了不必要的信息：

| 内部类型 | 外部类型 | 转换逻辑 | 代码位置 |
|---|---|---|---|
| `Message.Extras []byte` | `MessageExternal.Extras map[string]interface{}` | 原始 JSON 字节反序列化为 map，仅在 `len(msg.Extras) != 0` 时才设置 | [toExternalMessage](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/message.go#L405-L419) |
| `Message.Priority int` | `MessageExternal.Priority *int` | 值类型转为指针类型，允许 omitempty 语义区分「未设置」与「零值」 | [toExternalMessage](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/message.go#L411) |
| `User.Pass []byte` | — | 密码哈希完全不进入外部模型 | [toExternalUser](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/user.go#L484-L491) |

---

## 3. 敏感字段处理详解

### 3.1 用户密码（Pass）

**处理方式：完全剔除 + 写入时单向加密**

- 内部模型 `model.User.Pass` 类型为 `[]byte`，存储 bcrypt/argon2id 哈希值
- 外部模型 `UserExternal` 根本没有 `Pass` 字段
- 写入时通过 `password.CreatePassword()` 进行单向哈希，原文不落库
- 更新用户时，若新密码为空字符串则保留旧密码哈希：[UpdateUserByID](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/user.go#L470-L472)
- `CreateUserExternal.Pass` 和 `UpdateUserExternal.Pass` 使用 `json:"pass,omitempty"`，在响应中省略密码字段

```
请求输入 → CreateUserExternal.Pass (string) → password.CreatePassword() → User.Pass ([]byte 哈希)
响应输出 → User.Pass 完全不映射 → UserExternal（无 Pass 字段）
```

### 3.2 Token（应用令牌 / 客户端令牌 / 插件令牌）

**处理方式：Token 在响应中保留，但仅限所有者可见**

- `Application.Token`：使用 `json:"token"` 正常输出，因为应用创建时需要返回 token 给调用方
- `Client.Token`：使用 `json:"token"` 正常输出，同理
- `PluginConfExternal.Token`：正常输出

**安全机制：** Token 的保护不在序列化层，而在认证中间件层：
- 认证中间件 [auth/authentication.go](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/auth/authentication.go) 确保只有令牌所有者才能访问对应资源
- `GetApplicationsByUser` / `GetClientsByUser` 通过 `userID` 过滤，确保用户只能看到自己的 token

### 3.3 UserID（用户标识）

**处理方式：`json:"-"` 标签在序列化时跳过**

- `Application.UserID` 和 `Client.UserID` 使用 `json:"-"` 标签
- 在 JSON 输出中完全不可见
- 但在 GORM 数据库操作中正常使用（`gorm:"index"` 等）

### 3.4 插件内部数据（Config / Storage）

**处理方式：独立 API 端点，列表查询不返回**

- `PluginConf.Config` 和 `PluginConf.Storage` 不进入 `PluginConfExternal`
- Config 需通过专用端点 `GET /plugin/{id}/config` 单独获取，且需验证插件所有权
- Storage 完全不暴露给前端

---

## 4. 可空字段处理详解

### 4.1 指针类型 + omitempty 模式

项目中对可空字段统一使用 **Go 指针类型** 配合 `omitempty` JSON 标签：

| 字段 | 类型 | 为 nil 时的 JSON 输出 | 有值时的 JSON 输出 |
|---|---|---|---|
| `Application.LastUsed` | `*time.Time` | 字段不出现 | `"lastUsed": "2019-01-01T00:00:00Z"` |
| `Client.LastUsed` | `*time.Time` | 字段不出现 | `"lastUsed": "2019-01-01T00:00:00Z"` |
| `Client.ElevatedUntil` | `*time.Time` | 字段不出现 | `"elevatedUntil": "2019-01-01T00:00:00Z"` |
| `Client.ExpiresAt` | `*time.Time` | 字段不出现 | `"expiresAt": "2019-01-01T00:00:00Z"` |
| `CurrentUserExternal.ElevatedUntil` | `*time.Time` | 字段不出现 | `"elevatedUntil": "2019-01-01T00:00:00Z"` |
| `CurrentUserExternal.ClientID` | `uint` + omitempty | 字段不出现 | `"clientId": 5` |
| `MessageExternal.Priority` | `*int` | 字段不出现 | `"priority": 2` |
| `MessageExternal.Extras` | `map[string]interface{}` | 字段不出现 | `"extras": {...}` |

### 4.2 业务逻辑层对可空字段的条件填充

除了 `omitempty` 的自动省略，部分可空字段在 API 层还有额外的业务逻辑控制：

#### CurrentUserExternal 的条件填充

[GetCurrentUser](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/user.go#L129-L148)：

```go
result := &model.CurrentUserExternal{
    ID:        user.ID,
    Name:      user.Name,
    Admin:     user.Admin,
    CreatedAt: user.CreatedAt,
    // ClientID 和 ElevatedUntil 初始为零值/nil
}
client := auth.GetClient(ctx)
if client != nil {
    result.ClientID = client.ID                                    // 仅客户端认证时填充
    if client.ElevatedUntil != nil && time.Now().Before(*client.ElevatedUntil) {
        result.ElevatedUntil = client.ElevatedUntil               // 仅提权未过期时填充
    }
}
```

- `ClientID`：仅在通过 client token 认证时才填充，Basic Auth 认证时不填充（零值 + omitempty = 不输出）
- `ElevatedUntil`：双重条件——(1) 客户端存在且 (2) 提权时间未过期

#### Client 列表查询中的提权过期清除

[GetClients](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/client.go#L193-L206)：

```go
now := time.Now()
for _, client := range clients {
    if client.ElevatedUntil != nil && !now.Before(*client.ElevatedUntil) {
        client.ElevatedUntil = nil  // 已过期的提权信息在序列化前清除
    }
}
```

数据库中 `ElevatedUntil` 可能仍有过期时间戳，但 API 层在返回前将其置为 nil，配合 `omitempty` 使其不出现在 JSON 中。

#### Client 的 ExpiresAt 动态计算

[model/client.go](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/model/client.go#L56-L70)：

```go
func (c *Client) PopulateExpiresAt() {
    c.ExpiresAt = c.calculateExpiresAt()
}

func (c *Client) calculateExpiresAt() *time.Time {
    if c.ExpiresAfterInactivitySeconds == 0 {
        return nil  // 永不过期 → nil → omitempty 不输出
    }
    // 基于 CreatedAt 或 LastUsed 计算过期时间
    reference := c.CreatedAt
    if c.LastUsed != nil {
        reference = *c.LastUsed
    }
    expiry := reference.Add(time.Duration(c.ExpiresAfterInactivitySeconds) * time.Second)
    return &expiry
}
```

- `ExpiresAt` 不是数据库持久化字段，而是每次读取时动态计算
- 当 `ExpiresAfterInactivitySeconds == 0` 时返回 nil，序列化时省略

#### Message Priority 的指针语义

[MessageExternal.Priority](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/model/message.go#L49) 使用 `*int` 而非 `int`：

- `nil` 表示未设置优先级（创建时应使用应用的默认优先级）
- 非 nil 值表示明确指定了优先级
- 创建消息时的回退逻辑：[CreateMessage](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/message.go#L372-L374)

```go
if message.Priority == nil {
    message.Priority = &application.DefaultPriority
}
```

#### Message Extras 的空值省略

[toExternalMessage](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/message.go#L414-L417)：

```go
if len(msg.Extras) != 0 {
    res.Extras = make(map[string]interface{})
    json.Unmarshal(msg.Extras, &res.Extras)
}
```

- 仅当 `Extras` 原始字节非空时才反序列化并赋值
- 未赋值时 `Extras` 为 nil map，配合 `omitempty` 不输出

---

## 5. 特殊场景：WebSocket 流式推送

WebSocket 连接通过 [stream.API.Notify](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/stream/stream.go#L83-L91) 推送消息：

```go
func (a *API) Notify(userID uint, msg *model.MessageExternal) {
    // msg 已经是 MessageExternal 类型，通过 toExternalMessage() 转换
    // 使用 conn.WriteJSON(v) 序列化，遵循相同的 json 标签规则
}
```

WebSocket 推送的消息与 REST API 返回的消息使用**同一套 DTO**（`MessageExternal`），因此裁剪规则完全一致。

---

## 6. 完整数据流示意图

以 `GET /user/{id}` 为例：

```
┌─────────────────────────────────────────────────────────────────────┐
│  Database Layer                                                     │
│  model.User {                                                       │
│      ID: 5,                                                         │
│      Name: "alice",                                                 │
│      Pass: []byte("$argon2id$v=19$m=..."),  ← 密码哈希(敏感)        │
│      Admin: true,                                                   │
│      CreatedAt: 2024-01-01,                                         │
│      Applications: [...],                    ← 关联数据(不随用户返回) │
│      Clients: [...],                        ← 关联数据(不随用户返回) │
│      Plugins: [...],                        ← 关联数据(不随用户返回) │
│  }                                                                  │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ toExternalUser()
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  DTO Layer (model.UserExternal)                                     │
│  {                                                                  │
│      ID: 5,                                                         │
│      Name: "alice",                                                 │
│      Admin: true,                                                   │
│      CreatedAt: 2024-01-01,                                         │
│      // Pass 已剔除                                                 │
│      // Applications 已剔除                                         │
│      // Clients 已剔除                                              │
│      // Plugins 已剔除                                              │
│  }                                                                  │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ ctx.JSON(200, ...)
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  JSON Response                                                      │
│  {                                                                  │
│      "id": 5,                                                       │
│      "name": "alice",                                               │
│      "admin": true,                                                 │
│      "createdAt": "2024-01-01T00:00:00Z"                            │
│  }                                                                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. 裁剪规则总结

| 裁剪规则 | 触发机制 | 典型场景 |
|---|---|---|
| **字段剔除** | 外部 DTO 不定义该字段 | 密码哈希、关联列表、插件内部数据 |
| **json:"-"** | Go 结构体标签 | UserID 在 Application/Client 响应中隐藏 |
| **omitempty** | Go 结构体标签 | 零值/nil 字段不在 JSON 中输出 |
| **指针类型** | `*T` 区分零值与未设置 | Priority、ElevatedUntil、ExpiresAt |
| **业务逻辑清除** | API 层在序列化前置 nil | 已过期的提权时间、未认证时的 ClientID |
| **类型转换** | Internal → External 转换函数 | `[]byte` → `map[string]interface{}` (Extras) |
| **动态计算** | 查询时计算而非持久化 | Client.ExpiresAt 基于 InactivitySeconds 计算 |
