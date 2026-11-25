# 将 DetailX 连接到 AI 助手

[产品介绍](README.zh-CN.md) · [English](MCP.md) · [在线接入页](https://www.detail-x.ai/integrations/mcp)

DetailX 提供远程 MCP 服务，让助手查找视频素材、查询积分、估算增强费用、提交已授权任务并获取结果。

| 配置项 | 值 |
| --- | --- |
| 服务地址 | `https://www.detail-x.ai/api/v1/mcp/` |
| 传输协议 | Streamable HTTP |
| 认证 | 带 S256 PKCE 的浏览器 OAuth 授权 |
| 使用账户 | 你的 DetailX 账户 |
| 手动配置 API Key | MCP 不需要 |

请保留地址末尾的 `/`，并使用同时支持远程 Streamable HTTP 和浏览器 OAuth 的客户端。

## 1. 添加服务

### Codex

```sh
codex mcp add detailx --url "https://www.detail-x.ai/api/v1/mcp/"
codex mcp login detailx
codex mcp list
```

按提示在浏览器登录 DetailX 并允许连接。进入新的 Codex 会话，检查 MCP 连接，再请求查询账户。

如果使用配置文件，将以下内容合并到当前 Codex 实际使用的配置中，再执行 `codex mcp login detailx`：

```toml
[mcp_servers.detailx]
url = "https://www.detail-x.ai/api/v1/mcp/"
```

OAuth 连接不要同时配置固定 `Authorization` 请求头或 `bearer_token_env_var`。

客户端参考：[Codex 官方 MCP 文档](https://learn.chatgpt.com/docs/extend/mcp?surface=cli)。

### Claude Code

在工作目录执行：

```sh
claude mcp add --transport http --scope local detailx "https://www.detail-x.ai/api/v1/mcp/"
claude
```

会话中输入 `/mcp`，选择 `detailx`，点击 **Authenticate**。在浏览器登录 DetailX，核对客户端后允许连接。

也可把下方内容合并到项目 `.mcp.json`，按客户端提示启用项目服务，再在 `/mcp` 中认证：

```json
{
  "mcpServers": {
    "detailx": {
      "type": "http",
      "url": "https://www.detail-x.ai/api/v1/mcp/"
    }
  }
}
```

`--scope local` 供自己在当前项目使用；跨项目使用可选 `--scope user`，共享项目配置可选 `--scope project`。PowerShell 阻止 `claude.ps1` 时，使用 `claude.cmd` 执行相同命令。

客户端参考：[Claude Code 官方 MCP 文档](https://code.claude.com/docs/en/mcp)。

### 其他 MCP 客户端

在客户端的 MCP 设置中添加上述远程 HTTP 地址，并完成浏览器 OAuth。Cursor、Gemini CLI、VS Code/GitHub Copilot、OpenCode 等客户端按各自的 MCP 设置操作，实际支持以安装版本为准。

如果助手具备配置 MCP 的权限，也可以直接对它说：

> 帮我添加远程 MCP 服务 https://www.detail-x.ai/api/v1/mcp/，命名为 detailx。需要授权时打开浏览器，让我登录并确认。

## 2. 先用只读请求验证

> 使用 DetailX 查询我的积分余额和最近 5 个视频素材，先不要生成。

`get_account` 或 `list_uploads` 实际调用成功才表示连接可用。仅在配置文件中添加地址不能证明已经连通。

## 3. 估价、确认、增强

1. 调用 `get_capabilities`，通过 `list_uploads` / `get_upload` 选择可用输入。
2. 选择输出参数：分辨率为 `720p`、`1080p`、`2K`、`4K`；帧率为 `-1`（原帧率）、`24`、`30`、`60`。可选增强参数以实时能力为准。
3. 调用 `estimate_generation`，向用户展示素材、参数、预计积分 `credits` 和价格版本 `pricing_revision_id`。
4. 用户授权生成后，调用 `create_generation`，使用相同参数，将报价的 `credits` 原样作为 `expected_credits`，同时携带 `pricing_revision_id` 与预先保存的 `idempotency_key`。
5. 使用 `get_generation` 查询同一任务直至完成或失败，再返回可用结果链接。已受理不等于已完成。

对话示例：

> 把 “interview.mp4” 按 4K、30 FPS、开启颜色保持估价，提交前先告诉我所需积分。

> 按刚才的素材、参数和报价生成一次，保存幂等键，持续查询同一条任务直到完成。

遇到超时或响应丢失，先查询原任务；确需重放时必须沿用相同幂等键和完全一致的参数。遇到 `price_changed` 要重新报价并确认，不能新建第二个任务来检查第一个是否成功。

MCP 使用上述估价与确认流程。独立的 API Key 入口 `/api/v1/developer/generations` 有自己的请求协议，不要将它的请求规则套用到 MCP 工具。

## 4. 上传本地视频

本地自动上传要求客户端能读取指定文件、执行本地命令并访问 HTTPS 网络。官方脚本需要 Python 3.10+，首帧流程需要本机 FFmpeg。远程 MCP 服务本身不能读取你的磁盘，也不通过工具消息接收视频字节。

可以对助手说：

> 用 DetailX 上传本机 `C:\Videos\input.mp4`。先读取当前上传限制，上传完成后按 1080p、原帧率估价，先不要生成。

助手应按以下顺序执行：

1. 调用 `get_upload_instructions` 获取实时策略，调用 `get_local_upload_client` 获取正式脚本及 SHA256。
2. 原样保存脚本并校验哈希，在本机准备视频，遵守脚本的元数据和首帧要求。
3. 使用保存的身份与幂等键建立或恢复上传会话。
4. 通过 MCP 获取分片签名，由本机直接向存储上传；遵守返回的并发等限制，并保留脚本续传状态。
5. 提交上传完成请求，查询素材达到 `ready` 后再估价。

OAuth 始终由 MCP 客户端管理。本地脚本使用签名上传地址，不要求提取 OAuth 令牌或另配 API Key。签名 URL 与本地状态不要提交到公开仓库。

没有本地执行能力或 FFmpeg 时，改用 [DetailX 素材页](https://www.detail-x.ai/files)上传，再使用对应素材继续。每次新上传都读取实时限制，不写死大小或视频时长上限。

## 工具一览

| 用途 | 工具 |
| --- | --- |
| 能力与并发 | `get_capabilities`、`get_generation_capacity` |
| 账户与积分 | `get_account`、`list_credit_transactions` |
| 查找视频素材 | `list_uploads`、`get_upload` |
| 本地上传支持 | `get_upload_instructions`、`get_local_upload_client` |
| 创建与查询上传 | `create_upload_session`、`list_upload_sessions`、`get_upload_session` |
| 签名与完成上传 | `sign_upload_parts`、`sign_upload_thumbnail`、`resume_upload`、`complete_upload`、`cancel_upload` |
| 估价与生成 | `estimate_generation`、`create_generation` |
| 任务与结果 | `list_generations`、`get_generation` |
| 用户明确确认后删除媒体 | `delete_media`、`get_media_deletion` |

实际参数与返回值以在线连接返回的工具 schema 为准。

## 积分、权限与授权管理

MCP 复用现有 DetailX 余额和计费规则。报价是预计费用；增强任务成功后按服务的时长规则结算，失败或超时退回预扣。使用实时报价，不假定固定价格；在结果显示的到期时间前下载。

连接可操作本人的账户、素材和增强任务，包含媒体删除。支付、订阅变更、身份管理和管理员功能不在 MCP 权限范围内。

在 [DetailX 账户页](https://www.detail-x.ai/account)的 MCP 区域管理或撤销已连接客户端。撤销阻止新的认证调用，已受理任务按原生命周期继续。删除本机配置不会自动撤销服务端授权：

```sh
codex mcp remove detailx
```

```sh
claude mcp remove --scope local detailx
```

## 常见问题

| 现象 | 处理方式 |
| --- | --- |
| 401 或授权过期 | 重新执行 `codex mcp login detailx`，或在 Claude Code `/mcp` 中重新认证。 |
| 404 或连接失败 | 核对 HTTPS 完整地址，包含 `/api/v1/mcp/` 及末尾斜杠。 |
| 已配置但没有工具 | 核对实际启用的配置和服务，重连后调用 `get_account`；按客户端需要进入新会话。 |
| 本机回调显示连接被拒绝 | 保持登录命令运行，并在同一电脑授权；先用 `get_account` 检查，仍失败则重新登录，不刷新旧回调。 |
| 上传失败或响应丢失 | 恢复并对账同一会话，保留原会话身份和上传状态。 |
| 已上传但没有可用时长 | 提供从真实文件读取的元数据，或改用浏览器上传，不能虚构时长。 |
| 生成请求超时 | 查询原任务；必要重放时保留原幂等键与完整参数。 |
| 价格发生变化 | 重新报价并确认后再提交。 |
| 达到并发或频率限制 | 查询容量，遵守 `Retry-After`，不要反复创建任务。 |

[返回产品介绍](README.zh-CN.md) · [在线帮助](https://www.detail-x.ai/help/mcp) · [联系支持](mailto:support@detail-x.ai)
