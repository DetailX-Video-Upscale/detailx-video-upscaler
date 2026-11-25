# Connect DetailX to your AI assistant

[Product overview](README.md) · [简体中文](MCP.zh-CN.md) · [Online connection page](https://www.detail-x.ai/integrations/mcp)

DetailX provides a remote MCP server for discovering video assets, checking credits, quoting enhancements, submitting authorized jobs, and retrieving results.

| Setting | Value |
| --- | --- |
| Server URL | `https://www.detail-x.ai/api/v1/mcp/` |
| Transport | Streamable HTTP |
| Authentication | Browser OAuth with S256 PKCE |
| Account | Your DetailX account |
| Manual API key | Not required for MCP |

Keep the trailing slash in the server URL. Use a client that supports both remote Streamable HTTP and OAuth browser authorization.

## 1. Add the server

### Codex

```sh
codex mcp add detailx --url "https://www.detail-x.ai/api/v1/mcp/"
codex mcp login detailx
codex mcp list
```

Follow the browser login and consent flow. Open a new Codex session, inspect the MCP connection, and request an account query to verify it.

If you manage configuration files, merge this into the configuration used by your Codex installation, then run `codex mcp login detailx`:

```toml
[mcp_servers.detailx]
url = "https://www.detail-x.ai/api/v1/mcp/"
```

Do not add a fixed `Authorization` header or `bearer_token_env_var` for this OAuth connection.

Client reference: [official Codex MCP documentation](https://learn.chatgpt.com/docs/extend/mcp?surface=cli).

### Claude Code

Run in your working directory:

```sh
claude mcp add --transport http --scope local detailx "https://www.detail-x.ai/api/v1/mcp/"
claude
```

Run `/mcp`, choose `detailx`, and select **Authenticate**. Log in to DetailX in the browser, review the client, and approve the connection.

Alternatively, merge this into a project `.mcp.json`, enable the project server when prompted, and authenticate from `/mcp`:

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

`--scope local` applies to you in the current project. Use `--scope user` for your account across projects, or `--scope project` for shared project configuration. If PowerShell blocks `claude.ps1`, use `claude.cmd` for the same command.

Client reference: [official Claude Code MCP documentation](https://code.claude.com/docs/en/mcp).

### Other MCP clients

In the client's MCP settings, add the server URL as a remote HTTP server and complete browser OAuth. Clients such as Cursor, Gemini CLI, VS Code/GitHub Copilot, and OpenCode can be configured through their own MCP settings; available support depends on the installed version.

If an assistant can configure its own MCP connections, you can ask:

> Add the remote MCP server https://www.detail-x.ai/api/v1/mcp/ as detailx. When authorization is required, open the browser so I can log in and approve it.

## 2. Verify with a read-only request

> Use DetailX to query my credit balance and list my five most recent video assets. Do not create a generation.

A successful `get_account` or `list_uploads` call verifies the usable connection. An entry in a configuration file alone does not.

## 3. Quote, authorize, and enhance

1. Call `get_capabilities` and select an available input using `list_uploads` / `get_upload`.
2. Choose output settings. Enhancement resolutions are `720p`, `1080p`, `2K`, and `4K`; FPS values are `-1` (keep source), `24`, `30`, and `60`. Query capabilities for current optional controls.
3. Call `estimate_generation`. Show the source, parameters, `credits`, and `pricing_revision_id` to the user.
4. After the user authorizes generation, call `create_generation` with the same parameters, the quote's `credits` as `expected_credits`, its `pricing_revision_id`, and a saved `idempotency_key`.
5. Track that task with `get_generation` until it completes or fails. Return the result link when available; acceptance is not completion.

Example conversation:

> Estimate enhancing “interview.mp4” to 4K at 30 FPS with color preservation. Show me the quote before submitting.

> Proceed once with that source, settings, and quote. Keep the idempotency key and check the same task until it finishes.

On a timeout or lost response, inspect the original task first. Any replay must use the same idempotency key and identical parameters. If `price_changed` is returned, obtain a new quote and approval. Do not create a second task to check whether the first succeeded.

MCP follows the quote-and-confirm flow above. The separate API Key endpoint `/api/v1/developer/generations` has its own request contract; do not substitute that contract into MCP tool calls.

## 4. Upload a local video

Local automation requires a client with access to your file, local commands, and HTTPS networking. The official helper requires Python 3.10+ and local FFmpeg for its first-frame workflow. The remote MCP server does not read your disk or accept video bytes in a tool message.

Ask your assistant:

> Upload my local video at `C:\Videos\input.mp4` using DetailX. Discover the current upload limits first, then quote 1080p at the original frame rate. Do not generate yet.

The assistant should:

1. Call `get_upload_instructions` for the live policy and `get_local_upload_client` for the official script and SHA256.
2. Save the returned script unchanged, verify its hash, and prepare the file locally. Follow the helper's metadata and first-frame requirements.
3. Create or resume an upload session with its saved identity and idempotency key.
4. Use the MCP signing tools, then send signed multipart uploads directly from the local computer to storage. Follow the returned limits and preserve the helper's resume state.
5. Complete the upload and query until the asset is `ready`, then obtain the enhancement quote.

OAuth remains in the MCP client. The helper uses signed upload URLs; it does not require extracting an OAuth token or creating an API key. Keep signed URLs and local state out of public repositories.

If local execution or FFmpeg is unavailable, upload through [DetailX Files](https://www.detail-x.ai/files) and continue using the resulting asset. Discover upload limits for every new upload; do not assume a fixed file size or duration limit.

## Tool map

| Purpose | Tools |
| --- | --- |
| Capabilities and capacity | `get_capabilities`, `get_generation_capacity` |
| Account and credits | `get_account`, `list_credit_transactions` |
| Find video assets | `list_uploads`, `get_upload` |
| Discover local upload support | `get_upload_instructions`, `get_local_upload_client` |
| Create and inspect uploads | `create_upload_session`, `list_upload_sessions`, `get_upload_session` |
| Sign and complete uploads | `sign_upload_parts`, `sign_upload_thumbnail`, `resume_upload`, `complete_upload`, `cancel_upload` |
| Quote and generate | `estimate_generation`, `create_generation` |
| Track tasks and retrieve results | `list_generations`, `get_generation` |
| Delete media after explicit user confirmation | `delete_media`, `get_media_deletion` |

Use the tool schemas returned by the live connection as the source of truth for arguments and responses.

## Credits, access, and connection management

MCP uses your existing DetailX credit balance and billing rules. Quotes are estimates; successful enhancement settlement follows the service's duration-based rules, and failed or timed-out jobs refund their precharge. Check the live quote rather than assuming a fixed price. Download output before its displayed expiry.

The connection allows operations on your own account, assets, and enhancement jobs, including media deletion. Payments, subscription changes, identity management, and administration are outside MCP access.

Manage or revoke clients from the MCP section of your [DetailX account](https://www.detail-x.ai/account). Revocation prevents new authenticated calls; already accepted jobs continue through their normal lifecycle. Removing local configuration does not revoke server-side authorization:

```sh
codex mcp remove detailx
```

```sh
claude mcp remove --scope local detailx
```

## Troubleshooting

| Symptom | Next step |
| --- | --- |
| 401 or expired login | Run `codex mcp login detailx`, or reauthenticate in Claude Code's `/mcp`. |
| 404 or connection error | Check the complete HTTPS URL, including `/api/v1/mcp/` and its trailing slash. |
| Configured but no tools | Check the active configuration and enabled server, reconnect, and try `get_account`. Start a fresh session if required by the client. |
| Local callback says connection refused | Keep the login command running and authorize on the same computer. Test `get_account`; if it fails, start a new login rather than refreshing an old callback. |
| Upload fails or response is lost | Resume and reconcile the existing session; keep its original identity and upload state. |
| Uploaded asset has no usable duration | Supply metadata read from the actual file, or upload through the browser. Do not invent a duration. |
| Generation request times out | Query the original task. Reuse its saved key and identical request if replay is necessary. |
| Price changed | Obtain a new quote and approval before submitting. |
| Capacity or rate limit reached | Check capacity and honor `Retry-After`; do not send repeated create requests. |

[Back to DetailX](README.md) · [Online help](https://www.detail-x.ai/help/mcp) · [Support](mailto:support@detail-x.ai)
