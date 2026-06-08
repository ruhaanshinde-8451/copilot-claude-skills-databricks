Run integration tests against a deployed Databricks job or pipeline to verify output correctness.

Arguments: $ARGUMENTS (optional: target environment, specific test file or pattern)

Default target: `dev`. Tests run against the already-deployed bundle — this command does NOT deploy.

---

## Step 1 — Discover Tests

Check for test files in `tests/integration/`:

```bash
ls tests/integration/ 2>/dev/null || echo "No tests/integration/ directory found"
```

Classify what is present:
- `test_*.py` — pytest integration tests
- `*.sql` — SQL assertion files
- `notebook-tests.json` — notebook-based test definitions

If no tests directory exists: inform the user and suggest the scaffolding below.

---

## Step 2 — Run pytest Tests

If `test_*.py` files are found:

```bash
# Install test dependencies if present
pip install -r tests/requirements-test.txt 2>/dev/null || true

# Run with target injected as env var
DATABRICKS_TARGET=<target> python -m pytest tests/integration/ -v --tb=short 2>&1
```

The `DATABRICKS_TARGET` env var lets tests connect to the right workspace/catalog/schema.

Common test patterns to look for in test files:
- Row count assertions on output Delta tables
- Schema validation (column names, types)
- Data quality checks (null rates, range checks)
- Idempotency: running the job twice produces the same result

---

## Step 3 — Run SQL Assertion Tests

If `*.sql` files are found in `tests/integration/`:

```bash
# Run each SQL file — each should return zero rows if the assertion passes
databricks sql execute --warehouse-id <warehouse_id> -f tests/integration/<file>.sql
```

Convention: SQL assertion queries return rows **only on failure**. Zero rows = test passed.

Example assertion file (`test_bronze_not_empty.sql`):
```sql
-- Fails if bronze table has no rows
SELECT 'bronze table is empty' AS assertion_failure
FROM my_catalog.bronze.events
HAVING COUNT(*) = 0
```

---

## Step 4 — Run Notebook Tests

If `tests/integration/notebook-tests.json` exists, read it and run each entry:

```json
[
  {
    "notebook": "/Workspace/Shared/tests/test_silver_transform",
    "params": { "env": "dev" },
    "timeout_seconds": 300
  }
]
```

Run each via a one-time job:
```bash
databricks runs submit --json '{
  "run_name": "int-test-<notebook>",
  "existing_cluster_id": "<cluster_id>",
  "notebook_task": {
    "notebook_path": "<notebook>",
    "base_parameters": { "env": "<target>" }
  }
}'
```

Poll for completion and check final state — expect `SUCCESS`.

---

## Step 5 — Report

```
Int Test Results — [dev]
────────────────────────────────────
pytest        8 passed, 0 failed
SQL assertions 3 passed, 0 failed
Notebooks      2 passed, 0 failed

✓ All tests passed
```

On failure:
```
✗ Int Tests   2 failed

  FAILED tests/integration/test_bronze.py::test_row_count
    AssertionError: expected >= 1000 rows in bronze.events, got 0
    Hint: Check that the ingest job wrote to the correct catalog/schema for dev target

  FAILED tests/integration/test_schema.py::test_event_columns
    KeyError: column 'event_date' not found in schema
    Hint: Notebook task may be writing 'event_ts' instead — check column mapping
```

---

## Scaffolding New Tests

If `tests/integration/` does not exist, create the structure:

```
tests/
├── integration/
│   ├── test_<job_name>.py       # Main pytest file
│   ├── assertions.sql           # SQL quality checks
│   └── notebook-tests.json      # Notebook test registry
└── requirements-test.txt        # pytest, databricks-sdk, etc.
```

Minimal `requirements-test.txt`:
```
pytest>=7.0
databricks-sdk>=0.20
```

Minimal pytest test file:
```python
import os
from databricks.sdk import WorkspaceClient

TARGET = os.environ.get("DATABRICKS_TARGET", "dev")

def test_output_table_not_empty():
    w = WorkspaceClient()
    result = w.statement_execution.execute_statement(
        warehouse_id=os.environ["DATABRICKS_WAREHOUSE_ID"],
        statement=f"SELECT COUNT(*) as cnt FROM my_catalog_{TARGET}.bronze.events"
    ).result.data_array
    count = int(result[0][0])
    assert count > 0, f"Expected rows in bronze.events, got {count}"
```
