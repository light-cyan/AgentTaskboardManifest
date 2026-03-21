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
代理在评估任务路径、解析用户意图或生成任何工作流产物之前，**必须**阅读 `./reference/syntax_and_semantics.md`，该文件为权威的事实来源。

在完成这一基础加载之后，代理必须评估上下文并过渡到以下执行路径之一。在继续之前，**必须**阅读并遵循所选路径的特定参考文件：

- **路径：规划与生成**
  必需上下文：
  - `./reference/design_guidelines.md`
  - `./reference/plan_communication_guidelines.md`

- **路径：形式化与总结**
  必需上下文：
  - `./reference/design_guidelines.md`

- **路径：加载与执行**
  必需上下文：
  - `./reference/execution_guidelines.md`

