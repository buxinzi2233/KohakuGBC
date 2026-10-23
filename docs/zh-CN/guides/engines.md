# 配置引擎

GBC 执行机器的每个槽位（`vlm`、`detect`、`embed`、`cpu`）都在配置的 `engines` 中指定一个入口及其设置。配置文件采用 Python 格式；命令行靠后的配置文件会覆盖或逐项合并前面的文件。

## 通过 API 调用 VLM

支持任何 OpenAI 兼容服务器（vLLM、SGLang、llama.cpp server、OpenAI、OpenRouter）或 Anthropic Messages API。每个端点对应一个执行单元：

```python
engines = {
    "vlm": {
        "entry": "api",
        "protocol": "openai",
        "flavour": "vllm",
        "model": "qwen3.5-4b",
        "endpoints": [
            {"base_url": "http://10.0.0.5:8000/v1", "max_inflight": 64},
            {"base_url": "http://10.0.0.6:8000/v1", "max_inflight": 64},
        ],
        "metrics": "vllm",
    }
}
```

| 设置 | 说明 |
|---|---|
| `protocol` | `"openai"`（`/chat/completions`）或 `"anthropic"`（`/messages`） |
| `flavour` | OpenAI 兼容服务放置 JSON Schema 的位置及其关闭 Qwen3.5 思考模式的方式：`vllm`、`sglang`、`llama.cpp`、`openai`、`openrouter` |
| `model`、`sampling` | 模型名称，以及额外请求体字段（`top_p`、`min_p` 等） |
| `endpoints` | 每个单元的 `base_url`、`max_inflight`（默认 32）、`rate`（每秒请求数，或 `[rate, burst]`）、`api_key_env`、`headers`、`name` |
| `timeout_s`、`connect_timeout_s`、`health_path` | 请求超时；连接超时（超过后单元判为不可用）；通过 GET 检查并恢复不可用单元的健康检查路径 |
| `metrics` | 设为 `"vllm"` 时，在单元负载报告中包含服务器正在运行与等待的请求数 |

每个请求都携带 `X-GBC-Request` 头，值是请求内容的 SHA-256 摘要，便于上游缓存或日志系统识别不同请求与重复请求。

## 进程内 VLM（vLLM）

在执行机器的同一进程内运行 vLLM 引擎，每个 `AsyncLLM` 对应一个单元（Linux；安装：`pip install "kohakugbc[vllm]"`）：

```python
engines = {
    "vlm": {
        "entry": "vllm",
        "model": "Qwen/Qwen3.5-4B",
        "units": 2,
        "devices": [[0], [1]],
        "gpu_memory_utilization": 0.5,
        "max_num_seqs": 64,
    }
}
```

| 设置 | 说明 |
|---|---|
| `model` | 模型检查点名称或目录 |
| `units`、`devices` | 单元数量及每个单元使用的 CUDA 设备；一个单元使用多个设备时在这些设备上进行张量并行 |
| `gpu_memory_utilization`、`max_model_len`、`max_num_seqs`、`dtype` | vLLM 自身参数（默认分别为 0.5、16384、64、`bfloat16`） |
| `capacity` | 每单元允许的在途请求数（`max_num_seqs` 的两倍） |
| `engine_args`、`sampling` | 其他 `AsyncEngineArgs` 参数与额外的 `SamplingParams` 字段 |

该单元使用模型聊天模板构造提示词（关闭思考模式），按 API 路径的相同方式解码图像，将回复约束为请求的 JSON Schema；取消的请求会在 vLLM 内中止。由于 vLLM 会通过创建子进程启动核心引擎，使用此入口构建机器的脚本需要 `if __name__ == "__main__":` 保护。

## 进程内 VLM（qwen35 示例）

为这类工作负载专门开发的 Qwen3.5 引擎（`pip install -e examples/qwen35`；支持 Windows 和 Linux），每个设备一个单元：

```python
engines = {"vlm": {"entry": "qwen35", "model": "Qwen/Qwen3.5-4B", "memory": 0.65}}
```

参数、设计与测试详见 [qwen35 示例](../../../examples/qwen35/README.zh-CN.md)。`configs/machine/qwen35_local.py` 使用该引擎在单 GPU 上运行完整执行机器。

## 视觉模型（torch）

检测器（OWLv2）和嵌入模型（EmbeddingGemma）通过 Togeari 的 torch 引擎运行（`pip install "kohakugbc[torch]"`）；每个设备部署位置对应一个单元：

```python
engines = {
    "detect": {"entry": "torch", "devices": "auto", "max_batch": 8, "wait_ms": 5.0},
    "embed": {"entry": "torch", "devices": ["cpu"], "max_batch": 64},
}
```

| 设置 | 说明 |
|---|---|
| `model` | 检测器可用 `google/owlv2-base-patch16-ensemble`（PoC 原版）或 `facebook/sam3`；嵌入模型为 `google/embeddinggemma-300m` |
| `devices` | `"auto"`（所有 CUDA 设备，否则 CPU）、单个设备或设备列表；每个设备一个单元 |
| `dtype` | 默认 float32；`"auto"` 根据设备选择 bfloat16 或 float16 |
| `max_batch`、`wait_ms` | 批大小；一个批次为等待更多请求而停留的时间 |
| `memory_gb` | 部署规划中各设备的预计显存用量 |
| `engine` | Togeari torch 引擎选项：`schedule`、`isolation`、`slots`、`memory_cap_gb`、`prep_workers` |
| `box_threshold`、`max_texts` | 检测框分数下限（0.05）与单次前向计算中每行的文本数（16） |

在插画图像中，SAM 3 能检测出 OWLv2 漏掉的区域。以动漫风格人物肖像为例，OWLv2 对“girl”的最高分框（0.415）只覆盖腿部；SAM 3 给出完整人物框（0.89），并将头发（0.94）与尾巴（0.91）分别框出，而 OWLv2 错把尾巴识别成“hair”。SAM 3 对每个裁剪区域运行一次视觉编码器，并对每条文本执行一次解码器前向计算；批量计算与单次计算结果一致。`configs/machine/qwen35_sam3_local.py` 会在单 GPU 基础配置上叠加 bfloat16 的 SAM 3。

检测器默认使用 float32，是因为此时结果不依赖批处理方式：在 float16 下，批量裁剪的得分相比单独计算最多偏移 0.085；float32 下最多偏移 1.3e-4（检测结果不变）。PoC 使用 float16、`max_batch` 为 1，因此这一设置能完全复现 PoC 的检测框。检测器依赖 torchvision：transformers 较慢的图像处理器采用不同的缩放方式，会产生不同的检测框。

`gbc doctor CONFIG ...` 可打印设备、引擎入口与配置对应的部署规划。

## 回放（Replay）

使用录制轨迹（`runs/traces/...`）返回结果的伪引擎，无需实际模型即可执行 GBC 程序和机器：

```python
_TRACE = "runs/traces/poc30/text"
engines = {
    "vlm": {"entry": "replay", "trace": _TRACE, "capacity": 32},
    "detect": {"entry": "replay", "trace": _TRACE},
    "embed": {"entry": "replay", "trace": _TRACE},
}
source = {"kind": "trace", "path": _TRACE}
```

`python -m kohakugbc.engines.replay.server TRACE --port 8088` 会将轨迹作为兼容 OpenAI 和 Anthropic 的聊天服务器提供给 `api` 入口使用，无需模型。

## 跨节点引擎

在引擎所在节点运行 `gbc host CONFIG --slot vlm --listen 0.0.0.0:7501`，通过 `KOHAKUGBC_HOST_KEY` 提供共享密钥，并在机器配置中引用它：

```python
engines = {
    "vlm": {
        "entry": "remote",
        "hosts": ["10.0.0.5:7501/replay-vlm-0"],
        "runs": {"entry": "replay", "trace": "runs/traces/poc30/text"},
    }
}
```

`runs` 指定宿主实际运行的入口及参数；执行通道会采用该入口的适配器、批处理和通道选项。详见[跨节点引擎](../internals/engine-host.md)。

## CPU 工作单元

```python
engines = {"cpu": {"entry": "python", "workers": 8, "mode": "thread"}}
```

设置 `mode = "process"` 可在工作进程中执行 CPU 节点（裁剪、图像处理、解析和图解析）。
