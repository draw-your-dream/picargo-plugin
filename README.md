# picargo — Claude Code 插件

> 🚀 **新成员快速接入**:打开 <https://claude.ai/claude-code/onboard/cKGyn9PuPwbI>,按你的客户端选路径——Claude Code(CLI / VS Code)三步装插件;claude.ai 网页 / Claude 桌面端配连接器——即可接入 picaa-cargo 部署工具。

在 Claude Code 里部署与运维 [picaa-cargo](https://github.com/draw-your-dream/picaa-cargo) 的 artifacts。
插件通过**远程 OAuth MCP**(`https://cargo.picaa.com/mcp`)暴露 17 个工具:远程 deploy
(`deploy_begin` / `deploy_commit` / `deploy_files`)、`status` / `logs` / `rollback`、
secrets / tokens 管理等(SSO 登录、以你自己的身份走 RBAC),并配套部署 know-how skills
(`deploy` / `troubleshoot` / `conventions`)。

本仓同时是一个 Claude Code **plugin marketplace**(`.claude-plugin/marketplace.json` 里唯一的
插件条目 `source` 指向仓库根目录)。

## 支持矩阵

| 环境 | skills(know-how) | MCP 工具(deploy/ops) | 接入方式 |
|---|---|---|---|
| Claude Code CLI | ✅ | ✅ | **装本插件**(见下),首次 SSO 一次 |
| VS Code Claude 插件 | ✅ | ✅ | 同上(与 CLI 共用配置) |
| claude.ai 网页 / Claude 桌面 | ✅(手动) | ✅ | 在 claude.ai 里把 `cargo.picaa.com/mcp` 配成 **connector**(托管 OAuth) |

> 三端都走同一个远程 OAuth MCP、同一个 Authentik 公开客户端;区别只是 CLI/VS Code
> 由本插件预置配置,web/桌面由 claude.ai connector 托管。

## 安装(CLI / VS Code)

```bash
# 1) 添加本仓作为 marketplace
claude plugin marketplace add https://github.com/draw-your-dream/picargo-plugin.git
#   本地开发调试也可直接指向工作区:
#   claude plugin marketplace add /Users/<you>/Work/picaa/picargo-plugin

# 2) 安装插件(marketplace name 是 "picaa")
claude plugin install picargo@picaa
```

装好后,插件自带的远程 MCP server `picaa-cargo`(见 `.mcp.json`)即随会话启用。

## 首次授权(SSO)

装插件后,起一个新的交互式 `claude` 会话,输入 `/mcp`,对 `picaa-cargo` 选 **Authenticate**
即可走一次浏览器 **Authentik SSO**(授权码 + PKCE);登录成功后 token 安全存储并自动
续期(refresh),之后无感。首次也可能在你调用某个 picaa-cargo 工具时自动弹出授权。

- 使用的是 Authentik `picaa-cargo-mcp` **公开客户端**(client_id 内嵌在 `.mcp.json`,公开值非机密)。
- 回调走本机 loopback:`http://localhost:8611/callback`(`.mcp.json` 里 `oauth.callbackPort` 固定为 8611,
  与 Authentik 该客户端的 redirect URI 对齐)。
- 你的可见/可操作范围由你在 Authentik 里的身份 + picaa-cargo 的 RBAC 决定(与 web/桌面一致)。

> **注意插件 server 的全名带命名空间:`plugin:picargo:picaa-cargo`**(不是裸的 `picaa-cargo`)。
> 命令行操作要用全名。`/mcp` 面板按显示名操作,更省事。

若某工具报未授权,手动重登(注意用全名):

```bash
claude mcp login "plugin:picargo:picaa-cargo"
# 顽固时先登出再登:
claude mcp logout "plugin:picargo:picaa-cargo" && claude mcp login "plugin:picargo:picaa-cargo"
```

## 可用工具(远程 MCP)

`capabilities`(自描述)、`get_conventions`(部署约定)、`status` / `logs` / `secrets_list` /
`tokens_list` / `build_status`(只读)、`rollback` / `secrets_set` / `secrets_rm` /
`tokens_issue` / `tokens_revoke` / `delete` / `restore`(写操作,均带 confirm 门)、以及远程
deploy 三件套 `deploy_begin`(发上传坐标,无 confirm)→ `curl -T` 上传 → `deploy_commit`
(confirm 门;static 同步 / platform 异步用 `build_status` 轮询);纯聊天环境用
`deploy_files`(confirm 门,内联 ≤2MB,仅 static)。不确定能做什么时先调 `capabilities`。

> 注意:`preflight` / 裸 `deploy` 是本地 CLI 命令,**不在**远程 MCP 工具集里。
> sso artifact 支持**密级门禁** `access.tier: low|medium|high`(默认 low;发布者本人始终可见;
> 详见 conventions skill 或 `get_conventions`)。

## 仓库结构

```text
.claude-plugin/
  plugin.json         # 插件 manifest(mcpServers → ./.mcp.json;skills → ./skills/)
  marketplace.json    # 本仓即 marketplace,唯一条目指向仓库根目录
.mcp.json             # 远程 OAuth MCP server 定义(type:http + oauth.clientId/callbackPort)
skills/
  deploy/SKILL.md        # 部署流程:判环境选路径(CLI/远程上传/内联)→ 自查 → 选模式 → confirm 门 → 读结果
  troubleshoot/SKILL.md  # 排障:先取证(status+logs)再对症;错误对照表(现象/根因/处理)+ 回滚
  conventions/SKILL.md   # 可部署代码的硬约束(要求→违反后果)+ access.tier 密级 + 交付前自查清单
```

## 故障排查

- **工具显示未授权 / Needs authentication**:`/mcp` 里 Authenticate,或
  `claude mcp login "plugin:picargo:picaa-cargo"`(**用全名**);确认浏览器 SSO 走完、
  你的账号在 Authentik 里有效。
- **`No MCP server named "picaa-cargo"`**:插件 server 的名字带命名空间,是
  `plugin:picargo:picaa-cargo`,不是裸的 `picaa-cargo`。`claude mcp list` 可看到真名。
- **端口 8611 被占**:OAuth 回调需要在登录那几秒内占用本机 8611。极少数情况下若被占,
  关掉占用进程后重试。
- **改了配置 / 插件不生效**:`claude plugin marketplace update picaa`(拉最新)+
  `claude plugin update picargo@picaa`(升级已装插件),再**重启** `claude`。
