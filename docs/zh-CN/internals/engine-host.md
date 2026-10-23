# 跨节点引擎

一个 Togeari 执行机器本质上是单进程中的一个调度器。部署在其他节点上的引擎通过 **引擎宿主（engine host）**（`kohakugbc.engines.host.EngineHost`、`gbc host`）运行，执行机器的通道则为其创建一个 **远程单元（remote unit）**（`RemoteUnit`，对应 `remote` 入口）。任何槽位的任何入口都可以由宿主承载：宿主通过相同的配置创建该槽位的单元，执行通道则采用该入口的适配器、请求构造器及通道选项。

```
gbc host configs/engines/vlm.py --slot vlm --listen 0.0.0.0:7501   # 在引擎节点执行
```

```python
engines = {
    "vlm": {
        "entry": "remote",
        "hosts": ["10.0.0.5:7501/replay-vlm-0"],
        "authkey_env": "KOHAKUGBC_HOST_KEY",
        "runs": {"entry": "replay", "trace": "runs/traces/poc30/text"},
    }
}
```

如果执行机器所在节点无法构建被托管的入口（例如 Linux 节点上运行 vLLM，而机器部署在 Windows），可以让通道直接指定需求，而不是提供 `runs`：

```python
engines = {
    "vlm": {
        "entry": "remote",
        "hosts": ["10.0.0.5:7593/vllm-0"],
        "adapter": "vllm",
        "kind": "stream",
    }
}
```

`adapter` 是 `kohakugbc.engines.adapters.ADAPTERS` 中的一项（`vllm`、`torch`、`torch-text`）；`kind`、`protocol`、`max_batch`、`wait_ms` 描述执行单元的特性。

## 通信协议

使用 `multiprocessing.connection` 传输经过 pickle 序列化的元组，通过共享密钥（`KOHAKUGBC_HOST_KEY`）认证。只有持有密钥的客户端才能发送数据，这是安全反序列化所依赖的前提；因此共享密钥和网络访问权限必须受到保护。

连接时，单元发送所需的单元标识；宿主返回该单元的类型、批大小、协议和规格。提交消息包含预处理后的载荷；结果按票据（ticket）返回。单元状态包含容量、负载、就绪状态以及宿主已接收该连接多少次提交；状态在变化时发送，且至少每 0.2 秒发送一次。

## 引擎宿主

一个主循环线程负责向引擎发起所有调用（提交、取消、容量、负载，与 Togeari 的调度器契约一致）；每个连接有一个读取线程，将消息转交给主循环；结果通过引擎自身的线程返回。托管的工作项是一个 Togeari 工作项，其适配器仅返回已准备好的载荷并原样传回响应，机器再使用通道的适配器解码响应。

## 远程单元

| 成员 | 行为 |
|---|---|
| `submit` | 将工作加入发送线程队列（载荷在该线程中准备），并返回票据 |
| `capacity` | 宿主最后报告的容量，减去尚未被宿主接收的提交数 |
| `load` | 宿主正在运行和等待的任务数，加上传输途中的提交数 |
| 结果 | 读取线程按票据返回结果；容量增加时唤醒通道 |
| 连接断开 | 所有未完成票据返回 `UnitError`；`ready()` 尝试重连 |

通道与本地单元一样处理远程单元：在 `UnitError` 时切换其他单元；选择、批处理与任务准入都使用远程单元报告的容量和负载。

## 测试

`tests/engines/test_remote.py`：远程流式与批处理单元满足 Togeari 引擎契约（`check_engine`）；即使检测途中宿主被终止，下一次提交也会获得 `UnitError`。真实检测器在 `gbc host` 进程中运行时，机器仍能得到与 PoC 相同的图。Windows 上的机器还可调用 WSL 中由 `gbc host` 提供的 vLLM（`KOHAKUGBC_REMOTE_VLM`）。
