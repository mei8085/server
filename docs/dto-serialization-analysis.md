# Gotify Server DTO 响应序列化裁剪规则分析（修正版）

## 1. 架构总览

本项目采用 **Internal-External 双模型** 架构，数据在响应序列化阶段经历以下流程：

```
数据库层 (model.Xxx)  →  API 层手动转换  →  外部 DTO (model.XxxExternal)  →  Gin JSON 序列化  →  调用方
```

核心设计原则：**内部模型（Internal Model）** 包含完整的数据库字段（含敏感信息），**外部模型（External Model / DTO）** 仅暴露安全的、面向调用方的子集。两者之间通过 API 层的手写转换函数完成映射。

---

## 2. null 输出 vs 字段省略——关键边界

Go `encoding/json` 对可空字段的行为取决于两个因素的组合：

| 类型 | 是否有 omitempty | 零值/nil 时的 JSON 行为 |
|---|---|---|
| `*T`（指针） | **无** omitempty | 输出 `"field": null` |
| `*T`（指针） | **有** omitempty | 字段完全不出现在 JSON 中 |
| 值类型（uint/int/string 等） | **无** omitempty | 始终输出零值（0、""、false） |
| 值类型（uint/int/string 等） | **有** omitempty | 零值时字段完全不出现在 JSON 中 |
| `map`/`slice` | **无** omitempty | nil → `"field": null`；空值 → `"field": {}` / `"field": []` |
| `map`/`slice` | **有** omitempty | nil/空值时字段完全不出现在 JSON 中 |

**核心结论：指针类型不等于自动省略。只有 `omitempty` 才控制省略行为；没有 `omitempty` 的指针字段，nil 时输出 `null`。**

---

## 3. 逐 DTO 精确字段分类

### 3.1 Application（直接作为响应 DTO 使用）

定义位置：[model/application.go](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/model/application.go#L10-L68)

| 字段 | Go 类型 | json 标签 | nil/零值时的 JSON 行为 |
|---|---|---|---|
| ID | `uint` | `json:"id"` | 始终输出 |
| Token | `string` | `json:"token"` | 始终输出 |
| UserID | `uint` | `json:"-"` | **永不输出**（序列化时跳过） |
| Name | `string` | `json:"name"` | 始终输出 |
| Description | `string` | `json:"description"` | 始终输出（即使空字符串也输出 `""`） |
| Internal | `bool` | `json:"internal"` | 始终输出 |
| Image | `string` | `json:"image"` | 始终输出（经 [withResolvedImage](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/application.go#L445-L453) 处理后不会为空） |
| Messages | `[]MessageExternal` | `json:"-"` | **永不输出**（序列化时跳过） |
| DefaultPriority | `int` | `json:"defaultPriority"` | 始终输出（未设置时为 0） |
| CreatedAt | `time.Time` | `json:"createdAt"` | 始终输出 |
| **LastUsed** | **`*time.Time`** | **`json:"lastUsed"`（无 omitempty）** | **nil 时输出 `"lastUsed": null`** |
| SortKey | `string` | `json:"sortKey"` | 始终输出 |

> ⚠️ **`LastUsed` 是 `*time.Time` 且没有 `omitempty`**。当应用从未被使用过时，数据库中该字段为 NULL，反序列化后 Go 对象中为 nil，JSON 输出为 `"lastUsed": null`，而非省略该字段。

### 3.2 Client（直接作为响应 DTO 使用）

定义位置：[model/client.go](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/model/client.go#L10-L54)

| 字段 | Go 类型 | json 标签 | nil/零值时的 JSON 行为 |
|---|---|---|---|
| ID | `uint` | `json:"id"` | 始终输出 |
| Token | `string` | `json:"token"` | 始终输出 |
| UserID | `uint` | `json:"-"` | **永不输出**（序列化时跳过） |
| Name | `string` | `json:"name"` | 始终输出 |
| CreatedAt | `time.Time` | `json:"createdAt"` | 始终输出 |
| **LastUsed** | **`*time.Time`** | **`json:"lastUsed"`（无 omitempty）** | **nil 时输出 `"lastUsed": null`** |
| ElevatedUntil | `*time.Time` | `json:"elevatedUntil,omitempty"` | nil 时**省略**不出现在 JSON 中 |
| ExpiresAfterInactivitySeconds | `uint` | `json:"expiresAfterInactivitySeconds"` | 始终输出（0 表示永不过期） |
| ExpiresAt | `*time.Time` | `json:"expiresAt,omitempty"` | nil 时**省略**不出现在 JSON 中 |

> ⚠️ **`LastUsed` 与 `ElevatedUntil`/`ExpiresAt` 的行为不同**：
> - `LastUsed` 无 `omitempty` → nil 输出 `"lastUsed": null`
> - `ElevatedUntil` 有 `omitempty` → nil 时字段不存在
> - `ExpiresAt` 有 `omitempty` → nil 时字段不存在

### 3.3 UserExternal

定义位置：[model/user.go](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/model/user.go#L22-L45)

| 字段 | Go 类型 | json 标签 | nil/零值时的 JSON 行为 |
|---|---|---|---|
| ID | `uint` | `json:"id"` | 始终输出 |
| Name | `string` | `json:"name"` | 始终输出 |
| Admin | `bool` | `json:"admin"` | 始终输出 |
| CreatedAt | `time.Time` | `json:"createdAt"` | 始终输出 |

UserExternal **没有可空字段**，所有字段均为值类型且无 `omitempty`。密码哈希 `Pass`、关联列表 `Applications`/`Clients`/`Plugins` 在此 DTO 中**完全不定义**，从结构体层面杜绝泄露。

### 3.4 CurrentUserExternal

定义位置：[model/user.go](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/model/user.go#L95-L127)

| 字段 | Go 类型 | json 标签 | nil/零值时的 JSON 行为 |
|---|---|---|---|
| ID | `uint` | `json:"id"` | 始终输出 |
| Name | `string` | `json:"name"` | 始终输出 |
| Admin | `bool` | `json:"admin"` | 始终输出 |
| CreatedAt | `time.Time` | `json:"createdAt"` | 始终输出 |
| ClientID | `uint` | `json:"clientId,omitempty"` | 零值（0）时**省略** |
| ElevatedUntil | `*time.Time` | `json:"elevatedUntil,omitempty"` | nil 时**省略** |

### 3.5 MessageExternal

定义位置：[model/message.go](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/model/message.go#L23-L66)

| 字段 | Go 类型 | json 标签 | nil/零值时的 JSON 行为 |
|---|---|---|---|
| ID | `uint` | `json:"id"` | 始终输出 |
| ApplicationID | `uint` | `json:"appid"` | 始终输出 |
| Message | `string` | `json:"message"` | 始终输出 |
| Title | `string` | `json:"title"` | 始终输出（即使空字符串也输出 `""`） |
| **Priority** | **`*int`** | **`json:"priority"`（无 omitempty）** | 见下方详细分析 |
| Extras | `map[string]interface{}` | `json:"extras,omitempty"` | nil 时**省略** |
| Date | `time.Time` | `json:"date"` | 始终输出 |

> ⚠️ **`Priority` 是 `*int` 且没有 `omitempty`**。如果为 nil，理论上输出 `"priority": null`。但当前代码路径保证它**始终非 nil**——详见第 4 节。

### 3.6 PluginConfExternal

定义位置：[model/pluginconf.go](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/model/pluginconf.go#L23-L78)

| 字段 | Go 类型 | json 标签 | nil/零值时的 JSON 行为 |
|---|---|---|---|
| ID | `uint` | `json:"id"` | 始终输出 |
| CreatedAt | `time.Time` | `json:"createdAt"` | 始终输出 |
| Name | `string` | `json:"name"` | 始终输出 |
| Token | `string` | `json:"token"` | 始终输出 |
| ModulePath | `string` | `json:"modulePath"` | 始终输出 |
| Author | `string` | `json:"author,omitempty"` | 空字符串时**省略** |
| Website | `string` | `json:"website,omitempty"` | 空字符串时**省略** |
| License | `string` | `json:"license,omitempty"` | 空字符串时**省略** |
| Enabled | `bool` | `json:"enabled"` | 始终输出 |
| **Capabilities** | **`[]string`** | **`json:"capabilities"`（无 omitempty）** | **nil 时输出 `"capabilities": null`** |

> ⚠️ **`Capabilities` 是 `[]string` 且没有 `omitempty`**。如果 `inst.Supports().Strings()` 返回 nil 切片，输出 `"capabilities": null`；如果返回空切片，输出 `"capabilities": []`。

### 3.7 Paging

定义位置：[model/paging.go](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/model/paging.go#L8-L36)

| 字段 | Go 类型 | json 标签 | nil/零值时的 JSON 行为 |
|---|---|---|---|
| Next | `string` | `json:"next,omitempty"` | 空字符串时**省略** |
| Size | `int` | `json:"size"` | 始终输出 |
| Since | `uint` | `json:"since"` | 始终输出 |
| Limit | `int` | `json:"limit"` | 始终输出 |

### 3.8 OIDCExternalTokenResponse

定义位置：[model/oidc.go](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/model/oidc.go#L67-L81)

| 字段 | Go 类型 | json 标签 | nil/零值时的 JSON 行为 |
|---|---|---|---|
| Token | `string` | `json:"token"` | 始终输出 |
| **User** | **`*UserExternal`** | **`json:"user"`（无 omitempty）** | **nil 时输出 `"user": null`** |

> ⚠️ `User` 是 `*UserExternal` 指针且没有 `omitempty`。如果为 nil 会输出 `"user": null`。但当前代码中始终构造非 nil 的 UserExternal，所以实际不会出现 null。

---

## 4. 消息优先级 Priority 的完整生命周期

`MessageExternal.Priority` 是项目中**最特殊的可空字段**，其 `*int` 类型既服务于输入语义（区分"未设置"与"明确为零"），又影响了输出的 null 行为。

### 4.1 完整数据流

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│ 1. 客户端输入                                                                     │
│    请求 JSON: {"message": "hi", "priority": 2}  → Priority = &2 (非 nil)         │
│    请求 JSON: {"message": "hi"}                  → Priority = nil                │
│    请求 JSON: {"message": "hi", "priority": 0}  → Priority = &0 (非 nil)         │
└───────────────────────────────────────┬──────────────────────────────────────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ 2. CreateMessage 中的默认值填充                                                    │
│    [api/message.go L372-374]                                                      │
│    if message.Priority == nil {                                                   │
│        message.Priority = &application.DefaultPriority  // nil → 指向默认值       │
│    }                                                                              │
│    填充后: Priority 恒为非 nil                                                     │
└───────────────────────────────────────┬──────────────────────────────────────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ 3. toInternalMessage —— 外部 → 内部                                               │
│    [api/message.go L395-397]                                                      │
│    if msg.Priority != nil {                                                       │
│        res.Priority = *msg.Priority  // 解引用，存为 int 值类型                     │
│    }                                                                              │
│    内部 Message.Priority 类型为 int（值类型），永不为 nil                              │
│    若此处 Priority 仍为 nil（理论上不可能），res.Priority 保持零值 0                   │
└───────────────────────────────────────┬──────────────────────────────────────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ 4. 数据库存储                                                                      │
│    Message.Priority int → 数据库 INTEGER 列                                       │
│    值类型，始终有值（至少为 0）                                                       │
└───────────────────────────────────────┬──────────────────────────────────────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ 5. toExternalMessage —— 内部 → 外部                                               │
│    [api/message.go L411]                                                          │
│    Priority: &msg.Priority,  // 取地址，始终为非 nil 的 *int                        │
│    因为 msg.Priority 是 int 值类型，&msg.Priority 永远不会是 nil                     │
└───────────────────────────────────────┬──────────────────────────────────────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│ 6. JSON 序列化                                                                     │
│    json:"priority"（无 omitempty）                                                 │
│    因为 Priority 始终非 nil → 始终输出整数                                          │
│    输出: "priority": 2 或 "priority": 0                                            │
│    （不会出现 "priority": null，也不会省略该字段）                                    │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 为什么 Priority 使用 `*int` 而非 `int`

`*int` 类型的设计意图仅服务于**输入侧**：

- 输入 `{"message": "hi"}` → `Priority == nil` → 表示"未设置，使用应用默认优先级"
- 输入 `{"message": "hi", "priority": 0}` → `Priority == &0` → 表示"明确设置优先级为 0"

如果使用 `int` 类型，则无法区分"用户没传 priority"和"用户传了 priority: 0"这两种情况。

### 4.3 输出侧的实际行为

尽管 `json:"priority"` 没有 `omitempty`，理论上 nil 会输出 `"priority": null`，但：

1. **REST 响应**：[toExternalMessage](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/message.go#L405-L419) 中 `Priority: &msg.Priority` 对值类型取地址，**永远不为 nil**
2. **WebSocket 推送**：[stream.API.Notify](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/stream/stream.go#L83-L91) 接收的 `*MessageExternal` 同样经过 `toExternalMessage` 转换

**因此，在所有当前代码路径中，`Priority` 始终输出为整数值，不会出现 `null`，也不会被省略。**

但如果未来有人直接构造 `MessageExternal{Priority: nil}` 并序列化，由于没有 `omitempty`，结果将是 `"priority": null`，而非省略。这是结构体定义隐含的契约——**无 omitempty 的指针字段意味着：即使为 nil，也要在 JSON 中显式呈现为 null**。

---

## 5. null vs 省略 的完整对照表

### 5.1 输出 `null` 的字段（指针/切片/Map 类型 + 无 omitempty）

| DTO | 字段 | 类型 | json 标签 | 何时输出 null |
|---|---|---|---|---|
| Application | LastUsed | `*time.Time` | `json:"lastUsed"` | 应用从未被使用过时 |
| Client | LastUsed | `*time.Time` | `json:"lastUsed"` | 客户端从未被使用过时 |
| MessageExternal | Priority | `*int` | `json:"priority"` | **当前代码不会产生 null**（始终非 nil），但结构体定义允许 |
| PluginConfExternal | Capabilities | `[]string` | `json:"capabilities"` | 插件实例返回 nil 切片时 |
| OIDCExternalTokenResponse | User | `*UserExternal` | `json:"user"` | **当前代码不会产生 null**（始终非 nil），但结构体定义允许 |

### 5.2 省略不出现在 JSON 中的字段（有 omitempty）

| DTO | 字段 | 类型 | json 标签 | 省略条件 |
|---|---|---|---|---|
| CurrentUserExternal | ClientID | `uint` | `json:"clientId,omitempty"` | 非客户端认证时（零值 0） |
| CurrentUserExternal | ElevatedUntil | `*time.Time` | `json:"elevatedUntil,omitempty"` | 未提权 / 提权已过期 / Basic Auth 认证 |
| Client | ElevatedUntil | `*time.Time` | `json:"elevatedUntil,omitempty"` | 未提权 / 提权已过期（API 层清除过期值） |
| Client | ExpiresAt | `*time.Time` | `json:"expiresAt,omitempty"` | 永不过期的客户端（计算结果为 nil） |
| MessageExternal | Extras | `map[string]interface{}` | `json:"extras,omitempty"` | 无额外数据时 |
| Paging | Next | `string` | `json:"next,omitempty"` | 无下一页时 |
| PluginConfExternal | Author | `string` | `json:"author,omitempty"` | 插件未提供作者 |
| PluginConfExternal | Website | `string` | `json:"website,omitempty"` | 插件未提供网站 |
| PluginConfExternal | License | `string` | `json:"license,omitempty"` | 插件未提供许可证 |

### 5.3 永不输出的字段（json:"-"）

| DTO | 字段 | 说明 |
|---|---|---|
| Application | UserID | 用户 ID 不暴露给调用方 |
| Application | Messages | 关联消息不随应用返回 |
| Client | UserID | 用户 ID 不暴露给调用方 |

### 5.4 完全不进入 DTO 的字段（外部模型不定义）

| 内部模型 | 字段 | 原因 |
|---|---|---|
| User | Pass `[]byte` | 密码哈希永不返回 |
| User | Applications | 关联数据不随用户返回 |
| User | Clients | 关联数据不随用户返回 |
| User | Plugins | 关联数据不随用户返回 |
| PluginConf | Config `[]byte` | 内部配置不随列表返回 |
| PluginConf | Storage `[]byte` | 内部存储永不暴露 |
| PluginConf | UserID | 用户 ID 不暴露 |
| PluginConf | ApplicationID | 关联应用 ID 不暴露 |

---

## 6. 业务逻辑对可空字段的二次控制

API 层在序列化前对部分字段做了额外的条件判断，使某些字段在业务上不合理时被主动置为零值/nil，配合 `omitempty` 实现省略：

### 6.1 CurrentUserExternal 的条件填充

[GetCurrentUser](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/user.go#L129-L148)：

```go
result := &model.CurrentUserExternal{
    ID: user.ID, Name: user.Name, Admin: user.Admin, CreatedAt: user.CreatedAt,
    // ClientID = 0（零值），ElevatedUntil = nil
}
client := auth.GetClient(ctx)
if client != nil {
    result.ClientID = client.ID
    if client.ElevatedUntil != nil && time.Now().Before(*client.ElevatedUntil) {
        result.ElevatedUntil = client.ElevatedUntil
    }
}
```

| 场景 | ClientID | ElevatedUntil |
|---|---|---|
| Basic Auth 认证 | 0（omitempty → 省略） | nil（omitempty → 省略） |
| Client Token 认证，未提权 | client.ID（输出） | nil（omitempty → 省略） |
| Client Token 认证，提权有效 | client.ID（输出） | 时间指针（输出） |
| Client Token 认证，提权过期 | client.ID（输出） | nil（omitempty → 省略） |

### 6.2 Client 列表查询中的提权过期清除

[GetClients](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/client.go#L199-L204)：

```go
for _, client := range clients {
    if client.ElevatedUntil != nil && !now.Before(*client.ElevatedUntil) {
        client.ElevatedUntil = nil
    }
}
```

数据库中 `ElevatedUntil` 可能仍存储着已过期的时间戳，但 API 层在返回前主动置为 nil，配合 `json:"elevatedUntil,omitempty"` 使其不出现在 JSON 中。这确保了调用方不会收到语义上已失效的提权信息。

### 6.3 Client.ExpiresAt 的动态计算

[model/client.go](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/model/client.go#L56-L70)：

```go
func (c *Client) calculateExpiresAt() *time.Time {
    if c.ExpiresAfterInactivitySeconds == 0 {
        return nil
    }
    reference := c.CreatedAt
    if c.LastUsed != nil { reference = *c.LastUsed }
    expiry := reference.Add(time.Duration(c.ExpiresAfterInactivitySeconds) * time.Second)
    return &expiry
}
```

`ExpiresAt` 不是数据库持久化字段，而是每次读取时由 `PopulateExpiresAt()` 动态计算。当 `ExpiresAfterInactivitySeconds == 0` 时返回 nil，配合 `json:"expiresAt,omitempty"` 实现省略。

---

## 7. Application.LastUsed 和 Client.LastUsed 为何输出 null 而非省略

这是项目中**有意的设计选择**，两个 `LastUsed` 字段均使用 `*time.Time` 且**不带 omitempty**：

```go
// model/application.go L62
LastUsed *time.Time `json:"lastUsed"`

// model/client.go L39
LastUsed *time.Time `json:"lastUsed"`
```

**语义区别：**

- `"lastUsed": null` → 调用方可以明确知道"该字段存在但从未被使用"
- 字段省略 → 调用方无法区分"该字段不存在"和"该字段为空"

对于 `LastUsed`，显式输出 null 让客户端能区分"应用/客户端刚创建还没用过"和"数据缺失"的情况。这与 `ElevatedUntil`/`ExpiresAt` 的省略策略形成对比——后两者的"不存在"等价于"功能未启用"，省略更合理。

---

## 8. 裁剪规则总结

| 裁剪规则 | 触发机制 | 输出行为 | 典型字段 |
|---|---|---|---|
| 字段剔除 | 外部 DTO 不定义该字段 | 永不出现 | User.Pass, PluginConf.Config/Storage |
| `json:"-"` | Go 结构体标签 | 永不出现 | Application.UserID, Client.UserID |
| `*T` + 无 omitempty | 指针 + nil | **输出 null** | Application.LastUsed, Client.LastUsed |
| `*T` + omitempty | 指针 + nil | **字段省略** | Client.ElevatedUntil, Client.ExpiresAt, CurrentUserExternal.ElevatedUntil |
| 值类型 + omitempty | 零值 | **字段省略** | CurrentUserExternal.ClientID, Paging.Next |
| 值类型 + 无 omitempty | 零值 | **始终输出** | Application.DefaultPriority(0), Client.ExpiresAfterInactivitySeconds(0) |
| `string` + omitempty | 空字符串 | **字段省略** | PluginConfExternal.Author/Website/License |
| `map` + omitempty | nil | **字段省略** | MessageExternal.Extras |
| `[]T` + 无 omitempty | nil | **输出 null** | PluginConfExternal.Capabilities |
| 业务逻辑清除 | API 层置 nil/零值 | 配合 omitempty 省略 | 过期的 ElevatedUntil, 非 Client 认证的 ClientID |
| 类型转换 | Internal → External | 格式变化 | Message.Extras `[]byte` → `map[string]interface{}` |
