# ModelScope Skills

[English](README.md) | **中文**

ModelScope Skills 为 AI Agent 提供 ModelScope 魔搭社区的操作能力——模型/数据集管理、创空间部署、MCP 服务配置、技能中心使用。同时服务魔搭的两个站点：国内站 [modelscope.cn](https://modelscope.cn) 与国际站 [modelscope.ai](https://www.modelscope.ai)。兼容 Cursor、Claude Code、Codex、Gemini CLI 等主流 Coding Agent 工具。

本仓库遵循 [Agent Skills](https://agentskills.io/home) 标准格式。

> [!NOTE]
> 优先安装 [`ms-hub`](skills/ms-hub/SKILL.md) 技能——它是整个 ModelScope 平台的统一操作入口，让 Agent 可以通过 `ms` CLI、Python SDK 和 OpenAPI 完成绝大多数平台操作。

## 可用技能

| 名称 | 说明 | 文档 |
|------|------|------|
| `ms-hub` | 统一操作入口——模型/数据集搜索下载上传、仓库管理、创空间速查、MCP 服务、技能中心 | [SKILL.md](skills/ms-hub/SKILL.md) |
| `ms-studio-deploy` | 将本地项目部署到创空间（Gradio / Streamlit / Docker / 静态站），API 优先的完整部署流程和自动诊断 | [SKILL.md](skills/ms-studio-deploy/SKILL.md) |

> MCP 服务管理与技能中心发布已并入 `ms-hub`：SKILL.md 保留快速决策指南，操作速查与展开细节见 `ms-hub/references/mcp-services.md` 与 `skills-center.md`。

## 安装

### 通过 ModelScope CLI（通用）

```bash
pip install modelscope
modelscope skills add @ModelScope/ms-hub
# 若 modelscope/ms 入口无 skills 命令（被 modelscope_hub 覆盖），改用更可靠的：
# curl -fsSL https://modelscope.cn/skills/install.sh | bash -s -- @ModelScope/ms-hub
```

技能默认安装到 `~/.agents/skills/`，主流 Agent 工具会自动发现。

另外可以选择从源码安装：

```bash
git clone https://github.com/modelscope/modelscope-skills.git
cp -r modelscope-skills/skills/ms-hub ~/.agents/skills/
```

### Cursor

将 `skills/` 下的技能文件夹复制到项目的 `.cursor/skills/` 目录，Cursor 会自动发现其中的 SKILL.md。

### Claude Code

```
/plugin marketplace add modelscope/modelscope-skills
/plugin install ms-hub@modelscope/modelscope-skills
```

### Codex

将文件夹复制或软链接到 `$HOME/.agents/skills`（或 `$REPO_ROOT/.agents/skills`）。

### Gemini CLI

```bash
gemini extensions install https://github.com/modelscope/modelscope-skills.git --consent
```

## 使用

装好之后直接对 Agent 说你想做什么：

- "下载 Qwen2.5-7B-Instruct"
- "把这个项目部署到魔搭创空间"
- "找一个地图 MCP 服务配置到 Cursor"
- "发布我的 Skill 到技能中心"

Agent 会自动加载对应的 Skill 并执行。

## 前置要求

```bash
pip install modelscope
export MODELSCOPE_API_KEY="your_token"   # 从目标站点的 /my/myaccesstoken 获取
# 可选 —— 选择站点（默认国内站 modelscope.cn）：
export MODELSCOPE_ENDPOINT="https://www.modelscope.ai"   # 国际站；国内站可省略
```

## 站点（国内 / 国际）

魔搭有两个独立站点，账号、令牌、内容库各自独立：

| 站点 | Endpoint（`MODELSCOPE_ENDPOINT`） | 令牌页 |
|------|-----------------------------------|--------|
| 国内站（默认） | `https://modelscope.cn` | https://modelscope.cn/my/myaccesstoken |
| 国际站 | `https://www.modelscope.ai` | https://www.modelscope.ai/my/myaccesstoken |

设置 `MODELSCOPE_ENDPOINT` 即可一次性切换 OpenAPI 基址、CLI、SDK 与 git 地址。访问令牌**按站点隔离**——必须使用目标站点签发的令牌。Skill 会依据 `MODELSCOPE_ENDPOINT` / 令牌 / 显式意图判断目标站点，未指定时默认国内站。详见 [`ms-hub`](skills/ms-hub/SKILL.md) 的「站点选择与 endpoint 路由」。

## 技能关系

```
ms-hub（统一入口 / 速查）
 ├── Hub 模型/数据集/仓库操作
 ├── MCP 服务搜索·部署·配置  → references/mcp-services.md
 ├── 技能中心搜索·安装·发布   → references/skills-center.md
 └── ms-studio-deploy（创空间完整部署流程，API 优先）
```

`ms-hub` 覆盖常用操作的速查命令；MCP 与技能中心的展开细节下沉到 references，创空间的多步长流程独立为 `ms-studio-deploy`。

## 贡献

1. `skills/` 下新建文件夹
2. 编写 `SKILL.md`：
   ```markdown
   ---
   name: my-skill
   description: 技能用途和触发条件
   ---

   # 标题
   指引 + 示例 + 约束
   ```
3. 按需添加脚本、模板、参考文档
4. 提交 PR

## 许可证

Apache License 2.0

## 相关链接

- [ModelScope 官方文档](https://modelscope.cn/docs/)
- [ModelScope 技能中心](https://modelscope.cn/skills)
- [ModelScope MCP 广场](https://modelscope.cn/mcp)
- [Agent Skills 标准](https://agentskills.io/home)
