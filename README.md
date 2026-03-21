# Agent Workflow Specification (AgentTaskboardManifest)

## 概述 (Overview)
`Agent Taskboard Manifest` 是一套专为自主智能体设计的模块化任务编排规范。本规范基于声明式的 YAML 结构，定义了工作流的物理文件封闭组织形式与内部数据状态。它抛弃硬编码的复杂对象，采用“键名作为标识锚点，自然语言作为语义约束”的模式，用于严格约束、指导和记录 Agent 的任务执行生命周期，并支持运行时的懒加载与动态上下文推断。

## 核心理念 (Core Principles)
* **Semantic Constraints:** 抛弃死板的代码控制流，将 I/O 契约与动作规则转化为键值对映射，键名作为标识锚点，字符串作为语义约束。规范高度信任大语言模型的自然语言解析能力，在调度与校验中偏向描述执行意图与最终验收标准。
* **Managing Unknowns:** 承认静态编排时的信息盲区。对于缺失的参数或模糊的执行逻辑，严禁基于猜想进行静态假设，必须在契约中声明为 "unknown"，引导 Agent 将其推迟至运行态，通过工具探测或触发主动发问进行动态推断。
* **Hierarchical Depth & Isolation:** 建立树状的分层拓扑结构。宏观状态流转与微观环境执行垂直解耦。遵循最小知识原则，将底层操作参数下沉至独立的子任务节点，保持严密的作用域隔离，防止上下文污染与逻辑穿透。
* **Robust Validation & Routing:** 在执行具体任务时，要求前置依赖校验与步骤执行后自检。任务流转高度依赖运行态反馈，采用基于证据的状态路由控制执行流。Agent 需在实际运行结果中提取客观事实以证明路由条件成立。
* **Human-in-the-Loop:** 将人工干预确立为状态机的标准控制组件。在规划阶段，需在高危操作、产物验收或策略分流处主动设计阻塞式的审核断点；在执行阶段，当局部无法消解的异常冒泡至顶层时，必须强制终止其自主启动的所有自动化进程。需向用户准确汇报当前上下文或异常节点，并依赖明确的人工指令来决定后续路由，以防止错误级联与资源浪费。

## 文档索引 (Documentation Index)
* **[Syntax & Semantics](./agent-taskboard-manifest/reference/syntax_and_semantics.md):** 定义工作流的标准拓扑目录结构、YAML Schema 字段约束、基于自然语言的 I/O 定义、原子步骤结构、异常处理机制及状态机控制流。
* **[Design Guidelines](./agent-taskboard-manifest/reference/design_guidelines.md):** 指导 Agent 在任务规划与总结和生成阶段，如何构建树状分层结构、实施参数下沉与封装，以及如何为每个执行步骤设计完备的条件路由分支。
* **[Execution Guidelines](./agent-taskboard-manifest/reference/execution_guidelines.md):** 规范 Agent 在运行态下的动态解析行为，包括目标文件的按需懒加载、上下文数据梳理、基于证据的状态路由，以及异常抛递至顶层后的人工干预终端动作。

