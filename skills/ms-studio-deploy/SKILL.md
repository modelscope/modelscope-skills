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

> Verified live against ModelScope OpenAPI ｜ modelscope 1.37.1 / modelscope_hub 0.1.2 (2026-06-29)

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

## Reference Docs (read on demand)

Detailed material lives in `references/` — read it only when the task needs it:

| Topic | File | Covers |
|---|---|---|
| Deployment workflow | references/deployment-workflow.md | the full 10-step workflow: git check → analyze → create/update → secrets → git sync (LFS) → variables → deploy → logs → diagnose → finish |
| Docker templates | references/docker-templates.md | Dockerfile templates for Docker-type studios |
| OpenAPI endpoints | references/openapi-studio-endpoints.md | full Studio endpoint reference |
| Troubleshooting | references/troubleshooting.md | build/run failure diagnosis and repair |

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
