# 插件加载失败分类策略与测试夹具分析报告

## 一、失败原因分级口径

插件加载失败按照严重程度和可恢复性分为三个等级：

| 级别 | 严重程度 | 描述 | 典型场景 | 处理方式 |
|------|----------|------|----------|----------|
| **致命错误 (Fatal)** | 高 | 导致整个服务启动失败，无法继续运行 | 插件目录不存在、插件文件损坏无法打开 | 直接返回错误，终止服务启动 |
| **插件级错误 (Plugin-Level)** | 中 | 单个插件加载失败，不影响其他插件和服务启动 | 缺少导出符号、函数签名不匹配、模块路径重复 | 终止当前插件加载，记录错误日志 |
| **实例级错误 (Instance-Level)** | 低 | 某个用户的插件实例初始化失败，其他用户不受影响 | 配置验证失败、Enable() 返回错误 | 自动禁用该实例，保留配置待用户修复 |

### 1.1 致命错误（Fatal Errors）

**定义**：发生在 Manager 初始化阶段，导致整个服务无法启动的错误。

**错误类型**：
- 插件目录读取失败：`os.ReadDir(directory)` 返回错误
- 插件文件打开失败：`plugin.Open(pluginPath)` 返回错误（文件损坏、格式错误、非 .so 文件）
- 用户列表获取失败：`db.GetUsers()` 数据库连接错误
- 用户初始化失败：`initializeForUser()` 过程中发生数据库错误

**代码位置**：`plugin/manager.go:57-101` 的 `NewManager()` 函数

**影响**：服务启动失败，返回错误信息给上层调用者。

### 1.2 插件级错误（Plugin-Level Errors）

**定义**：发生在单个插件加载阶段，仅影响该插件，不影响其他插件和服务整体运行。

**错误类型**：
- **缺少导出符号**：`missing GetGotifyPluginInfo symbol` - 插件未导出必需的符号
- **缺少构造函数符号**：`missing NewGotifyPluginInstance symbol` - 未导出实例构造函数
- **构造函数签名不匹配**：`NewGotifyPluginInstance signature mismatch` - 函数类型与预期不符
- **未知版本签名**：`unknown plugin version (unrecognized GetGotifyPluginInfo signature)` - 返回类型不匹配
- **模块路径重复**：`plugin with module path %s is present at least twice` - 同一路径插件被多次加载

**代码位置**：
- 符号检查：`plugin/compat/wrap.go:11-35` 的 `Wrap()` 函数
- 重复检查：`plugin/manager.go:257-264` 的 `LoadPlugin()` 函数

**影响**：当前插件加载失败，已加载的其他插件正常运行。

### 1.3 实例级错误（Instance-Level Errors）

**定义**：发生在为用户创建插件实例阶段，仅影响该用户的该插件实例。

**错误类型**：
- **配置反序列化失败**：YAML 格式错误，无法解析配置
- **配置验证失败**：`ValidateAndSetConfig()` 返回错误
- **启用失败**：`Enable()` 方法返回错误
- **用户不存在**：`user with id %d not found`

**代码位置**：
- 配置初始化：`plugin/manager.go:366-397` 的 `initializeConfigurerForSingleUserPlugin()`
- 实例启用：`plugin/manager.go:353-362` 的 `initializeSingleUserPlugin()`

**影响**：该用户的插件实例被自动禁用，配置被保留（旧配置被注释），等待用户手动修复后重新启用。

## 二、加载链路上的中断点

插件加载流程是一个多阶段的链路，每个阶段都可能发生中断：

```
NewManager()
    │
    ├─→ loadPlugins(directory) [阶段1：批量加载插件文件]
    │     │
    │     ├─ 遍历目录文件
    │     │   ├─ 跳过目录、隐藏文件
    │     │   └─ plugin.Open() [中断点1：文件格式错误]
    │     │
    │     ├─ compat.Wrap(pRaw) [阶段2：符号检查与包装]
    │     │   ├─ Lookup("GetGotifyPluginInfo") [中断点2：缺少Info符号]
    │     │   ├─ 类型断言 func() papiv1.Info [中断点3：Info签名错误]
    │     │   ├─ Lookup("NewGotifyPluginInstance") [中断点4：缺少构造函数]
    │     │   └─ 类型断言 func(ctx) papiv1.Plugin [中断点5：构造函数签名错误]
    │     │
    │     └─ LoadPlugin(compatPlugin) [阶段3：注册插件]
    │         └─ 模块路径重复检查 [中断点6：重复加载]
    │
    ├─→ db.GetUsers() [阶段4：获取用户列表]
    │     └─ 数据库错误 [中断点7：数据访问失败]
    │
    └─→ initializeForUser(user) [阶段5：为每个用户初始化]
          │
          ├─ initializeSingleUserPlugin() [阶段6：初始化单个插件]
          │   ├─ 创建 PluginConf [中断点8：数据库写入失败]
          │   ├─ 注册实例到 manager.instances
          │   ├─ 设置各种 Handler（Message/Storage/Webhook）
          │   │
          │   ├─ [Configurer] 初始化配置 [阶段7：配置处理]
          │   │   ├─ yaml.Unmarshal() [中断点9：配置格式错误]
          │   │   └─ ValidateAndSetConfig() [中断点10：配置验证失败]
          │   │
          │   └─ 若 conf.Enabled == true [阶段8：启用插件]
          │       └─ instance.Enable() [中断点11：启用失败]
          │
          └─ 更新 Application.Internal 标记
```

### 2.1 各中断点详细说明

| 中断点 | 位置 | 错误级别 | 错误信息 | 恢复策略 |
|--------|------|----------|----------|----------|
| 1 | `plugin.Open()` | 致命 | 动态链接库加载失败 | 终止服务启动 |
| 2 | `Wrap()` Lookup Info | 插件级 | `missing GetGotifyPluginInfo symbol` | 跳过该插件 |
| 3 | `Wrap()` Info 类型断言 | 插件级 | `unknown plugin version` | 跳过该插件 |
| 4 | `Wrap()` Lookup 构造函数 | 插件级 | `missing NewGotifyPluginInstance symbol` | 跳过该插件 |
| 5 | `Wrap()` 构造函数类型断言 | 插件级 | `NewGotifyPluginInstance signature mismatch` | 跳过该插件 |
| 6 | `LoadPlugin()` 重复检查 | 插件级 | `plugin with module path ... is present at least twice` | 跳过该插件 |
| 7 | `GetUsers()` | 致命 | 数据库查询错误 | 终止服务启动 |
| 8 | `createPluginConf()` | 致命 | 数据库写入失败 | 终止服务启动 |
| 9 | `yaml.Unmarshal()` 配置 | 实例级 | YAML 解析错误 | 自动禁用，重置为默认配置 |
| 10 | `ValidateAndSetConfig()` | 实例级 | 插件自定义验证错误 | 自动禁用，重置为默认配置 |
| 11 | `instance.Enable()` | 实例级 | 插件自定义启用错误 | 自动禁用，保留配置 |

### 2.2 容错设计

**实例级错误的优雅降级**（`plugin/manager.go:353-362`）：

```go
if pluginConf.Enabled {
    err := instance.Enable()
    if err != nil {
        log.Printf("Plugin initialize failed for user %s: %s. Disabling now...", 
                   userCtx.Name, err.Error())
        pluginConf.Enabled = false
        m.db.UpdatePluginConf(pluginConf)
    }
}
```

**配置失败的智能处理**（`plugin/manager.go:366-397`）：
- 当配置解析或验证失败时，自动使用默认配置
- 原有配置被注释保留在文件中，方便用户排查
- 插件被自动禁用，需要用户确认后手动启用

## 三、对管理界面与运行时消息生成的影响

### 3.1 管理界面（UI）的表现

**插件列表页**（`ui/src/plugin/Plugins.tsx`）：
- 仅展示数据库中存在且实例加载成功的插件
- `GetPlugins()` API 会过滤掉 `Manager.Instance(conf.ID)` 返回错误的插件
- 悬挂配置（数据库有记录但插件文件已删除）不会出现在列表中

**启用/禁用操作**（`ui/src/plugin/PluginStore.ts:44-48`）：
- 前端通过 `POST /plugin/{id}/enable` 或 `/disable` 切换状态
- 后端验证：实例必须存在（`404 plugin instance not found`）
- 状态重复时返回 `400 config is already enabled/disabled`
- 插件内部 `Enable()`/`Disable()` 失败时返回 `500 Internal Server Error`

**配置更新**（`api/plugin.go:367-404`）：
- YAML 格式错误返回 `400 Bad Request`
- 插件验证失败返回 `400 Bad Request`（包含插件自定义错误信息）
- 更新成功后自动刷新插件列表

### 3.2 运行时消息生成的影响

**Messenger 能力插件**（`plugin/manager.go:335-341`）：
- 只有实例加载成功且具备 Messenger 能力的插件才会注册消息处理器
- 消息通过 channel `m.messages` 异步发送，不阻塞主流程
- 插件实例失败不会影响消息通道的运行

**Webhook 路由**（`plugin/manager.go:348-352`）：
- 路由注册使用 `requirePluginEnabled` 中间件
- 插件被禁用后，Webhook 自动返回 `400 Bad Request`
- 插件加载失败时，路由根本不会被注册，访问返回 `404 Not Found`

**内部应用标记**（`plugin/manager.go:294-310`）：
- 初始化时会扫描用户的所有应用
- 如果应用关联的插件已加载（存在于 `m.plugins`），标记为 `Internal: true`
- 否则标记为 `Internal: false`，用户可见可操作
- 这确保了插件卸载后，关联的应用可以被用户正常管理

## 四、测试夹具与 CI 失败场景复现

### 4.1 测试夹具分类

项目在 `plugin/testing/` 目录下提供了完整的测试夹具：

#### 4.1.1 Mock 插件（可控仿真）

**位置**：`plugin/testing/mock/mock.go`

**特点**：
- 纯 Go 实现，无需编译为 .so 文件
- 支持所有 5 种能力（Configurer, Storager, Messenger, Displayer, Webhooker）
- 提供错误注入能力：`ReturnErrorOnEnableForUser()` 和 `ReturnErrorOnDisableForUser()`
- 支持动态修改能力集：`SetCapability()`

**在测试中的用法**（`plugin/manager_test.go:72-77`）：

```go
p := new(mock.Plugin)
assert.Nil(s.T(), manager.LoadPlugin(p))
assert.Nil(s.T(), manager.initializeSingleUserPlugin(compat.UserContext{
    ID:    1,
    Admin: true,
}, p))
```

#### 4.1.2 Broken 插件（故障仿真）

**位置**：`plugin/testing/broken/`

包含 5 种故障场景，分别由不同测试套件覆盖：

| 目录 | 故障类型 | 对应中断点 | 测试覆盖 |
|------|----------|------------|----------|
| `nothing/` | 完全空的插件，无任何导出符号 | 中断点2（缺少 Info 符号） | ✅ wrap_test + manager_test |
| `noinstance/` | 有 Info 但无 NewGotifyPluginInstance | 中断点4（缺少构造函数） | ✅ wrap_test |
| `unknowninfo/` | Info 函数返回类型错误（string） | 中断点3（Info 签名错误） | ✅ wrap_test |
| `malformedconstructor/` | 构造函数返回类型错误（interface{}） | 中断点5（构造函数签名错误） | ✅ wrap_test |
| `cantinstantiate/` | 插件可加载但 Enable() 始终失败 | 中断点11（启用失败） | ⚠️ 间接覆盖（通过 mock） |

**注意**：5 个 broken 插件并非全部由 manager_test 覆盖，而是分层测试：
- **wrap_test.go** 负责测试 `compat.Wrap()` 层的符号检查失败
- **manager_test.go** 仅测试了 `nothing` 一个 broken 插件（验证 `loadPlugins()` 的错误传播）
- **cantinstantiate** 场景通过 mock 插件的错误注入间接测试

### 4.2 Broken 场景的测试覆盖详情

#### 4.2.1 wrap_test.go - TestWrapIncompatiblePlugins

**位置**：`plugin/compat/wrap_test.go:132-157`

**覆盖的 broken 场景**：4 个（nothing, noinstance, unknowninfo, malformedconstructor）

**构建方式**：
```go
for i, modulePath := range []string{
    "github.com/gotify/server/v2/plugin/testing/broken/noinstance",
    "github.com/gotify/server/v2/plugin/testing/broken/nothing",
    "github.com/gotify/server/v2/plugin/testing/broken/unknowninfo",
    "github.com/gotify/server/v2/plugin/testing/broken/malformedconstructor",
} {
    fName := tmpDir.Path(fmt.Sprintf("broken_%d.so", i))
    cmd := exec.Command("go", "build", "-buildmode=plugin", "-o="+fName, modulePath)
    assert.Nil(t, cmd.Run())
    
    plugin, err := plugin.Open(fName)
    assert.Nil(t, err)
    _, err = Wrap(plugin)
    assert.Error(t, err)  // 断言点：Wrap() 必须返回错误
}
```

**断言点**：`compat.Wrap(plugin)` 返回 `error != nil`

**覆盖的错误类型**：
- `nothing` → `missing GetGotifyPluginInfo symbol`
- `noinstance` → `missing NewGotifyPluginInstance symbol`
- `unknowninfo` → `unknown plugin version (unrecognized GetGotifyPluginInfo signature)`
- `malformedconstructor` → `NewGotifyPluginInstance signature mismatch`

#### 4.2.2 manager_test.go - TestInitializePlugin_brokenPlugin_expectError

**位置**：`plugin/manager_test.go:175-193`

**覆盖的 broken 场景**：仅 `nothing` 1 个

**构建方式**：
```go
func (s *ManagerSuite) TestInitializePlugin_brokenPlugin_expectError() {
    tmpDir := test.NewTmpDir("gotify_testbrokenplugin")
    defer tmpDir.Clean()
    test.WithWd(path.Join(test.GetProjectDir(), "./plugin/testing/broken/nothing"), 
        func(origWd string) {
            exec.Command("go", "get", "-d").Run()
            cmd := exec.Command("go", "build", "-buildmode=plugin", 
                "-o="+tmpDir.Path("empty.so"))
            assert.Nil(s.T(), cmd.Run())
    })
    assert.Error(s.T(), s.manager.loadPlugins(tmpDir.Path()))
}
```

**断言点**：`manager.loadPlugins()` 返回 `error != nil`

**测试目的**：验证 `Wrap()` 层的错误能够正确向上传播到 `Manager.loadPlugins()`，并被包装为 `pluginFileLoadError`。

#### 4.2.3 cantinstantiate 场景的间接覆盖

`cantinstantiate` 插件（Enable() 失败）没有被直接用于测试，而是通过 mock 插件的错误注入机制间接覆盖：

**覆盖方式**（`plugin/manager_test.go:224-238`）：
```go
func (s *ManagerSuite) TestInitializePlugin_alreadyEnabled_cannotEnable_disabledAutomatically() {
    s.db.NewUserWithName(4, "enable_fail_2")
    mock.ReturnErrorOnEnableForUser(4, errors.New("test error"))  // 错误注入
    s.db.CreatePluginConf(&model.PluginConf{
        UserID:     4,
        ModulePath: mockPluginPath,
        Token:      "P5478",
        Enabled:    true,
    })
    
    assert.Nil(s.T(), s.manager.InitializeForUserID(4))
    inst := s.getMockPluginInstance(4)
    assert.False(s.T(), inst.Enabled)  // 断言点：插件被自动禁用
    assert.False(s.T(), s.getConfForMockPlugin(4).Enabled)
}
```

**断言点**：
- `InitializeForUserID()` 不返回错误（实例级错误不向上传播）
- 实例 `Enabled` 状态为 `false`
- 数据库中 `PluginConf.Enabled` 为 `false`

### 4.3 测试用例覆盖的失败场景汇总

**manager_test.go** 中的 `ManagerSuite` 覆盖场景：

| 测试用例 | 覆盖场景 |
|----------|----------|
| `TestInitializePlugin_directoryInvalid_expectError` | 插件目录无效 |
| `TestInitializePlugin_invalidPlugin_expectError` | 非插件文件混入目录 |
| `TestInitializePlugin_brokenPlugin_expectError` | broken/nothing 插件（仅1个） |
| `TestInitializePlugin_alreadyLoaded_expectError` | 插件重复加载 |
| `TestInitializePlugin_alreadyEnabledInConf_failedToLoadConfig_disableAutomatically` | 无效 YAML 配置自动禁用 |
| `TestInitializePlugin_alreadyEnabled_cannotEnable_disabledAutomatically` | Enable() 返回错误自动禁用 |
| `TestInitializePlugin_userIDNotExist_expectError` | 用户不存在 |
| `TestSetPluginEnabled_EnableReturnsError_cannotEnable` | 启用时返回错误 |
| `TestSetPluginEnabled_DisableReturnsError_cannotDisable` | 禁用时返回错误 |
| `TestRemoveUser_DisableFail_cannotRemove` | 用户删除时禁用失败 |
| `TestNewManager_CannotLoadDirectory_expectError` | Manager 初始化目录错误 |
| `TestNewManager_NonPluginFile_expectError` | 非 .so 文件导致加载失败 |
| `TestNewManager_InternalApplicationManagement` | 插件卸载后应用标记修正 |

**wrap_test.go** 补充覆盖场景：

| 测试用例 | 覆盖场景 |
|----------|----------|
| `TestWrapIncompatiblePlugins` | 4 种符号检查失败场景 |

### 4.4 覆盖缺口分析

| 缺口 | 说明 | 风险 |
|------|------|------|
| cantinstantiate 未直接测试 | 该插件存在但未被任何测试直接加载，仅通过 mock 模拟 | 低，mock 已覆盖等价场景 |
| manager_test 未覆盖全部 wrap 错误类型 | manager_test 仅测试了 nothing，其余 3 种 wrap 错误未在 manager 层验证传播路径 | 中，wrap 层已测试，但 manager 的错误包装逻辑未全量验证 |
| 缺少 plugin.Open() 失败的专用测试 | 动态链接库加载失败（如文件损坏）通过非 .so 文件间接测试 | 低，`TestNewManager_NonPluginFile_expectError` 已覆盖等价路径 |

### 4.3 CI 中的测试执行

**CI 配置**（`.github/workflows/build.yml`）：

```yaml
- run: make test
```

**Makefile 测试命令**（`Makefile:20-22`）：

```makefile
test-coverage:
    go test --race -coverprofile=coverage.txt -covermode=atomic -coverpkg=./... ./...
```

**测试特点**：
- 使用 `--race` 标志检测并发问题
- 生成覆盖率报告上传到 Codecov
- 插件测试使用构建标签 `//go:build linux || darwin`，仅在 Linux/macOS 上运行（Windows 不支持 Go plugin）

### 4.4 悬挂配置处理的测试

**悬挂配置**（Dangling Config）指数据库中存在 PluginConf 记录，但对应的插件文件已被删除。

**测试构造**（`plugin/manager_test.go:109-118`）：

```go
func (s *ManagerSuite) makeDanglingPluginConf(uid uint) *model.PluginConf {
    conf := &model.PluginConf{
        UserID:     uid,
        ModulePath: danglingPluginPath, // "github.com/.../testing/removed"
        Token:      auth.GeneratePluginToken(),
        Enabled:    true,
    }
    s.db.CreatePluginConf(conf)
    return conf
}
```

**测试验证**：
- `TestAddRemoveNewUser`：验证悬挂配置不影响新用户初始化
- `TestRemoveUser_danglingConf_expectSuccess`：验证用户删除时能正确处理悬挂配置
- `TestNewManager_InternalApplicationManagement`：验证插件缺失时应用 Internal 标记被正确清除

## 五、总结

### 5.1 失败处理设计原则

1. **故障隔离**：单个插件失败不影响其他插件和服务整体运行
2. **优雅降级**：实例级错误自动禁用，保留配置供用户修复
3. **透明反馈**：管理界面清晰展示插件状态，错误信息明确
4. **数据一致性**：插件加载状态与数据库配置保持同步

### 5.2 测试夹具设计原则

1. **分层测试**：Mock 插件用于快速单元测试，Broken 插件用于真实加载场景
2. **故障注入**：支持在特定用户、特定操作上注入错误
3. **边界覆盖**：覆盖从文件读取到实例启用的全链路故障点
4. **CI 友好**：测试可自动化执行，支持覆盖率统计

### 5.3 关键代码参考

| 功能 | 文件 | 行号 |
|------|------|------|
| Manager 初始化 | `plugin/manager.go` | 57-101 |
| 插件文件加载 | `plugin/manager.go` | 219-254 |
| 符号检查与包装 | `plugin/compat/wrap.go` | 11-35 |
| 用户实例初始化 | `plugin/manager.go` | 315-364 |
| 配置失败处理 | `plugin/manager.go` | 366-397 |
| 启用失败处理 | `plugin/manager.go` | 353-362 |
| Mock 插件实现 | `plugin/testing/mock/mock.go` | 1-177 |
| Manager 测试套件 | `plugin/manager_test.go` | 1-452 |
