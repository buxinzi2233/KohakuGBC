# 部署规划（Placement）

`kohakugbc.engines.placement` 会在加载任何模型权重之前，检查每个引擎是否能放入指定设备的内存。

每个入口声明其单元会占用哪些资源（`Need`：所有者、设备、字节数、用途）：

| 入口 | 每设备需求 |
|---|---|
| `torch`（检测器、嵌入模型） | 模型权重与峰值激活的估计值（`memory_gb`；默认 OWLv2-base 为 2.0 GiB，EmbeddingGemma 为 1.4 GiB） |
| `vllm` | 预留显存：单元所用每台设备的 `gpu_memory_utilization` 比例 |

执行机器汇总每台设备的资源需求，并与构建时的可用内存减去余量后进行比较（`machine.placement_headroom`，默认设备容量的 5%）。每个放不下的设备都会在同一错误信息中单列一行，且不会启动任何单元。Togeari 的 torch 引擎启动模型时还会测量真实内存占用（`load().extra["footprint"]`）。

| 部署方式 | 配置 |
|---|---|
| 单设备运行所有模型 | 每个槽位的 `devices` 指向同一设备 |
| 每设备一个副本 | `devices` 列出多个设备：每设备一个单元，通道根据负载选择 |
| 跨设备运行同一模型 | `devices` 内再嵌套一个列表（Togeari 将 `nn.Sequential` 划分为不同阶段）；VLM 则使用 `vllm` 每单元的 `devices` 做张量并行 |
| 多节点部署 | 通过引擎宿主使用 `remote` 入口（参见[跨节点引擎](engine-host.md)） |
