# SQL Correctness

A distributed engine can fail a query in two ways. It can say it cannot run the
statement — noisy, but the client knows. Or it can return something that looks
like data and is not what the statement asked for. The second kind is worse:
nothing in the response distinguishes "no rows matched" from "the predicate was
dropped and matched nothing", or a computed column from a column the engine
quietly filled with NULL.

NeorunBase treats the second kind as a defect class in its own right. The rule
the engine holds to is:

> **A statement either produces the answer PostgreSQL would produce, or it
> raises. It never produces a different answer silently.**

This page documents what that means concretely, what is enforced, and how to
verify it against your own cluster.

## What the rule rules out

Each of the behaviours below existed at some point and is now closed. They are
listed because they are the shapes this class of bug takes, and because knowing
them tells you what to check when you extend the engine.

### A clause that is parsed and then dropped

The parser accepting a clause is not the same as the executor honouring it. A
dropped clause is invisible at the wire — the query succeeds, with the wrong
rows.

| Clause | What "dropped" looked like |
| --- | --- |
| `SELECT DISTINCT` | duplicates returned, as if `DISTINCT` had not been written |
| `ORDER BY <ordinal>` | rows in arbitrary order, no error |
| `TABLESAMPLE BERNOULLI (10)` | the whole table returned as the "sample" |
| `SUM(DISTINCT x)` | the plain total — a different number |
| `UNIQUE` / `CHECK` / `DEFAULT` at `CREATE TABLE` | accepted, then never enforced or applied |
| `OFFSET` on a derived table | the wrong page |

### An expression that becomes its own SQL text

When an expression could not be compiled, the engine used to fall back to
treating the statement's text as a string literal. `WHERE name SIMILAR TO 'a%'`
became a comparison against the characters ``` `name` SIMILAR TO 'a%' ```,
which is never true, so the query answered "no rows" — indistinguishable from an
empty table. In the `SELECT` list the same fallback returned the SQL text as the
column value.

There is no such fallback now. An expression the engine cannot compile is a
parse error.

### A write that stores NULL

`UPDATE t SET cnt = cnt + 1` reduced its right-hand side with a literal
extractor, which answers null for anything that is not a literal. The statement
reported rows affected and destroyed the column it was meant to compute. The
same held for `INSERT … VALUES (20 + 5)`. `INSERT … SELECT` went further: it
parsed the source query, dropped it, and reported `INSERT 0 0` on a statement
whose purpose was to copy rows.

### A distributed merge that does not preserve meaning

Some aggregates cannot be merged by combining per-shard results of the same
shape. `COUNT(DISTINCT city)` computed a distinct count per shard and added
them, so a city present on two shards counted twice — a plausible number, larger
than the number of cities, that changes after a reshard. The partial is now the
shard's distinct **values**; the coordinator unions them and counts once.

The same reasoning governs `LIMIT` pushdown. Whenever the coordinator still has
work to do over the gathered rows — `DISTINCT`, a set operator, a window frame,
`TABLESAMPLE` — the shard-side limit is withheld, because a shard that returned
only `limit` rows would change the result rather than just speed it up.

### An unknown name resolved to NULL

`WHERE no_such_col = 1` matched nothing and `SELECT no_such_col` returned blanks.
Both are now errors. A misspelled column is a mistake in the statement, and an
empty result reads exactly like an empty table.

## What is enforced

| Area | Guarantee |
| --- | --- |
| Filters | A `WHERE` / `HAVING` clause is never dropped. If it cannot be compiled the statement is refused — this matters most for `UPDATE` and `DELETE`, where running without the predicate would rewrite or remove every row. |
| Expressions | An unknown function, operator or column raises. Nothing is answered with the statement's own text. |
| Writes | `SET col = <expression>` and `VALUES (<expression>)` compute. `INSERT … SELECT` inserts the rows the query returns. |
| Aggregates | `DISTINCT` forms merge value sets across shards. Result types follow the input: `SUM` over `INT` is an integer, not `12.0`. |
| Constraints | `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK` and `DEFAULT` are upheld on write — see [PostgreSQL Compatibility](postgresql-compatibility.md#constraints-and-defaults) for the exact semantics, including the one caveat on `UNIQUE` under concurrency. |
| Ordering | An `ORDER BY` key that cannot be resolved raises instead of returning unsorted rows. Derived tables and CTEs sort and page like any other relation. |
| Metadata | A follower coordinator that missed a metadata push re-pulls on an interval, so it cannot keep answering "table not found" for a table that exists. |

## What is refused, and why that is the answer

These constructs raise rather than returning an approximation:

- **Correlated subqueries** — a subquery referring to a column of the outer
  query. Uncorrelated ones are executed and folded into the plan; a correlated
  one would need per-row execution, and reports the column it cannot resolve.
- **Explicit window frames** (`ROWS BETWEEN …`). The default frames — running
  with an `ORDER BY` inside the window, whole-partition without — are supported.
- **`UPDATE` that computes a `UNIQUE` column from the row itself.** The value
  only exists on the data node, per row, so the constraint could not be checked;
  the statement is refused rather than run unverified.

## Verifying it on your cluster

Two suites ship with the source tree. Both run against a live cluster over
`psql` and take the coordinator's port from the environment. Point them at the
**leader** coordinator (DDL is forwarded there, and the checks create tables).

```bash
# Every statement that must be refused, and every wrong answer that must not
# come back. A FAIL line is a live defect.
PGPORT=5432 bash tests/e2e-sql-correctness.sh

# The other half: the constructs the engine does support still answer, and
# answer correctly. Guards against a refusal that goes too far.
PGPORT=5432 bash tests/e2e-sql-surface.sh
```

The suites are deliberately split. `e2e-sql-correctness.sh` asserts the rule at
the top of this page — for each construct, either the PostgreSQL answer or an
error. `e2e-sql-surface.sh` sweeps the supported surface (scalar functions,
predicates, joins, aggregation, set operators, windows, DML, vector search) and
fails on an error, which is what catches a fix that closed a hole by refusing
too much.

Both are safe to run repeatedly: they create their own tables, prefixed
`sqlc_` and `rg`, and drop them at the end.
