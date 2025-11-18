# Generative Agents 规划模块 / 工作流分析文档

本文档基于 `generative_agents` 项目中 `reverie/backend_server/persona` 相关源码，总结智能体的**规划模块（Plan）**及其在整体认知工作流中的作用，作为后续在 Dify 等平台上复刻 / 改造规划模块的参考。

---

## 1. 整体认知工作流概览

在 `reverie/backend_server/persona/persona.py` 中，每个 Persona 在一个时间步（tick）内执行的主循环为：

1. `perceive(maze)`：感知当前环境中的事件（人物、对象、事件发生等）  
2. `retrieve(perceived)`：基于感知到的事件，从联想记忆（associative memory）中检索相关事件与思想  
3. `plan(maze, personas, new_day, retrieved)`：结合长期日程、当前时间与检索结果，确定**当前要做的事情**，并决定是否对他人/外部事件做出反应  
4. `reflect()`：对一天中的经历进行反思与抽象，总结出新的“思想”写入记忆，更新身份与目标  
5. `execute(maze, personas, plan)`：将「当前要做的事情」转化为具体的执行动作（移动到哪个 tile、使用哪个对象、显示什么 emoji 等）

因此，从“工作流”的视角，可以把 Persona 的认知过程抽象为：

> **感知 → 检索 → 规划 → 反思 → 执行**

其中的 **规划模块（Plan）** 既要消费检索模块的输出（`retrieved`），又需要为执行模块提供输入（当前 action 的地址、描述等），同时还与记忆/身份管理紧密耦合。

---

## 2. 规划模块的整体职责（`plan.py`）

`reverie/backend_server/persona/cognitive_modules/plan.py` 的核心职责可以分为四类：

1. **长期规划（Long-term Planning）**  
   - 生成“今天的总体日程”：起床时间、各时间段的大致活动、睡觉时间等。
   - 将这些计划以**小时级**和**分钟级**的结构化日程保存到短期记忆（Scratch）中。

2. **短期规划与任务分解（Short-term Planning & Decomposition）**  
   - 在当前时间点，根据日程表决定“此刻应该做什么”。  
   - 对大块任务（例如连续几小时的工作）进行分钟级细化分解，得到可执行的小任务序列。

3. **事件驱动的行为调整（Event-driven Adjustment）**  
   - 接收检索模块输出的 `retrieved`，挑选当前最需要关注的外部事件。  
   - 决定是否对其他 Persona 进行**对话**、**等待**或忽略。  
   - 将“聊天”“等待”等行为插入现有日程，并重新分解相关时间段。

4. **与身份 / 记忆的双向联动**  
   - 每天早上，根据最近的记忆调整 Persona 的“当前状态描述”（currently）与今日计划需求（daily plan requirement）。  
   - 将每日计划写入联想记忆，以便后续检索时可以回忆“我当天本来打算做什么”。

总体而言，Plan 模块扮演的是一个**计划与调度引擎**：

- 长期规划：负责“今天的大方向”。  
- 短期规划：负责“此刻具体做什么”。  
- 事件反应：允许外部事件打断或插入新的行为（例如临时聊天）。  

---

## 3. 长期规划：生成每日计划（Long-term Planning）

长期规划的核心入口是 `_long_term_planning(persona, new_day)`。它在 `Persona.plan(...)` 中被调用，仅在 `new_day` 为 `"First day"` 或 `"New day"` 时触发。

### 3.1 生成起床时间：`generate_wake_up_hour`

- 函数：`generate_wake_up_hour(persona)`  
- 实现：调用 `run_gpt_prompt_wake_up_hour(persona)`，通过 LLM 决定 Persona 在当天几点起床。  
- Prompt 主要依赖：
  - 身份稳定集（identity stable set）：`scratch.get_str_iss()`  
  - 生活方式：`scratch.lifestyle`  
  - 姓名：`scratch.first_name`
- 输出：一个整数小时数，例如 `8` 表示 `08:00`，作为后续全天日程的起点。

### 3.2 生成“日计划需求”：`daily_req` / `daily_plan_req`

Persona 暂存两类“日计划需求”：

- `scratch.daily_req`：结构化的**大致日计划条目**，如：
  - “wake up and complete the morning routine at 6:00 am”
  - “work on painting project from 8:00 am to 12:00 pm”
  - “have lunch at 12:00 pm”
- `scratch.daily_plan_req`：以文本形式表达的“今天打算做什么”，通常由 LLM 在身份修订时生成。

根据 `new_day` 的不同取值，处理逻辑略有差异：

#### 3.2.1 初始模拟日（`new_day == "First day"`）

- 调用 `generate_first_daily_plan(persona, wake_up_hour)`：  
  - 使用 LLM 直接根据 Persona 的身份与起床时间生成一串带时间的计划条目。  
  - 输出写入 `persona.scratch.daily_req`。

#### 3.2.2 新的一天（`new_day == "New day"`）

- 调用 `revise_identity(persona)`：  
  - 根据昨天及最近一段时间的记忆，更新 Persona 的当前状态 `currently` 与今日计划需求 `daily_plan_req`。  
- 当前代码中 `daily_req` 仍继承原值；但新的 `daily_plan_req` 会影响随后各类 LLM prompt 中对 Persona 的描述。

### 3.3 小时级 → 分钟级日程：`generate_hourly_schedule`

- 函数：`generate_hourly_schedule(persona, wake_up_hour)`  
- 主要步骤：
  1. 创建 24 小时的整点字符串列表：`"00:00 AM" ~ "11:00 PM"`。  
  2. 对每个小时：
     - 起床前的小时统一视为 `"sleeping"`。  
     - 起床后的每个小时由 LLM 生成该时段的主要活动描述。  
  3. 为增加多样性，会多次采样并对结果做简单筛选。  
  4. 将连续相同活动合并（压缩），然后把“小时数”转换为“分钟数”，得到结构类似：
     - `[['sleeping', 360], ['waking up and starting her morning routine', 60], ...]`

生成结果写入：

- `scratch.f_daily_schedule`：  
  - 以 `[任务描述, 持续分钟数]` 为元素的列表，既包含大块活动，也会在后续被进一步分解。  
- `scratch.f_daily_schedule_hourly_org`：  
  - 刚开始与 `f_daily_schedule` 一致，保留“未分解前”的小时级版本，后续在插入新行为（例如聊天）时用于确定所属大块时间段。

### 3.4 将当天计划写入联想记忆

长期规划完成后，Plan 模块会构造一条“思想（thought）”类型的记忆：

- 文本形如：
  - `This is Isabella's plan for Monday March 15: ...`（将 `daily_req` 拼接而成）  
- 添加关键词 `["plan"]`、设定一定的有效期（如 30 天），并附带文本 embedding。  
- 然后通过 `persona.a_mem.add_thought(...)` 写入 associative memory。

目的：

- 之后在检索阶段，如果需要“回忆自己今天原本计划做什么”，就可以检索到这条记忆作为上下文。

---

## 4. 身份与目标更新：`revise_identity`

函数位置：`plan.py` 中的 `revise_identity(persona)`。  
调用时机：当 `new_day == "New day"`，即**跨天后的新一天早晨**。

该函数旨在根据最近经历更新 Persona 的：

- 当前状态描述：`scratch.currently`  
- 今日计划需求：`scratch.daily_plan_req`

### 4.1 基于记忆检索近期关键事件

1. 构造关注点（focal points）：
   - `"{name}'s plan for {当前日期}."`  
   - `"Important recent events for {name}'s life."`
2. 调用增强版检索函数 `new_retrieve(persona, focal_points)`，获取一批按时间排序的记忆节点（事件与思想）。
3. 将这些节点整理成 `[Statements]` 文本列表，每条包含时间戳和简要描述（`embedding_key`）。

### 4.2 LLM 生成“计划建议”和“情感总结”

基于 `[Statements]`，Plan 模块向 LLM 提出两个问题：

1. `plan_prompt`：  
   - “根据上述陈述，在规划 *今天* 时，有什么是 {name} 需要记住的？如果有日程信息，请尽可能具体（日期、时间、地点）。”  
   - 要求用第一人称视角回答。→ 输出 `plan_note`。

2. `thought_prompt`：  
   - “如何总结 {name} 最近几天的感受？”  
   - 同样要求第一人称视角。→ 输出 `thought_note`。

### 4.3 更新当前状态：`scratch.currently`

再构造一个 `currently_prompt`，内容包括：

- 昨天的 `status`（`scratch.currently`）；  
- 昨天结束时的 `plan_note + thought_note`；  
- 当前日期。

要求 LLM 以第三人称视角写出**今天**的 Persona 状态描述（Status: ...），包括：

- 目前在做什么、在想什么；  
- 如果有日程信息，也要求具体（日期、时间、地点）。

LLM 的输出写回 `persona.scratch.currently`。

### 4.4 生成新的日计划需求：`scratch.daily_plan_req`

最后，Plan 模块用 `scratch.get_str_iss()` + 当前日期构造 prompt，要求 LLM：

- 写出 4–6 条带时间的“今日大致计划”（如“中午 12 点吃饭、晚上 7–8 点看电视”等）。  
- 输出写回 `persona.scratch.daily_plan_req`。

注意：

- `daily_plan_req` 更多用于描述性语境（作为后续 prompt 的一部分）；  
- 结构化的 `daily_req` 与 `f_daily_schedule` 仍由 `generate_first_daily_plan` / `generate_hourly_schedule` 决定。

---

## 5. 短期规划与任务分解：`_determine_action`

在主函数 `plan(...)` 中，当 `persona.scratch.act_check_finished()` 返回 True（当前行为结束）时，会调用 `_determine_action(persona, maze)` 生成下一个 action。

### 5.1 在日程中的时间定位

Plan 模块首先需要知道“当前时间在日程表中的位置”：

- `curr_index = scratch.get_f_daily_schedule_index()`  
  - 根据 `scratch.curr_time` 计算“今天已过去多少分钟”；  
  - 在 `f_daily_schedule` 里累计 duration，找到第一个“累计时间 > 已过去分钟数”的索引，即当前任务索引。  

- `curr_index_60 = scratch.get_f_daily_schedule_index(advance=60)`  
  - 表示“1 小时后”的对应索引，用于预先分解未来一段时间的任务。

### 5.2 判定任务是否需要分解：`determine_decomp`

内部 helper 函数 `determine_decomp(act_desp, act_dura)`：

- 睡眠相关任务通常不分解：
  - 包含 `"sleeping"`, `"asleep"`, `"in bed"` 等关键字；  
  - 或者包含 `"sleep"` / `"bed"` 且时长较长（>60 分钟）。  
- 其他长时任务（持续时间 ≥ 60 分钟）倾向分解成细粒度子任务。

### 5.3 分解策略：始终保持未来至少两小时的细粒度计划

核心思想：**当前时间点起向后看至少 2 小时，保证这些时间段中的任务已经分解成分钟级的小任务。**

- 如果 `curr_index == 0`（一天的第一个小时内）：
  - 对当前 `curr_index` 对应的任务（如果 ≥60 分钟且符合分解条件）执行分解；  
  - 同时对 `curr_index_60 + 1` 对应的任务尝试分解，以覆盖两小时窗口。

- 如果 `curr_index_60 < len(f_daily_schedule)` 且当前小时 < 23：
  - 对 `curr_index_60` 对应任务执行分解（若满足条件）。

分解调用：`generate_task_decomp(persona, act_desp, act_dura)`：

- LLM 将一个大块任务（例如“working on her painting for 240 minutes”）拆成若干小任务，每个子任务约 5–30 分钟，并带有更细致的描述。

### 5.4 全天时长补齐

为了保证整天跨度为 1440 分钟，Plan 模块会：

- 计算当前 `f_daily_schedule` 总时长 `x_emergency`；  
- 如果总时长 < 1440，则追加一个 `["sleeping", 1440 - x_emergency]` 的任务块。

这一步确保日程结构总是完整的。

### 5.5 生成当前 action 的完整描述

在完成必要的分解与补齐后：

1. 取当前任务：`act_desp, act_dura = f_daily_schedule[curr_index]`。  
2. 确定行为发生的地点：
   - 当前 tile → `maze.access_tile(curr_tile)["world"]` 得到 world 名称；  
   - `generate_action_sector(act_desp, persona, maze)` → 房屋/区域（sector）；  
   - `generate_action_arena(act_desp, persona, maze, act_world, act_sector)` → 具体房间/场景（arena）；  
   - 拼接成地址：`"{world}:{sector}:{arena}"`。  
   - `generate_action_game_object(act_desp, act_address, persona, maze)` → 选择在该地址下的具体对象（例如 `"bed"`, `"sofa"`），若无可选对象则可能返回 `"<random>"`。

3. 生成语义与事件结构：
   - `generate_action_pronunciatio(act_desp, persona)`：
     - 将行为描述转换为 emoji 序列；  
     - 如果调用失败，使用 🙂 作为 fallback。  
   - `generate_action_event_triple(act_desp, persona)`：
     - 将行为转换为一个 (subject, predicate, object) 事件三元组，便于写入联想记忆。  
   - `generate_act_obj_desc(...)` / `generate_act_obj_event_triple(...)`：
     - 生成与所用对象相关的描述与事件三元组。

4. 写入短期记忆：`scratch.add_new_action(...)`：
   - 设置：
     - `act_address`、`act_duration`、`act_description`、`act_pronunciatio`、`act_event`；  
     - 对象相关信息（`act_obj_description` 等）；  
     - 聊天状态初始化（`chatting_with` 等）；  
     - `act_start_time = curr_time`，`act_path_set = False`（供执行模块进行路径规划）。

从执行工作流的角度看，这一步就是：

> 根据当前时间从长期日程中取出下一件事 → 必要时先做任务分解 → 决定在哪里、做什么、使用什么对象 → 写入 Scratch 作为当前 action。

---

## 6. 事件驱动的行为调整：对他人和外部事件的反应

长期与短期计划只决定了“默认轨迹”。Plan 模块还需要基于 `retrieved` 对他人行为或外部事件做出反应。

### 6.1 聚焦一个待响应事件：`_choose_retrieved`

`retrieved` 的结构为：

```text
retrieved[event.description] = {
  "curr_event": <ConceptNode>,
  "events": [<ConceptNode>, ...],
  "thoughts": [<ConceptNode>, ...]
}
```

`_choose_retrieved(persona, retrieved)` 的逻辑：

1. 复制一份 `retrieved`，剔除所有“主体是自己”的事件（self events），因为这些不是“需要响应的外部刺激”。  
2. 第一优先级：选择 `curr_event.subject` 不含 `":"` 且不等于 Persona 本身的事件（即其他 Persona 的事件）。  
3. 如果没有符合条件的 Persona 事件，则从非 “is idle” 的事件中随机选择一个。  
4. 如果仍没有合适事件，则返回 `None`。

返回值是一个小字典（包含 `"curr_event" / "events" / "thoughts"`），在后续 `_should_react` 中使用。

### 6.2 决定是否与他人互动：`_should_react`

`_should_react(persona, focused_event, personas)` 的主要逻辑：

1. 如当前正在聊天（`scratch.chatting_with`）或当前地址包含 `<waiting>`，直接返回 False（不反应）。  
2. 从 `focused_event` 中取出 `curr_event`。  
3. 如果 `curr_event.subject` 不含 `":"`（代表另一个 Persona）：
   - 优先尝试 `lets_talk(init_persona, target_persona, retrieved)`：  
     - 条件包括：
       - 双方当前都有有效的 action/address/description；  
       - 双方不在睡觉、不在等待、不已经在聊天；  
       - 不是晚上 23 点；  
       - 不在冷却期（`chatting_with_buffer`）；  
     - 调用 `generate_decide_to_talk(init_persona, target_persona, retrieved)`（LLM）返回 yes/no。  
     - 如为 True，返回 `"chat with {name}"`。  
   - 如果不聊天，则尝试 `lets_react(init_persona, target_persona, retrieved)`：
     - 必须双方有 action/address/description；  
     - 不能睡觉、不能等待、init_persona 要有 `planned_path`（在执行某个行动路径）；  
     - 双方必须在同一地址（`act_address` 相等）；  
     - 不在晚上 23 点以后；  
     - 再调用 `generate_decide_to_react(...)`：返回 `"1"` / `"2"` / 其他。  
       - `"1"`：生成一个 `"wait: {time}"` 形式的反应（等待对方实施完当前行为）；  
       - `"2"` 或其他：当前实现大多返回 False（暂不采取特殊行为）。

最终 `_should_react` 的输出有三种：

- `"chat with {persona_name}"`：需要发起聊天；  
- `"wait: {具体时间}"`：需要等待到对方当前动作结束；  
- `False`：不基于该事件调整当前行为。

### 6.3 插入聊天 / 等待行为：`_chat_react` / `_wait_react` / `_create_react`

当 `_should_react` 返回非 False 时：

1. **聊天行为：`_chat_react`**
   - 调用 `generate_convo(maze, init_persona, target_persona)`：  
     - 使用对话模块（`converse.py`）生成多轮对话；  
     - 根据对话文本长度估算持续时间（分钟）。  
   - 使用 `generate_convo_summary(init_persona, convo)` 将对话压缩为一句总结语作为行为描述。  
   - 对发起者和目标 Persona 各生成一个新的 action：
     - 地址设为 `<persona> {对方姓名}`；  
     - `act_event` 为 `(self.name, "chat with", other.name)`；  
     - 设置 `chatting_with = 对方`, 并在 `chatting_with_buffer` 中为对方设置一个较大冷却值（如 800），防止对话刷屏。  
   - 使用 `_create_react(...)` 将该聊天任务插入当前日程，并重新分解对应时间段。

2. **等待行为：`_wait_react`**
   - 基于对方当前 action 的开始时间 + 时长，计算需要等待的结束时间；  
   - 构造一个描述类如 `"waiting to start ..."` 的等待任务和其持续时间；  
   - 同样通过 `_create_react(...)` 插入日程中合适位置。

3. **日程重排：`_create_react` + `generate_new_decomp_schedule`**
   - `_create_react` 首先通过 `f_daily_schedule_hourly_org` 确定当前所属的大块小时区间（`start_hour` ~ `end_hour`），然后在 `f_daily_schedule` 中找到对应 index 范围。  
   - 调用 `generate_new_decomp_schedule(persona, inserted_act, inserted_act_dur, start_hour, end_hour)`：  
     - 重新生成这一时间段的细粒度任务列表，在其中嵌入新插入的聊天/等待行为。  
   - 替换 `f_daily_schedule[start_index:end_index]` 为新的分解结果，并通过 `scratch.add_new_action(...)` 将新行为作为当前 action 生效。

### 6.4 聊天状态维护与冷却

在 `plan(...)` 末尾，Plan 模块负责清理与维护聊天相关状态：

- 若当前 `act_event` 不再是 `"chat with"`，则清空：
  - `scratch.chatting_with`  
  - `scratch.chat`  
  - `scratch.chatting_end_time`
- 针对 `chatting_with_buffer` 中所有 Persona：
  - 若当前未与其聊天，则每 tick 将缓冲值减 1，直到冷却结束，才允许再次触发聊天。

---

## 7. 规划模块与执行模块的接口

Plan 模块的对外输出主要有两部分：

1. `plan(...)` 的返回值：  
   - `persona.scratch.act_address`（当前行为的目标地点）。  
2. 对 `scratch` 的状态更新：  
   - `act_description`、`act_pronunciatio`（emoji）、`act_event`（事件三元组）；  
   - `act_obj_description` 及其 event；  
   - 聊天相关状态（`chatting_with` 等）。

执行模块 `execute(maze, personas, plan)` 在此基础上：

- 使用路径规划模块（`path_finder.py`）计算从当前 tile 到 `act_address` 的路径；  
- 驱动 Persona 在环境中逐步移动；  
- 向前端输出当前行为对应的 emoji 与文本描述。

因此可以理解为：

- Plan 决定“高层意图 + 目标地点 + 行为语义”；  
- Execute 决定“如何路径性地把这个行为执行出来”。

---

## 8. 作为 Dify / 其他平台实现的参考抽象

从一个更抽象的“工作流”视角，可以将原始 Plan 模块拆解为以下几个可组合组件（便于在 Dify、Orchestration Engine 等平台上实现）：

1. **长期计划生成节点（Long-term Planner）**
   - 输入：Persona 的 identity stable set、lifestyle、当前日期。  
   - 输出：
     - 起床时间；  
     - 日计划条目列表（`daily_req`）；  
     - 小时级日程（`f_daily_schedule_hourly_org`）；  
     - 分钟级日程（`f_daily_schedule`）。

2. **身份与目标更新节点（Identity Refiner）**
   - 输入：最近的事件与思想记忆（由检索模块提供）、上一日状态 `currently`。  
   - 输出：
     - 新的 `currently` 描述；  
     - 新的 `daily_plan_req` 文本；  
     - 若需要，也可回写日计划相关记忆。

3. **短期任务调度与分解节点（Short-term Scheduler & Decomposer）**
   - 输入：
     - 完整日程（`f_daily_schedule`）；  
     - 当前时间；  
     - Maze 中的当前位置信息。  
   - 输出：
     - 当前 action（描述、时长）；  
     - 动作发生地点地址（world/sector/arena/object 等）。

4. **事件反应节点（Event Reaction Planner）**
   - 输入：
     - 当前感知事件 + 检索结果（`retrieved`）；  
     - 所有 Persona 的当前行为状态。  
   - 输出：
     - 是否发起聊天；  
     - 是否等待某人；  
     - 若需要，在日程中插入新的行为，并返回更新后的当前 action。

5. **执行节点（Executor）**
   - 输入：
     - 当前 action 的地址、时长、描述、emoji；  
     - 当前 tile，Maze 拓扑结构。  
   - 输出：
     - 下一步移动位置、前端显示的表情与文本说明。

在 Dify 或类似编排平台中，可以将上述组件映射为一条链式或带条件分支的工作流，为每个组件定义：

- 所需输入变量（比如 persona_id、当前时间、事件列表）；  
- 中间结果存储表（例如 Postgres 中的日程表、当前 action 表）；  
- 与感知/检索模块的接口（事件/记忆 ID 列表等）。

本档主要侧重对原项目规划模块的还原性分析；在此基础上，可以针对具体平台分别设计对应的工作流配置与数据模型。

