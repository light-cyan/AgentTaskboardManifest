# 整体架构与核心原则 (Overall Architecture)

工作流作为一个目录进行组织，天然支持**懒加载 (Lazy Load)**：Agent 启动时只需读取 `workflow.yaml` 了解全局，并加载 `entry.task.yaml` 作为起点。其他的 `*.task.yaml` 只有在被 step 引用时才动态加载进入上下文，这极大地节省了 Token 并降低了幻觉率。

**目录结构：**
```text
<goal_task_name>/
├── workflow.yaml       # 全局工作流描述与边界设定
├── entry.task.yaml     # 工作流执行入口
└── *.task.yaml         # 可复用的子任务节点

```

**核心设计约定：**

* **语义化优于结构化：** 大量使用 `<string>` 自然语言描述，利用 Agent 的阅读理解能力。
* **未知容忍：** 任何设计时无法确定的字段，显式填入 `<unknowns>`，提示 Agent 在运行时动态推断或询问用户。
* **相对引用：** 任务间的相互调用统一使用相对路径（如 `./data_prep.task.yaml`）。

---

# 全局描述：`workflow.yaml`

这个文件是给 Agent 建立全局上下文（Global Context）的，让它知道“我在做什么”、“我的边界在哪里”。

```yaml
id: "<goal_task_name>"
domain: "<string>"             # 所属领域（如：Computational Chemistry, Data Processing）
applicable_if:                 # 适用场景
  - "<string>"

not_applicable_if:             # 不适用场景（Agent 看到命中此项应拒绝执行）
  - "<string>"

non_goals:                     # 总体边界（明确不要做的事情，防止发散）
  - "<string>"

prerequisites:                 # 全局前置条件
  - "<string>"

success_criteria:              # 成功标准
  - "<string>"

global_inputs:                 # 全局输入约束 (Key 为变量名，Value 为自然语言约束)
  <input_key>: "<string>"      # 举例: workspace_dir: "必填。当前工作流的根目录，绝对路径。"
  <input_key>: "<string>"      # 举例: target_molecule: "选填。目标分子的 SMILES 字符串，默认为 unknowns。"

global_outputs:                # 全局输出预期
  <output_key>: "<string>"     # 举例: optimized_structure: "最终优化后的结构文件，.xyz 格式。"

global_conventions:            # 全局约定
  naming: "<string>"           # 命名规范
  units: "<string>"            # 单位约定（如：时间用秒，距离用 Angstrom）

assumptions:                   # 潜在假设（如：假设网络通畅、假设用户提供的数据清洗过）
  - "<string>"

```

---

# 任务定义：`*.task.yaml` (包含 `entry.task.yaml`)

这是执行的最小功能单元。它定义了执行此任务所需的输入、输出、环境要求以及具体步骤。

```yaml
id: "<string>"                 # 任务唯一标识，如 "download_dataset"
purpose: "<string>"            # 本任务的目标
inputs:                        # 任务输入 (Key 为参数名，Value 包含必填性、来源、格式、默认值等)
  <param_key>: "<string>"      # 举例: gjf_file: "必填。待处理的 Gaussian 输入文件，来自上游任务，.gjf 格式。"
  <param_key>: "<string>"      # 举例: slurm_partition: "选填。提交任务的 Slurm 队列名称，默认为 'cpu' 或由用户指定。"

outputs:                       # 任务输出
  <output_key>: "<string>"     # 举例: job_id: "字符串。成功提交到 Slurm 后返回的 Job ID。"

prerequisites:                 # 执行本 task 前的环境与资源验证
  - spec: "<string>"           # 包含版本约束的描述（如 "Python >= 3.10"）
    how_to_check: "<string>"   # 验证方法（如 "run `python --version`"）

steps:                         # 执行步骤列表（见下方详细定义）
  - ...

```

---

# 步骤与动态路由：`<step>`

这是框架中最核心、最动态的部分。结合了**行为执行**、**人机交互**、**自我校验**和**状态机路由**。

```yaml
- id: "<step_id>"              # 步骤标识，方便路由跳转
  action: "<string>"           # 核心动作。直接用自然语言描述你要Agent做的事。
                               # 举例：
                               # - "Execute sub-task `./prepare_data.task.yaml`"
                               # - "Run shell command `pytest tests/`"
                               # - "Search the web for the latest Pydantic documentation"
                               # - "Inquire user: 'Do you want to overwrite the existing file?' (options: [yes, no])"
                               # - "Require user manual action: 'Please configure the API key in .env file and reply DONE'"

  notes:
    - "<string>"               # 注意事项（如 "可能会耗时较长，请设置 timeout"）

  produces:                    # 步骤产生的中间件/制品
    <artifact_key>: "<string>" # 举例: temp_log: "运行命令产生的标准输出流文本，用于后续正则提取。"

  checks:                      # 步骤执行后的自我检查
    <check_key>:               # 检查项描述
      method: "<string>"       # 检查手段（如 "读取 ./log.txt"）
      guard: "<string>"        # 可判断的规则表达式或自然语言条件

  # --- 路由与状态控制 (默认进入顺序排列的下一步) ---
  when:                        # 按顺序匹配，命中即停 (Short-circuit evaluation)
    - condition: "<string>"    # 触发条件（来自 action 反馈或 checks 结果）
      evidence: "<string>"     # 判定的证据来源（如 "日志中出现 OOM"）
      response: "<response>"   # 响应动作 (SUCCESS | FAIL | RETRY | goto(id) | raise(状态字符串))，具体见下方具体状态定义
      message: "<string>"      # 附带信息（如具体的修复建议、改动提示）

    # --- 具体的例子：处理环境不满足的情况 ---
    - condition: "CUDA is not available"
      evidence: "torch.cuda.is_available() returned False"
      response: "raise(unsupported_environment)"
      message: "This workflow heavily relies on GPU acceleration. Please inform the user to check the Nvidia drivers or switch to a GPU node on Slurm before retrying."

    # --- 具体的例子：自动重试 ---
    - condition: "Network timeout while fetching data"
      evidence: "TimeoutError in requests.get"
      response: "RETRY"
      message: "Wait for 5 seconds and increase the timeout parameter to 60s."

    # --- 具体的例子：提前成功 ---
    - condition: "The target file already exists and is valid"
      evidence: "File ./output.log size > 0 and contains 'Finished'"
      response: "SUCCESS"
      message: "Target already achieved from a previous run, skipping computation."

```

## `<response>` 状态定义列表
`<response>` 仅包含以下 5 种核心指令，分为“生命周期”与“流程跳转”两类：

**1. 生命周期状态 (Lifecycle States)**

* `SUCCESS`: 提前或正常达到预期目标，结束当前任务并返回上级。
* `FAIL`: 遭遇不可逆转的失败，立即强制终止整个工作流，并向用户报错。
* `RETRY`: 原地重新执行当前 Task 或 Step（必须结合 `message` 提供重试策略或参数修改建议）。

**2. 流程跳转与异常冒泡 (Flow Control & Bubbling)**

* `goto(<step_id>)`: 打破默认的顺序执行流，直接跳转至当前 Task 内指定的 `<step_id>`。
* `raise(<status_string>)`: 将当前层级无法处理的自定义状态/异常，抛给父 Task 或用户处理。
* *(常用自定义状态举例)*：`raise(permission_required)` (需凭证/授权)、`raise(unsupported_environment)` (环境依赖缺失，需人工介入)。

