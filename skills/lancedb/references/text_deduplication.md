# MinHash text deduplication in Enterprise SQL

Use MinHash when duplicates mean lexical overlap, such as copied documents with
small edits. Its distance estimates token-shingle Jaccard distance. Vector
deduplication answers a semantic-similarity question; exact hashes identify
identical text. Ask which criterion matters only when the request leaves it
unclear. Do not claim a performance or accuracy advantage without a comparison
using the same documents, quality target and complete processing costs.

## Check the server interface

`minhash`, `minhash_distance`, `text_duplicate_pairs` and `text_dedupe` here are
native Enterprise SQL functions. They are not methods on an Embedded or remote
SDK table, and a `db://` connection alone does not establish their availability.
Identify the deployed Enterprise release and use its SQL documentation. Use an
existing Flight SQL connection or the Enterprise `lancedb` CLI, for example
`lancedb --profile NAME --file example.sql`. The SQL endpoint can differ from the
HTTP endpoint described in [Remote connections](remote_connect.md).

This read-only query should return zero on a server with the scalar functions:

```sql
SELECT minhash_distance(minhash('a library shares research notes'),
                        minhash('a library shares research notes')) AS distance;
```

It does not prove that indexed batch deduplication is available. That route also
requires `CREATE INDEX ... USING MINHASHLSH`, the `index_name` query argument
and supporting plan executors in the deployed server release. Treat the indexed
steps below as conditional on that support. If it is absent, state the version
gap; do not invent an SDK index builder or relabel a scan as indexed execution.

## Compare the two query routes

Use a fresh native table without WAL, with a name that does not already exist.
Both deduplication queries return rows; neither deletes source rows. This keeps
the input identical when comparing results.

### 1. Write the source data

```sql
CREATE TABLE minhash_example (id BIGINT, body STRING);
INSERT INTO minhash_example VALUES
  (1, 'A library shares technical reports and research notes.'),
  (2, 'A library shares technical reports and research notes.'),
  (3, 'The coastal observatory measures rainfall and ocean temperature.'),
  (4, NULL),
  (5, '');
```

### 2. Deduplicate by scanning text

```sql
SELECT id FROM text_dedupe(
  'minhash_example', 'body', distance_threshold => 0.2
) ORDER BY id;
```

Expect IDs `1, 3, 4, 5`. NULL, empty text and text with no tokens remain in the
output. Without `index_name`, this query computes signatures from text even
when an index exists.

### 3. Build the index, on a supporting server

```sql
CREATE INDEX minhash_example_body ON minhash_example USING MINHASHLSH (body);
```

Wait for the returned index-build job to succeed before querying the index.
With the same CLI profile, use `lancedb --profile NAME job describe --job-id ID`
and `job events --job-id ID` for failures. `job describe` reports `job_state =
DONE` on success, while `job list` uses `succeeded`. Normal SQL index creation uses the
`reindex_minhash_lsh` job type. A submission response is not completion.

### 4. Deduplicate using the index

```sql
SELECT id FROM text_dedupe(
  'minhash_example', 'body', distance_threshold => 0.2,
  index_name => 'minhash_example_body'
) ORDER BY id;

EXPLAIN ANALYZE
SELECT id FROM text_dedupe(
  'minhash_example', 'body', distance_threshold => 0.2,
  index_name => 'minhash_example_body'
);
```

Expect the same IDs. The candidate operator should name `minhash_example_body`
and report `indexed_signatures=3`, `computed_signatures=0`. These counters prove
stored signatures were consumed for this fixture; they are not a benchmark.

## Semantics that affect the result

- `distance_threshold` is approximate. LSH candidate generation can miss pairs,
  and MinHash signature collisions can produce false positives. Use exact
  token-shingle comparison if an exact Jaccard threshold is required.
- The default survivor policy is connected components: a chain of near
  duplicates can leave one representative even if its endpoints do not match.
  For a direct match to the retained row, use
  `resolution => 'direct_representative'` with `text_dedupe`.
- `text_duplicate_pairs('table', 'column', distance_threshold => 0.2)` returns
  canonical row-ID pairs. Map them through `_rowid` on the same snapshot when
  returning application IDs; do not treat them as user-supplied IDs.
- Creation and query signature settings must match. Defaults are 128 hashes,
  16 bands and three-token shingles. Missing, incompatible or incompletely
  covering indices fail; update the index after appending unindexed rows.
  The indexed path does not support data overlays.
- The batch candidate stage uses one plan executor and reconstructs global
  groups from stored signatures, including matches across index segments.
  Retrieving retained source rows still reads the original table. Do not
  describe it as a top-k search or direct reuse of persisted band postings.

For a production-sized input, bound the workload and inspect its resource
requirements before running a complete batch. Returning a small result with
`LIMIT` does not make the underlying deduplication cover only those rows.
