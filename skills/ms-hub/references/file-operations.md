# File Operations Reference

> ms-hub file operations (list/read/download/upload/delete, atomic multi-file commit).
> Read on demand via the skill file list in SKILL.md.

## SDK Prerequisite

The Python examples below use the shared SDK client (authentication: `api.login(access_token=os.environ['MODELSCOPE_API_KEY'])` — see "Authentication" in SKILL.md):

```python
from modelscope.hub.api import HubApi

api = HubApi()
```

## File Operations

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
