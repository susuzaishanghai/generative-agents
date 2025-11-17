# 记忆流检索流程说明（Generative Agents）

本文档基于 `reverie/backend_server` 目录下的实际代码，对论文中“记忆流（Memory Stream）”在本项目中的落地实现，尤其是**检索流程**做一个结构化说明，便于后续改造与对比其它系统（如 Dify 检索模块）。

---

## 1. 记忆流数据结构：`AssociativeMemory`

- 位置：`reverie/backend_server/persona/memory_structures/associative_memory.py`
- 对应论文中的 Memory Stream，核心类为：
  - `class AssociativeMemory`
  - 内部节点类型 `class ConceptNode`

### 1.1 `ConceptNode` 结构

字段（简化）：

- 标识与类型：
  - `node_id`：节点 ID，如 `node_1`
  - `node_count`：全局计数（随记忆流增长）
  - `type_count`：在各自类型（event/thought/chat）内的编号
  - `type`：`"event" | "thought" | "chat"`
  - `depth`：思考深度（thought 可以叠加其它节点形成更深的思考）
- 时间信息：
  - `created`：创建时间（`datetime`）
  - `expiration`：到期时间，可为 `None`
  - `last_accessed`：最近一次被检索或使用的时间
- 语义内容：
  - `subject, predicate, object`：事件三元组（S/P/O）
  - `description`：人类可读描述
  - `embedding_key`：在 `embeddings.json` 中查向量的 key
  - `poignancy`：重要性分数（越高越重要）
  - `keywords`：关键词集合（通常来自 S/P/O）
  - `filling`：补充信息（如证据节点 id、聊天内容等）

### 1.2 `AssociativeMemory` 的核心成员

- 顺序序列（按时间维护）：
  - `seq_event`：事件流
  - `seq_thought`：想法流
  - `seq_chat`：对话流
- 倒排索引（关键词 → 节点列表）：
  - `kw_to_event`
  - `kw_to_thought`
  - `kw_to_chat`
- 关键词强度（用于触发反思）：
  - `kw_strength_event`
  - `kw_strength_thought`
- 向量库：
  - `embeddings`：从 `embeddings.json` 加载，`embedding_key -> 向量`

初始化流程：

1. 从磁盘路径 `f_saved` 中读取：
   - `nodes.json`
   - `kw_strength.json`
   - `embeddings.json`
2. 遍历 `nodes.json` 中所有节点：
   - 根据 `type` 分别调用 `add_event / add_thought / add_chat`
   - 在这些函数里完成：
     - 创建 `ConceptNode` 实例
     - 插入时间序列（新节点插入列表头部）
     - 更新关键词倒排索引
     - 更新 `kw_strength_*` 和 `embeddings`

> 这一层可以理解为“原始记忆流 + 关键词倒排表 + 向量索引”的组合。

---

## 2. 基础检索：`retrieve`（基于关键词的局部检索）

这一套对应 Persona 主循环里“当前一步要不要对周围事件做出反应”的检索逻辑。

主要代码位置：

- Persona 接口：`reverie/backend_server/persona/persona.py`
  - `Persona.retrieve(self, perceived)` → `cognitive_modules.retrieve.retrieve`
- 实现：`reverie/backend_server/persona/cognitive_modules/retrieve.py`
  - `def retrieve(persona, perceived): ...`
- 倒排索引检索函数：
  - `AssociativeMemory.retrieve_relevant_events/thoughts` 位于  
    `reverie/backend_server/persona/memory_structures/associative_memory.py`

### 2.1 调用链（在 Persona 主循环中的位置）

`Persona.move`（`persona.py`）中，核心顺序为：

1. `perceived = self.perceive(maze)`
2. `retrieved = self.retrieve(perceived)`
3. `plan = self.plan(maze, personas, new_day, retrieved)`
4. `self.reflect()`

其中：

- `perceive` 会将当前看到的新事件写入记忆流（调用 `add_event` 等），并返回一组新感知事件的 `ConceptNode` 列表。
- `retrieve` 只关注**刚感知到的这些事件**，为每个事件拉取与之相关的历史事件和想法。
- `plan` 内部会进一步根据检索结果决定是否与他人对话、等待或不反应，本节聚焦在检索本身。

### 2.2 `retrieve(persona, perceived)` 的逻辑

输入：

- `perceived`：一组刚刚感知到的事件节点 `ConceptNode`。

核心流程：

1. 初始化 `retrieved = {}`。
2. 对于每个感知事件 `event`：
   - 以 `event.description` 作为 key，在 `retrieved` 中新建一条记录：
     - `["curr_event"] = event`（当前事件本身）
     - `["events"] = persona.a_mem.retrieve_relevant_events(event.subject, event.predicate, event.object)`
     - `["thoughts"] = persona.a_mem.retrieve_relevant_thoughts(event.subject, event.predicate, event.object)`
3. 返回形如：

```python
retrieved = {
  "<event description A>": {
    "curr_event": <ConceptNode>,
    "events": [<ConceptNode>, ...],
    "thoughts": [<ConceptNode>, ...],
  },
  ...
}
```

在 `plan.plan(...)` 中：

- 通过 `_choose_retrieved(persona, retrieved)` 选出当前要聚焦的事件。
- 再通过 `_should_react(...)` 判断是否需要：
  - 发起对话（`"chat with ..."`）
  - 其它反应（`"wait ..." / "react"`）

### 2.3 `AssociativeMemory` 的关键词检索

新增记忆时（`add_event/add_thought/add_chat`）：

- 取 `keywords`（一般由 S/P/O 三元组组成），统一转为小写：
  - `keywords = [i.lower() for i in keywords]`
- 将每个关键词映射到对应节点列表：
  - `kw_to_event[kw]` / `kw_to_thought[kw]` / `kw_to_chat[kw]`

检索时：

- `retrieve_relevant_events(s_content, p_content, o_content)`：
  - 以 `[s_content, p_content, o_content]` 为检索词列表。
  - 对每个检索词，查 `kw_to_event` 中对应的节点列表，并去重得到 `set(ret)`。
- `retrieve_relevant_thoughts(...)` 类似，只是查 `kw_to_thought`。

> 逻辑上，这一层是“用当前事件 S/P/O 作为关键词，在倒排索引中查找历史记忆和想法”。  
> 注意实现中存在大小写细节：写入索引时转成小写，检索时没有统一 `lower()`，实际生产环境中最好做一次修正。

---

## 3. 加权检索：`new_retrieve`（全局记忆的多维打分检索）

第二套检索逻辑不是围绕“当前一步感知到的 event”，而是面向**主题/焦点（focal point）**，在整条记忆流上做加权排序检索。

主要代码位置：

- 实现：`reverie/backend_server/persona/cognitive_modules/retrieve.py`
  - `def new_retrieve(persona, focal_points, n_count=30): ...`
- 权重与超参数：短期记忆 `Scratch`：
  - `reverie/backend_server/persona/memory_structures/scratch.py`
    - `recency_w, relevance_w, importance_w, recency_decay, ...`

典型调用场景：

- **反思（Reflect）模块**：`reverie/backend_server/persona/cognitive_modules/reflect.py`
  - 生成反思主题 `focal_points = generate_focal_points(persona, 3)`
  - 调用 `retrieved = new_retrieve(persona, focal_points)`
  - 再用 GPT 生成新想法并写回记忆流。
- **规划模块中的身份更新**：`plan.revise_identity`（`plan.py` 中）
  - 用 focal points（如“某天的计划”、“最近的重要事件”）从记忆流中拉取相关节点作为上下文。
- **对话模块**：`converse`（`converse.py` 中）
  - 用对方 persona 的名字、关系总结、或当前对话内容作为 focal point，获取与该人相关及当前语境相关的记忆，为对话生成提供背景。

### 3.1 输入与输出

函数签名：

```python
new_retrieve(persona, focal_points, n_count=30)
```

- `persona`：当前 Persona 对象。
- `focal_points`：一组字符串，每个代表一个“检索焦点”（例如某个主题、某个人、某条计划）。
- `n_count`：每个 focal point 返回的最多节点数（Top-N）。

返回值：

```python
retrieved = {
  "<focal_pt_1>": [<ConceptNode>, ...],
  "<focal_pt_2>": [<ConceptNode>, ...],
  ...
}
```

### 3.2 候选集构造

对于每个 `focal_pt`：

1. 从记忆流中取所有事件与想法：

```python
nodes = [
  [i.last_accessed, i]
  for i in persona.a_mem.seq_event + persona.a_mem.seq_thought
  if "idle" not in i.embedding_key
]
```

2. 按 `last_accessed` 时间排序（最早访问在前），然后抽出节点列表：

```python
nodes = sorted(nodes, key=lambda x: x[0])
nodes = [i for created, i in nodes]
```

> 这里的“时间顺序”是按最近访问时间而非创建时间，体现“用过的记忆会变得更近”的设计。  
> 注意：后续的新近度得分**并不直接使用时间差**，而是对这个排序后的列表按位置应用指数衰减（第 1 个是 `decay^1`，第 2 个是 `decay^2`，依此类推），因此相邻节点间的得分差是固定的，与它们之间真实时间间隔长短无关。

### 3.3 三种原始评分

对候选节点 `nodes` 计算三种分数：**Recency / Importance / Relevance**。

1. **Recency（新近性）**：`extract_recency(persona, nodes)`
   - 生成一个衰减序列（基于节点在排序列表中的位置，而不是绝对时间差）：

   ```python
   recency_vals = [persona.scratch.recency_decay ** i
                   for i in range(1, len(nodes) + 1)]
   # 默认 recency_decay = 0.99
   ```

   - 按顺序赋值给节点：

   ```python
   recency_out[node.node_id] = recency_vals[count]
   ```

2. **Importance（重要性）**：`extract_importance(persona, nodes)`
   - 直接使用节点的 `poignancy`：

   ```python
   importance_out[node.node_id] = node.poignancy
   ```

3. **Relevance（相关性）**：`extract_relevance(persona, nodes, focal_pt)`
   - 对 `focal_pt` 调用 `get_embedding(focal_pt)` 得到向量。
   - 对每个节点，用其 `embedding_key` 从 `persona.a_mem.embeddings` 中取出向量。
   - 通过余弦相似度计算相关性：

   ```python
   relevance_out[node.node_id] = cos_sim(node_embedding, focal_embedding)
   ```

### 3.4 归一化与加权融合

1. **归一化**：调用 `normalize_dict_floats(d, 0, 1)` 将每一种分数分别 min–max 归一化到 `[0,1]`。
2. **全局权重（固定）**：

```python
gw = [0.5, 3, 2]  # [recency, relevance, importance]
```

3. **个体权重（可调）**：从 `persona.scratch` 中读取：

- `recency_w`
- `relevance_w`
- `importance_w`

当前默认值（见 `scratch.py`）为：

- `recency_w = 1.0`
- `relevance_w = 1.0`
- `importance_w = 1.0`
- `recency_decay = 0.99`

因此在默认配置下，三种成分的有效权重约为：

- 新近度：`1.0 * 0.5 = 0.5`
- 相关性：`1.0 * 3   = 3.0`
- 重要性：`1.0 * 2   = 2.0`

4. **最终得分**：

```python
master_out[key] = (
    persona.scratch.recency_w   * recency_out[key]   * gw[0] +
    persona.scratch.relevance_w * relevance_out[key] * gw[1] +
    persona.scratch.importance_w* importance_out[key]* gw[2]
)
```

在 debug 模式下，代码还会打印每个节点的 embedding_key 以及三种分量的单独贡献，便于调参。

### 3.5 Top-N 筛选与状态更新

1. 对 `master_out` 中的 `(node_id -> score)` 做排序，选出前 `n_count` 个：

```python
master_out = top_highest_x_values(master_out, n_count)
master_nodes = [persona.a_mem.id_to_node[key]
                for key in list(master_out.keys())]
```

2. 将这些被选中的节点的 `last_accessed` 更新为当前时间 `persona.scratch.curr_time`：

```python
for n in master_nodes:
    n.last_accessed = persona.scratch.curr_time
```

3. 将结果挂载到返回字典：

```python
retrieved[focal_pt] = master_nodes
```

> 通过更新 `last_accessed`，被多次作为检索结果的记忆会越来越“新近”，在未来的检索中不断得到加权，有利于构建稳定的“长期主题”。

---

## 4. 两套检索的分工与对比

综上，记忆流上的检索在本项目中分为两层：

1. **局部、事件驱动的检索：`retrieve`**
   - 输入：感知模块刚刚返回的新事件 `perceived`。
   - 使用方式：从倒排索引中找与该事件 S/P/O 相同或相似的历史记忆。
   - 输出：针对每个当前事件，给出相关事件与想法，供短期决策（是否对别人说话、是否对某个事件做出反应）使用。
   - 使用场景：`Persona.move` → `plan.plan` → `_choose_retrieved` / `_should_react`。

2. **全局、主题驱动的检索：`new_retrieve`**
   - 输入：若干“焦点字符串”（focal points），可以是人物、计划、关系、或者自然语言描述。
   - 使用方式：在整条事件 + 想法记忆流上，根据 Recency / Importance / Relevance 三种分数做加权排序，取 Top-N。
   - 输出：每个 focal point 对应的一组最相关记忆节点列表。
   - 使用场景：
     - 反思（Generate insights）：`reflect.run_reflect`
     - 身份/计划更新：`plan.revise_identity`
     - 对话上下文构建：`converse.agent_chat_v1` 等

简单理解：

- `retrieve` 更像“当前这件事让我想起了什么？”（基于关键词的联想）。
- `new_retrieve` 更像“围绕这个主题，回顾近期和重要的所有相关记忆。”（基于向量和权重的全局检索）。

---

## 5. 可配置点与改造建议（简要）

在不改变整体架构的前提下，记忆检索可以从以下几个方向进行调优或对比其它系统（如 Dify 检索工作流）：

1. **关键词检索层**
   - 修复大小写不一致问题：检索时统一 `lower()`，确保与索引一致。
   - 增加简单的同义词/短语拆分逻辑，提高匹配召回率。
2. **向量检索层**
   - 考虑引入“多模态” embedding（如感知模块中的视觉/空间信息）作为补充。
   - 对 `embedding_key` 的命名与分层（例如为不同类型事件使用不同命名空间），便于后期扩展。
3. **权重与触发机制**
   - 在 `Scratch` 中，`recency_w / relevance_w / importance_w / recency_decay` 与“反思触发阈值”等参数可以根据任务或 persona 类型做个性化调整。
   - 可以考虑通过离线评估、简单的 RL 或人类偏好标注，来学习这三个权重。

这些点可以作为今后将本项目的检索机制与 Dify 检索模块进行系统对比和融合设计的基础。

