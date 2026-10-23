# 记录与图解析

`kohakugbc.graph` 把单张图像任务收集的记录转换为 GBC 图。

## 记录

一条记录表示一次查询的结果，是 ml-gbc 的 `NodeInfo` 的纯数据形式（`kohakugbc.graph.records`）：

| 字段 | 内容 |
|---|---|
| `ids` | 此结果涉及的每个实体的 `[entity_id, label, text]`；第一个对应顶点，其余为合并到同一对象的实体 |
| `box` | 相对于整张图像的 `[left, top, right, bottom, confidence]`；`None` 表示整个图像 |
| `descs` | `[text, desc_label, [entity_id, ...]]`；其中的 ID 指向关系回答所涉及的实体 |
| `children` | 每个命名实体的短语对应 `[entity_id, label, text]`；其中的 `text` 将成为边标签 |

关系记录（首个 ID 的标签为 `relation`）指向提出关系问题时所针对的顶点。没有描述（`descs`）的记录不产生顶点。记录只包含列表，因此经过 JSON 序列化与反序列化往返后保持相等。

## 解析

ml-gbc 中 `node_infos_to_gbc_graph` 的处理步骤按以下顺序执行：

| 步骤 | 结果 | 复杂度 |
|---|---|---|
| ID | 将每个已合并的 ID 映射到其顶点；带编号的 ID（如 `cup_1`，根据 Python 的 `str.isdigit` 判断）归入不带编号的主体；没有组合顶点时，主体指向对应实例 | 线性 |
| 关系拆分 | 为回答中涉及的每组不同实体（`[a\|b]`）建立一个顶点，边界框取实体框的并集，再使用各实体 ID 的末尾部分作为标签，将其链接到被提问的顶点 | 回答数 × 实体数 |
| 顶点 | 每条有描述的记录对应一个顶点；按顶点类别标记描述；为每个命名短语建立指向 ID 映射到的所有顶点的出边 | 随边数线性 |
| 入边 | 按源顶点 ID 顺序，依次收集各源顶点的出边并去重 | 边数 × 入度 |
| 遮罩 | 对每个顶点按 ID 顺序找出主要包含于该顶点内部或主要包含该顶点的其他顶点（以内侧面积为基准的重叠比例超过阈值 0.85） | 顶点数的平方 |
| 描述文本 | 从第一个图像顶点提取图的原始描述、简短描述与详细描述 | 线性 |

最终图是 `GbcGraphFull.model_dump()` 逐键输出的字典，可原样被 ml-gbc 的 `GbcGraph` 和 `GbcGraphFull` 加载。统计信息（`kohakugbc.graph.stats`）包括顶点总数、各标签顶点数、边、描述文本、从根顶点出发的 BFS 深度、根顶点及叶顶点。

## 两种实现

- `kohakugbc.graph.reference`：Python 参考实现，直接移植自 ml-gbc。
- `kohakugbc._rs.resolve`（`src/kohakugbc-rust/graph/`）：基于同一定义编写的 Rust 实现；不受 GIL 限制地解析并序列化，返回图的 JSON 文本及其统计信息。

程序调用 `kohakugbc.graph.to_text`，它在导入时选择具体实现（构建了 Rust 扩展时默认使用 Rust；可通过 `KOHAKUGBC_RESOLVE=python|rust` 覆盖）。

## 如何验证

`scripts/reference/mlgbc_cases.py` 在安装 ml-gbc 的环境下生成 `tests/graph/fixtures/mlgbc_cases.jsonl.gz`。测试用例覆盖录制的 PoC 图像和带种子的生成记录集合：实体合并、含/不含组合顶点的编号实例、合并和编号实体上的关系、缺失记录、未知或空白子项、阈值附近的边界框、仅 Python `str.isdigit` 视为数字的 ID。每条用例包含记录、ml-gbc 生成图的摘要与计数，或 ml-gbc 抛出的异常。

`tests/graph` 根据规范化文本的摘要（`kohakugbc.graph.digest`）比对两种实现，并验证故意破坏遮罩逻辑、入边顺序和编号 ID 规则后，测试确实会失败。
