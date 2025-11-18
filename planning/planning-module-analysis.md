# Generative Agents 规划模块 / 工作流分析文档（含关键实现细节）

本文档基于 `generative_agents` 项目中 `reverie/backend_server/persona` 相关源码，总结智能体的**规划模块（Plan）**及其在整体认知工作流中的作用，并补充了在代码层面体现的各种阈值、边界条件和 Prompt 工程细节，方便在 Dify 等平台上高保真复现 Smallville 系统。

---

## 1. 整体认知工作流概览

在 `reverie/backend_server/persona/persona.py` 中，每个 Persona 在一个时间步（tick）内执行的主循环为：

1. `perceive(maze)`：感知当前环境中的事件（人物、对象、事件发生等）。  
2. `retrieve(perceived)`：基于感知到的事件，从联想记忆（associative memory）中检索相关事件与思想。  
3. `plan(maze, personas, new_day, retrieved)`：结合长期日程、当前时间与检索结果，确定**当前要做的事情**，并决定是否对他人 / 外部事件做出反应。  
4. `reflect()`：对一天中的经历进行反思，总结出新的“思想”写入记忆，更新身份与目标。  
5. `execute(maze, personas, plan)`：将「当前要做的事情」转化为具体的执行动作（移动到哪个 tile、使用哪个对象、显示什么 emoji 等）。

可以将 Persona 的认知过程抽象为：

> **感知 → 检索 → 规划 → 反思 → 执行**

其中的 **规划模块（`plan.py`）** 是“中枢”：  
一方面消费检索模块的输出（`retrieved`），另一方面驱动执行模块的输入（当前 Action），并与记忆 / 身份管理进行双向交互。

---

## 2. 规划模块的整体职责（`plan.py`）

`reverie/backend_server/persona/cognitive_modules/plan.py` 的核心职责可以分为四类：

1. **长期规划（Long-term Planning）**  
   - 生成“今天的总体日程”：起床时间、各时间段的大致活动、睡觉时间等。  
   - 以小时级 / 分钟级的结构化日程保存到短期记忆（Scratch）中。

2. **短期规划与任务分解（Short-term Planning & Decomposition）**  
   - 在当前时间点，根据日程表决定“此刻应该做什么”。  
   - 对长任务进行分钟级细化分解，得到可执行的小任务序列。

3. **事件驱动的行为调整（Event-driven Adjustment）**  
   - 接收检索模块输出的 `retrieved`，挑选当前最需要关注的外部事件。  
   - 决定是否对其他 Persona 进行对话、等待或忽略，并在日程中插入这些行为。

4. **与身份 / 记忆的双向联动**  
   - 每天早上，根据最近记忆更新 Persona 的“当前状态描述”（currently）与“今日计划需求”（daily plan requirement）。  
   - 将每日计划写入联想记忆，便于后续检索。

从工程视角看，Plan 模块就是一个**计划与调度引擎**：

- 长期规划：决定“今天的大方向”；  
- 短期规划：决定“此刻具体做什么”；  
- 事件反应：允许外部事件打断或插入新的行为；  
- 同时对“时间”“记忆”“社交节奏”等维度施加很多精细的硬编码规则。

---

## 3. Prompt 工程与模板体系

规划模块高度依赖一套精心设计的 Prompt 模板，这部分在高层架构中容易被忽略，但对行为稳定性至关重要。

### 3.1 多版本模板演进

- Prompt 模板目录：`reverie/backend_server/persona/prompt_template/`。  
- 包含多个版本：`v1`、`v2`、`v3_ChatGPT`，体现从早期 GPT-3 到 ChatGPT 的演进。  
- 例如 `v2` 中存在多版日程生成模板：`daily_planning_v3.txt`、`v4`、`v5`、`v6` 等；聊天与反应也有 `decide_to_talk_v1/v2`、`decide_to_react_v1/v2`。

### 3.2 标准化变量占位符

以 `v2/task_decomp_v3.txt` 为例，开头定义了变量语义：

- `!<INPUT 0>! -- Commonset`（identity stable set）  
- `!<INPUT 4>! -- Current action`  
- `!<INPUT 6>! -- Current action duration in min` 等。

这些占位符由 `persona/prompt_template/run_gpt_prompt.py` 中的 `generate_prompt` 统一替换，保证：

- Prompt 结构稳定；  
- 修改变量来源不需要逐处硬改 Prompt 文本。

### 3.3 少样本学习与 5 分钟增量分解

`task_decomp_v3.txt` 中包含一个完整的 Kelly Bronson 示例，用于指导任务分解：

- 指令明确要求：  
  > Describe subtasks in 5 min increments.  
  > In 5 min increments, list the subtasks Kelly does ...
- 示例中展示了一个 180 分钟任务如何被拆成若干子任务：  
  - 有统一格式：`(duration in minutes: X, minutes left: Y)`；  
  - `minutes left` 递减到 0。

实际运行时，`generate_task_decomp` 会调用对应的 Prompt，LLM 被强制学习这种 5 分钟粒度的标准化分解模式。

### 3.4 时间格式约束

多个 Prompt（如 `wake_up_hour_v1`、`daily_planning_v*`、`task_decomp_v3`）都要求使用统一时间串格式：

- 起床时间如 `"6:00 am"`；  
- 区间如 `"09:00am ~ 12:00pm"`。

这与 `run_gpt_prompt_*` 中的解析逻辑配合，降低了解析与对齐失败的概率。

---

## 4. 长期规划流程：生成每日计划

长期规划入口：`_long_term_planning(persona, new_day)`。  
只在 `new_day` 为 `"First day"` 或 `"New day"` 时调用。

### 4.1 起床时间：`generate_wake_up_hour`

- 调用 `run_gpt_prompt_wake_up_hour(persona)`，返回一个整数小时；  
- Prompt 使用：
  - `scratch.get_str_iss()`（identity stable set：姓名、年龄、性格、目前状况、生活方式等）；  
  - `scratch.lifestyle`；  
  - `scratch.first_name`。

### 4.2 日计划需求：`daily_req` 与 `daily_plan_req`

Persona 有两层“日计划需求”表示：

- `scratch.daily_req`：结构化的计划条目（带时间描述）；  
- `scratch.daily_plan_req`：由 `revise_identity` 生成的文本性“今日计划需求”。

根据 `new_day` 不同取值，处理略有不同：

1. `new_day == "First day"`：  
   - 调用 `generate_first_daily_plan(persona, wake_up_hour)`，由 LLM 直接生成若干条“带时间”的计划，写入 `daily_req`。  

2. `new_day == "New day"`：  
   - 调用 `revise_identity(persona)`（见第 6 节），更新 `currently` 和 `daily_plan_req`；  
   - `daily_req` 暂时沿用上一天的结构化描述。

### 4.3 小时级 → 分钟级日程：`generate_hourly_schedule`

`generate_hourly_schedule(persona, wake_up_hour)` 的关键点：

1. **3 次多样性采样 + 至少 5 种活动**  
   - `diversity_repeat_count = 3`；  
   - 每轮生成后统计去重后的活动种类数，若 `< 5` 则清空并重新生成；  
   - 目的是避免 LLM 给出“全程只做一件事”的极简输出。

2. **起床前自动填充睡眠**  
   - 起床时间之前的小时统一填 `"sleeping"`；起床后每个小时调用 `run_gpt_prompt_generate_hourly_schedule`。

3. **压缩连续相同行为并转为分钟**  
   - 将 `["sleeping", "sleeping", ...]` 等连续块合并为 `[task, hours]`；  
   - 再乘以 60 变成分钟数，得到：
     - `[['sleeping', 360], ['waking up and starting her morning routine', 60], ...]`

结果写入：

- `scratch.f_daily_schedule`：分钟级任务序列；  
- `scratch.f_daily_schedule_hourly_org`：未分解的小时级版本，用于事件插入时定位。

### 4.4 把计划写入联想记忆

在 `_long_term_planning` 尾部，Plan 模块构造一条“计划类思想”记忆：

- 文本：`This is {name}'s plan for {date}: ...`（拼接 `daily_req`）；  
- 有效期：`expiration = curr_time + 30 days`；  
- 权重：`thought_poignancy = 5`；  
- 关键词：`{"plan"}`；  
- 通过 `persona.a_mem.add_thought(...)` 写入 associative memory。

这些常数（30 天、权重 5）在代码中是硬编码的，对记忆的衰减和检索都有影响。

---

## 5. 短期规划与任务分解：`_determine_action`

当 `persona.scratch.act_check_finished()` 为 True 时，`plan(...)` 调用 `_determine_action(persona, maze)` 生成下一个 Action。

### 5.1 时间定位：当前在日程表中的位置

使用 Scratch 提供的两个索引函数：

- `get_f_daily_schedule_index()`：基于 `curr_time` 计算“今天已过去多少分钟”，在 `f_daily_schedule` 中累加 duration，找到当前任务索引。  
- `get_f_daily_schedule_index(advance=60)`：向后看一小时的任务索引，用于预先分解未来一段时间。

### 5.2 60 分钟分解阈值与睡眠判断

内部函数 `determine_decomp(act_desp, act_dura)`：

- 睡眠相关任务一般不分解：
  - 若描述中包含 `"sleeping"`, `"asleep"`, `"in bed"`，直接返回 False；  
  - 若只包含 `"sleep"` 或 `"bed"`，且时长 > 60 分钟，也视为不分解。  
- 其他任务：  
  - 若时长 ≥ 60 分钟，倾向进行分解。

### 5.3 分解策略：保证未来两小时细粒度

逻辑要点：

1. **第一小时特殊处理**  
   - 若 `curr_index == 0`，除了当前块外，还会对大约两小时后的块尝试分解（`curr_index_60 + 1`）。  

2. **全天其他时间**  
   - 若 `curr_index_60 < len(f_daily_schedule)` 且当前小时 `< 23`，对 `curr_index_60` 对应任务进行分解尝试。  
   - 23 点之后不再进行分解（夜间“静默”规则）。

实际分解通过 `generate_task_decomp(persona, act_desp, act_dura)` 调用 LLM，输出一组按 5 分钟增量拆解的子任务（参见 3.3）。

### 5.4 1440 分钟补齐：一天必须是 24 小时

在 `_determine_action` 中，Plan 模块会：

1. 遍历 `f_daily_schedule`，累加所有 duration，得到 `x_emergency`；  
2. 若 `1440 - x_emergency > 0`，则追加一条 `["sleeping", 1440 - x_emergency]`。

这既是“日程补齐”，也是一种时长纠错机制：即使前面 LLM 输出不精确，也会在这里被拉回到 24 小时总长。

### 5.5 当前 Action 的生成

在完成分解与补齐后：

1. 取当前任务：`act_desp, act_dura = f_daily_schedule[curr_index]`。  
2. 位置规划：
   - 从 `maze.access_tile(curr_tile)` 取到 `world`；  
   - `generate_action_sector` 决定 sector（类似建筑 / 区域）；  
   - `generate_action_arena` 决定 arena（具体房间 / 场景）；  
   - 拼成 `act_address = "{world}:{sector}:{arena}"`；  
   - `generate_action_game_object` 尝试在该地址下选择具体对象：
     - 若 `s_mem` 中无可用对象，则返回 `"<random>"` 作为回退占位符。

3. 语义与事件结构：
   - `generate_action_pronunciatio` 生成 emoji 序列：  
     - 内部采用 `try/except` 包裹，失败或返回空字符串时统一回退为 `"🙂"`。  
   - `generate_action_event_triple` 将行为描述转成事件三元组；  
   - `generate_act_obj_desc` / `generate_act_obj_event_triple` 类似地对参与对象生成描述和事件。

4. 写入 Scratch：`scratch.add_new_action(...)`  
   - 设置地址、时长、描述、emoji、事件三元组；  
   - 初始化聊天相关状态；  
   - 将 `act_start_time` 置为当前时间，并将 `act_path_set` 标记为 False（留给 `execute` 做路径规划）。

---

## 6. 身份与目标更新：`revise_identity`

函数 `revise_identity(persona)` 只在 `new_day == "New day"` 时调用，用于在跨天后重新整理 Persona 的身份叙事和今日目标。

### 6.1 基于记忆检索近期关键事件

1. 构造 focal points：
   - `"{name}'s plan for {curr_date}."`；  
   - `"Important recent events for {name}'s life."`。  
2. 调用 `new_retrieve(persona, focal_points)`，返回按时间排序的一组事件 / 思想节点。  
3. 将其格式化为 `[Statements]` 文本，每行包含时间和 `embedding_key`。

### 6.2 生成“计划建议”和“情感总结”

基于 `[Statements]`，向 LLM 提出两个问题：

1. `plan_prompt`：  
   - “上述陈述中，有什么是 {name} 在规划 *今天* 时需要记住的？若有日程信息，请给出具体时间 / 地点。”  
   - 要求第一人称视角。→ 输出 `plan_note`。  

2. `thought_prompt`：  
   - “如何总结 {name} 最近几天的感受？”  
   - 同样要求第一人称视角。→ 输出 `thought_note`。

### 6.3 更新当前状态：`scratch.currently`

构造 `currently_prompt`，内容包括：

- 昨天的 `status`（`scratch.currently` 原值）；  
- 昨天结束时的 `plan_note + thought_note`；  
- 当前日期。

要求 LLM 以第三人称视角写出**今天**的 Persona 状态描述，并尽量包含具体时间 / 地点信息。  
输出写回 `persona.scratch.currently`。

### 6.4 生成新的 `daily_plan_req`

之后使用 `scratch.get_str_iss()` + 当前日期构造 prompt，要求 LLM 写出 4–6 条带时间的“今日大致计划”，输出存入 `scratch.daily_plan_req`。  
这部分更偏“文本描述”，但会在后续各种 Prompt 中不断被引用，影响长程行为。

---

## 7. 事件驱动的行为调整：对他人和外部事件的反应

在主函数 `plan(...)` 中，长期与短期计划确定后，Plan 模块会基于 `retrieved` 对他人行为或外部事件做出反应。

### 7.1 聚焦一个事件：`_choose_retrieved`

`retrieved` 结构：

- `retrieved[event.description] = {"curr_event": E, "events": [...], "thoughts": [...]}`。

`_choose_retrieved(persona, retrieved)` 的逻辑：

1. 复制一份并剔除所有 `curr_event.subject == persona.name` 的事件，避免对“自发行为”产生反应。  
2. 优先选择 `"curr_event.subject"` 不含 `":"` 且不等于自己名字的事件（即其他 Persona 的事件）。  
3. 若无合适 Persona 事件，则从非 `"is idle"` 事件中随机选一个。  
4. 若仍无合适事件，返回 `None`。

### 7.2 是否反应、如何反应：`_should_react`

`_should_react(persona, focused_event, personas)` 内部包含两个 helper：

1. `lets_talk(init_persona, target_persona, retrieved)`：  
   - 条件检查（任一不满足则不聊天）：  
     - 双方当前都有有效的 `act_address` 和 `act_description`；  
     - 任一方行为描述中含 `"sleeping"` 则不聊天；  
     - 当前时间小时为 23 时（23 点）不聊天；  
     - 任一方处于 `<waiting>` 状态不聊天；  
     - 任一方已经在聊天 (`chatting_with`) 不聊天；  
     - 若 target 在 init 的 `chatting_with_buffer` 中，且 buffer 值 > 0，也不聊天。  
   - 若通过上述过滤，则调用 `generate_decide_to_talk`（LLM）返回 yes/no。  
   - 若为 yes，返回 `"chat with {target_name}"`。

2. `lets_react(init_persona, target_persona, retrieved)`：  
   - 条件检查：  
     - 双方必须有 `act_address` 与 `act_description`；  
     - 任一方行为描述含 `"sleeping"` 时不反应；  
     - 当前时间小时为 23 时不反应；  
     - 目标行为描述含 `"waiting"` 时不反应；  
     - `init_persona.scratch.planned_path` 不能为空，空则不反应；  
     - 双方 `act_address` 必须相等（同一地点）。  
   - 满足条件后调用 `generate_decide_to_react`（LLM），返回 `"1"` / `"2"` / 其他：  
     - `"1"`：返回 `"wait: {具体时间}"`，表示等待对方当前动作结束；  
     - `"2"` 或其他：当前实现大多返回 False（不做特殊反应）。

`_should_react` 顶层流程：

- 若当前 Persona 正在聊天（`chatting_with`）或当前地址包含 `<waiting>`，直接返回 False。  
- 否则对 `focused_event["curr_event"]` 执行上述逻辑，返回：
  - `"chat with {name}"`；  
  - `"wait: {time_str}"`；  
  - 或 False（不调整行为）。

### 7.3 插入聊天 / 等待行为：`_chat_react` / `_wait_react` / `_create_react`

1. **聊天行为：`_chat_react`**  
   - 通过 `generate_convo(maze, init, target)` 生成多轮对话，并估算持续分钟数；  
   - 用 `generate_convo_summary` 压缩成一句话行为描述；  
   - 对发起者和目标分别构造新的 Action：  
     - 地址：`"<persona> 对方姓名"`；  
     - 事件：`(self.name, "chat with", other.name)`；  
     - `chatting_with` 设为对方名字；  
     - `chatting_with_buffer[对方名字] = 800`，形成约 13.3 小时的冷却期。  
   - 调用 `_create_react(...)` 将聊天行为插入日程，并通过 `generate_new_decomp_schedule` 重分解对应时间段。

2. **等待行为：`_wait_react`**  
   - 根据目标 Persona 当前 action 的 `act_start_time` 和 `act_duration` 计算结束时间；  
   - 构造类似 `"waiting to start ..."` 的描述，并计算等待分钟数；  
   - 同样通过 `_create_react(...)` 插入日程。

3. **日程重排：`_create_react` + `generate_new_decomp_schedule`**  
   - `_create_react` 利用 `f_daily_schedule_hourly_org` 计算当前归属的大块小时区间（`start_hour` ~ `end_hour`），在 `f_daily_schedule` 中找到对应 index 范围；  
   - 调用 `generate_new_decomp_schedule(persona, inserted_act, inserted_act_dur, start_hour, end_hour)`：  
     - 重新生成这段时间的细粒度任务列表，在其中嵌入聊天 / 等待行为；  
   - 替换原有片段，并通过 `scratch.add_new_action(...)` 将新行为设为当前 Action。

### 7.4 聊天状态维护与冷却递减

在 `plan(...)` 尾部，Plan 模块对聊天状态进行维护：

- 若当前 `act_event` 不再是 `"chat with"`，则清空：  
  - `scratch.chatting_with`、`scratch.chat`、`scratch.chatting_end_time`。  
- 遍历 `chatting_with_buffer`：  
  - 对当前未在聊天的每个 Persona，将其 buffer 值减 1；  
  - 与 `_chat_react` 中的初始值 800 配合，实现“每 tick 递减”的冷却机制。

---

## 8. 规划模块与执行模块的接口

Plan 模块的对外输出主要有两部分：

1. `plan(...)` 函数返回值：  
   - `persona.scratch.act_address`（当前行为的目标地点）。  
2. 对 `scratch` 的状态更新：  
   - `act_description`、`act_pronunciatio`（emoji）、`act_event`（事件三元组）；  
   - `act_obj_description` 及其事件；  
   - 聊天相关状态（`chatting_with` 等）。

执行模块 `execute(maze, personas, plan)` 在此基础上：

- 使用 `path_finder.py` 规划从当前 tile 到 `act_address` 的路径；  
- 驱动 Persona 在环境中逐步移动；  
- 向前端输出当前行为对应的 emoji 与文本描述。

可以理解为：

- Plan 决定“高层意图 + 目标地点 + 行为语义”；  
- Execute 决定“如何把这个意图以路径形式执行出来”。

---

## 9. 阈值、边界条件与错误处理体系小结

从上述各节可以看到，原始实现中存在一整套**硬编码的阈值与边界规则**，它们并非“细枝末节”，而是保证行为真实感和系统稳定性的关键：

- 时间相关常数：  
  - 5 分钟为任务分解的最小粒度（由 Prompt 约束）；  
  - 60 分钟为“需要分解”的任务时长阈值；  
  - 1440 分钟为每日时间基准，通过补齐睡眠块确保完整性；  
  - 30 天为“计划记忆”的有效期；  
  - 聊天结束与行动结束时间均对齐到“整分钟”（秒级处理后再加分钟）。

- 社交节奏与边界条件：  
  - 23 点以后不再分解任务，也不发起新的交互（夜间静默）；  
  - 睡眠相关动作既不分解，也不作为触发交互的候选；  
  - 自我事件自动排除，不对自身行为反应；  
  - “idle”“waiting” 状态被过滤，避免过度反应；  
  - 聊天冷却 800 tick，并在每个 tick 上递减，防止两人短时间内反复对话。

- 防御性编程与回退机制：  
  - emoji 生成失败时统一回退为 🙂；  
  - 对象选择失败时使用 `"<random>"` 占位；  
  - 日程总长不足时自动追加睡眠补齐；  
  - 多处 `if not ...` / 状态检查防止空值、未初始化状态导致崩溃。

这些细节在初版文档中确实大多没有系统性点明，你朋友的反馈在这一点上是合理的：  
如果只按“架构图”来复现，而忽略这些数值与边界条件，得到的系统行为会明显偏离 Smallville 原作。

---

## 10. 作为 Dify / 其他平台实现的参考抽象

从一个抽象“工作流”视角，可以将 Plan 模块拆解为以下几个可组合组件（便于在 Dify、Orchestration Engine 等平台上实现）：

1. **长期计划生成节点（Long-term Planner）**  
   - 输入：Persona 的 identity stable set、lifestyle、当前日期。  
   - 输出：起床时间、日计划条目列表（`daily_req`）、小时级日程（`f_daily_schedule_hourly_org`）、分钟级日程（`f_daily_schedule`）。

2. **身份与目标更新节点（Identity Refiner）**  
   - 输入：最近的事件与思想记忆（由检索模块提供）、上一日状态 `currently`。  
   - 输出：新的 `currently` 描述、新的 `daily_plan_req` 文本、以及必要的计划型记忆写入。

3. **短期任务调度与分解节点（Short-term Scheduler & Decomposer）**  
   - 输入：完整日程（`f_daily_schedule`）、当前时间、Maze 中的位置信息。  
   - 输出：当前 Action（描述、时长），以及动作发生地点地址（world/sector/arena/object）。

4. **事件反应节点（Event Reaction Planner）**  
   - 输入：当前感知事件 + 检索结果（`retrieved`）、所有 Persona 的当前行为状态。  
   - 输出：是否发起聊天、是否等待、是否插入新行为，以及更新后的当前 Action。

5. **执行节点（Executor）**  
   - 输入：当前 Action 的地址、时长、描述、emoji，以及 Maze 拓扑结构。  
   - 输出：下一步移动位置、前端显示的表情与文本说明。

在 Dify 或类似编排平台中，可以把这些组件映射为一条链式或带条件分支的工作流，并且：

- 显式配置所有关键阈值（5、60、800、1440、30 天等），不依赖 LLM 的“常识”；  
- 为每个节点设计完备的错误回退路径（默认 emoji、默认对象、时间补齐等）；  
- 将 Prompt 模板管理为可版本化资源，配合变量替换机制使用；  
- 在数据模型中保留“计划记忆有效期”“计划重要性”等语义字段。

本文件前几节主要聚焦于模块职责和数据流动，后几节则补齐了你朋友指出的那些“实现层面的关键细节”。二者结合，才能为在 Dify 等平台上高保真复现 Smallville 的规划模块提供完整蓝图。

