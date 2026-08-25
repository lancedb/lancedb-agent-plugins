# Column Metadata Authoring

Write column-level descriptions, tags, and logical groupings onto a LanceDB table's schema. Use this when the user wants to document, annotate, tag, or classify what their table columns ARE (embeddings vs labels vs eval metrics, model provenance, version families, etc.).

Column metadata is arbitrary Arrow field key/value pairs. Read the schema through the table handle, write through `update_field_metadata` (Python) / `updateFieldMetadata` (TypeScript). Works on local/OSS and remote Enterprise/Cloud tables alike.

Don't work from memory — read the public docs for the current API:

- **Update field metadata (concepts + Python/TypeScript/Rust examples, merge vs `replace`, deleting a key):** <https://docs.lancedb.com/tables/schema>
- **Python API reference:** <https://lancedb.github.io/lancedb/python/python/#lancedb.table.Table.update_field_metadata> (async: `AsyncTable.update_field_metadata`)
- **TypeScript API reference** (`updateFieldMetadata`, `FieldMetadataUpdate`): <https://lancedb.github.io/lancedb/js/classes/Table/#updatefieldmetadata>

Notes the docs may not state prominently:

- **Arrow field metadata is bytes-keyed in Python** — `field.metadata` is `dict[bytes, bytes]` (e.g. `{b"lancedb:description": b"..."}`), not `str`. In TypeScript it's a `Map<string, string>`.
- Read existing metadata before writing, so you don't clobber or redundantly rewrite it. Merge (the default) preserves keys you don't mention; only use `replace: true` if the user explicitly asks to overwrite.
- Batch all field updates into a single call. The call returns the new table version — report it.
- For struct/nested fields, recurse into the field's children and address them as dot-paths (e.g. `parent.child`).
- `replace_field_metadata` is deprecated — use `update_field_metadata`.

## Key conventions

These namespaced keys are this skill's convention, not a LanceDB API contract:

| Key | Purpose | Example value |
|-----|---------|---------------|
| `lancedb:description` | Human-readable explanation of what the column contains | `"CLIP ViT-L/14 image embedding, L2-normalized (768-dim)"` |
| `lancedb:tag:<name>` | Flexible key-value tag; the suffix names the tag category | `lancedb:tag:field_type: "embedding"`, `lancedb:tag:model: "clip"`, `lancedb:tag:project_id: "foo"` |
| `lancedb:logical-column` | Logical group/family this column belongs to | `"clip_features"` |

Tags are open-ended — the suffix describes *what is being classified* (`field_type`, `model`, `project_id`, `version`), the value describes *how*. Multiple tags per column are fine; each is a separate key. All values are strings.

## Authoring heuristics

If the user hasn't specified which columns to update, work with all columns.

**Descriptions** — base them on the column name, its Arrow type, and any user-supplied context (upstream pipeline, sample values, domain knowledge). Name patterns: `_embedding`/`_vec`/`_embed` → vector; `_label`/`_class` → label; `_score`/`_eval`/`_metric` → evaluation metric. Be specific and concise. Good: `"Sentence-BERT embedding of the query text (768-dim)."` Not: `"An embedding column."`

**Tags** — choose key names matching what the user asked to annotate. Common: `field_type` (`embedding`/`text`/`image`/`label`/`eval`/`id`/`metadata`), `model` (`clip`/`bert`/`vit`), `project_id`, `version`. Use the Arrow type as a hint: `FixedSizeList` + float → embedding; `Utf8`/`LargeUtf8` → text; `Binary` → image or blob.

**Logical groupings** — look for version patterns across column names (`clip_v1`/`clip_v2`/`clip_v3` → logical column `"clip"`; `text_embed_20240101`/`text_embed_20240601` → `"text_embed"`). Write `lancedb:logical-column` on every member of the group, and mark the newest with `lancedb:tag:latest: "true"` in addition to its version tag.

## Workflow

1. Read the schema and existing field metadata.
2. Generate descriptions / tags / groupings per the heuristics above.
3. Write everything in one batched `update_field_metadata` call.
4. Report which columns were updated and what was written, the new table version, and any columns skipped (e.g. already up to date). For a grouping task, show the grouping.
