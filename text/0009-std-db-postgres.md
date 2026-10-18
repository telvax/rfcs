# RFC: a SQL client for PostgreSQL in std.db

* Status: accepted
* Pull request: #9
* Issues: telvax/telvax#134
* Implementation: (none yet)

## Summary

`std.db.postgres.PostgresClient` runs SQL on a PostgreSQL server: `execute` for statements and `query` for
rows, which come back as a list of the program's own `type`. `std.db` has only `pgvector`, which stores vectors.

## Motivation

[sentiment-analysis-api](https://github.com/thentsation/sentiment-analysis-api) keeps jobs and results in
ordinary tables (id, status, a JSON column). A Telvax service that needs a table has nothing to use.

## Design

```telvax
import std.db.postgres.PostgresClient

type Job:
    id: Text
    status: Text
    attempts: Int
    result: Text?

let db = PostgresClient.connect(std.env.get("DATABASE_URL").unwrap())

fn finish(id: Text, result: Text) -> Result[Int, Text]:
    db.execute("UPDATE jobs SET status = $1, result = $2 WHERE id = $3", ("done", result, id))

fn pending() -> Result[List[Job], Text]:
    let rows: List[Job] = db.query("SELECT id, status, attempts, result FROM jobs WHERE status = $1", ["queued"])?
    Ok(rows)
```

* `PostgresClient.connect(url: Text)`. The connection is opened by the first statement, not by `connect`, so a
  program can start before the database does.
* `execute(sql: Text, params) -> Result[Int, Text]`: the number of rows changed.
* `query(sql: Text, params) -> Result[List[Row], Text]`: `Row` is a `type` of the program, taken from the
  expected type as `std.json.parse` does. Each field is read from the column of the same name; a missing
  column is an error that names it. Extra columns are ignored.
* `params` is a list (`["queued"]`) when all are of one type, or a tuple (`("done", 3, id)`) when they are not.
  They are bound as parameters, never written into the SQL, so there is no injection through them. `$1`, `$2`
  are the places. A `None` is `NULL`.
* Types. `Text`, `Int`, `Float` and `Bool` map to `text`, `int2/4/8`, `float4/8` and `bool` (the value is
  converted to the type the server expects for that place, so an `Int` can fill an `int4`). A `T?` field is
  read from a column that may be `NULL`; `NULL` in a field that is not `T?` is an error. `json` and `jsonb`
  are read and written as `Text` holding JSON. Anything else (`timestamptz`, `numeric`, `uuid`, arrays) is
  an error that names the column and says to cast: `write created_at::text`.
* The statement runs on one connection shared by the program, one at a time. If the connection was lost it
  is opened again once and the statement retried only when it had not reached the server.
* It needs `net` and `--live`. There is no offline stand-in: without `--live` the first call is an error that
  says so, as for the other services that are not simulated.
* Errors are `Err(Text)`: the message of the server (`duplicate key value violates unique constraint ...`).

### Where it is built

`telvax_runtime` (feature `live`), on the `postgres` crate that `pgvector` already uses, with its `serde_json`
support for the JSON columns. A test runs against a real server when `TELVAX_TEST_PG_URL` is set, and the
`pgvector` job of the integration workflow sets it.

## Alternatives

* **A table type** (`Table[Row]` over a connection). It would unify tables in memory and in PostgreSQL, but the
  query model is different; the SQL string is what a service already has.
* **Row as a `Map[Text, ...]`.** There is no type for a value that may be of several types, and a typed row
  is checked.
* **A connection pool and transactions.** Needed by a busy service, but each is a design of its own
  (`db.transaction(fn)`), and one connection is enough to start.

## Open questions

* Transactions (`db.transaction(tx => ...)`) and a pool.
* Dates and timestamps as typed values, when the language has them as more than `Text`.
