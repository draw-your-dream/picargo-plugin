---
name: picargo-conventions
description: 为将要部署到 picaa-cargo(*.at.sowii.net)的代码——前端、后端服务、Dockerfile、cargo.yaml——生成或改造时使用,让代码天生满足平台约束、不在部署期被扫描/校验硬阻断。纯知识型,任何环境(含无工具的 claude.ai 网页对话)都适用。只管"代码怎么写才可部署",不负责执行部署(执行用 picargo-deploy)、也不负责排障(用 picargo-troubleshoot)。
---

# picaa-cargo 部署约定

**这个 skill 是什么**:一份"让代码天生可部署"的硬约束清单。你在**生成或修改**将要部署到 picaa-cargo 的代码时,把每条当作必须满足的约束。纯知识型,不调用任何工具,任何环境都能用。

**怎么用**:边写边遵守;每条规则写成「要求 → 违反后果」,据此判断某写法会不会在部署期被拦、以及被拦时的表现。**拿不准平台的确切行为就别臆测**——能调工具时用 `preflight` / `capabilities` / `get_conventions` 核实,否则明确告诉用户"这点需在平台侧确认"。写完后用文末的**交付前自查清单**逐条核对再交付。

## 通用(所有 type 都适用)

- 密钥(API key、AWS access key、PEM 私钥、令牌等)**绝不**写进源码或提交进 git → 用运行时密钥(`secrets_set`)注入为容器环境变量(部署后读不回明文)。构建期会扫描源码与产物,命中密钥模式(含裸 `Authorization: Bearer <token>`)即**硬阻断部署(HTTP 422)**,不是警告。
- 前端调用自身后端一律用**相对路径**(如 `/api/...`) → 硬编码 `localhost:<port>` / `127.0.0.1:<port>` 会被构建期标记为**警告**(不阻断部署,但按必修 bug 处理,别当噪音)。
- `/_cargo/*` 是平台整段保留路径(健康检查 / 登录 / 回调 / 登出 / me / AI 代理) → 不要在应用里注册任何 `/_cargo/...` 路由;注册了也不会生效。
- 不要注册**根作用域**(scope 覆盖 `/`)的 Service Worker → 会被平台 403 拦截。普通 Web Worker(`new Worker(...)`)不受影响。
- 不要给自设 cookie 指定比自身 host 更宽的 `Domain`(同注册域 `at.sowii.net` 下的兄弟子域可能读到);也不要信任非自身设置的 cookie → 身份**只**从 `X-Cargo-Identity` 验签获取,不要解析平台自己的会话 cookie。

## 动态服务(`type: dynamic`)

- 必须监听 `0.0.0.0:$PORT`(`$PORT` 平台注入,默认 `8080`) → 绑 `127.0.0.1` 或写死其它端口,平台探针/网关连不上容器进程 → 部署失败。
- 只有 `/data` 目录(需 `cargo.yaml` 里 `data: true` 显式开启)在重启/重部署后保留 → 其余文件系统(含 `/tmp`、`$HOME`、工作目录)都是临时的,重启即丢。
- 请求鉴权的**唯一**可信依据是请求头 `X-Cargo-Identity`(裸 EdDSA JWS,**无** `Bearer ` 前缀)。验证时必须同时满足:`alg=EdDSA`(显式拒绝 `none`/`HS256`/RSA,防算法混淆)、`iss=https://cargo.picaa.com`、`aud=<你自己的 slug>`、`exp` 未过期;公钥从 `https://auth.cargo.picaa.com/.well-known/jwks.json` 按 JWT header 的 `kid` 选取 → **少验任一项 = 鉴权可被绕过**。便捷头 `X-Cargo-User-Id` / `X-Cargo-User-Name` 未签名,只能用于展示/日志,**绝不能**作鉴权依据。
- 不要用 `Authorization` 头做自定义鉴权 → 在 `*.at.sowii.net` 上该头被平台保留给机器令牌 `cargo_tok_*`,任何非法值都会被网关直接 **401**、到不了应用代码。自定义凭证请改用别的头(如 `X-My-Key`)。

## 静态站点(`type: static`)

- 部署产物 = `build.output`(默认 `output/`)目录下的静态文件全树,原样上传、由 static-server 直接返回 → 没有服务端渲染/服务端路由;SPA 的深链/刷新需前端自行处理(平台不做 history fallback)。

## Dockerfile(`type: dynamic` 的 `platform` / `local` 两种 `build.mode` 都必需)

- 源码目录必须自带 `Dockerfile`(平台不自动生成) → `platform` 模式缺失会报 `error resolving dockerfile path`。
- **`platform` 模式(推荐给 AI/agent:只上传源码,平台在 kaniko 沙箱内构建,无需本地 docker)**:`FROM` 必须用 ACR 私有仓 `cargo/base/*` 前缀的基础镜像(集群内 `docker.io` / `gcr.io` **不可达**),现有:`golang:1.25`、`golang:1.25-alpine`、`distroless-static:nonroot`、`node:22-alpine`、`python:3.12-alpine` → 用其它镜像源会因拉不到而构建失败。Go 构建显式设 `GOPROXY=https://goproxy.cn,direct`(平台对检测到 `go.mod` 的构建会自动注入,Dockerfile 里再设一遍作纵深)。
- `local` 模式仅当本机确有 docker 且用户明确要求时用(本地 `docker build` 后由平台代发短时令牌推送)。
- 必须带**依赖锁文件**:Go 用 `go.mod` + `go.sum`(纯标准库项目给一个**空 `go.sum`** 仅为过门)、Node 用 `package-lock.json`、Python 用 `requirements.txt` 等 → 缺锁文件 = **422**。

## 访问控制(`cargo.yaml` 的 `access` 块)

- `access.browser: sso | anonymous | key`(默认 `sso`,三选一互斥) → `sso`=登录后按身份放行;`anonymous`=公开;`key`=共享访问密钥网关。
- `access.tier: low | medium | high`(**可选,且仅在 `access.browser: sso` 下有效**) → 按篇"密级门禁",限定哪些用户组可在浏览器访问本 artifact。默认 `low`(最低密、面向最广人群);**层级包含**——能访问 `high` 的人 ⊆ 能访问 `medium` ⊆ 能访问 `low`(高密 = 更小人群)。发布者本人(owner/maintainer)**始终**能访问自己发布的内容,不受密级限制。
  - ⚠️ 只能写 `low` / `medium` / `high`:拼错或写未知值 → **除发布者外无人可访问**(fail-closed),且**部署时不报错**(发布者自己仍看得到,极易误判"正常")——务必核对拼写。
  - `access.browser` 为 `anonymous` / `key` 时写了 `tier` → **422**。
  - 密级 → 具体用户组的映射由平台侧维护、可随时调整,清单里**只写别名、不写组名**。

## `cargo.yaml` 最小形状

```yaml
name: my-app          # 必填,小写 DNS label:^[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?$
                      # 决定域名 <name>.at.sowii.net 与身份 JWT 的 aud;console 是保留名
type: dynamic         # dynamic(配 build.mode)| static(配 build.output)
build:
  mode: platform      # dynamic 专用:platform(推荐)| local
  # output: output    # static 专用:产物目录,默认 output/
access:
  browser: sso        # sso(默认)| anonymous | key
  tier: low           # 可选,仅 sso:low | medium | high(默认 low)
data: false           # dynamic 可选:true 才有持久化 /data
```

## 交付前自查清单(逐条核对你生成/改动的代码,不满足就先改)

- [ ] 无任何密钥硬编码进源码/配置/git —— 改用 `secrets_set` 注入。
- [ ] 前端调后端全用相对路径,无 `localhost` / `127.0.0.1` 写死。
- [ ] 未注册任何 `/_cargo/*` 路由;未注册根作用域 Service Worker。
- [ ] 未用 `Authorization` 头做自定义鉴权(改用别的头)。
- [ ] (dynamic)监听 `0.0.0.0:$PORT`;持久化只用 `/data`(且 `data: true`)。
- [ ] (dynamic 鉴权)只信 `X-Cargo-Identity` 验签,`alg`/`iss`/`aud`/`exp` 全验。
- [ ] 有 `Dockerfile`;`FROM` 用 `cargo/base/*`;带依赖锁文件(Go 允许空 `go.sum`)。
- [ ] `cargo.yaml`:`name` 合法;`type`/`build` 对应;`access.browser` 合法;`access.tier`(若写)∈ {`low`,`medium`,`high`} 且 `browser` 为 `sso`。
