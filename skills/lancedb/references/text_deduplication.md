# MinHash signatures and text deduplication in Enterprise SQL

Use MinHash when duplicates mean lexical overlap, such as copies with small
edits. Its distance estimates token-shingle Jaccard distance. Vector
deduplication answers a semantic-similarity question; exact hashes identify
identical text. Confirm the criterion when the request leaves it unclear.
Do not claim a performance advantage without comparable measured evidence.

## Generate signatures with SQL

The simplest form is a scalar call:

```sql
SELECT minhash('the quick brown fox') AS signature;
```

For an existing text column:

```sql
SELECT id, minhash(body) AS signature FROM documents;
```

These are native Enterprise SQL functions, not portable SDK table methods.
Identify the deployed server release and its SQL capabilities; a newer client
or a `db://` connection does not establish support. Use an existing Flight SQL
connection or the Enterprise CLI, for example
`lancedb --profile NAME --file example.sql`. The SQL endpoint can differ from the
HTTP endpoint described in [Remote connections](remote_connect.md).

## Persist signatures in a new table

Choose a fresh destination name:

```sql
CREATE TABLE documents_with_signatures AS
SELECT *, minhash(body) AS signature FROM documents;
```

Signatures are `FixedSizeList<UInt16>` values with configuration metadata.
Preserve both the values and field metadata. Use
`minhash_distance` to compare compatible signatures; do not interpret a
signature as an embedding or assume raw Lance index values carry the required
SQL metadata. This statement materializes source columns and signatures in a
new table. It does not build an LSH index.

## Add and reuse a signature column

On a server with eager MinHash column-writing support, the Python SDK can add
signatures to an existing table:

```python
table.add_columns({"signature": "minhash(body)"})
```

Then compare both paths through the same SQL API:

```sql
-- Read the stored hashes.
SELECT * FROM dedupe('documents', 'signature', distance_threshold => 0.2);
-- Compute hashes from text during this query.
SELECT * FROM dedupe('documents', 'body', distance_threshold => 0.2);
```

Saved-signature input requires its own server support in addition to column
writing. The field metadata identifies a MinHash signature and supplies omitted
hash-count and shingle settings; ordinary integer lists are not signatures.
Explicit settings must agree with the metadata. Keep eager signatures updated
when their source text changes. Verify `computed_signatures=0` and
`column_signatures` in EXPLAIN ANALYZE when demonstrating reuse.

The [two-path Sophon demo](https://github.com/lancedb/sophon/tree/main/tools/minhash-demo)
writes data, calls the native eager endpoint used by the SDK, and compares both
results and execution metrics. It does not require an LSH index.

Older servers can reject MinHash column expressions even when SELECT works.
Deferred `computed=` columns and SQL `ALTER TABLE ... ADD COLUMN ... AS` use a
different path; eager support does not establish support for those forms.

## Find lexical duplicates

```sql
SELECT minhash_distance(minhash('a red fox'), minhash('a red fox')) AS distance;
SELECT * FROM dedupe('documents', 'body', distance_threshold => 0.2);
SELECT * FROM duplicate_pairs('documents', 'body', distance_threshold => 0.2);
```

The first result is zero. These batch queries compute signatures from source text.
The unified functions dispatch between text, tagged signatures and vector input;
the `text_dedupe` and `text_duplicate_pairs` names remain text aliases.
They require native tables without WAL and use one plan executor. A small
result LIMIT does not restrict the underlying deduplication workload. Returned
rows do not delete the source, and pair row IDs are not user IDs: map them with
`_rowid` at the same snapshot when returning application IDs.

Defaults are 128 hashes, 16 candidate bands and three-token shingles. NULL,
empty and tokenless text produce no signature and remain in deduplicated output.
LSH can miss qualifying pairs and signatures can collide; use exact token-shingle
comparison when the decision must enforce an exact Jaccard threshold.
The default policy keeps one representative per connected group, including
transitive matches. Use `resolution => 'direct_representative'` with
`dedupe` when every removed row must directly match its retained row.

Index-backed batch queries require additional server support. Do not infer it
from these scalar or scan functions, or relabel a scan as indexed execution.
