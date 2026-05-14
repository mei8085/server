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

### 2.1 生命周期状态图

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
                  ┌──────────▼──────────┐
                  │ 检测是否支持 Storager │
                  │ (HasSupport 检测)     │
                  └──────────┬──────────┘
                             │
              ┌──────────────┼──────────────┐
              │ 支持         │ 不支持        │
              ▼              ▼               │
    ┌──────────────────┐    ┌───────────────┐│
    │ 设置 Storage     │    │ 跳过设置      ││
    │ Handler 绑定     │    │ 继续初始化    ││
    └────────┬─────────┘    └───────┘│
             │                  │
             ▼                  ▼
    ┌──────────────────┐    ┌───────────────┐
    │  插件启用        │    │  保持禁用     │
    │  (instance.Enable())  │  (Enabled=false)│
    └────────┬────────┘    └───────────────┘
             │
             ▼
    ┌──────────────────┐
    │  运行时数据操作   │◄──┐
    │  (Save/Load)     │   │
    └────────┬────────┘   │
             │             │
             ▼             │
    ┌──────────────────┐   │
    │  插件禁用        │   │
    │  (instance.Disable())│
    └────────┬────────┘   │
             │             │
             ▼             │
    ┌──────────────────┐   │
    │  用户删除/清理   │   │
    │  (RemoveUser)    │   │
    └──────────────────┘
```

### 2.2 详细状态说明

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

#### 状态 3: Storage Handler 绑定

- **位置**: `plugin/manager.go:342-344`
- **触发时机**: 实例创建后，能力检测通过时
- **关键操作**:
  ```go
  if compat.HasSupport(instance, compat.Storager) {
      instance.SetStorageHandler(dbStorageHandler{pluginConf.ID, m.db})
  }
  ```

#### 状态 4: 插件启用 (`Enable`)

- **位置**: `plugin/manager.go:134-135`
- **触发时机**: 插件配置中 `Enabled=true` 时或用户主动启用
- **关键操作**:
  - 调用 `instance.Enable()` 激活插件
  - 插件可以开始使用 `StorageHandler` 进行数据操作

#### 状态 5: 运行时数据操作 (Save/Load)

- **触发时机**: 插件业务逻辑需要持久化时
- **关键操作**:
  - `Save(b []byte)`: 将二进制数据保存到数据库 `PluginConf.Storage` 字段
  - `Load()`: 从数据库读取之前保存的二进制数据

#### 状态 6: 插件禁用 (`Disable`)

- **位置**: `plugin/manager.go:137-138`
- **触发时机**: 用户主动禁用或初始化失败时
- **关键操作**:
  - 调用 `instance.Disable()` 停止插件运行
  - 更新数据库中 `PluginConf.Enabled = false`

#### 状态 7: 实例回收 (`RemoveUser`)

- **位置**: `plugin/manager.go:184-208`
- **触发时机**: 用户被删除时
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

## 3. 跨进程持久化机制

### 3.1 持久化实现原理

**核心设计**: 通过数据库作为跨进程共享存储介质

1. **存储路径**: 插件实例 → `dbStorageHandler` → 数据库 → `PluginConf.Storage` 字段
2. **读取路径**: `PluginConf.Storage` 字段 → 数据库 → `dbStorageHandler` → 插件实例

### 3.2 进程间数据共享流程

```
    进程 A (Gotify Server)                  数据库                     进程 B (Gotify Server)
        │                                       │                              │
        │  1. 插件调用 Save(data)              │                              │
        ├──────────────────────────────────────►│                              │
        │                                       │                              │
        │  2. dbStorageHandler.Save()          │                              │
        │    - GetPluginConfByID()             │                              │
        │    - conf.Storage = data             │                              │
        │    - UpdatePluginConf(conf)          │                              │
        │                                       │  [数据持久化完成]            │
        │                                       │                              │
        │                                       │◄─────────────────────────────┤
        │                                       │  3. 进程 B 重启时重新加载    │
        │                                       │                              │
        │                                       │  4. initializeSingleUserPlugin│
        │                                       │    - 创建新实例               │
        │                                       │    - 绑定 dbStorageHandler   │
        │                                       │                              │
        │                                       ├─────────────────────────────►│
        │                                       │  5. 插件调用 Load()          │
        │                                       │                              │
        │                                       │◄─────────────────────────────┤
        │                                       │  6. 返回进程 A 保存的数据     │
        │                                       │                              │
```

### 3.3 关键持久化特性

#### 3.3.1 原子性保证

- **数据库事务**: 通过 GORM 的数据库操作保证原子性
- **更新操作**: `UpdatePluginConf()` 是原子操作，确保数据一致性

#### 3.3.2 数据隔离

- **按用户隔离**: 每个用户的插件实例有独立的 `PluginConf` 记录
- **按插件隔离**: 每个插件类型通过 `ModulePath` 区分
- **主键关联**: `pluginID` 唯一标识一个用户的一个插件实例

#### 3.3.3 重启恢复机制

在 `plugin/manager.go:353-362` 中实现重启时自动恢复：

```go
if pluginConf.Enabled {
    err := instance.Enable()  // 恢复启用状态
    if err != nil {
        // 启用失败时降级处理
        pluginConf.Enabled = false
        m.db.UpdatePluginConf(pluginConf)
    }
}
```

---

## 4. 回收逻辑与资源清理

### 4.1 正常回收流程

#### 4.1.1 用户删除时的回收 (`RemoveUser`)

- **位置**: `plugin/manager.go:184-208`
- **回收步骤**:
  1. 遍历所有已加载的插件
  2. 获取该用户的插件配置
  3. 如果插件处于启用状态，先调用 `Disable()`
  4. 从 `m.instances` 映射中删除实例引用
  5. 等待 GC 回收内存

#### 4.1.2 插件禁用时的状态

禁用操作并不会立即回收 Storage Handler，只是停止插件的运行：

```go
// plugin/manager.go:137-138
err = instance.Disable()  // 仅停止运行，实例仍保留在内存中
```

### 4.2 异常情况处理

#### 4.2.1 初始化失败降级

在 `plugin/manager.go:353-361` 中：

```go
if pluginConf.Enabled {
    err := instance.Enable()
    if err != nil {
        log.Printf("Plugin initialize failed...")
        pluginConf.Enabled = false  // 标记为禁用
        m.db.UpdatePluginConf(pluginConf)  // 持久化禁用状态
    }
}
```

#### 4.2.2 配置验证失败降级

在 `plugin/manager.go:374-396` 中：

```go
if yaml.Unmarshal(pluginConf.Config, c) != nil || instance.ValidateAndSetConfig(c) != nil {
    pluginConf.Enabled = false  // 自动禁用
    // 保存默认配置和注释后的原配置
    m.db.UpdatePluginConf(pluginConf)
    instance.ValidateAndSetConfig(instance.DefaultConfig())
}
```

### 4.3 内存回收机制

1. **实例引用移除**: `delete(m.instances, pluginConf.ID)` 移除对实例的强引用
2. **GC 自动回收**: Go 垃圾回收器会自动回收不再被引用的对象
3. **数据库连接**: `dbStorageHandler` 持有的数据库连接由连接池管理，不随插件实例回收

---

## 5. 关键代码位置索引

| 功能模块 | 文件路径 | 行号 |
|---------|---------|------|
| StorageHandler 接口定义 | `plugin/compat/instance.go` | 79-83 |
| dbStorageHandler 实现 | `plugin/storagehandler.go` | 3-23 |
| Storage Handler 绑定 | `plugin/manager.go` | 342-344 |
| 插件启用/禁用 | `plugin/manager.go` | 134-138 |
| 用户插件初始化 | `plugin/manager.go` | 315-364 |
| 用户删除回收 | `plugin/manager.go` | 184-208 |
| PluginConf 数据模型 | `model/pluginconf.go` | 4-13 |
| 能力检测函数 | `plugin/compat/instance.go` | 51-59 |

---

## 6. 设计要点总结

### 6.1 优点

1. **数据库持久化**: 天然支持跨进程、跨重启的数据共享
2. **用户隔离**: 每个用户的插件数据完全独立
3. **优雅降级**: 初始化失败时自动禁用，不影响主程序运行
4. **简洁接口**: `Save/Load` 接口设计简单，插件易于使用

### 6.2 注意事项

1. **无版本控制**: Storage 字段是原始字节数组，插件需要自行处理数据版本兼容
2. **无事务支持**: 多次 Save 操作间没有事务保证，插件需要自行处理一致性
3. **内存泄漏风险**: 插件禁用后实例仍在内存中，只有用户删除时才会清理
4. **无缓存机制**: 每次 Load 都直接查询数据库，频繁操作可能影响性能

### 6.3 潜在优化点

1. 可添加 Storage 数据的缓存层，减少数据库查询
2. 可实现插件禁用后定时清理空闲实例
3. 可添加 Storage 数据的版本字段，支持数据迁移
