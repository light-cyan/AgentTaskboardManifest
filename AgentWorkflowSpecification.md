# Agent Workflow Specification (AgentTaskboardManifest)

**AgentTaskboardManifest** 旨在定义一套让 Agent 学习、执行和自我总结的标准工作流语言。

* **String-heavy（语义化优于结构化）：** 尽可能使用 `<string>` 自然语言描述动作和约束，将解析工作交给 Agent 的大脑，而非硬编码的规则引擎。
* **Unknown Tolerance（未知容忍）：** 任何在设计时无法确定的字段，显式填入 `<unknowns>`。这是一种授权机制，提示 Agent 在运行时动态推断或向用户发起询问。
* **Conciseness（简洁至上）：** AI 按照标准生成或总结的工作流方案必须是极度简洁的，去除一切不必要的嵌套和冗余字段，直击任务核心。

## Architecture Overview

工作流作为一个独立目录进行组织，天然支持**懒加载 (Lazy Load)** 机制。Agent 启动时只需读取 `workflow.yaml` 了解全局边界，并加载 `entry.task.yaml` 作为起跑点。其他的 `*.task.yaml` 只有在被具体步骤引用时，才会被动态加载进入上下文。

**标准目录结构：**

```text
<goal_task_name>/
├── workflow.yaml       # 全局工作流描述与边界设定 (全局上下文)
├── entry.task.yaml     # 工作流执行主入口 (起点)
└── *.task.yaml         # 可复用的子任务节点 (动态加载)

```

**相对引用约定：**
任务间的相互调用必须使用相对路径（如 `./xxx_xx.task.yaml`）。

## Schema Definition

摒弃了繁琐的类型字段，采用**“键名作为锚点，字符串包揽约束”**的极简映射规则。

### `workflow.yaml`

此文件为 Agent 建立全局上下文（Global Context），让它知道“我在做什么”、“我的边界在哪里”。

```yaml
id: "<goal_task_name>"
domain: "<string>"             # 所属领域（如：Computational Chemistry, Data Processing）
applicable_if:                 # 适用场景列表
  - "<string>"
not_applicable_if:             # 不适用场景列表（命中此项应拒绝执行）
  - "<string>"
non_goals:                     # 总体边界列表（明确不要做的事情，防止思维发散）
  - "<string>"
prerequisites:                 # 全局前置条件列表
  - "<string>"
success_criteria:              # 成功标准列表
  - "<string>"

global_conventions:            # 全局约定(Key 为变量名，Value 为自然语言约束)
  <convention_key>: "<string>" # 举例: workspace_dir: "必填。当前工作流的根目录，绝对路径。"
  <convention_key>: "<string>" # 举例: units: "时间用秒，距离用 Angstrom"

assumptions:                   # 潜在假设（如：假设网络通畅、假设用户已登录 VPN、假设用户提供的数据清洗过）
  - "<string>"

```

### `*.task.yaml`

这是执行的最小功能单元。定义了完成特定目标所需的输入、输出、环境要求以及具体执行步骤。
**父任务的 inputs/outputs 仅定义本层级流转所需的核心参数。**严禁在父任务中提前声明其所有子任务所需的细枝末节参数，子任务所需的专属输入应在执行到该 <step> 时，由 Agent 自主通过上下文传递或触发问询。

```yaml
id: "<string>"                 # 任务唯一标识，如 "submit_slurm_job"
purpose: "<string>"            # 本任务的目标
inputs:                        # 任务输入列表 (Key 为变量名，Value 为自然语言约束)
  # **声明在此处的输入代表整个 Task 级别的 I/O 契约，但无需在任务启动时全部就绪。**
  # 变量的来源可以是：
  #   1. Context: 由上级调用任务隐式传入。
  #   2. Dynamic: 初始态为 <unknowns>，但在内部执行某个 <step> 时通过工具动态生成。
  #   3. Human-in-the-loop: 初始态为 <unknowns>，在特定的 <step> 中主动询问用户获取。
  <param_key>: "<string>"      # 举例: gjf_file: "必需。待处理的 Gaussian 输入文件，默认来自上游上下文，.gjf 格式。"
  <param_key>: "<string>"      # 举例: password: "可选。连接目标服务器的密码。初始为 unknowns，如果公钥验证失败，需在后续 step 中向用户索取。"
outputs:                       # 任务输出列表 (Key 为变量名，Value 为自然语言约束)
  <output_key>: "<string>"     # 举例: job_id: "成功提交到 Slurm 后返回的数字 ID 字符串。"
prerequisites:                 # 执行本 task 前的环境与资源验证列表
  - spec: "<string>"           # 包含版本约束的描述
    how_to_check: "<string>"   # 验证方法（如 "Run `sinfo` to check cluster status"）
steps:                         # 执行步骤列表（按序执行）
  - ...

```

### `<step>`

框架中最核心、最动态的部分。包含行为动作、产物预期、自我校验和路由跳转。

```yaml
- id: "<step_id>"              # 步骤标识，方便路由跳转
  action: "<string>"           # 核心动作。直接用自然语言描述 Agent 需执行的行为。
  # 举例：
  # - "Execute sub-task `./prepare_data.task.yaml`"
  # - "Run shell command `pytest tests/`"
  # - "Search the web for the latest Python documentation"
  # - "Inquire user: 'Do you want to overwrite?' (options: [yes, no])"
  # - "Inquire user: 'What substance do you want to inquire about?'"

  notes:
    - "<string>"               # 注意事项列表（如 "该命令可能会卡死，请设置 timeout 60s"）
  produces:                    # 步骤产生的中间件/制品列表 (Key 为变量名，Value 为自然语言约束)
    <artifact_key>: "<string>" # 举例: temp_log: "运行命令产生的 stdout 文本，用于正则提取。"
  checks:                      # 步骤执行后的自我检查机制列表
    <check_key>:  
      method: "<string>"       # 检查手段（如 "读取 ./slurm-*.out 的最后五行"）
      guard: "<string>"        # 可判断的规则表达式或自然语言条件（如 "必须包含 'Normal termination'"）

  # --- 路由与状态控制 (默认进入顺序排列的下一步) ---
  when:                        # 按顺序匹配，命中即停 (Short-circuit evaluation)
    - condition: "<string>"    # 触发条件（来自 action 反馈或 checks 结果）
      evidence: "<string>"     # 判定的证据来源（如 "日志中出现 OOM"）
      response: "<response>"   # 响应动作 (SUCCESS | FAIL | RETRY | goto(id) | raise(状态字符串))，具体见下方具体状态定义
      message: "<string>"      # 附带信息（如具体的修复建议、改动提示）
```

#### State Machine & Routing

采用**按顺序匹配，命中即停 (Short-circuit evaluation)** 的动态路由机制。在 `<step>` 的 `when` 模块中，`<response>` 仅包含以下 5 种核心指令，分为“生命周期”与“流程跳转”两类：

**生命周期状态 (Lifecycle States)**

* `SUCCESS`: 提前或正常达到预期目标，结束当前任务并返回上级调用者。
* `FAIL`: 遭遇不可逆转的失败，立即强制终止整个工作流，并向用户报错。
* `RETRY`: 原地重新执行当前 Task 或 Step（必须结合 `message` 提供重试策略或参数修改建议）。

**流程跳转与异常冒泡 (Flow Control & Bubbling)**

* `goto(<step_id>)`: 打破默认的顺序执行流，直接跳转至当前 Task 内指定的 `<step_id>`。
* `raise(<status_string>)`: 将当前层级无法处理的自定义状态/异常，抛给父 Task 或用户处理。
* *(常用范例)*：`raise(permission required)` (向需凭证/授权)、`raise(unsupported environment)` (环境不满足)。

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

```

## Case Studies

以下展示一个自动化计算化学工作流的片段。该工作流需要 Agent 检索目标分子的信息，生成 Gaussian 作业文件（`.gjf`），并提交到 HPC 的 Slurm 队列中。

### 示例 1: `workflow.yaml`

```yaml
id: "automated_gaussian_optimization"
domain: "Computational Chemistry"
applicable_if:
  - "User wants to optimize a molecular structure using Gaussian on a Slurm cluster."
success_criteria:
  - "The Gaussian job terminates normally and the optimized coordinates are extracted."
global_conventions:
  naming: "所有生成的文件以 `<species_name>_opt` 为前缀。"

```

### 示例 2: `entry.task.yaml`

```yaml
id: "main_execution_flow"
purpose: "Orchestrate data retrieval, file assembly, and job submission."
inputs:
  species: "必需。目标分子的名称或 SMILES 字符串。"
  slurm_partition: "选填。目标计算节点队列，默认为 'cpu'。"
steps:
  - id: "retrieve_molecule_data"
    action: "Call sub-task `./search_species.task.yaml` with input species."
    produces:
      xyz_coords: "分子的初始 3D 坐标数据。"
    when:
      - condition: "Species not found in any database"
        evidence: "Sub-task raised 'not_found' exception"
        response: "FAIL"
        message: "无法在数据库中找到该分子，工作流终止。"

  - id: "assemble_input_file"
    action: "Call sub-task `./assemble_gjf.task.yaml` passing xyz_coords."
    produces:
      gjf_file: "组装完成的 Gaussian .gjf 文件路径。"

  - id: "submit_to_cluster"
    action: "Run shell command `sbatch run_gaussian.sh <gjf_file>`"
    checks:
      Verify whether the job has been successfully enqueued:  ""
        method: "Run `squeue -u $USER`"
        guard: "The output contains the submitted job name"
    when:
      - condition: "Slurm rejects the submission due to partition limits"
        evidence: "stderr contains 'Requested node configuration is not available'"
        response: "goto(ask_user_partition)"
      - condition: "Job submitted successfully"
        evidence: "stdout contains 'Submitted batch job'"
        response: "SUCCESS"
        message: "Job is in queue, task complete."

  - id: "ask_user_partition"
    action: "Inquire user: 'The default Slurm partition is full or unavailable. Please provide an alternative partition name, or type cancel.' (options: open_text)"
    when:
      - condition: "User means 'cancel'"
        response: "FAIL"
        message: "用户手动取消了作业提交。"
      - condition: "User provides a new partition name"
        response: "goto(submit_to_cluster)"
        message: "Update the slurm_partition parameter and retry submission."

```
