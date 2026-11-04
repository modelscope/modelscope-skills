# Repository Management & Version Control Reference

> ms-hub §3 repository management, §6 version control.
> Moved verbatim from the former SKILL.md inline sections; read on demand via the skill file list.

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
