---
name: agent-taskboard-manifest
description: >
    It is a specification for semantic workflows used by agents to plan, generate, formalize, summarize, and execute complex tasks, projects, experiments,and research efforts for agents, requiring explicit structure, lazy loading,scoped context, evidence-grounded routing, and human review at critical checkpoints.
    USE WHEN the user asks for a complex task, project, experiment, or research effort that needs to be carefully planned before execution
    USE WHEN the user provides a text-based plan and wants it to be made more detailed and formalized according to this specification.
    USE WHEN the user asks to summarize ongoing or completed work into a reusable workflow manifest.
    USE WHEN the user specifies the location of an existing agent workflow and wants it loaded and executed according to the specification.
metadata:
  author: light-cyan
  version: '0.1.0'
  repository: https://github.com/light-cyan/AgentTaskboardManifest
---

# Agent Workflow Specification (AgentTaskboardManifest)

## Execution Procedure
代理必须**强制**阅读 `./reference/syntax_and_semantics.md`，将其视为最终权威信息来源，然后才能评估任务路径、解释用户意图或生成任何工作流产物。

在建立此基础之后，代理必须确定适用的执行路径并严格执行相应的标准操作流程：

### 路径 1：规划 / 生成
*适用于从零开始协作构建新工作流的情况。*
1. **强制阅读：** `./reference/plan_communication_guidelines.md`。
2. **初步对齐与基线：** 与用户沟通，概述宏观结构。生成最小基线产出 (`-workflow.yaml` 和 `!entry.task.yaml`)，向用户报告初步架构，并**暂停执行**以等待用户确认。
3. **强制阅读：** `./reference/design_guidelines.md`（仅在用户批准基线后继续）。
4. **迭代优化：** 通过持续与用户协商，逐步扩展工作流程的详细信息和深层任务。严格遵守语法和设计约束。严禁随意编造步骤或依赖关系。 

### 路径 2：形式化 / 总结 
*适用于将文本计划转换为清单，或将已完成的工作抽象为可重用工作流程。* 
1. **必读文件：** `./reference/design_guidelines.md`。 
2. **上下文处理与抽象：** 
    * *针对基于文本的计划：* 逻辑性地扩展提供的细节。主动向用户询问任何模糊逻辑、缺失约束或边缘情况。对未经验证的参数使用 `unknown`。 
    * *针对总结已执行的工作：* 将硬编码值提取并抽象为可重用变量。将运行特定实例与通用逻辑分开，确保生成的工作流程高度可扩展，而不是一次性脚本。
3. **最终确定：** 严格遵循语法约束输出完整工作流。严禁生成不支持的操作或在未经确认的情况下假设用户意图。 

### 路径 3：加载与执行 
*适用于指示执行现有工作流的情况。* 
1. **执行前检查：** 验证目标工作流目录是否存在且可访问。读取 `-workflow.yaml` 文件，以确认其在当前环境下的适用性和前提条件。 
2. **必读文件：** `./reference/execution_guidelines.md`。 
3. **严格执行：** 动态按步骤加载任务，严格从 `!entry.task.yaml` 开始。代理必须严格遵循已定义的顺序，完全基于运行时证据评估路由条件，并遵守所有已定义的人工干预断点。严格禁止偏离或臆造定义的工作流路径。