# 案例分析 (Case Studies)
本节展示一个设想的标准计算化学自动化工作流的部分切片，用于演示层级调度与动态路由的实际写法。

## 场景描述
目标：检索目标分子的信息，生成 Gaussian 作业文件 (`.gjf`)，并提交至 HPC 的 Slurm 队列。

## 1. 全局定义 (`-workflow.yaml`)

```yaml
id: "automated_gaussian_optimization"
domain: "Computational Chemistry"
applicable_if:
  - "User wants to optimize a molecular structure using Gaussian on a Slurm cluster."
success_criteria:
  - "The Gaussian job terminates normally and the optimized coordinates are extracted."
global_conventions:
  naming: "All generated files MUST use the prefix `<species_name>_opt`."
```

## 2. 主入口逻辑 (`!entry.task.yaml`)

```yaml
id: "main_execution_flow"
purpose: "Orchestrate data retrieval, file assembly, and job submission."
inputs:
  species: "Required. Target molecule name or SMILES string."
  slurm_partition: "Optional. Target compute node queue. Defaults to 'cpu'."
steps:
  - id: "retrieve_molecule_data"
    action: "Execute sub-task `./search_species.task.yaml` with input species."
    produces:
      xyz_coords: "Initial 3D coordinates of the molecule."
    when:
      - condition: "Species not found in any database"
        evidence: "Sub-task raised 'not_found' exception"
        response: "FAIL"
        message: "Molecule not found in databases. Workflow terminated."

  - id: "assemble_input_file"
    action: "Execute sub-task `./assemble_gjf.task.yaml` passing xyz_coords."
    produces:
      gjf_file: "Path to the assembled Gaussian .gjf file."

  - id: "submit_to_cluster"
    action: "Run shell command `sbatch run_gaussian.sh <gjf_file>`"
    checks:
      verify_enqueued:  
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
        message: "User manually cancelled the job submission."
      - condition: "User provides a new partition name"
        response: "goto(submit_to_cluster)"
        message: "Update the slurm_partition parameter and retry submission."
```

