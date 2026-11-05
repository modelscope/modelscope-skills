# Studio Operations Reference

> ms-hub studio operations (create/configure/deploy/logs/variables/code sync).
> Read on demand via the skill file list in SKILL.md.

## Studio Operations

> For the complete deployment workflow (including code sync, diagnosis and repair) → ms-studio-deploy (**OpenAPI is the source of truth**, CLI is an equivalent alias)
>
> The CLI is driven by modelscope_hub and is a 1:1 thin wrapper around OpenAPI; the API-first Python entrypoint is `from modelscope_hub import HubApi`.
> If the agent already has the studio-mcp tools configured, you can also use the MCP tools (`createStudio`, `deployStudio`, etc.).

### Create Studio

```bash
# CLI
ms create USERNAME/my-app --repo-type studio --sdk-type gradio --private

# OpenAPI
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/studios" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"owner": "USERNAME", "repo_name": "my-app", "sdk_type": "gradio", "visibility": "private"}'
```

| sdk_type | Use case |
|----------|----------|
| `gradio` | Gradio app (entrypoint app.py) |
| `streamlit` | Streamlit app |
| `docker` | Custom Docker (port must be 7860) |
| `static` | Pure static website (already built) |

### Query Available Configs

```bash
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/hardware?sdk_type=gradio" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/sdk-versions?sdk_type=gradio" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/base-images" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
```

When a Studio already exists, you can append `&studio=USERNAME/my-app` to the hardware query. `hardware` uses the `name` of the returned items; the paid-resource format is `paid/<InstanceType>`. Gradio `sdk_version` uses the `version` of the returned items, and `base_image` uses the `name` of the returned items.

**Paid-resource authorization requirement:** Using `paid/<InstanceType>` or a returned item with `resource_type=paid` incurs charges on the Alibaba Cloud account bound to the user's ModelScope; you must first clearly inform the user and obtain their explicit authorization before creating, updating settings, or redeploying.

### Deploy/Restart

```bash
# CLI
ms deploy USERNAME/my-app --repo-type studio

# OpenAPI
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/studios/USERNAME/my-app/deploy" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
```

### View Status and Logs

```bash
# CLI
ms logs USERNAME/my-app --log-type run
ms logs USERNAME/my-app --log-type build  # Docker type

# OpenAPI
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/USERNAME/my-app" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/USERNAME/my-app/logs/run" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
```

### Stop

```bash
# CLI
ms stop USERNAME/my-app --repo-type studio

# OpenAPI
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/studios/USERNAME/my-app/stop" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
```

### Update Settings

```bash
# CLI
ms settings USERNAME/my-app --repo-type studio display_name="New name" private=false

# OpenAPI
curl -X PATCH "$MODELSCOPE_ENDPOINT/openapi/v1/studios/USERNAME/my-app/settings" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"display_name": "New name", "visibility": "public", "sdk_type": "gradio"}'
```

Updatable fields: `display_name`, `description`, `visibility`, `sdk_type`, `sdk_version`, `base_image`, `hardware`, `license`. `private` is deprecated; OpenAPI prefers `visibility`.

### Variable Management

Plaintext variables return both key and value, and are used only for non-sensitive configuration:

```bash
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/USERNAME/my-app/variables" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/studios/USERNAME/my-app/variables" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"key": "GRADIO_TEMP_DIR", "value": "/tmp/gradio"}'
curl -X PUT "$MODELSCOPE_ENDPOINT/openapi/v1/studios/USERNAME/my-app/variables" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"key": "GRADIO_TEMP_DIR", "value": "/mnt/workspace/tmp"}'
curl -X DELETE "$MODELSCOPE_ENDPOINT/openapi/v1/studios/USERNAME/my-app/variables" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"key": "GRADIO_TEMP_DIR"}'
```

Secrets return only the key, not the value, and are used for sensitive information such as API keys, tokens, and passwords:

```bash
# CLI
ms secret list USERNAME/my-app
ms secret add USERNAME/my-app API_KEY sk-xxx
ms secret update USERNAME/my-app API_KEY new-value
ms secret delete USERNAME/my-app API_KEY

# OpenAPI
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/USERNAME/my-app/secrets" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/studios/USERNAME/my-app/secrets" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"key": "API_KEY", "value": "sk-xxx"}'
curl -X PUT "$MODELSCOPE_ENDPOINT/openapi/v1/studios/USERNAME/my-app/secrets" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"key": "API_KEY", "value": "new-value"}'
# Delete: put the key in the body, not as a path parameter (DELETE .../{key} returns 404)
curl -X DELETE "$MODELSCOPE_ENDPOINT/openapi/v1/studios/USERNAME/my-app/secrets" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"key": "API_KEY"}'
```

### Code Sync

```bash
git remote add modelscope https://oauth2:${MODELSCOPE_API_KEY}@www.modelscope.cn/studios/${owner}/${repo}.git
git push -u modelscope master
```
