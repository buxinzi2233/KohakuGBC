# 运行轨迹（Traces）

轨迹会记录一次运行中的每个模型调用，并按照提问内容生成键（`kohakugbc.nodes.specs.trace`）。程序可以回放轨迹而不运行模型：既能在 KIR 参考解释器（`kohakugbc.nodes.specs.replay.ReplayModels`）下运行，也能在执行机器的伪引擎上运行。

## 格式

一个轨迹目录包含 `meta.json`（数据源、设置、`image_root`；相对路径以轨迹目录为基准）和按记录类别拆分的 JSONL 文件。

| 文件 | 每行代表 | 字段 |
|---|---|---|
| `jobs.jsonl` | 一张图像的任务 | `key`、`image`、`inputs`（描述、标签、控制项）、原始实现生成的 `records` 和 `graph`、`rounds` |
| `vlm.jsonl` | 一次 VLM 请求 | `key`、`request`（系统提示词、查询、图像尺寸、Schema、最大 token 数、采样）、`reply`、`error`、延迟、首 token 时间、用量 |
| `detect.jsonl` | 一次检测器调用 | `key`、`size`（裁剪图像像素大小）、`params`（每段文本的 NMS 阈值、高置信度容差、面积上限、top-k）、`raw`（像素坐标框、得分、文本索引）、`selected`（每段文本保留的框） |
| `embed.jsonl` | 编码后的名称 | `text`、`vector`（float32，base64） |
| `same.jsonl` | 是否同一对象的判断 | `a`、`b`、`same`、`score` |

## 键

键是请求语义的规范化 JSON：包括节点、提示词集合、图像任务，以及具体询问内容。

| 节点 | 键字段 |
|---|---|
| `image_query` | 图像 |
| `seed_elements` | 图像、来源（`text` 或 `image`）、标签提示 |
| `coverage` | 图像、根节点中的实体名称 |
| `entity` | 图像、实体 ID、边界框 |
| `entity_batch` | 图像、各区域的 `[entity id, box]` |
| `relation` | 图像、被提问顶点、对应框、每个实体的 `[entity id, text]` |
| `composition` | 图像、组合的 ID（首先是被提问的那个）、每个组成部分的 `[part id, text]` 与框 |
| `detect` | 图像、父级边界框（`None` 表示整图）、文本 |

边界框的四个坐标会纳入键。关系和组合的键必须保留框，因为区域合并会让框扩大；组合键必须保留全部 ID，因为其他短语可能再次命名同一组组成部分。两种情况下，重新提出的问题都是不同请求。

`Trace.conflicts` 列出同一个键被记录两次且回复不同的情况；完整的键应当不存在此类冲突。

## 录制 PoC

`scripts/trace/record_poc.py` 在 PoC 环境中运行 `run_iterative.py` 并写入轨迹；`scripts/trace/poc_arms.py` 录制用于验证程序的实验组。录制器用当前程序的规则（`kohakugbc.nodes.specs.draw`）替换 PoC 中未设种子的随机选择：

| 随机选择 | 生成方式 |
|---|---|
| 关系查询中要询问多少条关系 | `rng("relation-count", image, vertex).randint(2, n)` |
| 用于命名一组子节点的短语 | `rng("relational-text", image, vertex, *phrases).choice(phrases)` |
| 组合根的判定及树中各边的对角方向措辞 | `rng("composition", image, composition id)`：先 `choice`，再按树的顺序对每条边调用 `random()` |
| 关系句子 | 保留首次出现者，按出现顺序去重 |

`scripts/trace/subset.py` 可从轨迹中截取任务及其对应图像，保存到 `tests/fixtures/traces`。
