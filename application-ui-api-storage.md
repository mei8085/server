# 应用管理页创建/更新操作流程分析报告

## 一、整体流程概览

应用管理页的创建和更新操作遵循以下完整链路（从前端到数据库闭环）：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              前端层 (UI Layer)                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. 用户点击 Create/Edit 按钮                                                │
│  2. 打开 Add/UpdateApplicationDialog                                        │
│  3. 前端校验：name 非空 → 按钮可用                                           │
│  4. 提交表单 → 调用 AppStore.create/update                                   │
│  5. Axios 发送 HTTP 请求: POST /application 或 PUT /application/:id         │
│  6. 接收响应 → refresh 刷新列表 → Snack 提示成功/失败                        │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Gin 路由与中间件层 (Middleware)                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. 请求进入 Gin Engine → Logger + Recovery 中间件                           │
│  2. CORS 跨域处理 → Content-Type 设置为 application/json                    │
│  3. 鉴权中间件 authentication.RequireClient                                 │
│     - 读取 Token: Header / Query / Cookie / Authorization: Bearer          │
│     - 查询 Client 存在性 → 更新 LastUsed                                     │
│     - 注册认证信息到 Gin Context                                             │
│  4. 路由匹配: POST /application 或 PUT /application/:id                      │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           API 处理层 (API Handler)                           │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. ctx.Bind() 绑定请求体到 ApplicationParams                                │
│     - Gin binding 校验: Name `binding:"required"`                           │
│     - 校验失败 → 400 Bad Request + 错误详情                                  │
│  2. 创建操作: 生成 Token → 设置 UserID/Internal=false                        │
│  3. 更新操作: withID 解析 ID → 查询应用存在 → 校验权限                       │
│  4. 调用数据库层方法                                                         │
│  5. 捕获数据库错误 → 处理重复键/其他错误                                     │
│  6. 成功: 返回 200 + withResolvedImage 处理图片路径                          │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          数据库层 (Database Layer)                           │
├─────────────────────────────────────────────────────────────────────────────┤
│  创建操作:                                                                  │
│    1. 开启 Serializable 隔离级别事务                                        │
│    2. SortKey 为空 → 查询用户最后一个应用 → fracdex 生成新 SortKey          │
│    3. GORM tx.Create() 执行 INSERT                                          │
│    4. 事务提交 / 回滚                                                       │
│                                                                             │
│  更新操作:                                                                  │
│    1. GORM DB.Save() 执行 UPDATE (全字段保存)                               │
│    2. 无事务包裹                                                           │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          数据库约束层 (DB Constraints)                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. PRIMARY KEY: id 自增                                                    │
│  2. UNIQUE INDEX: uix_applications_token (Token 唯一)                       │
│  3. UNIQUE INDEX: uix_application_user_id_sort_key (UserID+SortKey 唯一)    │
│  4. 字段长度限制: token varchar(180), sort_key bytes(255)                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、密钥展示状态实现

### 2.1 展示组件位置
- **位置**：`ui/src/application/Applications.tsx:257`
- **组件**：`CopyableSecret` 组件

### 2.2 密钥展示机制
**文件**：`ui/src/common/CopyableSecret.tsx`

```typescript
const CopyableSecret = ({value, style}: IProps) => {
    const [visible, setVisible] = React.useState(false);
    const text = visible ? value : '•••••••••••••••';
    
    // 切换可见性
    const toggleVisibility = () => setVisible((b) => !b);
    
    // 复制到剪贴板
    const copyToClipboard = async () => {
        try {
            await navigator.clipboard.writeText(value);
            snackManager.snack('Copied to clipboard');
        } catch (error) {
            console.error('Failed to copy to clipboard:', error);
            snackManager.snack('Failed to copy to clipboard');
        }
    };
};
```

**状态说明**：
1. **默认状态**：密钥隐藏，显示为 `•••••••••••••••`（15个点）
2. **切换按钮**：眼睛图标，点击切换可见性
3. **可见状态**：显示真实token值，使用等宽字体 `fontFamily: 'monospace'`
4. **复制功能**：一键复制到剪贴板，有成功/失败提示

---

## 三、表单提交完整路径（前后端逐段对应）

### 3.1 创建操作完整路径

| 阶段 | 前端文件位置 | 后端文件位置 | 关键操作 |
|------|------------|------------|---------|
| **1. UI触发** | `ui/src/application/Applications.tsx:105-111` | - | 点击"Create Application"按钮，setCreateDialog(true) |
| **2. 对话框渲染** | `ui/src/application/Applications.tsx:159-164` | - | 渲染 AddApplicationDialog 组件 |
| **3. 表单输入与校验** | `ui/src/application/AddApplicationDialog.tsx:17-26` | - | 输入name/description/priority，name非空则按钮可用 |
| **4. Store层调用** | `ui/src/application/AppStore.ts:87-99` | - | `appStore.create(name, description, defaultPriority)` |
| **5. HTTP请求发送** | `ui/src/application/AppStore.ts:92-96` | - | `POST /application` 携带JSON body |
| **6. Gin路由匹配** | - | `router/router.go:189` | `app.POST("", applicationHandler.CreateApplication)` |
| **7. 鉴权中间件** | - | `auth/authentication.go:52-54` | `RequireClient` 验证Client Token |
| **8. 参数绑定校验** | - | `api/application.go:93-94` | `ctx.Bind(&applicationParams)` + `binding:"required"` |
| **9. 业务逻辑处理** | - | `api/application.go:95-108` | 生成Token、设置UserID、构建Application对象 |
| **10. 数据库操作** | - | `database/application.go:39-55` | 事务、SortKey生成、INSERT |
| **11. 响应处理** | - | `api/application.go:109` | `ctx.JSON(200, withResolvedImage(&app))` |
| **12. 前端刷新** | `ui/src/application/AppStore.ts:97` | - | `await this.refresh()` 刷新应用列表 |
| **13. 用户反馈** | `ui/src/application/AppStore.ts:98` | - | `this.snack('Application created')` |

### 3.2 更新操作完整路径

| 阶段 | 前端文件位置 | 后端文件位置 | 关键操作 |
|------|------------|------------|---------|
| **1. UI触发** | `ui/src/application/Applications.tsx:265-267` | - | 点击编辑图标，`setToUpdateApp(app)` |
| **2. 对话框渲染** | `ui/src/application/Applications.tsx:165-174` | - | 渲染 UpdateApplicationDialog，预填充初始值 |
| **3. 表单输入与校验** | `ui/src/application/UpdateApplicationDialog.tsx:27-35` | - | 修改字段，name非空则按钮可用 |
| **4. Store层调用** | `ui/src/application/AppStore.ts:74-84` | - | `appStore.update({id, name, description, defaultPriority})` |
| **5. HTTP请求发送** | `ui/src/application/AppStore.ts:81` | - | `PUT /application/{id}` 携带JSON body |
| **6. Gin路由匹配** | - | `router/router.go:192` | `app.PUT("/:id", applicationHandler.UpdateApplication)` |
| **7. 鉴权中间件** | - | `auth/authentication.go:52-54` | `RequireClient` 验证Client Token |
| **8. ID解析校验** | - | `api/internalutil.go:11-16` | `withID` 解析路径参数ID，无效则400 |
| **9. 存在性校验** | - | `api/application.go:254-257` | 查询应用是否存在，不存在则404 |
| **10. 权限校验** | - | `api/application.go:258` | `app.UserID == auth.GetUserID(ctx)`，不匹配则404 |
| **11. 参数绑定** | - | `api/application.go:259-260` | `ctx.Bind(&applicationParams)` |
| **12. 字段更新** | - | `api/application.go:261-266` | 只更新指定字段，SortKey非空才更新 |
| **13. 数据库操作** | - | `database/application.go:74-76` | `DB.Save(app)` 执行UPDATE |
| **14. 响应处理** | - | `api/application.go:268-272` | `ctx.JSON(200, withResolvedImage(app))` |
| **15. 前端刷新** | `ui/src/application/AppStore.ts:82` | - | `await this.refresh()` 刷新应用列表 |
| **16. 用户反馈** | `ui/src/application/AppStore.ts:83` | - | `this.snack('Application updated')` |

---

## 四、路由挂载与鉴权拦截详解

### 4.1 路由挂载配置
**文件**：`router/router.go:183-199`

```go
clientAuth := g.Group("")
{
    clientAuth.Use(authentication.RequireClient)  // 所有子路由都需要Client认证
    app := clientAuth.Group("/application")
    {
        app.GET("", applicationHandler.GetApplications)
        app.POST("", applicationHandler.CreateApplication)          // POST /application
        app.POST("/:id/image", applicationHandler.UploadApplicationImage)
        app.DELETE("/:id/image", applicationHandler.RemoveApplicationImage)
        app.PUT("/:id", applicationHandler.UpdateApplication)       // PUT /application/:id
    }
}
```

**关键点**：
1. `clientAuth` 路由组统一使用 `RequireClient` 中间件
2. `/application` 及其子路由都需要客户端认证
3. 删除操作需要额外的 `RequireElevatedClient` 权限（`router.go:224`）

### 4.2 鉴权拦截流程
**文件**：`auth/authentication.go:45-270`

#### RequireClient 中间件执行流程
```go
func (a *Auth) RequireClient(ctx *gin.Context) {
    // 按顺序尝试两种认证方式：User密码认证 → Client Token认证
    a.evaluateOr401(ctx, a.handleUser(), a.handleClient())
}
```

#### 1. handleClient 认证流程
```go
func (a *Auth) handleClient(checks ...) func(ctx *gin.Context) (authState, error) {
    return func(ctx *gin.Context) (authState, error) {
        // 1. 从4种途径读取Token
        token, isCookie := a.readTokenFromRequest(ctx)
        //   - Query参数: ?token=xxx
        //   - Header: X-Gotify-Key: xxx
        //   - Authorization: Bearer xxx
        //   - Cookie: gotify-client-token
        
        if token == "" { return authStateSkip, nil }
        
        // 2. 数据库查询Client是否存在
        client, err := a.DB.GetClientByToken(token)
        if client == nil { return authStateSkip, nil }
        
        // 3. 注册Client到Context（后续GetUserID使用）
        RegisterClient(ctx, client)
        
        // 4. 更新LastUsed时间（每5分钟更新一次）
        now := timeNow()
        if client.LastUsed == nil || client.LastUsed.Add(5*time.Minute).Before(now) {
            a.DB.UpdateClientTokensLastUsed([]string{client.Token}, &now)
            // Cookie模式则刷新Cookie
            if isCookie { SetCookie(ctx.Writer, client.Token, CookieMaxAge, a.SecureCookie) }
        }
        
        // 5. 执行额外检查（如Elevated权限）
        for _, check := range checks {
            if state, err := check(client); err != nil || state != authStateOk {
                return state, err
            }
        }
        
        return authStateOk, nil
    }
}
```

#### 2. 认证状态流转
```
请求到达
    ↓
readTokenFromRequest → 4种途径尝试读取Token
    ↓
Token为空 → authStateSkip → 尝试下一种认证方式
    ↓
Token非空 → 查询数据库
    ↓
Client不存在 → authStateSkip → 尝试下一种认证方式
    ↓
Client存在 → RegisterClient到Context
    ↓
更新LastUsed时间 → 设置Cookie
    ↓
执行权限检查 → authStateOk → ctx.Next() 进入业务处理
    ↓
所有认证方式都失败 → abort401() → 返回401
```

#### 3. 用户ID获取机制
**文件**：`auth/util.go:38-60`

```go
func GetUserID(ctx *gin.Context) uint {
    id := TryGetUserID(ctx)
    if id == nil { panic("token and user may not be null") }
    return *id
}

func TryGetUserID(ctx *gin.Context) *uint {
    info := getInfo(ctx)  // 从Context读取认证信息
    switch {
    case info.user != nil:   return &info.user.ID    // User密码认证
    case info.client != nil: return &info.client.UserID  // Client Token认证
    case info.app != nil:    return &info.app.UserID   // Application Token认证
    default: return nil
    }
}
```

---

## 五、创建与更新校验差异对比（含失败分支）

### 5.1 前端校验差异

#### 创建操作校验 (`AddApplicationDialog.tsx:22`)
```typescript
const submitEnabled = name.length !== 0;
```
- **初始值**：name='', description='', defaultPriority=0
- **仅校验**：name 非空
- **禁用状态**：name 为空时按钮禁用，Tooltip提示"name is required"
- **失败分支**：按钮点击无响应（被disabled阻止）

#### 更新操作校验 (`UpdateApplicationDialog.tsx:31`)
```typescript
const submitEnabled = name.length !== 0;
```
- **初始值**：使用应用现有数据预填充
- **校验逻辑**：与创建相同，仅校验 name 非空
- **禁用状态**：同创建逻辑
- **失败分支**：按钮点击无响应（被disabled阻止）

### 5.2 后端参数绑定与校验

#### 公共参数模型 (`api/application.go:39-57`)
```go
type ApplicationParams struct {
    Name            string `binding:"required"`  // 必填校验
    Description     string                       // 可选，无校验
    DefaultPriority int                          // 可选，无校验
    SortKey         string                       // 可选，无校验
}
```

**Gin binding 校验失败分支**：
```go
// 校验失败时，Gin将错误写入 ctx.Errors
// 由 error.Handler() 中间件统一处理
// error/handler.go:14-41
func Handler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()  // 执行业务Handler
        if len(c.Errors) > 0 {
            for _, e := range c.Errors {
                switch e.Type {
                case gin.ErrorTypeBind:
                    errs, ok := e.Err.(validator.ValidationErrors)
                    // 转换为可读的错误信息
                    // Field 'name' is required
                    writeError(c, strings.Join(stringErrors, "; "))
                }
            }
        }
    }
}
```

#### 创建操作特有校验与失败分支

**文件**：`api/application.go:92-111`

| 校验点 | 代码位置 | 失败条件 | 错误响应 |
|-------|---------|---------|---------|
| **参数绑定** | `api/application.go:93` | JSON格式错误 / Name为空 | 400 Bad Request + "Field 'name' is required" |
| **Token生成** | `api/application.go:100` | 极端情况无法生成唯一Token | 理论上不会失败，循环直到成功 |
| **UserID获取** | `api/application.go:101` | 认证中间件已保证，panic为防御性编程 | 500 Internal Server Error |
| **数据库插入** | `api/application.go:105` | 数据库连接错误 / 约束冲突 | handleApplicationError处理 |
| **SortKey唯一** | `api/application.go:486-490` | `errors.Is(err, gorm.ErrDuplicatedKey)` | 400 "sort key is not unique" |
| **其他DB错误** | - | 其他数据库错误 | 500 Internal Server Error |

#### 更新操作特有校验与失败分支

**文件**：`api/application.go:252-278`

| 校验点 | 代码位置 | 失败条件 | 错误响应 |
|-------|---------|---------|---------|
| **ID格式** | `api/internalutil.go:12-15` | 路径参数:id不是有效数字 | 400 "invalid id" |
| **应用存在** | `api/application.go:254` | ID对应的应用不存在 | 404 "app with id {id} doesn't exists" |
| **权限校验** | `api/application.go:258` | `app.UserID != auth.GetUserID(ctx)` | 404 "app with id {id} doesn't exists" |
| **参数绑定** | `api/application.go:259` | JSON格式错误 / Name为空 | 400 Bad Request |
| **数据库更新** | `api/application.go:268` | 数据库连接错误 / 约束冲突 | 同创建错误处理 |
| **SortKey唯一** | `api/application.go:486-490` | 新SortKey与其他应用重复 | 400 "sort key is not unique" |

---

## 六、数据库存储约束详解

### 6.1 Application 数据模型
**文件**：`model/application.go:10-62`

```go
type Application struct {
    ID              uint        `gorm:"primaryKey;autoIncrement" json:"id"`
    Token           string      `gorm:"type:varchar(180);uniqueIndex:uix_applications_token" json:"token"`
    UserID          uint        `gorm:"index;uniqueIndex:uix_application_user_id_sort_key,priority:1" json:"-"`
    Name            string      `gorm:"type:text" json:"name" binding:"required"`
    Description     string      `gorm:"type:text" json:"description"`
    Internal        bool        `json:"internal"`
    Image           string      `gorm:"type:text" json:"image"`
    DefaultPriority int         `json:"defaultPriority"`
    LastUsed        *time.Time  `json:"lastUsed"`
    SortKey         string      `gorm:"type:bytes;uniqueIndex:uix_application_user_id_sort_key,priority:2,length:255" json:"sortKey"`
}
```

### 6.2 数据库约束汇总

| 约束类型 | 约束名称 | 字段组合 | 作用 |
|---------|---------|---------|------|
| **主键约束** | PRIMARY | `id` | 每行唯一标识，自增 |
| **唯一索引** | `uix_applications_token` | `token` | 全局Token唯一，防止Token冲突 |
| **联合唯一索引** | `uix_application_user_id_sort_key` | `user_id` + `sort_key` | 同一用户下SortKey唯一，分数索引排序 |
| **外键约束** | - | `user_id` → `users.id` | 应用必须属于存在的用户，级联删除 |
| **长度约束** | - | `token`: varchar(180) | Token长度限制 |
| **长度约束** | - | `sort_key`: bytes(255) | SortKey长度限制 |

### 6.3 SortKey 生成机制（创建操作特有）

**文件**：`database/application.go:39-55`

```go
func (d *GormDatabase) CreateApplication(application *model.Application) error {
    return d.DB.Transaction(func(tx *gorm.DB) error {
        // SortKey 为空时自动生成
        if application.SortKey == "" {
            sortKey := ""
            // 1. 查询用户最后一个应用的SortKey（按sort_key降序取第一条）
            err := tx.Model(&model.Application{}).
                Select("sort_key").
                Where("user_id = ?", application.UserID).
                Order("sort_key DESC").
                Limit(1).
                Find(&sortKey).Error
            
            if err != nil && err != gorm.ErrRecordNotFound {
                return err
            }
            
            // 2. 使用 fracdex 在现有SortKey之后生成新的
            // 第一个应用: KeyBetween("", "") → 生成初始SortKey
            // 后续应用: KeyBetween(lastSortKey, "") → 在末尾追加
            application.SortKey, err = fracdex.KeyBetween(sortKey, "")
            if err != nil {
                return err
            }
        }
        
        // 3. 执行INSERT
        return tx.Create(application).Error
        
    }, &sql.TxOptions{Isolation: sql.LevelSerializable})  // Serializable隔离级别
}
```

**事务隔离级别说明**：
- **Serializable**：最高隔离级别，防止脏读、不可重复读、幻读
- **目的**：确保 SortKey 生成时不会并发冲突（两个请求同时查询并生成相同的SortKey）
- **代价**：性能开销较大，可能导致事务回滚需要重试

---

## 七、完整失败分支流程图

### 7.1 创建应用失败分支
```
POST /application
    ↓
┌─ 鉴权中间件失败 ──────────────────────────────────────┐
│  • 无Token / Token无效 → 401 "need valid access token"│
└───────────────────────────────────────────────────────┘
    ↓ 鉴权通过
┌─ 参数绑定失败 ────────────────────────────────────────┐
│  • JSON格式错误 → 400 Bad Request                    │
│  • Name字段为空 → 400 "Field 'name' is required"     │
└───────────────────────────────────────────────────────┘
    ↓ 参数校验通过
┌─ Token生成（理论上不会失败） ─────────────────────────┐
│  • 循环生成直到找到唯一Token                         │
└───────────────────────────────────────────────────────┘
    ↓
┌─ 数据库事务开始 ──────────────────────────────────────┐
│  • 连接错误 → 500 Internal Server Error              │
└───────────────────────────────────────────────────────┘
    ↓
┌─ SortKey自动生成 ─────────────────────────────────────┐
│  • 查询用户应用失败 → 事务回滚 → 500                  │
│  • fracdex生成失败 → 事务回滚 → 500                  │
└───────────────────────────────────────────────────────┘
    ↓
┌─ 执行INSERT ──────────────────────────────────────────┐
│  • UserID外键约束失败 → 500（理论上不会发生）        │
│  • Token唯一索引冲突 → 500（理论上不会发生）          │
│  • SortKey+UserID唯一索引冲突 → 事务回滚 → 400        │
│    "sort key is not unique"                           │
│  • 其他数据库错误 → 500                               │
└───────────────────────────────────────────────────────┘
    ↓ 事务提交
    ↓
返回 200 OK + Application对象
```

### 7.2 更新应用失败分支
```
PUT /application/:id
    ↓
┌─ 鉴权中间件失败 ──────────────────────────────────────┐
│  • 无Token / Token无效 → 401 "need valid access token"│
└───────────────────────────────────────────────────────┘
    ↓ 鉴权通过
┌─ 路径参数ID解析 ──────────────────────────────────────┐
│  • :id不是有效数字 → 400 "invalid id"                 │
└───────────────────────────────────────────────────────┘
    ↓ ID有效
┌─ 查询应用是否存在 ────────────────────────────────────┐
│  • 数据库查询失败 → 500                               │
│  • 应用不存在 → 404 "app with id {id} doesn't exists" │
└───────────────────────────────────────────────────────┘
    ↓ 应用存在
┌─ 权限校验 ────────────────────────────────────────────┐
│  • app.UserID != 当前用户ID → 404 "doesn't exists"    │
│    （安全考虑，不提示权限不足，只说不存在）            │
└───────────────────────────────────────────────────────┘
    ↓ 权限通过
┌─ 参数绑定失败 ────────────────────────────────────────┐
│  • JSON格式错误 → 400 Bad Request                    │
│  • Name字段为空 → 400 "Field 'name' is required"     │
└───────────────────────────────────────────────────────┘
    ↓ 参数校验通过
┌─ 字段更新（只更新允许的字段） ─────────────────────────┐
│  • Name / Description / DefaultPriority 直接更新      │
│  • SortKey 非空才更新                                 │
│  • Token / UserID / Internal 不可更新                 │
└───────────────────────────────────────────────────────┘
    ↓
┌─ 执行UPDATE ──────────────────────────────────────────┐
│  • SortKey+UserID唯一索引冲突 → 400 "sort key not unique"│
│  • 其他数据库错误 → 500                               │
└───────────────────────────────────────────────────────┘
    ↓
返回 200 OK + 更新后的Application对象
```

---

## 八、关键技术点

### 8.1 分数索引 (Fractional Indexing)
用于应用排序，在以下位置实现：
- 前端：`ui/src/application/AppStore.ts:63-66` - 使用 `generateKeyBetween` 拖拽排序
- 后端：`database/application.go:47` - 使用 `fracdex.KeyBetween` 自动生成

**优势**：无需重排所有元素，只需在两个元素之间生成新的排序键。

### 8.2 Token 生成机制
- **位置**：`api/application.go:100`
- **方法**：`auth.GenerateNotExistingToken(generateApplicationToken, a.applicationExists)`
- **特性**：循环生成直到找到数据库中不存在的唯一 Token

### 8.3 错误处理策略
- **SortKey 重复**：`api/application.go:486-488` - 返回 400 "sort key is not unique"
- **权限不足伪装**：更新操作中权限不足返回404而非403，防止信息泄露
- **统一错误格式**：所有错误返回统一JSON格式 `{error: "", errorCode: 400, errorDescription: ""}`

### 8.4 认证信息传递链
```
1. 请求携带Token → 2. RequireClient中间件
   → 3. 查询Client → 4. RegisterClient(ctx, client)
   → 5. 业务Handler调用 auth.GetUserID(ctx)
   → 6. 从Context的authentication对象读取client.UserID
```

---

## 九、前端Axios错误处理补充

**文件**：`ui/src/application/AppStore.ts`

虽然前端代码中没有显式的 `.catch()` 处理，但由于是 `async/await` 调用，异常会向上冒泡，由上层错误边界或全局错误处理器处理：

```typescript
// AppStore.ts 中的调用链
public create = async (name: string, description: string, defaultPriority: number): Promise<void> => {
    await axios.post(...)      // HTTP错误会抛出异常
    await this.refresh()       // 刷新失败也会抛出异常
    this.snack('Application created')  // 只有成功才执行
}

// 调用处: AddApplicationDialog.tsx
const submitAndClose = async () => {
    await fOnSubmit(name, description, defaultPriority)  // 异常未捕获，向上传递
    fClose()
}
```

**注意**：前端实际部署中通常有全局axios拦截器处理HTTP错误，统一处理401重定向登录、403权限提示等。
