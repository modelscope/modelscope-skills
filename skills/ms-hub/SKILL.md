---
name: ms-hub
description: >-
  ModelScope unified operations entrypoint. Covers model/dataset search, download, and upload; repository management; Studio deployment;
  MCP service search, deployment, and configuration; and Skills Center search, install, and publish. Use this skill whenever the user mentions ModelScope or any platform operation.
  Use ms-studio-deploy for complex Studio deployment workflows; see this skill's references for the expanded MCP and Skills Center details.
---

# ModelScope Unified Operations Entrypoint

> Verified with modelscope 1.37.1, Python 3.12 (2026-06-23); OpenAPI endpoint specs cross-checked against the modelscope_service master branch (2026-10-23)

Operate the full range of ModelScope platform capabilities through OpenAPI, CLI, and SDK, covering Hub (models/datasets), Studio, MCP services, and the Skills Center (Skills). This Skill serves as a quick-reference entrypoint; complex operational workflows require the dedicated Skill.

## Requirements

```bash
pip install modelscope
```

`pip install modelscope` installs both the SDK (`modelscope.hub.api.HubApi`) and two sets of command-line entrypoints:

- **`ms` (driven by modelscope_hub v0.1.2)**: Hub / Studio / MCP operations (`download`/`upload`/`create`/`deploy`/`mcp`/`secret`/…). This document uses `ms` uniformly for Hub/Studio/MCP commands.
- **`modelscope` (legacy CLI)**: additionally provides commands such as `skills` (`modelscope skills add`) — `ms` (modelscope_hub) **does not have** a `skills` subcommand.

> ⚠️ Both packages register the `ms` and `modelscope` entrypoints; which one actually takes effect depends on install order. If an entrypoint lacks a required subcommand (typically: `ms` has no `skills`), switch to the other entrypoint, or use the SDK / `curl install.sh` (see §9 and `references/skills-center.md`).

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
│   ├── List Studios → GET /studios (search/status/owner/mcp_support/hardware_type/sort)
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

## 1. Resource Search

### OpenAPI Method

**Search models:**

```bash
curl "$MODELSCOPE_ENDPOINT/openapi/v1/models?search=Qwen&sort=downloads&page_size=20" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
```

**Search parameters:**

| Parameter | Description | Example |
|------|------|------|
| `search` | Keyword | `"Qwen"`, `"text generation"` |
| `owner` | Author/organization | `"Qwen"`, `"ZhipuAI"` |
| `sort` | Sort | `default`, `downloads`, `likes`, `last_modified` |
| `page_size` | Items per page (max 50) | `20` |
| `filter.task` | Task type | `text-generation`, `image-captioning` |
| `filter.library` | Framework | `pytorch`, `safetensors`, `diffusers` |
| `filter.model_type` | Model type | `qwen3_moe`, `glm4v`, `llama` |
| `filter.license` | License | `Apache License 2.0`, `MIT License` |

**Common filter combinations:**

```bash
# PyTorch text-generation models, sorted by downloads
/models?filter.library=pytorch&filter.task=text-generation&sort=downloads

# All models from a specific author
/models?owner=Qwen&sort=last_modified
```

**Search datasets:**

```bash
curl "$MODELSCOPE_ENDPOINT/openapi/v1/datasets?search=dialogue&sort=downloads&page_size=10" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
```

**OpenAPI response structure (model list):**

```json
{"data": {"models": [{"id": "Qwen/...", "downloads": N, "likes": N, "license": "...", "tasks": [...]}], "total_count": N}}
```

### SDK Method

```python
from modelscope.hub.api import HubApi

api = HubApi()

# Search models → dict{"Models": [...], "TotalCount": N}
result = api.list_models(owner_or_group="Qwen", page_number=1, page_size=20)
for m in result["Models"]:
    print(f"{m['Path']} ({m['Downloads']} downloads)")

# Search datasets → dict{"datasets": [...], "total_count": N}
result = api.list_datasets(owner_or_group="AI-ModelScope", page_number=1, page_size=20)
for d in result["datasets"]:
    print(f"{d['id']} ({d['downloads']} downloads)")
```

> **SDK vs OpenAPI field name differences**: SDK `list_models` returns PascalCase (`Path`, `Downloads`), while OpenAPI `/models` returns snake_case (`id`, `downloads`). `list_datasets` is snake_case on both sides.

## 2. View Details

### OpenAPI Method

```bash
# Model details
curl "$MODELSCOPE_ENDPOINT/openapi/v1/models/Qwen/Qwen2.5-72B-Instruct" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"

# Dataset details
curl "$MODELSCOPE_ENDPOINT/openapi/v1/datasets/AI-ModelScope/alpaca-gpt4-data-zh" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
```

### SDK Method (more complete info)

```python
info = api.model_info("Qwen/Qwen2.5-72B-Instruct")
# info.readme_content  - Full README text
# info.tags            - Tag list
# info.downloads       - Download count
# info.siblings        - File list (incl. rfilename, size, sha)
# info.visibility      - Visibility (1=private, 5=public)

info = api.dataset_info("AI-ModelScope/alpaca-gpt4-data-zh")
```

### Get User Info

```bash
curl "$MODELSCOPE_ENDPOINT/openapi/v1/users/me" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
```

## 3. Repository Management

### Create Repository

```bash
# CLI (--repo-type required)
ms create owner/repo-name --repo-type model
ms create owner/repo-name --repo-type model --visibility private
ms create owner/dataset-name --repo-type dataset
```

```python
# SDK: general creation — note that create_repo's visibility uses the string "public"/"private"
api.create_repo(
    repo_id="owner/repo-name",
    repo_type="model",            # "model" or "dataset"
    visibility="public",          # "public" or "private" (string, not integer)
    license="Apache License 2.0",
    exist_ok=True
)

# Model-specific — create_model/create_dataset visibility uses integers 1=private, 5=public
api.create_model(model_id="owner/model-name", visibility=5)

# Dataset-specific
api.create_dataset(
    dataset_name="dataset-name",
    namespace="owner",
    visibility=5
)

# AIGC/LoRA models: via the SDK's aigc_model parameter (create_repo/create_model); no corresponding CLI flag
```

### Check Whether a Repository Exists

```python
exists = api.repo_exists(repo_id="owner/repo", repo_type="model")
```

### Set Visibility

```python
# visibility uses the string "public" / "private" (not an integer)
api.set_repo_visibility(repo_id="owner/repo", repo_type="model", visibility="private")
```

### Delete Repository

> ⚠️ **Repository deletion has been restricted by the platform to the web console only**: `api.delete_repo(...)` returns 401 under token authentication ("Deletion is restricted to web console") and cannot be deleted programmatically. Please perform this operation on the web at https://modelscope.cn.

## 4. File Operations

### List Files

```python
files = api.get_model_files(
    model_id="Qwen/Qwen2.5-7B-Instruct",
    revision="master",
    recursive=True
)
for f in files:
    print(f"  {f['Name']}  Size: {f.get('Size', 'unknown')}")

# Check whether a file exists
exists = api.file_exists(repo_id="owner/repo", filename="config.json", revision="master")
```

### Read File Content

```bash
# Use the helper script
uv run scripts/ms_read_file.py \
    --repo_id "Qwen/Qwen2.5-7B-Instruct" \
    --file_path "config.json" \
    --repo_type model
```

```python
# SDK manual download
from modelscope.hub.file_download import model_file_download

local_path = model_file_download(
    model_id="Qwen/Qwen2.5-7B-Instruct",
    file_path="config.json"
)
```

### Download Models/Files

```bash
# CLI: download the full model
ms download Qwen/Qwen2.5-7B-Instruct

# CLI: download a specific file
ms download Qwen/Qwen2.5-7B-Instruct config.json

# CLI: filter by pattern
ms download Qwen/Qwen2.5-7B-Instruct --include "*.json" --exclude "*.safetensors"
```

```python
# SDK: download snapshot
from modelscope import snapshot_download

local_dir = snapshot_download(
    model_id="Qwen/Qwen2.5-7B-Instruct",
    cache_dir="/tmp/models",
    allow_file_pattern=["*.json", "*.md"],    # Download only matching files
    ignore_file_pattern=["*.safetensors"]     # Exclude large files
)

# Download a single dataset file
from modelscope.hub.file_download import dataset_file_download
local_path = dataset_file_download(dataset_id="owner/dataset", file_path="data/train.jsonl")
```

### Upload Files

```bash
# CLI: upload a directory
ms upload owner/repo ./local-dir
```

```python
# SDK: upload a single file
api.upload_file(
    path_or_fileobj="/path/to/file.txt",
    path_in_repo="data/file.txt",
    repo_id="owner/repo",
    repo_type="model",
    commit_message="Add data file"
)

# SDK: upload a directory
api.upload_folder(
    repo_id="owner/repo",
    folder_path="/path/to/folder",
    commit_message="Upload model files",
    repo_type="model",
    ignore_patterns=["*.pyc", "__pycache__", ".git"]
)
```

### Delete Files

> ⚠️ **File deletion is likewise restricted to the web console only**: `api.delete_files(...)` has no effect under token authentication (the old SDK silently returns `failed_files` while the files remain; the new `modelscope_hub` reports 401 "Deletion is restricted to web console"). To delete files, go to the web console at https://modelscope.cn, or clone the repository (git), delete the files, then commit and push.

### Atomic Multi-File Commit

```python
from modelscope.hub.api import CommitOperationAdd

operations = [
    CommitOperationAdd(path_in_repo="config.json", path_or_fileobj="/local/config.json"),
    CommitOperationAdd(path_in_repo="README.md", path_or_fileobj="/local/README.md"),
]
api.create_commit(
    repo_id="owner/repo", operations=operations,
    commit_message="Update config and docs", repo_type="model"
)
```

### Notebook / Tutorial Adaptation

When migrating from HuggingFace, Colab, or GitHub tutorials to ModelScope, the core task is replacing the asset sources:

1. **Model download**: Replace `hf_hub_download` / HF `snapshot_download` with the `ms download` above or the SDK `snapshot_download`
2. **Dataset loading**: Replace `datasets.load_dataset("hf_id")` with `MsDataset.load("ms_id")` or `dataset_snapshot_download`
3. **Repository search**: Use OpenAPI or the SDK to search for equivalent resources on ModelScope

> For the complete adaptation workflow (asset mapping, license checking, execution validation), see: `modelscope skills add VoyagerX/modelscope-notebook-develop`

### Error-Prevention Comparison

```python
# ✅ CORRECT — Explicitly specifying repo_type improves readability
api.upload_folder(repo_id="owner/repo", folder_path="./local", repo_type="model")

# ⚠️ Also works (repo_type defaults to model), but explicit is recommended
api.upload_folder(repo_id="owner/repo", folder_path="./local")

# ✅ CORRECT — Use the model_id parameter
snapshot_download(model_id="Qwen/Qwen2.5-7B-Instruct")

# ⚠️ repo_id also works, but model_id is semantically clearer
snapshot_download(repo_id="Qwen/Qwen2.5-7B-Instruct")
```

## 5. Dataset Inspection

ModelScope currently supports dataset exploration through a combination of the API/SDK/CLI described above:

- **Metadata**: `api.dataset_info()` or OpenAPI `GET /datasets/{id}` → description, tags, file list
- **File browsing**: `api.get_dataset_files()` → list all files and their sizes
- **Content reading**: `ms_read_file.py --repo_type dataset` → download and view the content of a single file
- **Deep inspection**: `ms_inspect_dataset.py` → wraps the above capabilities + `MsDataset.load` to perform schema extraction and sample preview in one step (requires downloading data locally)

### Using the Helper Script

Use the helper script `scripts/ms_inspect_dataset.py` to quickly understand a dataset's file structure, field schema, and sample content. Internally the script calls the SDK (`HubApi.dataset_info` + `MsDataset.load`) to perform the operations.

> The examples below use `uv run` (zero-config); in an environment with modelscope already installed you can also run `python scripts/ms_inspect_dataset.py ...` directly — see "Helper Scripts" at the end of the document.

```bash
# Full inspection (file structure + schema + sample preview)
uv run scripts/ms_inspect_dataset.py \
    --dataset_id "AI-ModelScope/alpaca-gpt4-data-zh" \
    --operation full

# View file structure only
uv run scripts/ms_inspect_dataset.py \
    --dataset_id "AI-ModelScope/alpaca-gpt4-data-zh" \
    --operation overview

# View schema only
uv run scripts/ms_inspect_dataset.py \
    --dataset_id "AI-ModelScope/alpaca-gpt4-data-zh" \
    --operation schema --split train

# Preview samples only
uv run scripts/ms_inspect_dataset.py \
    --dataset_id "AI-ModelScope/alpaca-gpt4-data-zh" \
    --operation samples --num_samples 5
```

## 6. Version Control

### List Branches and Tags

```python
branches, tags = api.get_model_branches_and_tags(model_id="Qwen/Qwen2.5-7B-Instruct")

# Detailed info (incl. commit hash, timestamps, etc.)
details = api.get_model_branches_and_tags_details(model_id="owner/repo")
```

### Validate Revision

```python
# First argument is model_id (not repo_id)
valid = api.get_valid_revision(model_id="owner/repo", revision="v1.0")
```

### Create Tag

```python
api.create_model_tag(
    model_id="owner/model-name",
    tag_name="v1.0"
)
```

### View Commit History

```python
commits = api.list_repo_commits(
    repo_id="owner/repo",
    repo_type="model",
    revision="master",
    page_number=1,
    page_size=20
)
```

### Known Limitations

| Operation | Status | Notes |
|------|------|------|
| List branches/tags | ✅ | `get_model_branches_and_tags()` |
| Create tag | ✅ | `create_model_tag()` |
| Delete tag | ❌ | No API |
| Create branch | ❌ | Requires `git clone` → `git checkout -b` → `git push` |
| Delete branch | ❌ | No API |
| Merge branch | ❌ | ModelScope has no PR system |

## 7. Studio Operations

> For the complete deployment workflow (including code sync, diagnosis and repair) → ms-studio-deploy (**OpenAPI is the source of truth**, CLI is an equivalent alias)
>
> The CLI is driven by modelscope_hub and is a 1:1 thin wrapper around OpenAPI; the API-first Python entrypoint is `from modelscope_hub import HubApi`.
> If the agent already has the studio-mcp tools configured, you can also use the MCP tools (`createStudio`, `deployStudio`, etc.).

### List Studios

```bash
# OpenAPI (status: running|all; hardware_type: xGPU; mcp_support: true|false)
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios?search=my-app&status=running&page_number=1&page_size=20" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
# Response data: {studios: [...], total_count, page_number, page_size}; page_number × page_size ≤ 3000
```

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

Create fields: `owner`/`repo_name` required; optional `sdk_type` (defaults `gradio`), `visibility` (`public`/`protected`/`private`), `display_name`, `description`, `cover_image`, `license`, `hardware` (may be omitted — server normalizes to the default free tier), `base_image`, `sdk_version`. `private` (boolean) is deprecated. The OAuth app group (`oauth_enabled` + `oauth_scope` / `oauth_token_expire_time` / `oauth_callback_urls` / `oauth_homepage_url`) can also be set at creation with the same semantics as in Update Settings.

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

Logs support query params: `page_num` (default 1), `page_size` (default 100, max 500), `keyword`, `start_timestamp`/`end_timestamp` (Unix seconds, default last 30 days). `log_type` only accepts `run`/`build`.

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

The Studio OAuth app group `oauth_enabled` / `oauth_scope` / `oauth_token_expire_time` / `oauth_callback_urls` / `oauth_homepage_url` is also updatable: the server applies the group only when `oauth_enabled` is present in the payload (`true` = enable/reconfigure, `false` = disable). Valid scopes: `openid` (ModelScope user identity), `profile` (public user profile), `manage-mcp-deployment`, `list-operational-mcp`, `read-repos`, `api-inference`; `oauth_token_expire_time` enum: `1`=15min, `2`=1hour, `3`=8hours, `4`=1day, `5`=1week, `6`=30days.

```bash
# CLI
ms settings USERNAME/my-app --repo-type studio \
  --oauth-enabled --oauth-scope "openid profile" \
  --oauth-token-expire-time 4 --oauth-callback-url https://example.org/cb

# OpenAPI
curl -X PATCH "$MODELSCOPE_ENDPOINT/openapi/v1/studios/USERNAME/my-app/settings" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"oauth_enabled": true, "oauth_scope": "openid profile", "oauth_token_expire_time": 4, "oauth_callback_urls": ["https://example.org/cb"]}'
```

### Variable Management

Plaintext variables return key, value, and an optional `description` (≤ 512 chars), and are used only for non-sensitive configuration. On update, `description` keeps its original value when omitted and is cleared when set to `""`:

```bash
curl "$MODELSCOPE_ENDPOINT/openapi/v1/studios/USERNAME/my-app/variables" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/studios/USERNAME/my-app/variables" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"key": "GRADIO_TEMP_DIR", "value": "/tmp/gradio", "description": "Gradio temp dir"}'
curl -X PUT "$MODELSCOPE_ENDPOINT/openapi/v1/studios/USERNAME/my-app/variables" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"key": "GRADIO_TEMP_DIR", "value": "/mnt/workspace/tmp"}'
curl -X DELETE "$MODELSCOPE_ENDPOINT/openapi/v1/studios/USERNAME/my-app/variables" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"key": "GRADIO_TEMP_DIR"}'
```

Secrets return key and description (not the value) and are used for sensitive information such as API keys, tokens, and passwords:

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
  -d '{"key": "API_KEY", "value": "sk-xxx", "description": "Upstream API key"}'
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

## 8. MCP Service Operations

> Full orchestration, IDE configuration templates, SDK↔OpenAPI field differences → see `references/mcp-services.md`
>
> Pagination limit `page_number × page_size ≤ 100` (server-enforced; exceeding it returns HTTP 403).

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

When deploying, **`transport_type` is required**; valid values `sse` / `streamable_http` (and high-spec variants `SSE` / `STREAMABLE_HTTP`, which the server normalizes to the base types); the deployed transport determines the unique URL returned (`sse`→`.../sse`, `streamable_http`→`.../mcp`).

Optional deploy body fields: `environment_variables` (map `{KEY: value}`, needed by services requiring an API Key; legacy alias `env_info`), `expiration_minutes` (`-1` = permanent), `auth_check` (bool), `reset` (bool, redeploy from scratch).

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

## 9. Skills Center Operations

> Category system, packaging spec, complete publish/update/install → see `references/skills-center.md`

### Search Skills

```bash
curl "$MODELSCOPE_ENDPOINT/openapi/v1/skills?search=code-review&filter.category=developer-tools&page_size=20" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
# Filters: filter.category / filter.developer / filter.license / filter.custom_tag / filter.owner
# Pagination: page_size 1–100; total result quota 2400 skills
```

### View Skill Details

```bash
curl "$MODELSCOPE_ENDPOINT/openapi/v1/skills/@ModelScope/modelscope-oauth-skill" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
# Returns an install_command array containing multiple installation methods
```

### Install Skill

> ⚠️ `skills add` is a command of the **legacy `modelscope` CLI**; `ms` (modelscope_hub) **does not have** `skills`. If the `modelscope`/`ms` entrypoint is overridden by modelscope_hub and reports "no skills command", switch to the `curl install.sh` or SDK `download_skill` below (both are the most reliable).

```bash
# Option 1: modelscope (legacy) CLI
modelscope skills add @author/skill-name
modelscope skills add @author/skill-name --local_dir ./my-skills   # Specify directory
modelscope skills add @author/skill-1 @author/skill-2              # Batch

# Option 2: Shell script (most reliable, no entrypoint conflicts)
curl -fsSL https://modelscope.cn/skills/install.sh | bash -s -- @author/skill-name
curl -fsSL https://modelscope.cn/skills/install.sh | bash -s -- @author/skill-name --agent cursor
```

```python
# Option 3: SDK (verified reliable)
from modelscope.hub.mcp_api import MCPApi
MCPApi().download_skill(skill_id="@author/skill-name", local_dir="./my-skills")
```

`modelscope skills add` parameters:

| Parameter | Description |
|------|------|
| `skill_ids` | Positional argument; one or more skill IDs (format `@author/name`) |
| `--local_dir DIR` | Install directory (default `~/.agents/skills`) |
| `--token TOKEN` | Access Token |
| `--max-workers N` | Number of concurrent downloads (default 8) |

### Publish Skill (Quick Reference)

```bash
# Step 1: Upload the zip package (the zip root must contain exactly 1 SKILL.md; max 5MB, one file per request)
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/files/upload" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -F "file=@my-skill.zip"
# → Response {"data": {"id": "<uuid>"}}; use data.id as the skill_file for the next step (note the key is id, not file_id)

# Step 2: Create the skill (skill_file takes the data.id from the previous step)
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/skills" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"owner": "username", "skill_name": "my-skill", "display_name": "My Skill", "skill_file": "<data.id>", "category": "developer-tools"}'
```

Required fields: `owner`, `skill_name` (lowercase letters/digits/hyphens, ≤ 64 chars), `category` (must be an allowed enum — see `references/skills-center.md`), `skill_file`. Optional: `display_name` (≤ 128), `description` (≤ 2000), `license` (defaults `Apache-2.0`), `tags`, `source_url` (valid URL), `logo_url` (valid URL), `private` (bool, default public), `auto_sync_enabled` (bool — GitHub auto-sync, effective only when `source_url` is a GitHub repo).

### Update Skill Settings

```bash
curl -X PATCH "$MODELSCOPE_ENDPOINT/openapi/v1/skills/{owner}/{skill_name}/settings" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"display_name": "New name", "description": "Updated description", "skill_file": "<new_file_id>"}'
```

Updatable fields: `display_name`, `description`, `skill_file`, `tags`, `source_url`, `category`, `license`, `private` (bool), `logo_url`, `auto_sync_enabled`. Not modifiable: `owner`, `skill_name`. Skill visibility changes are synced to the GitLab mirror repo.

### GitHub Source Sync

For skills published with a GitHub `source_url`:

```bash
# Check whether the upstream GitHub repo has updates
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/skills/{owner}/{skill_name}/check-update" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
# → {"data": {"status": "..."}} indicating whether an update is available

# Manually trigger a sync from GitHub
curl -X POST "$MODELSCOPE_ENDPOINT/openapi/v1/skills/{owner}/{skill_name}/sync" \
  -H "Authorization: Bearer $MODELSCOPE_API_KEY"
# → {"data": {"success": true|false, "message": "..."}}
```

With `auto_sync_enabled: true`, the platform syncs from GitHub automatically; `sync_status` in the skill detail reflects the last sync result.

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
| Studio | Port fixed at 7860 | 8080 cannot be used |
| Studio | No programmatic deletion | OpenAPI `DELETE /studios/{id}` returns 404; SDK/CLI `delete_repo` is deprecated and does not support studio. Deletion requires the web console; programmatically you can only `stop` |
| MCP | Deployment `transport_type` required | Valid `sse`/`streamable_http` (or high-spec `SSE`/`STREAMABLE_HTTP`), otherwise HTTP 400 |
| MCP | Pagination limit ≤ 100 | `page × size > 100` returns HTTP 403 |
| MCP | SDK has no deploy/undeploy | Use the CLI `ms mcp deploy/undeploy` or OpenAPI |
| Skills | CLI supports only `add` | No `list`/`update`/`remove` subcommands yet |
| Skills | Pagination limits | `page_size` 1–100; total result quota 2400 skills (HTTP quota error beyond) |

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

| Domain | In ms-hub | Expanded location |
|----|-----------|----------|
| Studio | Quick reference: create/deploy/stop/logs | **ms-studio-deploy** (full deployment workflow, code sync, diagnosis and repair, API-first) |
| MCP | Quick reference: search/details/deploy/undeploy/deployed | `references/mcp-services.md` (IDE configuration templates, full orchestration, field differences) |
| Skills | Quick reference: search/details/install/publish/update | `references/skills-center.md` (category system, packaging spec, publish workflow) |
