# 静态图标缓存行为深度代码分析（v2 修正版）

> 本文基于 Go 标准库 `net/http/fs.go`、`embed` 包源码行为，以及 Gin/gin-contrib/gzip 的真实实现，逐行核对 Gotify 源码。

---

## 一、四类静态资源的返回路径与响应头总览

| 资源类别 | 典型 URL | 谁返回 | 资源来源 | ETag | Last-Modified | 其他缓存/编码头 |
|---------|---------|--------|---------|------|---------------|-----------------|
| 页面入口 `index.html` | `/`、`/index.html` | `serveFile()` → `ctx.String()` | `embed.FS` 读内存 + `%CONFIG%` 注入 | ❌ 无 | ❌ 无 | `Content-Encoding: gzip`、`Vary: Accept-Encoding`（条件触发） |
| 应用清单 `manifest.json` | `/manifest.json` | `serveFile()` → `ctx.String()` | `embed.FS` 读内存（`noop` 不做转换） | ❌ 无 | ❌ 无 | `Content-Encoding: gzip`、`Vary: Accept-Encoding`（条件触发） |
| 内置静态图标 | `/static/favicon-32x32.png`、`/static/defaultapp.png` 等 | `http.FileServer(http.FS(embed.FS))` → `gin.WrapH()` | `embed.FS` 只读内存文件系统 | ❌ 无 | ❌ **无**（ModTime 为零值） | `Content-Type`、`Content-Length`、`Accept-Ranges: bytes` |
| 用户上传图标 | `/image/aB3dE_fGhIjKlMnOpQrStUvWx.png` | `gin.StaticFS()` → 底层 `http.FileServer(gin.Dir(...))` | 本地磁盘 `data/images/` | ❌ 无 | ✅ **有**（真实文件 mtime） | `Content-Type`、`Content-Length`、`Accept-Ranges: bytes`；无 gzip 压缩 |

---

## 二、逐类资源深度代码分析

### 2.1 页面入口文件：index.html

#### 谁返回、资源来自哪里

路由注册在 [ui/serve.go#L25-L44](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/serve.go#L25-L44)：

```go
func Register(r *gin.Engine, ...) {
    // ... 提前把配置序列化成 JSON ...
    replaceConfig := func(content string) string {
        return strings.Replace(content, "%CONFIG%", string(uiConfigBytes), 1)
    }

    // gzip 中间件仅作用在这个 Group 下
    ui := r.Group("/", gzip.Gzip(gzip.DefaultCompression))
    ui.GET("/", serveFile("index.html", "text/html", replaceConfig))
    ui.GET("/index.html", serveFile("index.html", "text/html", replaceConfig))
    // ...
}
```

`serveFile` 实现 [ui/serve.go#L51-L61](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/serve.go#L51-L61)：

```go
func serveFile(name, contentType string, convert func(string) string) gin.HandlerFunc {
    // ① 服务器启动时一次性从 embed.FS 读入内存并做转换
    content, _ := box.ReadFile("build/" + name)  // box 是 //go:embed build/*
    converted := convert(string(content))         // replaceConfig 注入 %CONFIG%

    // ② 返回一个闭包 handler，后续每次请求直接从内存返回
    return func(ctx *gin.Context) {
        ctx.Header("Content-Type", contentType)  // 仅设置这一个响应头
        ctx.String(200, converted)                // 写入响应体 + 状态码
    }
}
```

**关键事实：**
- 文件内容在进程启动时就已读入内存并完成 `%CONFIG%` 替换，后续请求不做任何磁盘 I/O
- `box.ReadFile` 来自 `embed.FS`，Go 标准库源码保证其 `ModTime()` 返回 `time.Time{}`（零值）
- 但这里根本没走 `http.FileServer`，而是直接 `ctx.String()`

#### 代码里实际会带的响应头

`gin.Context.String()` 的行为（Gin 框架 `render.String` 渲染器）：
1. 若未设置 `Content-Type`，自动设为 `text/plain; charset=utf-8`
2. 向 `ResponseWriter` 写入字符串内容
3. **不设置任何与缓存相关的头**（无 ETag、无 Last-Modified、无 Cache-Control、无 Expires）

结合 gzip 中间件 `gin-contrib/gzip` 的行为：
- 当请求头含 `Accept-Encoding: gzip` 且响应体超过最小阈值（默认 1024 字节）时，压缩响应
- 压缩后额外设置：`Content-Encoding: gzip`、`Vary: Accept-Encoding`
- gzip 中间件默认排除 `.png`、`.gif`、`.jpeg`、`.jpg` 等已压缩格式（但 index.html 是 text/html，会被压缩）

**index.html 的实际响应头清单：**

| 响应头 | 是否存在 | 来源 |
|-------|---------|------|
| `Content-Type: text/html` | ✅ 始终 | `ctx.Header("Content-Type", ...)` |
| `Content-Length` / `Transfer-Encoding: chunked` | ✅ 始终 | Go `net/http` 标准库自动处理 |
| `Content-Encoding: gzip` | ✅ 条件 | gzip 中间件（Accept-Encoding 含 gzip 且 body 够大） |
| `Vary: Accept-Encoding` | ✅ 条件 | gzip 中间件 |
| `ETag` | ❌ 无 | — |
| `Last-Modified` | ❌ 无 | — |
| `Cache-Control` | ❌ 无 | — |
| `Expires` | ❌ 无 | — |

**浏览器缓存行为：** 由于没有任何缓存头，浏览器每次访问都会重新发起请求（无协商缓存可用）。配合 gzip 压缩，每次都是 200 OK + 压缩后的 HTML 内容。

---

### 2.2 应用清单：manifest.json

#### 谁返回、资源来自哪里

路由注册 [ui/serve.go#L38](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/serve.go#L38)：

```go
ui.GET("/manifest.json", serveFile("manifest.json", "application/json", noop))
```

与 index.html 完全相同的 `serveFile` 路径，区别仅在于：
- `Content-Type: application/json`
- `convert` 函数是 `noop`（不做任何内容替换）
- 源文件是 [ui/public/manifest.json](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/public/manifest.json)，构建时被 Vite 复制到 `ui/build/manifest.json`，再被 `embed.FS` 嵌入

manifest.json 的内容：

```json
{
  "short_name": "Gotify",
  "name": "Gotify WebApp",
  "start_url": "./index.html",
  "display": "standalone",
  "theme_color": "#3f51b5",
  "background_color": "#303030"
}
```

#### 代码里实际会带的响应头

与 index.html 完全一致：

| 响应头 | 是否存在 |
|-------|---------|
| `Content-Type: application/json` | ✅ 始终 |
| `Content-Encoding: gzip`、`Vary: Accept-Encoding` | ✅ 条件 |
| `ETag` / `Last-Modified` / `Cache-Control` / `Expires` | ❌ 无 |

---

### 2.3 内置静态图标：/static/*

#### 谁返回、资源来自哪里

路由注册 [ui/serve.go#L40-L44](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/serve.go#L40-L44)：

```go
subBox, _ := fs.Sub(box, "build")           // 从 embed.FS 中提取 build/ 子目录
ui.GET("/static/*any", gin.WrapH(http.FileServer(http.FS(subBox))))
```

调用链：
1. `box` 是 `//go:embed build/*` 的 `embed.FS`
2. `fs.Sub(box, "build")` 把子目录变成根
3. `http.FS(subBox)` 把 `fs.FS` 适配成 `http.FileSystem`
4. `http.FileServer(...)` 创建标准库文件处理器
5. `gin.WrapH(...)` 把 `http.Handler` 适配成 `gin.HandlerFunc`，使 `http.ResponseWriter` 和 `*http.Request` 正常透传

**物理资源路径**（[ui/public/static/](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/public/static)）：

```
ui/public/static/
├── apple-touch-icon-57x57.png ~ 152x152.png  (8种)
├── favicon-16x16.png ~ 196x196.png            (5种)
├── favicon.ico
├── mstile-70x70.png ~ 310x310.png             (5种)
├── defaultapp.png            ← 应用默认图标
└── notification.ogg
```

构建流程：Vite 原样复制 → `ui/build/static/*` → `//go:embed build/*` 嵌入二进制。

#### 代码里实际会带的响应头

关键在于两个 Go 标准库的行为：

**行为 A：`embed.FS` 的 ModTime 为零值**

Go 标准库 `embed` 包源码（[embed.go#L220](https://cs.opensource.google/go/go/+/refs/tags/go1.23.1:src/embed/embed.go;l=220)）：

```go
type file struct { /* ... */ }
func (f *file) ModTime() time.Time { return time.Time{} }  // 返回零值：0001-01-01 00:00:00 UTC
```

设计原因：构建可重现性（reproducible builds）——相同内容的文件在不同时间编译应产生完全相同的二进制。

**行为 B：`http.FileServer` 只在 ModTime 非零时才设置 Last-Modified**

Go 标准库 `net/http/fs.go`（[fs.go#L616-L620](https://cs.opensource.google/go/go/+/refs/tags/go1.23.1:src/net/http/fs.go;l=616-620)）：

```go
func setLastModified(w ResponseWriter, modtime time.Time) {
    if !isZeroTime(modtime) {
        w.Header().Set("Last-Modified", modtime.UTC().Format(TimeFormat))
    }
}
```

**行为 C：`http.FileServer` 默认不生成 ETag**

Go 标准库的 `http.ServeContent` / `http.FileServer` **默认不会自动计算并设置 ETag 头**。ETag 需要开发者手动实现（例如基于文件内容哈希或 size+mtime 拼接）。

**行为 D：gzip 中间件跳过图片格式**

`gin-contrib/gzip` 默认不压缩 `.png`、`.gif`、`.jpeg`、`.jpg`（已压缩格式），因此请求 `/static/*.png` 时：
- gzip 中间件检测到扩展名，跳过压缩
- 不会出现 `Content-Encoding: gzip` 和 `Vary: Accept-Encoding`

**/static/* 图标的实际响应头清单：**

| 响应头 | 是否存在 | 来源/原因 |
|-------|---------|----------|
| `Content-Type`（如 `image/png`） | ✅ 始终 | `http.FileServer` 按扩展名或内容嗅探 |
| `Content-Length` | ✅ 始终 | `embed.FS` 文件元数据 |
| `Accept-Ranges: bytes` | ✅ 始终 | `http.FileServer` 默认支持 |
| `Last-Modified` | ❌ **无** | `embed.FS.ModTime()` 返回零值，`setLastModified` 判断为 isZeroTime，跳过设置 |
| `ETag` | ❌ **无** | Go `http.FileServer` 默认不生成 ETag |
| `Cache-Control` | ❌ 无 | 项目代码未显式设置 |
| `Expires` | ❌ 无 | 项目代码未显式设置 |
| `Content-Encoding: gzip` | ❌ 无 | gzip 中间件跳过图片格式 |

**浏览器缓存行为推论：**
- 没有 ETag，没有 Last-Modified → 浏览器无法发起条件请求（`If-None-Match` / `If-Modified-Since`）
- 没有 `Cache-Control` / `Expires` → 浏览器无法做有效缓存决策
- 实际表现：每次请求都会 200 OK 下载完整内容（除非浏览器做了启发式缓存，但因缺少校验头，刷新时通常会重新下载）

---

### 2.4 用户上传图标：/image/*

#### 谁返回、资源来自哪里

路由注册 [router/router.go#L131](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/router/router.go#L131)：

```go
g.StaticFS("/image", &onlyImageFS{inner: gin.Dir(conf.UploadedImagesDir, false)})
```

调用链：
1. `conf.UploadedImagesDir` 默认值在 [config/config.go#L26](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/config/config.go#L26)：`"data/images"`
2. `gin.Dir("data/images", false)` 创建 `http.FileSystem`（`false` 表示不列出目录）
3. `onlyImageFS` 包裹一层做扩展名白名单校验（只允许 `.gif/.png/.jpg/.jpeg`）
4. `g.StaticFS()` 内部用 `http.FileServer` 提供服务

文件类型校验层 [router/router.go#L279-L289](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/router/router.go#L279-L289)：

```go
type onlyImageFS struct {
    inner http.FileSystem
}

func (fs *onlyImageFS) Open(name string) (http.File, error) {
    ext := filepath.Ext(name)
    if !api.ValidApplicationImageExt(ext) {  // 白名单：.gif .png .jpg .jpeg
        return nil, fmt.Errorf("invalid file")
    }
    return fs.inner.Open(name)
}
```

**重要路由顺序说明**：
- `/image` 路由（第 131 行）注册在 gzip 中间件组之外（gzip 只在第 35 行的 `ui.Group` 下）
- `/image` 路由也注册在全局 `Content-Type: application/json` 中间件（第 135-140 行）**之前**
- 因此 `/image/*` **不受 gzip 压缩**，也**不会被强制设置 `Content-Type: application/json`**

#### 代码里实际会带的响应头

与 `/static/*` 的核心区别：底层是真实磁盘文件，有正常的 mtime。

Go 标准库 `http.FileServer` 对真实文件的行为：
- `os.File.Stat().ModTime()` 返回文件实际修改时间 → 非零值 → `setLastModified` 会设置 `Last-Modified` 头
- 仍然默认**不生成 ETag**
- 支持 `If-Modified-Since` 条件请求 → 命中返回 `304 Not Modified`

**/image/* 图标的实际响应头清单：**

| 响应头 | 是否存在 | 来源/原因 |
|-------|---------|----------|
| `Content-Type`（如 `image/png`） | ✅ 始终 | `http.FileServer` 按扩展名或内容嗅探 |
| `Content-Length` | ✅ 始终 | 磁盘文件元数据 |
| `Accept-Ranges: bytes` | ✅ 始终 | `http.FileServer` 默认支持 |
| `Last-Modified` | ✅ **有** | 磁盘文件真实 mtime，`setLastModified` 正常设置 |
| `ETag` | ❌ **无** | Go `http.FileServer` 默认不生成 |
| `Cache-Control` | ❌ 无 | 项目代码未显式设置 |
| `Expires` | ❌ 无 | 项目代码未显式设置 |
| `Content-Encoding: gzip` | ❌ 无 | 不在 gzip 中间件作用域内 + 图片格式被跳过 |

**浏览器缓存行为推论：**
- 有 `Last-Modified` 但无 `ETag` 和 `Cache-Control`
- 浏览器可以发起 `If-Modified-Since` 条件请求：
  - 若文件 mtime 未变 → 服务端返回 `304 Not Modified`（无响应体，节省带宽）
  - 若文件 mtime 变化 → 返回 `200 OK` + 完整内容
- 但由于项目设计是「每次上传生成全新随机文件名 + 删除旧文件」，实际场景中 mtime 很少被依赖

---

## 三、默认图标 vs 上传图标：替换/删除时的不同回源方式

### 3.1 数据库层的两种状态

[model/application.go#L45](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/model/application.go#L45)：

```go
type Application struct {
    // ...
    Image    string `gorm:"type:text" json:"image"`  // 空字符串 = 默认图，非空 = 上传图文件名
}
```

### 3.2 路径组合：withResolvedImage 函数

[api/application.go#L445-L453](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go#L445-L453)：

```go
func withResolvedImage(app *model.Application) *model.Application {
    if app.Image == "" {
        // 返回给前端：走 /static/ 路由（embed.FS，无缓存头）
        app.Image = "static/defaultapp.png"
    } else {
        // 返回给前端：走 /image/ 路由（磁盘文件，有 Last-Modified）
        app.Image = "image/" + app.Image
    }
    return app
}
```

### 3.3 场景一：从「默认图标」→「上传自定义图标」

#### 触发操作
用户在应用设置页上传图片 → `POST /application/:id/image`

#### 服务端处理 [api/application.go#L327-L379](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go#L327-L379)

```go
func (a *ApplicationAPI) UploadApplicationImage(ctx *gin.Context) {
    // 校验：文件类型、扩展名白名单
    // 生成全新 25 位随机文件名，如 "aB3dE_fGhIjKlMnOpQrStUvWx.png"
    name := generateNonExistingImageName(a.ImageDir, func() string {
        return generateImageName() + ext  // generateImageName 来自 auth/token.go#L52
    })
    ctx.SaveUploadedFile(file, a.ImageDir+name)  // 保存到 data/images/
    // 注意：旧 Image 为空（默认图），无需 os.Remove
    app.Image = name
    a.DB.UpdateApplication(app)
    ctx.JSON(200, withResolvedImage(app))  // 返回: "image/aB3dE...Wx.png"
}
```

#### 浏览器回源方式
| 维度 | 变化 |
|-----|------|
| **URL 变化** | `static/defaultapp.png` → `image/aB3dE_fGhIjKlMnOpQrStUvWx.png` |
| **路由变化** | `/static/*` → `/image/*`（完全不同的路由前缀） |
| **浏览器行为** | URL 完全不同，浏览器将其视为全新资源 → **无条件回源**（200 OK 下载） |
| **缓存头差异** | 旧 URL：无 Last-Modified / 无 ETag<br>新 URL：有 Last-Modified / 无 ETag |
| **前端触发** | `AppStore.uploadImage()` → `await this.refresh()` → MobX 响应式更新 `<img src>` |

**核心机制：URL 前缀从 `static/` 变为 `image/`，加上随机文件名，浏览器不可能命中任何已有缓存。**

---

### 3.4 场景二：从「上传图标 A」→「替换为上传图标 B」

#### 触发操作
用户重新上传一张不同的图片 → `POST /application/:id/image`

#### 服务端处理（关键在旧文件删除）

```go
func (a *ApplicationAPI) UploadApplicationImage(ctx *gin.Context) {
    // ... 生成新文件名 name = "def456.png" ...
    ctx.SaveUploadedFile(file, a.ImageDir+name)

    // 关键：删除旧文件！
    if app.Image != "" {                        // app.Image = "abc123.png"
        os.Remove(a.ImageDir + app.Image)       // 删除 data/images/abc123.png
    }

    app.Image = "def456.png"                    // 数据库更新
    a.DB.UpdateApplication(app)
    ctx.JSON(200, withResolvedImage(app))       // 返回: "image/def456.png"
}
```

测试验证 [api/application_test.go#L384-L411](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application_test.go#L384-L411)：

```go
// Test_UploadAppImage_WithImageFile_DeleteExstingImageAndGenerateNewName
existingImageName := "2lHMAel6BDHLL-HrwphcviX-l.png"
// ... 上传新图 ...
_, err = os.Stat(existingImageName)
assert.True(s.T(), os.IsNotExist(err))  // 旧文件已物理删除
```

#### 浏览器回源方式
| 维度 | 变化 |
|-----|------|
| **URL 变化** | `image/abc123.png` → `image/def456.png` |
| **路由变化** | 同属 `/image/*` 路由，但文件名完全不同 |
| **浏览器行为** | 文件名随机，URL 全新 → **无条件回源**（200 OK 下载新图） |
| **旧 URL 状态** | 磁盘文件已删除，请求 `GET /image/abc123.png` → **404 Not Found** |
| **缓存头差异** | 新旧 URL 都有 `Last-Modified`（各自的文件 mtime） |

**核心机制：25 位随机文件名保证 URL 唯一不重复，旧文件物理删除杜绝旧 URL 意外命中。**

---

### 3.5 场景三：从「上传图标」→「删除回退默认图」

#### 触发操作
用户点击「删除图标」按钮 → `DELETE /application/:id/image`

#### 服务端处理 [api/application.go#L420-L443](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go#L420-L443)

```go
func (a *ApplicationAPI) RemoveApplicationImage(ctx *gin.Context) {
    // ... 权限校验 ...
    if app.Image == "" {
        ctx.AbortWithError(400, fmt.Errorf("app with id %d does not have a customized image", id))
        return
    }

    image := app.Image           // 保存旧文件名用于删除
    app.Image = ""               // 数据库置空
    a.DB.UpdateApplication(app)
    os.Remove(a.ImageDir + image)  // 物理删除磁盘文件
    ctx.JSON(200, withResolvedImage(app))  // 返回: "static/defaultapp.png"
}
```

#### 浏览器回源方式
| 维度 | 变化 |
|-----|------|
| **URL 变化** | `image/abc123.png` → `static/defaultapp.png` |
| **路由变化** | `/image/*` → `/static/*`（反向切换） |
| **浏览器行为** | URL 从 `image/` 变回 `static/`，全新 URL → **回源获取默认图** |
| **旧 URL 状态** | 磁盘文件已删除 → 404 |
| **缓存头差异** | 旧 URL：有 Last-Modified<br>新 URL：无 Last-Modified（embed.FS 零值问题） |

**前端配套逻辑** [ui/src/application/Applications.tsx#L209](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/application/Applications.tsx#L209)：

```typescript
// 与后端 withResolvedImage 的硬编码值严格同步
const isDefaultImage = app.image === 'static/defaultapp.png';
// 控制删除按钮是否禁用：默认图时不可删除
```

---

### 3.6 场景四：删除整个应用

#### 服务端处理 [api/application.go#L186-L207](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go#L186-L207)

```go
func (a *ApplicationAPI) DeleteApplication(ctx *gin.Context) {
    // ... 权限校验 ...
    a.DB.DeleteApplicationByID(id)
    if app.Image != "" {
        os.Remove(a.ImageDir + app.Image)  // 级联删除其上传图标
    }
}
```

#### 浏览器回源方式
应用被删除后，API 不再返回该应用对象，前端不会再渲染其图标 URL，旧 URL 自然失效（文件已删除 → 404）。

---

### 3.7 不同回源方式对比表

| 操作场景 | 旧 URL | 新 URL | URL 是否变化 | 路由是否变化 | 旧文件状态 | 浏览器回源触发原因 |
|---------|--------|--------|-------------|-------------|-----------|-------------------|
| 默认 → 上传 | `static/defaultapp.png` | `image/随机.png` | ✅ 是 | ✅ `/static/` → `/image/` | 无旧文件 | **URL 彻底变化**，全新资源 |
| 上传 A → 上传 B | `image/abc123.png` | `image/def456.png` | ✅ 是（文件名不同） | ❌ 同属 `/image/` | `os.Remove` 删除 → 404 | **随机文件名唯一不重复** |
| 上传 → 删除回退 | `image/abc123.png` | `static/defaultapp.png` | ✅ 是 | ✅ `/image/` → `/static/` | `os.Remove` 删除 → 404 | **URL 前缀切换** |

**共同的设计哲学：** 绝不依赖 HTTP 协商缓存（ETag / Last-Modified）来处理图标替换，而是通过 **URL 唯一性** 强制浏览器回源。这是因为：
1. `/static/*` 走 `embed.FS`，连 `Last-Modified` 都没有，协商缓存不可用
2. 即使 `/image/*` 有 `Last-Modified`，文件名随机化策略更简单可靠，不依赖浏览器实现细节
3. 旧文件物理删除确保「删除操作」不可逆，不会出现因缓存导致旧图标幽灵重现的问题

---

## 四、前端图标渲染与数据刷新机制

### 4.1 URL 拼接：前端拿到相对路径后补 base URL

[ui/src/index.tsx#L20-L27](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/index.tsx#L20-L27)：

```typescript
// 根据浏览器当前 URL 计算服务端 base
const {port, hostname, protocol, pathname} = window.location;
const slashes = protocol.concat('//');
const path = pathname.endsWith('/') ? pathname : pathname.substring(0, pathname.lastIndexOf('/'));
const url = slashes.concat(port ? hostname.concat(':', port) : hostname) + path;
const urlWithSlash = url.endsWith('/') ? url : url.concat('/');
config.set('url', urlWithSlash);  // 例: "http://localhost:8080/"
```

[ui/src/application/Applications.tsx#L244-L249](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/application/Applications.tsx#L244-L249)：

```tsx
<img
    src={config.get('url') + app.image}
    // 例: "http://localhost:8080/" + "image/aB3dE...Wx.png"
    // 或: "http://localhost:8080/" + "static/defaultapp.png"
    alt="app logo"
    width="40"
    height="40"
/>
```

### 4.2 操作后强制刷新：AppStore.refresh()

上传、删除图标后都显式调用刷新：

[ui/src/application/AppStore.ts#L29-L48](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/application/AppStore.ts#L29-L48)：

```typescript
@action
public uploadImage = async (id: number, file: Blob): Promise<void> => {
    const formData = new FormData();
    formData.append('file', file);
    await axios.post(`${config.get('url')}application/${id}/image`, formData, {
        headers: {'Content-Type': 'multipart/form-data'},
    });
    await this.refresh();   // ← 重新拉取应用列表，获取新 image URL
    this.snack('Application image updated');
};

public async deleteImage(id: number): Promise<void> {
    await axios.delete(`${config.get('url')}application/${id}/image`);
    await this.refresh();   // ← 重新拉取
    this.snack('Application image deleted');
}
```

`refresh()` 触发消息 Store 的缓存清理 [ui/src/application/AppStore.ts#L110-L121](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/application/AppStore.ts#L110-L121)：

```typescript
public refresh = async () => {
    const {data} = await axios.get<Array<IApplication>>(`${config.get('url')}application`);
    this.applications.replace(data);
    // 消息列表中的图标是从 appStore 派生的，需要清理其缓存
    if (this.messageStore) {
        this.messageStore.clearCache();
    }
};
```

### 4.3 消息列表：动态派生映射

消息本身没有 `image` 字段，是在渲染时通过 `appid` 从 `appStore` 派生 [ui/src/message/MessagesStore.ts#L198-L208](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/message/MessagesStore.ts#L198-L208)：

```typescript
private getUnCached = (appId: number): Array<IMessage> => {
    // 构建 appId → image URL 的映射表
    const appToImage: Partial<Record<string, string>> = this.appStore
        .getItems()
        .reduce((all, app) => ({...all, [app.id]: app.image}), {});

    // 为每条消息注入关联应用的 image
    return this.stateOf(appId, false)
        .messages
        .filter((message) => !this.pendingDeletes.has(message.id))
        .map((message: IMessage): IMessage => ({...message, image: appToImage[message.appid]}));
};

// MobX createTransformer：appStore 变化时自动重新计算派生值
public get = createTransformer(this.getUnCached);
```

效果：应用图标 URL 变化后，所有历史消息关联的图标也同步更新（不需要逐条刷新消息）。

---

## 五、关键代码索引

| 功能 | 文件 | 行号 |
|-----|------|------|
| `serveFile()` 返回 index.html/manifest.json（无缓存头） | [ui/serve.go](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/serve.go) | L51-L61 |
| `/static/*` 走 `http.FileServer(http.FS(embed.FS))` | [ui/serve.go](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/serve.go) | L40-L44 |
| gzip 中间件组 | [ui/serve.go](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/serve.go) | L35 |
| `/image/*` 走 `gin.StaticFS` + `onlyImageFS` | [router/router.go](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/router/router.go) | L131, L279-L289 |
| 全局响应头中间件（/image 路由在其之前，不受影响） | [router/router.go](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/router/router.go) | L135-L140 |
| `withResolvedImage()` 路径转换核心 | [api/application.go](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go) | L445-L453 |
| 上传图标 + 删除旧图 | [api/application.go](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go) | L327-L379 |
| 删除图标 + 回退默认图 | [api/application.go](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go) | L420-L443 |
| 25 位随机文件名生成 | [auth/token.go](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/auth/token.go) | L52-L68 |
| 前端 isDefaultImage 判断（与后端硬编码同步） | [ui/src/application/Applications.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/application/Applications.tsx) | L209 |
| 前端 uploadImage/deleteImage → refresh() | [ui/src/application/AppStore.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/application/AppStore.ts) | L29-L48 |
| 消息图标派生映射 | [ui/src/message/MessagesStore.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/message/MessagesStore.ts) | L198-L208 |
