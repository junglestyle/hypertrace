<!-- junglestyle/.github code-standards/sql.md at b3d3bb1eb1b997100738fb5bb9dd2a2b9f45a9d4. Change it there, then run make sync-standards. -->
# SQL standards

Decisions for SQL and the databases that hold it, beneath the org [standards](https://github.com/junglestyle/.github/blob/main/code-standards/README.md). In Python, [python.md](https://github.com/junglestyle/.github/blob/main/code-standards/python.md) also covers SQL: parameterized queries through `sqlite3` or psycopg 3, numbered `NNN_name.sql` migrations, and a scratch database for each test run.

## Queries

- Bind every value. The only things formatted into SQL text are identifiers the code defines, such as a table name, and placeholder lists built from a count.
- A `LIMIT` has an `ORDER BY` that decides which rows it keeps. An existence check, where any row will do, needs none.
- A write of more than one statement commits once, in one transaction.
- A non-default isolation level is set on the one transaction that needs it, not left on the connection.

## Schema

- Names are snake_case, and table names are plural nouns.
- A column that holds a closed set of names has a `CHECK` that lists them. The `CHECK` is the set's list: any other copy, such as a Python `StrEnum`, is tested against it. A repo that writes another repo's column relies on that repo's `CHECK`, not a copy of it.

## Postgres

- Every timestamp is `timestamptz`.
- A surrogate key is `GENERATED ALWAYS AS IDENTITY` or a uuid, not `serial`.
- Production, CI, tests and the dev compose file run the same Postgres major version.
- CI runs the database tests against a Postgres service container, and they don't skip there.
- Another program that reads the database connects as its own role, granted only the views and functions it uses, in a schema published for it.
- A `SECURITY DEFINER` function sets its own `search_path`.

## SQLite

- A schema that declares foreign keys turns them on: `PRAGMA foreign_keys = ON` on every connection.
- A database that more than one process opens uses WAL, and each write transaction starts with `BEGIN IMMEDIATE`.
