# Full-Text Search

NeorunBase ships a built-in BM25 full-text search engine alongside its vector and relational paths. The surface is PostgreSQL-compatible — `tsvector`, `tsquery`, the `@@` match operator, and the `ts_rank` family — so existing client code works unchanged. Underneath, every shard owns its own Lucene-based inverted index, persisted as an encrypted sidecar in the same way the HNSW vector index is.

## Why It's Built In

Most stacks bolt search onto a separate Elasticsearch / OpenSearch cluster, which forces a synchronisation pipeline (Debezium, Kafka, custom batch jobs) and gives up cross-system ACID. NeorunBase does it inside the same database so:

- **One transaction** writes a document and its searchable form atomically.
- **One SQL statement** can `JOIN` searchable text with relational metadata, filter by `WHERE`, and rank by BM25 — no glue layer.
- **One operational surface** — the same backup, replication, IAM, and encryption story already in place for OLTP applies to search.
- **Iceberg auto-sync** carries searchable text into the lakehouse with no extra work.

## TSVECTOR / TSQUERY Types

Two PG-native types are added with their canonical OIDs (`3614` / `3615`) so JDBC, psycopg, and ORM clients recognise them without driver changes:

```sql
-- tsvector / tsquery flow as opaque text on the wire; Lucene tokenises
-- on the data node when CREATE INDEX is built and when search runs.
SELECT to_tsvector('english', 'machine learning fundamentals');
SELECT plainto_tsquery('korean', '한국어 검색');
```

The function set tracks PostgreSQL: `to_tsvector`, `plainto_tsquery`, `phraseto_tsquery`, `websearch_to_tsquery`, `ts_rank`, `ts_rank_cd`. The match operator `a @@ b` is recognised before the SQL reaches the planner and lowered to `tsmatch(a, b)` — no driver-side transformation needed.

## CREATE INDEX … USING fts

A single DDL line creates the per-shard inverted index. Configuration sits in the standard `WITH` clause:

```sql
CREATE TABLE articles (
    id   BIGINT PRIMARY KEY,
    body TEXT
) SHARD KEY (id) SHARDS 16;

-- No language declared: the configured default analyzer (see below).
CREATE INDEX idx_body
  ON articles
  USING fts (body);

-- PostgreSQL spells a full-text index USING gin; it is accepted and means the same thing.
CREATE INDEX idx_body_pg
  ON articles
  USING gin (body);

-- Korean: Lucene's official Nori morphological analyzer.
CREATE INDEX idx_body_ko
  ON articles
  USING fts (body)
  WITH (lang = 'korean');
```

### `lang` values

`lang` accepts any analyzer NeorunBase registers:

| Group | Names |
| --- | --- |
| Generic | `standard`, `simple`, `cjk` |
| CJK morphology | `korean`/`ko` (Nori), `japanese`/`ja` (Kuromoji), `chinese`/`zh` (SmartCN) |
| English | `english`/`en` (Porter stemmer + stop words) |
| European | `french`, `german`, `spanish`, `italian`, `portuguese`, `dutch`, `russian`, `norwegian`, `swedish`, `danish`, `finnish`, `czech`, `hungarian`, `romanian`, `bulgarian`, `greek`, `latvian`, `lithuanian`, `galician`, `catalan`, `basque`, `irish`, `armenian`, `brazilian` |
| Middle East / South Asia | `arabic`, `persian`, `turkish`, `hindi`, `bengali`, `thai`, `indonesian` |

Two-letter codes (`ko`, `ja`, `de`, …) work as aliases. An unrecognised name falls back to the
default **and logs a warning** — a typo like `lang = 'korea'` will not silently give you
whitespace tokenisation.

### The default analyzer

An index that declares no `lang` uses `neorunbase.fts.default.analyzer`, which is **`cjk`**.

That default is deliberate. `cjk` tokenises Latin text the way a standard analyzer does and
additionally bigrams CJK runs. A standard analyzer cuts Korean on whitespace only, so `방수자재`
becomes a single token and a search for `방수` does not match it — and it fails to match *only
once an index exists*, because the unindexed path matches substrings. Adding an index would
change the query's answer, which is the one thing an index must never do.

For a corpus you can name a language for, **declare it**: `WITH (lang = 'korean')` runs real
morphological analysis (Nori) rather than bigrams, and gives better recall and smaller indexes.

### Primary Key Requirement

Like HNSW, FTS addresses documents by a numeric primary key (`INT`, `BIGINT`, or `SMALLINT`). This keeps a search hit round-tripping straight back to the row store via PK lookup — no secondary `id → pk` map.

## Querying — `@@` and `ts_rank`

```sql
-- Simple match — boolean filter only.
SELECT id, body
FROM articles
WHERE body @@ 'machine learning'
LIMIT 10;

-- Ranked retrieval — the planner recognises this shape and routes it
-- through the per-shard FTS scatter-gather, returning the global top-K
-- by BM25 score.
SELECT id, body, ts_rank(body, 'machine learning') AS score
FROM articles
WHERE body @@ 'machine learning'
ORDER BY score DESC
LIMIT 10;
```

Lucene query syntax is accepted in the right operand: plain words, `AND` / `OR` / `NOT`, `"quoted phrases"`, `field:term`, and `+required` / `-excluded` prefixes.

### When the query is *not* pushed down

`@@` is an ordinary operator, exactly as in PostgreSQL: it works without an index, and an index
only makes it fast. The planner routes a query through the per-shard BM25 index when all of the
following hold, and evaluates it on the coordinator otherwise:

- an **FTS index covers the column** (a plain `CREATE INDEX` is a BTREE and cannot serve `@@` —
  check with `\di`, which shows each index's method);
- the query has a **`LIMIT`** (there is no top-K to fetch without one);
- the `@@` predicate is the **whole `WHERE` clause**, not one side of an `AND`;
- there is no `GROUP BY` / aggregation, and any `ORDER BY ts_rank(...)` is `DESC`.

The unindexed path returns the **same rows** — all query terms must be present, CJK terms match
inside a word — and `ts_rank` still produces a usable relevance ordering, computed locally from
term frequency and document length rather than corpus-wide BM25. It is slower, not different.
When a query is declined, the reason is logged at debug by `FtsPlanAnalyzer`, so "why is this
slow" has an answer that does not require reading the planner.

### Seeing your indexes

```
\di
        Schema |   Name    |    Type      | Owner      | Table
       --------+-----------+--------------+------------+----------
        public | idx_body  | index (fts)  | neorunbase | articles
        public | idx_id    | index (btree)| neorunbase | articles
```

The method matters: an index created **without** a `USING` clause is a BTREE. It will be created
successfully, it will appear here, and it cannot serve a full-text query.

## Analyzer Lifecycle — Index-Time and Query-Time

The analyzer chosen at `CREATE INDEX` time is **baked into that index** and applied uniformly afterwards:

1. At `CREATE INDEX … WITH (lang='korean')`, the chosen `lang` is recorded in the index definition and travels alongside every shard's sidecar.
2. Every subsequent `INSERT` that reaches a shard goes through `FtsIndexMaintenance.onInsert`, which loads the same analyzer for that index and tokenises the new document with it before adding to the per-shard Lucene index.
3. Every `WHERE col @@ q` query on that index parses the right-hand expression through the **same analyzer**. The coordinator reads `lang` from the catalog and carries it on the search request, so the Data Node opens the Lucene index with the analyzer it was *built* with — including when a search is the first thing to touch that index after a restart, before any write has re-opened it. Index-time and query-time tokens always agree by construction.

That's why the `lang` choice flows in the index DDL rather than per-INSERT: keeping it on the index means a single decision at CREATE time governs both writes and reads. The same value is carried on the `HYBRID_SEARCH(...)` path, so the FTS half of a hybrid blend analyses its query text identically to a plain `@@` query.

### What That Means in Practice

The analyzer doesn't language-detect. A Korean-configured index runs the **Nori morphological analyzer on every input it sees**, whether the text is Korean, English, or mixed:

```sql
-- Korean-configured index over a TEXT column.
CREATE INDEX idx_body_ko
  ON articles
  USING fts (body)
  WITH (lang='korean');

-- Korean with a particle (조사) attached. Nori decomposes it.
INSERT INTO articles VALUES (1, '한국어는 어렵습니다');

-- Bare noun query — matches because Nori split '한국어는' → '한국어' + '는'
-- on the way in, and lower-cased '한국어' on the query side too.
SELECT id FROM articles WHERE body @@ '한국어';
-- → id 1

-- Mixed Korean + English: Nori still tokenises Korean morphemes and
-- lower-cases the English half. One analyzer, both languages.
INSERT INTO articles VALUES (2, 'Spring Boot 한국 사용자 모임');

-- Either side matches the same row.
SELECT id FROM articles WHERE body @@ 'spring';   -- → id 2
SELECT id FROM articles WHERE body @@ '사용자';   -- → id 2
```

### Supported Languages

The bundled analyzers cover the languages listed below. Each name is recognised in long form and as the ISO 639-1 / locale code (e.g. `'korean'`, `'ko'`, `'ko_kr'`).

| Group | Languages |
|---|---|
| **CJK morphology** | Korean (Nori), Japanese (Kuromoji), Chinese (SmartCN), generic CJK bigram |
| **Major European** | English, French, German, Spanish, Italian, Portuguese (incl. Brazilian), Dutch, Russian |
| **Northern / Eastern European** | Norwegian, Swedish, Danish, Finnish, Latvian, Lithuanian, Czech, Polish (Czech-style fallback), Hungarian, Romanian, Bulgarian, Greek, Galician, Catalan, Basque, Irish, Armenian |
| **Middle East / South Asia** | Arabic, Persian, Turkish, Hindi, Bengali, Thai, Indonesian |

Each comes with the appropriate stemmer / lemmatiser and stopword list bundled by Apache Lucene — no external dictionary install is required at runtime.

```sql
-- Japanese — Kuromoji (IPADIC dictionary).
CREATE INDEX idx_body_ja ON articles USING fts (body) WITH (lang='japanese');

-- Chinese — Smart Chinese (HMM segmenter).
CREATE INDEX idx_body_zh ON articles USING fts (body) WITH (lang='chinese');

-- German — Snowball stemmer + lower-casing.
CREATE INDEX idx_body_de ON articles USING fts (body) WITH (lang='german');

-- Arabic — handles RTL + Arabic-specific stopwords.
CREATE INDEX idx_body_ar ON articles USING fts (body) WITH (lang='arabic');
```

If `lang` names a configuration the server doesn't know, NeorunBase falls back to the standard analyzer rather than refusing the DDL — same as PostgreSQL's `default_text_search_config` fallback.

### Default Is English

When `lang` is omitted, the index uses Lucene's `StandardAnalyzer` (English-style tokenisation, lower-casing, light stop words). This matches PostgreSQL's behaviour where `default_text_search_config` is `english` out of the box. For Korean-language content, declare `WITH (lang='korean')` explicitly — the cost of running Nori on a row that happens to be all English is negligible, but the cost of running the English Standard analyzer over Korean text is loss of meaningful tokenisation (no particle decomposition, no compound splitting).

A typical Korean schema:

```sql
CREATE TABLE products (
    id          BIGINT PRIMARY KEY,
    title       TEXT,
    description TEXT
) SHARD KEY (id) SHARDS 16;

CREATE INDEX idx_title_ko ON products USING fts (title)       WITH (lang='korean');
CREATE INDEX idx_desc_ko  ON products USING fts (description) WITH (lang='korean');
```

## Distributed BM25 Across Shards

A FTS-indexed table has **one Lucene index per shard**. On a top-K query, the Coordinator scatters an `FTS_SEARCH_REQ` to each shard's reading replica; every Data Node runs its local Lucene search, applies `WHERE` pre-filter pushdown if present, and returns its local top results; the Coordinator merges them into the global top-K by descending BM25. Each per-shard leg is bounded by `neorunbase.search.scatter.stage.timeout.ms` (default 30000) — the same knob shared by the Vector and Hybrid scatters.

Each shard is searched on exactly **one** replica — the one [read placement](replication-ha.md#read-placement-failover) selects — so a replicated shard contributes its documents once. If that node does not answer, its shards are regrouped onto their surviving replicas and retried, so one unreachable Data Node degrades into a retry instead of failing the query. Under `round_robin` read placement the shards spread across replicas, so BM25 scoring for one query runs on more nodes at once.

The index that answers a search is opened with the analyzer recorded at `CREATE INDEX` time — the coordinator carries the index's `lang` on the request — and can be [prewarmed](vector-database.md#resident-memory-budget-eviction-prewarm) at startup so the first query after a restart does not pay the sidecar open.

NeorunBase uses **shard-local BM25** scoring by default — each shard scores against its own document frequency. This matches OpenSearch's default `query_then_fetch` mode and trades a small recall hit on heavily-skewed data distributions for one fewer network round-trip. A future `dfs_query_then_fetch` mode (coordinator collects global df first, then re-scores) is on the roadmap; the wire protocol leaves room for it without breaking changes.

## Hybrid Retrieval — BM25 + Vector in One SQL

Because vector search and full-text search are both first-class operations on the same table, hybrid retrieval is just an arithmetic blend in the SELECT:

```sql
SELECT
    d.id, d.title, d.body,
    ts_rank(d.body, plainto_tsquery('한국어 검색'))    AS text_score,
    d.embedding <=> :query_embedding                  AS vec_distance,
    -- α=0.4 BM25 + β=0.6 vector similarity (1 / (1 + distance))
    ts_rank(d.body, plainto_tsquery('한국어 검색')) * 0.4
      + (1 / (1 + (d.embedding <=> :query_embedding))) * 0.6 AS rank
FROM documents d
JOIN tenants t ON d.tenant_id = t.id
WHERE t.id = :tenant_id
  AND d.body @@ plainto_tsquery('한국어 검색')
ORDER BY rank DESC
LIMIT 10;
```

This single statement enforces tenant isolation (`JOIN` + `WHERE`), restricts via BM25 (`@@`), and ranks by a hybrid of BM25 score and vector similarity — all in one ACID transaction over the same indexes.

For workloads where the application doesn't want to manage the score-blending arithmetic by hand — and especially when the blending strategy needs to switch between min-max and Reciprocal Rank Fusion at the call site — NeorunBase exposes the same retrieval as a `HYBRID_SEARCH(...)` table-valued function. It runs the FTS and ANN scatter-gathers in parallel inside the engine and returns a single sorted `(id, score)` list ready to JOIN against the row store:

```sql
SELECT d.*, h.score
FROM HYBRID_SEARCH(table     => 'public.documents',
                   ts_query  => '한국어 검색',
                   ts_index  => 'idx_doc_body',
                   vec_query => :embedding_literal,
                   vec_index => 'idx_doc_embedding',
                   alpha => 0.4, beta => 0.6, k => 50,
                   blend => 'minmax')      -- or 'rrf' for rank fusion
       h
JOIN documents d ON d.id = h.id
ORDER BY h.score DESC
LIMIT 10;
```

See [Hybrid Search](hybrid-search.md) for the full argument reference and the SMBA-style composition with `PERSONALIZED_PAGERANK` + `GRAPH_PATH_EXISTS`.

## Storage Layout — Per-Segment KMS Envelope on Disk

A Lucene index can't live efficiently inside RocksDB (per-segment file format, hundreds of random hops per query). NeorunBase stores each shard's Lucene index as a tree of **independently KMS-enveloped segment files** under `<shardDir>/sidecars/fts/<index>/`. The unit of encryption is one Lucene file (segment / commit pointer / metadata), not the whole index — so the index can grow to disk capacity while only the hot working set is materialised in heap.

```
shard-N/
└── sidecars/
    └── fts/
        ├── public.articles.idx_body/
        │   ├── _0.cfe          ← encrypted Lucene segment (chunked AES-GCM,
        │   ├── _0.cfs              per-file KMS-wrapped DEK in the header)
        │   ├── _0.si
        │   ├── _1.cfe
        │   ├── …
        │   └── segments_3      ← encrypted commit pointer
        └── public.articles.idx_title/
            └── …
```

Each file's header carries a fresh AES-256 DEK wrapped by the shard's KMS key. Disk content is therefore always ciphertext; the OS page cache never sees plaintext index pages.

### Why per-segment, not per-page

Two boundary points were rejected on the way to this design:

1. **Whole-index in JVM heap** (the prior MVP) — strong envelope, but the index size ceiling is node RAM. Doesn't scale to PB.
2. **Standard `MMapDirectory` + OS-level disk encryption (LUKS / EBS)** — Elasticsearch's path. PB scale, but the OS page cache holds plaintext. NeorunBase's "every byte at rest is wrapped by KMS" invariant doesn't hold strictly.
3. **Per-segment KMS envelope + bounded plaintext LRU** — the chosen path. Snowflake / Databricks micro-partition pattern adapted for Lucene segments. Disk encrypted, hot working set fast, KMS calls scale with file count (segments) not query rate.

### Plaintext LRU cache

A `EncryptedSegmentDirectory` opens with a configurable cap (`neorunbase.fts.plaintext.cache.bytes`, default 256 MiB **per FTS index**). On first access a segment is decrypted in full and pinned in the cache; subsequent reads are memory-speed. When the cap is reached, the oldest segment plaintext is evicted — a future read on the same segment re-decrypts it.

| Access pattern | Latency |
|---|---|
| Hot segment (in cache) | memory speed — same as the prior MVP |
| Cold segment (first hit after eviction) | one decrypt round-trip, ~100 MB/s on commodity AES-GCM hardware |
| Index-time write | encrypt-on-close; KMS wrap once per new segment |

### Why this scales

KMS interactions are bounded by **file count**, not query rate:

- Segment write: 1 KMS `wrapDek` call.
- Segment first read: 1 KMS `unwrapDek` call.
- Subsequent reads of the same segment: 0 KMS calls (plaintext cached).
- Page faults inside a hot segment: 0 KMS calls.

A 1 TB index with ~10 000 segments produces ~10 000 KMS wrap calls over the lifetime of the index. That's noise compared to the per-INSERT row-encryption traffic the cluster already handles.

### Configuration

```
# Per-FTS-index plaintext cache cap. Hot working set fits in RAM, cold
# segments stay encrypted on disk and decrypt on demand.
neorunbase.fts.plaintext.cache.bytes=268435456    # 256 MiB

# AES-GCM chunk size inside each encrypted segment file. Larger chunks
# reduce per-tag overhead; smaller chunks cap the per-decrypt memory window.
neorunbase.fts.chunk.size.bytes=1048576           # 1 MiB

# Reserved for the rare case an operator wants the prior all-in-heap path.
# Default `encrypted_disk` is what production should use.
neorunbase.fts.storage.default=encrypted_disk
```

## Freshness, Flush, and Backup Safety

The encrypted-disk model removes the prior in-memory authoritative state — Lucene segments hit disk on every commit, already encrypted. There is no "dirty in-heap" window to flush.

- **Incremental writes** — every INSERT goes through `IndexWriter.addDocument`. Lucene's standard segment lifecycle picks them up; small flushes happen automatically as the in-progress segment fills.
- **Periodic commit** — `FtsFlushScheduler` issues a Lucene `commit()` at a configurable interval (`neorunbase.fts.flush.interval.seconds`, default 30 s) so a search opened by a fresh DirectoryReader picks up recently-inserted docs.
- **Commit on shard close** — the close path commits any pending writes so a graceful shutdown never loses uncommitted documents.
- **Backup safety** — encrypted segment files on disk are the canonical state. Backups copy them as-is; no re-encrypt step. A backup taken seconds after an INSERT contains that INSERT once the next commit fires (worst case 30 s).

## Observability

The same Codahale / Prometheus pipeline used elsewhere in NeorunBase. The admin endpoint `GET /admin/fts/status` (mirrors `/admin/ann/status`) returns one row per cached `(shard, index)` with size, dirty flag, pending WAL seqno, and last flushed seqno.

## Current Limitations

- **MVP is INSERT-only** — UPDATE / DELETE on FTS-indexed tables update the row store but don't tombstone in Lucene yet. Append-only workloads (logs, articles, knowledge bases) are unaffected; row-mutation-heavy schemas should wait for the tombstone follow-up.
- **Shard-local BM25** — see the "Distributed BM25" section. Adequate for evenly-distributed data; less precise on heavily skewed shards. `dfs_query_then_fetch` mode is on the roadmap.
- **No phrase positions in scoring** — `"quoted phrase"` matching works (positional postings are written) but proximity-aware scoring is left to a follow-up.

## Compatibility Notes

- **PostgreSQL FTS shape** — `tsvector` / `tsquery` types, `@@` operator, `ts_rank` family, `to_tsvector(config, text)` with a per-language `config` argument. Standard pg JDBC / psycopg / ORM clients see them as native.
- **Lucene 9.11** — analyzer + index format. The Nori artifact (`lucene-analysis-nori`) bundles `mecab-ko-dic`, so Korean tokenisation works out of the box without external dictionary install.
- **CREATE INDEX … USING fts** — internal name. Existing PostgreSQL DDL with `USING gin (col tsvector_ops)` aliases to the same FTS index path on the roadmap.
