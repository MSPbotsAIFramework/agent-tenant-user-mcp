# agent-tenant-user-mcp

MCP server for the **MSPbots Agent Platform directory API** — lets an Agent list platform tenants and the users of a tenant.

> **Naming note:** this is not a third-party vendor integration. It wraps an **internal MSPbots App API** on the Agent Platform. "MSP" in the header names below refers to the MSPbots Agent Platform itself, not an external MSP tool vendor.

<div align="center">

## MSPbots Connector

**The MCP Gateway built for MSPs: visible, controlled, and your data stays where it belongs.**

[**Get started at connect.mspbots.ai →**](https://connect.mspbots.ai)

</div>

---

MSPbots Connector gives your AI clients one MCP URL to reach your entire MSP stack (RMM, PSA, documentation, security, finance), and gives you control over who can call which tool and a record of everything that was called.

**Hosted service: [connect.mspbots.ai](https://connect.mspbots.ai)** · Free to use.

### Why Connector

![MSPbots Connector](https://stgoss.mspbots.ai/files/mcp/mspbots-mcp.png)

| | |
|---|---|
| **Visible** | Every tool call is logged: who, which vendor, when. Usage dashboard, CSV export, SIEM streaming. |
| **Controlled** | Role-based, per-tool allowlist. Deny by default. |
| **Protected** | Credentials encrypted per record. AI clients never see your vendor passwords or tokens. |
| **MSP-native** | Built for the MSP stack only: RMM, PSA, documentation, security, finance. |

### Quick start

1. **Sign up** at [connect.mspbots.ai](https://connect.mspbots.ai).
2. **Connect your vendors.** Pick from 150+ connectors and authorize each one on the vendor's own page, or bring your own API keys. No extra vendor accounts needed.
3. **Set access.** Create roles and decide which tools each role may call. Nothing is allowed until you allow it.
4. **Add your MCP URL to your AI client.** Copy the URL from your Connector dashboard into Claude, ChatGPT, Cursor, VS Code, Gemini, Codex, or any MCP-capable client.
5. **Ask your AI to do real work**, and watch every call appear in the audit log.

### Features

**Unified gateway**
- One MCP URL for all your AI clients
- 150+ connectors across the MSP stack
- Custom MCP servers supported, free
- Authorization happens on each vendor's own page; the AI never sees passwords or tokens
- Bring your own credentials: use the vendor API keys you already have
- Multiple accounts per app; connect once, reuse across AI clients
- Tool search: finds the right action for the task
- One sign-in; vendor credentials are injected behind the scenes and OAuth tokens refresh automatically

**Visibility**
- Audit log for every tool call (who, which vendor, when)
- Usage dashboard: call volume, top vendors, error rate, per-user activity
- Export audit logs to CSV, or stream to Loki, Graylog, or LogScale (SIEM)
- Optional recording of tool arguments and result summaries (off by default, enabled per organization)

**Control**
- Per-role, per-tool allowlist, deny by default
- owner / admin / member roles
- SCIM provisioning with Entra ID, Okta, JumpCloud, and Google Workspace

**Security**
- Credentials encrypted one record at a time (AES-256-GCM, per-record key)
- AI clients authenticate with OAuth 2.1 + PKCE
- Vendor responses are scanned for prompt-injection patterns and returned as structured data. This is one layer of defense, not a guarantee against every attack.
- Static egress IP for customer firewall and API allowlisting

### Pricing

**Free.**

Links: [connect.mspbots.ai](https://connect.mspbots.ai) · [mspbots.ai](https://mspbots.ai)

---

## Overview

- Stateless HTTP service. No credentials are ever persisted — each request supplies its own credential and host via headers, used only for the lifetime of that single request.
- Supports concurrent requests; per-request credential isolation is done via Python `contextvars`, not a global/shared client instance.
- Entry points: `POST /mcp` (MCP protocol) and `GET /health` (health check).
- Default port: `8080` (configurable via `MCP_HTTP_PORT`).

## Scope

**2 tools:**

- `mspbots_user_list_tenants` — paginated + filterable, returns platform tenants (needs a platform-level credential).
- `mspbots_user_list_users` — paginated + filterable, returns the users of one tenant (the set of assignable owners).

## Authentication

This server requires, per request:
- A credential in `X-API-Key` — normally a tenant **API key** (`mbk_…`), minted on the platform's API Keys page. A user JWT also works for debugging.
- The Agent Platform host, supplied per-request since different environments may use different hosts.

There is **no tenant header**. The credential is already tenant-scoped: an API key belongs to the tenant that minted it, and a JWT carries its tenant. To read a different tenant, pass `mspbots_user_list_users`' `tenant_id` argument — which only a platform-level credential is allowed to use.

**How credentials are forwarded**: when `X-API-Key` starts with `mbk_`, the downstream receives `X-API-Key: <token>`; otherwise it is handled as a JWT and the downstream receives `Authorization: Bearer <token>`. This service does no exchange/validation logic.

### HEADER authentication parameters

| Header | Type | Required | Default | Allowed values | Description | Example |
|---|---|---|---|---|---|---|
| `X-API-Key` | string | Yes | None | None (free text) | Platform credential. A tenant API key (`mbk_` prefix) is recommended; a user JWT is also accepted (mainly for debugging). | `X-API-Key: mbk_xxx` |
| `X-MSP-Host` | string | Yes | None | None (free text, base URL) | Host where the Agent Platform runs. This service appends the internal App API path to get the full request URL. | `X-MSP-Host: <platform-host>` |

> ⏳ **Transition-period compatibility (since 2026-09-21)**: `X-API-Key` replaces the former `X-MSP-Token`. Tenant credentials written before the rename still exist in the registry under the old name and are injected as-is by the gateway, so the middleware still falls back to reading `X-MSP-Token` when `X-API-Key` is absent. When both are present, `X-API-Key` wins. Once every tenant's credentials have been re-saved under the new name, delete that fallback in `server.py` and `tests/test_middleware.py::test_legacy_token_header_is_still_accepted`.
>
> 🚫 `X-MSP-Tenant-Id` has been removed and is no longer read; it is ignored if sent. The default tenant of `mspbots_user_list_users`, which it used to provide, can now only be specified via the tool parameter `tenant_id`.

Missing a required header returns `401`:
```json
{
  "error": "Missing credentials",
  "message": "This server requires the X-API-Key header (Agent Platform API key, or a user JWT) and the X-MSP-Host header (Agent Platform host).",
  "required_headers": ["X-API-Key", "X-MSP-Host"]
}
```

## Environment Variables

| Variable | Type | Required | Default | Description |
|---|---|---|---|---|
| `MCP_HTTP_PORT` | int | No | `8080` | HTTP listen port |
| `MCP_HTTP_HOST` | string | No | `0.0.0.0` | HTTP listen address |

## MCP Endpoint

- `POST /mcp` — MCP protocol (streamable HTTP transport)
- `GET /health` — health check, returns `{"status": "ok", "service": "agent-tenant-user-mcp", "transport": "http"}`

## Tool List

| Tool | Function | Parameters |
|---|---|---|
| `mspbots_user_list_tenants` | Get the platform-wide tenant list with pagination; supports filtering by keyword/enabled status/registration time range (requires a platform-level credential) | page (optional, default 1), page_size (optional, default 20), search (optional), is_active (optional), created_from (optional), created_to (optional) |
| `mspbots_user_list_users` | Get the user list of a single tenant with pagination (can be assigned as owners; returns id/email/username/displayName/department/role, etc.) | page (optional, default 1), page_size (optional, default 20), tenant_id (optional), search (optional), department (optional), is_active (optional), sort_by (optional, default createdAt), sort_order (optional, default desc) |

`mspbots_user_list_tenants`: `search`/`is_active`/`created_from`/`created_to` combine as AND; results are always sorted by `createdAt` descending; an invalid date value in `created_from`/`created_to` is silently ignored (not an error).

`mspbots_user_list_users` always returns the users of **one tenant** only: without `tenant_id` it is the credential's own tenant; only a platform-level credential can use `tenant_id` to specify another tenant, and for other credentials it is ignored downstream. `sort_by` accepts `createdAt` / `email` / `displayName` / `department` / `lastLoginAt`. `page_size` is capped at 100; larger values are clamped to 100.

## Test Example

```bash
# Health check
curl -s http://localhost:8080/health

# Call a tool via the MCP protocol (streamable HTTP) — requires an
# initialize handshake first per the MCP spec; abbreviated example below
# shows the tool-call request body only:
curl -s -X POST http://localhost:8080/mcp \
  -H "X-API-Key: mbk_xxx" \
  -H "X-MSP-Host: <platform-host>" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "mcp-session-id: <session-id-from-initialize>" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "mspbots_user_list_users",
      "arguments": {"page": 1, "page_size": 20}
    }
  }'
```
