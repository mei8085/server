# API 文档生成与 Spec 校验测试说明

## 1. 概述

本文档详细说明 Gotify 服务器对外 API 文档的生成机制、文档结构约束、与控制器实现的漂移检测方法，以及测试夹具如何反向校验 spec 与实现的一致性。

## 2. 路由元信息抽取成文档

### 2.1 注释驱动的文档生成

项目采用 **go-swagger** 工具链，通过代码注释直接生成 API 文档。路由元信息的抽取主要依赖以下两种注释方式：

#### 2.1.1 操作级注释 (`swagger:operation`)

在每个 API 处理函数上方，使用 `swagger:operation` 注释定义路由元信息。格式如下：

```go
// swagger:operation GET /message message getMessages
//
// Return all messages.
//
// ---
// produces: [application/json]
// security: [clientTokenAuthorizationHeader: [], clientTokenHeader: [], clientTokenQuery: [], basicAuth: []]
// parameters:
// - name: limit
//   in: query
//   description: the maximal amount of messages to return
//   required: false
//   maximum: 200
//   minimum: 1
//   default: 100
//   type: integer
// responses:
//   200:
//     description: Ok
//     schema:
//         $ref: "#/definitions/PagedMessages"
//   401:
//     description: Unauthorized
//     schema:
//         $ref: "#/definitions/Error"
func (a *MessageAPI) GetMessages(ctx *gin.Context) {
    // 实现代码
}
```
[api/message.go:47-88](api/message.go:47-88)

**注释字段说明：**
- `swagger:operation <METHOD> <PATH> <TAG> <OPERATION_ID>`：定义 HTTP 方法、路径、标签和操作 ID
- `summary/description`：接口描述
- `produces/consumes`：指定 MIME 类型
- `security`：声明所需的认证方式
- `parameters`：定义请求参数（路径、查询、请求体等）
- `responses`：定义响应状态码和数据结构

#### 2.1.2 模型级注释 (`swagger:model`)

数据模型通过 `swagger:model` 注释定义，用于生成文档中的 `definitions` 部分：

```go
// MessageExternal Model
//
// The MessageExternal holds information about a message which was sent by an Application.
//
// swagger:model Message
type MessageExternal struct {
    // The message id.
    //
    // read only: true
    // required: true
    // example: 25
    ID uint `json:"id"`
    // The message. Markdown (excluding html) is allowed.
    //
    // required: true
    // example: **Backup** was successfully finished.
    Message string `json:"message" binding:"required"`
    // ... 其他字段
}
```
[model/message.go:18-66](model/message.go:18-66)

**模型注释支持：**
- `required: true/false`：标记必填字段
- `read only: true`：标记只读字段
- `example: <value>`：提供示例值
- `description`：字段描述

#### 2.1.3 元数据注释 (`swagger:meta`)

在 `docs/package.go` 中定义 API 的全局元信息：

```go
// Package docs Gotify REST-API.
//
// This is the documentation of the Gotify REST-API.
//
// # Authentication
// ... 认证说明 ...
//
// Schemes: http, https
// Host: localhost
// Version: 2.1.0
// License: MIT https://github.com/gotify/server/blob/master/LICENSE
//
// Consumes:
// - application/json
//
// Produces:
// - application/json
//
// SecurityDefinitions:
//    appTokenQuery:
//       type: apiKey
//       name: token
//       in: query
// ... 其他安全定义 ...
//
// swagger:meta
package docs
```
[docs/package.go:1-62](docs/package.go:1-62)

### 2.2 生成命令

文档生成通过 Makefile 中的 `update-swagger` 目标执行：

```makefile
update-swagger:
    swagger generate spec --scan-models -o docs/spec.json
    sed -i 's/"uint64"/"int64"/g' docs/spec.json
```
[Makefile:41-43](Makefile:41-43)

**生成过程：**
1. `swagger generate spec --scan-models`：扫描所有 Go 文件，提取注释生成 Swagger 2.0 规范
2. `sed` 命令：将 `uint64` 类型替换为 `int64`（Swagger 2.0 不支持无符号整数）
3. 输出结果保存为 `docs/spec.json`

## 3. 文档结构与版本约束

### 3.1 文档结构

生成的 `docs/spec.json` 遵循 Swagger 2.0 规范，主要包含以下部分：

```json
{
  "swagger": "2.0",
  "info": {
    "title": "Gotify REST-API.",
    "description": "...",
    "version": "2.1.0",
    "license": { "name": "MIT", "url": "..." }
  },
  "host": "localhost",
  "schemes": ["http", "https"],
  "consumes": ["application/json"],
  "produces": ["application/json"],
  "securityDefinitions": { ... },
  "paths": {
    "/application": {
      "get": { ... },
      "post": { ... }
    },
    ...
  },
  "definitions": {
    "Application": { ... },
    "Message": { ... },
    "Error": { ... },
    ...
  }
}
```

**核心结构说明：**
- `info`：API 基本信息（标题、描述、版本、许可证）
- `securityDefinitions`：定义 6 种认证方式（app/client token 的 header/query/authorization 形式 + basic auth）
- `paths`：所有 API 路由定义
- `definitions`：所有数据模型定义

### 3.2 版本约束

#### 3.2.1 API 版本号

- 当前版本：`2.1.0`（定义在 `docs/package.go` 中）
- 版本号遵循 [语义化版本](https://semver.org/) 规范
- 主版本号变更表示不兼容的 API 改动

#### 3.2.2 规范版本

- 使用 Swagger 2.0 规范（非 OpenAPI 3.x）
- 这是由 go-swagger 工具链的支持决定的

### 3.3 文档服务

生成的 spec.json 通过 `/swagger` 端点提供服务：

```go
// Serve serves the documentation.
func Serve(ctx *gin.Context) {
    base := location.Get(ctx).Host
    if basePathFromQuery := ctx.Query("base"); basePathFromQuery != "" {
        base = basePathFromQuery
    }
    ctx.Writer.WriteString(getSwaggerJSON(base))
}

func getSwaggerJSON(base string) string {
    return strings.Replace(spec, "localhost", base, 1)
}
```
[docs/swagger.go:14-25](docs/swagger.go:14-25)

**特性：**
- 支持通过 `?base=` 查询参数自定义 host
- 动态替换文档中的 `localhost` 为实际访问的 host
- 配合 Swagger UI 在 `/docs` 端点提供交互式文档界面

## 4. 与控制器实现的漂移检测

### 4.1 漂移检测机制

为防止 API 文档与实际实现产生漂移（文档描述与实际行为不一致），项目采用 **Git 状态检查** 机制：

```makefile
check-swagger: update-swagger
## add the docs to git, this changes line endings in git, otherwise this does not work on windows
    git add docs
    if [ -n "$(shell git status --porcelain | grep docs)" ]; then \
        echo Swagger Spec is not up-to-date; \
        exit 1; \
    fi
```
[Makefile:45-51](Makefile:45-51)

**检测流程：**
1. 执行 `update-swagger` 重新生成 spec.json
2. 将 `docs/` 目录的变更添加到 git 暂存区
3. 检查 `docs/` 目录是否有未提交的变更
4. 如果有变更，说明代码注释与已提交的 spec.json 不一致，测试失败

### 4.2 漂移类型与检测覆盖

| 漂移类型 | 检测方式 | 触发场景 |
|---------|---------|---------|
| 新增/删除 API 路由 | git diff 检测 | 新增或删除 swagger:operation 注释 |
| 修改路由路径 | git diff 检测 | 修改 swagger:operation 中的 PATH |
| 修改参数定义 | git diff 检测 | 修改 parameters 块中的内容 |
| 修改响应结构 | git diff 检测 | 修改 responses 块或模型定义 |
| 修改认证要求 | git diff 检测 | 修改 security 块 |
| 修改数据模型 | git diff 检测 | 修改 swagger:model 注释或字段 tag |

### 4.3 CI 集成

`check-swagger` 目标被集成到 CI 流程中：

```makefile
check-ci: check-swagger check-js
```
[Makefile:14](Makefile:14)

这确保每次代码提交都必须保持 API 文档与实现的一致性。

### 4.4 路由注册、注释声明与 Spec 生成的逐层校验关系

API 文档生成涉及三层独立的定义，每层之间可能产生漂移，当前校验机制如下：

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 1: 路由注册 (router/router.go)                        │
│  - gin 路由定义: g.GET("/message", handler)                 │
│  - 中间件绑定: Use(authentication.RequireClient)           │
│  - HTTP 方法与路径的最终来源                               │
├─────────────────────────────────────────────────────────────┤
│ Layer 2: 注释声明 (api/*.go)                                │
│  - swagger:operation 注释定义路径、方法、参数、响应        │
│  - swagger:model 注释定义数据模型                           │
│  - 这是 spec.json 的直接输入源                              │
├─────────────────────────────────────────────────────────────┤
│ Layer 3: Spec 生成 (docs/spec.json)                         │
│  - go-swagger 工具从注释生成                                │
│  - 对外发布的 API 规范                                      │
└─────────────────────────────────────────────────────────────┘
```

**逐层校验关系：**

| 校验方向 | 当前机制 | 校验粒度 |
|---------|---------|---------|
| Layer 2 → Layer 3 | `check-swagger` 通过 git diff 检测 | 完整 spec 重生成对比 |
| Layer 1 → Layer 2 | **无自动校验**，依赖人工确保 | 无 |
| Layer 1 → Layer 3 | **无自动校验**，依赖集成测试间接覆盖 | 部分覆盖 |

**关键脱节风险：**
- **路由路径不一致**：`router.go` 中注册的路径与 `swagger:operation` 注释中的路径可能不同
- **HTTP 方法不一致**：实际绑定的方法与注释声明的方法可能不同
- **中间件不一致**：实际使用的认证中间件与注释中的 `security` 声明可能不同
- **参数绑定不一致**：代码中实际绑定的参数与注释声明的参数可能不同

### 4.5 漂移类型分类：可自动发现 vs 无法发现

#### 4.5.1 可自动发现的漂移类型

| 漂移类型 | 发现机制 | 触发条件 |
|---------|---------|---------|
| 新增/删除 API 注释 | `check-swagger` git diff | 增删 `swagger:operation` 注释 |
| 修改注释中的路径 | `check-swagger` git diff | 修改 `swagger:operation <METHOD> <PATH>` 中的 PATH |
| 修改注释中的 HTTP 方法 | `check-swagger` git diff | 修改 `swagger:operation <METHOD> <PATH>` 中的 METHOD |
| 修改参数定义 | `check-swagger` git diff | 修改 `parameters` 块 |
| 修改响应结构 | `check-swagger` git diff | 修改 `responses` 块或模型字段 |
| 修改注释中的认证声明 | `check-swagger` git diff | 修改 `security` 块 |
| 修改数据模型字段 | `check-swagger` git diff | 修改 `swagger:model` 结构体字段或 tag |
| 修改全局元数据 | `check-swagger` git diff | 修改 `swagger:meta` 注释 |

#### 4.5.2 当前无法自动发现的漂移类型

| 漂移类型 | 风险说明 | 为何无法发现 |
|---------|---------|-------------|
| **路由注册路径 vs 注释路径不一致** | 注释写 `/message`，实际注册 `/messages` | go-swagger 只扫描注释，不扫描 gin 路由注册代码 |
| **HTTP 方法不匹配** | 注释写 `GET`，实际注册 `POST` | 同上，注释与路由注册是独立的 |
| **认证中间件 vs security 声明不一致** | 注释声明需要 client token，实际路由未加认证中间件 | 中间件绑定在 router.go，security 声明在注释中，无交叉校验 |
| **实际响应状态码缺失** | 代码实际返回 409 Conflict，但注释未声明 | check-swagger 只检查注释是否变化，不检查实现是否匹配注释 |
| **参数验证规则不一致** | 注释声明 `minimum: 1`，代码中 `binding:"min=0"` | 注释约束与 Go struct tag 独立，无交叉校验 |
| **请求体模型不匹配** | 注释引用 `ApplicationParams`，代码实际绑定 `Application` | 需要静态分析或运行时测试发现 |
| **响应模型不匹配** | 注释声明返回 `User`，代码实际返回 `UserExternal` | 同上 |
| **路由实际不存在** | 注释声明了 API，但 router.go 中未注册 | 注释存在但无实际路由，go-swagger 仍会生成 spec |

## 5. 测试夹具反向校验 Spec 与实现一致性

### 5.1 测试架构概述

项目采用 **分层测试策略**，通过测试夹具和断言工具确保 API 实现与文档规范一致。

#### 5.1.1 测试层级

```
┌─────────────────────────────────────────┐
│ 集成测试 (router/router_test.go)        │
│  - 启动完整 HTTP 服务器                 │
│  - 端到端验证路由、认证、响应           │
├─────────────────────────────────────────┤
│ API 单元测试 (api/*_test.go)            │
│  - 直接调用 API 处理函数                │
│  - 验证业务逻辑和响应格式               │
├─────────────────────────────────────────┤
│ 测试夹具 (test/testdb/, test/*.go)      │
│  - 内存数据库                           │
│  - 认证模拟                             │
│  - 断言工具                             │
└─────────────────────────────────────────┘
```

### 5.2 测试夹具详解

#### 5.2.1 数据库夹具 (`test/testdb`)

提供内存 SQLite 数据库和流畅的构建器 API：

```go
// Database is the wrapper for the gorm database with sleek helper methods.
type Database struct {
    *database.GormDatabase
    t *testing.T
}

// 使用示例：创建用户 -> 创建应用 -> 创建消息
func (s *MessageSuite) Test_GetMessages() {
    user := s.db.User(5)           // 创建 ID 为 5 的用户
    first := user.App(1).NewMessage(1)   // 用户创建应用 1，再创建消息 1
    second := user.App(2).NewMessage(2)  // 用户创建应用 2，再创建消息 2
    
    test.WithUser(s.ctx, 5)        // 模拟用户认证
    s.a.GetMessages(s.ctx)         // 调用 API
    
    // 验证响应
    expected := &model.PagedMessages{...}
    test.BodyEquals(s.T(), expected, s.recorder)
}
```
[test/testdb/database.go:13-248](test/testdb/database.go:13-248)

**构建器模式：**
- `db.User(id)` → AppClientBuilder
- `user.App(id)` → MessageBuilder
- `app.NewMessage(id)` → 创建消息

#### 5.2.2 认证夹具 (`test/auth.go`)

模拟用户认证，跳过实际的 token 验证：

```go
// WithUser fake an authentication for testing.
func WithUser(ctx *gin.Context, userID uint) {
    auth.RegisterUser(ctx, &model.User{ID: userID})
}
```
[test/auth.go:9-11](test/auth.go:9-11)

#### 5.2.3 断言工具 (`test/asserts.go`)

提供 JSON 比较和响应验证工具：

```go
// BodyEquals 验证响应体与预期对象的 JSON 表示一致
// JSONEquals 比较两个对象的 JSON 序列化结果
```

### 5.3 反向校验机制

测试夹具通过以下方式确保实现与 spec 一致：

#### 5.3.1 请求参数校验

测试用例根据文档中的参数定义构造请求，验证实现是否正确处理：

```go
func (s *MessageSuite) Test_GetMessages_BadRequestOnInvalidLimit() {
    s.db.User(5)
    test.WithUser(s.ctx, 5)
    s.withURL("http", "example.com", "/messages", "limit=555")  // 超过文档中的 maximum: 200
    s.a.GetMessages(s.ctx)
    
    assert.Equal(s.T(), 400, s.recorder.Code)  // 验证返回 400 Bad Request
}
```
[api/message_test.go:132-139](api/message_test.go:132-139)

#### 5.3.2 响应结构校验

测试用例根据文档中的响应 schema 验证实际返回：

```go
func (s *MessageSuite) Test_ensureCorrectJsonRepresentation() {
    t, _ := time.Parse("2006/01/02", "2017/01/02")
    
    actual := &model.PagedMessages{
        Paging: model.Paging{Limit: 5, Since: 122, Size: 5, Next: "http://example.com/message?limit=5&since=122"},
        Messages: []*model.MessageExternal{{ID: 55, ApplicationID: 2, Message: "hi", Title: "hi", Date: t, Priority: intPtr(4)}},
    }
    
    // 验证 JSON 结构与文档定义一致
    test.JSONEquals(s.T(), actual, `{"paging": {"limit":5, "since": 122, "size": 5, "next": "..."}, "messages": [...]}`)
}
```
[api/message_test.go:51-65](api/message_test.go:51-65)

#### 5.3.3 集成测试端到端校验

`router/router_test.go` 中的集成测试启动完整的 HTTP 服务器，验证整个请求链路：

```go
func (s *IntegrationSuite) TestVersionInfo() {
    req := s.newRequest("GET", "version", "")
    doRequestAndExpect(s.T(), req, 200, `{"version":"1.0.0", "commit":"asdasds", "buildDate":"2018-02-20-17:30:47"}`)
}
```
[router/router_test.go:56-60](router/router_test.go:56-60)

### 5.4 校验覆盖范围

| Spec 元素 | 测试校验方式 | 示例文件 |
|----------|-------------|---------|
| HTTP 方法 | 集成测试构造对应方法请求 | router/router_test.go |
| 路径参数 | 测试用例构造包含路径参数的 URL | api/message_test.go |
| 查询参数 | 验证参数绑定、验证逻辑 | api/message_test.go |
| 请求体 | 构造 JSON 请求体验证绑定 | api/application_test.go |
| 认证要求 | 测试有无认证时的不同响应 | api/user_test.go |
| 响应状态码 | 断言 HTTP 状态码 | 所有测试 |
| 响应 JSON 结构 | JSONEquals 比较完整结构 | api/message_test.go |
| 错误响应格式 | 验证 Error 模型结构 | api/*_test.go |

## 6. 工作流总结

### 6.1 开发流程

```
开发者修改 API 代码
        ↓
添加/更新 swagger 注释
        ↓
运行 make update-swagger 生成 spec.json
        ↓
编写/更新测试用例
        ↓
运行 make test 执行测试
        ↓
运行 make check-swagger 检查文档一致性
        ↓
提交代码
```

### 6.2 保障机制

1. **代码即文档**：API 文档直接从代码注释生成，避免人工维护
2. **漂移检测**：CI 自动检查 spec.json 是否与代码注释一致
3. **测试驱动**：测试用例根据 spec 编写，验证实现正确性
4. **分层验证**：单元测试 + 集成测试双重保障

## 7. 最佳实践与注意事项

### 7.1 注释编写规范

1. **必须包含**：每个公开 API 必须有 `swagger:operation` 注释
2. **参数完整**：所有参数（路径、查询、请求体）必须在注释中声明
3. **响应完整**：所有可能的响应状态码必须声明
4. **模型引用**：使用 `$ref` 引用定义好的模型，避免内联定义
5. **示例值**：为字段提供合理的 example 值，便于文档使用者理解

### 7.2 测试编写规范

1. **正向测试**：验证合法输入返回预期结果
2. **边界测试**：验证参数边界值（如 limit 的最大/最小值）
3. **错误测试**：验证非法输入返回正确的错误响应
4. **认证测试**：验证有无认证、不同权限的响应差异
5. **JSON 结构**：使用 `JSONEquals` 验证完整的响应结构

### 7.3 常见陷阱

1. **忘记更新注释**：修改代码后必须同步更新 swagger 注释
2. **注释与实现不一致**：注释中的参数名、类型必须与实际代码一致
3. **模型不匹配**：Go 结构体字段的 json tag 必须与 spec 中的字段名一致
4. **uint64 问题**：Swagger 2.0 不支持 uint64，生成时会自动替换为 int64

## 8. 附录

### 8.1 相关文件清单

| 文件路径 | 作用 |
|---------|------|
| `docs/package.go` | API 元数据和全局安全定义 |
| `docs/swagger.go` | Swagger JSON 服务端点 |
| `docs/spec.json` | 生成的 Swagger 规范文件 |
| `docs/ui.go` | Swagger UI 页面 |
| `Makefile` | 包含 update-swagger 和 check-swagger 目标 |
| `api/*.go` | API 控制器实现及 swagger:operation 注释 |
| `model/*.go` | 数据模型定义及 swagger:model 注释 |
| `test/testdb/database.go` | 数据库测试夹具 |
| `test/auth.go` | 认证测试夹具 |
| `test/asserts.go` | 测试断言工具 |
| `router/router.go` | 路由注册 |
| `router/router_test.go` | 集成测试 |

### 8.2 工具版本

- go-swagger: `717e3cb29becaaf00e56953556c6d80f8a01b286`
- Swagger 规范版本: 2.0
- API 版本: 2.1.0
