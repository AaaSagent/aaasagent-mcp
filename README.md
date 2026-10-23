# AaaSagent MCP

A public, read-only remote MCP server for browsing 632 MIT-licensed agent templates. No account, no API key.

This repository holds documentation only. The server runs on AaaSagent's own infrastructure — there is no server code here.

```
https://aaasagent.xyz/api/v1/mcp
```

| | |
|---|---|
| Transport | Streamable HTTP |
| Protocol | JSON-RPC 2.0, MCP `2025-06-18` |
| Auth | None |
| Tools | 1 (`list_templates`) |
| Templates | 632, MIT |

## What it does

- Exposes one tool, `list_templates`, that returns agent templates from the AaaSagent catalog.
- Every template uses the same structure: inputs, outputs, and conditions of use.
- Every response carries a `counts` object, so you can see how many entries matched and how many were returned.

## What it doesn't do

- It doesn't run templates or call models. Templates run in your own environment with your own model keys (BYOK).
- It never asks for, receives, or stores your model keys.
- It doesn't write anything. The endpoint is read-only.
- It doesn't require an AaaSagent account or invite code.

## Quick start with curl

Initialize:

```bash
curl -sS https://aaasagent.xyz/api/v1/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2025-06-18",
      "capabilities": {},
      "clientInfo": { "name": "curl", "version": "0.0.0" }
    }
  }'
```

List tools:

```bash
curl -sS https://aaasagent.xyz/api/v1/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H 'MCP-Protocol-Version: 2025-06-18' \
  -d '{ "jsonrpc": "2.0", "id": 2, "method": "tools/list" }'
```

Call `list_templates`:

```bash
curl -sS https://aaasagent.xyz/api/v1/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H 'MCP-Protocol-Version: 2025-06-18' \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": { "name": "list_templates", "arguments": {} }
  }'
```

The accepted arguments are defined by the tool's `inputSchema` in the `tools/list` response. Treat that as the source of truth over this README.

If the `initialize` response includes an `Mcp-Session-Id` header, send it back verbatim on subsequent requests.

## Response shape

`tools/list` returns exactly one tool (abbreviated):

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [
      {
        "name": "list_templates",
        "description": "...",
        "inputSchema": { "...": "..." }
      }
    ]
  }
}
```

The `list_templates` payload carries a `counts` object alongside the templates. Values below are illustrative:

```json
{
  "counts": {
    "matched": 632,
    "returned": 20,
    "all": 649,
    "free": 632,
    "restricted": 17
  }
}
```

| Field | Meaning |
|---|---|
| `matched` | Entries matching your arguments |
| `returned` | Entries included in this response |
| `all` | Every entry in the catalog manifest |
| `free` | MIT-licensed entries, which this release covers |
| `restricted` | Manifest entries outside this release |

## Client setup

Claude Code:

```bash
claude mcp add --transport http aaasagent https://aaasagent.xyz/api/v1/mcp
```

Most other clients accept a JSON block shaped like this — check your client's documentation for exact field names:

```json
{
  "mcpServers": {
    "aaasagent": {
      "url": "https://aaasagent.xyz/api/v1/mcp"
    }
  }
}
```

Any client that supports Streamable HTTP can use the endpoint URL directly. No headers or tokens are needed.

## Behavior and limits

| Request | Response |
|---|---|
| `POST` | JSON-RPC handling |
| `GET` | `405 Method Not Allowed`, `Allow: POST, OPTIONS` |
| `OPTIONS` | `204 No Content` |

- CORS: `Access-Control-Allow-Origin: *`. Allowed headers include `MCP-Protocol-Version`, so browser-based clients work.
- Rate limit: 60 requests per 60 seconds. Only `tools/call` counts; `initialize` and `tools/list` don't.
- Over the limit: `429 Too Many Requests` with a `Retry-After` header. Wait that many seconds before retrying.
- Health check:

```bash
curl -sS https://aaasagent.xyz/api/v1/health
```

The AaaSagent website is in invite-only beta, with a daily cap on invite codes. The MCP endpoint is separate and open to everyone.

## Where it's listed

- Official MCP Registry: `xyz.aaasagent/mcp`
- [Glama connector page](https://glama.ai/mcp/connectors/xyz.aaasagent/mcp) — current health status is shown there
- [awesome-remote-mcp-servers](https://github.com/punkpeye/awesome-remote-mcp-servers)

## License

The 632 templates served through this endpoint are MIT-licensed. The documentation in this repository is released under MIT — see [LICENSE](./LICENSE).

## Changelog

### 2026-10-24

- Initial public release of this documentation.
- Endpoint `https://aaasagent.xyz/api/v1/mcp`, MCP `2025-06-18`, Streamable HTTP.
- One tool: `list_templates`. 632 MIT templates.
- `GET` returns `405` with `Allow: POST, OPTIONS`; CORS enabled; health check moved to `/api/v1/health`.
- `list_templates` gained a `counts` object; the tool description no longer hardcodes a template count.
- Listed on the official MCP Registry (`xyz.aaasagent/mcp`), Glama, and awesome-remote-mcp-servers.

## 中文说明

AaaSagent MCP 是一个公开、只读的远程 MCP 服务器，用来检索 632 个采用 MIT 许可的 agent 模板。不需要账号，也不需要 key。

- 端点：`https://aaasagent.xyz/api/v1/mcp`，Streamable HTTP，JSON-RPC 2.0，协议版本 `2025-06-18`
- 只有一个工具 `list_templates`，每个模板统一包含输入、输出和适用条件
- 返回体带 `counts`，能看到匹配数、返回数和总数
- 模板在你自己的环境里运行，用你自己的模型 key。平台不代跑推理，也不接触你的 key
- 限流为每 60 秒 60 次，只统计 `tools/call`，超限返回 429 和 `Retry-After`
- 网站目前是邀请制内测，每日发码有上限；MCP 端点本身对所有人开放
- 已收录于官方 MCP Registry（`xyz.aaasagent/mcp`）和 Glama

---

站点：[aaasagent.xyz](https://aaasagent.xyz) · 沪ICP备2026041302号
