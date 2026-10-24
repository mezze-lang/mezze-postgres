# mezze-postgres

PostgreSQL for Mezze: a `Db` performer, and nothing else.

```mezze
use mezze_postgres::postgres::{ Postgres, pg_config, pg_url }
use std::effects::db::{ Db, DbStr, sql, BindParams, value_at }

let main = {} ->
  perform Postgres { config = pg_config {
    url = pg_url { host = "127.0.0.1", port = 5432, database = "app",
                   user = "svc", password = "" },
  } } in
    match Db.run { q = sql { text = "select name from users where id = :id" }
                     .bind { params = { id = 7 } } } is
      Ok { val = rows } -> value_at { rows = rows, row = 0, col = 0 }
      Err { error = _ } -> None
```

```toml
[dependencies]
mezze_postgres = "github:<owner>/mezze-postgres@v0.1.0"
```

Then `mez sync`. That fetches this package and, through its `[maven]` block,
the PgJDBC driver.

## What this package is, and is not

`std::effects::db` in the standard library owns the **effect** — `Query`,
`Rows`, `DbError`, the parameter and row codecs, the mock performers. All of
that is database-independent and belongs with the language.

This package owns the part that knows about one product: a URL, a pool, and a
performer. That is all it is. Everything you actually write — building queries,
binding parameters by name, decoding rows into records, testing a route with
`MockDb` and no database at all — is stdlib and works the same here as against
SQLite.

SQLite is the one driver that stays in the standard library, for the reason
Python keeps `sqlite3` in its own: an embedded database with no install step is
the point, and it ships as platform natives rather than a service to run.

## It ships no natives, and could not

Nothing here is privileged. Six of the runtime's seven `Db` natives never knew
which database they addressed — query, exec, begin, commit, rollback, close —
and `__native_db_open` takes any JDBC URL. So a driver is ordinary Mezze over a
maven coordinate, which is what makes a third-party one possible at all.

If you want to write a driver for another database, copy `src/postgres.mz`. It
is under 200 lines and most of them are comments.

## Loom

PgJDBC is pure Java and blocks on a **socket**, so a virtual thread waiting for
a query unmounts and its carrier goes back to work. That is the difference from
the bundled SQLite driver, which is JNI: a native frame cannot unmount, so it
pins its carrier for the length of a call.

Pooling matters more here too. Postgres has real concurrent writers, where
SQLite has exactly one however many connections you open.

`max_connections` is a **shared** resource: the server has a limit of its own
(100 by default, some reserved), so every instance of your service spends from
one budget. Size it at the number of cores doing work, not hopefully — a
connection waiting on a lock is not faster for having company.

## Session settings go in `init_sql`

```mezze
perform Postgres { config = {
  url = …,
  max_connections = 8,
  init_sql = ["set search_path to app, public", "set statement_timeout to '5s'"],
} } in …
```

These run on **every** connection the pool opens. That is the right place for
them, because a pooled connection is not the one you set a session variable on
last time, and setting them per query would be two round trips instead of one.

## Known rough edges

These are honest limitations, not oversights:

- **A `uuid`, `timestamptz` or `jsonb` column arrives as text.** `DbValue` has
  cases for integers, floats, strings, booleans and bytes; anything else falls
  through to its string form. Usable, but you parse it yourself.
- **`NUMERIC` arrives as `DbFloat`, which is lossy.** Postgres `NUMERIC` is
  exact and a double is not. For money, bind and read an `Int` of minor units,
  or read the column as text with `::text` and parse it.
- **Placeholder names are checked when the statement runs, not when it
  compiles.** A `:nmae` against a record holding `name` is a `Rejected` error
  that names the placeholder — caught, but at run time.
- **Work forked inside a transaction gets a different connection** and so does
  not see that transaction's uncommitted writes. Keep a transaction's work on
  its own thread.
- **Cancellation does not reach the server.** A cancelled Mezze task leaves its
  query running to completion; `Statement.cancel` is not wired up.

## Running the tests

The example is the test, and it needs a server:

```sh
nix-shell -p postgresql --run "
  initdb -D /tmp/mzpg -A trust -U mezze
  pg_ctl -D /tmp/mzpg -o \"-p 55432 -k '' -c listen_addresses=127.0.0.1\" -l /tmp/mzpg/log start
  createdb -h 127.0.0.1 -p 55432 -U mezze mezzetest"

cd examples/roundtrip && mez sync && mez dev --once
```

Expected:

```
select answered: from-the-package
pg_url builds: jdbc:postgresql://h:5432/d?user=u&password=p%26p%3Dx
```
