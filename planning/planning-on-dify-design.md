# Dify 上实现 Smallville 规划模块的详细步骤说明

本文件基于 `planning-module-analysis.md` 中对规划模块的分析，总结如何在 Dify 上实现 Smallville 的 Plan 模块（长期规划 + 短期调度 + 事件反应），并与现有的感知 / 检索模块及 `dify_postgres_tool` 集成。

假设前提：

- 已有：  
  - 感知模块工作流（感知环境事件，并写入 / 返回事件列表）；  
  - 检索模块工作流（对 `associative_nodes` 做向量检索）；  
  - `dify_postgres_tool` 已配置好，可以对 Postgres 执行 SQL。  
- 规划模块需要复刻 Smallville 中 `plan.py` + `scratch.py` 的核心行为与工程规则。

---

## 1. 数据模型与存储设计

### 1.1 Persona 状态（Scratch）表设计

目标：将 Smallville 中 `Scratch` 类的关键字段落地到 Postgres 中，使 Dify 工作流能通过工具读写这些状态。

建议创建（或扩展）一个 `persona_scratch` 表（或等价结构），核心字段包括：

- 身份与状态：
  - `persona_id` —— 主键或联合主键的一部分  
  - `currently` TEXT —— Persona 当前状态描述（Status）  
  - `daily_plan_req` TEXT —— 今日计划需求（文本）  
  - `daily_req` JSONB —— 大致日计划条目列表，例如：  
    - `["wake up and ... at 6:00 am", "work on ... from 9:00 am to 12:00 pm", ...]`

- 日程计划：
  - `f_daily_schedule` JSONB —— 分钟级日程列表，形如：  
    - `[[ "sleeping", 360 ], [ "wakes up and ...", 60 ], ...]`  
  - `f_daily_schedule_hourly_org` JSONB —— 小时级原始日程（未分解版本），同样是 `[ [task, duration_min], ... ]`。

- 当前行为（Action）：
  - `act_address` TEXT —— 行为发生地址，如 `"world:sector:arena"` 或 `"<persona> name"`  
  - `act_start_time` TIMESTAMP —— 行为开始时间（游戏世界时间）  
  - `act_duration_min` INTEGER —— 行为持续分钟数  
  - `act_description` TEXT —— 行为描述  
  - `act_pronunciatio` TEXT —— 行为 emoji 串  
  - `act_event` JSONB —— 行为事件三元组 `(s, p, o)`  
  - `act_obj_description` TEXT —— 行为对象描述（可选）  
  - `act_obj_event` JSONB —— 与对象相关的事件三元组

- 聊天相关状态：
  - `chatting_with` TEXT —— 当前聊天对象姓名（无则 NULL）  
  - `chatting_end_time` TIMESTAMP —— 本次聊天结束时间（对齐到整分钟）  
  - `chatting_with_buffer` JSONB —— 对象 -> 冷却 buffer 值（对应 Plan 中的 800 冷却）  
  - `chat` JSONB/TEXT —— 当前对话内容（可按需要存）

- 其他辅助字段：
  - `curr_time` TIMESTAMP —— Persona 感知的当前世界时间  
  - `planned_path` JSONB —— 如你也实现 Execute 层，可用于存路径  
  - 其他身份字段（name/age/innate/learned/lifestyle）可放在 persona 基础信息表或这个表中。

> 简化方案：也可以将整个 Scratch 结构作为一个 JSONB 字段存储，只要工具层能方便读写关键字段即可。

### 1.2 日程明细表（可选）

如果你希望日程可被 SQL 细粒度操作，而不仅仅是 JSON，可以额外建表：

```sql
CREATE TABLE daily_schedule_items (
  persona_id    TEXT,
  date          DATE,
  seq           INT,
  description   TEXT,
  duration_min  INT,
  is_hourly_org BOOLEAN,
  PRIMARY KEY (persona_id, date, seq)
);
```

你可以只维护 JSON 版（简单），也可以将 JSON 与行级结构同步（灵活）。

### 1.3 计划记忆：复用 `associative_nodes`

延用检索模块使用的记忆表（如下名称仅为示例）：

- `associative_nodes`：
  - `id` UUID  
  - `type` TEXT —— `'event' | 'thought' | ...'`  
  - `subject` TEXT  
  - `predicate` TEXT  
  - `object` TEXT  
  - `embedding_key` TEXT —— 用于简要描述该记忆（如 “Isabella's plan for Monday March 15”）  
  - `embedding` VECTOR/JSONB —— 向量  
  - `poignancy` INT —— 重要性；计划记忆固定为 5  
  - `expires_at` TIMESTAMP —— 计划记忆固定为 “当前时间 + 30 天”

规划模块在长程规划阶段，会写入一条“计划类 Thought”，供检索模块后续使用。

---

## 2. Dify 工具接口设计（基于 `dify_postgres_tool`）

在 `dify_postgres_tool` 的 `openapi.yaml` 中，为规划模块准备一组工具 API。可以是「通用 SQL 执行 + 模板 SQL」，也可以是语义化工具。推荐语义化，便于重用：

1. `get_persona_scratch`  
   - 输入：`persona_id`  
   - 输出：一整行 Scratch 状态（JSON）。

2. `update_persona_scratch`  
   - 输入：`persona_id` + 一个 JSON patch（包含需要更新的字段及新值）。  
   - SQL：用动态更新或多列更新实现。

3. `upsert_daily_schedule`  
   - 输入：`persona_id`, `date`, `f_daily_schedule`, `f_daily_schedule_hourly_org`（均 JSON）。  
   - 输出：无。  
   - 内部实现：更新 `persona_scratch` 表中的日程字段。

4. `append_sleep_padding`  
   - 输入：`persona_id`, `date`（或直接只用 `persona_id`，从 Scratch 中取当前日程）。  
   - 逻辑：  
     - 读取 `f_daily_schedule`，累加 `duration_min` 总和；  
     - 若 `< 1440`，追加一条 `["sleeping", 1440 - sum]` 并写回。

5. `insert_plan_thought`  
   - 输入：`persona_id`, `date`, `plan_text`, `embedding`。  
   - 输出：新插入行的 `id`（可选）。  
   - 固定逻辑：  
     - `type = 'thought'`；  
     - `subject = persona_name`；`predicate = 'plan'`；`object = date_str`；  
     - `poignancy = 5`；  
     - `expires_at = curr_time + interval '30 days'`。

6. 事件反应辅助工具（可选）：  
   - `get_persona_curr_state(persona_id)` —— 获取 act_address / act_description / planned_path，多用于聊天 / 等待决策；  
   - `update_daily_schedule_with_reaction` —— 封装 “找到对应小时区间，重分解并插入聊天 / 等待行为” 的 SQL/脚本逻辑。

在 Dify 中，将这些接口注册为工具，供工作流节点调用。

---

## 3. 长期规划工作流：`planning_long_term`

### 3.1 工作流定位

- 名称：`planning_long_term`  
- 触发时机：  
  - 每个 Persona 的「第一天」；  
  - 每次跨天（日期变化）时。  
- 输入：  
  - `persona_id`  
  - `curr_time`（当前游戏世界时间）

### 3.2 步骤拆分

**步骤 1：加载 Persona 状态**

- 工具节点：`get_persona_scratch`。  
- 根据返回的 `curr_time` 判断：
  - 若为空：视为第一天 `new_day = "First day"`；  
  - 若日期与输入 `curr_time` 不同：`new_day = "New day"`；  
  - 否则：`new_day = False`（本工作流可直接返回）。

**步骤 2：生成起床时间**

- LLM 节点：`generate_wake_up_hour`  
  - Prompt 对齐 Smallville：输入 identity stable set + lifestyle + first_name，输出一个整数小时（0–23）。  
  - 在 Dify 中可要求 LLM 只输出数字。

**步骤 3：生成 / 刷新日计划需求**

- 条件节点：  
  - 若 `new_day == "First day"`：  
    - LLM 节点 `generate_first_daily_plan`：输入 Persona 信息和 `wake_up_hour`，输出 `daily_req` 列表；  
  - 若 `new_day == "New day"`：  
    - 调用「记忆检索工作流」获得最近一段计划 / 重要事件；  
    - LLM 节点 `revise_identity`：生成新的 `currently` 与 `daily_plan_req`；  
    - `daily_req` 暂时沿用原值（与 Smallville 保持一致）。  
- 工具节点：`update_persona_scratch`，更新：
  - `currently`、`daily_plan_req`、`daily_req`。

**步骤 4：生成小时级 / 分钟级日程**

- LLM 节点：`generate_hourly_schedule`  
  - 输入：`daily_req` + `wake_up_hour`；  
  - 输出：24 小时的任务列表。建议让 LLM 直接输出分钟级数组（方便后续操作），你可以再从中派生小时级版本。  
  - 在 Prompt 中要求活动要有多样性（至少若干不同活动）。  
- 表达式节点：从 LLM 输出里构造：
  - `f_daily_schedule_hourly_org`（原始块列表）；  
  - `f_daily_schedule`（相同内容，后续会被分解）。  
- 工具节点：`upsert_daily_schedule` 写入上述两字段。

**步骤 5：补齐 24 小时**

- 工具节点：`append_sleep_padding`。  
  - 遍历 `f_daily_schedule`，若总时长 `< 1440`，追加 `"sleeping"`。

**步骤 6：写入“计划记忆”**

- LLM 节点：根据 `daily_req` 生成一条完整的计划描述（类似 `This is X's plan for Monday March 15: ...`）；  
- 调用 embedding 工具得到向量；  
- 工具节点：`insert_plan_thought` 写入 `associative_nodes`。

**步骤 7：更新 `curr_time`**

- 工具节点：`update_persona_scratch` 将 `curr_time` 更新为当前游戏时间。

> 输出：无显式输出，效果是 Persona 当天的长期计划和记忆已更新，供后续短期规划和执行使用。

---

## 4. 短期规划 / 调度工作流：`planning_short_term`

### 4.1 工作流定位

- 名称：`planning_short_term`  
- 每个 tick（或每 N 秒）在主仿真工作流中调用，位于「感知 → 检索」之后、「执行」之前。  
- 输入：
  - `persona_id`  
  - `curr_time`  
  - `retrieved`（来自检索模块的结构化事件 / 思想集合）

### 4.2 步骤拆分

**步骤 1：加载 Scratch 与日程**

- 工具节点：`get_persona_scratch`。  
- 如发现 `curr_time` 日期 != Scratch 中日期，可在上层工作流先调用 `planning_long_term`。

**步骤 2：判断当前行为是否结束**

- 表达式节点（复刻 `act_check_finished`）：  
  - 若 `act_address` 为空 → 视为需要新 Action；  
  - 若 `chatting_with` 非空 → 使用 `chatting_end_time` 与 `curr_time` 比较；  
  - 否则：
    - 取 `act_start_time`，若秒不为 0，则对齐到下一个整分钟；  
    - 加上 `act_duration_min` 得 `end_time`；  
    - 若 `end_time == curr_time`（精确到秒或分钟） → 行为结束。  
- 若未结束，直接输出当前 `act_address` 与描述，短期规划在本 tick 不做其它动作。

**步骤 3：计算当前所在的日程 index**

- 表达式节点：  
  - 将 `curr_time` 转为当天已过去分钟数 `today_min_elapsed`；  
  - 遍历 `f_daily_schedule`（JSON 数组），累加 duration，找到第一个 `elapsed > today_min_elapsed` 的 index，作为 `curr_index`；  
  - 同样可计算 `advance=60` 时的 `curr_index_60` 用于“向前看一小时”。

**步骤 4：根据阈值做任务分解**

- 条件：若当前小时 `< 23` 才进入分解流程。  
- 内部逻辑对齐 Smallville：
  - 若 `curr_index == 0`：对第 0 个任务和 `curr_index_60+1` 任务做分解考虑；  
  - 否则，对 `curr_index_60` 对应任务做分解考虑；  
  - 对每个候选任务：
    - 若 `duration >= 60` 且描述不包含睡眠关键词（`sleeping`, `asleep`, `in bed` 等）→ 触发分解；  
    - LLM 节点：`task_decomp`（对齐 `task_decomp_v3`），输出一组 `[sub_task, sub_duration_min]`，最小粒度 5 分钟；  
    - 工具节点：用新子任务列表替换 `f_daily_schedule` 中对应区间。

**步骤 5：再次补齐 1440 分钟**

- 再调用一次 `append_sleep_padding`，保证全天总长为 1440。

**步骤 6：生成当前 Action**

- 重新计算一次 `curr_index`，读取 `act_desp, act_dura`。  
- 位置与对象选择：
  - 如果你已实现地图 / 房间结构，可：  
    - 工具节点：根据当前 tile 取 world；  
    - LLM 节点：`generate_action_sector`、`generate_action_arena` → sector / arena；  
    - 拼接地址为 `"{world}:{sector}:{arena}"`；  
    - 如需要对象级别，可用 LLM 或工具从该地址下挑选对象（失败时默认 `"<random>"`）。  
  - 若暂不需要精细空间建模，也可以简单让 LLM 返回一个逻辑地址字符串（例如 `"home:studio:desk"`）。
- 语义生成：
  - LLM 节点：  
    - `generate_action_pronunciatio` → emoji（失败回退 🙂）；  
    - `generate_action_event_triple` → 行为事件三元组；  
    - `generate_act_obj_desc` / `generate_act_obj_event_triple` → 对象描述与事件。  
- 工具节点：`update_persona_scratch` 写入：
  - `act_address`, `act_start_time = curr_time`, `act_duration_min = act_dura`；  
  - `act_description`, `act_pronunciatio`, `act_event` 等；  
  - 清空 `chatting_with` / `chat` / `chatting_end_time`；  
  - 将 `act_path_set = false`（留给 Execute 模块使用）。

**步骤 7：输出给执行模块**

- 将以下字段作为工作流输出，交给“执行工作流”使用：
  - `act_address`  
  - `act_description`  
  - `act_pronunciatio`

---

## 5. 事件反应集成（聊天 / 等待）

事件反应可以作为 `planning_short_term` 的后半段（或单独工作流）实现。核心逻辑来自 `_choose_retrieved` + `_should_react` + `_chat_react` + `_wait_react`。

### 5.1 选择要反应的事件

1. 从 `retrieved` 中剔除所有 `curr_event.subject == persona_name` 的条目。  
2. 优先选择 `curr_event.subject` 不含 `":"` 且不等于当前 persona 名字的条目（其他 Persona 的事件）。  
3. 若不存在上述条目，则从非 `"is idle"` 的事件中随机选一个。  
4. 若仍无可选事件，则不进入反应流程。

### 5.2 过滤条件与 LLM 决策

1. 基础过滤（对应 `_should_react` 顶层）：  
   - 若当前 persona 正在聊天（`chatting_with` 非空）→ 不反应；  
   - 若当前 `act_address` 包含 `<waiting>` → 不反应。

2. 尝试聊天：`lets_talk`  
   - 获取目标 Persona 的 Scratch 状态（工具 `get_persona_scratch`）。  
   - 条件过滤：
     - 双方都有 `act_address` 和 `act_description`；  
     - 任一方行为描述包含睡眠关键词 → 不聊天；  
     - 当前小时为 23 → 不聊天；  
     - 目标 `act_address` 包含 `<waiting>` → 不聊天；  
     - 任一方 `chatting_with` 非空 → 不聊天；  
     - 若目标在 `init_persona.chatting_with_buffer` 中，且 buffer 值 > 0 → 不聊天。  
   - 若通过上述过滤：LLM 节点 `decide_to_talk`（Prompt 对齐 `decide_to_talk_v2`），输出 yes/no。  
   - 若为 yes → `reaction_mode = "chat with {target_name}"`。

3. 如果不聊天，再尝试「等待 / 其他反应」：`lets_react`  
   - 条件过滤：
     - 同上，先排除睡眠、23 点、waiting 等；  
     - `init_persona.planned_path` 不能为空；  
     - `init_persona.act_address == target_persona.act_address`（同一地点）。  
   - 若通过上述过滤：LLM 节点 `decide_to_react`，输出 `"1"` / `"2"` / 其他：  
     - `"1"`：生成等待行为（`"wait: {time}"`）；  
     - 其他：不反应。

4. 根据 `reaction_mode` 分支：  
   - `"chat with X"` → 进入聊天插入流程；  
   - `"wait: ..."` → 进入等待插入流程；  
   - 否则：不调整日程。

### 5.3 插入聊天 / 等待行为

1. 聊天插入（对应 `_chat_react` + `_create_react`）：
   - 生成对话内容和持续时间：
     - 可调用已有的对话模块工作流，生成多轮对话文本；  
     - 用字符数 / token 数估算持续分钟数。  
   - LLM 节点：`generate_convo_summary`，生成摘要描述，用作聊天行为描述。  
   - 计算聊天结束时间：
     - 若 `curr_time.second != 0`，对齐到下一个整分钟，再加上持续分钟数。  
   - 使用一个“更新日程工具”（或 SQL 脚本）：
     - 基于 `f_daily_schedule_hourly_org` 计算当前所属的大块小时区间（`start_hour` / `end_hour`）；  
     - 以该区间为范围，调用一个逻辑类似 `generate_new_decomp_schedule` 的过程重新生成细粒度任务，并插入聊天行为。  
   - 更新双方 Scratch：
     - `act_address = "<persona> 对方名"`；  
     - `act_event = (self_name, "chat with", other_name)`；  
     - `chatting_with = 对方名`；  
     - `chatting_with_buffer[对方名] = 800`；  
     - `chatting_end_time = 上面计算的结束时间`；  
     - `act_description` 使用对话摘要，`act_duration_min` 为聊天时长。

2. 等待插入（对应 `_wait_react`）：
   - 根据目标 Persona 当前 `act_start_time` 和 `act_duration_min` 计算其结束时间；  
   - 构造等待行为描述（例如 `"waiting to start ..."`）及等待时长。  
   - 使用同样的“更新日程工具”插入等待行为；  
   - 更新发起者 Scratch：
     - `act_address` 中包含 `<waiting>`；  
     - `act_event` 记录为等待型事件。

3. 聊天冷却递减：
   - 在 `planning_short_term` 末尾，增加一个表达式 + 工具节点：  
     - 对 `chatting_with_buffer` 中所有 key，若当前 `chatting_with` 不等于该 key，则将 buffer 值减 1（下限 0）；  
     - 回写 JSON 到 Scratch。

---

## 6. 测试与验证建议

为了确保 Dify 实现与 Smallville 行为一致，建议在以下层面做测试：

1. **阈值测试**  
   - 构造固定 persona 与日程配置，验证：  
     - 时长 < 60 分钟的任务不会被分解；  
     - 全天总时长 < 1440 时，会自动补齐睡眠；  
     - 23 点之后不会触发新的聊天 / 分解。

2. **边界条件测试**  
   - 构造以下场景：  
     - Persona 正在睡觉 / waiting / chatting；  
     - `planned_path` 为空；  
     - `chatting_with_buffer` > 0。  
   - 验证在这些状态下，聊天 / 等待不会被错误触发。

3. **事件反应 & 冷却测试**  
   - 让两个 Persona 多次在同一地点相遇，观测：  
     - 第一次可以聊天；  
     - 之后在冷却 800 tick 内不会立刻再次触发聊天；  
     - 冷却递减后，可以再次聊天。

4. **回归测试**  
   - 选择 1 个典型 Persona（例如 Isabella），为其设计几天的脚本化事件流：  
     - 多次跨天、与不同人物相遇、不同任务长度组合；  
   - 在 Dify 工作流跑完整模拟后，检查数据库中的：  
     - `f_daily_schedule` 变化；  
     - `act_*` 字段演化；  
     - 计划类记忆写入情况；  
     - 行为是否符合文档中的规则和直觉。

---

## 7. 小结

本文件将 `planning-module-analysis.md` 中对 Smallville 规划模块的分析，转换为在 Dify 上可落地的工程实现步骤，重点包括：

- 数据模型：如何用 Postgres/JSONB 表达 Scratch 与日程；  
- 工具接口：如何通过 `dify_postgres_tool` 暴露读写 Scratch、日程与计划记忆的能力；  
- 长期规划工作流：`planning_long_term` 的节点拆分与 LLM / 工具调用顺序；  
- 短期规划 / 调度工作流：`planning_short_term` 如何做分解、补齐与当前 Action 生成；  
- 事件反应集成：聊天 / 等待插入以及冷却机制如何迁移；  
- 测试建议：如何验证阈值、边界条件和交互行为的正确性。

在此基础上，你可以参照已有的“感知模块实现文档” / “检索模块实现文档”，为规划模块在 Dify 中补充截图级配置说明（每个节点的 Prompt 文本、工具绑定、变量映射等），进一步形成完整的实现文档集。

---

## 8. 并发、事务与性能优化建议（可选增强）

你朋友的补充意见主要集中在「并发控制」「事务完整性」「时间对齐」「LLM 调用频率」「内存使用」等工程层面，这些都是在真正上线时非常有价值的优化点。它们不会改变前文的整体架构，但可以作为实施阶段的增强选项参考。

### 8.1 并发控制：乐观锁防止日程覆盖

问题：  
当多个工作流实例（或多个 Persona 进程）同时更新同一 Persona 的 `f_daily_schedule` 时，可能出现“后写覆盖前写”的竞态。

建议做法（示例）：

- 在 `persona_scratch` 新增一个版本列：

```sql
ALTER TABLE persona_scratch
ADD COLUMN schedule_version INTEGER DEFAULT 0;
```

- 在工具 `upsert_daily_schedule` 中：  
  - 先读取当前 `schedule_version`；  
  - 更新时使用 `WHERE persona_id = ? AND schedule_version = ?` 做乐观锁；  
  - 若受影响行数为 0，说明有并发修改，工作流可选择重试或放弃本次更新。

### 8.2 事务完整性：工具内部打包关键更新

问题：  
类似 `f_daily_schedule` 与 `f_daily_schedule_hourly_org` 这种“必须同时更新”的字段，如果分别在多个工具或多个 SQL 里更新，任何一步失败都可能导致状态不一致。

建议：

- 将“多字段必须一致”的更新封装到同一个工具中，由该工具在数据库层面开启事务：  
  - 例如 `upsert_daily_schedule` 同时更新两个字段；  
  - 或者新增一个 `update_schedule_and_scratch` 工具，用单次事务完成多个相关更新。  
- 对特别关键的操作（如插入聊天 / 等待并重写一段日程）也建议通过单个工具接口完成，以便在工具内部使用事务。

### 8.3 时间同步：统一在数据库层面对齐

问题：  
在工作流里用表达式处理秒级对齐（`if curr_time.second != 0: ...`）会增加复杂度，也容易出现前后不一致。

建议：

- 在数据库中新增一个对齐后的时间字段，例如 `aligned_curr_time`；  
- 每次写入 / 更新 `curr_time` 时，统一用：

```sql
UPDATE persona_scratch
SET curr_time = :raw_time,
    aligned_curr_time = DATE_TRUNC('minute', :raw_time);
```

- 后续所有基于时间的判断（行为结束、日程 index 计算等）都基于 `aligned_curr_time`，这样可以在表达式节点中简化逻辑，同时保证一致性。

### 8.4 LLM 调用频率与缓存策略

问题：  
`planning_short_term` 在每个 tick 都可能触发 `task_decomp` 等 LLM 调用，如果 Persona 数量多、tick 频率高，会消耗大量配额。

建议：

- 在日程数据结构中加一个标记，表示某个任务是否已经分解，例如：

```json
["working on painting", 180, {"decomposed": true}]
```

- 在触发分解前增加条件：  
  - `duration >= 60`；  
  - `decomposed` 不为 true；  
  - （可选）距离上次分解尝试已超过一个时间阈值。  
- 可以在工具或中间表中做简单缓存（例如以 “任务描述 + 时长” 为 key 的 task_decomp 结果），重复场景可以复用已有分解结果，而不是每次都重新询问 LLM。

### 8.5 内存与存储优化：归档与压缩

问题：  
若长期保留所有 Persona 的全天日程 JSONB，且每天写入一份完整拷贝，数据量会持续膨胀。

建议：

- 已在第 1 节中建议将日程拆到 `daily_schedule_items` 表；在此基础上可进一步：  
  - 对连续相同任务做压缩：例如 `["sleeping", 60]` 连续多条可合并；  
  - 对历史日程定期归档到历史表，主表只保留最近 N 天；  
  - 若只关心“计划记忆”，可以在归档前生成一条摘要 Thought（类似 `_long_term_planning` 已做的事）。

---

## 9. 实施与运维建议（分阶段落地）

为了降低一次性实现全部功能的风险，可以按你朋友建议的方式分阶段实施：

1. **第一阶段：核心功能**  
   - 先打通长期规划：起床时间 + `daily_req` + 全天日程。  
   - 实现最简版的短期规划：根据当前时间取当前任务，生成一个基本 `act_address` + 描述即可。  
   - 暂时不实现事件反应（聊天 / 等待）和复杂冷却机制。

2. **第二阶段：完善功能**  
   - 引入任务分解（`task_decomp`）和 60 分钟阈值逻辑。  
   - 引入事件反应工作流，支持聊天 / 等待插入。  
   - 实现聊天冷却（800）以及聊天状态管理。

3. **第三阶段：优化体验与性能**  
   - 引入乐观锁与事务封装；  
   - 优化 LLM 调用频率与缓存策略；  
   - 增加监控与调试工具（见下）。

### 9.1 监控与调试

建议在系统中增加以下监控与调试能力：

- 统计：  
  - 每类 LLM 调用的次数、失败率、平均耗时；  
  - 每类工具调用（SQL）的耗时与错误率；  
  - 工作流执行成功 / 失败次数。  
- 调试：  
  - 日程变更历史追踪（可在 `daily_schedule_items` 或单独 log 表中记录）；  
  - Persona 状态快照（定期保存 Scratch 快照用于回放）；  
  - 工作流执行链路的可视化（Dify 自带可视化 + 自定义日志）。

### 9.2 配置管理

将所有关键阈值、LLM 参数、性能开关放入统一配置（环境变量或配置表），例如：

```yaml
planning:
  thresholds:
    task_decomposition: 60      # 分钟
    chat_cooldown: 800          # tick
    day_total_minutes: 1440
    memory_expiry_days: 30
  llm:
    max_retries: 3
    timeout: 30s
    temperature: 0.7
  performance:
    enable_cache: true
    cache_ttl: 300s
    batch_size: 10
```

这样一方面便于不同环境调整参数，另一方面也方便将来调优。
