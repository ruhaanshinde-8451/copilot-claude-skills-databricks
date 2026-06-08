Trigger and monitor a Databricks job or DLT pipeline run.

Job/pipeline key and target: extract from $ARGUMENTS.

Steps:
1. If no resource key given: run `databricks bundle summary -t <target>` to list available keys, then ask
2. Run the resource:
   ```
   databricks bundle run <resource_key> -t <target>
   ```
   Map any parameters from $ARGUMENTS:
   - Named params → `--python-named-params "k=v,k2=v2"`
   - Notebook widgets → `--notebook-params "k=v"`
   - DLT full refresh → `--refresh-all`
   - Specific task → `--task <task_key>`

3. Stream output — report progress as it arrives
4. On completion: show status, Run ID, duration, and link to Databricks UI
5. On failure: identify which task failed, extract the error message, explain the root cause

Common failure causes:
| Error | Likely Cause |
|-------|-------------|
| `CLUSTER_LAUNCH_FAILURE` | Cluster policy, instance type, or IAM role issue |
| `NOTEBOOK_NOT_FOUND` | Notebook path wrong or deploy not run first |
| `USER_ERROR` in notebook | Logic error — check Spark logs |
| `LIBRARY_INSTALLATION_ERROR` | Wheel version or PyPI availability |
| `TIMEOUT` | Increase `timeout_seconds` in job task definition |
| Pipeline `FAILED` | Check DLT event log for expectation violations |
