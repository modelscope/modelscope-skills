# Resource Search, Details & Dataset Inspection Reference

> ms-hub resource search, view details, and dataset inspection.
> Read on demand via the skill file list in SKILL.md.

## Resource Search

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

## View Details

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


## Dataset Inspection

ModelScope currently supports dataset exploration through a combination of the API/SDK/CLI described above:

- **Metadata**: `api.dataset_info()` or OpenAPI `GET /datasets/{id}` → description, tags, file list
- **File browsing**: `api.get_dataset_files()` → list all files and their sizes
- **Content reading**: `ms_read_file.py --repo_type dataset` → download and view the content of a single file
- **Deep inspection**: `ms_inspect_dataset.py` → wraps the above capabilities + `MsDataset.load` to perform schema extraction and sample preview in one step (requires downloading data locally)

### Using the Helper Script

Use the helper script `scripts/ms_inspect_dataset.py` to quickly understand a dataset's file structure, field schema, and sample content. Internally the script calls the SDK (`HubApi.dataset_info` + `MsDataset.load`) to perform the operations.

> The examples below use `uv run` (zero-config); in an environment with modelscope already installed you can also run `python scripts/ms_inspect_dataset.py ...` directly — see "Helper Scripts" in SKILL.md.

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
