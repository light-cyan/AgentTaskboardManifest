# 执行指南 (Execution Guidelines)

## 懒加载策略 (Lazy Loading Strategy)
* **初始化：** 启动时，Agent 必须且仅读取 `-workflow.yaml` 加载全局上下文，并读取 `!entry.task.yaml` 建立主执行栈。
* **按需加载：** 仅当执行流运行至包含引用子任务（如 `action: "Execute sub-task ./xxx.task.yaml"`）的 `<step>` 时，Agent 才被允许将目标 YAML 文件加载至内存上下文。

## 状态机评估逻辑 (State Evaluation Logic)
对于 `<step>` 中的 `when` 数组，采用**短路求值 (Short-circuit evaluation)**。
* Agent 必须自上而下逐条评估 `condition`。
* 一旦发现匹配当前上下文 `evidence` 的条件，立即执行其对应的 `response` 与 `message`，并**忽略该数组中后续的所有条件**。
* 如果所有定义好的条件均未匹配，默认行为是进入数组外定义的下一顺序步骤。

## 变量传递与作用域
* **上下文继承：** 子任务被调用时，自动继承全局上下文 (`global_conventions`) 和父任务显式传递的 `inputs`。
* **动态赋值：** 当遇到标记为 `<unknowns>` 的参数时，Agent 必须通过当前 `action`（如工具调用、询问人类介入）获取确切值，并在内存中覆写该变量状态，随后才可进入下一步校验 (`checks`)。

## 异常冒泡机制 (Exception Bubbling)
当 `<step>` 触发 `raise(<status_string>)` 时，遵循以下状态流转与上报规则：

1. **执行挂起：** 当前子任务的自动执行流立即挂起，不再向后计算。
2. **状态抛递：** 状态信息（包含异常枚举值及上下文栈）向上抛递给调用该子任务的父级 `*.task.yaml` 所在的 `<step>`。
3. **逐级捕获：** 父级任务的 `when` 列表中应当包含捕获此 `status_string` 的条件分支以执行降级或重试逻辑。若未被捕获，异常将沿调用栈继续向上冒泡，直至被顶级入口 `!entry.task.yaml` 捕获。
4. **Human-in-the-loop Fallback：** 若异常冒泡至最顶层仍未命中任何预设的路由策略，或直接触发了全局 `FAIL` 状态，Agent 必须严格执行以下终端动作：
   * 中止所有自动化尝试，防止错误级联。
   * 向向用户寻求协助（如提示检查特定的环境配置或提供缺失凭证），描述问题并输出格式化的异常报告（包含抛出异常的节点 ID、`status_string` 以及相关的 `message` 证据）。
   * 停止一切行动直到接收到用户的明确指令（如提供新参数、指示重试或手动确认终止）。

