# 插件 Storage Handler 生命周期与跨进程持久化分析

## 1. 核心架构概述

### 1.1 Storage Handler 接口定义

在 `plugin/compat/instance.go:79-83` 中定义了 `StorageHandler` 接口：

```go
type StorageHandler interface {
    Save(b []byte) error
    Load() ([]byte, error)
}
```

### 1.2 数据库存储实现

`dbStorageHandler` 结构体在 `plugin/storagehandler.go:3-23` 中实现：

```go
type dbStorageHandler struct {
    pluginID uint
    db       Database
}

func (c dbStorageHandler) Save(b []byte) error {
    conf, err := c.db.GetPluginConfByID(c.pluginID)
    if err != nil {
        return err
    }
    conf.Storage = b
    return c.db.UpdatePluginConf(conf)
}

func (c dbStorageHandler) Load() ([]byte, error) {
    pluginConf, err := c.db.GetPluginConfByID(c.pluginID)
    if err != nil {
        return nil, err
    }
    return pluginConf.Storage, nil
}
```

### 1.3 数据模型

`PluginConf` 模型在 `model/pluginconf.go:4-13` 中定义，包含 `Storage []byte` 字段用于持久化存储：

```go
type PluginConf struct {
    ID            uint `gorm:"primaryKey;autoIncrement"`
    UserID        uint
    ModulePath    string `gorm:"type:text"`
    Token         string `gorm:"type:varchar(180);uniqueIndex:uix_plugin_confs_token"`
    ApplicationID uint
    Enabled       bool
    Config        []byte
    Storage       []byte  // 存储插件持久化数据
}
```

---

## 2. Storage Handler 生命周期状态流转

### 2.1 生命周期状态图 - 双独立路径

```
                    ┌─────────────────┐
                    │   插件文件加载   │
                    │  (plugin.Open)  │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  兼容层包装     │
                    │  (compat.Wrap)  │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  插件实例创建   │
                    │ (NewPluginInst) │
                    └────────┬────────┘
                             │
          ┌──────────────────┴──────────────────┐
          │                                       │
          ▼                                       ▼
┌───────────────────────┐             ┌───────────────────────┐
│ 路径 A: 存储能力检测  │             │ 路径 B: 启用状态判断  │
│ (HasSupport Storager) │             │ (PluginConf.Enabled)  │
└───────────┬───────────┘             └───────────┬───────────┘
            │                                     │
  ┌─────────┴─────────┐                 ┌─────────┴─────────┐
  │ 支持             │不支持            │ true             │ false
  ▼                   ▼                 ▼                   ▼
┌───────────┐     ┌───────────┐     ┌───────────┐     ┌───────────┐
│ 设置      │     │ 跳过      │     │ 调用      │     │ 保持      │
│ Storage   │     │ 设置      │     │ Enable()  │     │ 禁用      │
│ Handler   │     │           │     │           │     │ 状态      │
└─────┬─────┘     └───────────┘     └─────┬─────┘     └───────────┘
      │                                   │
      ▼                                   ▼
┌───────────────────────────────────────────────────────────────┐
│                  实例运行时（两条路径汇合）                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    │
│  │ 插件已启用   │    │ 插件已启用   │    │ 插件已禁用   │    │
│  │ + 有存储    │───►│ + 无存储    │───►│ + 有/无存储  │    │
│  │ 能力        │    │ 能力        │    │ 能力        │    │
│  └──────┬──────┘    └──────────────┘    └──────┬──────┘    │
│         │                                       │           │
│         ▼                                       ▼           │
│  ┌──────────────┐                         ┌──────────────┐  │
│  │ 调用        │                         │ 仅保留      │  │
│  │ Save/Load   │                         │ 实例引用    │  │
│  └──────┬──────┘                         └──────────────┘  │
└─────────┼───────────────────────────────────────────────────┘
          │
          ▼
    ┌──────────────────┐
    │  插件禁用        │
    │  (Disable API)   │
    └────────┬────────┘
             │
             ▼
    ┌──────────────────┐
    │  用户删除/清理   │
    │  (RemoveUser)    │
    └────────┬────────┘
             │
             ▼
    ┌──────────────────┐
    │  数据库记录删除  │
    │  (级联删除)      │
    └──────────────────┘
```

### 2.2 两条独立路径说明

#### 路径 A: 存储能力检测（独立路径一）

- **位置**: `plugin/manager.go:342-344`
- **触发时机**: 实例创建后立即执行，与启用状态无关
- **关键操作**:
  ```go
  if compat.HasSupport(instance, compat.Storager) {
      instance.SetStorageHandler(dbStorageHandler{pluginConf.ID, m.db})
  }
  ```
- **独立性**: 无论插件是否启用，只要插件实现了 `Storager` 接口，就会绑定 Storage Handler
- **结果**:
  - ✅ 支持: 实例持有 `dbStorageHandler` 引用，可随时调用 Save/Load
  - ❌ 不支持: 实例不持有 Storage Handler，无法进行持久化操作

#### 路径 B: 启用状态判断（独立路径二）

- **位置**: `plugin/manager.go:353-362`
- **触发时机**: 实例创建后，与存储能力检测并行执行
- **关键操作**:
  ```go
  if pluginConf.Enabled {
      err := instance.Enable()  // 与是否支持存储无关
      if err != nil {
          pluginConf.Enabled = false
          m.db.UpdatePluginConf(pluginConf)
      }
  }
  ```
- **独立性**: 无论是否支持存储能力，只要数据库中 `Enabled=true` 就会尝试启用插件
- **结果**:
  - ✅ 启用成功: 插件进入运行状态，可执行业务逻辑
  - ❌ 启用失败: 标记为禁用，记录错误日志

### 2.3 四种组合状态矩阵

| 状态组合 | 存储能力 | 启用状态 | 行为描述 |
|---------|---------|---------|---------|
| 组合 1 | ✅ 支持 | ✅ 启用 | 完整功能：插件运行中，可随时调用 Save/Load 持久化数据 |
| 组合 2 | ✅ 支持 | ❌ 禁用 | 待机状态：插件未运行，但 Storage Handler 已绑定，启用后即可使用存储 |
| 组合 3 | ❌ 不支持 | ✅ 启用 | 无存储运行：插件运行中，但无法持久化任何数据 |
| 组合 4 | ❌ 不支持 | ❌ 禁用 | 完全停用：插件未运行且无存储能力 |

### 2.4 详细状态说明

#### 状态 1: 插件文件加载 (`loadPlugins`)

- **位置**: `plugin/manager.go:219-254`
- **触发时机**: Manager 初始化时
- **关键操作**:
  - 读取插件目录下的 `.so` 文件
  - 使用 `plugin.Open()` 加载插件
  - 通过 `compat.Wrap()` 包装为兼容层
  - 调用 `LoadPlugin()` 注册到 `m.plugins` 映射

#### 状态 2: 插件实例创建 (`initializeSingleUserPlugin`)

- **位置**: `plugin/manager.go:315-364`
- **触发时机**: 用户初始化或新用户创建时
- **关键操作**:
  ```go
  instance := p.NewPluginInstance(userCtx)  // 创建实例
  m.instances[pluginConf.ID] = instance     // 注册到实例映射
  ```

#### 状态 3: Storage Handler 绑定（路径 A）

- **位置**: `plugin/manager.go:342-344`
- **触发时机**: 实例创建后立即执行
- **与启用状态的关系**: 完全独立，先于启用状态判断执行
- **关键操作**:
  ```go
  if compat.HasSupport(instance, compat.Storager) {
      instance.SetStorageHandler(dbStorageHandler{pluginConf.ID, m.db})
  }
  ```

#### 状态 4: 插件启用判断（路径 B）

- **位置**: `plugin/manager.go:353-362`
- **触发时机**: Storage Handler 绑定后（或跳过绑定后）
- **与存储能力的关系**: 完全独立，不依赖存储能力检测结果
- **关键操作**:
  ```go
  if pluginConf.Enabled {
      err := instance.Enable()  // 无论是否支持存储都会执行
      // ...
  }
  ```

#### 状态 5: 运行时数据操作 (Save/Load)

- **前提条件**: 必须同时满足"支持存储能力" AND "插件已启用"
- **触发时机**: 插件业务逻辑需要持久化时
- **关键操作**:
  - `Save(b []byte)`: 将二进制数据保存到数据库 `PluginConf.Storage` 字段
  - `Load()`: 从数据库读取之前保存的二进制数据

#### 状态 6: 插件禁用 (`Disable`)

- **位置**: `plugin/manager.go:137-138`
- **触发时机**: 用户主动调用 Disable API 或初始化失败时
- **对存储的影响**: Storage Handler 绑定关系仍然保留，只是插件停止运行
- **关键操作**:
  - 调用 `instance.Disable()` 停止插件运行
  - 更新数据库中 `PluginConf.Enabled = false`
  - **Storage Handler 仍保留在实例中**，下次启用后可直接使用

#### 状态 7: 实例回收 (`RemoveUser`)

- **位置**: `plugin/manager.go:184-208`
- **触发时机**: 用户被删除时
- **对存储的影响**: 仅删除内存实例，数据库中的 Storage 数据仍然保留
- **关键操作**:
  ```go
  for _, p := range m.plugins {
      pluginConf, err := m.db.GetPluginConfByUserAndPath(userID, ...)
      if pluginConf.Enabled {
          inst.Disable()  // 先禁用
      }
      delete(m.instances, pluginConf.ID)  // 从实例映射中删除
  }
  ```

---

## 3. 跨进程持久化完整闭环

### 3.1 完整生命周期闭环图

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                         跨进程持久化完整闭环                                   │
└───────────────────────────────────────────────────────────────────────────────┘
              │
              ▼
    ┌─────────────────────────┐
    │  1. 数据写入阶段        │
    │     (进程 A)            │
    │  - 插件调用 Save(data)  │
    │  - 写入 PluginConf      │
    └───────────┬─────────────┘
                │
                ▼
    ┌─────────────────────────┐
    │  2. 数据库持久化        │
    │     (共享存储)          │
    │  - conf.Storage = data  │
    │  - UpdatePluginConf()   │
    └───────────┬─────────────┘
                │
    ┌───────────┴─────────────┐
    │                         │
    ▼                         ▼
┌─────────────────┐   ┌─────────────────┐
│ 3. 跨进程读取   │   │ 4. 进程重启恢复 │
│    (进程 B)     │   │    (任意进程)   │
│ - 插件 Load()   │   │ - 重新创建实例  │
│ - 获取 data     │   │ - 绑定 Handler  │
└────────┬────────┘   └────────┬────────┘
         │                      │
         └──────────┬───────────┘
                    │
                    ▼
          ┌─────────────────────────┐
          │  5. 实例禁用阶段        │
          │     (任意进程)          │
          │  - 调用 Disable()       │
          │  - Enabled = false      │
          │  - Storage 数据仍保留   │
          └───────────┬─────────────┘
                      │
                      ▼
          ┌─────────────────────────┐
          │  6. 内存实例回收        │
          │     (RemoveUser)        │
          │  - delete(m.instances)  │
          │  - GC 回收内存           │
          │  - 数据库记录仍保留     │
          └───────────┬─────────────┘
                      │
                      ▼
          ┌─────────────────────────┐
          │  7. 数据库记录删除      │
          │     (用户删除级联)      │
          │  - DELETE PluginConf    │
          │  - Storage 字段清除     │
          │  - 数据永久删除         │
          └─────────────────────────┘
```

### 3.2 闭环各阶段详细说明

#### 阶段 1: 数据写入（主动持久化）

**触发条件**: 插件支持 Storager + 插件已启用 + 业务需要持久化

**执行流程** (`plugin/storagehandler.go:8-15`):
```go
// 插件业务代码调用
storageHandler.Save(serializedData)

// dbStorageHandler 实现
func (c dbStorageHandler) Save(b []byte) error {
    conf, err := c.db.GetPluginConfByID(c.pluginID)  // 读取当前配置
    if err != nil {
        return err
    }
    conf.Storage = b                                 // 更新存储字段
    return c.db.UpdatePluginConf(conf)               // 原子写入数据库
}
```

**跨进程影响**: 写入完成后，其他进程中同一用户的同一插件实例下次调用 `Load()` 时可读取到新数据

#### 阶段 2: 数据库持久化（共享存储层）

**数据位置**: `model/pluginconf.go:12` - `Storage []byte` 字段

**持久化特性**:
- **原子性**: `UpdatePluginConf()` 是单条记录更新，数据库保证原子性
- **一致性**: 每次 Save 覆盖之前的 Storage 数据，无历史版本
- **隔离性**: 按 `UserID` + `ModulePath` 隔离，用户间数据不共享
- **持久性**: 写入后永久保留，直到显式删除

#### 阶段 3: 跨进程读取

**触发时机**: 其他进程中插件实例调用 `Load()` 时

**执行流程** (`plugin/storagehandler.go:17-22`):
```go
// 插件业务代码调用
data, err := storageHandler.Load()

// dbStorageHandler 实现
func (c dbStorageHandler) Load() ([]byte, error) {
    pluginConf, err := c.db.GetPluginConfByID(c.pluginID)  // 直接查数据库
    if err != nil {
        return nil, err
    }
    return pluginConf.Storage, nil  // 返回最新持久化数据
}
```

**关键特性**: 无缓存，每次 `Load()` 都直接查询数据库，保证跨进程数据一致性

#### 阶段 4: 进程重启恢复

**触发时机**: Gotify Server 进程重启后

**恢复流程** (`plugin/manager.go:90-98`):
```go
// Manager 初始化时
users, err := manager.db.GetUsers()
for _, user := range users {
    manager.initializeForUser(*user)  // 为每个用户重建插件实例
}

// initializeSingleUserPlugin 中:
// 1. 重新创建插件实例
// 2. 检测存储能力，重新绑定 dbStorageHandler（路径 A）
// 3. 检查 Enabled 状态，决定是否启用插件（路径 B）
// 4. 插件启用后，调用 Load() 可获取重启前持久化的数据
```

**恢复保证**: 只要数据库中 Storage 数据存在，重启后即可恢复

#### 阶段 5: 实例禁用（软停用）

**触发时机**: 用户调用 Disable API 或插件初始化失败

**对持久化数据的影响** (`plugin/manager.go:137-147`):
```go
err = instance.Disable()                  // 停止插件运行
if newConf, err := m.db.GetPluginConfByID(pluginID); err == nil {
    conf = newConf
}
conf.Enabled = false
return m.db.UpdatePluginConf(conf)        // 只更新 Enabled 字段
```

**重要说明**:
- ✅ Storage 数据 **完全保留** 在数据库中
- ✅ Storage Handler **绑定关系保留** 在内存实例中
- ✅ 下次 Enable 后可直接继续使用之前的存储数据
- ❌ 仅停止插件运行，不清理任何存储

#### 阶段 6: 内存实例回收（用户删除）

**触发时机**: 用户被删除时调用 `RemoveUser`

**执行流程** (`plugin/manager.go:184-208`):
```go
func (m *Manager) RemoveUser(userID uint) error {
    for _, p := range m.plugins {
        pluginConf, err := m.db.GetPluginConfByUserAndPath(userID, ...)
        if pluginConf.Enabled {
            inst.Disable()                // 先禁用插件
        }
        delete(m.instances, pluginConf.ID) // 从内存映射中移除引用
    }
    return nil
}
```

**对持久化数据的影响**:
- ✅ 数据库中 `PluginConf` 记录 **仍保留**
- ✅ Storage 字段数据 **完整保留**
- ❌ 仅回收内存中的插件实例对象
- ⚠️ 此时虽然数据库记录存在，但已无内存实例可访问存储数据

#### 阶段 7: 数据库记录删除（数据清除）

**触发时机**: 用户被从数据库中删除时的级联操作

**删除流程** (`database/user.go` 中的用户删除逻辑):
```go
// 删除用户前，先删除关联的插件配置
func (d *GormDatabase) DeleteUser(id uint) error {
    // 1. 删除该用户所有插件配置（包含 Storage 字段）
    if err := d.DB.Where("user_id = ?", id).Delete(&model.PluginConf{}).Error; err != nil {
        return err
    }
    // 2. 删除用户
    return d.DB.Delete(&model.User{}, id).Error
}
```

**最终结果**:
- `PluginConf` 记录被 DELETE
- Storage 字段的数据永久丢失
- 跨进程持久化闭环完成

---

## 4. 关键持久化特性总结

### 4.1 数据生命周期矩阵

| 操作阶段 | 内存实例状态 | Storage Handler 绑定 | 数据库 Storage 数据 | 跨进程可见性 |
|---------|------------|-------------------|-------------------|------------|
| 实例创建 | ✅ 存在 | ⚠️ 检测中 | ✅ 空/初始值 | ✅ |
| 写入数据 | ✅ 存在 | ✅ 已绑定 | ✅ 已更新 | ✅ 立即可见 |
| 读取数据 | ✅ 存在 | ✅ 已绑定 | ✅ 读取最新 | ✅ 读取其他进程写入 |
| 插件禁用 | ✅ 存在 | ✅ 仍绑定 | ✅ 保留 | ✅ 其他进程仍可见 |
| 内存回收 | ❌ 已删除 | ❌ 随实例回收 | ✅ 仍保留 | ⚠️ 无实例可访问 |
| 数据库删除 | ❌ 已删除 | ❌ 随实例回收 | ❌ 永久删除 | ❌ 不可访问 |

### 4.2 跨进程一致性保证

1. **无缓存设计**: 每次 Load 都直接查数据库，避免缓存不一致
2. **原子更新**: Save 操作是单条数据库记录更新，保证原子性
3. **最终一致**: 由于无缓存，进程间数据立即可见，实现强一致性

### 4.3 资源回收分层策略

| 层级 | 回收时机 | 回收方式 |
|-----|---------|---------|
| 插件运行态 | 用户调用 Disable | 调用 Disable() 方法 |
| 内存实例 | 用户被删除 | delete(map) + GC |
| 数据库记录 | 用户被删除 | DELETE SQL 级联删除 |

---

## 5. 关键代码位置索引

| 功能模块 | 文件路径 | 行号 |
|---------|---------|------|
| StorageHandler 接口定义 | `plugin/compat/instance.go` | 79-83 |
| dbStorageHandler 实现 | `plugin/storagehandler.go` | 3-23 |
| Storage Handler 绑定（路径 A） | `plugin/manager.go` | 342-344 |
| 启用状态判断（路径 B） | `plugin/manager.go` | 353-362 |
| 插件启用/禁用 | `plugin/manager.go` | 134-147 |
| 用户插件初始化 | `plugin/manager.go` | 315-364 |
| 用户删除内存回收 | `plugin/manager.go` | 184-208 |
| PluginConf 数据模型 | `model/pluginconf.go` | 4-13 |
| 能力检测函数 | `plugin/compat/instance.go` | 51-59 |

---

## 6. 设计要点总结

### 6.1 核心设计优点

1. **双独立路径设计**: 存储能力检测与启用状态解耦，逻辑清晰
2. **数据库持久化**: 天然支持跨进程、跨重启的数据共享
3. **用户级隔离**: 每个用户的插件数据完全独立
4. **优雅降级**: 初始化失败时自动禁用，不影响主程序运行
5. **分层回收**: 运行态、内存、数据库三层独立回收，灵活可控

### 6.2 注意事项与限制

1. **无版本控制**: Storage 字段是原始字节数组，插件需要自行处理数据版本兼容
2. **无事务支持**: 多次 Save 操作间没有事务保证，插件需要自行处理一致性
3. **内存泄漏风险**: 插件禁用后实例仍在内存中，只有用户删除时才会清理内存
4. **无缓存机制**: 每次 Load 都直接查询数据库，频繁操作可能影响性能
5. **禁用后数据保留**: 禁用插件不会清理 Storage，需要手动处理数据清理需求

### 6.3 潜在优化点

1. 可添加 Storage 数据的缓存层，减少数据库查询
2. 可实现插件禁用后定时清理空闲实例（LRU 策略）
3. 可添加 Storage 数据的版本字段，支持数据迁移
4. 可提供插件级别的 Storage 清理 API，允许用户手动清除存储数据
5. 可添加 Storage 数据大小限制，防止单个插件占用过多数据库空间
