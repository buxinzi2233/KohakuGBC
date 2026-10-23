# GBC 执行机器

`kohakugbc.machine.build(*layers)` 在 Togeari 上构建 GBC 程序，并返回 `(machine, source, sink)`，符合 Togeari 的机器模块契约。模块形式位于 `kohakugbc/machine/module.py`，由 `KOHAKUGBC_CONFIG` 配置。

## 配置

配置层既可以是字典，也可以是 Python 文件；文件中小写的模块级变量构成一个配置层（`kohakugbc.machine.config`）。后面的配置层覆盖前面的配置层，字典按键逐层合并。

| 名称 | 内容 |
|---|---|
| `engines` | 各槽位（`vlm`、`detect`、`embed`、`cpu`）的 `entry` 及入口设置 |
| `machine` | `vlm_waiting`、`max_jobs`（任务准入）、`window`、`start_timeout_s`（单元启动超时，默认 600 秒）、`record`、`store`（`"memory"`）、`report`（RunReport 目录）、`knobs`（其他 Togeari 参数） |
| `source` | `{"kind": "folder" \| "jsonl" \| "trace", "path": ...}` |
| `sink` | `{"kind": "memory"}` 或 `{"kind": "graphs", "path": ...}` |
| `controls` | 被每个任务自己的控制项覆盖的默认值 |

## 槽位与入口

在 `kohakugbc.engines.registry` 中，入口是为槽位注册的工厂。构建时每个槽位指定一个入口并调用一次；所有配置问题会集中呈现为一个错误。槽位提供通道的执行单元、请求构造器和选择器，还会提供适配器（单元根据请求执行该槽位对应的节点）或 `call`（单元调用节点的 Python 函数）。

| 槽位 | 节点 | 可用入口 |
|---|---|---|
| `vlm` | `chat` | `api`、`vllm`（[指南](../guides/engines.md)）、`replay`、`remote` |
| `detect` | `detector` | `torch`（OWLv2 或 SAM 3）、`replay`、`remote` |
| `embed` | `embedder` | `torch`（EmbeddingGemma）、`replay`、`remote` |
| `cpu` | 计算开销较大的普通节点 | `python`（Togeari Python 工作线程或进程）、`remote` |

`replay` 单元是 Togeari 的模拟引擎，根据轨迹和录制延迟返回结果。可选参数包括：`latency_ms`（固定单次延迟）、`speed`（延迟缩放倍数）、每单元的 `speeds`，以及 `units`、`capacity`、`max_batch`、`item_error_rate`、`outages`。当 `unknown="hash"` 时，对于轨迹未记录向量的名称，会产生基于哈希的嵌入，以支持偏离录制路径的运行。单元载荷可以通过 pickle 序列化，所以也能通过引擎宿主运行。`remote` 入口见[跨节点引擎](engine-host.md)。

## 故障处理

| 失败对象 | 任务行为 |
|---|---|
| VLM 请求 | 继续执行并使用空回复，符合 GBC 对失败请求的处理：实体缺失、根节点不命名任何对象或关系没有句子 |
| 检测器或嵌入模型调用 | 任务失败，并给出节点名称及原因 |
| 执行单元（`UnitError`、故障停机、宿主连接丢失） | 尝试在其他单元上重试；如果没有可用单元，则最多等待通道的 `lane_dead_s` 时长 |

## 快照

任务只能在特定边界生成快照：执行窗口为空且只剩一个执行分支。启用提前执行（run-ahead）后，GBC 任务总有推测性工作，因此无法到达该边界。所以使用 `knobs = {"snapshot_every": N}` 时必须设置 `"run_ahead": False`，否则构建会拒绝该配置。

关闭提前执行后，任务大约每 15 条语句创建一个快照，并可从最后一个快照继续。在伪引擎上的运行时间约增加 9%（参见[性能数据](../performance/pseudo-engines.md)）。

## 绑定与策略

| 节点 | 绑定 |
|---|---|
| `chat`、`detector`、`embedder` | 对应槽位的通道；若设置了 `store`，`embedder` 使用内容存储 |
| `open_image`、`crop`、`crop_many`、`image_root`、`seeded_root`、`detected`、`entity_record`、`batch_replies`、`coverage_root`、`relational_queries`、`relational_request`、`relational_record`、`to_gbc` | CPU 通道 |
| 其他所有节点和 `kir.std` | 内联执行 |

准入策略是在 VLM 通道上运行 `TargetLoad`（`vlm_waiting`、`max_jobs`）：通道队列及其单元等待数未超过目标时，允许新的图像任务进入。任务发出策略为 `OldestJobFirst`。

## 测试

`tests/machine` 使用回放引擎运行各组录制的 PoC 任务：每张图都与 PoC 摘要相等，每个录制任务都满足 KIR 参考解释器的一致性检查；当每次请求耗时 50 ms 时，单个任务实际耗时不足其请求耗时之和的一半（同一批区域并行运行）；30 个任务使容量为 8 的 VLM 通道总耗时保持在理论下限的 1.4 倍以内。

`tests/machine/test_configurations.py` 测试三 VLM 单元、同进程与跨进程宿主远程单元、运行中单元故障、VLM 请求或检测器故障、快慢单元共存、单元中途加入或退出，以及运行停止后通过快照恢复等情形。凡成功结束的任务，其结果图均与 PoC 相同。`scripts/bench/pseudo_machine.py` 测量[性能章节](../performance/pseudo-engines.md)所述的配置。
