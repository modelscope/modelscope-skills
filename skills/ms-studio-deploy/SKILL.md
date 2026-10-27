---
name: ms-studio-deploy
description: >-
  Deploy a local project to a ModelScope Studio. Supports Gradio, Streamlit, Docker, and static website types.
  Covers creation, code sync, deployment, log monitoring, plaintext/secret variable management, and automatic diagnosis and repair.
  Use when the user mentions deploying to ModelScope, Studio, the ModelScope community, ModelScope, Studio deployment, Gradio deployment, Streamlit deployment,
  Docker deployment, FastAPI deployment, or static website deployment, or wants to publish a local app, web app, or API service to the cloud.
  Also applies when the user encounters a Studio build failure or runtime error and needs to inspect logs, wants to update an already-deployed Studio, or wants to manage a Studio's plaintext variables or secrets.
  Not applicable to: Hub repository management, model/dataset operations, MCP/skill management, model training, or model evaluation.
---

# ModelScope Studio Deployment

> Cross-checked against ModelScope OpenAPI (modelscope_service master 2026-10-27 + `feat/openAPI_studios_ouath` branch for Studio OAuth app configuration); previously live-verified with modelscope 1.37.1 / modelscope_hub 0.1.2 (2026-06-29)

Deploy a local project to a ModelScope Studio. **This Skill uses OpenAPI as the source of truth**, with the `ms` CLI as an equivalent convenience alias, `modelscope_hub.HubApi` as the API-first Python client, and MCP tools as optional.

## Quick Decision Guide

```
User wants to...
│
├── Deploy a local project to a Studio
│   └── Follow the 10 steps in "Full Deployment Workflow" below
│
├── Update an already-deployed Studio
│   └── Modify code → git push → POST /studios/{o}/{r}/deploy (or ms deploy)
│
├── List / browse Studios
│   └── GET /studios?search=...&status=running|all&owner=...&mcp_support=&hardware_type=xGPU
│
├── View Studio status/logs
│   └── GET /studios/{o}/{r}/logs/run (or ms logs --log-type run)
│
├── Manage variables
│   ├── Plaintext variables: GET/POST/PUT/DELETE /studios/{o}/{r}/variables
│   └── Secrets: GET/POST/PUT/DELETE /studios/{o}/{r}/secrets (or ms secret ...)
│
└── Repository file management / model & dataset operations
    └── hand off → ms-hub
```

## Prerequisites

### 1. MODELSCOPE_API_KEY

The entire workflow depends on this token (API authentication, Git push, variable/secret configuration); it must be confirmed first.

```bash
# 1. First check the environment variable
echo $MODELSCOPE_API_KEY

# 2. If empty, try extracting from the git remote (may have been configured before)
git remote -v 2>/dev/null | grep modelscope.cn
```

If the environment variable is empty but the git remote contains an address of the form `https://oauth2:<token>@www.modelscope.cn/studios/...` (or `www.modelscope.ai/...` for the international site), extract `<token>` from it as `MODELSCOPE_API_KEY`.
If neither is available, guide the user:

1. Visit $MODELSCOPE_ENDPOINT/my/myaccesstoken to obtain a token
2. `export MODELSCOPE_API_KEY=your_token`

> **Site routing:** this skill targets whichever site `$MODELSCOPE_ENDPOINT` points to (default domestic `https://modelscope.cn`; international `https://www.modelscope.ai`). Export `MODELSCOPE_ENDPOINT` plus a **site-scoped** token before deploying — the OpenAPI base and the git remote host below both derive from it. Full guidance: ms-hub → "Site selection & endpoint routing".

### 2. Runtime Environment

```bash
pip install modelscope          # Also provides the OpenAPI/SDK and the ms CLI
```

OpenAPI basic conventions:

| Item | Value |
|------|-----|
| **Base URL** | `$MODELSCOPE_ENDPOINT/openapi/v1` (default `https://modelscope.cn`) |
| **Authentication** | `Authorization: Bearer $MODELSCOPE_API_KEY` |
| **Success response** | `{"success": true, "data": {...}, "request_id": "..."}` |
| **Default branch** | `master` (not main) |

### 3. Git Environment

Studio code is synced via Git; ensure Git is installed and user information is configured.

## Operations Overview: OpenAPI (source of truth) ↔ CLI ↔ Python

| Operation | OpenAPI (source of truth) | ms CLI (equivalent alias) | `modelscope_hub.HubApi` (Python) | MCP tool (optional) |
|------|---------------------|---------------------|-----------------------------------|------------------|
| User info | `GET /users/me` | `ms whoami` | `api.whoami()` | `getCurrentUser` |
| List Studios | `GET /studios` (query: `search`, `status`, `owner`, `author`, `mcp_support`, `hardware_type`, `sort`, `page_number`, `page_size`) | none | none | none |
| Create Studio | `POST /studios` | `ms create o/r --repo-type studio` | `api.create_repo("o/r", repo_type="studio", ...)` | `createStudio` |
| Get details | `GET /studios/{o}/{r}` | `ms info o/r --repo-type studio` | `api.get_repo("o/r", repo_type="studio")` | `getStudio` |
| Deploy/restart | `POST /studios/{o}/{r}/deploy` | `ms deploy o/r --repo-type studio` | `api.deploy_repo("o/r")` | `deployStudio` |
| Stop | `POST /studios/{o}/{r}/stop` | `ms stop o/r --repo-type studio` | `api.stop_repo("o/r")` | `stopStudio` |
| Logs | `GET /studios/{o}/{r}/logs/{run\|build}` | `ms logs o/r --log-type run` | `api.get_repo_logs("o/r", log_type="run")` | `getStudioLogs` |
| Update settings | `PATCH /studios/{o}/{r}/settings` | `ms settings o/r --repo-type studio k=v` | `api.update_repo_settings("o/r","studio",**kw)` | `updateStudioSettings` |
| Query available hardware | `GET /studios/hardware?sdk_type=gradio[&studio=o/r]` | none | none | `listHardware` |
| Query SDK versions | `GET /studios/sdk-versions?sdk_type=gradio` | none | none | `listSdkVersions` |
| Query base images | `GET /studios/base-images` | none | none | `listBaseImages` |
| List plaintext variables | `GET /studios/{o}/{r}/variables` | none | none | `listStudioVariables` |
| Add plaintext variable | `POST /studios/{o}/{r}/variables` | none | none | `addStudioVariable` |
| Update plaintext variable | `PUT /studios/{o}/{r}/variables` | none | none | `updateStudioVariable` |
| Delete plaintext variable | `DELETE /studios/{o}/{r}/variables` | none | none | `deleteStudioVariable` |
| List secrets | `GET /studios/{o}/{r}/secrets` | `ms secret list o/r` | `api.list_secrets("o/r")` | `listStudioSecrets` |
| Add secret | `POST /studios/{o}/{r}/secrets` | `ms secret add o/r K V` | `api.add_secret("o/r","K","V")` | `addStudioSecret` |
| Update secret | `PUT /studios/{o}/{r}/secrets` | `ms secret update o/r K V` | `api.update_secret("o/r","K","V")` | `updateStudioSecret` |
| Delete secret | `DELETE /studios/{o}/{r}/secrets` | `ms secret delete o/r K` | `api.delete_secret("o/r","K")` | `deleteStudioSecret` |

> **`HubApi` is the same engine that drives the `ms` CLI** and is the API-first Python entry point:
> ```python
> from modelscope_hub import HubApi
> api = HubApi(); api.login("$MODELSCOPE_API_KEY")
> ```
> Note it is a different pair of classes from the legacy `modelscope.hub.api.HubApi`: the legacy version has not yet wired up Studio methods such as deploy/stop, whereas **the new `modelscope_hub.HubApi` has** (its `deploy_repo`/`stop_repo`/`get_repo_logs`/secret series default to `repo_type="studio"`).

> **MCP tools (optional)** — the last column of the table above lists the equivalent tools provided by the `studio-mcp` service, with the same semantics as the OpenAPI call in the same row; when the agent has this service configured it can be called directly, and when it is not configured this column can be ignored. See the appendix at the end for first-time setup.

## Full Deployment Workflow

> Each step leads with OpenAPI and provides an equivalent one-line CLI command. `${owner}`/`${repo}` are the Studio's owner and name.

### Step 1: Check the local Git repository

```bash
[ -d .git ] && git remote -v 2>/dev/null | grep -E 'modelscope\.(cn|ai)/studios'
```

- Already has a Studio remote address → extract `owner` and `repo` from the URL, skip to Step 3
- None → continue to Step 2

### Step 2: Analyze the project and get user info

#### 2.1 Determine the SDK type

| Type | Detection condition | Entry file | Notes |
|------|----------|----------|------|
| `gradio` | `app.py` imports gradio | `app.py` | Query `sdk_version`, `base_image`, `hardware` as needed |
| `streamlit` | `app.py` uses streamlit | `app.py` | Query `base_image`, `hardware` as needed |
| `docker` | `Dockerfile` present | `Dockerfile` | Port must be 7860 |
| `static` | `index.html` present (pre-built) | `index.html` | Build step not supported; no hardware selection |

Selection advice: `static` does not support a build step and the files must already be built; use `docker` for frontend projects that need building; use `docker` when unsure.

> The `docker` type requires first completing Alibaba Cloud account binding and passing real-name verification on the ModelScope platform: https://modelscope.cn/docs/studios/docker , otherwise the image cannot be built.

#### 2.2 Get user info

```bash
curl "$MODELSCOPE_ENDPOINT/openapi/v1/users/me" -H "Authorization: Bearer $MODELSCOPE_API_KEY"
# Equivalent CLI: ms whoami
```

The `repo` is taken from the project directory name or specified by the user.

### Step 3: Create or update the Studio

Before creating or changing settings, query the available options to avoid hardcoding outdated configuration:

```bash
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/hardware?sdk_type=${sdk_type}" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/sdk-versions?sdk_type=gradio" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/base-images" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
```

For an existing Studio, you can append `&studio=${owner}/${repo}` to the hardware query so that free resources are returned according to that Studio's available quota. When selecting `hardware`, use the returned item's `name`; the paid resource format is `paid/<InstanceType>`. When selecting a Gradio `sdk_version`, use the returned item's `version`. When selecting `base_image`, use the returned item's `name`.

**Paid-resource authorization requirement:** If you intend to set `hardware` to `paid/<InstanceType>` or the returned item has `resource_type=paid`, you must first explicitly tell the user that this will incur charges on the Alibaba Cloud account bound to their ModelScope account, and only create, update settings, or redeploy after obtaining the user's explicit authorization. Without authorization, only free resources may be selected.

Check whether it already exists:

```bash
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/${owner}/${repo}" -H "Authorization: Bearer $MODELSCOPE_API_KEY"
# Equivalent CLI: ms info ${owner}/${repo} --repo-type studio
```

**Does not exist → create** (before creating, proactively ask the user "public or private?", defaulting to private):

```bash
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/studios" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "owner": "'"${owner}"'",
    "repo_name": "'"${repo}"'",
    "sdk_type": "gradio",
    "visibility": "private",
    "hardware": "platform/2v-cpu-16g-mem",
    "display_name": "My App"
  }'
# Equivalent CLI: ms create ${owner}/${repo} --repo-type studio --sdk-type gradio --private
```

Create request fields: `owner`/`repo_name` (required), `sdk_type` (defaults to `gradio`), `visibility` (`public`/`protected`/`private`), `display_name`, `description`, `cover_image`, `license` (defaults to `apache-2.0`), `hardware`, `base_image`, `sdk_version`. `private` (boolean) is deprecated — use `visibility`. `hardware` may be omitted: the server then normalizes to the current default free instance tier (free tiers use `platform/…`/`xgpu/…` prefixes; paid uses `paid/<InstanceType>`). OAuth app fields (`oauth_enabled`, `oauth_scope`, `oauth_token_expire_time`, `oauth_callback_urls`, `oauth_homepage_url`) may also be set at creation — see the endpoint reference for the tri-state group semantics.

**Already exists → update settings as needed:**

```bash
curl -X PATCH "$MODELSCOPE_ENDPOINT/openapi/v1/studios/${owner}/${repo}/settings" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" -H "Content-Type: application/json" \
  -d '{"sdk_type": "gradio", "sdk_version": "6.2.0", "base_image": "ubuntu22.04-py311-torch2.9.1-modelscope1.35.0"}'
# Equivalent CLI: ms settings ${owner}/${repo} --repo-type studio sdk_type=gradio sdk_version=6.2.0
```

Changes to `sdk_type`, `sdk_version`, `base_image`, and `hardware` require redeployment to take effect. xGPU requires an application (https://modelscope.cn/docs/studios/xGPU).

Settings also accepts the OAuth app group `oauth_enabled` / `oauth_scope` / `oauth_token_expire_time` / `oauth_callback_urls` / `oauth_homepage_url` — applied only when `oauth_enabled` is present (`true` = enable/reconfigure, `false` = disable). Valid scopes: `openid`, `profile`, `manage-mcp-deployment`, `list-operational-mcp`, `read-repos`, `api-inference`. Equivalent CLI: `ms settings o/r --repo-type studio --oauth-enabled --oauth-scope "openid profile" --oauth-token-expire-time 4 --oauth-callback-url https://example.org/cb`. See `references/openapi-studio-endpoints.md` for the full contract.

### Step 4: Handle sensitive information

Before pushing code, scan files for hardcoded sensitive information (API keys, tokens, passwords, etc.) and change them to read from environment variables instead:

```python
# ❌ WRONG
api_key = "sk-xxxxxxxxxxxx"
# ✅ CORRECT
import os
api_key = os.environ.get("API_KEY")
```

Compile a list of variables for use in Step 6: put non-sensitive configuration in plaintext variables, and sensitive information such as API keys, tokens, and passwords in secrets.

### Step 5: Sync code to the Studio ⚠️ The only non-API step

Code sync is done via **Git**; there is no corresponding OpenAPI/CLI endpoint. The default branch is `master`; force push is prohibited.

```bash
# 1. Configure the remote repository (skip init if .git exists, skip add if the modelscope remote exists)
[ -d .git ] || git init
git remote remove modelscope 2>/dev/null || true
MS_HOST="${MODELSCOPE_ENDPOINT:-https://modelscope.cn}"; MS_HOST="${MS_HOST#https://}"   # endpoint host (defaults to modelscope.cn)
git remote add modelscope https://oauth2:${MODELSCOPE_API_KEY}@${MS_HOST}/studios/${owner}/${repo}.git

# 2. Large file handling (files over 100MB must use LFS)
git lfs install

# 3. Fetch the remote and merge
git fetch modelscope master
git merge modelscope/master --allow-unrelated-histories -m "Merge remote"
```

On merge conflicts, keep the local version:

```bash
git checkout --ours . && git add . && git commit -m "Resolve conflicts, keep local version"
```

Commit and push:

```bash
git add . && git commit -m "Deploy to ModelScope Studio"
git push -u modelscope master
```

### Step 6: Configure plaintext variables/secrets

Configure based on the list from Step 4 (API-first). Plaintext variables return both key and value (plus an optional `description`) and are only for non-sensitive configuration:

```bash
# List plaintext variables
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/${owner}/${repo}/variables" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
# Add (description is optional, max 512 chars)
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/studios/${owner}/${repo}/variables" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" -H "Content-Type: application/json" \
  -d '{"key": "GRADIO_TEMP_DIR", "value": "/tmp/gradio", "description": "Gradio temp dir"}'
# Update (description: omit to keep, pass "" to clear)
curl -X PUT "$MODELSCOPE_ENDPOINT/openapi/v1/studios/${owner}/${repo}/variables" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" -H "Content-Type: application/json" \
  -d '{"key": "GRADIO_TEMP_DIR", "value": "/mnt/workspace/tmp"}'
# Delete
curl -X DELETE "$MODELSCOPE_ENDPOINT/openapi/v1/studios/${owner}/${repo}/variables" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" -H "Content-Type: application/json" \
  -d '{"key": "GRADIO_TEMP_DIR"}'
```

Secrets do not return the value (the list returns key + optional `description`) and are used for sensitive information such as API keys, tokens, and passwords:

```bash
# List secrets (returns key and description only)
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/${owner}/${repo}/secrets" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
# Add (description is optional, max 512 chars)
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/studios/${owner}/${repo}/secrets" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" -H "Content-Type: application/json" \
  -d '{"key": "API_KEY", "value": "sk-xxx", "description": "Upstream API key"}'
# Update (description: omit to keep, pass "" to clear)
curl -X PUT "$MODELSCOPE_ENDPOINT/openapi/v1/studios/${owner}/${repo}/secrets" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" -H "Content-Type: application/json" \
  -d '{"key": "API_KEY", "value": "new-value"}'
# Delete
curl -X DELETE "$MODELSCOPE_ENDPOINT/openapi/v1/studios/${owner}/${repo}/secrets" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" -H "Content-Type: application/json" \
  -d '{"key": "API_KEY"}'
# Equivalent CLI (secrets only): ms secret {list,add,update,delete} ${owner}/${repo}
```

> ⚠️ To delete a plaintext variable or a secret, use `DELETE .../variables` or `DELETE .../secrets` and pass `{"key":"..."}` in the **body**, not the `DELETE .../{key}` path form. Limits: `key` ≤ 128 chars, `value` ≤ 4096 chars, `description` ≤ 512 chars.

### Step 7: Deploy the Studio

```bash
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/studios/${owner}/${repo}/deploy" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
# Equivalent CLI: ms deploy ${owner}/${repo} --repo-type studio
```

No request body. The response `data` is the runtime snapshot: `status`, `active_config` (the config actually running: `hardware`, `base_image`, `sdk_type`, `sdk_version`), `created_at`, `error_message` (failure states only). Stop (`POST .../stop`) returns the same shape.

### Step 8: Monitor status and logs

```bash
# Run logs (supports pagination/filter query params)
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/${owner}/${repo}/logs/run?page_num=1&page_size=100&keyword=Error" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
# Build logs (Docker type)
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/${owner}/${repo}/logs/build" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
# Equivalent CLI: ms logs ${owner}/${repo} --log-type run|build
```

Log query params: `page_num` (default 1), `page_size` (default 100, capped at 500), `keyword` (substring filter), `start_timestamp`/`end_timestamp` (Unix seconds; default window is the last 30 days). The response contains `logs[]`, `page_num`, `page_size`, `total_count`, `total_page_num`. `log_type` only accepts `run` or `build`.

View logs by SDK type:

- `docker`: check `build` first, then check `run` after the build completes
- Other types: check `run` directly

Keep alternating between them until `Running` or an error is found.

### Step 9: Automatic diagnosis and repair

**Common errors:**

| Error signature | Fix |
|----------|----------|
| `ModuleNotFoundError` | Add to requirements.txt |
| `SyntaxError` | Fix the code syntax |
| `MemoryError` | Suggest optimizing memory usage or upgrading the hardware configuration; if switching to a paid resource is needed, first obtain explicit authorization per Step 3 |
| `Permission denied` | Check file permissions |
| Empty variable | Check Step 6 |

**Docker-specific errors:**

| Error signature | Fix |
|----------|----------|
| Port issue / `Address already in use` | Ensure it listens on `0.0.0.0:7860`, not 8080 |
| `COPY failed` | Check the Dockerfile file paths |
| `RUN` step fails | Check the dependency installation commands |
| Image pull failure | Check the FROM base image address |

Repair flow: analyze logs → modify code → `git push` → `POST .../deploy` (or `ms deploy`) → check logs again

### Step 10: Complete the deployment

Provide the Studio URL: `$MODELSCOPE_ENDPOINT/studios/${owner}/${repo}`

> If the Studio is private, prompt the user: "Deployment succeeded and it is currently private. Would you like to make it public?" After confirmation, `PATCH .../settings` with `{"visibility": "public"}` (or `ms settings ${owner}/${repo} --repo-type studio private=false`).

## Docker Studio Reference

For applications that go beyond the scope of Gradio/Streamlit, such as FastAPI, Golang, and Node.js.
Prerequisite: you must complete Alibaba Cloud account binding and pass real-name verification on the ModelScope platform; see https://modelscope.cn/docs/studios/docker for details. See `references/docker-templates.md` for details.

**Key requirements:**

- The port must expose `0.0.0.0:7860`; using `8080` is prohibited (occupied by the platform)
- HTTP headers must not use `Authorization`, `X-modelscope-*`, or `X-studio-*`

## Data Persistence

- By default, data is lost on every restart
- Persistence directory: `/mnt/workspace`
- Data is still lost when transferring/renaming a Studio
- For high-reliability needs, use external storage (OSS, databases)

## Notes

1. The default branch is `master`; force push is prohibited
2. Files over 100MB must use Git LFS
3. The first Docker build takes about 3-5 minutes; other types start faster
4. Free quotas have time limits
5. Switching to paid hardware resources will incur charges on the Alibaba Cloud account bound to the user's ModelScope account; the user's explicit authorization must be obtained first
6. Hardcoding sensitive information in code is prohibited; it must be injected via secrets; non-sensitive configuration may use plaintext variables
7. A Studio **cannot be deleted through a programmatic interface**: OpenAPI `DELETE /openapi/v1/studios/{id}` returns 404; the SDK/CLI `delete_repo` is deprecated and does not support studio. Deletion is only possible via the web console at https://modelscope.cn (which uses a cookie-authenticated internal interface; a Bearer token call returns 401). Programmatically, you can only stop (`stop`).

## Reference Documentation

Consult when you run into problems or need more detail:

- `references/openapi-studio-endpoints.md` — detailed Studios OpenAPI endpoint parameters (source-of-truth specification)
- `references/docker-templates.md` — Dockerfile templates for the Docker type (FastAPI, Node.js, Golang, etc.)
- `references/troubleshooting.md` — deployment-failure troubleshooting process, common errors, and fixes

<details>
<summary>Appendix: MCP tools first-time setup (optional)</summary>

#### Query existing deployment links

```bash
curl -s "$MODELSCOPE_ENDPOINT/openapi/v1/mcp/servers/maasadmin/studio-mcp?get_operational_url=true" \
  -H "Authorization: Bearer ${MODELSCOPE_API_KEY}"
```

Check `data.operational_urls` in the returned JSON; if there is an entry with `accessible: true`, extract its `url`.

#### Deploy the MCP service (when no link is available)

```bash
ms mcp deploy maasadmin/studio-mcp --transport-type streamable_http
```

> The `transport_type` for MCP deploy is required, with valid values `sse` / `streamable_http`; see the MCP section of ms-hub for details.

#### Write the local MCP configuration (Cursor `.cursor/mcp.json`)

```json
{
  "mcpServers": {
    "modelscope-studio": {
      "type": "streamable_http",
      "url": "<deployment URL>",
      "name": "modelscope-studio"
    }
  }
}
```

For other clients, add a Streamable HTTP type MCP service per their respective documentation, using the same URL. After configuration, call the MCP tool `getCurrentUser` to verify the connection.

</details>

## Related Skills

- Hub repository management / models & datasets / MCP / skill management → ms-hub
