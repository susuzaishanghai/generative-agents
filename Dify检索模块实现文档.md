# Dify 检索模块实现文档

> 基于 Stanford Generative Agents 项目，在 Dify 平台上实现 Memory Retrieval（记忆检索）模块  
> 目标：在不强行改动原有算法的前提下，用 Dify 工作流 + PostgreSQL 复现 `new_retrieve` 的核心行为。

---

## 1. 整体设计思路

### 1.1 功能目标

- **输入**：
  - `persona_name`：当前 Agent 名称
  - `focal_text`：当前检索的焦点描述（可以是一句话、一个事件或一个主题）
  - `top_k`：需要返回的记忆条数（默认 30）
- **输出**：
  - 一组按综合得分排序的记忆节点列表，每个节点包含：
    - `id / node_id`
    - `type`（event/thought）
    - `created_at / last_accessed`
    - `description`
    - `poignancy`
    - 其他需要透传给上游 LLM 的字段

### 1.2 概念映射（原项目 → Dify）

- `AssociativeMemory.seq_event + seq_thought`  
  → PostgreSQL 表 `memory_nodes` 中 `type in ('event','thought')` 的记录
- `embedding_key` + `embeddings.json`  
  → `memory_nodes.embedding` 向量列（pgvector）
- `last_accessed`  
  → `memory_nodes.last_accessed` 时间戳列
- `poignancy`  
  → `memory_nodes.poignancy` 数值列
- `new_retrieve(persona, focal_points, n)`  
  → Dify 工作流 `Memory Retrieval`，一轮检索处理一个 `focal_text`，通过 Code 节点实现加权打分。

---

## 2. 数据层准备（PostgreSQL）

### 2.1 核心表：`memory_nodes`

参考 `check_database_structure.sql` 与现有实现，确保存在如下字段（字段名可以略有不同，但含义要一致）：

- `id`：主键（UUID 或自增 ID）
- `persona_name`：所属 Agent 名
- `type`：`event` / `thought` / `chat`（检索时主要用前两类）
- `created_at`：创建时间
- `last_accessed`：最近一次被检索或访问的时间
- `description`：自然语言描述（对应 `embedding_key`）
- `poignancy`：重要性评分（整数或浮点）
- `embedding`：`vector` 类型列，保存文本向量（例如 bge-m3，维度 1024 / 768 / 512 等）

推荐建立：

- `embedding` 上的 pgvector 索引（方便将来用向量检索加速）
- `(persona_name, type)` 普通索引（便于按角色+类型过滤）

### 2.2 向量写入与更新

- 记忆节点写入时，需在应用或 Dify 工作流中对 `description` 做一次 embedding，并写入 `memory_nodes.embedding`。
- 当某条记忆被检索后，需要更新其 `last_accessed = now()`，以保留“使用过的记忆会更近”的语义。

---

## 3. Dify 工具层：接入 PostgreSQL

### 3.1 使用 `dify_postgres_tool`

已有工具：`dify_postgres_tool`（见 `openapi.yaml`），提供：

- 通用 SQL 执行：`/execute_sql`, `/execute_sql_simple`, `/bulk_execute`
- CRUD 封装：`/crud/select`, `/crud/update` 等

在 Dify 后台：

1. 新建一个 **自定义 API 工具**，导入 `dify_postgres_tool/openapi.yaml`。
2. 配置基础 URL，例如：`http://localhost:8000`。
3. 在 Tool 的参数中配置数据库连接（dsn 等），可复用你在感知模块中已经用过的配置。

### 3.2 需要用到的几个接口

在检索工作流中，通常会用到：

- `crud/select` 或 `execute_sql_simple`：拉取候选记忆节点
- `crud/update` 或 `execute_sql_simple`：更新 `last_accessed`（可选）

---

## 4. Dify 工作流设计（Memory Retrieval）

### 4.1 工作流输入变量

在 Start 节点定义输入：

| 变量名        | 类型    | 说明                       |
|---------------|---------|----------------------------|
| `persona_name`| String  | 当前检索的 Agent 名称      |
| `focal_text`  | String  | 检索焦点描述               |
| `top_k`       | Number  | 需要返回的记忆条数（可选） |

### 4.2 步骤 1：计算 `focal_text` 的向量

- **节点类型**：Embedding 节点（或自定义 Code 节点 + embedding 模型调用）
- **输入**：`focal_text`
- **输出**：`focal_embedding`（向量，List[float]）

建议使用与你在 `memory_nodes.embedding` 中相同的模型（例如 BAAI/bge-m3），保持向量空间一致。

### 4.3 步骤 2：查询候选记忆（SQL 节点）

- **节点类型**：API 工具节点（调用 `executeSqlSimple` 或 `crudSelect`）
- **节点作用**：按 persona 过滤，取 `event` + `thought` 类型的记忆作为候选集。

示例（伪 SQL，实际根据工具入参格式调整）：

```sql
SELECT
  id,
  persona_name,
  type,
  created_at,
  last_accessed,
  description,
  poignancy,
  embedding
FROM memory_nodes
WHERE persona_name = :persona_name
  AND type IN ('event', 'thought')
  AND embedding IS NOT NULL
  AND (description NOT ILIKE '%idle%')
ORDER BY last_accessed ASC;
```

工具输出中，应至少包含字段：`id, type, last_accessed, description, poignancy, embedding`。

### 4.4 步骤 3：在 Code 节点里计算三维评分并排序

- **节点类型**：Code 节点（Python）
- **输入**：
  - `db_rows`：上一步工具返回的记录数组
  - `focal_embedding`：焦点向量
  - 可选超参数：`recency_decay, recency_w, relevance_w, importance_w, top_k`

代码核心逻辑（简化版伪代码）：

```python
import math
from typing import List, Dict

def cos_sim(a, b):
    dot = sum(x*y for x, y in zip(a, b))
    na = math.sqrt(sum(x*x for x in a))
    nb = math.sqrt(sum(x*x for x in b))
    if na == 0 or nb == 0:
        return 0.0
    return dot / (na * nb)

def normalize(d: Dict[str, float]) -> Dict[str, float]:
    if not d:
        return {}
    vals = list(d.values())
    vmin, vmax = min(vals), max(vals)
    if vmax == vmin:
        return {k: 0.5 for k in d.keys()}
    return {k: (v - vmin) / (vmax - vmin) for k, v in d.items()}

def main(arg):
    rows = arg["db_rows"]          # 来自 SQL 节点
    focal_emb = arg["focal_embedding"]
    recency_decay = arg.get("recency_decay", 0.99)
    recency_w = arg.get("recency_w", 1.0)
    relevance_w = arg.get("relevance_w", 1.0)
    importance_w = arg.get("importance_w", 1.0)
    gw = [0.5, 3.0, 2.0]           # 与原项目一致
    top_k = int(arg.get("top_k", 30))

    # 1) 构造 ID 列表，保证顺序与 last_accessed 排序一致
    node_ids = [str(r["id"]) for r in rows]

    # 2) Recency：按位置指数衰减
    recency_vals = [recency_decay ** i for i in range(1, len(node_ids) + 1)]
    recency_out = {node_ids[i]: recency_vals[i] for i in range(len(node_ids))}

    # 3) Importance：直接用 poignancy
    importance_out = {str(r["id"]): float(r.get("poignancy", 0.0)) for r in rows}

    # 4) Relevance：embedding 余弦相似度
    relevance_out = {}
    for r in rows:
        nid = str(r["id"])
        emb = r["embedding"]       # 确保转换成 List[float]
        relevance_out[nid] = cos_sim(emb, focal_emb)

    # 5) 归一化到 [0,1]
    recency_n = normalize(recency_out)
    importance_n = normalize(importance_out)
    relevance_n = normalize(relevance_out)

    # 6) 加权合成最终得分
    master = {}
    for nid in node_ids:
        master[nid] = (
            recency_w   * recency_n[nid]   * gw[0] +
            relevance_w * relevance_n[nid] * gw[1] +
            importance_w* importance_n[nid]* gw[2]
        )

    # 7) 排序取 Top-K
    sorted_ids = sorted(master.keys(), key=lambda x: master[x], reverse=True)
    sorted_ids = sorted_ids[:top_k]

    # 8) 组装输出（带上原始行 + 得分）
    id_to_row = {str(r["id"]): r for r in rows}
    result = []
    for nid in sorted_ids:
        row = dict(id_to_row[nid])
        row["score"] = master[nid]
        result.append(row)

    return {"retrieved_nodes": result}
```

> 这段代码是对原 `new_retrieve` 中 `extract_recency / extract_importance / extract_relevance` + 加权部分的等价翻译，放在 Dify 的 Code 节点即可。

### 4.5 步骤 4：更新 `last_accessed`（可选）

如果希望保留“被检索的记忆会变新”的行为，可以增加一个工具节点：

- **节点类型**：API 工具节点（`crudUpdate` 或执行 `UPDATE`）
- **输入**：上一节点输出的 `retrieved_nodes` 中的 `id` 列表
- **行为**：

```sql
UPDATE memory_nodes
SET last_accessed = now()
WHERE id = ANY(:id_array);
```

在 Dify 中，可以将 `id` 列表序列化成 JSON 数组，作为工具参数传入。

### 4.6 步骤 5：输出给上游 LLM / 应用

最终在工作流的 End / Response 节点中输出：

- `retrieved_nodes`：按分数排序的记忆列表
- 可选：仅输出 `description` 拼接成上下文，供 Planner LLM 使用

---

## 5. 示例：在 Dify 中的节点编排

可以按照下面的顺序在 Dify 的「工作流编排」界面中创建节点：

1. **开始节点（Start）**
   - 输入：`persona_name`, `focal_text`, `top_k`
2. **Embedding 节点：`生成_focal_text_向量`**
   - 输入：`focal_text`
   - 输出：`focal_embedding`
3. **PostgreSQL 工具节点：`查询候选记忆`**
   - 调用 `executeSqlSimple` 或 `crudSelect`
   - 条件：`persona_name` + `type in ('event','thought')`
   - 输出：`db_rows`
4. **Code 节点：`计算检索得分并排序`**
   - 输入：`db_rows`, `focal_embedding`, 以及超参数
   - 输出：`retrieved_nodes`
5. **PostgreSQL 工具节点（可选）：`更新_last_accessed`**
   - 根据 `retrieved_nodes.id` 列表执行 `UPDATE`
6. **结束节点（End / Response）**
   - 返回 `retrieved_nodes` 或格式化后的上下文字符串

---

## 6. 与原项目的差异与注意事项

- **多 focal_points 支持**：
  - 原项目 `new_retrieve` 支持同时对多个 `focal_points` 检索，这里示例以“一个工作流处理一个 focal_text”为主。
  - 如果需要，可以在 Start 节点传入一个 JSON 数组，在 Code 节点内循环处理，每个 focal 返回一组节点。
- **embedding 模型不同**：
  - 原项目使用 OpenAI `text-embedding-ada-002`，当前实现使用 bge-m3，只要读取/写入使用的都是同一模型，算法本身不受影响。
- **权重调参**：
  - 可以在 Dify 的变量中暴露 `recency_w / relevance_w / importance_w / recency_decay`，允许在不同应用、不同 persona 间做个性化调节。
- **性能优化**：
  - 如果记忆量很大，可以在 SQL 层先做一次粗筛（例如只取最近 N 条，或先做向量相似度 top-k），再在 Code 节点做精排。

通过上述设计，你就可以在 Dify 中较完整地复现 Generative Agents 中的记忆检索逻辑，并且保留了进一步调参和扩展的空间。 

