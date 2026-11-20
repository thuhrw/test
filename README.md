# 🤖 AgentForge：面向 MCP 工具的 Agent 任务生成与训练数据闭环

AgentForge 是一个用于构造、验证和迭代 Agent 训练数据的工具链。项目围绕 MCP 工具环境，将任务能力抽象、任务合成、教师模型 rollout、结果评判和失败反馈串成一条可断点续跑的流水线，适合批量生成高质量的工具调用任务与 SFT 数据。

## ✨ 主要能力

- 🧩 **能力抽象**：从 benchmark 任务中提取可迁移的求解结构（archetype），减少按领域或实体名称进行的重复分类。
- 🏭 **任务合成**：结合能力描述、MCP server、persona 和可控世界状态，自动生成带 world、工具注册表和 manifest 的任务包。
- 🧪 **任务验证**：对生成任务执行 schema、契约、世界状态和一致性检查，失败任务可自动重建或记录。
- 🧑‍🏫 **教师 Rollout**：让模型在真实工具环境中完成任务，保存消息、工具调用轨迹、最终答案和 judge 结果。
- 🔁 **反馈闭环**：读取外部评测的 JSONL 记录，诊断失败原因，分析能力供需，并生成下一轮 capabilities、variants 和 curriculum。
- ⚡ **工程化运行**：CLI 和 Shell 脚本支持并发、断点续跑、配额补缺、缓存和 API 重试。

## 🚀 快速开始

### 1. 安装

项目要求 Python 3.10 或更高版本。推荐使用已有虚拟环境和 `uv`：

```bash
git clone <your-repository-url>
cd AgentForge
uv sync
uv pip install -e . --no-build-isolation --no-deps
```

也可以使用 pip：

```bash
pip install -e .
```

依赖包括 OpenAI 兼容客户端、Jinja2、Requests、NumPy、PyYAML、FileLock、Tenacity 和 JSON Schema；需要 Transformers 时安装 `pip install -e '.[sft]'`。

### 2. 配置模型接口

复制环境变量模板并填写 API 地址、模型和密钥文件路径：

```bash
cp .env.example .env
```

所有脚本均使用 OpenAI 兼容接口。密钥通过文件传入，避免直接写入命令行；合成模型和 world 模型可以使用不同的 endpoint。也可以在执行命令时直接传入 `--base-url`、`--model` 和 `--api-key-file`。

### 3. 能力拆分

输入任务目录、JSON/JSONL 或 CSV，生成能力池：

```bash
export OPENAI_API_KEY=sk-...
uv run python -m capability_split.run_split \
  --benchmark mcp-atlas \
  --tasks /path/to/tasks.csv \
  --field PROMPT \
  --out outputs/capabilities.json
```

输出数组中的每项包含 `archetype`、`archetype_desc` 和 `verfiy` 三个字段。详细输入格式和两阶段归并流程见 [capability_split/README.md](capability_split/README.md)。

### 4. 合成任务

准备 `configs/mcp_pool.json` 和能力文件后运行：

```bash
bash scripts/generate_tasks.sh
```

常用参数可通过环境变量覆盖，例如 `COUNT`、`WORKERS`、`SEED_START`、`SYNTHESIS_MODEL`、`AGENTFORGE_WORLD_MODEL` 和 `AGENTFORGE_OUTPUT_ROOT`。直接调用 Python 入口时：

```bash
python -m agentforge.task_gen.generate_tasks \
  --tool-info configs/mcp_pool.json \
  --capabilities outputs/capabilities.json \
  --output-dir outputs/task_gen \
  --count 100 \
  --workers 10 \
  --base-url "$LLM_BASE_URL" --model "$SYNTHESIS_MODEL" \
  --api-key-file "$AGENT_API_KEY_FILE" \
  --world-base-url "$WORLD_BASE_URL" --world-model "$WORLD_MODEL" \
  --world-api-key-file "$WORLD_API_KEY_FILE"
```

生成目录通常包含 `tasks/`、`server_selection/`、world 资源和进度日志。`--servers` 可固定 MCP server；省略时系统会根据能力缺口选择锚点和伴随 server。

### 5. Rollout 与评判

让教师模型完成生成的任务，并执行过程奖励、答案等价性和 judge：

```bash
bash scripts/rollout_tasks.sh
```

单独运行某一步可使用 `python -m agentforge.task_infer.rollout_tasks --stage infer|judge|all`。默认跳过已完成结果，适合中断后继续运行；`scripts/debug_infer_one_task.sh` 可用于单题调试。

### 6. 反馈闭环与配额补缺

外部评测结果应整理为 `records.jsonl`，每行至少包含 `record_id`、`task.instruction`、`trajectory` 和 `evaluation`。运行：

```bash
bash scripts/run_feedback.sh
```

反馈目录会生成失败诊断卡、供给画像、审计报告、下一轮能力池和课程配置。需要严格补齐各能力数量时运行：

```bash
bash scripts/fill_quota.sh
```

## 📁 项目结构

```text
AgentForge/
├── src/agentforge/task_gen/       # 能力驱动的任务合成、world 与校验
├── src/agentforge/task_infer/     # 教师模型 rollout、过程奖励与 judge
├── src/agentforge/feedback/       # 失败诊断、供需审计与下一轮配置
├── src/agentforge/utils/          # LLM、配置、消息和路径工具
├── capability_split/              # benchmark 能力抽取与归并
├── prompts/                       # 合成、推理、审计和 MCP 系统提示词
├── configs/mcp_pool.json          # MCP server 与工具池
├── scripts/                       # 端到端运行及调试脚本
├── data/                          # persona 等输入数据
└── outputs/                       # 本地生成产物（默认不纳入版本控制）
```

## 📦 任务与评测记录

任务以 JSON bundle 形式保存，包含自然语言任务、能力/领域元数据、world 引用和校验 manifest；rollout 结果保存完整对话及工具调用轨迹。反馈模块兼容外部评测器生成的标准 JSONL，因此可以把 AgentForge 作为数据生成器接入现有 benchmark 或训练流水线。
