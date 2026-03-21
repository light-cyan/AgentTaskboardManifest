# 执行指南 (Execution Guidelines)

## 懒加载策略 (Lazy Loading Strategy)
* **初始化：** 启动时，Agent 必须且仅读取 `-workflow.yaml` 加载全局上下文，并读取 `!entry.task.yaml` 建立主执行栈。
* **按需加载：** 仅当执行流运行至包含引用子任务（如 `action: "Execute sub-task ./xxx.task.yaml"`）的 `<step_object>` 时，Agent 才被允许将目标 YAML 文件动态加载至上下文。

## 执行流转与状态转移逻辑 (Execution Flow & State Transition Logic)
* **Prerequisite Validation:** 加载 `*.task.yaml` 后，必须优先验证 `prerequisites` 中的 `<dependency>`。任意检查失败触发异常并中断流程，严禁进入 `steps` 核心逻辑。
* **顺序执行与短路路由：** 前置校验通过后，按序推进 `steps` 列表。对于每个 `<step_object>` 中的 `when` 数组，采用**短路求值 (Short-circuit evaluation)**：
    * 自上而下逐条评估 `condition`，一旦发现匹配当前 `evidence` 的条件，立即触发**状态转移**（执行对应的 `response` 与 `message`），并忽略该数组中的后续条件。
    * 若所有定义的条件均未命中，默认流转行为是顺延进入当前列表外定义的下一个 `<step_object>`。

## 变量传递与作用域 (Variables Passing & Scope)
* **上下文继承：** 子任务被调用时，自动继承全局上下文 (`global_conventions`) 和父任务的 `inputs`。
* **Context Consolidation：** 当通过 `action` 动态解析了被描述为"unknown"的参数或生成了新的 `<artifact>` 时，Agent 应当在工作记忆中对其进行显式的状态梳理与记录。将获取到的确切值、物理路径或实体特征同步至当前执行域的上下文中，确保下游的 `checks` 自查 与 `when` 状态转移具备准确的数据基准。

## 执行流转与状态评估 (Execution Flow & State Evaluation)
* **前置依赖拦截 (Prerequisite Interception)：** 进入任何 `*.task.yaml` 节点后，Agent 必须优先验证 `prerequisites`。若收集到的证据表明条件未满足，必须立即挂起或抛出异常，严禁盲目进入 `steps` 核心序列。
* **基于证据的状态路由 (Evidence-Grounded Routing)：** `<step_object>` 的流转强依赖于对实际运行反馈的严谨核验。步骤执行后，Agent 必须审视真实产生的上下文（如 stdout 日志、API 响应、用户输入或自检结果）。在依据 `when` 数组路由时，`condition`（触发条件）与 `evidence`（判定依据）是协同工作的：Agent 需将 `evidence` 中声明的具象观测特征作为检验基准，在实际运行结果中提取事实支撑。仅当在实际上下文中检索到符合 `evidence` 的确凿事实，从而证明 `condition` 成立时，该路由分支才被视为命中，进而触发相应的状态机指令。
匹配策略补充：
    * `when` 数组的评估遵循*Short-circuit evaluation*规则。自上而下逐条比对，命中首个拥有事实支撑的条件即刻触发状态转移，并严格忽略该数组中的后续所有分支。
    * 若当前 `<step_object>` 实际运行反馈未能提供 `when` 列表中任何 `evidence` 所要求的确凿证据(，或当前步骤未显式声明 `when` 数组)，默认继续执行列表中的下一个 `<step_object>`。

## 异常冒泡与人工干预 (Exception Bubbling & Human Fallback)
当 `<step_object>` 触发 `raise(<status_string>)` 时，遵循以下状态流转与上报规则：
1. **自动流中断：** 当前子任务的自动化探索立即停止，不再向后进行。
2. **状态抛递与逐级捕获：** 状态信息向上抛递给调用该子任务的父级 `*.task.yaml` 所在的 `<step_object>`。父级任务的 `when` 列表应当尝试匹配此 `status_string` 以执行降级或重试逻辑。若未被捕获，异常将沿调用栈继续向上冒泡。
3. **Human-in-the-loop Fallback：** 若异常冒泡至顶级 `!entry.task.yaml` 仍未命中任何预设路由，或直接触发了全局 `FAIL` 状态，Agent 必须严格执行以下终端动作：
   * 停止所有自动化尝试，防止错误级联与资源浪费。
   * 立即向用户汇报问题，输出格式化的异常报告（包含抛出异常的节点 ID、`status_string` 以及具体的 `message` 证据），以寻求协助。
   * 等待用户的明确指令（如补充缺失凭证、修改环境配置、指示重试或手动终止）后再决定后续行为。

