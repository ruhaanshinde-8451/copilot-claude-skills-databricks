Deploy a Databricks Asset Bundle to a target environment.

Target and options: extract from $ARGUMENTS (e.g. "deploy to staging", "deploy ingest_job prod").

Steps:
1. Auth check: `databricks --version && databricks auth env`
2. Validate first: `databricks bundle validate -t <target>` — do not proceed if validation fails
3. For `prod` target: run `databricks bundle summary -t prod`, show the user what will change, and require explicit confirmation before deploying
4. Deploy: `databricks bundle deploy -t <target>`
   - Pass any `--var` overrides extracted from $ARGUMENTS
   - On failure: diagnose, attempt one fix (e.g. add missing `--var`), retry once
   - On second failure: report the full error with diagnosis and stop
5. Verify: `databricks bundle summary -t <target>` — confirm resources appear

Common errors:
| Error | Fix |
|-------|-----|
| `cannot parse databricks.yml` | YAML syntax — check indentation |
| `PERMISSION_DENIED` | Service principal needs `CAN_MANAGE` on workspace/folder |
| `Variable ... not defined` | Add to `variables:` block or pass `--var="key=val"` |
| `Authentication error` | Run `databricks configure` or set `DATABRICKS_TOKEN` env var |
