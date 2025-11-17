# Dify 检索模块实现文档

> 基于 Stanford Generative Agents 项目，在 Dify 平台上实现 Memory Retrieval（记忆检索）模块。  
> 目标：在不强行改动原有算法的前提下，用 Dify 工作流 + PostgreSQL 复现 `new_retrieve` 的核心行为，并为后续 `retrieve()` 关键词检索预留扩展空间。

---

## 1. 整体设计思路

### 1.1 功能目标

- **输入**：
  - `persona_name`：当前 Agent 名称
  - `focal_text`：当前检索的焦点描述（可以是一句话、一个事件或一个主题）
  - `top_k`：需要返回的记忆条数（默认 30）
  - 检索超参数（可选）：`recency_w`、`relevance_w`、`importance_w`、`recency_decay`
- **输出**：
  - 一组按综合得分排序的记忆节点列表，每个节点包含：
    - `id / node_id`
    - `type`（event/thought）
    - `created_at / last_accessed`
    - `description`
    - `poignancy`
    - 其他用于规划 / 反思 / 对话的必要字段
  - 辅助信息：
    - `accessed_ids`：被检索到的节点 ID 列表（用于更新 `last_accessed`）
    - `status`：`ok` / `no_candidates` / `error`
    - `debug`：调试信息（候选数、得分范围等）

### 1.2 概念映射（原项目 → Dify）

- `AssociativeMemory.seq_event + seq_thought`  
  → PostgreSQL 表 `memory_nodes` 中 `type in ('event','thought')` 的记录
- `embedding_key` + `embeddings.json`  
  → `memory_nodes.embedding` 向量列（pgvector），`description` 直接作为 embedding 的文本来源
- `last_accessed`  
  → `memory_nodes.last_accessed` 时间戳列
- `poignancy`  
  → `memory_nodes.poignancy` 数值列
- `subject / predicate / object / keywords`  
  → `memory_nodes.subject / predicate / object / keywords`，用于后续实现 `retrieve()` 的关键词检索
- `filling`  
  → `memory_nodes.filling`（JSONB，用于保存证据节点 ID 等）
- `new_retrieve(persona, focal_points, n)`  
  → Dify 工作流 `Memory Retrieval`，一轮检索处理一个 `focal_text`，在 Code 节点中实现三因子加权打分。

### 1.3 完整字段映射表

| 原项目字段      | Dify/PostgreSQL 字段 | 类型           | 说明                              |
|-----------------|----------------------|----------------|-----------------------------------|
| `node_id`       | `id`                 | UUID / VARCHAR | 主键                              |
| `type`          | `type`               | VARCHAR        | `event` / `thought` / `chat`      |
| `created`       | `created_at`         | TIMESTAMP      | 创建时间                          |
| `last_accessed` | `last_accessed`      | TIMESTAMP      | 最近访问时间                      |
| `description`   | `description`        | TEXT           | 自然语言描述                      |
| `embedding_key` | —                    | —              | 直接用 `description` 作为 key     |
| embeddings.json | `embedding`          | `vector`       | pgvector 向量                     |
| `poignancy`     | `poignancy`          | FLOAT          | 重要性分数                        |
| `subject`       | `subject`            | VARCHAR        | S/P/O 三元组                      |
| `predicate`     | `predicate`          | VARCHAR        | S/P/O 三元组                      |
| `object`        | `object`             | VARCHAR        | S/P/O 三元组                      |
| `keywords`      | `keywords`           | TEXT[]         | 关键词数组                        |
| `filling`       | `filling`            | JSONB          | 补充信息 / 证据                   |
| `node_count`    | `node_count`         | INTEGER        | 全局计数                          |
| `type_count`    | `type_count`         | INTEGER        | 类型内计数                        |
| `depth`         | `depth`              | INTEGER        | 思考深度                          |

> 说明：本篇文档的检索实现主要使用 `type / description / embedding / last_accessed / poignancy`，但表结构上建议一次性补齐这些字段，方便后续扩展基于关键词的 `retrieve()` 以及更丰富的分析能力。

---

## 2. 数据层准备（PostgreSQL）

### 2.1 核心表：`memory_nodes`

参考 `check_database_structure.sql` 与原项目，建议 `memory_nodes` 至少包含如下字段：

- 基础字段（本篇检索实现必需）：
  - `id`：主键（UUID 或自增 ID）
  - `persona_name`：所属 Agent 名
  - `type`：`event` / `thought` / `chat`
  - `created_at`：创建时间
  - `last_accessed`：最近一次被检索或访问的时间
  - `description`：自然语言描述（对应原项目的 `embedding_key`）
  - `poignancy`：重要性评分（整数或浮点）
  - `embedding`：`vector` 类型列，保存文本向量（例如 bge-m3，维度 768/1024 等）

- 扩展字段（为 `retrieve()` 和调试预留）：
  - `subject` / `predicate` / `object`：S/P/O 三元组
  - `keywords`：TEXT[]，保存拆分后的关键词（例如从 S/P/O + description 中提取）
  - `filling`：JSONB，用于保存证据节点 ID 列表等结构化信息
  - `node_count` / `type_count` / `depth`：可选，用于复原原项目的统计信息与思考深度

推荐索引：

- `embedding` 上的 pgvector 索引（便于后续向量粗排）
- `(persona_name, type)` 普通索引（便于按角色 + 类型过滤）
- 如果要支持 `retrieve()` 的关键词检索，可对 `keywords` 建 GIN 索引。

### 2.2 向量写入与更新

- 记忆节点写入时，需在应用或 Dify 工作流中对 `description` 做一次 embedding，并写入 `memory_nodes.embedding`。
- 当某条记忆被检索后，需要更新其 `last_accessed = now()`，以保留“使用过的记忆会更近”的语义。
- 如果未来要实现类似原项目中“关键词强度统计 + 反思触发”机制，建议增加一张 `kw_strength` 统计表：

```sql
CREATE TABLE IF NOT EXISTS kw_strength (
  persona_name VARCHAR NOT NULL,
  keyword      VARCHAR NOT NULL,
  memory_type  VARCHAR NOT NULL, -- 'event' or 'thought'
  strength     INTEGER NOT NULL DEFAULT 1,
  PRIMARY KEY (persona_name, keyword, memory_type)
);
```

写入记忆时同步更新该表：

```sql
INSERT INTO kw_strength (persona_name, keyword, memory_type, strength)
VALUES (:persona_name, :keyword, :type, 1)
ON CONFLICT (persona_name, keyword, memory_type)
DO UPDATE SET strength = kw_strength.strength + 1;
```

> 这样就可以在 Dify 中复现原项目里 `kw_strength_event / kw_strength_thought` 的统计逻辑，用于反思触发或统计分析。

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
- `crud/update` 或 `execute_sql_simple`：更新 `last_accessed`（推荐）

---

## 4. Dify 工作流设计（Memory Retrieval）

### 4.1 工作流输入变量

在 Start 节点定义输入：

| 变量名          | 类型    | 说明                         | 默认值 |
|-----------------|---------|------------------------------|--------|
| `persona_name`  | String  | 当前检索的 Agent 名称        | -      |
| `focal_text`    | String  | 检索焦点描述                 | -      |
| `top_k`         | Number  | 需要返回的记忆条数           | 30     |
| `recency_w`     | Number  | 新近度权重（乘 gw[0]）       | 1.0    |
| `relevance_w`   | Number  | 相关性权重（乘 gw[1]）       | 1.0    |
| `importance_w`  | Number  | 重要性权重（乘 gw[2]）       | 1.0    |
| `recency_decay` | Number  | 新近度衰减系数               | 0.99   |

> 实际有效权重 = `scratch` 权重 × 全局增益 `gw`。默认情况下：
> - recency：`1.0 × 0.5 = 0.5`
> - relevance：`1.0 × 3.0 = 3.0`
> - importance：`1.0 × 2.0 = 2.0`

### 4.2 步骤 1：计算 `focal_text` 的向量

- **节点类型**：Embedding 节点（或自定义 Code 节点 + embedding 模型调用）
- **输入**：`focal_text`
- **输出**：`focal_embedding`（向量，List[float]）

建议使用与你在 `memory_nodes.embedding` 中相同的模型（例如 BAAI/bge-m3），保持向量空间一致。

### 4.3 步骤 2：查询候选记忆（SQL 节点）

- **节点类型**：API 工具节点（调用 `executeSqlSimple` 或 `crudSelect`）
- **节点作用**：按 persona 过滤，取 `event` + `thought` 类型的记忆作为候选集。

基础查询示例（伪 SQL，实际根据工具入参格式调整）：

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
ORDER BY last_accessed ASC;
```

> 说明：原项目在 Python 里用 `"idle" not in i.embedding_key` 过滤“空闲事件”。在 Dify 侧可以有三种实现方式（任选其一）：  
> - 在表中增加 `is_idle BOOLEAN` 字段，在 SQL 中 `AND is_idle = FALSE`；  
> - 如果有 `keywords` 字段，可 `AND NOT (keywords && ARRAY['idle'])`；  
> - 保持 SQL 简单，在 Code 节点中用 `if "idle" in description.lower(): continue` 做过滤（最易实现，本方案采用这一种）。

**性能优化选项**（可按数据量选择是否开启）：

- 限制时间范围，例如只看最近 7 天的记忆：

```sql
AND created_at > now() - interval '7 days'
```

- 使用 pgvector 做一次粗排，只取相似度最高的前 N 条再进 Code 精排：

```sql
WHERE persona_name = :persona_name
ORDER BY embedding <=> :focal_embedding
LIMIT 100;
```

### 4.4 步骤 3：在 Code 节点里计算三维评分并排序

- **节点类型**：Code 节点（Python）
- **输入**：
  - `db_rows`：上一步工具返回的记录数组
  - `focal_embedding`：焦点向量
  - 可选超参数：`recency_decay, recency_w, relevance_w, importance_w, top_k`

核心代码示例（与原 `new_retrieve` 等价的三因子打分）：

```python
import math
from typing import List, Dict, Any

def cos_sim(a: List[float], b: List[float]) -> float:
    """计算余弦相似度，处理维度不一致 / 空向量等边界情况。"""
    if not a or not b or len(a) != len(b):
        return 0.0

    dot = sum(x * y for x, y in zip(a, b))
    na = math.sqrt(sum(x * x for x in a))
    nb = math.sqrt(sum(x * x for x in b))

    if na < 1e-8 or nb < 1e-8:
        return 0.0

    return dot / (na * nb)

def normalize(d: Dict[str, float], target_min: float = 0.0, target_max: float = 1.0) -> Dict[str, float]:
    """归一化到指定范围 [target_min, target_max]，与原项目 normalize_dict_floats 逻辑一致。"""
    if not d:
        return {}

    vals = list(d.values())
    vmin, vmax = min(vals), max(vals)
    range_val = vmax - vmin

    if range_val == 0:
        mid = (target_max - target_min) / 2.0
        return {k: mid for k in d.keys()}

    return {
        k: ((v - vmin) * (target_max - target_min) / range_val + target_min)
        for k, v in d.items()
    }

def ensure_list_float(v: Any) -> List[float]:
    """将数据库返回的 embedding 转为 List[float]，并处理 None / 非列表情况。"""
    if v is None:
        return []
    if isinstance(v, list):
        return [float(x) for x in v]
    # 如果是字符串（例如 "[0.1, 0.2, ...]"），可在这里加 json.loads 解析
    return []

def main(arg):
    rows = arg.get("db_rows", [])          # 来自 SQL 节点
    focal_emb = ensure_list_float(arg.get("focal_embedding"))

    recency_decay = float(arg.get("recency_decay", 0.99))
    recency_w = float(arg.get("recency_w", 1.0))
    relevance_w = float(arg.get("relevance_w", 1.0))
    importance_w = float(arg.get("importance_w", 1.0))

    # 全局增益系数，与原项目 gw = [0.5, 3, 2] 一致
    # 实际有效权重 = scratch 权重 × gw：
    #   recency:    1.0 × 0.5 = 0.5
    #   relevance:  1.0 × 3.0 = 3.0
    #   importance: 1.0 × 2.0 = 2.0
    gw = [0.5, 3.0, 2.0]

    top_k = int(arg.get("top_k", 30))

    # 先过滤 idle 相关描述
    filtered_rows = []
    for r in rows:
        desc = (r.get("description") or "").lower()
        if "idle" in desc:
            continue
        filtered_rows.append(r)

    rows = filtered_rows

    # 没有候选记忆的情况
    if not rows:
        return {
            "retrieved_nodes": [],
            "accessed_ids": [],
            "status": "no_candidates",
            "message": "no candidate memories for this persona"
        }

    # 焦点向量为空或无效
    if not focal_emb:
        return {
            "retrieved_nodes": [],
            "accessed_ids": [],
            "status": "error",
            "message": "focal_embedding is empty or invalid"
        }

    # 1) 构造 ID 列表，保证顺序与 last_accessed 排序一致
    node_ids = [str(r["id"]) for r in rows]

    # 2) Recency：按位置指数衰减（与原 extract_recency 一致）
    recency_vals = [recency_decay ** i for i in range(1, len(node_ids) + 1)]
    recency_out = {node_ids[i]: recency_vals[i] for i in range(len(node_ids))}

    # 3) Importance：直接用 poignancy
    importance_out = {
        str(r["id"]): float(r.get("poignancy", 0.0)) for r in rows
    }

    # 4) Relevance：embedding 余弦相似度
    relevance_out = {}
    for r in rows:
        nid = str(r["id"])
        emb = ensure_list_float(r.get("embedding"))
        relevance_out[nid] = cos_sim(emb, focal_emb)

    # 5) 归一化到 [0,1]
    recency_n = normalize(recency_out, 0.0, 1.0)
    importance_n = normalize(importance_out, 0.0, 1.0)
    relevance_n = normalize(relevance_out, 0.0, 1.0)

    # 6) 加权合成最终得分
    master = {}
    for nid in node_ids:
        master[nid] = (
            recency_w   * recency_n.get(nid, 0.0)   * gw[0] +
            relevance_w * relevance_n.get(nid, 0.0) * gw[1] +
            importance_w* importance_n.get(nid, 0.0)* gw[2]
        )

    # 7) 排序取 Top-K
    sorted_ids = sorted(master.keys(), key=lambda x: master[x], reverse=True)
    sorted_ids = sorted_ids[:top_k]

    # 8) 组装输出（带上原始行 + 得分，并收集需要更新 last_accessed 的 ID）
    id_to_row = {str(r["id"]): r for r in rows}
    result = []
    accessed_ids = []
    for nid in sorted_ids:
        row = dict(id_to_row[nid])
        row["score"] = master[nid]
        result.append(row)
        accessed_ids.append(nid)

    debug = {
        "total_candidates": len(rows),
        "retrieved_count": len(result),
        "min_score": min(master.values()) if master else 0.0,
        "max_score": max(master.values()) if master else 0.0,
    }

    return {
        "retrieved_nodes": result,
        "accessed_ids": accessed_ids,
        "status": "ok",
        "debug": debug,
    }
```

> 上面的 `cos_sim / normalize` 与原项目 `cos_sim / normalize_dict_floats` 的逻辑保持一致，并额外处理了维度不一致、向量为空等边界情况；`status` / `debug` 字段便于在 Dify 中调试和做错误分支。

### 4.5 步骤 4：更新 `last_accessed`（推荐）

如果希望保留“被检索的记忆会变新”的行为，可以增加一个工具节点：

- **节点类型**：API 工具节点（`executeSqlSimple` / `crudUpdate`）
- **输入**：上一节点输出的 `accessed_ids` 数组
- **行为示例**：

SQL 示例（使用 `ANY` 批量更新）：

```sql
UPDATE memory_nodes
SET last_accessed = now()
WHERE id = ANY(:id_array::uuid[]);
```

在 Dify 中：

- 将 Code 节点输出的 `accessed_ids` 作为 JSON 数组传给工具节点参数，比如：  
  `{"id_array": {{code_node.accessed_ids}}}`
- 如果担心更新压力，可以：
  - 只更新前若干条（例如 top 10）
  - 或把更新操作放到单独的异步工作流里，由事件驱动执行。

> 并发注意：在多 persona / 多实例并发检索同一条记忆的场景下，`last_accessed` 更新会存在竞态。生产环境中可以考虑：
> - 在读取后更新前对行加锁：`SELECT ... FOR UPDATE`；
> - 增加乐观锁字段（如 `version`），更新时带上版本号；
> - 或降低更新频率（例如只每 N 次检索更新一次）。

### 4.6 步骤 5：输出给上游 LLM / 应用

最终在工作流的 End / Response 节点中输出：

- `retrieved_nodes`：按分数排序的记忆列表
- `status`：`ok / no_candidates / error`，上游可以根据此决定是否 fallback
- `debug`（可选）：用于调试的统计信息（可在生产环境中忽略）

---

## 5. 示例：在 Dify 中的节点编排

可以按照下面的顺序在 Dify 的「工作流编排」界面中创建节点：

1. **开始节点（Start）**
   - 输入：`persona_name`, `focal_text`, `top_k`, （可选）`recency_w` 等超参数
2. **Embedding 节点：`生成_focal_text_向量`**
   - 输入：`focal_text`
   - 输出：`focal_embedding`
3. **PostgreSQL 工具节点：`查询候选记忆`**
   - 调用 `executeSqlSimple` 或 `crudSelect`
   - 条件：`persona_name` + `type in ('event','thought')`
   - 输出：`db_rows`
4. **Code 节点：`计算检索得分并排序`**
   - 输入：`db_rows`, `focal_embedding`, 以及超参数
   - 输出：
     - `retrieved_nodes`：按综合得分排序的记忆列表
     - `accessed_ids`：被检索到的节点 ID 列表（用于更新 `last_accessed`）
     - `status` / `debug`：辅助信息（可用于错误分支和调试）
5. **PostgreSQL 工具节点（推荐）：`更新_last_accessed`**
   - 根据 `accessed_ids` 执行 `UPDATE`
6. **结束节点（End / Response）**
   - 返回 `retrieved_nodes`，以及可选的 `status` / `debug` 信息

---

## 6. 与原项目的差异与注意事项

- **多 `focal_points` 支持**：
  - 原项目 `new_retrieve` 支持同时对多个 `focal_points` 检索，这里示例以“一个工作流处理一个 `focal_text`”为主。
  - 如果需要，可以在 Start 节点传入一个 JSON 数组，在 Code 节点内循环处理，每个 focal 返回一组节点。

- **embedding 模型差异**：
  - 原项目使用 OpenAI `text-embedding-ada-002`，当前实现使用 bge-m3，只要读写时使用的都是同一模型，算法本身不受影响。

- **错误处理与数据质量**：
  - 原项目中，如果节点没有 embedding，会在写入时补全；在 Dify 实现中，建议在写入记忆时就确保 `embedding` 不为空。
  - 在 Code 节点中，如果 `embedding` 为 `None` 或维度与 `focal_embedding` 不一致，`cos_sim` 会返回 0，避免抛异常。
  - 如果 PostgreSQL 驱动返回的 `embedding` 是字符串（如 `'[0.1, 0.2, ...]'`），需要在 `ensure_list_float` 中增加 `json.loads` 解析逻辑。

- **性能优化**：
  - 对于记忆量较大（几十万条以上）的场景，应优先在 SQL 层做预过滤（时间范围 / 向量粗排），再在 Code 节点做精排。
  - 可以对“热 persona”做缓存，避免每次都从数据库拉全量候选。

- **权重调参**：
  - 可以在 Dify 的变量中暴露 `recency_w / relevance_w / importance_w / recency_decay`，允许在不同应用、不同 persona 间做个性化调节。

- **并发控制**：
  - 原项目在单进程内存中运行，不涉及数据库层面的并发问题。
  - 在 Dify + PostgreSQL 架构下，需要注意：
    - 记忆写入时的 `node_count` / `type_count` 自增可以用 SEQUENCE 或数据库自动生成列替代；
    - `last_accessed` 更新的并发控制（见 4.5 节的建议）；
    - 如果在工具或 Code 节点中调用外部 embedding 服务，需要确认客户端是线程安全的（或在工作流中串行使用）。

### 6.6 生产环境部署建议

- 使用连接池管理数据库连接（例如 pgBouncer / 应用层连接池），避免每个请求新建连接。
- 对 embedding 模型调用增加熔断和降级机制（例如超时返回默认 embedding 或跳过该记忆）。
- 监控 `last_accessed` 更新频率，必要时将更新操作异步化，以减轻主检索路径的压力。
- 根据业务需求定期备份 `memory_nodes` 和相关统计表（如 `kw_strength`），并考虑归档历史记忆（冷数据分表或归档库）。

通过上述设计，你就可以在 Dify 中较完整地复现 Generative Agents 中的记忆检索逻辑，并且保留了进一步调参、扩展 `retrieve()` 关键词检索和性能优化的空间。 
