Run the full Databricks pipeline end-to-end: validate → deploy → run → integration tests → report.

Arguments: $ARGUMENTS (job or pipeline key, target environment, optional runtime params)

Execute every step below sequentially. Use TodoWrite to track progress. Do NOT pause between steps unless the target is prod (requires explicit confirmation before Step 4).

---

## Step 1 — Auth & CLI Check

```bash
databricks --version
databricks auth env
```

Stop if auth fails. Tell the user to run `databricks configure` or set `DATABRICKS_HOST` + `DATABRICKS_TOKEN`.

---

## Step 2 — Read Bundle Config

```bash
cat databricks.yml
```

- Extract resource keys from `resources.jobs` and `resources.pipelines`
- Extract available targets
- Infer resource key and target from $ARGUMENTS; default target = `dev`
- If one resource exists and none specified, use it automatically
- If multiple exist and none specified, list them and ask

---

## Step 3 — Validate

```bash
databricks bundle validate -t <target>
```

- On failure: identify error type, fix if possible (YAML syntax, unresolved variable), re-validate
- If fix requires human input: stop and explain exactly what to change

---

## Step 4 — Deploy

```bash
databricks bundle deploy -t <target>
```

**If target is `prod`:** STOP here. Run `databricks bundle summary -t prod`, show resources that will change, and require explicit user confirmation before continuing.

- On failure: diagnose, attempt one fix (e.g. add `--var` for a missing variable), retry once
- On second failure: stop with full error and diagnosis

---

## Step 5 — Run

```bash
databricks bundle run <resource_key> -t <target>
```

Map parameters from $ARGUMENTS:
- Named params → `--python-named-params "k=v"`
- Notebook widgets → `--notebook-params "k=v"`
- DLT full refresh → `--refresh-all`

- Stream output as it arrives
- On failure: identify the failing task and root cause from the output

---

## Step 6 — Integration Tests

Discover tests in `tests/integration/`. If the directory does not exist, skip and note it.

**Priority order:**

1. **pytest** (`test_*.py`) — install deps then run:
   ```bash
   pip install -r tests/requirements-test.txt 2>/dev/null || true
   DATABRICKS_TARGET=<target> python -m pytest tests/integration/ -v --tb=short 2>&1
   ```

2. **SQL assertions** (`*.sql`) — for each file:
   ```bash
   databricks sql execute --warehouse-id <warehouse_id> -f tests/integration/<file>.sql
   ```

3. **Notebook tests** (`notebook-tests.json`) — for each entry `{ "notebook": "...", "params": {} }`:
   ```bash
   databricks jobs run-now --job-id <test_job_id> --notebook-params "{...}"
   ```

On test failure: report failing test names and assertion messages. Do NOT re-run the job.

---

## Step 7 — Outcome Report

End with a structured summary:

```
✓ Validated    databricks.yml — no errors
✓ Deployed     [dev] my_bundle → https://<host>/jobs/<id>
✓ Run          my_ingest_job  → Run ID: 12345 | Duration: 4m 32s | SUCCESS
✓ Int Tests    8 passed, 0 failed
```

On any failure, report which step failed, the exact error, and what to fix.
