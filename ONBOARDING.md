# picaa-cargo 部署工具 · 团队接入指南

在 Claude 里用自然语言部署与运维 [picaa-cargo](https://cargo.picaa.com) 的 artifacts ——
`deploy` / `status` / `logs` / `rollback` / `secrets` / `tokens` 等。以你自己的身份走
SSO + RBAC;Claude Code(CLI / VS Code)、claude.ai 网页、Claude 桌面端连的是**同一个**
远程 MCP 服务(`https://cargo.picaa.com/mcp`),权限一致。

## 先选接入路径

| 你用什么 | 走哪条路径 | 能力差异 |
|---|---|---|
| **Claude Code CLI / VS Code Claude 插件** | 路径 A:装 picargo 插件 | MCP 工具全量 + 部署 know-how **skills** + 有 Bash(可走 `deploy_begin` + `curl -T` 上传大包,≤1GB) |
| **claude.ai 网页 / Claude 桌面端** | 路径 B:配 picaa-cargo 连接器(connector) | MCP 工具全量;无 Bash → 部署走 `deploy_files`(内联 ≤2MB,仅 static);skills 不可用 |

## 前提(两条路径都需要)

- 你在公司 Authentik(`auth.picaa.com`)有一个有效账号(找管理员开通 / 加入相应 group)。
- 对应的 Claude 客户端已登录你的 Claude 账号。

## 路径 A:Claude Code(CLI / VS Code 插件)

### 三步接入

```bash
# 1) 添加插件市场(picaa)
claude plugin marketplace add https://github.com/draw-your-dream/picargo-plugin.git

# 2) 安装插件
claude plugin install picargo@picaa
```

**3) 首次授权(SSO)** —— 打开一个交互式 `claude` 会话,输入 `/mcp`,对 `picaa-cargo`
选 **Authenticate**,浏览器会弹出 Authentik 登录;登录完成即接入。之后 token 自动续期,无感。

> 命令行等价方式(注意插件 server 全名带命名空间前缀):
>
> ```bash
> claude mcp login "plugin:picargo:picaa-cargo"
> ```

### 验证(路径 A)

```bash
claude mcp list | grep picaa-cargo
# 期望:plugin:picargo:picaa-cargo: https://cargo.picaa.com/mcp (HTTP) - ✔ Connected
```

VS Code Claude 插件与 CLI 共用同一份配置,CLI 装好即两处可用。

## 路径 B:claude.ai 网页 / Claude 桌面端(连接器)

连接器配置挂在你的 **claude.ai 账号**上:网页配一次,Claude 桌面端同账号登录后自动可用,无需重复配置。

1. 打开 claude.ai → **设置(Settings)** → **连接器(Connectors)** → **添加自定义连接器(Add custom connector)**。
2. 名称填 `picaa-cargo`,远程 MCP 服务器 URL 填:`https://cargo.picaa.com/mcp`。
3. 展开**高级设置(Advanced settings)**,**OAuth Client ID** 填:
   `38jwbecDT1qsYUoG8YAE0bGnal9BnGxygf5shat1`
   (公开客户端标识,非机密;**Client Secret 留空**——平台用的是公开客户端 + PKCE)。
4. 保存后点 **连接(Connect)** → 浏览器跳 Authentik SSO 登录授权,完成即接入。
5. 对话中通过「搜索和工具」(🔧)菜单确认 picaa-cargo 工具已启用。

### 验证(路径 B)

在对话里直接说:

> 用 picaa-cargo 的 status 工具列出我的 artifact

能返回你的 artifact 列表 = 全链路通。

## 你能做什么

- **只读**:`status`(状态)、`logs`(日志)、`secrets_list` / `tokens_list`、`access_key_show`(访问密钥元数据/取回明文)、`get_conventions`(部署约定)、`capabilities`(自描述)、`build_status`(构建进度)。
- **写操作(均有 confirm 确认门)**:`rollback`、`secrets_set` / `secrets_rm`、`tokens_issue` / `tokens_revoke`、`delete` / `restore`、`access_key_rotate`(访问密钥签发/换新,可带 ttl)。
- **远程部署**:`deploy_begin`(发上传坐标,无 confirm)→ `curl -T` 上传 → `deploy_commit`(confirm 门;static 同步返回 / platform 异步,用 `build_status` 轮询到 ready;≤1GB)。纯聊天小站用 `deploy_files`(confirm 门,内联 ≤2MB,仅 static)。`preflight` / 裸 `deploy` 是本地 CLI 命令,不在远程工具集里。
- **密级门禁**:sso artifact 可在部署时指定 `access.tier: low | medium | high`(默认 low=全员;medium=业务管理层;high=公司管理层;发布者本人始终可见自己的内容)。只能写这三个别名——拼错会导致除发布者外无人可访问且部署不报错。
- **CDN 直连(灰度中)**:`anonymous` + 静态站点检测合格会自动升级为直连(`m.sowii.net/sites/<slug>/`,更快;旧域名自动 301),不合格自动退回普通模式,不影响部署成功;该特性平台侧未全量开启前,部署结果里不会出现这行,详见 conventions skill。
- **skills(部署 know-how,仅路径 A)**:`deploy`(判环境选路径 → 自查 → 部署 → 读结果)、`troubleshoot`(先取证再对症 + 错误对照表)、`conventions`(可部署代码硬约束 + 交付前自查清单)。

不确定能做什么时,先让它调 `capabilities`。

## 常见问题

- **(A)`No MCP server named "picaa-cargo"`**:插件 server 的名字带命名空间,是
  `plugin:picargo:picaa-cargo`(不是裸的 `picaa-cargo`)。`claude mcp list` 可看真名;命令行用全名。
- **(A)工具报 Needs authentication / 未授权**:`/mcp` 里 Authenticate,或
  `claude mcp login "plugin:picargo:picaa-cargo"`;确认浏览器 SSO 走完、你的 Authentik 账号有效。
- **(A)装了插件但工具没出来 / 版本不对**:

  ```bash
  claude plugin marketplace update picaa
  claude plugin update picargo@picaa
  ```

  然后**重启** `claude`。
- **(A)端口 8611 被占**:OAuth 回调需要登录那几秒占用本机 8611;若被占,关掉占用进程后重试授权。
- **(B)连接器授权失败 / 连不上**:确认你的 Authentik 账号有效且在允许的用户组;在连接器设置里
  **断开(Disconnect)后重新 Connect** 走一遍 SSO。
- **(B)对话里看不到工具**:检查「搜索和工具」菜单里 picaa-cargo 是否已启用;写操作类工具
  每次调用都会先请求你的确认(confirm 门 + 客户端授权,双层都是刻意设计)。

## 支持

问题找 picaa-cargo 维护者(#picaa-cargo)。插件源码:`https://github.com/draw-your-dream/picargo-plugin`。
