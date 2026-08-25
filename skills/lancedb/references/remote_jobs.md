# Job operations on LanceDB Enterprise/Cloud

Jobs are server-side async operations (index builds, column backfills, materialized
view refreshes). There are **two separate job registries**; work out which one your
job is in before looking it up.

## Docs to read first

- REST auth headers (`x-api-key`, `x-lancedb-database`) and the endpoint index:
  <https://docs.lancedb.com/api-reference/rest>
- Endpoints that *start* a job and return a `job_id`:
  [trigger a column backfill](https://docs.lancedb.com/api-reference/rest/table/trigger-an-async-column-backfill-job),
  [trigger a materialized view refresh](https://docs.lancedb.com/api-reference/rest/materializedview/trigger-an-async-materialized-view-refresh)
- Geneva (feature engineering) jobs — states, job state manager (`jsm.get`, `jsm.list_jobs`),
  checkpoints: <https://docs.lancedb.com/geneva/jobs/lifecycle>; also
  [console](https://docs.lancedb.com/geneva/jobs/console),
  [troubleshooting](https://docs.lancedb.com/geneva/jobs/troubleshooting), and the
  Python `Job`/`JobResult` API: <https://lancedb.github.io/geneva/api/jobs/>

## Server job registry — `POST {base_url}/v1/jobs/*`

These four endpoints are not yet in the public REST reference, so their shapes are
recorded here. Resolve the connection per `remote_connect.md`; all are POST with a JSON
body and the standard headers. `501` on every call means job APIs are disabled on this
deployment — report that, don't retry.

| Endpoint | Body | Returns |
|---|---|---|
| `/v1/jobs/list` | optional filters: `limit`, `table_name`, `job_type`, `job_subtype`, `state`, `page_token` | `{"jobs": [{job_id, table, job_type, job_subtype, state, created_at_millis}], "page_token"}` — pass `page_token` back to page |
| `/v1/jobs/describe` | `{"job_id"}` | `{job_id, job_type, job_subtype, job_state, creation_ms, spec, status}`; `job_state` is `IN_PROGRESS`, `CANCELLED`, `FAILED`, or `DONE`; `404` unknown id |
| `/v1/jobs/cancel` | `{"job_id"}` | echoes `{"job_id"}`. Needs admin-level auth (same as `/admin` routes); `409` already terminal, `429` retry |
| `/v1/jobs/query_events` | `{"job_id"}` or `{"job_ids": [...]}`; optional `limit`, `limit_per_job`, `filter` (SQL over `state`, `updated_by`, `owner_component`, `claim_entity`) | **Arrow IPC stream, not JSON** — decode with `pyarrow.ipc.open_stream(resp.content).read_all()` |

Gotchas: list rows use lowercase `state`, describe uses uppercase `job_state`. To wait
on a job, poll `describe` until `job_state` leaves `IN_PROGRESS`; on `FAILED`, read
`status` and `query_events` for detail.

## Geneva jobs

UDF backfills and materialized view refreshes run through Geneva are tracked
**separately** — in a `geneva_jobs` table in the database's `__system` namespace, not in
`/v1/jobs`. Connect with `geneva.connect("db://<database>", api_key=..., host_override=...)`
(same credentials as `remote_connect.md`) and use the job state manager as shown in the
lifecycle docs above. Things the docs don't say:

- `list_jobs` defaults to `status="RUNNING"`; pass `status=None` for all jobs.
  `CANCELLED` is also a valid status.
- For filters `list_jobs` lacks (e.g. time ranges), query the table directly:
  `jsm.get_table(True).search().where(...)` — `True` checks out the latest version.
- Nothing reaps dead jobs. Treat `RUNNING`/`PENDING` as failed if the job has run
  > ~36h or `updated_at` is > ~2h old (the console UI applies the same heuristic).
