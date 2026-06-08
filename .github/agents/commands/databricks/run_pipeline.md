---
description: Validate, deploy, run, and summarize a Databricks bundle job
argument-hint: "--job <job_key> [--target <target>] [--profile <profile>] [--skip-deploy] [--no-strict] [--no-wait] [--integration-test]"
allowed-tools: [execute, read, search]
---

# /databricks:run-pipeline

Run end-to-end Databricks bundle automation for any DAB repo.

## Available Jobs
Discover available jobs by reading `databricks.yml`:
```bash
cat databricks.yml
```
List all keys under `resources.jobs` and `resources.pipelines`. If `--job` is missing, present these keys and stop — ask the user which to run.

## Goal
1. Validate bundle configuration
2. Deploy bundle for the selected target/profile
3. Run the selected Databricks job
4. Fetch run metadata and run output
5. If `--integration-test` is passed, validate parameters per task
6. Return a concise status summary with failure details

## Required Defaults
- If target is not provided, use `dev`.
- If profile is not provided, use the same value as target.
- Require an explicit job key before execution.
- Use `scripts/databricks/run_pipeline.sh` if it exists at the repo root; otherwise invoke `databricks bundle` CLI commands directly.

## Usage
```bash
/databricks:run-pipeline --job <job_key> [--target <target>] [--profile <profile>] [--skip-deploy] [--no-strict] [--no-wait] [--integration-test]
```

## Workflow
1. Parse arguments.
2. If `--job` is missing, read `databricks.yml`, list available job/pipeline keys, and ask the user to choose.
3. Execute:
```bash
   # If scripts/databricks/run_pipeline.sh exists:
   scripts/databricks/run_pipeline.sh --job <job_key> --target <target> --profile <profile> [...flags]
   # Otherwise:
   databricks bundle validate -t <target> --profile <profile>
   databricks bundle deploy -t <target> --profile <profile>
   databricks bundle run <job_key> -t <target> --profile <profile>
```
4. If `--integration-test` is passed, use databricks-integration-test to validate parameters.
5. Surface output exactly as returned.

## Safety Rules
- Never run destructive git commands.
- Never print or request secrets.
- If Databricks auth fails, stop and report the exact command that failed.
- Never run against `prd` without explicit user confirmation.

## Output Format
- `Target:` value
- `Profile:` value
- `Job:` value
- `Validation:` passed/failed
- `Deploy:` passed/failed/skipped
- `Run ID:` value or `not found`
- `Run State:` lifecycle/result state if available
- `Failure Class:` value from script summary
- `Notebook Output:` short excerpt when available
- `Parameter Validation:` passed/failed/skipped
- `Next Action:` one concrete next step when something fails