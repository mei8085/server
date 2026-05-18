# 后台管理员用户管理协作链路详解

本文档深入解析 Gotify 后台管理员对普通用户执行创建、修改密码、删除等操作时，前端表单、后端鉴权校验、密码哈希、权限位字段之间的完整协作链路。

---

## 一、核心数据模型层

### 1.1 用户实体分层设计

系统采用**内部存储模型**与**外部传输DTO**分离的设计模式：

```go
// model/user.go

// 内部存储模型 - 数据库持久化
type User struct {
    ID           uint   `gorm:"primaryKey;autoIncrement"`
    Name         string `gorm:"type:varchar(180);uniqueIndex:uix_users_name"`
    Pass         []byte // bcrypt 哈希后的密码，永不返回给前端
    Admin        bool   // 权限位字段：true=管理员，false=普通用户
    Applications []Application
    Clients      []Client
    Plugins      []PluginConf
}

// 外部传输模型 - API交互
type UserExternal struct {
    ID    uint   `json:"id"`
    Name  string `json:"name"`
    Admin bool   `json:"admin"` // 权限位暴露给前端
}

// 创建用户专用DTO
type CreateUserExternal struct {
    Name  string `binding:"required" json:"name"`
    Admin bool   `json:"admin"`
    Pass  string `binding:"required" json:"pass"` // 明文密码，仅用于创建
}

// 更新用户专用DTO
type UpdateUserExternal struct {
    Name  string `binding:"required" json:"name"`
    Admin bool   `json:"admin"`
    Pass  string `json:"pass"` // 可选，空字符串表示不修改密码
}
```

**设计要点**：
- `Pass` 字段在内部模型中为 `[]byte` 类型，存储 bcrypt 哈希值
- 外部 DTO 中 `Pass` 为 `string` 类型，仅在创建/更新时传输明文
- `UserExternal` 完全剥离密码字段，确保数据安全
- `Admin` 布尔字段作为唯一权限位，控制用户是否可访问管理接口

---

## 二、密码哈希机制

### 2.1 bcrypt 密码处理

```go
// auth/password/password.go

// 创建哈希密码 - 强度参数从配置读取
func CreatePassword(pw string, strength int) []byte {
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(pw), strength)
    if err != nil {
        panic(err)
    }
    return hashedPassword
}

// 验证密码
func ComparePassword(hashedPassword, password []byte) bool {
    return bcrypt.CompareHashAndPassword(hashedPassword, password) == nil
}
```

**关键特性**：
- 使用 Go 官方推荐的 `golang.org/x/crypto/bcrypt` 库
- 哈希强度（cost）从配置文件 `passStrength` 读取，默认为 10
- 哈希结果包含 salt，无需单独存储盐值
- 相同密码每次哈希结果不同，防止彩虹表攻击

---

## 三、鉴权中间件体系

### 3.1 鉴权状态机

```go
// auth/authentication.go

type authState int

const (
    authStateSkip       authState = iota // 不匹配，尝试下一种鉴权方式
    authStateForbidden                   // 已认证但无权限
    authStateNotElevated                 // 会话未提升（敏感操作需要）
    authStateOk                          // 认证通过
)
```

### 3.2 三级鉴权中间件

| 中间件 | 适用场景 | 校验逻辑 |
|--------|----------|----------|
| `RequireAdmin` | 用户管理接口 | 必须是管理员用户，支持 Basic Auth 或已提升的 Client Token |
| `RequireElevatedClient` | 敏感操作（改密、删除） | 会话需处于提升状态（ElevatedUntil 未过期） |
| `RequireClient` | 普通用户接口 | 仅需有效 Client Token 或 Basic Auth |

### 3.3 管理员校验实现

```go
// auth/authentication.go:265-270
func (a *Auth) checkUserAdmin(user *model.User) (authState, error) {
    if !user.Admin {
        return authStateForbidden, nil // 权限位校验
    }
    return authStateOk, nil
}
```

### 3.4 会话提升机制（Elevation）

敏感操作（用户管理、删除应用等）需要会话处于提升状态：

```go
// auth/authentication.go:258-263
func (a *Auth) checkClientElevated(client *model.Client) (authState, error) {
    if client.ElevatedUntil == nil || !timeNow().Before(*client.ElevatedUntil) {
        return authStateNotElevated, nil
    }
    return authStateOk, nil
}
```

---

## 四、路由与权限映射

### 4.1 用户管理路由配置

```go
// router/router.go:229-236

// 创建用户 - 特殊处理：支持公开注册
g.Group("/user").Use(authentication.Optional).POST("", userHandler.CreateUser)

// 管理员专用路由组 - 全部需要 RequireAdmin
authAdmin := g.Group("/user")
authAdmin.Use(authentication.RequireAdmin)
{
    authAdmin.GET("", userHandler.GetUsers)          // 用户列表
    authAdmin.DELETE("/:id", userHandler.DeleteUserByID)  // 删除用户
    authAdmin.GET("/:id", userHandler.GetUserByID)   // 用户详情
    authAdmin.POST("/:id", userHandler.UpdateUserByID)    // 更新用户
}

// 当前用户改密 - 需要提升权限
clientElevated.POST("/current/user/password", userHandler.ChangePassword)
```

### 4.2 权限控制矩阵

| 操作 | 路由 | HTTP方法 | 所需权限 | 鉴权方式 |
|------|------|----------|----------|----------|
| 创建用户 | `/user` | POST | 管理员 或 注册开放 | Optional |
| 用户列表 | `/user` | GET | 管理员 | RequireAdmin |
| 用户详情 | `/user/:id` | GET | 管理员 | RequireAdmin |
| 更新用户 | `/user/:id` | POST | 管理员 | RequireAdmin |
| 删除用户 | `/user/:id` | DELETE | 管理员 | RequireAdmin |
| 修改自己密码 | `/current/user/password` | POST | 登录用户+会话提升 | RequireElevatedClient |

---

## 五、API层业务逻辑详解

### 5.1 创建用户流程

```go
// api/user.go:187-238
func (a *UserAPI) CreateUser(ctx *gin.Context) {
    user := model.CreateUserExternal{}
    if err := ctx.Bind(&user); err == nil {
        // 1. 密码哈希
        internal := &model.User{
            Name:  user.Name,
            Admin: user.Admin,
            Pass:  password.CreatePassword(user.Pass, a.PasswordStrength),
        }

        // 2. 检查用户名是否已存在
        existingUser, _ := a.DB.GetUserByName(internal.Name)

        // 3. 权限校验
        var requestedBy *model.User
        uid := auth.TryGetUserID(ctx)
        if uid != nil {
            requestedBy, _ = a.DB.GetUserByID(*uid)
        }

        if requestedBy == nil || !requestedBy.Admin {
            // 非管理员调用
            if !a.Registration {
                ctx.AbortWithError(401/403, "you are not allowed")
                return
            }
            if internal.Admin {
                ctx.AbortWithError(401/403, "cannot create admin user")
                return
            }
        }

        // 4. 持久化
        if existingUser == nil {
            a.DB.CreateUser(internal)
            a.UserChangeNotifier.fireUserAdded(internal.ID) // 触发插件初始化等
            ctx.JSON(200, toExternalUser(internal))
        } else {
            ctx.AbortWithError(400, "username already exists")
        }
    }
}
```

**创建用户协作链路**：
```
前端表单提交(name, pass, admin)
    ↓
UserStore.create() → POST /user
    ↓
API层: CreateUser
    ├─ 密码哈希 (bcrypt)
    ├─ 检查用户名唯一性
    ├─ 权限校验:
    │   ├─ 管理员调用 → 允许创建任意用户
    │   └─ 非管理员调用 → 仅当注册开放且不创建管理员时允许
    ├─ 数据库写入
    └─ 触发 UserAdded 事件 → 插件初始化
    ↓
返回 UserExternal (无密码)
    ↓
UserStore.refresh() → 刷新用户列表
```

### 5.2 更新用户流程

```go
// api/user.go:445-480
func (a *UserAPI) UpdateUserByID(ctx *gin.Context) {
    withID(ctx, "id", func(id uint) {
        var user *model.UpdateUserExternal
        if err := ctx.Bind(&user); err == nil {
            oldUser, _ := a.DB.GetUserByID(id)
            
            // 安全校验：不能取消最后一个管理员
            adminCount, _ := a.DB.CountUser(&model.User{Admin: true})
            if !user.Admin && oldUser.Admin && adminCount == 1 {
                ctx.AbortWithError(400, "cannot delete last admin")
                return
            }

            internal := &model.User{
                ID:    oldUser.ID,
                Name:  user.Name,
                Admin: user.Admin,
                Pass:  oldUser.Pass, // 默认保留原密码
            }
            
            // 仅当密码字段非空时重新哈希
            if user.Pass != "" {
                internal.Pass = password.CreatePassword(user.Pass, a.PasswordStrength)
            }

            a.DB.UpdateUser(internal)
            ctx.JSON(200, toExternalUser(internal))
        }
    })
}
```

**更新用户协作链路**：
```
前端编辑表单提交(name, pass?, admin)
    ↓
UserStore.update(id, name, pass, admin) → POST /user/:id
    ↓
API层: UpdateUserByID
    ├─ 绑定 UpdateUserExternal DTO
    ├─ 查询原用户信息
    ├─ 安全校验: 不能取消最后一个管理员
    ├─ 密码处理:
    │   ├─ pass == "" → 保留原哈希
    │   └─ pass != "" → 重新 bcrypt 哈希
    ├─ 数据库更新
    └─ 返回更新后的 UserExternal
    ↓
UserStore.refresh() → 刷新用户列表
```

### 5.3 删除用户流程

```go
// api/user.go:329-353
func (a *UserAPI) DeleteUserByID(ctx *gin.Context) {
    withID(ctx, "id", func(id uint) {
        user, _ := a.DB.GetUserByID(id)
        if user != nil {
            // 安全校验：不能删除最后一个管理员
            adminCount, _ := a.DB.CountUser(&model.User{Admin: true})
            if user.Admin && adminCount == 1 {
                ctx.AbortWithError(400, "cannot delete last admin")
                return
            }
            
            // 触发删除事件（清理WebSocket连接、插件资源等）
            a.UserChangeNotifier.fireUserDeleted(id)
            a.DB.DeleteUserByID(id)
        }
    })
}

// database/user.go:55-69
func (d *GormDatabase) DeleteUserByID(id uint) error {
    // 级联删除用户关联资源
    apps, _ := d.GetApplicationsByUser(id)
    for _, app := range apps { d.DeleteApplicationByID(app.ID) }
    
    clients, _ := d.GetClientsByUser(id)
    for _, client := range clients { d.DeleteClientByID(client.ID) }
    
    pluginConfs, _ := d.GetPluginConfByUser(id)
    for _, conf := range pluginConfs { d.DeletePluginConfByID(conf.ID) }
    
    return d.DB.Where("id = ?", id).Delete(&model.User{}).Error
}
```

**删除用户协作链路**：
```
前端点击删除 → ConfirmDialog 确认
    ↓
UserStore.remove(id) → DELETE /user/:id
    ↓
API层: DeleteUserByID
    ├─ 查询用户是否存在
    ├─ 安全校验: 不能删除最后一个管理员
    ├─ 触发 UserDeleted 事件:
    │   ├─ streamHandler.NotifyDeletedUser → 关闭WebSocket连接
    │   └─ pluginManager.RemoveUser → 清理插件资源
    └─ 数据库层级联删除:
        ├─ 删除用户所有 Application
        ├─ 删除用户所有 Client
        ├─ 删除用户所有 PluginConf
        └─ 删除 User 记录
    ↓
UserStore.refresh() → 刷新用户列表
```

---

## 六、前端状态层与UI交互

### 6.1 前端状态管理（MobX）

```typescript
// ui/src/user/UserStore.ts
export class UserStore extends BaseStore<IUser> {
    // 创建用户
    @action
    public create = async (name: string, pass: string, admin: boolean) => {
        await axios.post(`${config.get('url')}user`, {name, pass, admin});
        await this.refresh();
        this.snack('User created');
    };

    // 更新用户
    @action
    public update = async (id: number, name: string, pass: string | null, admin: boolean) => {
        await axios.post(config.get('url') + 'user/' + id, {name, pass, admin});
        await this.refresh();
        this.snack('User updated');
    };

    // 删除用户（继承自 BaseStore）
    protected requestDelete(id: number): Promise<void> {
        return axios.delete(`${config.get('url')}user/${id}`)
            .then(() => this.snack('User deleted'));
    }
}
```

### 6.2 表单组件设计

```typescript
// ui/src/user/AddEditUserDialog.tsx
const AddEditUserDialog = ({fClose, fOnSubmit, isEdit, name, admin}: IProps) => {
    const [name, setName] = React.useState(initialName);
    const [pass, setPass] = React.useState('');
    const [admin, setAdmin] = React.useState(initialAdmin);

    // 表单验证
    const namePresent = name.length !== 0;
    const passPresent = pass.length !== 0 || isEdit; // 编辑时密码可选

    return (
        <Dialog>
            <TextField label="Username *" value={name} onChange={setName} />
            <TextField 
                type="password" 
                label={isEdit ? 'Password (empty if no change)' : 'Password *'}
                value={pass} 
                onChange={setPass} 
            />
            <FormControlLabel
                control={<Switch checked={admin} onChange={setAdmin} />}
                label="has administrator rights"
            />
            <Button 
                disabled={!passPresent || !namePresent}
                onClick={() => {
                    fOnSubmit(name, pass, admin);
                    fClose();
                }}>
                {isEdit ? 'Save' : 'Create'}
            </Button>
        </Dialog>
    );
};
```

### 6.3 用户列表页面

```typescript
// ui/src/user/Users.tsx
const Users = observer(() => {
    const [deleteUser, setDeleteUser] = React.useState<IUser>();
    const [editUser, setEditUser] = React.useState<IUser>();
    const [createDialog, setCreateDialog] = React.useState(false);
    const {userStore} = useStores();
    
    React.useEffect(() => void userStore.refresh(), []);
    const users = userStore.getItems();

    return (
        <DefaultPage title="Users" rightControl={
            <Button onClick={() => setCreateDialog(true)}>Create User</Button>
        }>
            <Table>
                {users.map((user: IUser) => (
                    <UserRow
                        name={user.name}
                        admin={user.admin}
                        fDelete={() => setDeleteUser(user)}
                        fEdit={() => setEditUser(user)}
                    />
                ))}
            </Table>
            
            {/* 创建用户对话框 */}
            {createDialog && (
                <AddEditDialog 
                    fClose={() => setCreateDialog(false)} 
                    fOnSubmit={userStore.create} 
                />
            )}
            
            {/* 编辑用户对话框 */}
            {editUser && (
                <AddEditDialog
                    fClose={() => setEditUser(undefined)}
                    fOnSubmit={userStore.update.bind(this, editUser.id)}
                    name={editUser.name}
                    admin={editUser.admin}
                    isEdit={true}
                />
            )}
            
            {/* 删除确认对话框 */}
            {deleteUser && (
                <ConfirmDialog
                    title="Confirm Delete"
                    text={'Delete ' + deleteUser.name + '?'}
                    fClose={() => setDeleteUser(undefined)}
                    fOnSubmit={() => userStore.remove(deleteUser.id)}
                />
            )}
        </DefaultPage>
    );
});
```

---

## 七、完整协作链路总览

### 7.1 创建用户完整链路

```
┌─────────────────────────────────────────────────────────────┐
│                     前端 UI 层                              │
├─────────────────────────────────────────────────────────────┤
│  1. Users.tsx: 点击 "Create User" 按钮                       │
│     ↓ 打开 AddEditUserDialog                                │
│  2. AddEditUserDialog: 填写 name/pass/admin，点击 Create     │
│     ↓ 调用 props.fOnSubmit(name, pass, admin)               │
│  3. UserStore.create(): POST /user {name, pass, admin}      │
│     ↓ 等待响应成功                                          │
│  4. UserStore.refresh(): GET /user 刷新列表                  │
└─────────────────────────────────────────────────────────────┘
                              ↓ HTTP 请求
┌─────────────────────────────────────────────────────────────┐
│                     后端 API 层                             │
├─────────────────────────────────────────────────────────────┤
│  5. Router: POST /user → authentication.Optional            │
│     ↓ 鉴权中间件                                            │
│  6. Auth: Optional 鉴权                                      │
│     ├─ 有有效凭证 → 识别用户身份                             │
│     └─ 无有效凭证 → 匿名访问（仅当注册开放时允许）            │
│     ↓                                                       │
│  7. UserAPI.CreateUser()                                     │
│     ├─ 绑定 CreateUserExternal DTO                          │
│     ├─ password.CreatePassword() → bcrypt 哈希               │
│     ├─ 检查用户名唯一性                                      │
│     ├─ 权限校验:                                            │
│     │  ├─ 管理员 → 允许创建任意用户                          │
│     │  └─ 非管理员/匿名 → 仅当注册开放且非管理员用户          │
│     ├─ DB.CreateUser() → 写入数据库                          │
│     └─ fireUserAdded() → 触发插件初始化                      │
│     ↓ 返回 UserExternal                                     │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 修改用户密码完整链路

```
┌─────────────────────────────────────────────────────────────┐
│                     前端 UI 层                              │
├─────────────────────────────────────────────────────────────┤
│  1. Users.tsx: 点击编辑图标                                   │
│     ↓ 打开 AddEditUserDialog (isEdit=true)                   │
│  2. AddEditUserDialog: 修改 name/pass/admin，点击 Save       │
│     ↓ 调用 props.fOnSubmit(name, pass, admin)               │
│  3. UserStore.update(id, name, pass, admin)                  │
│     → POST /user/:id {name, pass, admin}                    │
│     ↓ 等待响应成功                                          │
│  4. UserStore.refresh(): GET /user 刷新列表                  │
└─────────────────────────────────────────────────────────────┘
                              ↓ HTTP 请求
┌─────────────────────────────────────────────────────────────┐
│                     后端 API 层                             │
├─────────────────────────────────────────────────────────────┤
│  5. Router: POST /user/:id → authentication.RequireAdmin    │
│     ↓ 鉴权中间件                                            │
│  6. Auth: RequireAdmin 校验                                  │
│     ├─ 检查是否是管理员用户（Admin=true）                     │
│     └─ 支持 Basic Auth 或已提升的 Client Token               │
│     ↓                                                       │
│  7. UserAPI.UpdateUserByID()                                 │
│     ├─ 绑定 UpdateUserExternal DTO                          │
│     ├─ 查询原用户信息                                        │
│     ├─ 安全校验: 不能取消最后一个管理员                       │
│     ├─ 密码处理:                                            │
│     │  ├─ pass == "" → 保留原哈希                            │
│     │  └─ pass != "" → password.CreatePassword() 重新哈希    │
│     └─ DB.UpdateUser() → 更新数据库                          │
│     ↓ 返回 UserExternal                                     │
└─────────────────────────────────────────────────────────────┘
```

### 7.3 删除用户完整链路

```
┌─────────────────────────────────────────────────────────────┐
│                     前端 UI 层                              │
├─────────────────────────────────────────────────────────────┤
│  1. Users.tsx: 点击删除图标                                   │
│     ↓ 打开 ConfirmDialog 确认                                │
│  2. ConfirmDialog: 点击确认删除                               │
│     ↓ 调用 fOnSubmit()                                      │
│  3. UserStore.remove(id): DELETE /user/:id                   │
│     ↓ 等待响应成功                                          │
│  4. UserStore.refresh(): GET /user 刷新列表                  │
└─────────────────────────────────────────────────────────────┘
                              ↓ HTTP 请求
┌─────────────────────────────────────────────────────────────┐
│                     后端 API 层                             │
├─────────────────────────────────────────────────────────────┤
│  5. Router: DELETE /user/:id → authentication.RequireAdmin   │
│     ↓ 鉴权中间件                                            │
│  6. Auth: RequireAdmin 校验                                  │
│     ↓                                                       │
│  7. UserAPI.DeleteUserByID()                                 │
│     ├─ 查询用户是否存在                                      │
│     ├─ 安全校验: 不能删除最后一个管理员                       │
│     ├─ fireUserDeleted(id):                                 │
│     │  ├─ streamHandler.NotifyDeletedUser → 关闭WebSocket    │
│     │  └─ pluginManager.RemoveUser → 清理插件资源            │
│     └─ DB.DeleteUserByID(id):                               │
│        ├─ 级联删除 Applications                             │
│        ├─ 级联删除 Clients                                  │
│        ├─ 级联删除 PluginConfs                              │
│        └─ 删除 User 记录                                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 八、关键安全设计要点

### 8.1 权限位设计
- 使用单一布尔字段 `Admin` 作为权限位，简化权限模型
- 管理员权限为全局权限，无细粒度角色区分
- 最后一个管理员不可删除/降级，防止系统锁死

### 8.2 密码安全
- 密码永不以明文形式存储或返回
- bcrypt 单向哈希，含内置 salt
- 哈希强度可配置，随硬件提升可调整
- 更新用户时空密码字段保留原哈希，避免泄露

### 8.3 会话提升机制
- 敏感操作需要 Elevated 会话
- 提升状态有过期时间（ElevatedUntil）
- 支持 Basic Auth 直接获得提升权限
- OIDC 用户通过弹窗流程获得提升

### 8.4 API 权限控制
- 用户管理接口统一使用 `RequireAdmin` 中间件
- 创建用户接口特殊处理以支持公开注册
- 非管理员无法创建管理员用户
- 所有操作返回结构化错误信息

---

## 九、前端状态流转

```
初始状态: users = []
    ↓
组件挂载 → userStore.refresh()
    ↓
GET /user → 加载用户列表 → users = [user1, user2, ...]
    ↓
操作触发:
├─ 创建用户 → POST /user → refresh() → users 更新
├─ 编辑用户 → POST /user/:id → refresh() → users 更新
└─ 删除用户 → DELETE /user/:id → refresh() → users 更新
    ↓
UI 响应式更新（MobX observer）
```

---

## 十、总结

本系统用户管理协作链路体现了以下设计原则：

1. **分层清晰**：Model → DTO → API → Router → Middleware → Frontend Store → UI Component
2. **安全优先**：密码哈希、权限校验、会话提升、级联清理
3. **状态一致性**：操作后自动刷新列表，MobX 响应式更新 UI
4. **容错设计**：最后管理员保护、用户名唯一性校验、空密码字段处理
5. **可扩展性**：UserChangeNotifier 事件机制支持插件扩展

整个链路从前端表单输入到后端数据库持久化，每一层都有明确的职责边界和安全校验，确保用户管理操作的安全性和可靠性。
