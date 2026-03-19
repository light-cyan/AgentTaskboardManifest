---
name: agent-taskboard-manifest
description: >
  A specification to define, load, or execute semantic workflow descriptions for agents.
  USE WHEN the user mentions a complex task: search for a suitable existing workflow, and if found, ask the user for confirmation to execute it.
  USE WHEN the user requests a text-based plan or asks to summarize executed actions into a workflow: load this specification to generate it.
metadata:
  author: light-cyan
  version: '0.1.0'
  repository: https://github.com/light-cyan/AgentTaskboardManifest
---

# Agent Workflow Specification (AgentTaskboardManifest)

## 概述 (Overview)
`AgentTaskboardManifest` 是一种专为智能体 (Agent) 设计的标准工作流规范语言。该规范旨在提供一套机器可读且语义丰富的文件结构，用于约束、指导和记录 Agent 的任务执行生命周期。本规范采用分层树状结构组织任务，支持动态推断与懒加载机制。

## 核心原则 (Core Principles)
* **语义化约束 (Semantic Constraints):** 优先使用自然语言字符串描述动作与状态约束，降低硬编码规则的耦合度，依赖大语言模型的语义解析能力。
* **未知状态容忍 (Unknown Tolerance):** 允许在静态设计阶段存在未决定的参数，通过显式的 `<unknowns>` 标识，授权 Agent 在运行时进行动态推断或交互式问询。
* **层级化编排 (Hierarchical Orchestration):** 宏观调度与微观执行解耦。上层任务负责状态流转与边界定义，具体操作细节下沉至独立的子任务节点。
* **结构精简 (Structural Conciseness):** 消除不必要的嵌套与冗余逻辑层，确保工作流拓扑的最简性。

## 文档索引 (Documentation Index)
* [语法与语义规范 (Syntax & Semantics)](./reference/syntax_and_semantics.md): 定义工作流的目录结构、YAML Schema 及状态机枚举。
* [生成指南 (Generation Guidelines)](./reference/generation_guidelines.md): 指导 Agent 如何根据自然语言需求标准地**生成**或**修改**一套工作流。
* [执行指南 (Execution Guidelines)](./reference/execution_guidelines.md): 指导 Agent 在运行态下如何**解析**、**加载**与**执行**该工作流，以及如何处理异常与路由跳转。
* [案例分析 (Case Studies)](./reference/case_studies.md): 提供基于该规范的实际工程化应用示例（如计算化学工作流）。
* [工作流实例库 (Workflow Instances)](./workflows-instances/): 存放已构建完成并经过验证的标准工作流目录，供 Agent 直接加载执行或作为扩展模板参考。