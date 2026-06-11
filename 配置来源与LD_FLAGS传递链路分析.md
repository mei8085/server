# 两个细节问题的深入分析报告

## 问题一：登录页/提权页的注册入口与 OIDC 入口数据来源

### 结论先行

**登录页和提权页完全依赖启动时注入的 `window.config` 配置对象，不会调用 `/gotifyinfo` API。**

### 证据链

#### 1. 登录页（Login.tsx）

登录页通过 `config.get()` 读取两个开关：

- **注册按钮显示逻辑**：`ui/src/user/Login.tsx` 第 26 行
  ```tsx
  if (config.get('register'))
      return (<Button id="register" ...>Register</Button>);
  ```

- **OIDC 登录按钮显示逻辑**：`ui/src/user/Login.tsx` 第 84 行
  ```tsx
  {config.get('oidc') && (
      <>
          <Divider ...>or</Divider>
          <Button ... href={config.get('url') + 'auth/oidc/login?...'}>
              Login with OIDC
          </Button>
      </>
  )}
  ```

两处均直接从 `config` 模块读取，没有任何 `axios.get('/gotifyinfo')` 或 `fetch()` 调用。

#### 2. 提权页（ElevationForm.tsx）

提权页同样通过 `config.get('oidc')` 判断是否显示 OIDC 提权按钮：

- `ui/src/common/ElevationForm.tsx` 第 18 行、第 82 行
  ```tsx
  const oidcEnabled = config.get('oidc');
  // ...
  {oidcEnabled && (
      <>
          <Divider ...>or</Divider>
          <Button ... onClick={() => elevateStore.oidcElevate(ElevateDuration)}>
              Elevate via OIDC
          </Button>
      </>
  )}
  ```

#### 3. 全局搜索确认

在 `ui/src` 目录下搜索 `gotifyinfo`、`/gotifyinfo`、`fetch(`、`axios.*info` 等关键词，**无任何匹配结果**。前端代码库中不存在调用 `/gotifyinfo` API 的逻辑。

#### 4. config 模块的数据来源

`ui/src/config.ts` 第 16-22 行：
```typescript
const config: IConfig = {
    url: 'unset',
    register: false,
    version: {commit: 'unknown', buildDate: 'unknown', version: 'unknown'},
    oidc: false,
    ...window.config,  // 展开覆盖默认值
};
```

`window.config` 是在 `ui/index.html` 第 38 行由服务端模板注入的：
```html
<script>window.config = %CONFIG%;</script>
```

而 `%CONFIG%` 占位符由 Go 后端 `ui/serve.go` 的 `Register` 函数在每次请求首页时替换：
```go
uiConfigBytes, _ := json.Marshal(uiConfig{
    Version:  version,     // model.VersionInfo
    Register: register,    // 来自 conf.Registration
    OIDC:     oidcEnabled, // 来自 conf.OIDC.Enabled
})
replaceConfig := func(content string) string {
    return strings.Replace(content, "%CONFIG%", string(uiConfigBytes), 1)
}
```

### 小结

| 页面 | 功能开关 | 数据来源 | 是否调用 /gotifyinfo |
|---|---|---|---|
| 登录页 Login.tsx | `register`（注册入口） | `window.config.register` → 服务端注入 | ❌ 否 |
| 登录页 Login.tsx | `oidc`（OIDC 登录入口） | `window.config.oidc` → 服务端注入 | ❌ 否 |
| 提权页 ElevationForm.tsx | `oidc`（OIDC 提权入口） | `window.config.oidc` → 服务端注入 | ❌ 否 |

> 注：`/gotifyinfo` API 端点虽然存在（`router/router.go` 第 187-189 行），返回同样的 `version`、`register`、`oidc` 字段，但前端代码从未调用它。它的实际消费者是外部脚本和第三方客户端，前端只依赖 `window.config` 的 SSR 式注入。

---

## 问题二：三种路径下 LD_FLAGS 的完整传递链路

### 核心载体总览

LD_FLAGS 在传递过程中会经过以下几种载体形式：
1. **CI 环境变量文件**（GitHub Actions `$GITHUB_ENV`）
2. **Shell 进程环境变量**（父进程 → 子进程继承）
3. **Makefile 变量**（通过 `$$LD_FLAGS` 展开 Shell 变量）
4. **Docker 容器环境变量**（`docker run -e LD_FLAGS=...`）
5. **Docker 构建参数**（`docker buildx build --build-arg LD_FLAGS=...`）
6. **Dockerfile ARG 指令**（`ARG LD_FLAGS=""`）
7. **go build 命令行参数**（`go build -ldflags "..."`）

---

### 路径 A：Tag 发布构建（CI → 二进制文件 + Docker 镜像）

Tag 发布是最复杂的路径，同时产出两类产物：**跨平台二进制文件**（`make build`）和 **多架构 Docker 镜像**（`make build-docker`），LD_FLAGS 走两条子路径。

#### 子路径 A-1：CI → 跨平台二进制文件（make build）

```
CI 环境变量文件 ($GITHUB_ENV)
       ↓ 第 1 步：GitHub Actions 读取 $GITHUB_ENV 设置当前 Job 环境
CI Runner Shell 环境变量 (LD_FLAGS=...)
       ↓ 第 2 步：执行 make build，子进程继承环境
Makefile Shell 变量 ($$LD_FLAGS 展开)
       ↓ 第 3 步：DOCKER_RUN 定义中 -e LD_FLAGS="$$LD_FLAGS"
       ↓        示例：docker run --rm -e LD_FLAGS="-w -s -X main.Version=2.5.0 ..." gotify/build:...
Docker 容器环境变量（进入 gotify/build 跨平台编译容器）
       ↓ 第 4 步：容器内执行 make _build_within_docker
容器内 Shell 环境变量
       ↓ 第 5 步：DOCKER_GO_BUILD 定义中 -ldflags "$$LD_FLAGS"
       ↓        示例：go build -mod=readonly -a -installsuffix cgo -ldflags "-w -s -X main.Version=2.5.0 ..."
go build 链接器参数（-ldflags）
       ↓ 第 6 步：Go 链接器在编译期重写 main 包变量
Go 二进制 .data 段
```

涉及文件：
- `.github/workflows/build.yml` 第 32-36 行：设置 LD_FLAGS 到 $GITHUB_ENV，并执行 `make build`
- `Makefile` 第 8 行：`DOCKER_RUN=docker run --rm -e LD_FLAGS="$$LD_FLAGS" ...`
- `Makefile` 第 9 行：`DOCKER_GO_BUILD=go build ... -ldflags "$$LD_FLAGS"`
- `Makefile` 第 133-154 行：`build-linux-amd64` 等各平台目标调用 `${DOCKER_RUN}`
- `Makefile` 第 126-128 行：`_build_within_docker` 目标调用 `${DOCKER_GO_BUILD}`

#### 子路径 A-2：CI → Docker 镜像（make build-docker）

```
CI 环境变量文件 ($GITHUB_ENV)
       ↓ 第 1 步
CI Runner Shell 环境变量
       ↓ 第 2 步：执行 make build-docker → build-docker-multiarch
Makefile Shell 变量 ($$LD_FLAGS 展开)
       ↓ 第 3 步：docker buildx build --build-arg LD_FLAGS="$$LD_FLAGS"
       ↓        传递给 Dockerfile 构建参数
Dockerfile ARG LD_FLAGS=""（docker/Dockerfile 第 35 行）
       ↓ 第 4 步：Dockerfile RUN 指令中 LD_FLAGS=${LD_FLAGS} make _build_within_docker
       ↓        显式赋值给 make 命令的执行环境
Builder 容器内 Shell 环境变量
       ↓ 第 5 步：容器内 make _build_within_docker → ${DOCKER_GO_BUILD}
       ↓        go build ... -ldflags "$$LD_FLAGS"
go build 链接器参数
       ↓ 第 6 步
Go 二进制 .data 段
```

涉及文件：
- `.github/workflows/build.yml` 第 57-59 行：执行 `make DOCKER_BUILD_PUSH=true build-docker`
- `Makefile` 第 66-108 行：`build-docker-multiarch` 目标，第 106 行传入 `--build-arg LD_FLAGS="$$LD_FLAGS"`
- `docker/Dockerfile` 第 35 行：`ARG LD_FLAGS=""` 接收构建参数
- `docker/Dockerfile` 第 53 行：`LD_FLAGS=${LD_FLAGS} make OUTPUT=/target/app/gotify-app _build_within_docker`
- `Makefile` 第 9 行、第 126-128 行：容器内编译

> **关键细节**：Dockerfile 中 `ARG LD_FLAGS` 只是构建参数，不会自动成为容器内的环境变量。必须在 RUN 指令中显式写 `LD_FLAGS=${LD_FLAGS} make ...` 才能将 ARG 值传递给 make 子进程。

---

### 路径 B：Master 分支镜像构建

Master 构建**只产出 Docker 镜像**，不产出独立二进制文件（CI 中 master 分支不会执行 `make build`，只执行 `make build-docker-multiarch-master`）。

LD_FLAGS 的注入方式与 Tag 发布不同：**不经过 CI 环境变量，直接在 Makefile 中内联构造完整字符串**。

```
Makefile 中内联 LD_FLAGS 字符串
    （Makefile 第 120 行直接硬编码完整 ldflags）
       ↓ 第 1 步：--build-arg LD_FLAGS="-w -s -X main.Version=master-$(shell git rev-parse --short HEAD) ..."
Dockerfile ARG LD_FLAGS=""
       ↓ 第 2 步：LD_FLAGS=${LD_FLAGS} make _build_within_docker
Builder 容器内 Shell 环境变量
       ↓ 第 3 步：go build ... -ldflags "$$LD_FLAGS"
go build 链接器参数
       ↓ 第 4 步
Go 二进制 .data 段
```

涉及文件：
- `.github/workflows/build.yml` 第 60-62 行：master 分支执行 `make DOCKER_BUILD_PUSH=true build-docker-multiarch-master`
- `Makefile` 第 110-122 行：`build-docker-multiarch-master` 目标，第 120 行直接内联完整 LD_FLAGS
- `docker/Dockerfile` 第 35 行、第 53 行：与 Tag 发布路径相同

> **区别于 Tag 发布**：Tag 发布的 LD_FLAGS 是在 CI 中拼接后通过环境变量传给 Makefile，而 Master 构建的 LD_FLAGS 是 Makefile 自己通过 `$(shell ...)` 函数即时构造后传给 Docker 的，少了中间的 CI 环境变量环节。

---

### 路径 C：本地默认值（无 LD_FLAGS 传递）

本地开发场景下，开发者执行 `go run app.go` 或不带 ldflags 的 `go build`，**整个 LD_FLAGS 传递链路不发生**。

```
（无 LD_FLAGS 参与）
       ↓
Go 源码包级变量默认值
    app.go 第 16-25 行：
        Version   = "unknown"
        Commit    = "unknown"
        BuildDate = "unknown"
        Mode      = mode.Dev
       ↓
后续链路同前（封装 VersionInfo → router.Create → 四个出口）
```

涉及文件：
- `app.go` 第 16-25 行：包级变量的字符串字面量默认值

---

### 三条路径传递对比表

| 传递环节 | Tag 发布（二进制） | Tag 发布（Docker 镜像） | Master 镜像 | 本地默认 |
|---|---|---|---|---|
| CI 环境变量文件 `$GITHUB_ENV` | ✅ 经过 | ✅ 经过 | ❌ 不经过 | ❌ 不经过 |
| CI Runner Shell 环境变量 | ✅ 经过 | ✅ 经过 | ❌ 不经过 | ❌ 不经过 |
| Makefile `$$LD_FLAGS` 展开 | ✅ 经过 | ✅ 经过 | ✅ 经过（但 Makefile 内联构造） | ❌ 不经过 |
| `docker run -e LD_FLAGS=`（跨平台编译容器） | ✅ 经过 | ❌ 不经过 | ❌ 不经过 | ❌ 不经过 |
| `docker buildx build --build-arg LD_FLAGS=` | ❌ 不经过 | ✅ 经过 | ✅ 经过 | ❌ 不经过 |
| Dockerfile `ARG LD_FLAGS` | ❌ 不经过 | ✅ 经过 | ✅ 经过 | ❌ 不经过 |
| Dockerfile RUN 中 `LD_FLAGS=${LD_FLAGS} make` | ❌ 不经过 | ✅ 经过 | ✅ 经过 | ❌ 不经过 |
| 容器内 Shell 环境变量 | ✅ 经过 | ✅ 经过 | ✅ 经过 | ❌ 不经过 |
| `go build -ldflags "$$LD_FLAGS"` | ✅ 经过 | ✅ 经过 | ✅ 经过 | ❌ 不经过 |
| Go 源码默认值兜底 | ❌ 被覆盖 | ❌ 被覆盖 | ❌ 被覆盖 | ✅ 使用默认值 |

---

### 关键传递机制补充说明

1. **Makefile 中 `$$LD_FLAGS` 的含义**：
   - Makefile 中 `$` 是特殊字符（用于引用 Make 变量，如 `$(GO_VERSION)`）
   - 写 `$$` 会被展开为单个 `$`，传递给 Shell
   - 因此 `$$LD_FLAGS` 在 Make 展开后变成 Shell 中的 `$LD_FLAGS`，即读取 Shell 环境变量

2. **Docker ARG 与 ENV 的区别**：
   - `ARG LD_FLAGS=""`：仅在构建阶段（`docker build` 期间）可用，不会成为镜像内运行时的环境变量
   - `RUN LD_FLAGS=${LD_FLAGS} make ...`：显式将 ARG 值注入到该条 RUN 命令的 Shell 执行环境中
   - 如果写成 `RUN make ...`，则 make 进程读不到 LD_FLAGS，因为 ARG 不会自动出现在 RUN 的环境里

3. **跨平台编译容器的环境变量传递**：
   - `DOCKER_RUN` 中的 `-e LD_FLAGS="$$LD_FLAGS"`：将当前宿主机 Shell 的 LD_FLAGS 传入启动的编译容器
   - 编译容器内的 Shell 可以直接读到 `$LD_FLAGS`，再由 `_build_within_docker` 目标通过 `${DOCKER_GO_BUILD}` 传给 `go build -ldflags`
