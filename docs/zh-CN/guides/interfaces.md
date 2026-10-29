# 接口参考：Python、CLI 与数据契约

[中文文档](../README.md) · [使用指南](README.md) · [English documentation](../../README.md)

本页以当前源码为准，补齐已有[引擎指南](engines.md)、[执行机器](../internals/machine.md)、
[图解析](../internals/graph.md)和[轨迹格式](../internals/traces.md)中的调用契约，不另造一套架构文档。
面向运行 GBC、接入数据、消费图结果和编写引擎的调用者；内部节点算法、性能脚本以及
`src/kohakugbc_models/vlm/qwen35` 的实现不作为本包稳定 API 逐项罗列。这里描述现有行为，不承诺额外的版本兼容保证。

## 1. 入口与导出范围

Python 导入根为 `kohakugbc`；根包提供 `__version__: str`（未安装分发元数据时为 `"0+unknown"`），
并不在根包重导出 `build`。以下显式导出共 30 项（按模块计，包括类型、常量和重导出）：

| 导入模块 | 显式导出 | 本页对应内容 |
|---|---|---|
| `kohakugbc.machine` | `CPU_NODES`, `Parts`, `bindings`, `build`, `build_parts`, `lanes_of` | 构建与生命周期 |
| `kohakugbc.program` | `CHOICES`, `DEFAULTS`, `ENTRY`, `PROGRAM`, `complete`, `text` | 程序与控制项 |
| `kohakugbc.graph` | `ResolutionError`, `canonical`, `digest`, `from_node_info`, `python_text`, `record`, `ref`, `resolve`, `rust_text`, `stats`, `to_text` | 图转换 |
| `kohakugbc.nodes.library` | `MODULES`, `Models`, `build` | 节点库与模型契约 |
| `kohakugbc.engines.host` | `EngineHost`, `RemoteUnit`, `authkey` | 远程引擎 |
| `kohakugbc.engines.replay` | `models_of` | 回放 |

`kohakugbc.io`、`kohakugbc.nodes`、`kohakugbc.engines` 的包初始化文件不重导出上述子模块入口；
请按下文的完整模块路径导入。签名中没有类型标注的参数，在说明中给出实际使用类型，不意味着源码进行了运行时类型校验。

## 2. 执行机器、配置与生命周期

依据：[构建实现](../../../src/kohakugbc/machine/build.py)、[配置层](../../../src/kohakugbc/machine/config.py)、
[机器模块](../../../src/kohakugbc/machine/module.py)。

```text
from kohakugbc.machine import build, build_parts, Parts, lanes_of, bindings, CPU_NODES
build(*layers) -> tuple
build_parts(*layers) -> Parts
Parts(machine: Machine, source: object, sink: object, lanes: dict, slots: dict)
lanes_of(slots: dict) -> dict[str, Lane]
bindings(lanes: dict, slots: dict, store: bool) -> dict

from kohakugbc.machine.config import load, layer, merge
load(*layers) -> dict
layer(item) -> dict
merge(base: dict, top: dict) -> dict
```

- `layers` 按顺序接受字典或 Python 配置文件路径（`str` / `Path`）；后层覆盖前层，嵌套字典递归合并，其他值（包括列表）整体替换。没有“删除键”的特殊语法。
- 文件通过 `runpy.run_path` **执行**，只收集名字满足 `islower()` 且不以 `_` 开头的模块级变量。不是 JSON/YAML，也不是沙箱；只加载可信文件。配置中的普通相对路径不自动相对配置文件重定位，按使用它的 API/当前工作目录解释。
- `load` 从默认配置叠加；`merge` 返回新字典但不是任意对象的深拷贝，未变的嵌套对象可能共享引用。不要修改导出的默认值来保存某次运行状态。
- `build` 返回 `(togeari.Machine, source, sink)`；`build_parts` 额外暴露以槽位名索引的 `lanes` 和 `slots`。数据源未配置时 `source is None`。
- 构建会导入/调用引擎工厂、检查部署内存、创建 source/sink；不等于纯函数，例如图输出端会创建父目录。正常引擎在机器启动阶段加载模型，第三方工厂仍可能有自己的构建副作用。
- `lanes_of` 接受 `dict[str, Slot]`，`Slot.lane` 优先于工厂的 `former` / `select`。`bindings` 需要 `vlm/detect/embed/cpu` 通道；`store=True` 使嵌入节点使用内容存储。缺失键会产生 `KeyError`。`CPU_NODES: tuple[str, ...]` 是重计算普通节点的 CPU 绑定名单，不是用户需维护的配置。

实际运行示例（从仓库根目录，需先安装对应引擎并准备图像；会运行模型）：

```python
from kohakugbc.machine import build

machine, source, sink = build(
    "configs/machine/qwen35_local.py",
    {"source": {"kind": "folder", "path": "images"}},
)
with machine:
    summary = machine.run(source, sink)
```

这是同步调用，不应 `await build(...)` 或 `await machine.run(...)`。`Machine` 是 Togeari 的对象，
其运行摘要和调度方法由 Togeari 定义，不是 KohakuGBC 自有返回模型。按示例使用上下文管理器结束机器资源；
直接管理机器时调用 `machine.stop()`。`gbc check` 使用 `togeari.no_unit_start()` 构建后停止机器，
但仍执行配置文件、工厂和输出端构建，不能拿它验证不可信代码。

构建的主要失败：引擎配置问题合并为 `ValueError`；部署内存不满足、未知 store、
启用 `snapshot_every` 却未设置 `run_ahead=False` 也会拒绝。配置文件的 `OSError`、
执行异常、依赖导入异常及 Togeari 检查异常可直接传播，API 不统一包装所有失败。

### 配置顶层及默认值

| 顶层 | 默认值 / 接受形式 |
|---|---|
| `engines` | 仅默认配置 `cpu={"entry": "python", "workers": 8, "mode": "thread"}`；`vlm/detect/embed` 必须另配 |
| `machine` | 下表；`knobs` 传给 Togeari `Machine` |
| `source` | `None`；或 `{"kind": "folder" / "jsonl" / "trace", "path": str}`；trace 可加 `repeat`（默认 1，构建时转 `int`） |
| `sink` | `{"kind": "memory"}`；或 `{"kind": "graphs", "path": str}` |
| `controls` | `{}`；被单个任务的 controls 覆盖，再由程序补齐默认值 |

| `machine` 字段 | 使用类型 / 默认值 | 行为 |
|---|---|---|
| `vlm_waiting`, `max_jobs`, `window` | `int`: 64, 512, 64 | VLM 等待目标、最大任务数、执行窗口 |
| `start_timeout_s` | 秒数: 600 | 执行单元启动超时 |
| `record`, `report` | 默认 `None` | `record` 原样传给 Togeari；`report` 为 RunReport 目录路径 |
| `store` | `None`, `"memory"` 或 store 对象；默认 `None` | 其他字符串/字典报 `ValueError` |
| `knobs` | `dict`, `{}` | 额外 Togeari 参数；不要重复传已由构建器指定的参数 |
| `placement_headroom` | 浮点数, 0.05 | 部署检查预留比例（构建时读取的缺省值） |

每个 `engines.<slot>` 包含 `entry` 及该入口参数；具体模型配置沿用[引擎指南](engines.md)。
槽位是 `vlm`、`detect`、`embed`、`cpu`。内置注册组合为 vlm 的 `api/vllm/replay/remote`、
detect/embed 的 `torch/replay/remote`、cpu 的 `python/remote`；vLLM/torch 是否能注册取决于可选依赖。
插件可增加入口，不能把 `doctor` 输出的安装环境清单当成跨环境固定清单。

Togeari 机器模块入口为 `kohakugbc.machine.module.build()`（无参数），返回同样的三元组；
配置读取 `KOHAKUGBC_CONFIG`，按 `os.pathsep` 分割路径（POSIX 为 `:`，Windows 为 `;`），忽略空项。
它与直接传字典的 `kohakugbc.machine.build(*layers)` 是不同入口。

## 3. 程序、任务输入与控制项

依据：[程序入口](../../../src/kohakugbc/program/__init__.py)、[控制项](../../../src/kohakugbc/program/controls.py)、
[根节点校验](../../../src/kohakugbc/nodes/root.py)、[图像输入](../../../src/kohakugbc/nodes/images.py)。

```text
from kohakugbc.program import complete, text, PROGRAM, ENTRY, DEFAULTS, CHOICES
complete(controls: dict | None) -> dict
text() -> str
```

`PROGRAM: Path` 指向随包分发的 `gbc.kir`，`ENTRY == "gbc"`；`text()` 以 UTF-8 读取它，
文件问题传播 `OSError`。KIR 入口是 `gbc(image, captions=None, tags=None, controls=None)`，输出 `graph, stats`。
这不是一个同名 Python 函数；在机器上运行，或使用下文参考解释器。

| 任务输入 | 契约 |
|---|---|
| `image` | 必需，图像路径字符串或 `{"path": str, "key": str?}`。内部补充 `size=[width, height]`；缺省 key 为文件名。Pillow 读取图像，像素转 RGB |
| `captions` | `dict` 或 `None`；种子模式使用 `detail: str`，可选 `short: str` |
| `tags` | `dict` 或 `None`；使用 `general: dict[str, float]`，分数按 `tag_threshold` 筛选，不是标签字符串列表 |
| `controls` | `dict` 或 `None`；见下表 |

`root != "query"` 时要求有非空 `captions["detail"]`；`root == "tags"` 还要求 tags 为真值，
不满足时节点抛 `ValueError`。图片不存在、不可解码或输入缺键会在读取/执行时失败，不会自动跳过。

`DEFAULTS: dict` 的 21 项如下；`CHOICES: dict[str, tuple]` 只约束下表列出的枚举：

| 名称 | 默认值 | 使用类型 / 枚举 |
|---|---|---|
| `root` | `"query"` | `query`, `text`, `image_text`, `tags` |
| `min_nodes`, `max_nodes`, `max_rounds` | 8, 20, 4 | `int`，节点预算和轮数 |
| `round_size` | `None` | `int` 或 `None` |
| `priority` | `"area_depth"` | `area_depth`, `area`, `deep` |
| `max_breadth_rounds`, `coverage_max_new` | 1, 4 | `int` |
| `max_top_level`, `max_features_per_entity`, `max_instances_per_text`, `max_depth` | 6, 4, 6, 3 | `int` |
| `entity_batch` | 1 | `int`，至少 1 |
| `leaf_caption` | `"query"` | `query`, `parent` |
| `relational` | `"both"` | `both`, `relation`, `composition`, `none` |
| `relate` | `"end"` | `end`, `every_round` |
| `max_side` | 1568 | `int`，VLM 图片最长边像素数 |
| `same_threshold`, `overlap_threshold` | 0.725, 0.85 | 浮点数 |
| `tag_threshold`, `tag_hint` | 0.35, `False` | 浮点数、`bool` |

`complete` 返回新的字典；未知名称、非法枚举、`min_nodes > max_nodes` 或 `entity_batch < 1`
抛 `ValueError`。它不是完整的类型/范围校验器：例如错误的比较类型可能产生 `TypeError`，
其他数值并未在这里全部检查为非负。不要把表中的建议使用类型误当作完整 JSON Schema。

```python
from kohakugbc.program import complete

controls = complete({"root": "text", "max_nodes": 12})
assert controls["min_nodes"] == 8
assert controls["max_nodes"] == 12
```

## 4. 数据源、输出端与文件格式

依据：[sources.py](../../../src/kohakugbc/io/sources.py)、[sinks.py](../../../src/kohakugbc/io/sinks.py)。

```text
from kohakugbc.io.sources import Folder, Jsonl, TraceJobs
Folder(root: str, controls: dict | None = None, patterns=IMAGES)
Jsonl(path: str, controls: dict | None = None)
TraceJobs(path: str, controls: dict | None = None, repeat: int = 1)
source.shard(rank: int, world: int)  # 返回浅拷贝的数据源
source.items()                    # 迭代 (key, inputs)
iter(source)                      # 迭代 togeari.SourceItem(key, inputs)

from kohakugbc.io import sources, sinks
sources.build(spec: dict | None, controls: dict)
sinks.build(spec: dict | None)

from kohakugbc.io.sinks import GraphJsonl
GraphJsonl(path: str)
sink.write(record) -> None
sink.close() -> None
```

- `Folder` 递归、去重并排序路径，`IMAGES=("*.jpg", "*.jpeg", "*.png", "*.webp", "*.bmp")`；
  大小写匹配遵循平台，不保证匹配大写扩展名。key 为相对 root 的 POSIX 路径。配置工厂不转发自定义 patterns；需直接构造 `Folder`。
- `Jsonl` 以 UTF-8 逐行读取 JSON，忽略空行；每行必需 `image` 路径，可选 `key/captions/tags/controls`。
  相对 image 相对 JSONL 所在目录，缺省/空 key 回退为图像文件名。不同目录的同名文件应显式指定不同 key。
- `TraceJobs` 使用录制输入（包含 root 与 PoC 控制名转换）；`repeat > 1` 时 key 为 `<key>#<n>`，n 从 0 开始。
  它构造时就读取轨迹，不是惰性读取全部文件。`repeat` 不在类内验证为正值。
- `iter(source)` 将配置 controls 与每任务 controls 浅合并，后者优先；`items()` 本身不执行这一步。
  `shard` 按枚举序号 `n % world == rank` 过滤，不复制图像；调用者需保证 `world > 0` 且 `0 <= rank < world`。
- 未指定 source kind 返回 `None`；未知 kind 抛 `ValueError`。缺少 path、坏 JSON、文件权限等错误分别传播
  `KeyError`、`JSONDecodeError`、`OSError`，并不转成逐行失败记录。

JSONL 最小种子任务（`image` 是用户准备的真实文件路径；本例没有创建该文件）：

```json
{"image":"images/cat.jpg","key":"cat-001","captions":{"detail":"A cat sits on a chair."},"controls":{"root":"text"}}
```

`GraphJsonl.write` 接受 Togeari 的结果记录对象（读取 `.ok/.outputs/.key/.failure`），不是图字典。
`outputs["graph"]` 必须是 JSON **字符串**或 `None`，`outputs["stats"]` 是字典。对于 `result.jsonl`：

| 文件 | 每行内容 |
|---|---|
| `result.jsonl` | 成功且非空的 graph JSON 原文；不额外包一层 key |
| `result.stats.jsonl` | `{"key": ..., **stats}` |
| `result.failures.jsonl` | `{"key": ..., **failure}`；无图且无 failure 时为 `kind="empty", message="no described record"` |

文件以 UTF-8 **追加**，每条记录有换行；不清空旧文件、不去重、不提供跨文件事务或重跑幂等性。
每个实例有线程锁，每次写入单独打开并关闭文件，`close()` 是空操作；锁不保证多实例/多进程共同写文件安全。
`counts={"ok": 0, "failed": 0}` 只计当前对象写入次数，空图也计 failed。写入 `OSError` 直接传播；
图已写入但 stats 写入失败时可能只留下部分输出。`sinks.build(None)` / 空字典默认返回 Togeari `Memory()`，
显式 `kind="memory"` 也一样；未知 kind 抛 `ValueError`。

轨迹目录的既有逐文件字段见[轨迹格式](../internals/traces.md)，读取依据
[trace.py](../../../src/kohakugbc/nodes/specs/trace.py)：`meta.json` 必需，五类 JSONL 文件不存在时视为空；
均为 UTF-8。`image_root` 相对轨迹目录，图像路径继续拼接 job 的 image。VLM/检测重复 key 保留第一条；
jobs/embed 的字典构造保留最后一条。`vector` 是 NumPy `float32.tobytes()` 的 base64 ASCII，
未额外固定跨架构字节序或加入格式版本头。不要将它误认为可任意迁移的模型权重格式。

## 5. 图转换、摘要与 Rust 边界

依据：[导出与选择器](../../../src/kohakugbc/graph/__init__.py)、[记录](../../../src/kohakugbc/graph/records.py)、
[Python 解析](../../../src/kohakugbc/graph/reference.py)、[统计](../../../src/kohakugbc/graph/orders.py)、
[摘要](../../../src/kohakugbc/graph/digest.py)。

```text
from kohakugbc.graph import (
    ref, record, from_node_info, resolve, to_text, python_text, rust_text,
    stats, canonical, digest, ResolutionError,
)
ref(entity_id: str, label: str, text: str | None) -> list
record(ids: list, box: list | None, descs: list, children: list) -> dict
from_node_info(dump: dict) -> dict
resolve(records: list[dict], img_path: str | None,
        img_size: list | None = None, threshold: float = 0.85) -> dict | None
python_text(records: list[dict], img_path: str | None,
            img_size: list | None = None, threshold=0.85) -> tuple[str | None, dict]
rust_text(records: list[dict], img_path: str | None,
          img_size: list | None = None, threshold=0.85) -> tuple[str | None, dict]
to_text = python_text 或 rust_text
stats(graph: dict | None) -> dict
canonical(graph: dict | str | None) -> str
digest(graph: dict | str | None) -> str
```

- `ref` 返回 `[entity_id, label, text]`，`record` 返回 `ids/box/descs/children` 四键字典，二者不做验证或深拷贝。
  `from_node_info` 接收 ml-gbc `NodeInfo.model_dump()` 的字典，不接收实例；读取 `input_dict`、`query_result`，
  转换 bbox 坐标为 float，缺键/结构不符的原生异常直接传播。
- record 的 `ids` / `children` 是 ref 列表；`box` 是整图归一化的 `[left, top, right, bottom, confidence]`，
  `None` 表示全图；`descs` 为 `[[text, desc_label, [entity_id, ...]], ...]`。
  顶点标签为 `image/entity/composition/relation`；描述标签为 `short/relation/composition/detail/original/hardcode/bagofwords`。
- `resolve` **始终是 Python 参考实现**，返回字典或 `None`；`img_path` 必须显式传入，可为 `None`，
  只是结果元数据，不会读取图片；`img_size` 推荐传 `[width, height]` 整数列表或 `None`。
- `to_text` 返回 `(JSON文本或None, stats字典)`，不返回图字典。导入时根据 `KOHAKUGBC_RESOLVE=python|rust`
  选择一次：有扩展默认 Rust，否则 Python。指定不可用的 rust 或未知值会使导入抛 `ImportError`，
  导入后修改环境变量不改变已选函数。
- 所有解析调用都是同步。无描述的记录不形成顶点，全部无描述时返回 `None`；不是空 `vertices` 图。
  `ResolutionError` 继承 `ValueError`，用于重复实体 ID、无 ID 的有描述记录、非法描述标签、
  关系指向不存在顶点、child text 为 `None` 等语义错误。任意坏字典也可能直接抛 `KeyError/TypeError/IndexError`，
  不保证全部归一为 `ResolutionError`。
- `canonical` 将 JSON 字符串先解析，再输出保留 Unicode 的紧凑 JSON，**不排序字典键**；
  `digest` 是该文本 UTF-8 字节的 SHA-256 十六进制字符串，不是文件/图片摘要。
  坏 JSON 传播 `JSONDecodeError`，不可序列化对象传播 `TypeError`。

无需模型或图片的最小调用：

```python
import json

from kohakugbc.graph import digest, python_text, record, ref

records = [record([ref("", "image", None)], None, [["A cat.", "short", [""]]], [])]
graph_text, counts = python_text(records, img_path=None, img_size=[640, 480])
assert graph_text is not None
assert counts["vertices"] == 1
assert json.loads(graph_text)["short_caption"] == "A cat."
assert len(digest(graph_text)) == 64
```

输出图与 ml-gbc `GbcGraphFull.model_dump()` 字段对应：

| 层级 | 字段 / 类型 |
|---|---|
| 图 | `vertices: list[dict]`, `img_url: None`, `img_path: str或None`, `original_caption/short_caption/detail_caption: str或None`, `img_size: [int,int]或None`, `mask_inside_threshold: float` |
| 顶点 | `vertex_id: str`, `label: str`, `bbox: dict`, `descs: list[dict]`, `in_edges/out_edges: list[dict]`, `sub_masks/super_masks: list[str]` |
| bbox | `left/top/right/bottom: float`, `confidence: float或None` |
| 边 | `source/text/target: str` |
| 描述 | `text/label/full_label: str`, `statistics: None`, `clip_scores={"scores": None, "truncation": False}`, `toxicity_scores: None` |

`stats(graph)` 返回 `vertices`、`edges`、`captions` 数量，`labels`（各顶点标签计数）、BFS `depth`、
`roots` 和 `leaves` 数量。`stats(None)` 只有 `vertices=0, labels={}, edges=0, captions=0, depth=-1`，
**不含 roots/leaves**；消费者不能无条件索引这两个键。遍历的顺序语义见[图解析](../internals/graph.md)。

### 原生模块不是另一套高层 API

[模块注册](../../../src/kohakugbc-rust/lib.rs)实际只注册两项：

```text
from kohakugbc import _rs
_rs.version() -> str
_rs.resolve(records, img_path, img_size=None, threshold=0.85) -> tuple[str | None, dict]
```

`version()` 返回 Cargo 包版本；`resolve` 接受相同 record 形状，`img_path` 是字符串或 None，
`img_size` 需为两项整数序列（Rust `Vec<i64>`），`threshold` 为浮点数。
[PyO3 边界](../../../src/kohakugbc-rust/graph/python.rs)先提取 Python 数据，再在释放 GIL 后进行解析、
JSON 序列化和统计，最后返回 Python 元组；没有需用户释放的原生句柄，也不是 awaitable。
尺寸长度不为 2、语义/序列化失败产生 `ValueError`；类型/字段提取可产生其他 Python 异常。
`graph.rust_text` 仅把 `ValueError` 转成 `ResolutionError`，不是全异常包装；未构建扩展时不要直接调用它
（该函数没有缺失扩展的友好检查）。优先使用 `to_text` 或显式 `python_text`。

## 6. 集成与引擎扩展

### 节点库及参考解释器

依据：[节点库导出](../../../src/kohakugbc/nodes/library.py)、[模型 Protocol 与构建器](../../../src/kohakugbc/nodes/registry.py)、
[参考解释器](../../../src/kohakugbc/program/reference.py)。

```text
from kohakugbc.nodes.library import Models, build, MODULES
build(models: Models | None = None, name: str = "gbc") -> NodeLibrary
Models.chat(self, request: dict) -> str
Models.detector(self, patch, request: dict) -> dict
Models.embedder(self, text: str)

from kohakugbc.program.reference import run
run(inputs: dict, models, step_budget: int | None = None) -> dict
```

`MODULES` 是注册普通节点的模块元组；`build` 返回 KohakuNodeIR `NodeLibrary`。
不提供 models 时模型节点是需要机器绑定的占位函数，直接执行会抛 `NotImplementedError`。
`Models` 是同步 Protocol，不负责模型启动/关闭；提供者自行管理资源。
`run` 在 KIR 参考解释器中同步执行单张图，返回 `{"graph": JSON字符串或None, "stats": dict}`，
`step_budget` 直接传给解释器；未处理的解析/执行异常由依赖传播。不是批量/异步执行机器的替代品。
[GBC 程序的 ask](../../../src/kohakugbc/program/gbc.kir) 会捕获 VLM chat 失败并用空回复继续，
而检测器/嵌入失败会使任务失败；不要把“产生了图”当作所有模型调用均成功的证明。

模型调用载荷（不是公开 HTTP REST 图生成接口）：

- `chat` 请求含 `key/system/query/images/schema/max_tokens/temperature`。
  `images` 是 `[label, picture]` 列表，picture 为 `{"jpeg": bytes, "size": [w,h]}` 或空裁剪的 `None`；
  schema 为随包 schema 名或 `None`。返回原始回复字符串。
  依据：[chat specs](../../../src/kohakugbc/nodes/specs/chat.py)。
- `detector` 的 patch 为 RGB NumPy 数组，请求含 `key/texts/box`，返回
  `{"boxes": [[l,t,r,b], ...], "scores": [...], "labels": [文本索引, ...]}`；
  raw boxes 使用**裁剪图像的像素坐标**，不是 record 的归一化整图坐标。
  依据：[detect specs](../../../src/kohakugbc/nodes/specs/detect.py)。
- `embedder` 返回名称的归一化向量；回放缺失时可返回 `None`。不假定所有引擎有固定维数。

### 引擎注册

依据：[registry.py](../../../src/kohakugbc/engines/registry.py)、[builtin.py](../../../src/kohakugbc/engines/builtin.py)。

```text
from kohakugbc.engines.registry import Slot, entry, build_slot, build
Slot(units: list, former=None, select=None, adapter=None, call=None, lane=<新dict>, needs=<新list>)
entry(slot: str, name: str)  # 返回装饰器
build_slot(slot: str, settings: dict | None, shared: dict) -> Slot
build(engines: dict) -> dict[str, Slot]
```

`@entry(slot, name)` 注册同步工厂 `(settings: dict, shared: dict) -> Slot` 并返回原工厂；
同名注册会替换旧工厂，未知槽位报 `ValueError`。`shared` 是每次四槽位构建共用的字典。
`settings` 传入工厂前复制并移除 `entry`；缺少入口或未注册时报 `ValueError`。
工厂抛 `TypeError/ValueError/KeyError/OSError` 时会加上槽位/入口名包装为 `ValueError`，四槽位 build 汇总这些错误。

`Slot.units` 必需，其余默认为 None 或独立空容器；`call` 是可调用对象，`lane` 是通道设置字典，
`needs` 是部署需求列表。`adapter=None` 表示执行节点 Python 函数并经 `call` 调模型，
否则采用适配器转换；执行单元遵循 Togeari 引擎契约（提交票据、完成回调、容量/负载、取消、启动/停止）。
本页不重复定义 Togeari 的调度协议。插件通过 Python 分发入口组 `kohakugbc.entries` 加载注册模块；
失败记录在 `builtin.PLUGIN_ERRORS`，qwen35 是独立插件示例而非固定内置入口。

### 远程引擎与回放

依据：[host 导出](../../../src/kohakugbc/engines/host/__init__.py)、[宿主](../../../src/kohakugbc/engines/host/host.py)、
[远程单元](../../../src/kohakugbc/engines/host/remote.py)、[wire](../../../src/kohakugbc/engines/host/wire.py)。

```text
from kohakugbc.engines.host import EngineHost, RemoteUnit, authkey
authkey(settings: dict) -> bytes
EngineHost(units: list, address: tuple[str, int], authkey: bytes, heartbeat_s=0.2)
host.start() -> tuple[str, int]
host.serve_forever() -> None
host.stop() -> None
RemoteUnit(address: str, authkey: bytes, kind: str, max_batch: int = 1,
           protocol: str | None = None, name: str | None = None,
           connect_timeout_s: float = 10.0, retry_s: float = 1.0)
```

`authkey` 优先取配置的 `authkey`（`str(value).encode()`），否则读取 `authkey_env` 指定环境变量，
缺省变量名为 `KOHAKUGBC_HOST_KEY`；不存在时报 `ValueError`。编码为 UTF-8。
地址为 `host:port` 或远程单元的 `host:port/unit`，缺失 host 默认本机；端口转整数失败抛 `ValueError`。
`kind` 与宿主匹配（stream/batch/call），不匹配被拒绝。`EngineHost.start()` 启动各单元及后台线程，
返回实际监听地址；`serve_forever()` 阻塞到停止；调用者应在 `finally` 中 `stop()`。
启动过程中的底层异常可传播，不保证部分启动失败时自动回滚全部资源。

`RemoteUnit.start()/stop()` 交由机器生命周期管理；直接使用时由调用者成对管理。
`submit(work, done)` 返回票据，结果由回调交付（不是 asyncio 协程）；断连时未完成票据得到 `UnitError`，
`ready()` 按 retry 间隔尝试重连。启动连接失败最终产生 `ConnectionError`，
`connect_timeout_s` 是重试循环期限，不保证截断底层每次阻塞连接操作。更多行为见[跨节点引擎](../internals/engine-host.md)。

通信是认证的 `multiprocessing.connection` + pickle，不是 HTTP/JSON，也没有这里实现的 TLS；
认证不把不可信 pickle 变安全。只允许可信持钥对端和受控网络访问，不要暴露到公网。

```text
from kohakugbc.engines.replay import models_of
models_of(trace: str | Path) -> ReplayModels
```

依据：[进程内回放缓存](../../../src/kohakugbc/engines/replay/entries.py)、[ReplayModels](../../../src/kohakugbc/nodes/specs/replay.py)。
按绝对轨迹目录缓存对象，同进程重复调用返回同一对象，计数/未命中列表也共享；没有公开缓存释放方法。
读取错误传播。chat/detector 未命中抛 `ReplayMiss(KeyError)`；embedder 缺失默认返回 `None`。
这一路不运行模型，仍要求有效轨迹及执行任务所需的图像。

## 7. CLI 与回放 HTTP 辅助服务

入口由 [pyproject.toml](../../../pyproject.toml) 注册为 `gbc = kohakugbc.cli.main:main`。
[main(argv: list[str] | None = None) -> int](../../../src/kohakugbc/cli/main.py) 支持：

| 命令 | 参数、默认值和结果 |
|---|---|
| `gbc --version` | 打印 `kohakugbc <版本>` 后 argparse 正常退出 |
| `gbc run CONFIG [CONFIG ...]` | 至少一层；source 缺失时 stderr 提示并返回 2；完成后输出 seconds/summary 的 JSON，返回 0 |
| `gbc check CONFIG [CONFIG ...]` | 至少一层；禁止单元启动，打印 warning 与 `ok: <entries>`，停止机器，返回 0 |
| `gbc doctor [CONFIG ...]` | 可无配置；打印版本、设备内存、注册入口；有配置时检查部署，正常返回 0；部署检查 `ValueError` 被打印并返回 1 |
| `gbc host CONFIG [CONFIG ...] --slot S` | S 必须为 `vlm/detect/embed/cpu`；`--listen=127.0.0.1:7501`，`--authkey-env=KOHAKUGBC_HOST_KEY`；阻塞服务，Ctrl-C 后停止并返回 0 |

所有命令均有 argparse 的 `-h/--help`。参数错误由 argparse `SystemExit(2)` 处理；
`main` 不包住全部运行异常，构建/启动失败可能显示 traceback。`run` 的 0 表示机器调用结束，
**不保证每个任务成功**；应检查 summary 和失败输出。doctor 的入口构建错误在部署检查 try 块外，
不能把所有 doctor 错误都解释为其显式返回的 1。没有 `--config`、`--output`、`--dry-run` 等未实现选项。

辅助命令（并非新增 `gbc` 子命令）：

```text
python -m kohakugbc.engines.replay.server TRACE [--host 127.0.0.1] [--port 8088] [--speed 1.0]
```

依据：[replay/server.py](../../../src/kohakugbc/engines/replay/server.py)。TRACE 为轨迹目录，
port 为 int、speed 为 float；使用正 speed（实现未提前校验零/负值）。服务仅用于回放：

| 请求 | 契约 |
|---|---|
| `POST /v1/chat/completions` | OpenAI 兼容 messages body，回复 `choices[0].message.content` |
| `POST /v1/messages` | Anthropic 兼容 messages body，回复 `content` 文本块和 `stop_reason` |
| `GET /health` 或 `/v1/health` | 200，`{"ok": true}` |
| `GET /v1/models` | 200，`{"data": [{"id": "replay"}]}` |
| `GET /metrics` | 200，vLLM 风格 running/waiting 文本指标 |

POST 必须有能匹配录制请求的 `X-GBC-Request`（请求语义 key 的 UTF-8 SHA-256），
缺失/未知返回 404 error。实现按路径是否以 `/messages` 结尾选回复形式，不严格校验其余 POST 路由；
未知 GET 返回 404。JSON 请求/响应使用 UTF-8；查询文本和图片尺寸不匹配记录到 `mismatches`，并非自动拒绝。
这不是通用模型推理服务，没有认证层和全面的坏请求状态码转换，不应作为生产公网 API。

程序化调试入口为 `ReplayServer(trace: str, speed: float = 1.0, latency_s: float | None = None)`，
导入路径 `kohakugbc.engines.replay.server`；`start(host: str = "127.0.0.1", port: int = 0) -> str`
返回含 `/v1` 的 URL，在后台线程运行；用 `stop() -> None` 关闭。
注意 Python start 默认临时端口 0，而 CLI 默认 8088；`latency_s` 仅 Python 构造器提供，CLI 没有此选项。

## 8. 调用前核对

1. 用 `kohakugbc.machine` 导入构建器，不把 `io` 包本身当成类导出入口。
2. 分清图字典（`resolve`）与图 JSON 文本（`to_text` / KIR / GraphJsonl outputs）。
3. 选择 JSONL/source/trace 时按各自相对路径规则准备数据；root 种子模式需 detail caption。
4. 构建和 `check` 仍执行可信配置；正式运行可能加载模型、下载权重或访问配置的服务。
5. 机器、宿主、回放 HTTP 服务分别由其生命周期关闭；不要把线程回调接口当成 awaitable。

本页覆盖显式导出、直接用户构建/IO/插件入口、两项 Rust 导出、四个主 CLI 子命令和回放服务。
模型参数细节、图解析算法和轨迹字段继续以链接的现有文档及对应源码为依据，不把内部辅助函数全部升级为公共承诺。
