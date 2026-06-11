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

#### 3.8.1 OIDC 响应中 User.CreatedAt 的零值行为（重要）

OIDC 外部 token 交换响应的 User 对象**不是通过 `toExternalUser()` 转换**，而是在 [ExternalTokenHandler](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/oidc.go#L380-L383) 中手动构造的：

```go
ctx.JSON(http.StatusOK, &model.OIDCExternalTokenResponse{
    Token: client.Token,
    User:  &model.UserExternal{ID: user.ID, Name: user.Name, Admin: user.Admin},
})
```

只显式赋值了 **三个字段**：`ID`、`Name`、`Admin`。`CreatedAt` 没有被赋值，保持 `time.Time` 的零值（Go 中的零时间：`0001-01-01T00:00:00Z`）。

由于 `UserExternal.CreatedAt` 的 json 标签是 `json:"createdAt"`（**无 omitempty**），即使是零值也会被序列化输出。

| 字段 | 理论行为 | OIDC 响应中的实际行为 |
|---|---|---|
| CreatedAt | time.Time 零值 → `"0001-01-01T00:00:00Z"` | **始终输出零时间**（与其他 UserExternal 响应不一致） |

> 🔍 **代码不一致性提示**：其他返回 UserExternal 的端点（如 `GET /user/{id}`、`GET /current/user`）均通过 `toExternalUser()` 或完整构造，正确携带了实际的 `CreatedAt`。唯独 OIDC 外部 token 交换响应遗漏了 `CreatedAt` 字段的赋值，导致返回无意义的零时间。

---

## 4. Token 字段的暴露边界与鉴权过滤机制

尽管 `Application.Token`、`Client.Token`、`PluginConfExternal.Token` 等 token 字段在 DTO 中直接明文展示，但它们的暴露范围受到**四层防护**的严格约束。

### 4.1 第一层：认证中间件（Authentication Middleware）

所有可能返回 token 的 API 端点都受到认证中间件的保护。

| 中间件 | 位置 | 保护的端点 | 说明 |
|---|---|---|---|
| `RequireClient` | [auth/authentication.go L52-54](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/auth/authentication.go#L52-L54) | 应用列表、客户端列表、插件列表、消息查询等 | 客户端 token 或 Basic Auth 皆可 |
| `RequireAdmin` | [auth/authentication.go L46-48](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/auth/authentication.go#L46-L48) | 用户管理、获取指定用户 | 管理员权限，支持 user Basic Auth 或 elevated client token |
| `RequireElevatedClient` | [auth/authentication.go L57-59](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/auth/authentication.go#L57-L59) | 删除应用/客户端、修改密码等 | 需提权的敏感操作 |
| `RequireApplicationToken` | [auth/authentication.go L62-76](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/auth/authentication.go#L62-L76) | `POST /message`（发送消息） | 仅应用 token 可访问，用户 auth 会被 403 拒绝 |

认证成功后，认证信息通过 `RegisterUser` / `RegisterClient` / `RegisterApplication` 存储到 gin context 中：[auth/util.go L16-29](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/auth/util.go#L16-L29)。

### 4.2 第二层：用户 ID 推导

无论以何种方式认证，都可以通过 `auth.GetUserID(ctx)` 推导所属用户：

- **User 认证**（Basic Auth）→ `info.user.ID`
- **Client 认证** → `info.client.UserID`
- **Application 认证** → `info.app.UserID`

[auth/util.go L39-60](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/auth/util.go#L39-L60)

这确保了后续的数据查询始终有一个明确的用户上下文。

### 4.3 第三层：数据库层按用户过滤

列表查询通过 `GetXxxByUser(userID)` 方法在 SQL 层面过滤，确保用户只能看到自己的数据。

| 数据库方法 | 位置 | 过滤逻辑 |
|---|---|---|
| `GetApplicationsByUser(userID)` | [database/application.go L63-66](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/database/application.go#L63-L66) | `WHERE user_id = ?` |
| `GetClientsByUser(userID)` | [database/client.go L45-48](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/database/client.go#L45-L48) | `WHERE user_id = ?` |
| `GetMessagesByUser(userID)` | [database/message.go L27-31](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/database/message.go#L27-L31) | `JOIN applications ON app.id = messages.application_id WHERE apps.user_id = ?` |
| `GetPluginsByUser(userID)` | [database/plugin.go](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/database/plugin.go) | `WHERE user_id = ?` |

示例（应用列表过滤）：

```go
// database/application.go
func (d *GormDatabase) GetApplicationsByUser(userID uint) ([]*model.Application, error) {
    var apps []*model.Application
    err := d.DB.Where("user_id = ?", userID).Order("sort_key asc").Find(&apps).Error
    return apps, err
}
```

### 4.4 第四层：API 层的所有权校验

单个资源的操作（读取/更新/删除）在 API 层进行所有权二次校验——先通过 ID 查出实体，再比较 `UserID` 是否匹配。

| 操作 | 校验位置 | 校验逻辑 |
|---|---|---|
| 获取单条消息 | [api/message.go L186](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/message.go#L186) | `app.UserID == auth.GetUserID(ctx)` |
| 删除单条消息 | [api/message.go L264](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/message.go#L264) | `application.UserID == auth.GetUserID(ctx)` |
| 获取单个应用 | [api/application.go L192](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/application.go#L192) | `app.UserID == auth.GetUserID(ctx)` |
| 更新应用 | [api/application.go L258](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/application.go#L258) | `app.UserID == auth.GetUserID(ctx)` |
| 删除应用 | [api/application.go L333](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/application.go#L333) | `app.UserID == auth.GetUserID(ctx)` |
| 获取单个客户端 | [api/client.go L98](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/client.go#L98) | `client.UserID == auth.GetUserID(ctx)` |
| 更新客户端 | [api/client.go L251](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/client.go#L251) | `client.UserID == auth.GetUserID(ctx)` |
| 删除客户端 | [api/client.go L310](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/client.go#L310) | `client.UserID != auth.GetUserID(ctx)` → 404 |

**通用模式**：

```go
func handler(ctx *gin.Context) {
    id := parseID(ctx)
    entity, err := a.DB.GetEntityByID(id)
    if err != nil || entity == nil {
        // 返回 404
    }
    if entity.UserID != auth.GetUserID(ctx) {
        // 返回 403 或 404（隐蔽性）
    }
    // 正常处理，entity.Token 也会正常输出给调用方
}
```

这种"先查再校验"的模式保证了即使攻击者能猜到实体 ID，也无法越权访问他人的 token。

### 4.5 特殊场景：应用 token 只能"用"不能"见"

一个值得注意的设计：应用 token（Application Token）的使用者是应用本身，但应用 token 持有者**看不到自己的 token**。

- `POST /message` 用应用 token 认证 → 可以发送消息，但看不到应用详情
- 查看应用列表/详情 → 需要 client token 或 Basic Auth 认证 → 才能看到应用的 token

这形成了一条规则：**token 的展示（DTO 中的 token 字段）受客户端/user 认证保护，而 token 的使用（作为认证凭证）是另一回事。**

### 4.6 特殊场景：OIDC 外部 token 响应无鉴权

`POST /auth/oidc/external/token` 端点是**唯一无认证中间件保护**但返回 token 的端点。

它的安全机制是：
1. 通过 `req.State` 参数查找已存在的 OIDC session（内部状态，由 `/auth/oidc/external/authorize` 生成）
2. state 是一次性的，使用后即销毁（`popPendingSession`）
3. 调用方必须持有有效的 `code` 和 `code_verifier`（PKCE 流程）
4. token 通过 OIDC 服务商验证后，才能换取 Gotify 客户端 token

### 4.7 四层防护总结

```
┌─────────────────────────────────────────────────────────────────┐
│ 第一层：认证中间件                                                 │
│   RequireClient / RequireAdmin / RequireApplicationToken        │
│   → 无有效凭证直接 401/403                                        │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 第二层：用户 ID 推导                                               │
│   auth.GetUserID(ctx)                                            │
│   → 从 user/client/app 认证中统一推出 userID                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 第三层：数据库层按用户过滤（列表查询）                                │
│   GetApplicationsByUser(userID) / GetClientsByUser(userID)      │
│   → SQL WHERE 条件只返回当前用户的数据                               │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 第四层：API 层所有权校验（单个实体操作）                              │
│   entity.UserID == auth.GetUserID(ctx)                          │
│   → 越权访问返回 403/404                                          │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
                    DTO 中的 token 字段可见
                    （只能看到自己的 token）
```

---

## 5. 消息优先级 Priority 的完整生命周期

`MessageExternal.Priority` 是项目中**最特殊的可空字段**，其 `*int` 类型既服务于输入语义（区分"未设置"与"明确为零"），又影响了输出的 null 行为。

### 5.1 完整数据流

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

### 5.2 为什么 Priority 使用 `*int` 而非 `int`

`*int` 类型的设计意图仅服务于**输入侧**：

- 输入 `{"message": "hi"}` → `Priority == nil` → 表示"未设置，使用应用默认优先级"
- 输入 `{"message": "hi", "priority": 0}` → `Priority == &0` → 表示"明确设置优先级为 0"

如果使用 `int` 类型，则无法区分"用户没传 priority"和"用户传了 priority: 0"这两种情况。

### 5.3 输出侧的实际行为

尽管 `json:"priority"` 没有 `omitempty`，理论上 nil 会输出 `"priority": null`，但：

1. **REST 响应**：[toExternalMessage](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/message.go#L405-L419) 中 `Priority: &msg.Priority` 对值类型取地址，**永远不为 nil**
2. **WebSocket 推送**：[stream.API.Notify](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/stream/stream.go#L83-L91) 接收的 `*MessageExternal` 同样经过 `toExternalMessage` 转换

**因此，在所有当前代码路径中，`Priority` 始终输出为整数值，不会出现 `null`，也不会被省略。**

但如果未来有人直接构造 `MessageExternal{Priority: nil}` 并序列化，由于没有 `omitempty`，结果将是 `"priority": null`，而非省略。这是结构体定义隐含的契约——**无 omitempty 的指针字段意味着：即使为 nil，也要在 JSON 中显式呈现为 null**。

---

## 6. null vs 省略 的完整对照表

### 6.1 输出 `null` 的字段（指针/切片/Map 类型 + 无 omitempty）

| DTO | 字段 | 类型 | json 标签 | 何时输出 null |
|---|---|---|---|---|
| Application | LastUsed | `*time.Time` | `json:"lastUsed"` | 应用从未被使用过时 |
| Client | LastUsed | `*time.Time` | `json:"lastUsed"` | 客户端从未被使用过时 |
| MessageExternal | Priority | `*int` | `json:"priority"` | **当前代码不会产生 null**（始终非 nil），但结构体定义允许 |
| PluginConfExternal | Capabilities | `[]string` | `json:"capabilities"` | 插件实例返回 nil 切片时 |
| OIDCExternalTokenResponse | User | `*UserExternal` | `json:"user"` | **当前代码不会产生 null**（始终非 nil），但结构体定义允许 |

### 6.2 省略不出现在 JSON 中的字段（有 omitempty）

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

### 6.3 永不输出的字段（json:"-"）

| DTO | 字段 | 说明 |
|---|---|---|
| Application | UserID | 用户 ID 不暴露给调用方 |
| Application | Messages | 关联消息不随应用返回 |
| Client | UserID | 用户 ID 不暴露给调用方 |

### 6.4 完全不进入 DTO 的字段（外部模型不定义）

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

## 7. 业务逻辑对可空字段的二次控制

API 层在序列化前对部分字段做了额外的条件判断，使某些字段在业务上不合理时被主动置为零值/nil，配合 `omitempty` 实现省略：

### 7.1 CurrentUserExternal 的条件填充

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

### 7.2 Client 列表查询中的提权过期清除

[GetClients](file:///d:/fz/0601-1/solo-dogfeeding/code/12-server/api/client.go#L199-L204)：

```go
for _, client := range clients {
    if client.ElevatedUntil != nil && !now.Before(*client.ElevatedUntil) {
        client.ElevatedUntil = nil
    }
}
```

数据库中 `ElevatedUntil` 可能仍存储着已过期的时间戳，但 API 层在返回前主动置为 nil，配合 `json:"elevatedUntil,omitempty"` 使其不出现在 JSON 中。这确保了调用方不会收到语义上已失效的提权信息。

### 7.3 Client.ExpiresAt 的动态计算

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

## 8. Application.LastUsed 和 Client.LastUsed 为何输出 null 而非省略

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

## 9. 裁剪规则总结

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
