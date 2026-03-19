# 生成指南 (Generation Guidelines)


## 深度层级与职责分离 (Hierarchical Depth and Separation of Duties)
生成的工作流拓扑必须呈现为**树状分层结构 (Tree-structured Orchestration)**，严禁构建缺乏结构深度的扁平化任务流。
* **职责垂直划分 (Vertical Separation of Concerns)：** 节点的层级高度必须与其职责的抽象程度严格对应。越靠近工作流顶层（如 `!entry.task.yaml`），其核心职责越偏向于宏观规划、状态流转、异常捕获与路由分发；越靠近叶子节点（深层 `*.task.yaml`），其核心职责越偏向于具体的环境交互、数据处理与原子级执行。
* **禁止冗余嵌套 (No Redundant Nesting)：** 仅当某个逻辑块包含多个步骤、复杂的内部状态流转或具备高复用价值时，才允许将其剥离为独立的子任务。若某项功能可通过当前任务内单一步骤 (Step) 的 `action` 完成（如执行一条 Shell 命令或单次 API 请求），则严禁为其创建独立的 `.task.yaml` 文件，以避免过度设计导致的调用栈膨胀。

## 按需声明与作用域隔离 (On-Demand Declaration and Scope Isolation)
* **参数下沉与封装 (Parameter Subordination and Encapsulation)：** 遵循最小知识原则 (Principle of Least Knowledge)。父任务的 `inputs` 和 `outputs` 仅允许声明维持当前层级路由、条件判定和上下游衔接所必需的**核心流转参数**。
* **严禁顶层穿透 (Prohibit Top-Level Penetration)：** 严禁在顶层或高层任务的接口契约中，提前穷举并声明深层子任务所需的底层执行参数。深层依赖的细枝末节参数，必须下沉至实际需要消费该参数的底层节点中。若底层节点在初始化时无法获取该参数，应将其声明为 `<unknowns>`，并在该层级的局部执行流中通过 `action` 动态获取。

## 处理不确定性 (Managing Unknowns)
在生成阶段，若遭遇缺乏环境先验信息或用户输入不足的参数，遵循以下原则：
* 不要捏造或假定静态数据。
* 将该变量的值显式初始化为 `<unknowns>`。
* 在对应的 `<step>` 中，通过 `action` 设计主动收集机制（如：`action: "Inquire user for..."` 或 `action: "Run heuristic script to determine..."`）。

## 语义化动作描述
`action` 和 `guard` 字段的编写应偏向描述“意图”和“验收标准”，而非受限于特定编程语言的死板语法。应信任下游 Agent 在执行态能够将自然语言指令转换为实际的环境操作指令。

## 健壮的路由机制 (Robust Routing)
为每个关键 `<step>` 设计完备的 `when` 分支。必须至少考虑以下三种状态并生成相应的路由响应：
1.  **预期成功** (`goto` 下一步或 `SUCCESS`)。
2.  **可恢复异常**（基于条件触发 `RETRY` 或转移到纠错 `<step_id>`）。
3.  **不可恢复异常**（向上冒泡 `raise` 或触发全局 `FAIL`）。

