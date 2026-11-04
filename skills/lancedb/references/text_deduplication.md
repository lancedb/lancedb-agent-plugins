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

The signature is binary, with a versioned envelope. Use `minhash_distance` to
compare compatible signatures; do not interpret it as an embedding or as the
raw uint16 payload of a Lance index. This statement materializes source columns
and signatures in a new table. It does not build an LSH index.

The current eager SDK `add_columns({"signature": "minhash(body)"})` and SQL
`ALTER TABLE ... ADD COLUMN ... AS minhash(body)` paths use a separate expression
planner and reject `minhash` as an unknown function. Do not recommend those
forms merely because SELECT succeeds. Verify support on the target server
before using a newer column-writing path.

## Find lexical duplicates

```sql
SELECT minhash_distance(minhash('a red fox'), minhash('a red fox')) AS distance;
SELECT * FROM text_dedupe('documents', 'body', distance_threshold => 0.2);
SELECT * FROM text_duplicate_pairs('documents', 'body', distance_threshold => 0.2);
```

The first result is zero. The batch queries compute signatures from source text.
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
`text_dedupe` when every removed row must directly match its retained row.

Index-backed batch queries require additional server support. Do not infer it
from these scalar or scan functions, or relabel a scan as indexed execution.
