Validate a Databricks Asset Bundle configuration before deploying.

Run: `databricks bundle validate -t <target>`

Target: extract from $ARGUMENTS, default to `dev`.

Steps:
1. Run `databricks bundle validate -t <target>`
2. On success: confirm clean with a summary of resolved resources and targets
3. On failure: identify the error type and explain the fix:
   - YAML syntax error → show the offending line and correct indentation
   - Unresolved variable → show where to add it in the `variables:` block
   - Invalid resource reference → show the exact key mismatch
   - Schema version mismatch → check `bundle.databricks_version` vs CLI version

Checklist to verify in output:
- All `${var.*}` references have defaults or target overrides
- Job task `job_cluster_key` values match defined `job_clusters` keys
- DLT pipelines have `target` (schema) set
- `run_as` identity is set for production jobs
- Permissions blocks use valid levels: `CAN_VIEW`, `CAN_MANAGE_RUN`, `CAN_MANAGE`
