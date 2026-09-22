# Landing Lance files directly under a root URI

Use this when a user wants to know where new Lance-format dataset files must
live for a table to be recognized against a LanceDB Enterprise/Cloud
deployment's root URI (e.g. `s3://<bucket>/<prefix>`), or asks about
registering an already-written Lance dataset as a table.

Don't work from memory — this is not fully covered in the public docs as of
this writing. It was derived by reading the directory-backed namespace
implementation (`lance-namespace-impls/src/dir.rs`, in the `lance` submodule).
Re-check that source if behavior here seems off; namespace internals can
change.

## Two namespace modes, very different answers

A LanceDB Enterprise/Cloud namespace backed by a bare root URI runs in one of
two modes, and which one applies changes the answer completely:

- **Plain directory mode** (no separate catalog/manifest store configured) —
  the default for a namespace that's just a root URI with nothing else set
  up. Table locations are **derived, not chosen**.
- **Manifest mode** (a catalog/metastore layered on top of the root URI) —
  table locations can be arbitrary; tables are looked up through the catalog
  instead of by listing the root.

Ask (or check the deployment's namespace config) which mode applies before
assuming either answer below. If you don't know, default to assuming plain
directory mode — it's the common case for a namespace that's just a root URI
with no catalog mentioned.

## Plain directory mode: the location is fixed

The namespace derives every table's location the same way:

```rust
fn table_full_uri(&self, table_name: &str) -> String {
    format!("{}/{}.lance", self.root, table_name)
}
```

So for root `s3://<bucket>/<prefix>` and table name `foo`, the **only** valid
location is:

```
s3://<bucket>/<prefix>/foo.lance/
```

Flat, one level under root, `<table_name>.lance`. Nested/non-root namespace
paths aren't supported in this mode at all (`list_tables` and friends error
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

So to land files directly and have a table "just appear": write a standard
Lance dataset (the normal on-disk layout — `data/`, `_versions/` with at
least one manifest, etc.) straight to `<root>/<table_name>.lance/`. As long as
directory listing is enabled (the default absent a catalog), `list_tables` /
`describe_table` pick it up with no separate call. The straightforward way to
produce that layout is plain OSS `pylance`:

```python
import lance
lance.write_dataset(table, "s3://<bucket>/<prefix>/<table_name>.lance")
```

or copying an existing Lance dataset's files to that exact path verbatim.

### `declare_table` — reserve the name first, still same-path only

Reserves a table name atomically (writes a `.lance-reserved` marker) before
data lands — useful when an external job will write the dataset out-of-band
and you want the name claimed first so nothing else can take it. It enforces
the identical location constraint: pass a `location` other than
`<root>/<table_name>.lance` and it's rejected outright:

```
"Cannot declare table {name} at location {location}, must be at location {expected}"
```

It does not relax the convention — it only lets you claim the name before the
manifest exists.

### `register_table` — only exists in manifest mode

Pointing an arbitrary, already-existing location at a table name (bypassing
the naming convention) is a real operation on the lance-namespace protocol —
but in plain directory mode it hard-errors:

```
"register_table is only supported when manifest mode is enabled"
```

If the user needs to register a dataset sitting at some other path, that
requires a manifest/catalog namespace on this deployment. Confirm that before
telling them `register_table` is an option — absent a catalog, it is not.

## Workflow

1. Determine which mode the deployment runs (ask if unclear; default to
   assuming plain directory mode).
2. **Plain directory mode:** compute `<root>/<table_name>.lance`, tell the
   user their Lance dataset must land there exactly, and that no separate
   registration step is required once `_versions/` has a manifest —
   `declare_table` is available to reserve the name first, but the location
   argument must match the derived path exactly.
3. **Manifest mode:** `register_table` can point at an arbitrary existing
   location; use it instead of the fixed-path convention.
4. Either way, don't assume a location will simply be picked up without at
   least one real manifest under `_versions/` — a directory with only raw
   data files and no manifest is invisible to `list_tables`.
