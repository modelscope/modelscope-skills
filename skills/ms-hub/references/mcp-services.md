# MCP Service Discovery and Configuration Reference

> Expanded reference for ms-hub MCP service operations: end-to-end discovery→deploy→configure-IDE orchestration, IDE config templates, SDK↔OpenAPI field differences, plus the operations quick reference at the end.
>
> MCP marketplace: https://www.modelscope.cn/mcp | Verified live 2026-06-29

## CLI / SDK / OpenAPI Comparison

| Operation | CLI | SDK (`MCPApi`) | OpenAPI |
|------|-----|----------------|---------|
| Search services | `ms mcp list --search "..."` | `list_mcp_servers()` | `PUT /mcp/servers` |
| Service details | `ms mcp info {id}` | `get_mcp_server()` | `GET /mcp/servers/{id}` |
| My services | — | `list_operational_mcp_servers()` | `GET /mcp/servers/operational` |
| Deploy service | `ms mcp deploy {id} --transport-type sse` | — | `POST /mcp/servers/{id}/deploy` |
| Undeploy service | `ms mcp undeploy {id}` | — | `DELETE /mcp/servers/{id}/undeploy` |

The SDK only supports search / details / deployed list; use the CLI or OpenAPI for deploy and undeploy.

```python
from modelscope.hub.mcp_api import MCPApi
mcp = MCPApi(); mcp.login(access_token="YOUR_TOKEN")
```

## Key Constraints of the Deploy Endpoint

The body of `POST /mcp/servers/{id}/deploy`:

| Field | Required | Description |
|------|------|------|
| `transport_type` | **Yes** | Valid values `sse` / `streamable_http`. **Omitting it returns HTTP 400 `invalid transport_type`** |
| `expiration_minutes` | No | Validity period (minutes) |
| `auth_check` | No | Whether to verify authentication |
| `env_info` | No | Environment variables in `{KEY: value}` form (some services require an API Key) |

- **The deployed transport determines the single returned URL**: `sse` → `.../sse`; `streamable_http` → `.../mcp`. The operational list returns only the transport you deployed, not both.
- `ms mcp deploy {id}` can omit it because the CLI defaults to `transport_type=sse`; for streamable use `--transport-type streamable_http`.
- **Pagination limit**: `page_number × page_size ≤ 100`, enforced server-side (exceeding it returns HTTP 403 `QuotaLimitExceed`), different from the 3000 for Hub/Skills.

```bash
# OpenAPI deploy (transport_type required)
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/mcp/servers/@amap/amap-maps/deploy" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" -H "Content-Type: application/json" \
  -d '{"transport_type": "streamable_http"}'
```

## SDK ↔ OpenAPI Field Differences

The SDK internally converts key names from the raw OpenAPI response:

| OpenAPI raw | SDK converted |
|--------------|-----------|
| `data.mcp_server_list` | `data.servers` |
| Per-entry `operational_urls: [{url, transport_type, accessible, ...}]` | `mcp_servers: [{type, url}]` |

Search response entries contain: `id`, `name`, `chinese_name`, `description`, `categories`, `tags`, `logo_url`, `is_hosted`, `env_schema`.

## End-to-End Orchestration: Configuring an MCP Tool for an Agent

```python
from modelscope.hub.mcp_api import MCPApi
import os, requests

TOKEN = "YOUR_TOKEN"
BASE = os.environ.get("MODELSCOPE_ENDPOINT", "https://modelscope.cn") + "/openapi/v1"
mcp = MCPApi(); mcp.login(access_token=TOKEN)

# 1. Search for a suitable service
for s in mcp.list_mcp_servers(search="weather")["servers"]:
    print(s["id"], s["description"])

# 2. View details (confirm is_hosted, whether env_schema needs an API Key)
detail = mcp.get_mcp_server(server_id="selected service ID")

# 3. Deploy (transport_type required)
requests.post(f"{BASE}/mcp/servers/{detail['id']}/deploy",
              headers={"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"},
              json={"transport_type": "streamable_http"})

# 4. Get the connection URL from operational
mcp_url = None
for s in mcp.list_operational_mcp_servers()["servers"]:
    if s["id"] == detail["id"]:
        for ep in s["mcp_servers"]:
            if ep["type"] == "streamable_http":
                mcp_url = ep["url"]

# 5. Write mcp_url into the IDE / Agent MCP config (see template below)
print("MCP URL:", mcp_url)
```

> The URL obtained from the operational API is the complete MCP endpoint; no additional concatenation is needed.

## IDE / Agent Config Templates

**Cursor** (`.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "service-name": {
      "type": "streamable_http",
      "url": "<deploy-URL>",
      "name": "service-name"
    }
  }
}
```

**Claude Code** (`.mcp.json`):

```json
{
  "mcpServers": {
    "service-name": { "type": "streamable_http", "url": "<deploy-URL>" }
  }
}
```

**VS Code (Copilot)** / **Codex** / generic Agent: following each client's docs, add a remote MCP service of type Streamable HTTP (or SSE), using the same URL obtained above. If the client only supports SSE, deploy with `transport_type=sse` instead and use the `.../sse` URL.

When unsure how to configure a client, print the URL and let the user add it manually.

## MCP Service Operations — Quick Reference

> Pagination limit `page_number × page_size ≤ 100` (server-enforced; exceeding it returns HTTP 403).
> Full orchestration, IDE configuration templates, SDK↔OpenAPI field differences → the sections above in this file.

### Search MCP Services

```bash
# CLI
ms mcp list --search "map" --page-size 20

# OpenAPI
curl -X PUT "$MODELSCOPE_ENDPOINT/openapi/v1/mcp/servers" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"search": "map", "page_size": 20}'
```

### View Service Details

```bash
# CLI
ms mcp info @amap/amap-maps

# OpenAPI
curl "$MODELSCOPE_ENDPOINT/openapi/v1/mcp/servers/@amap/amap-maps" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
```

### Deploy and Undeploy

When deploying, **`transport_type` is required**, with valid values `sse` / `streamable_http`; the deployed transport determines the unique URL returned (`sse`→`.../sse`, `streamable_http`→`.../mcp`).

```bash
# CLI (defaults to sse; the other option uses --transport-type streamable_http)
ms mcp deploy @amap/amap-maps
ms mcp undeploy @amap/amap-maps

# OpenAPI (must include transport_type, otherwise HTTP 400 invalid transport_type)
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/mcp/servers/@amap/amap-maps/deploy" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" -H "Content-Type: application/json" \
  -d '{"transport_type": "streamable_http"}'
curl -X DELETE "$MODELSCOPE_ENDPOINT/openapi/v1/mcp/servers/@amap/amap-maps/undeploy" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
```

### View My Deployed Services

```bash
curl "$MODELSCOPE_ENDPOINT/openapi/v1/mcp/servers/operational" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
```

### SDK Method

```python
from modelscope.hub.mcp_api import MCPApi

mcp = MCPApi()
mcp.login(access_token="YOUR_TOKEN")

# Search → {"total_count": N, "servers": [{"name", "id", "description"}, ...]}
result = mcp.list_mcp_servers(search="weather", total_count=20)
for s in result["servers"]:
    print(f"{s['id']}: {s['description']}")

# Details → {"name", "description", "id", "servers": [{"type", "url"}, ...]}
detail = mcp.get_mcp_server(server_id="@amap/amap-maps")

# Deployed → {"total_count": N, "servers": [{"name", "id", "mcp_servers": [{"type", "url"}]}, ...]}
operational = mcp.list_operational_mcp_servers()
```

> `modelscope.hub.mcp_api.MCPApi` supports only search, details, and the deployed list. Use the CLI `ms mcp deploy/undeploy` or OpenAPI for deploy/undeploy.
