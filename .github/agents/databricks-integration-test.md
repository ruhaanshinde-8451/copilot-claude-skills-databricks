---
name: databricks-integration-test
description: "Validate that correct parameters were passed to each task in a Databricks job run. Use after a successful bundle run to confirm task parameters match expected values declared in databricks.yml."
argument-hint: "run_id, job key, target (default dev), profile (default target)"
user-invocable: false
---

# Databricks Integration Test

Validate that a completed Databricks job run received the correct parameters per task.

## When To Use
- Called by databricks-pipeline agent after a successful run.
- When parameter validation is needed for any job defined in this repo's `databricks.yml`.

## Inputs
- `run_id`: required — from the pipeline run summary
- `job`: required — job key that was run
- `target`: default `dev`
- `profile`: default same as `target`

## Deriving Expected Parameters

Do NOT use hardcoded parameter lists. Read `databricks.yml` to discover the expected `base_parameters` for each task in the specified job:

```bash
cat databricks.yml
```

For the job matching `<job>`, find each entry under `tasks:`. For every task, the keys declared in `notebook_task.base_parameters` (or `python_wheel_task.parameters`, `spark_python_task.parameters`) are the **expected parameter keys** for that task.

Example — for a task like:
```yaml
tasks:
  - task_key: ingest
    notebook_task:
      base_parameters:
        env: ${bundle.environment}
        working_directory: ${var.working_directory}
```
The expected parameter keys for `ingest` are: `env`, `working_directory`.

## Procedure
1. Read `databricks.yml` and extract expected `base_parameters` keys for each task in the specified job.
2. Fetch run metadata:
```bash
   databricks jobs get-run <run_id> --profile <profile> -o json
```
3. Extract the actual `base_parameters` keys for each task from the run metadata.
4. Compare actual vs expected: flag any missing or extra parameter keys.

## Output Contract
- `Run ID:`
- `Job:`
- `Parameter Validation:`
  - Per task: `task_key` — `passed` or `failed`
  - `Missing:` list or `none`
  - `Extra:` list or `none`
  - `Incorrect Values:` list or `none`
- `Overall:` `passed` or `failed`
- `Next Action:` one concrete step if validation failed

## Guardrails
- Do not expose secrets or credential material.
- Do not invent parameter values — only compare against what was actually passed.
- If run metadata is incomplete, report uncertainty explicitly.