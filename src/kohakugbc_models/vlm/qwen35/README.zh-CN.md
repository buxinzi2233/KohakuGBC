# qwen35

[English](README.md) | **简体中文**

这是面向 Qwen3.5 模型家族（2B、4B、9B）开发的 VLM 引擎，在 KohakuGBC 的 `vlm` 槽位中注册为 `qwen35` 入口。它展示了如何基于 Togeari 的流式引擎契约实现专用引擎：对于执行机器而言，它与 vLLM 或 API 引擎具有相同的单元接口。

## 安装与运行

```
pip install "kohakugbc[qwen35]"
GBC_IMAGES=path/to/images gbc run configs/machine/qwen35_local.py
```

`configs/machine/qwen35_local.py` 将整套机器部署到一张 24 GiB GPU 上：该引擎（`Qwen/Qwen3.5-4B`，或通过 `QWEN35_MODEL` 指向本地目录）、位于同一设备的检测器，以及运行在 CPU 上的嵌入模型。图输出到 `runs/machine/qwen35_local/graphs.jsonl`（可用 `GBC_OUT` 更改）。在 RTX 4090 上，平均每张图约 3 秒生成一个 GBC 图（参见[实测数据](../../../../docs/zh-CN/performance/real-engines.md)）。

配置中的入口写法：

```python
engines = {
    "vlm": {
        "entry": "qwen35",
        "model": "Qwen/Qwen3.5-4B",
        "device": "cuda:0",
        "memory": 0.65,
        "state_dtype": "bfloat16",
    }
}
```

| 设置 | 说明 |
|---|---|
| `model` | 模型检查点目录或 Hugging Face 仓库名称 |
| `device`、`devices` | 每个设备一个执行单元 |
| `memory` | 该单元可占用的设备内存比例，包含权重（默认 0.5） |
| `max_seqs`、`max_model_len`、`page_size` | 同时生成的序列数（64）、每条序列最大 token 数（16384）、KV 缓存页大小（16） |
| `prefill_tokens`、`prefill_min`、`prefill_wait_ms` | 每步的预填充 token 数（4096）；将预填充并入解码步骤的条件（已有 512 个待处理 token，或提示词等待达 50 ms） |
| `snapshots`、`boundaries` | 保留的前缀快照数量（16）；快照位置：`system`（系统提示词后）、`images`（最后一张图像后） |
| `state_dtype` | DeltaNet 状态的数据类型：`float32`（默认）或 `bfloat16`（显存占用减半） |
| `any_whitespace` | JSON 回复是否允许在任意位置包含空白字符，与 vLLM 的 Schema 编译方式一致（True）；False 则保持紧凑 |
| `jump`、`jump_max` | 对语法约束确定的 token 无须前向计算直接跳过：`exact`（唯一允许 token）、`string`（强制文本重新分词）、`off`；一次最多 16 个 token |
| `cuda_graphs` | 解码步骤回放事先捕获的 CUDA 图（True） |
| `vision_patches` | 每次视觉塔调用处理的图像 patch 数（32768） |
| `prefer`、`race` | 算子后端偏好顺序；是否通过计时竞速选择最快实现 |
| `prep_workers`、`capacity`、`seed` | 数据准备线程数（4）、在途请求数（`max_seqs` 的两倍）、采样随机种子 |
| `step_log` | 步骤日志路径：每个引擎步骤和视觉调用写入一行 JSON |

`unit.stats()` 提供步骤计数（总步骤、图执行步骤、提示词 / 缓存 / 生成 / 强制 token 数、快照命中率、视觉调用）、各阶段耗时、按问题大小选定的算子后端，以及失败阶段的异常回溯。某个阶段失败时，该阶段及其后续请求都会以单元错误返回。

## 工作原理

| 部件 | 文件 | 作用 |
|---|---|---|
| 模型形状 | `model/shape.py` | 从检查点的 `config.json` 读取各项模型尺寸 |
| 前向计算 | `model/forward.py` | 对打包的多序列执行一步文本模型计算：每条序列在状态槽和 KV 页已保存的内容后追加一个或多个 token；并包含检查点的视觉塔 |
| 算子 | `ops/` | `delta`（门控 Delta Rule）、`conv`（DeltaNet 短卷积）、`attention`（分页注意力）；每种算子具有即时执行参考实现和多种后端（flash-linear-attention、仓库内 Triton 内核、PyTorch），通过计时竞速为每种问题尺寸选定后端 |
| 池管理 | `serve/pools.py` | 状态槽（每层 DeltaNet 的循环状态和卷积状态）、引用计数 KV 页、前缀快照（保存前缀后状态的槽位，共享完整页并复制不完整页） |
| 输入 | `serve/inputs.py` | 在准备线程中将请求转换为 token ID、mrope 位置、像素和快照边界 |
| 视觉处理 | `serve/vision.py` | 独立线程及 CUDA 流：批量处理等待中的图像，并向各请求交付嵌入及标记处理完成的事件 |
| 引擎 | `serve/engine.py` | 单线程负责准入（状态槽与 KV 页有余量时接纳请求，从最长的已保存快照恢复）、将待解码 token 与到期提示词块一起前向计算、在设备运行时准备语法遮罩、采样并确认输出 |
| CUDA 图 | `serve/graphs.py` | 按批次大小分桶捕获的解码步骤 |
| 语法约束 | `serve/grammar.py` | 针对各 JSON Schema 的 xgrammar token 遮罩及强制 token 跳转 |
| 单元 | `kohakugbc/engines/vlm/qwen35.py` | 流式引擎契约：提交、取消、容量、负载、统计；以及部署规划中的资源预留 |

## 测试

上游已把 qwen35 的测试并入仓库整体的检查策略：`tests/` 只做数据与格式检查（CPU），引擎与执行机器改由显卡上的实跑验证；随包迁移后不再附带原 `examples/qwen35/tests` 目录。以下为这些测试原先覆盖的内容：

- `test_ops.py`：在状态槽与 KV 页打乱顺序的打包序列上，将所有后端与即时执行参考实现比较（输出、回写状态、未使用槽位）；还会验证错误状态或页表确实能触发测试失败。
- `test_model.py`：在含图像的录制请求上比较前向计算与 transformers 的结果（48 个回复 token 的 argmax 一致性和 KL 散度；省略图像的前向计算会被检测出来）；从任一快照边界恢复应与完整预填充等价（错误卷积状态会被发现）；捕获的解码 CUDA 图结果应等于即时执行解码。
- `test_unit.py`：在三种配置下，用录制的 GBC 请求验证 Togeari 引擎契约（类似 vLLM 的 Schema、使用 `exact` 的紧凑 JSON、使用 bfloat16 状态和两处快照边界的 `string`）；还会测试将该入口作为 VLM 的整个机器。

## 性能测量

`scripts/bench/vlm_units.py` 通过单元契约在录制请求上执行任意 `vlm` 入口，写入每次请求的结果、内存和单元统计；`scripts/bench/real_machine.py` 运行录制任务并把生成的图与 PoC 比较。结果见：[真实引擎性能](../../../../docs/zh-CN/performance/real-engines.md)。
