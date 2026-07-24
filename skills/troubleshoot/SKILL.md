---
name: picargo-troubleshoot
description: 当 picaa-cargo 上的 artifact 部署失败、运行/访问异常(401/403/404/超时/崩溃)、或用户要查日志/回滚时使用。提供"先取证再对症"的诊断流程 + 错误对照表 + 回滚步骤。写可部署的代码用 picargo-conventions skill;首次部署的完整流程用 picargo-deploy skill。
---

# picaa-cargo 排障

**这个 skill 是什么**:部署失败或运行/访问异常时的诊断手册。**核心原则:先取证(status + logs)再对症,不要凭报错字面猜根因。**

## 动手前先自省:我能不能取证?

- **有 MCP 工具(`status`/`logs`/`rollback`)或本地 CLI** → 按下面先取证、再对症。
- **皆无**(claude.ai 网页对话)→ 不要空转找 CLI;把诊断步骤 + 确切命令写给用户回 Claude Code / 本地终端跑。**但下面的错误对照表可直接用于解读用户贴回的报错**——这是无工具环境下本 skill 最有价值的用法。

## 第 1 步:取证(先做,别跳过)

1. `status`(MCP)或 `picargo status --slug <s>` → 看当前激活版本、部署历史、你的角色。**先确认"线上到底是哪个版本、部署有没有成功"**,很多"没生效"其实是新版本没 active。
2. `logs`(MCP)或 `picargo logs --slug <s> --tail 200`(默认 200 行,上限 2000)→ 看运行日志;`--follow` 实时流(Ctrl-C 停);**`--previous` 看崩溃/重启前那个容器实例的日志,排查"启动即崩溃"必用**。日志逐行脱敏密钥模式,且随 Pod 重建丢失(无持久聚合)。

## 第 2 步:对症(错误对照表)

| 现象 | 根因 | 处理 / 下一步 |
|---|---|---|
| `409 slug is soft-deleted` | 该名字处于软删除保留期(数据/密钥/镜像仍在) | `restore`(MCP 工具,confirm 门)或 CLI `picargo restore --slug <s>` 恢复,或换一个 `name` 部署。 |
| `422 ... secrets detected` | 构建期扫描在源码/产物里命中密钥模式(API key / AWS key / PEM 私钥 / 裸 Bearer token) | 从代码里删掉,改用 `secrets_set` 注入环境变量。 |
| `422 ... lockfile required` | `platform` 构建要求依赖锁文件(`go.sum`/`package-lock.json`/`requirements.txt`/…) | 补齐并提交;纯标准库 Go 项目给个**空 `go.sum`** 即可过门。 |
| `error resolving dockerfile path` | 源码目录缺 `Dockerfile`(`platform`/`local` 都需要,平台不自动生成) | 按 `picargo-conventions` 补一个。 |
| 拉基础镜像 `UNAUTHORIZED` / 拉取失败 | `Dockerfile` 的 `FROM` 用了 `docker.io`/`gcr.io`(集群内不可达) | 改成 ACR `cargo/base/*` 系列基础镜像。 |
| 带 `Authorization` 头访问却 `401` | `Authorization` 在 `*.at.sowii.net` 被平台保留给机器令牌 `cargo_tok_*`,其它任何值(自定义 JWT/Basic/无效令牌)都被网关直接 401 | 换一个自定义头做鉴权,或用 `tokens_issue` 签发有效 `cargo_tok_*` 令牌。 |
| **sso artifact 某些用户访问被 `403`「无权访问/需要更高密级」** | 该 artifact 的 `access.tier` 密级高于用户所在用户组 | 正常拦截:确认该用户是否应有权限。若应有 → 平台侧把用户加入相应组;若门开得太严 → 改低 `access.tier` 重新部署。 |
| **发布者本人能看、其他所有人一律 `403`** | 极可能 `access.tier` **拼错**(未知 tier → fail-closed,只有 owner 可见,且部署不报错) | 核对 `access.tier` 拼写(只能 `low`/`medium`/`high`),改正后重新部署。 |
| `key` 模式访客总被弹回密钥输入页 | 密钥错误/过期/已被 rotate,或该 artifact 已被软删;**经远程 MCP 部署的 key 模式 artifact 从未签发过密钥**也是此症状 | 确认 artifact 未软删;分发最新密钥:CLI `picargo access-key show --reveal` 取回 / `rotate [--ttl <dur|never>]` 换新,远程 MCP 用 `access_key_show`(`reveal:true`)/ `access_key_rotate`。rotate 后旧密钥**立即失效**且在线访客被断开,需重新分发;频繁过期烦人 → 设 `access.key_ttl: never`(或 rotate `--ttl never`)。 |
| 部署成功但访问 `404` | `name` 不合法(大写/带点,本应被两侧校验拦下);或动态服务没监听 `$PORT`/`0.0.0.0` | 检查 `name` 合法性;检查进程确实 `0.0.0.0:$PORT` 监听。 |
| **CDN 直连站(`m.sowii.net/sites/<slug>/`)某些资源 `404`** | 产物里有根绝对路径引用(`src=`/`href=`/`srcset=`/`<base href>`/CSS `url(...)` 以单个 `/` 开头),在共享域下解析到错路径;或是**检测漏过**的启发式盲区——JS 里动态拼出的根绝对引用(如 `fetch('/x')`)测不到 | 查产物里有没有上述根绝对引用,改成相对路径;若是运行时才发现的动态拼接盲区,在 `cargo.yaml` 写 `access.cdn: "origin"` 切到**全代理模式**(不直连、资源也不再走 CDN,全部经 `<slug>.at.sowii.net` 流式回源),不用等平台改检测逻辑;注意翻转会清空该站在 CDN 上的既有内容(`sites/`/`artifacts/` 双前缀,边缘缓存 ≤1h 收敛)。 |
| **直连站前端路由深链 `404`(如 `/foo/bar`)** | 直连入口是纯静态目录服务,不支持 history 模式 SPA 回退,不存在的路径直接收到 OSS 原生 404 XML(不会退回 `index.html`) | 前端改用 hash 路由(`#/foo/bar`);或该站放弃直连,`cargo.yaml` 写 `access.cdn: "origin"` 切到全代理(不再是"仍 302"的现状架构,而是零重定向全经 `<slug>.at.sowii.net`,支持 SPA 前端自行处理)。 |
| 硬编码 `localhost` 警告 | 前端写死了 `localhost:port`/`127.0.0.1:port` | 前端改相对路径调后端(`/api/...`)。 |
| `/data` 写入报磁盘满 | 超过 `data_quota`(软执行,有分钟级滞后) | 调大 `data_quota`(**只增不减**)或清理数据。 |
| platform 构建提交后迟迟没上线 | `deploy_commit` 的 platform 构建是**异步**的,不是卡住 | 用 `build_status` 轮询:`building` 继续等;`ready` 即已上线;`failed_build` → 按其返回信息对照上表(密钥/锁文件/Dockerfile/基础镜像)排查后重新提交。 |
| secrets / 新配置没生效 | 改密钥/清单需要 reconcile 滚动几秒 | `status` 确认新版本已 active 再判断。 |

## 回滚

`rollback`(MCP)/ `picargo rollback --slug <s> --to <version>`:同样走 `confirm` 二次确认门 —— 先 `confirm:false` 预览目标版本,用户同意后再 `confirm:true`。近 10 个历史版本可回滚。⚠️ **`/data` 数据不随代码回滚**,回滚只切代码版本。

## 401 兜底(CLI 层面,而非某个 artifact 返回的)

任何 `401`(尤其 CLI 自身报的,非 artifact 业务 401)→ 先让用户跑 `picargo login`,或检查 `PICARGO_TOKEN` 是否已过期。

## 何时停止试探、上报平台维护者

取证后仍无法定位,或问题涉及**平台侧配置**——用户组白名单 / 密级(tier)→ 组的映射 / ACR 凭证 / 证书 / 集群资源——**不要反复试参数或改代码去撞**。明确告诉用户这需要 picaa-cargo 维护者介入,并附上你已取到的证据(status 输出、相关日志片段、确切报错)。
