# Studios OpenAPI Endpoint Reference

Base URL: `$MODELSCOPE_ENDPOINT/openapi/v1`

Authentication: `Authorization: Bearer $MODELSCOPE_API_KEY`

> Field-level spec cross-checked against the modelscope_service master branch (2026-10-27) and the `feat/openAPI_studios_ouath` branch (Studio OAuth app configuration; `oauth_*` fields are ignored by master deployments that predate the feature).

## List Studios

```bash
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios?search=my-app&status=running&page_number=1&page_size=20" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
```

Query parameters:

| Parameter | Description |
|------|------|
| `search` | Keyword, max 100 chars |
| `author` / `owner` | Filter by author/owner |
| `status` | `running` (default) or `all` |
| `mcp_support` | `true`/`false` — studios usable via MCP |
| `hardware_type` | Hardware filter, currently only `xGPU` |
| `sort` | `default`, `view_count`, `likes`, `last_modified` |
| `page_number` / `page_size` | Pagination (`page_number × page_size ≤ 3000`) |

Response `data`: `studios[]` (same shape as Get Studio details), `total_count`, `page_number`, `page_size`.

## Create a Studio

```bash
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/studios" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "owner": "username",
    "repo_name": "my-app",
    "sdk_type": "gradio",
    "display_name": "My Application",
    "description": "Application description",
    "visibility": "private",
    "hardware": "platform/2v-cpu-16g-mem",
    "base_image": "ubuntu22.04-py311-torch2.9.1-modelscope1.35.0",
    "sdk_version": "6.2.0"
  }'
```

Request fields:

| Field | Required | Notes |
|------|------|------|
| `owner` | Yes | Username or organization (org membership with write/admin role required) |
| `repo_name` | Yes | ≤ 64 chars; alphanumeric, hyphens, underscores, dots; not a reserved word |
| `sdk_type` | No | Defaults to `gradio` |
| `visibility` | No | `public` / `protected` / `private` |
| `private` | No | Deprecated — use `visibility` |
| `display_name` | No | ≤ 128 chars |
| `description` | No | ≤ 2000 chars |
| `cover_image` | No | Cover image URL; platform default when empty |
| `license` | No | Defaults to `apache-2.0` |
| `hardware` | No | Free: `platform/…`/`xgpu/…` prefixes; paid: `paid/<InstanceType>`. May be omitted — the server normalizes to the current default free instance tier |
| `base_image` | No | Defaults to the latest |
| `sdk_version` | No | Defaults to the latest for `gradio`; validated against the available versions |
| `oauth_enabled` | No | Enable the Studio's OAuth app configuration (defaults to `false`); requires `oauth_scope`, `oauth_token_expire_time`, `oauth_callback_urls` when `true` |
| `oauth_scope` | No | Space-separated OAuth scopes; valid values: `openid` (ModelScope user identity), `profile` (public user profile), `manage-mcp-deployment` (manage MCP deployments), `list-operational-mcp` (list deployed MCP servers), `read-repos` (read personal repositories), `api-inference` (call API Inference) |
| `oauth_token_expire_time` | No | Token lifetime enum: `1`=15min, `2`=1hour, `3`=8hours, `4`=1day, `5`=1week, `6`=30days |
| `oauth_callback_urls` | No | Array of OAuth callback URLs; each entry needs a scheme and a host |
| `oauth_homepage_url` | No | OAuth app homepage URL |

### sdk_type values

| Value | Description | Entry file |
|-----|------|----------|
| `gradio` | Gradio app | `app.py` |
| `streamlit` | Streamlit app | `app.py` |
| `docker` | Docker container | `Dockerfile` |
| `static` | Static website | `index.html` |

## Query available configuration

These endpoints are used to dynamically select configuration before creating or updating a Studio, avoiding hardcoding stale values.

```bash
# Hardware configuration; sdk_type is optional, and for an existing Studio you can append studio=owner/repo_name
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/hardware?sdk_type=gradio" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"

# SDK versions; currently only sdk_type=gradio returns a Gradio version list
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/sdk-versions?sdk_type=gradio" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"

# Base images
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/base-images" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
```

When using the returned values: `hardware` takes the hardware item's `name` (the paid-resource format is `paid/<InstanceType>`), `sdk_version` takes the SDK version item's `version`, and `base_image` takes the base image item's `name`.

Hardware items include: `name`, `resource_type` (`free`/`paid`), `supported_sdk_types[]`; paid items additionally carry `instance_type`, `cpu`, `gpu`, `memory`, `gpu_type`, `gpu_memory`, `has_stock`, `stock`, `cost_after_discount`, `original_cost`.

**Paid-resource authorization requirement:** If `hardware` uses `paid/<InstanceType>` or the returned hardware item has `resource_type=paid`, charges will be incurred against the Alibaba Cloud account bound to the user's ModelScope account. You must first explain the cost risk to the user and obtain explicit authorization before you can create, update settings, or redeploy.

## Get Studio details

```bash
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/{owner}/{repo_name}" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
```

Response `data` fields: `id` (`owner/repo_name`), `repo_name`, `display_name`, `owner`, `description`, `cover_image`, `likes`, `view_count`, `tags[]`, `visibility` (`public`/`protected`/`private`), `private` (deprecated), `status`, `created_at`, `last_modified`, `sdk_type`, `sdk_version`, `hardware` (desired config), `base_image` (omitted for Docker), `license`, `host`, `mcp_support`, `runtime` (`status`, `active_config` {`hardware`, `base_image`, `sdk_type`, `sdk_version`} — the config actually running, `created_at`, `error_message` on failure), and when the OAuth app is configured: `oauth_enabled`, `oauth_scope`, `oauth_token_expire_time`, `oauth_callback_urls`, `oauth_homepage_url`.

## Deploy/restart a Studio

```bash
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/studios/{owner}/{repo_name}/deploy" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
```

No request body. Response `data` is the runtime snapshot: `status`, `active_config`, `created_at`, `error_message` (failure states only).

## Stop a Studio

```bash
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/studios/{owner}/{repo_name}/stop" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
```

No request body; response shape is the same as deploy.

## Get logs

`log_type` only accepts `run` or `build`.

```bash
# Run logs (supports pagination/filter query params)
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/{owner}/{repo_name}/logs/run?page_num=1&page_size=100&keyword=Error" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"

# Build logs (Docker type)
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/{owner}/{repo_name}/logs/build" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
```

Query parameters:

| Parameter | Default | Notes |
|------|------|------|
| `page_num` | 1 | Page number |
| `page_size` | 100 | Capped at 500 |
| `keyword` | — | Substring filter |
| `start_timestamp` / `end_timestamp` | last 30 days | Unix seconds |

Response `data`: `logs[]`, `page_num`, `page_size`, `total_count`, `total_page_num`.

## Update settings

```bash
curl -X PATCH "$MODELSCOPE_ENDPOINT/openapi/v1/studios/{owner}/{repo_name}/settings" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "sdk_type": "gradio",
    "visibility": "public",
    "sdk_version": "6.2.0",
    "base_image": "ubuntu22.04-py311-torch2.9.1-modelscope1.35.0",
    "hardware": "platform/2v-cpu-16g-mem"
  }'
```

Send only the fields you want to change; omitted fields keep their original values. Updatable: `display_name` (≤ 128), `description` (≤ 2000), `cover_image`, `license`, `visibility` (`public`/`protected`/`private`), `private` (deprecated), `sdk_type`, `sdk_version`, `base_image`, `hardware`, `oauth_enabled`, `oauth_scope`, `oauth_token_expire_time`, `oauth_callback_urls`, `oauth_homepage_url`.

The `oauth_*` group is tri-state: the server applies it **only when `oauth_enabled` is present in the payload** — pass `true` to enable/reconfigure, `false` to disable (the other `oauth_*` fields are then dropped together). Omit `oauth_enabled` entirely and the other `oauth_*` fields are ignored, so always send the flag alongside them.

```bash
# Enable / reconfigure the OAuth app
curl -X PATCH "$MODELSCOPE_ENDPOINT/openapi/v1/studios/{owner}/{repo_name}/settings" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "oauth_enabled": true,
    "oauth_scope": "openid profile read-repos",
    "oauth_token_expire_time": 4,
    "oauth_callback_urls": ["https://example.org/oauth/callback"],
    "oauth_homepage_url": "https://example.org"
  }'

# Disable it
curl -X PATCH "$MODELSCOPE_ENDPOINT/openapi/v1/studios/{owner}/{repo_name}/settings" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"oauth_enabled": false}'
```

Valid `oauth_scope` values (space-separated, multiple allowed): `openid`, `profile`, `manage-mcp-deployment`, `list-operational-mcp`, `read-repos`, `api-inference`. `oauth_token_expire_time` uses the enum `1`=15min … `6`=30days. `oauth_callback_urls` entries need a scheme and a host.

Changes to `sdk_type`, `sdk_version`, `base_image`, and `hardware` require a redeploy to take effect.

## Plaintext variables (Variables)

Plaintext variables return key, value, and an optional `description`; use them only for non-sensitive configuration. Use secrets for sensitive information.

```bash
# List (returns key, value, description)
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/{owner}/{repo_name}/variables" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"

# Add (description optional, max 512 chars)
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/studios/{owner}/{repo_name}/variables" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" -H "Content-Type: application/json" \
  -d '{"key": "GRADIO_TEMP_DIR", "value": "/tmp/gradio", "description": "Gradio temp dir"}'

# Update (description: omit to keep the original, pass "" to clear)
curl -X PUT "$MODELSCOPE_ENDPOINT/openapi/v1/studios/{owner}/{repo_name}/variables" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" -H "Content-Type: application/json" \
  -d '{"key": "GRADIO_TEMP_DIR", "value": "/mnt/workspace/tmp"}'

# Delete (put key in the body, not as a path parameter)
curl -X DELETE "$MODELSCOPE_ENDPOINT/openapi/v1/studios/{owner}/{repo_name}/variables" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" -H "Content-Type: application/json" \
  -d '{"key": "GRADIO_TEMP_DIR"}'
```

Limits: `key` ≤ 128 chars, `value` ≤ 4096 chars, `description` ≤ 512 chars.

## Secrets (Secrets)

The secrets list returns keys and descriptions (not values) and is used for sensitive information such as API keys, tokens, and passwords.

```bash
# List (returns key and description)
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/{owner}/{repo_name}/secrets" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"

# Add (description optional, max 512 chars)
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/studios/{owner}/{repo_name}/secrets" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" -H "Content-Type: application/json" \
  -d '{"key": "API_KEY", "value": "sk-xxx", "description": "Upstream API key"}'

# Update (description: omit to keep the original, pass "" to clear)
curl -X PUT "$MODELSCOPE_ENDPOINT/openapi/v1/studios/{owner}/{repo_name}/secrets" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" -H "Content-Type: application/json" \
  -d '{"key": "API_KEY", "value": "new-value"}'

# Delete (put key in the body, not as a path parameter)
curl -X DELETE "$MODELSCOPE_ENDPOINT/openapi/v1/studios/{owner}/{repo_name}/secrets" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" -H "Content-Type: application/json" \
  -d '{"key": "API_KEY"}'
```

Limits: `key` ≤ 128 chars, `value` ≤ 4096 chars, `description` ≤ 512 chars.

> ⚠️ Deleting either plaintext variables or secrets uses `DELETE .../variables` or `DELETE .../secrets` + body `{"key": "..."}`.
> The path form `DELETE .../{key}` returns 404 and is a no-op.
