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
执行的最小功能单元，声明 I/O 契约、环境要求、及执行序。

```yaml
id: "<string>"                 # 任务唯一标识，如 "submit_slurm_job"
purpose: "<string>"            # 本任务的目标
prerequisites:                 # 执行本 task 前的环境与资源验证列表
  - spec: "<string>"           # 包含版本约束的描述
    how_to_check: "<string>"   # 验证方法（如 "Run `sinfo` to check cluster status"）

# --- 此处声明整个 Task 的 I/O 契约，用于**对后续步骤中将会用到的各项变量进行集中定义与规范说明**。
inputs:                        # 任务输入列表 (Key 为变量名，Value 为自然语言约束)
  # 变量的来源可以是：
  #   1. Context: 由上级调用任务隐式传入。
  #   2. Dynamic: 初始态为 <unknowns>，但在内部执行某个 <step> 时通过工具动态生成。
  #   3. Human-in-the-loop: 初始态为 <unknowns>，在特定的 <step> 中主动询问用户获取。
  <param_key>: "<string>"      # 举例: gjf_file: "必需。待处理的 Gaussian 输入文件，默认来自上游上下文，.gjf 格式。"
  <param_key>: "<string>"      # 举例: password: "可选。连接目标服务器的密码。初始为 unknowns，如果公钥验证失败，需在后续 step 中向用户索取。"
outputs:                       # 任务输出列表 (Key 为变量名，Value 为自然语言约束)
  <output_key>: "<string>"     # 举例: job_id: "成功提交到 Slurm 后返回的数字 ID 字符串。"

steps:                         # 执行指令序列（默认按序执行）
  - <step_object>
```

### `<step_object>`
任务序列中的原子操作单元。包含行为动作、产物预期、自我校验和路由跳转。

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

