# AGENTS.md

本文件适用于整个仓库。后续智能体或贡献者在修改本项目时，应先阅读本文件，再根据具体任务下钻到相关源码、测试和文档。

## 项目定位

Hyper-Extract 是一个基于 LLM 的知识抽取框架，提供 Python SDK 与 `he` 命令行工具。核心目标是把非结构化文本抽取成强类型的 Knowledge Abstract，并支持增量喂入、合并去重、语义索引、搜索、问答和可视化。

项目的三层主线是：

- Auto-Types：`hyperextract/types/` 中的 8 种强类型知识结构。
- Methods：`hyperextract/methods/` 中的内置抽取算法和 RAG 变体。
- Templates：`hyperextract/templates/presets/` 中的 YAML 预设模板，通过模板引擎创建 AutoType 实例。

## 重要目录

- `hyperextract/__init__.py`：SDK 公开入口，导出 AutoType、`Template`、client 工厂和日志工具。
- `hyperextract/types/`：核心数据结构。`BaseAutoType` 负责抽取、切块、合并、索引、搜索/聊天和序列化；具体类型实现状态容器、去重、合并、索引和可视化。
- `hyperextract/utils/template_engine/`：模板发现、YAML 校验/本地化、动态 Pydantic schema 创建、identifier/display/options/guideline 解析，以及 `TemplateFactory` 实例化逻辑。
- `hyperextract/methods/`：内置 method registry。新增 method 必须注册到 `hyperextract/methods/registry.py`，才能被 `Template.create("method/<name>")` 和 CLI 使用。
- `hyperextract/cli/`：Typer CLI。主命令在 `cli.py`，`list` 和 `config` 子命令拆到 `cli/commands/`。
- `hyperextract/templates/presets/`：多领域 YAML 模板库，按 `general`、`finance`、`legal`、`medicine`、`tcm`、`industry` 分域。
- `hyperextract-skills/`：面向模板设计的 agent skills，与运行时包不同，不要误当成 SDK 源码。
- `tests/`：单元、模板引擎、CLI、集成测试与双语测试样本文档。
- `docs/`：MkDocs Material 双语文档。新增页面时同步检查 `mkdocs.yml` 的 i18n nav。
- `examples/`：英语/中文的 provider、AutoType、method、template 示例。
- `.github/workflows/`：CI 约定。单元测试强制清空 `OPENAI_API_KEY` 使用 mock；集成测试单独跑真实 API。

## 架构与数据流

典型流程是：输入文本 -> 按 `chunk_size` 切块 -> LLM structured output -> 合并/去重 -> AutoType 状态 -> 可选 FAISS 索引 -> search/chat/show/dump/load。

关键行为：

- `parse(text)` 返回新的 Knowledge Abstract 实例，不修改当前实例。
- `feed_text(text)` 修改当前实例，适合增量更新。
- 所有数据变更都应使旧索引失效，通常要调用 `clear_index()` 或走已有状态 hook。
- `build_index()` 后才能使用多数 `search()`/`chat()` 能力。
- `dump(path)` 约定输出 `data.json`、`metadata.json` 和 `index/`。
- LLM batch 可能返回 `None`；新增抽取逻辑时要沿用或等价处理 `_filter_none_results()`，避免合并阶段崩溃。

## 开发环境与常用命令

本项目要求 Python 3.11+，仓库含 `uv.lock`，优先使用 uv。

```powershell
uv sync --all-extras --dev
```

运行离线单元测试时，显式清空 API key，避免误打真实模型：

```powershell
$env:OPENAI_API_KEY = ""
uv run pytest -v
```

按范围运行常用测试：

```powershell
uv run pytest tests/types -v
uv run pytest tests/template_engine -v
uv run pytest tests/cli -v
uv run pytest tests/utils/test_client.py -v
```

真实 API 集成测试只在明确需要时运行，并确保环境变量已配置：

```powershell
uv run pytest -m integration -v --tb=short
```

CI 的 lint 命令是：

```powershell
ruff check hyperextract
ruff format --check hyperextract
```

文档本地预览与构建：

```powershell
uv run mkdocs serve
uv run mkdocs build
```

CLI 快速检查：

```powershell
uv run he --help
uv run he list template --lang en
uv run he list method
```

## 修改规则

优先保持现有分层，不要把模板解析、AutoType 状态、method 算法和 CLI 交互逻辑揉在一起。

修改 AutoType 时：

- 遵循 `BaseAutoType` 的 hook 模式：`_init_data_state()`、`_set_data_state()`、`_update_data_state()`、`_init_index_state()`。
- Pydantic 使用 v2 风格：`model_dump()`、`model_validate()`、`model_copy()`。
- 维护 `metadata["created_at"]`、`metadata["updated_at"]` 的语义。
- 图结构要保证边只引用已存在节点；不要绕过悬挂边裁剪。
- 新增或改变 list/set 操作时，同步测试 Pythonic 行为、schema 校验、索引失效和空状态。

修改模板引擎时：

- YAML 字段类型只支持当前 parser schema 中声明的类型：`str`、`int`、`float`、`bool`、`list`。
- 多语言模板使用 `language: [zh, en]`，并为 description、field description、guideline 提供双语内容。
- `identifiers`、`display` 必须与输出字段一致；graph/hypergraph/time/space 类型还要检查 relation members、time_field、location_field。
- `Gallery` 的模板 key 是 `<domain>/<name>`；不带 slash 时只默认查 `general/<name>`。
- 自定义 YAML 文件可通过路径加载，但预设模板应放在 `hyperextract/templates/presets/<domain>/`。

新增 method 时：

- 优先继承已有 `AutoGraph` 或 `AutoHypergraph`，固定 schema、prompt、key extractor、合并策略和索引字段。
- 在 `hyperextract/methods/registry.py` 注册 name、class、autotype、description。
- method 模板固定使用英语 prompt，`Template.create("method/<name>")` 不需要 language。
- Graph_RAG 的 community detection 依赖可选包 `networkx`/`graspologic`；不要让普通导入或普通测试强依赖这些可选能力。

修改 CLI 时：

- 主生命周期命令在 `hyperextract/cli/cli.py`：`parse`、`feed`、`build-index`、`search`、`talk`、`show`、`info`。
- 配置命令在 `hyperextract/cli/commands/config.py`，读取/写入 `~/.he/config.toml`。
- `OPENAI_API_KEY` 和 `OPENAI_BASE_URL` 是环境变量兜底；不要把密钥写入测试、示例输出或文档。
- CLI 输出使用 Rich/Typer 现有风格；测试使用 `typer.testing.CliRunner`。
- 现有设计不支持全局 `--verbose`，日志由 `HYPER_EXTRACT_LOG_LEVEL` 控制。

修改文档或示例时：

- 双语文档通常需要同步 `docs/en/` 与 `docs/zh/`。
- 新增文档页面要更新 `mkdocs.yml` 对应语言的导航。
- `docs_hooks.py` 过滤 mkdocstrings-autorefs 的重复 primary URL 警告，这是双语 API 文档的预期噪声，不要随意删除。
- 示例可能调用真实模型，除非任务明确要求，不要在验证阶段运行会消耗 API 的示例。

## 测试策略

- 默认单元测试必须可离线运行。`tests/conftest.py` 会在没有有效 `OPENAI_API_KEY` 时使用 `MockChatModel` 与 `MockEmbeddings`。
- 改 `hyperextract/types/`：至少跑 `tests/types`，并按影响范围补充索引、序列化、操作符或 graph consistency 测试。
- 改模板 parser/factory/gallery：跑 `tests/template_engine`，必要时新增最小 YAML fixture。
- 改 client/config：跑 `tests/utils/test_client.py` 和相关 CLI config 测试。
- 改 CLI：跑 `tests/cli`，并用 `uv run he <command> --help` 做一次人工冒烟。
- 改文档：跑 `uv run mkdocs build`。
- 涉及真实抽取质量时，才运行 `pytest -m integration`，并在结果说明中明确这会调用外部 API。

## 编码与风格

- 源码与文档按 UTF-8 处理；当前 PowerShell 终端可能把中文显示成乱码，不代表文件一定损坏。新增中文内容时保持 UTF-8。
- Python 代码遵循 PEP 8、类型注解和 Google 风格 docstring。
- 使用 `pathlib.Path` 处理路径，读写文本显式 `encoding="utf-8"`。
- 日志使用 `hyperextract.utils.logging.get_logger()`，不要散落 `print()` 到库代码；CLI 用户输出除外。
- 不要引入与任务无关的大重构；保持 public API 和 metadata/dump 格式的兼容性。
- 依赖变更才更新 `pyproject.toml` 和 `uv.lock`。
- 不要提交 `.env`、真实 API key、模型输出目录、临时 KA 输出或本地索引产物。

## 常见坑

- `AutoList` 允许重复；需要按 key 去重时用 `AutoSet`。
- `AutoModel.search()` 返回字段字典；图类 `search()` 返回节点/边元组；CLI/调用方不要假设所有 AutoType 的返回形状相同。
- `AutoSet.update()`、集合操作和直接 memory 操作都要留意索引失效。
- FAISS `load_local(..., allow_dangerous_deserialization=True)` 只应加载可信的本地索引。
- `Template.create()` 缺少 language 会对知识模板报错；method 模板例外。
- 修改 YAML `name` 会改变模板 ID，可能破坏文档、示例和用户命令。
- Graph/hypergraph 的 edge key 应稳定；hyperedge 的成员通常要排序，避免同一关系因顺序不同被当作不同边。
- 运行 CLI `parse` 默认会 build index，可能触发 embedding 调用；测试或演示时可用 `--no-index` 降低外部依赖。
