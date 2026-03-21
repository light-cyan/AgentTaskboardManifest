# 生成指南 (Generation Guidelines)
本指南设定了构建工作流拓扑、定义 I/O 契约及编写语义化动作时必须遵循的结构化标准与设计原则。

## 深度层级与职责分离 (Hierarchical Depth and Separation of Duties)
生成的工作流拓扑必须呈现为**树状分层结构 (Tree-structured Orchestration)**，严禁构建缺乏结构深度的扁平化任务流。
* **职责垂直划分 (Vertical Separation of Concerns)：** 节点的层级高度必须与其职责的抽象程度严格对应。越靠近工作流顶层（如 `!entry.task.yaml`），其核心职责越偏向于宏观规划、状态流转、异常捕获与路由分发；越靠近叶子节点（深层 `*.task.yaml`），其核心职责越偏向于具体的环境交互、数据处理与原子级执行。
* **禁止冗余嵌套 (No Redundant Nesting)：** 仅当某个逻辑块包含多个步骤、复杂的内部状态流转或具备高复用价值时，才允许将其剥离为独立的子任务。若某项功能可通过当前任务内单一步骤 (Step) 的 `action` 完成（如执行一条 Shell 命令或单次 API 请求），则严禁为其创建独立的 `.task.yaml` 文件，以避免过度设计导致的调用栈膨胀。

## 按需声明与作用域隔离 (On-Demand Declaration and Scope Isolation)
在设计工作流的 I/O 契约时，必须严格控制参数的可见性边界与生命周期，确保每个任务节点仅声明和维护其执行所必需的最小上下文。
* **参数下沉与封装 (Parameter Subordination and Encapsulation)：** 遵循最小知识原则 (Principle of Least Knowledge)。父任务的 `inputs` 和 `outputs` 仅允许声明维持当前层级路由、条件判定和上下游衔接所必需的**核心流转参数**。
* **严禁顶层穿透 (Prohibit Top-Level Penetration)：** 严禁在顶层或高层任务的接口契约中，提前穷举并声明深层子任务所需的底层执行参数。深层依赖的细枝末节参数，必须下沉至实际需要消费该参数的底层节点中。若底层节点在初始化时无法获取该参数，应将其描述为 "unknown"，并在该层级的局部执行流中通过 `action` 动态获取。

## 语义化动作描述
action 和 checks 的 guard 字段的编写应偏向描述“意图”与“验收标准”，而非受限于特定编程语言的死板语法。**在规划具体动作时，必须应优先调度 Agent Skills，绝不可自行编码底层脚本以执行操作，或自行生成本应由专用工具生成的结构化数据文件。**信任下游 Agent 在运行时能够将自然语言指令准确转换为实际的环境操作。

## 处理不确定性 (Managing Unknowns)
在生成阶段，严禁基于猜想进行任何形式的静态假设或捏造输出。面对不确定性，必须将未知状态推迟至运行态 (Runtime) 动态解决：
* **参数与数据缺失 (Parameter & Data Absence)：** 
    - 若遭遇缺乏环境先验信息或用户输入不足的变量，严禁编造默认值。
    - 在 I/O 契约的自然语言说明中将其**描述为 "unknown"**。
    - 在关联的 `<step_object>` 中通过 `action` 设计主动收集机制（如：`action: "Inquire user for the required credentials..."` 或 `action: "Run environment probe script to determine..."`）。
* **执行逻辑模糊 (Execution Logic Ambiguity)：** 
    - 若对实现某个目标的具体工具链、命令语法或底层操作流转存在模糊认知，严禁生搬硬套或凭空伪造具体的执行步骤。
    - 必须将该环节降级为“探索性节点”，在 `action` 中明确表达未知，并依赖运行态的外部检索或人工介入来打通逻辑（如：`action: "Search the official documentation to find the correct command for..."` 或 `action: "Inquire user about the preferred method/tool to process this data."`）。

## 人机协同与断点设计 (Human-in-the-Loop & Breakpoint Design)
在规划工作流时，必须将人类视为执行流中的关键审核节点。Agent 应在以下高价值或高风险场景中，主动设计阻塞式的审核断点（如通过 `action: "Request user to review the current work..."` 发起问询并等待确认），严禁全程静默执行：
* **高危与不可逆操作前 (Before High-Risk Actions)：** 在执行诸如批量删除文件、覆写核心数据、提交占用大量资源的计算任务或调用计费 API 等关键动作前，必须插入请求用户授权与确认的 `<step_object>`。
* **阶段性产物验收 (Milestone Validation)：** 在工作流完成复杂的逻辑推导、代码生成或数据转换后，进入下游消费环节前，应设计断点向用户展示中间件或生成的产物，请求人类审查其准确性后再放行。
* **策略分发与选择 (Strategic Branching)：** 当任务推进面临多个并行的可行策略，且无法通过预设规则自动抉择时，应在 `action` 中列出选项，将路由走向的决策权交由用户（如：`"Inquire user to select the preferred processing method: [Option A, Option B]"`）。

## 健壮的路由机制 (Robust Routing)
为每个关键 `<step_object>` 设计完备的 `when` 分支。必须至少考虑以下三种状态并生成相应的路由响应：
1. **预期成功** (`goto` 下一步或 `SUCCESS`)。
2. **可恢复异常或人为干预**（基于运行报错或**用户在审核断点提出的修改意见**，触发 `RETRY` 或转移到纠错 `<step_id>`）。
3. **不可恢复异常**（向上冒泡 `raise` 或触发全局 `FAIL`）。

## Case Studies


