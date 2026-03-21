# 语法与语义规范 (Syntax and Semantics)
本规范定义了 `AgentTaskboardManifest` 的物理文件组织形式及内部数据结构。映射规则采用“键名作为标识锚点，字符串作为语义约束”的模式。

## 目录结构与作用域
工作流以独立目录的形式进行封闭组织，其标准拓扑结构如下：

* `-workflow.yaml`: **全局上下文 (Global Context)**。定义工作流的宏观边界、全局约定与最终成功标准。
* `!entry.task.yaml`: **主入口节点 (Entry Node)**。工作流初始化的首个加载对象。
* `*.task.yaml`: **子任务节点 (Sub-task Nodes)**。可复用的具体执行单元，支持相对路径引用（如 `./xyz.task.yaml`）。

## Schema 定义

### `-workflow.yaml`
用于确立全局执行边界，建立全局上下文（Global Context）。

```yaml
workflow_id: "<goal_task_name>"
domain: "<business_domain>"    # 所属领域（如：Computational Chemistry, Data Processing）
applicable_if:                 # 适用场景
  - "<scenario_description>"
not_applicable_if:             # 不适用场景（命中此项应拒绝执行）
  - "<exclusion_condition>"
non_goals:                     # 总体边界（明确不要做的事情，防止思维发散）
  - "<out_of_scope_behavior>"
success_criteria:              # 成功标准
  - "<observable_outcome>"
  
prerequisites:                 # 全局前置依赖条件声明
  <dependency>: "<requirement_description_and_verification_method>"
  # 举例: `python_env: "Require Python 3.10 or higher. Execute 'python --version' to verify compatibility."`

global_conventions:            # 全局约定
  <convention>: "<rule_description_and_constraint>" 
  # 举例: `workspace_dir: "必填。当前工作流的根目录，绝对路径。"`
  # 举例: `units: "时间用秒，距离用 Angstrom"`
assumptions:                   # 潜在假设
  - "<environmental_assumption>"
  # 举例: `"网络通畅"`
  # 举例: `"用户已登录VPN"`
  # 举例: `"用户提供的数据清洗过"`
```

### `*.task.yaml`
执行的最小功能单元，声明 I/O 契约、环境要求、及执行序。

```yaml
task_id: "<task_identifier>"                 # 任务唯一标识，如 "submit_slurm_job"
purpose: "<task_objective_descriptionx>"            # 本任务的目标

# --- 此处声明整个 Task 的 I/O 契约，用于**对后续步骤中将会用到的各项变量进行集中定义与规范说明**。
inputs:                        # Input Definition
  # 变量的来源可以是：
  # - Context: 由上级调用任务隐式传入。
  # - Dynamic: 初始未被描述为 "unknown"，但在内部执行某个 <step_object> 时通过工具动态生成。
  # - Human-in-the-loop: 初始未被描述为 "unknown"，在特定的 <step_object> 中主动询问用户获取。
  <variable>: "<source_and_semantic_constraint>"    
  # 举例: `gjf_file: "必需。待处理的 Gaussian 输入文件，来自上游上下文，.gjf 格式。"`
  # 举例: `password: "可选。连接目标服务器的密码。初始为 unknown，如果公钥验证失败，需在后续 step 中向用户索取。"`
outputs:                       # Output Definition
  <variable>: "<output_description_and_constraint>"     
  # 举例: `job_id: "成功提交到 Slurm 后返回的数字 ID 字符串。"`

prerequisites:                 # 本task的前置依赖条件声明
  <dependency>: "<requirement_description_and_verification_method>"
  # 举例: `slurm_access: "Must have partition submission rights. Run `sinfo` to check cluster status."`

steps:                         # 执行指令序列（默认按序执行）
  - <step_object>
```

### `<step_object>`
任务序列中的原子操作单元。包含行为动作、产物预期、自我校验和路由跳转。

```yaml
- id: "<step_id>"              # 步骤标识，方便路由跳转
  action: "<executable_action_description>"           # 核心动作。直接用自然语言描述 Agent 需执行的行为。
  # 举例：
  # - `"Execute sub-task `./prepare_data.task.yaml`"`
  # - `"Run shell command `pytest tests/`"`
  # - `"Request user to review the intermediate results and confirm before proceeding."`
  # - `"Search the web for the latest Python documentation"`
  # - `"Inquire user: 'Do you want to overwrite?' (options: [yes, no])"`
  # - `"Inquire user: 'What substance do you want to inquire about?'"`

  notes:
    - "<execution_note_or_warning>"               # 注意事项（如 "该命令可能会卡死，请设置 timeout 60s"）
  produces:                    # 步骤产生的中间件/制品说明
    <artifact>: "<artifact_description_and_use>" 
    # 举例: `temp_log: "运行命令产生的 stdout 文本，用于正则提取。"`
  checks:                      # 步骤执行后的自我检查清单
    <assertion>:  
      method: "<data_collection_method>"       # 检查手段（如 "读取 ./slurm-*.out 的最后五行"）
      guard: "<validation_rule_or_condition>"        # 可判断的规则表达式或自然语言条件（如 "必须包含 'Normal termination'"）

  # --- 路由与状态控制 (默认进入顺序排列的下一步) ---
  when:                        # 按顺序匹配，命中即停 (Short-circuit evaluation)
    - condition: "<trigger_condition_description>"    # 触发条件（来自 action 的运行反馈或 checks 的自检结果）
      evidence: "<observable_evidence_source>"     # 判定的证据来源（如 "日志中出现 OOM"，或 "用户在检查断点表达不满并给出修改建议"）
      response: "<response>"   # 响应动作，具体见下方具体状态定义
      message: "<state_transition_message_or_instruction>"      # 状态转移的附带指令或上下文（如抛出异常时的排查建议，或指导下一步任务的调整方向，例如`"解析并执行用户在此断点补充的修改建议"`）
```

### State Machine & Routing
`<response>` 字段仅接受以下五种指令：

**生命周期指令：**
* `SUCCESS`: 确认当前层级任务完成，返回值并向调用栈上层返回。
* `FAIL`: 触发不可逆失败，中断并销毁当前工作流实例。
* `RETRY`: 保持当前执行栈，重新执行当前 Task 或 Step。

**控制流与异常指令：**
* `goto(<step_id>)`: 栈内跳转，转移执行流至当前文件内的指定步骤。
* `raise(<status_string>)`: 异常冒泡，将当前处理栈无法消解的状态挂起，并交由父任务或人类介入处理。

具体的例子:
```yaml
  when:
    # --- 处理环境不满足的情况 ---
    - condition: "CUDA is not available"
      evidence: "torch.cuda.is_available() returned False"
      response: "raise(unsupported environment)"
      message: "This workflow heavily relies on GPU acceleration. Please inform the user to check the Nvidia drivers or switch to a GPU node on Slurm before retrying."

    # --- 自动重试 ---
    - condition: "Network timeout while fetching data"
      evidence: "TimeoutError in requests.get"
      response: "RETRY"
      message: "Wait for 5 seconds and increase the timeout parameter to 60s."

    # --- 提前成功 ---
    - condition: "The target file already exists and is valid"
      evidence: "File ./output.log size > 0 and contains 'Finished'"
      response: "SUCCESS"
      message: "Target already achieved from a previous run, skipping computation."

