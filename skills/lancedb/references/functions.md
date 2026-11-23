# LanceDB Functions

Read this reference first for Functions in LanceDB Enterprise: registration, computed columns, refresh jobs, failed rows, Secrets, GPUs, and chunking.
Use the legacy job guidance only for an existing workload that explicitly uses the `geneva` package.

## Check the release first

Functions is early access. Check the installed LanceDB SDK and Enterprise server versions before selecting APIs.
The Enterprise 1.14.0 documentation used a 0.42 preview Python client; stable 0.41 does not provide these APIs.
Do not assume that any preview build supports every option below.
Use the installation instructions for the deployed release, and inspect the installed SDK signatures.
Registration and worker execution require Enterprise support; upgrading the client alone is insufficient.
Python authors Functions. Do not infer TypeScript or Rust authoring support from their refresh or inspection APIs.

This reference was checked on 2026-11-23 against the public SDK documentation at commit `de81c35ba09eee802b2e916800e6d61c75b714e2`.
Documentation on `main` can describe unreleased capabilities. Match each example to the deployed release.

## Register, bind, and refresh

Resolve the Enterprise connection with [remote_connect.md](remote_connect.md).
Supply the deployment's Flight SQL endpoint when the selected operation requires it.
Keep Function definitions in a Python file or notebook so the SDK can inspect their source.
For an existing Enterprise table with a non-nullable `float64` column named `value`:

```python
from lancedb import col, function

@function
def double(value: float) -> float:
    return value * 2

version = db.create_function(double)
table.add_columns(
    {"doubled": version(value=col("value"))},
    auto_refresh_enabled=False,
)
job = table.refresh_column_async("doubled", error_policy="fail")
result = job.wait()
print(result.rows_assigned, result.rows_failed, result.rows_remaining)
rows = table.search().select(["value", "doubled"]).limit(10).to_list()
```

`create_function_async(...).wait()` is the asynchronous registration-job alternative.
Registration publishes a version; calling that version with named `col(...)` inputs creates a binding.
One `add_columns` call declares one Function binding. It does not fill existing rows.
Use `refresh_column_async(...).wait()` for Enterprise, rather than the blocking `refresh_column` method.
Inspect row failures and remaining work after completion; a completed job does not establish that every row succeeded.
Use `db.get_function(name, version=...)` to select an explicit published version.

Prefer SQL expressions for calculations that SQL can express.
Use `computed={"doubled": "value * 2"}` to retain a SQL expression for later refreshes.
An eager `add_columns({"doubled": "value * 2"})` computes existing rows once; it is a different operation.

## Authoring and dependencies

Use annotations for supported scalar signatures, or supply both `input_schema` and `output_schema` for exact Arrow types.
Check the signature matrix before using nested values, nullable outputs, Blob values, maps, or decimals.
For a model, use a class Function and initialize the model in `__init__` once per instance.
Declare worker dependencies with `pip` or `conda`; installing packages only in the client does not install them on workers.
Do not combine `pip` and `conda`. Check release support before selecting system packages or a custom runtime.
Construct unsupported captured objects inside the Function or its initializer.
A cached image does not guarantee an initialized model for a later query.

## Refresh jobs and failed rows

Functions use Enterprise jobs, not the legacy `geneva_jobs` table.
Use the returned job's `status()`, `progress()`, `wait()`, and `cancel()` methods where the installed SDK supports them.
Use `SHOW JOBS`, `DESCRIBE JOB`, and `SHOW JOB EVENTS` for SQL inspection.
A client timeout does not cancel the server job. Cancellation does not undo committed outputs.
See [remote_jobs.md](remote_jobs.md) for connection and REST inspection details.

Set `error_policy` explicitly. Defaults and skip behavior have changed between releases.
Use `fail` to stop on a row error, or `skip` with explicit failure limits when partial results are acceptable.
Inspect `SHOW ERRORS` or `table.function_errors(column="embedding")` before planning recovery.
Initialization and infrastructure failures can fail a job regardless of its row-error policy.

Fix the cause before submitting another refresh. Prefer stable `row_id` values from error records when the release supplies them:

```sql
REFRESH COLUMN embedding ON documents WHERE _rowid IN (506, 507)
  WITH (error_policy = 'fail');
```

Replace these example IDs with the actual failed row IDs.
If records contain only physical locations, calculate `_rowaddr` as `(fragment_id << 32) | row_offset`.
Physical addresses can change after compaction or rewrites; do not use stale addresses.
A `WHERE` refresh recomputes all matching live rows, including rows with existing outputs.
An unfiltered refresh normally skips current outputs, but recovery can repeat completed work.
Do not promise exactly-once execution or invent a `retry_failed` API.
Check errors and outputs again after recovery. Limit retries for external calls to operations that are safe to repeat.

## Secrets

Create a named Secret with `db.create_secret(name, value)` using a credential from the environment or a secret manager.
Bind it during registration with `secrets=[EnvVarSecret(secret_name=name, env_variable="API_KEY")]` from `lancedb`.
Read the environment variable inside the Function. Do not capture, print, or return the credential.
Secret operations require deployment encryption configuration and the applicable creation and usage permissions.
Check the Secret namespace and binding before refresh. Registration alone does not prove the Secret exists or is accessible.
Altering the named Secret supplies updated credentials without publishing a new Function version.
Follow the release's [Secrets walkthrough](https://docs.lancedb.com/enterprise/function-secrets) for endpoint and permission requirements.

## GPU resources

Discover configured GPU classes with `db.list_gpu_classes()` or `SHOW GPU CLASSES` when supported and authorized.
Use `@function(gpu=True, gpu_class="configured-class")` with a returned class name.
For a refresh, select `gpu_class` and an aggregate `gpu_count`, or use the release's supported resource budget.
The budget caps resources across workers; it does not request that many GPUs in one Python process.
Configured classes describe placement options, not currently free capacity.
Do not invent a per-worker multi-GPU option or assume model memory spans devices.

## Chunkers and automatic refresh

A chunker returns a list, commonly a list of structs. Register it as a Function.
Use `CROSS JOIN LATERAL UNNEST(...)` to expand its result into rows.
Store the expansion in a materialized view when the task needs persistent derived rows.
This one-to-many operation does not establish support for arbitrary many-to-many transforms.

Automatic refresh requires both column settings and deployment support.
Use `auto_refresh_enabled` and, where supported, `auto_refresh_interval_seconds` to configure scheduling.
Do not promise synchronous refresh on every write. Incremental refresh can recompute unchanged rows within affected fragments.
Check the release's refresh rules before estimating work or promising freshness.

## Release review

For each Enterprise release, compare this reference with the matching documentation and supported SDK versions.
Recheck registration, signatures, binding, refresh defaults, error records, Secrets, GPU budgets, and automatic refresh.
Run the Functions trigger prompts in the agent eval suite and inspect reference selection before accepting an update.
Record the reviewed release and source revision here. Coordinate recurring documentation review with UAX-245.

Canonical sources:

- [Function authoring](https://docs.lancedb.com/enterprise/function-authoring) and [operations](https://docs.lancedb.com/enterprise/function-operations)
- [Function management](https://docs.lancedb.com/management/function) and [signatures](https://docs.lancedb.com/curation/create-function)
- [Failure handling](https://docs.lancedb.com/management/function-failures)
- [Column refresh](https://docs.lancedb.com/feature-engineering/refresh-column) and [SQL options](https://docs.lancedb.com/sql/dml/refresh-column)
- [Automatic refresh](https://docs.lancedb.com/enterprise/auto-refresh) and [incremental refresh](https://docs.lancedb.com/enterprise/incremental-refresh)
- [Chunking](https://docs.lancedb.com/curation/chunking) and [SQL Functions](https://docs.lancedb.com/sql/ddl/function)
