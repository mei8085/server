# 应用图标链路深度分析报告

## 目录

1. [浏览器标签图标与启动图标的暴露位置](#1-浏览器标签图标与启动图标的暴露位置)
2. [默认应用图标与用户上传图标的路径组合](#2-默认应用图标与用户上传图标的路径组合)
3. [缓存相关响应头实现](#3-缓存相关响应头实现)
4. [图片替换/删除后浏览器强制回源机制](#4-图片替换删除后浏览器强制回源机制)

---

## 1. 浏览器标签图标与启动图标的暴露位置

### 1.1 主界面 (index.html)

在 [ui/index.html](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/index.html) 中，通过 `<link>` 和 `<meta>` 标签暴露了多种尺寸的图标：

```html
<!-- 浏览器标签页 Favicon（多尺寸） -->
<link rel="icon" type="image/png" href="static/favicon-196x196.png" sizes="196x196" />
<link rel="icon" type="image/png" href="static/favicon-96x96.png" sizes="96x96" />
<link rel="icon" type="image/png" href="static/favicon-32x32.png" sizes="32x32" />
<link rel="icon" type="image/png" href="static/favicon-16x16.png" sizes="16x16" />
<link rel="icon" type="image/png" href="static/favicon-128.png" sizes="128x128" />
<link rel="icon" href="static/favicon.ico">

<!-- iOS 主屏启动图标 (apple-touch-icon) -->
<link rel="apple-touch-icon-precomposed" sizes="57x57" href="static/apple-touch-icon-57x57.png" />
<link rel="apple-touch-icon-precomposed" sizes="114x114" href="static/apple-touch-icon-114x114.png" />
<link rel="apple-touch-icon-precomposed" sizes="72x72" href="static/apple-touch-icon-72x72.png" />
<link rel="apple-touch-icon-precomposed" sizes="144x144" href="static/apple-touch-icon-144x144.png" />
<link rel="apple-touch-icon-precomposed" sizes="60x60" href="static/apple-touch-icon-60x60.png" />
<link rel="apple-touch-icon-precomposed" sizes="120x120" href="static/apple-touch-icon-120x120.png" />
<link rel="apple-touch-icon-precomposed" sizes="76x76" href="static/apple-touch-icon-76x76.png" />
<link rel="apple-touch-icon-precomposed" sizes="152x152" href="static/apple-touch-icon-152x152.png" />

<!-- Windows Metro 磁贴图标 (mstile) -->
<meta name="msapplication-TileColor" content="#FFFFFF" />
<meta name="msapplication-TileImage" content="static/mstile-144x144.png" />
<meta name="msapplication-square70x70logo" content="static/mstile-70x70.png" />
<meta name="msapplication-square150x150logo" content="static/mstile-150x150.png" />
<meta name="msapplication-wide310x150logo" content="static/mstile-310x150.png" />
<meta name="msapplication-square310x310logo" content="static/mstile-310x310.png" />

<!-- PWA 应用清单 -->
<link rel="manifest" href="manifest.json">
<meta name="theme-color" content="#3f51b5">
```

**服务端路由注册**（[ui/serve.go#L25-L44](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/serve.go#L25-L44)）：

```go
func Register(r *gin.Engine, version model.VersionInfo, register, oidcEnabled bool) {
    // 特殊文件：index.html（注入 %CONFIG%）和 manifest.json
    ui := r.Group("/", gzip.Gzip(gzip.DefaultCompression))
    ui.GET("/", serveFile("index.html", "text/html", replaceConfig))
    ui.GET("/index.html", serveFile("index.html", "text/html", replaceConfig))
    ui.GET("/manifest.json", serveFile("manifest.json", "application/json", noop))

    // /static/*any 路由：所有 png/ico 图标通过标准库 http.FileServer 提供
    subBox, _ := fs.Sub(box, "build")  // box 是 //go:embed build/* 的 embed.FS
    ui.GET("/static/*any", gin.WrapH(http.FileServer(http.FS(subBox))))
}
```

### 1.2 Swagger UI 界面

在 [docs/ui.go#L13-L14](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/docs/ui.go#L13-L14) 中也引用了 favicon：

```go
var ui = `
...
    <link rel="icon" type="image/png" href="./favicon-32x32.png" sizes="32x32" />
    <link rel="icon" type="image/png" href="./favicon-16x16.png" sizes="16x16" />
...
`
```

Swagger UI 的路由在 [router/router.go#L130](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/router/router.go#L130) 注册：

```go
g.GET("/swagger", docs.Serve)
```

### 1.3 PWA 应用清单

在 [ui/public/manifest.json](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/public/manifest.json) 中声明应用基本信息（不含图标字段，图标由 HTML 标签提供）：

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

### 1.4 图标资源物理位置

所有内置图标资源位于 [ui/public/static/](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/public/static) 目录，构建时被 Vite 原样复制到 `ui/build/static/`，再由 Go `//go:embed build/*` 嵌入二进制：

```
ui/public/static/
├── apple-touch-icon-57x57.png ~ apple-touch-icon-152x152.png  (8种尺寸)
├── favicon-16x16.png ~ favicon-196x196.png                     (5种尺寸)
├── favicon.ico
├── mstile-70x70.png ~ mstile-310x310.png                        (5种尺寸)
├── defaultapp.png           ← 应用默认图标
└── notification.ogg
```

---

## 2. 默认应用图标与用户上传图标的路径组合

### 2.1 数据库层：原始存储格式

在 [model/application.go#L45](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/model/application.go#L45) 中，`Application.Image` 字段存储的是**纯文件名**（不含路径前缀）：

```go
type Application struct {
    // ...
    // example: image/image.jpeg (但实际数据库中存的是纯文件名如 "abcd.jpg")
    Image    string `gorm:"type:text" json:"image"`
    // ...
}
```

数据库中 Image 字段的两种状态：

| 状态 | 数据库存储值 | 含义 |
|-----|------------|------|
| 使用默认图 | `""` (空字符串) | 用户未上传自定义图标 |
| 使用上传图 | `"aB3dE_fGhIjKlMnOpQrStUvWx.png"` | 25位随机字符 + 扩展名 |

### 2.2 服务端 API 层：路径前缀注入

核心函数 [api/application.go#L445-L453](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go#L445-L453)：

```go
func withResolvedImage(app *model.Application) *model.Application {
    if app.Image == "" {
        // 默认图：走 /static/ 路由（embed.FS 内置资源）
        // 注释明确要求与前端同步：
        // This must stay in sync with the isDefaultImage check in ui/src/application/Applications.tsx.
        app.Image = "static/defaultapp.png"
    } else {
        // 上传图：走 /image/ 路由（本地文件系统 data/images/）
        app.Image = "image/" + app.Image
    }
    return app
}
```

**所有返回 Application 的 API 均调用此函数进行路径转换：**

| API 端点 | 调用位置 | 代码行 |
|---------|---------|-------|
| `POST /application` (创建) | `CreateApplication` | [application.go#L109](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go#L109) |
| `GET /application` (查询列表) | `GetApplications` | [application.go#L143-L145](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go#L143-L145) |
| `PUT /application/:id` (更新) | `UpdateApplication` | [application.go#L272](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go#L272) |
| `POST /application/:id/image` (上传) | `UploadApplicationImage` | [application.go#L374](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go#L374) |
| `DELETE /application/:id/image` (删除) | `RemoveApplicationImage` | [application.go#L438](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go#L438) |

### 2.3 测试验证

在 [api/application_test.go#L253-L285](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application_test.go#L253-L285) 中有明确的测试用例验证路径转换逻辑：

```go
// Test_GetApplications：空 Image → "static/defaultapp.png"
first.Image = "static/defaultapp.png"
second.Image = "static/defaultapp.png"
test.BodyEquals(s.T(), []*model.Application{first, second}, s.recorder)

// Test_GetApplications_WithImage：非空 Image → "image/" + 文件名
first.Image = "abcd.jpg"  // 数据库中原始值
s.db.UpdateApplication(first)
// API 返回结果：
first.Image = "image/abcd.jpg"      // 非空值加上 image/ 前缀
second.Image = "static/defaultapp.png"  // 空值使用默认图
```

### 2.4 前端渲染层：最终 URL 拼接

#### 2.4.1 基础 URL 注入

在 [ui/src/index.tsx#L20-L27](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/index.tsx#L20-L27) 和 [ui/src/config.ts#L16-L30](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/config.ts#L16-L30) 中，前端根据 `window.location` 计算服务端 base URL：

```typescript
// index.tsx 中计算：
const {port, hostname, protocol, pathname} = window.location;
const slashes = protocol.concat('//');
const path = pathname.endsWith('/') ? pathname : pathname.substring(0, pathname.lastIndexOf('/'));
const url = slashes.concat(port ? hostname.concat(':', port) : hostname) + path;
const urlWithSlash = url.endsWith('/') ? url : url.concat('/');
config.set('url', urlWithSlash);  // 例如 "http://localhost:8080/"
```

同时 `index.html` 中的 `<script>window.config = %CONFIG%;</script>`（由 [serve.go#L31-L33](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/serve.go#L31-L33) 替换）注入额外配置，但 `url` 以 index.tsx 的计算为准。

#### 2.4.2 应用列表页图标渲染

在 [ui/src/application/Applications.tsx#L209](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/application/Applications.tsx#L209) 和 [L244-L249](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/application/Applications.tsx#L244-L249)：

```typescript
// 与后端严格同步的硬编码判断！
const isDefaultImage = app.image === 'static/defaultapp.png';

// 最终渲染：
<img
    src={config.get('url') + app.image}
    // 当 app.image = "static/defaultapp.png" 时
    // → src = "http://host:port/static/defaultapp.png"
    // 当 app.image = "image/aB3dE_fGhIjKlMnOpQrStUvWx.png" 时
    // → src = "http://host:port/image/aB3dE_fGhIjKlMnOpQrStUvWx.png"
    alt="app logo"
    width="40"
    height="40"
/>
```

#### 2.4.3 消息列表页图标渲染（动态映射）

消息本身不存储图片字段，而是在运行时通过 `appid` 从 `appStore` 中动态映射。见 [ui/src/message/MessagesStore.ts#L198-L208](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/message/MessagesStore.ts#L198-L208)：

```typescript
private getUnCached = (appId: number): Array<IMessage> => {
    // 构建 appId → image 路径的映射表（来自已加载的应用列表）
    const appToImage: Partial<Record<string, string>> = this.appStore
        .getItems()
        .reduce((all, app) => ({...all, [app.id]: app.image}), {});

    // 为每条消息动态注入关联应用的 image 字段
    return this.stateOf(appId, false)
        .messages.filter((message) => !this.pendingDeletes.has(message.id))
        .map((message: IMessage): IMessage => ({...message, image: appToImage[message.appid]}));
};

// 使用 MobX createTransformer 做派生缓存
public get = createTransformer(this.getUnCached);
```

然后在 [ui/src/message/Message.tsx#L232-L240](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/message/Message.tsx#L232-L240) 中渲染：

```tsx
<div className={classes.imageWrapper}>
    {image !== null ? (
        <img
            src={config.get('url') + image}
            alt={`${appName} logo`}
            width="50"
            height="50"
            className={classes.image}
        />
    ) : null}
</div>
```

#### 2.4.4 浏览器桌面通知图标

在 [ui/src/snack/browserNotification.ts#L18-L27](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/snack/browserNotification.ts#L18-L27)：

```typescript
export function notifyNewMessage(msg: IMessage) {
    const notify = new Notify(msg.title, {
        body: removeMarkdown(msg.message),
        icon: msg.image,  // 使用消息上已注入的 image 路径（相对路径）
        silent: true,
        notifyClick: closeAndFocus,
        notifyShow: closeAfterTimeout,
    });
    notify.show();
}
```

### 2.5 路径组合全链路总结

```
数据库存储
    │
    ├─ Image = "" (空)
    │     │
    │     ▼
    │   withResolvedImage() → "static/defaultapp.png"
    │     │
    │     ▼
    │   前端 config.get('url') + image
    │     │
    │     ▼
    │   "http://host:port/static/defaultapp.png"
    │     │
    │     ▼
    │   路由: ui.GET("/static/*any", gin.WrapH(http.FileServer(embed.FS)))
    │   来源: Go embed.FS 内存文件系统 (ui/build/static/defaultapp.png)
    │
    └─ Image = "aB3dE...Wx.png" (25位随机文件名)
          │
          ▼
        withResolvedImage() → "image/aB3dE...Wx.png"
          │
          ▼
        前端 config.get('url') + image
          │
          ▼
        "http://host:port/image/aB3dE...Wx.png"
          │
          ▼
        路由: g.StaticFS("/image", onlyImageFS{gin.Dir("data/images/")})
        来源: 本地文件系统 (data/images/aB3dE...Wx.png)
```

---

## 3. 缓存相关响应头实现

### 3.1 静态资源服务的两条路径

#### 路径 A：`/static/*` — 内置资源（通过 `http.FileServer` + `embed.FS`）

注册代码见 [ui/serve.go#L44](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/serve.go#L44)：

```go
ui.GET("/static/*any", gin.WrapH(http.FileServer(http.FS(subBox))))
```

Go 标准库 `net/http` 的 `fileHandler.ServeHTTP` 自动处理缓存响应头。根据 Go 源码（`net/http/fs.go`），`http.FileServer` 会：

| 响应头 | 生成方式 | embed.FS 的行为 |
|-------|---------|----------------|
| **Content-Type** | 根据文件扩展名或内容嗅探 | 正常，MIME 类型识别 |
| **Content-Length** | 文件大小 | 正常，来自 embed.FS 元数据 |
| **Last-Modified** | 文件修改时间 | embed.FS 的文件 ModTime 为 Go 工具链链接时的时间戳（零值或构建时间） |
| **ETag** | 弱校验 ETag（基于内容） | Go 1.19+ 中 `http.ServeContent` 会对可 `io.Seeker` 的内容生成弱 ETag |
| **Accept-Ranges** | `bytes` | 正常支持 |

**条件请求处理：**

- 请求带 `If-None-Match` 头 → 与 ETag 比较 → 命中返回 `304 Not Modified`（空响应体）
- 请求带 `If-Modified-Since` 头 → 与 Last-Modified 比较 → 命中返回 `304 Not Modified`
- 两者同时存在 → ETag 优先（HTTP 规范）

#### 路径 B：`/image/*` — 用户上传资源（通过 `gin.StaticFS` + `onlyImageFS`）

注册代码见 [router/router.go#L131](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/router/router.go#L131)：

```go
g.StaticFS("/image", &onlyImageFS{inner: gin.Dir(conf.UploadedImagesDir, false)})
```

`gin.StaticFS` 内部也调用了 `http.FileServer`，因此缓存行为与路径 A 完全一致，区别在于：

- **文件来源**：本地文件系统 `data/images/`（真实磁盘文件）
- **Last-Modified**：文件实际的修改时间（mtime）
- **文件类型校验**：`onlyImageFS.Open()` 在打开文件前校验扩展名必须是 `.gif/.png/.jpg/.jpeg`

### 3.2 GZIP 中间件对缓存头的影响

`/static/*` 路由被包裹在 `gzip.Gzip(gzip.DefaultCompression)` 中间件下（[ui/serve.go#L35](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/serve.go#L35)）。

`gin-contrib/gzip` 中间件的行为：
- 当请求头 `Accept-Encoding: gzip` 存在时，压缩响应体
- 设置响应头 `Content-Encoding: gzip`
- 设置响应头 `Vary: Accept-Encoding`（确保不同编码的缓存隔离）
- **不修改** ETag 和 Last-Modified（由内部 `http.FileServer` 先生成）

### 3.3 自定义响应头（全局配置）

在 [router/router.go#L135-L140](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/router/router.go#L135-L140) 中有全局响应头中间件：

```go
g.Use(func(ctx *gin.Context) {
    ctx.Header("Content-Type", "application/json")
    for header, value := range conf.Server.ResponseHeaders {
        ctx.Header(header, value)
    }
})
```

**注意：** 该中间件注册在 UI 路由（第 117 行 `ui.Register`）和 `/image` 路由（第 131 行 `g.StaticFS`）**之后**，因此对静态资源请求**不生效**。静态资源请求不会经过此中间件。

### 3.4 实际返回的响应头汇总

| 响应头 | `/static/*` (内置) | `/image/*` (上传) | 来源 |
|-------|--------------------|-------------------|------|
| `Content-Type` | ✅ `image/png` 等 | ✅ `image/png` 等 | `http.DetectContentType` / 扩展名 |
| `Content-Length` | ✅ 文件大小 | ✅ 文件大小 | 文件系统元数据 |
| `Last-Modified` | ✅ 构建时间戳 | ✅ 文件 mtime | 文件系统 ModTime |
| `ETag` | ✅ 弱校验 (W/"...") | ✅ 弱校验 (W/"...") | `http.ServeContent` |
| `Accept-Ranges` | ✅ `bytes` | ✅ `bytes` | `http.FileServer` |
| `Content-Encoding` | ✅ `gzip` (条件) | ❌ 无 | gin-contrib/gzip 中间件（仅 `/static/*`） |
| `Vary` | ✅ `Accept-Encoding` (条件) | ❌ 无 | gin-contrib/gzip 中间件 |
| `Cache-Control` | ❌ 无 | ❌ 无 | 项目未显式设置 |
| `Expires` | ❌ 无 | ❌ 无 | 项目未显式设置 |

### 3.5 浏览器缓存行为推导

由于缺少显式 `Cache-Control` 和 `Expires` 头，浏览器使用**启发式缓存**（RFC 7234）：

> 如果响应中没有显式的缓存控制头，且资源有 Last-Modified，浏览器通常假设缓存有效期为 `(当前时间 - Last-Modified) × 10%`。

实际表现：
- **首次访问**：`200 OK` + 完整响应体 + ETag/Last-Modified 响应头 → 存入浏览器缓存
- **二次访问**：浏览器自动发送 `If-None-Match: <etag>` 和 `If-Modified-Since: <date>` → 服务端校验 → 内容未变返回 `304 Not Modified`（无响应体），浏览器从缓存加载
- **内容变更后**：ETag 或 Last-Modified 变化 → 服务端返回 `200 OK` + 新内容

---

## 4. 图片替换/删除后浏览器强制回源机制

### 4.1 核心设计：文件名随机化（Cache Busting）

上传图片时生成全新的随机文件名，从根本上规避浏览器缓存问题。

#### 4.1.1 随机文件名生成

在 [auth/token.go#L52-L55](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/auth/token.go#L52-L55)：

```go
// GenerateImageName generates an image name.
func GenerateImageName() string {
    return generateRandomString(25)  // 25 位随机字符串
}

func generateRandomString(length int) string {
    // 字符集: a-z, A-Z, 0-9, ., -, _  (共 65 个字符)
    tokenCharacters := []byte("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789.-_")
    res := make([]byte, length)
    for i := range res {
        index := randIntn(len(tokenCharacters))  // crypto/rand 安全随机
        res[i] = tokenCharacters[index]
    }
    return string(res)
}
```

**碰撞概率：** 65^25 ≈ 6.5 × 10^45，实际可视为唯一。

#### 4.1.2 上传流程中的旧图清理

在 [api/application.go#L327-L379](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go#L327-L379)：

```go
func (a *ApplicationAPI) UploadApplicationImage(ctx *gin.Context) {
    // 1. 校验文件类型
    if !filetype.IsImage(head) { return 400 }
    if !ValidApplicationImageExt(ext) { return 400 }

    // 2. 生成全新的随机文件名（保证不冲突）
    name := generateNonExistingImageName(a.ImageDir, func() string {
        return generateImageName() + ext  // e.g. "aB3dE_fGhIjKlMnOpQrStUvWx.png"
    })

    // 3. 保存新文件到磁盘
    ctx.SaveUploadedFile(file, a.ImageDir+name)  // data/images/aB3dE...Wx.png

    // 4. 删除旧文件（物理删除！）
    if app.Image != "" {
        os.Remove(a.ImageDir + app.Image)  // 旧文件彻底消失
    }

    // 5. 更新数据库为新文件名
    app.Image = name
    a.DB.UpdateApplication(app)

    // 6. 返回新路径（通过 withResolvedImage 注入 image/ 前缀）
    ctx.JSON(200, withResolvedImage(app))
}
```

**测试验证**（[api/application_test.go#L384-L411](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application_test.go#L384-L411)）：

```go
// Test_UploadAppImage_WithImageFile_DeleteExstingImageAndGenerateNewName
existingImageName := "2lHMAel6BDHLL-HrwphcviX-l.png"
// ... 上传新图片 ...
// 断言：旧文件已被 os.Remove 删除
_, err = os.Stat(existingImageName)
assert.True(s.T(), os.IsNotExist(err))  // 旧文件不存在
// 断言：新文件已生成
_, err = os.Stat(secondGeneratedImageName)
assert.Nil(s.T(), err)  // 新文件存在
```

### 4.2 图片删除流程

在 [api/application.go#L420-L443](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go#L420-L443)：

```go
func (a *ApplicationAPI) RemoveApplicationImage(ctx *gin.Context) {
    // ... 权限校验 ...
    if app.Image == "" {
        ctx.AbortWithError(400, fmt.Errorf("app with id %d does not have a customized image", id))
        return
    }

    // 1. 保存旧文件名用于删除
    image := app.Image

    // 2. 数据库置空（触发 withResolvedImage 返回默认图）
    app.Image = ""
    a.DB.UpdateApplication(app)

    // 3. 物理删除磁盘文件
    os.Remove(a.ImageDir + image)

    // 4. 返回：Image = "static/defaultapp.png"
    ctx.JSON(200, withResolvedImage(app))
}
```

### 4.3 应用删除时的级联清理

在 [api/application.go#L186-L207](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go#L186-L207)：

```go
func (a *ApplicationAPI) DeleteApplication(ctx *gin.Context) {
    // ... 权限校验 ...
    a.DB.DeleteApplicationByID(id)
    if app.Image != "" {
        os.Remove(a.ImageDir + app.Image)  // 删除应用 → 同时删除其图片文件
    }
}
```

### 4.4 浏览器强制回源的四种触发路径

#### 路径 1：上传新图片（替换）

```
时间点 T0：应用使用图片 A
  URL: /image/abc123.png
  浏览器缓存: 已缓存 abc123.png
  数据库: app.Image = "abc123.png"

时间点 T1：用户上传新图片 B
  ① 服务端生成全新随机文件名: "def456.png"
  ② os.Remove("data/images/abc123.png")  ← 旧文件物理删除
  ③ 保存新文件: "data/images/def456.png"
  ④ 数据库更新: app.Image = "def456.png"
  ⑤ API 返回: app.image = "image/def456.png"

时间点 T2：前端刷新应用列表（AppStore.refresh()）
  ① axios GET /application → 返回新 image 字段
  ② MobX 响应式更新 → <img src={url + "image/def456.png"} />
  ③ 浏览器发现 def456.png 是全新 URL → 本地缓存无此资源 → 强制回源请求
  ④ GET /image/def456.png → 200 OK + 新图片内容

结果：用户立即看到新图标，旧 URL /image/abc123.png 返回 404（文件已删除）
```

#### 路径 2：删除自定义图片（回退默认图）

```
时间点 T0：应用使用自定义图片
  URL: /image/abc123.png

时间点 T1：用户点击删除图标
  ① os.Remove("data/images/abc123.png")
  ② 数据库: app.Image = ""
  ③ API 返回: app.image = "static/defaultapp.png"

时间点 T2：前端刷新
  <img src={url + "static/defaultapp.png"} />
  → URL 从 /image/... 变为 /static/... → 全新 URL → 强制回源（或命中已有默认图缓存）
```

#### 路径 3：前端 Store refresh() 机制

上传和删除操作后，前端都会显式调用 `refresh()` 重新拉取数据：

- 上传：[ui/src/application/AppStore.ts#L29-L37](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/application/AppStore.ts#L29-L37)
  ```typescript
  @action
  public uploadImage = async (id: number, file: Blob): Promise<void> => {
      await axios.post(`${config.get('url')}application/${id}/image`, formData, ...);
      await this.refresh();  // ← 重新拉取应用列表，获取新 image URL
      this.snack('Application image updated');
  };
  ```

- 删除：[ui/src/application/AppStore.ts#L39-L48](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/application/AppStore.ts#L39-L48)
  ```typescript
  public async deleteImage(id: number): Promise<void> {
      await axios.delete(`${config.get('url')}application/${id}/image`);
      await this.refresh();  // ← 重新拉取
      this.snack('Application image deleted');
  }
  ```

`refresh()` 触发 `MessagesStore.clearCache()` → `createTransformer` 重新计算 → 消息列表中的图标也同步更新。

#### 路径 4：旧 URL 的 404 兜底

即使浏览器缓存了旧 URL（如 `/image/abc123.png`），由于：
1. 服务器端文件已被 `os.Remove` 删除
2. 数据库中已不再引用该文件名

访问旧 URL 会得到：
```
GET /image/abc123.png → 404 Not Found
```

`gin.StaticFS` 底层的 `http.FileServer` 对不存在的文件返回 404，配合前端的刷新机制，用户不会看到失效图标。

### 4.5 机制总结

| 机制 | 实现方式 | 解决的问题 |
|-----|---------|-----------|
| **URL 彻底变化** | 25位随机字符文件名，每次上传必不重复 | 浏览器 URL 级缓存隔离 |
| **旧文件物理删除** | `os.Remove()` 删除磁盘文件 | 杜绝旧 URL 命中服务器文件 |
| **数据库引用更新** | `app.Image` 字段同步更新 | API 返回的路径始终有效 |
| **前端强制刷新** | `AppStore.refresh()` 重新拉取 | UI 立即显示新 URL |
| **消息动态映射** | `MessagesStore.getUnCached()` 派生计算 | 消息列表中的图标同步变化 |
| **默认图兜底** | `withResolvedImage()` 空值回退 | 删除自定义图标后自动显示默认图 |

---

## 附录：关键代码索引

| 功能模块 | 文件位置 | 关键行号 |
|---------|---------|---------|
| 静态资源路由注册 | [ui/serve.go](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/serve.go) | L25-L45 |
| 上传图片路由注册 | [router/router.go](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/router/router.go) | L131, L200-L201 |
| 路径转换核心函数 | [api/application.go](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go) | L445-L453 |
| 上传图片实现 | [api/application.go](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go) | L327-L379 |
| 删除图片实现 | [api/application.go](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application.go) | L420-L443 |
| 随机文件名生成 | [auth/token.go](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/auth/token.go) | L52-L68 |
| 前端应用图标渲染 | [ui/src/application/Applications.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/application/Applications.tsx) | L209, L244-L249 |
| 前端消息图标映射 | [ui/src/message/MessagesStore.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/message/MessagesStore.ts) | L198-L208 |
| 前端消息图标渲染 | [ui/src/message/Message.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/message/Message.tsx) | L232-L240 |
| 前端桌面通知图标 | [ui/src/snack/browserNotification.ts](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/snack/browserNotification.ts) | L18-L27 |
| 前端基础 URL 计算 | [ui/src/index.tsx](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/src/index.tsx) | L20-L27, L54 |
| 浏览器标签图标 HTML | [ui/index.html](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/ui/index.html) | L10-L30 |
| 上传图片测试 | [api/application_test.go](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application_test.go) | L358-L433 |
| 路径转换测试 | [api/application_test.go](file:///d:/fz/0601-1/solo-dogfeeding/code/10-server/api/application_test.go) | L253-L285 |
