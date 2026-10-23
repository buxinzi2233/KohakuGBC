# KohakuGBC：仓库说明（中文版，仅供阅读）

项目把基于图的图像描述（Graph-Based Captioning，GBC）实现为运行在 Togeari 数据流机器上的 KIR 程序，并提供执行它的引擎。按 Togeari 对任务的划分，仓库分成三个主要层次：

- `src/kohakugbc/program/`、`nodes/`、`graph/`：GBC 程序（`gbc.kir`）、含模型节点规格的 `gbc` 节点库、记录以及将记录解析为 GBC 图的逻辑。
- `src/kohakugbc/machine/`、`io/`：Togeari 上的程序执行（绑定、通道、策略、机器模块），以及数据源与输出端。
- `src/kohakugbc/engines/`：满足 Togeari 引擎契约的实现（`vlm/`、`torch/`、`host/`、`replay/`）。
- `src/kohakugbc-rust/`：Rust 编写的图解析扩展 `kohakugbc._rs`（crate 的 `Cargo.toml` 位于仓库根目录）。
- `examples/qwen35/`：专用 Qwen3.5 引擎，作为引擎层之上的独立软件包。
- `configs/`（`engines/`、`machine/`）、`scripts/`（`trace/`、`bench/`）、`docs/`（`concepts/`、`guides/`、`internals/`、`performance/`）。
- `plans/` 和 `runs/` 均被 Git 忽略。

## 常用命令

开发环境为通过 uv 管理的 `.venv`。安装过程会使用 maturin 构建扩展，因此修改 `src/kohakugbc-rust/` 后需要再次执行 `uv pip install -e ".[dev]"`。版本号只在 `Cargo.toml` 中维护。

```
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test --no-default-features
.venv\Scripts\python.exe -m pytest tests -q -m "not gpu and not serial and not weights" -n auto --maxprocesses 8 --dist worksteal
.venv\Scripts\python.exe -m pytest tests -q -m "serial and not gpu"
.venv\Scripts\python.exe -m black --line-length 100 src tests examples scripts
.venv\Scripts\python.exe -m ruff check src tests examples scripts
.venv\Scripts\python.exe scripts\check_imports.py
.venv\Scripts\python.exe scripts\check_md_black.py .
```

`gpu` 测试需要 CUDA 设备；`weights` 测试会加载真实模型权重；`serial` 测试会验证实际耗时限制或调度结果，因此需要独立运行。

## 开发规则

- **导入边界**（`scripts/check_imports.py`）：程序层只允许导入 `kohakunode`、图像库和自身模块；机器与 IO 层导入 `togeari`；引擎层可导入 torch、transformers 和 vLLM（可选依赖），但不能导入机器层；`examples/qwen35` 构建在引擎层之上。
- GBC 程序及其节点不得指定设备、引擎或模型；引擎也不得直接依赖某个 GBC 节点。
- **代码风格**：不同逻辑组之间留空行；每组上方用简短注释说明作用；优先使用 `match` 而非冗长的 `if / elif`；不要在函数内部导入；使用 ruff 与 black，行宽 100。
- **测试**：不使用假模型。程序和机器利用回放录制轨迹的伪引擎测试；模型则加载真实权重并与参考前向计算对比。
- 耗时较长的运行任务采用后台作业。
