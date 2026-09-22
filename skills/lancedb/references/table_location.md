# Adopting an already-written Lance dataset as a table

Use this when a user has (or will have) Lance-format data that was written
outside the normal `create_table` path — an external pipeline, a batch job,
a dataset copied in from somewhere else — and wants it to become a queryable
table against a LanceDB Enterprise/Cloud deployment, without re-uploading or
retyping the rows.

Don't work from memory — this is not fully covered in the public docs as of
this writing. It was derived by reading the LanceDB Enterprise query-node
server's REST route table and handlers (`clone_table.rs`, `declare_table.rs`
under `phalanx/src/rest/table/`) and the directory-backed namespace
implementation underneath (`lance-namespace-impls/src/dir.rs`, in the `lance`
submodule), then verified live against a running server. Re-check that
source if behavior here seems off; these internals can change.

## The answer: `clone_table`, not manual file placement

The server exposes a REST endpoint built exactly for this:

```
POST /v1/table/{name}/clone
{"source_location": "<any URI or local path the server can read>",
 "source_version": <optional>, "source_tag": <optional>}
```

Python SDK: `db.clone_table(target_table_name, source_uri, source_version=None, source_tag=None)`
(`lancedb/db.py`; the remote client routes it to the same endpoint).

`source_location` accepts an **arbitrary existing location** — a local path,
or `s3://`, `gs://`/`gcs://`, `az://`/`abfs://`/`abfss://`, `s3a://`,
`s3+ddb://`, `file-object-store://` — validated only against that scheme
allowlist, not constrained to any naming convention. It performs a **shallow
clone**: no data bytes are copied, only a new manifest is written at the
table's normal managed location, referencing the source's existing data
files. The result is a fully ordinary table — `describe`/`count_rows`/queries
all work immediately, no follow-up step needed. Verified live: cloning from
an arbitrary out-of-tree path produces a table whose version 1 is stamped
with operation `"Clone"` (`GET .../version/list?include_operations=true`) —
a good way to confirm after the fact that a table was adopted this way
rather than rebuilt from scratch (`create_table` stamps `"Overwrite"`
instead).

Two real constraints:

- **It's gated to superusers today** ("This API remains gated to superusers
  until source authorization is defined" — the handler's own comment). If the
  caller isn't a superuser, this 403s and there is currently no other
  supported way to point a table at an arbitrary existing location; say so
  plainly rather than improvising a workaround.
- Only shallow clone is supported — passing `is_shallow: false` is a 400
  ("Deep clone is not supported").
- Cloning onto a name that already exists as a real table fails outright
  ("Cannot clone to an existing table") — same collision behavior as
  `create_table`.

This is almost always the right tool when a user describes data that
*already exists* somewhere as Lance files and needs to become a table. Reach
for it before anything below.

## There is no REST path to *register* an arbitrary location either

Distinct from cloning (which materializes a new manifest), the underlying
`lance-namespace` protocol also defines a separate `register_table`
operation (point an already-existing location at a table name with no new
manifest at all). The directory namespace implementation *can* honor it, but
only when a `manifest_enabled` config flag is set on that backend — and that
flag doesn't help a real user either way: **the query-node server's REST
route table has no `/register` route at all**, only `/create`, `/declare`,
`/clone`, and `/drop` at the table level. `manifest_enabled` is an internal
deployment toggle for how the namespace tracks metadata, not a
customer-facing switch — `register_table` isn't reachable through the SDK,
CLI, or raw REST regardless of how the deployment is configured. Don't tell
a user it's an option. (This is about LanceDB Enterprise's own server; a
different lance-namespace-protocol implementation, e.g. an Iceberg/Glue/
Unity-backed catalog, could expose it — out of scope here.)

## Background: the managed-location convention

Useful for understanding what `clone_table`/`create_table` actually do, or
for the rare case where cloning genuinely isn't available (no superuser
access, and the user still needs to understand today's constraint rather
than get unblocked):

Every table's *managed* location is derived, not chosen — the namespace
computes it the same way for every table:

```rust
fn table_full_uri(&self, table_name: &str) -> String {
    format!("{}/{}.lance", self.root, table_name)
}
```

`self.root` here is already the per-database root (verified live: a table's
`describe` reports `location: "file:///<configured_root>/<database>/<table_name>.lance"`).
So for a database's root and table name `foo`, the *managed copy* always
ends up at `<database_root>/foo.lance/` — flat, one level down, `<name>.lance`
— whether it got there via `create_table`, `clone_table`, or (see below)
manual file placement. Nested/non-root namespace paths aren't supported
without a manifest/catalog namespace at all (`list_tables` and friends error
with "child namespace requires manifest" for any non-empty namespace path).

**Discovery is existence-based, not a registration call.** The namespace
considers a table "real" (versus just a reserved name) purely by checking for
at least one manifest under `_versions/`:

```rust
pub(crate) async fn path_has_actual_manifests(...) -> Result<bool> {
    let versions_path = table_path.join(VERSIONS_DIR); // "_versions"
    Ok(object_store.list(Some(versions_path)).try_next().await?.is_some())
}
```

So writing a standard Lance dataset (`data/`, `_versions/` with at least one
real manifest, etc.) directly to `<database_root>/<table_name>.lance/` also
makes it show up in `list_tables`/`describe_table` with no API call at all —
this is what a bare directory-mode namespace is doing under the hood. Don't
lead with this over `clone_table` for a user, though: it requires the
*caller* (not the server) to have write access to the raw storage path,
produces no clean audit trail (no "Clone" operation, no request id), and
silently does nothing if the manifest is missing or malformed — `clone_table`
is the supported, server-validated way to get the same outcome.

### `declare_table` — reserve the managed name first, still same-path only

Reserves a table name atomically (writes a `.lance-reserved` marker) before
data lands — useful when an external job will write the dataset out-of-band
*to the managed path itself* and the caller wants the name claimed first so
nothing else can take it. It enforces the identical location constraint:
pass a `location` other than `<database_root>/<table_name>.lance` and it's
rejected outright ("Cannot declare table {name} at location {location}, must
be at location {expected}"). It does not help adopt data sitting somewhere
else — for that, use `clone_table`.

## Workflow

1. Data already exists as Lance files somewhere (any path/URI the server can
   reach) and needs to become a table: use `clone_table` /
   `POST /v1/table/{name}/clone` with `source_location` pointing at it. This
   is the answer for the vast majority of "I have Lance files, make them a
   table" questions.
2. If that 403s (no superuser access) and the user needs to understand why,
   or is asking purely about the location convention rather than requesting
   an action: explain the managed-path convention above (`<database_root>/
   <table_name>.lance`, existence-based discovery via `_versions/`) and that
   `declare_table` only reserves a name at that same derived path — it
   doesn't relocate anything.
3. Never suggest `register_table` — it has no REST route on this server,
   regardless of configuration.
