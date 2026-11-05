---
name: ms-hub
description: >-
  ModelScope unified operations entrypoint. Covers model/dataset search, download, and upload; repository management; Studio deployment;
  MCP service search, deployment, and configuration; and Skills Center search, install, and publish. Use this skill whenever the user mentions ModelScope or any platform operation.
  Use ms-studio-deploy for complex Studio deployment workflows; see this skill's references for the expanded MCP and Skills Center details.
---

# ModelScope Unified Operations Entrypoint

> Verified with modelscope 1.37.1, Python 3.12 (2026-06-23)

Operate the full range of ModelScope platform capabilities through OpenAPI, CLI, and SDK, covering Hub (models/datasets), Studio, MCP services, and the Skills Center (Skills). This Skill serves as a quick-reference entrypoint; complex operational workflows require the dedicated Skill.

## Requirements

```bash
pip install modelscope
```

`pip install modelscope` installs both the SDK (`modelscope.hub.api.HubApi`) and two sets of command-line entrypoints:

- **`ms` (driven by modelscope_hub v0.1.2)**: Hub / Studio / MCP operations (`download`/`upload`/`create`/`deploy`/`mcp`/`secret`/…). This document uses `ms` uniformly for Hub/Studio/MCP commands.
- **`modelscope` (legacy CLI)**: additionally provides commands such as `skills` (`modelscope skills add`) — `ms` (modelscope_hub) **does not have** a `skills` subcommand.

> ⚠️ Both packages register the `ms` and `modelscope` entrypoints; which one actually takes effect depends on install order. If an entrypoint lacks a required subcommand (typically: `ms` has no `skills`), switch to the other entrypoint, or use the SDK / `curl install.sh` (see `references/skills-center.md`).

## Authentication

All operations rely on unified authentication:

```bash
# Environment variable
export MODELSCOPE_API_KEY="your_token"

# Token retrieval URL
# $MODELSCOPE_ENDPOINT/my/myaccesstoken
```

| Operation method | Authentication method |
|----------|----------|
| OpenAPI | `Authorization: Bearer $MODELSCOPE_API_KEY` |
| CLI | `ms login --token $MODELSCOPE_API_KEY` |
| SDK | `api.login(access_token=os.environ['MODELSCOPE_API_KEY'])` |

## Site selection & endpoint routing

ModelScope runs two independent sites — the **domestic** site `https://modelscope.cn` (default) and the **international** site `https://www.modelscope.ai`. They have separate accounts, access tokens, and content catalogs. Every operation here targets whichever site `$MODELSCOPE_ENDPOINT` points to.

### Pick the target site (intent analysis)

1. **Respect an existing setting** — if `MODELSCOPE_ENDPOINT` is already exported, use it as-is.
2. **Explicit intent** — "international" / "modelscope.ai" / "overseas" ⇒ international; "domestic" / "modelscope.cn" ⇒ domestic.
3. **Match the token or URL the user provides** — a `modelscope.ai` token or link ⇒ international (and vice versa).
4. **Default to the domestic site** (`https://modelscope.cn`) when there is no signal; ask the user if the task clearly targets one audience but the site is ambiguous.

### Configure

```bash
# Export the endpoint first — the examples below reference $MODELSCOPE_ENDPOINT:
export MODELSCOPE_ENDPOINT="https://modelscope.cn"          # domestic (default)
# export MODELSCOPE_ENDPOINT="https://www.modelscope.ai"   # international
export MODELSCOPE_API_KEY="<token issued by THAT site>"     # tokens are site-scoped — must match the site
```

One `MODELSCOPE_ENDPOINT` reroutes everything derived from it: the OpenAPI base (`$MODELSCOPE_ENDPOINT/openapi/v1`), the `ms` CLI, the `modelscope_hub` SDK, and git push URLs (`$MODELSCOPE_ENDPOINT/{models,datasets,studios}/…`). Resolution precedence (modelscope_hub): explicit arg > `MODELSCOPE_ENDPOINT` > `MODELSCOPE_DOMAIN` (deprecated) > default `https://modelscope.cn`. For public reads you may also set `MODELSCOPE_PREFER_AI_SITE=true` to try `.ai` before `.cn`.

> **Tokens are site-scoped** (stored per endpoint host): a `modelscope.cn` token will not authorize write/private operations on `modelscope.ai`. Get each site's token from `$MODELSCOPE_ENDPOINT/my/myaccesstoken`.
>
> All examples below use `$MODELSCOPE_ENDPOINT/openapi/v1` as the base — **export `MODELSCOPE_ENDPOINT` first** (raw `curl` needs it set; the `ms` CLI and SDK additionally fall back to `https://modelscope.cn` when it is unset). A few marketplace/doc links (skills `install.sh`, `/docs/…`) show the domestic host — swap to your site's host when targeting international.

## Conventions

| Item | Value |
|------|-----|
| **OpenAPI Base URL** | `$MODELSCOPE_ENDPOINT/openapi/v1` (default `https://modelscope.cn`) |
| **Success response** | `{"success": true, "data": {...}, "request_id": "..."}` |
| **Error response** | `{"success": false, "code": "ERROR_CODE", "message": "..."}` |
| **HTTP status codes** | `200` success / `401` unauthorized / `404` not found / `500` server error |
| **Default branch** | `master` (not main) |
| **Pagination limit** | `page_number × page_size ≤ 3000` |

## Quick Decision Guide

```
User wants to...
│
├─── Hub: models/datasets ─────────────────────────────
│   ├── Search models/datasets → OpenAPI GET /models or /datasets
│   ├── View details → GET /models/{owner}/{repo} or SDK model_info()
│   ├── Download → CLI: ms download owner/repo
│   ├── Upload → CLI: ms upload owner/repo ./local
│   ├── Create repository → CLI: ms create owner/repo
│   ├── Browse files → SDK: api.get_model_files()
│   ├── Inspect dataset → uv run scripts/ms_inspect_dataset.py
│   └── Version management → SDK: api.get_model_branches_and_tags()
│
├─── Studio ──────────────────────────────
│   ├── Create Studio → POST /studios or CLI: ms create owner/repo --repo-type studio
│   ├── Deploy/restart → CLI: ms deploy owner/repo --repo-type studio
│   ├── View status → GET /studios/{owner}/{repo}
│   ├── View logs → CLI: ms logs owner/repo --log-type run
│   ├── Stop → CLI: ms stop owner/repo --repo-type studio
│   ├── Update settings → CLI: ms settings owner/repo key=value --repo-type studio
│   ├── Available configs → GET /studios/hardware, /studios/sdk-versions, /studios/base-images
│   ├── Plaintext variables → GET/POST/PUT/DELETE /studios/{owner}/{repo}/variables
│   ├── Secrets → GET/POST/PUT/DELETE /studios/{owner}/{repo}/secrets (or ms secret ...)
│   └── Full deployment workflow → see ms-studio-deploy
│
├─── MCP: service management ──────────────────────────────
│   ├── Search MCP services → CLI: ms mcp list --search "..."
│   ├── View details → CLI: ms mcp info @author/name
│   ├── Deploy service → CLI: ms mcp deploy @author/name
│   ├── Undeploy service → CLI: ms mcp undeploy @author/name
│   ├── My deployed → GET /mcp/servers/operational
│   └── IDE configuration / full orchestration → see references/mcp-services.md
│
├─── Skills: Skills Center ────────────────────────────
│   ├── Search skills → GET /skills?search=...
│   ├── View details → GET /skills/{id}
│   ├── Install skill → modelscope skills add @author/skill-name (legacy CLI; ms has no skills)
│   ├── Publish skill → POST /files/upload + POST /skills
│   ├── Update skill → PATCH /skills/{owner}/{skill_name}/settings
│   └── Category system / packaging spec / full publish → see references/skills-center.md
│
├─── User info ───────────────────────────────────
│   └── GET /users/me
│
└─── Not supported ─────────────────────────────────────
    ├── Pull Request (ModelScope has no PR system)
    └── Delete tags/branches (no API)
```

> ⚠️ **Paid Studio resources**: setting `hardware` to `paid/<InstanceType>` (or any returned item with `resource_type=paid`) charges the Alibaba Cloud account bound to the user's ModelScope account. Obtain the user's **explicit authorization** before creating, updating settings, or redeploying; without it, only free resources may be selected. Details: `references/studio-operations.md`.

## Reference Docs (read on demand)

Detailed sections live in `references/` — read them only when the task needs them:

| Topic | File | Covers |
|---|---|---|
| Search & details | references/search-and-details.md | Resource search, view details, dataset inspection |
| Repository management | references/repo-management.md | Create/delete repos & visibility, branches/tags/commits |
| File operations | references/file-operations.md | List/read/download/upload/delete, atomic multi-file commit |
| Studio operations | references/studio-operations.md | Studio create/configure/deploy/logs/variables/code sync |
| MCP services | references/mcp-services.md | Search/details/deploy/undeploy quick reference + full orchestration guide |
| Skills Center | references/skills-center.md | Search/install/publish/update quick reference + full packaging spec |

Full Studio deployment workflow → use the dedicated ms-studio-deploy skill.

## Cache Management

```bash
ms scan-cache                              # View local cache
ms scan-cache --dir ~/.cache/modelscope    # Specify cache directory
ms clear-cache                             # Clear cache
```

## Known Limitations

| Domain | Limitation | Notes |
|----|------|------|
| Hub | No PR system | Collaboration is done via direct push |
| Hub | No row-level preview API | Requires SDK local loading to inspect |
| Hub | Pagination limit | `page_number × page_size ≤ 3000`; `/models` single page `page_size ≤ 50` |
| Hub | Default branch master | Not main |
| Hub | Tags cannot be deleted | Can only be created |
| Hub | Repository/file deletion via web console only | `delete_repo`/`delete_files` return 401 under token; cannot delete programmatically |
| Studio | Docker requires real-name verification | Alibaba Cloud account binding |
| Studio | Paid resources need explicit user authorization | `paid/<InstanceType>` hardware charges the bound Alibaba Cloud account; see references/studio-operations.md |
| Studio | Port fixed at 7860 | 8080 cannot be used |
| Studio | No programmatic deletion | OpenAPI `DELETE /studios/{id}` returns 404; SDK/CLI `delete_repo` is deprecated and does not support studio. Deletion requires the web console; programmatically you can only `stop` |
| MCP | Deployment `transport_type` required | Valid `sse`/`streamable_http`, otherwise HTTP 400 |
| MCP | Pagination limit ≤ 100 | `page × size > 100` returns HTTP 403 |
| MCP | SDK has no deploy/undeploy | Use the CLI `ms mcp deploy/undeploy` or OpenAPI |
| Skills | CLI supports only `add` | No `list`/`update`/`remove` subcommands yet |

## Helper Scripts

| Script | Purpose |
|------|------|
| `scripts/ms_read_file.py` | Download and read repository file content |
| `scripts/ms_inspect_dataset.py` | Deeply inspect dataset structure and content |

Two ways to run (choose either):

```bash
# Option 1: use python directly in an environment with modelscope installed (consistent with "Requirements")
python scripts/ms_inspect_dataset.py --dataset_id "AI-ModelScope/alpaca-gpt4-data-zh" --operation full

# Option 2: uv zero-config (the script includes PEP 723 inline dependencies and auto-creates a temporary environment)
uv run scripts/ms_inspect_dataset.py --dataset_id "AI-ModelScope/alpaca-gpt4-data-zh" --operation full
```

## Relationship to the Dedicated Skill / references

| Domain | Quick reference | Expanded location |
|----|-----------|----------|
| Studio | `references/studio-operations.md` (create/deploy/stop/logs) | **ms-studio-deploy** (full deployment workflow, code sync, diagnosis and repair, API-first) |
| MCP | `references/mcp-services.md` (search/details/deploy/undeploy/deployed) | same file (IDE configuration templates, full orchestration, field differences) |
| Skills | `references/skills-center.md` (search/details/install/publish/update) | same file (category system, packaging spec, publish workflow) |
