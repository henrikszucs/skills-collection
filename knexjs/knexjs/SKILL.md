---
name: knexjs
description: Knex.js 3.x SQL query builder for Node (PostgreSQL, MySQL/MariaDB, SQLite, MSSQL) - connection and pool config, query builder, transactions, raw SQL, schema builder, migrations, seeds, the knex CLI and TypeScript table typing. Use whenever code imports `knex`, has a knexfile or a migrations/seeds folder, or the task is writing or debugging a knex query, upsert, transaction, migration or schema change - including "Timeout acquiring a connection", "Undefined binding(s) detected", "migration directory is corrupt" and results that come back in an unexpected shape.
---

# knex

A SQL query builder, not an ORM: it builds one SQL statement per chain and hands
back plain rows. Written against **knex 3.3.0** (current release, June 2026).

```bash
node -e "console.log(require('knex/package.json').version)"
```

When this skill and the installed version disagree, the installed source wins:
`node_modules/knex/CHANGELOG.md` and `types/index.d.ts`. 3.2.0-3.2.7 had
import/`exports` regressions that 3.2.8 reverted: use 3.2.10 or later.

## Setup

```bash
npm i knex pg          # or: mysql2 | mariadb | better-sqlite3 | sqlite3 | tedious (mssql) | oracledb
```

```js
const knex = require('knex')({
  client: 'pg',                               // pg | mysql2 | mariadb | better-sqlite3 | sqlite3 | mssql | oracledb | cockroachdb
  connection: process.env.DATABASE_URL,       // or { host, port, user, password, database, ssl }
  pool: { min: 0, max: 10 },                  // default min 2, max 10 (SQLite: 1); min 0 lets idle connections close
  acquireConnectionTimeout: 10000,            // default 60000
});
```

- Create **one** instance per process and export it. Call `await knex.destroy()`
  at shutdown and at the end of scripts and test runs, or Node will not exit.
- Native ESM: `import knex from 'knex'`. `import { knex } from 'knex'` throws
  "Named export 'knex' not found" under plain Node ESM (it only works through
  TypeScript-to-CJS or a bundler).
- SQLite: `{ client: 'better-sqlite3', connection: { filename: './dev.sqlite3' }, useNullAsDefault: true }`
  (`':memory:'` for tests). `better-sqlite3` enforces foreign keys; `sqlite3`
  does not until you add
  `pool: { afterCreate: (conn, done) => conn.run('PRAGMA foreign_keys = ON', done) }`.
- Relative paths (`filename`, migration `directory`) resolve against
  `process.cwd()` in your app, but the CLI first changes into the knexfile's
  folder. Make them absolute (`path.join(__dirname, …)` / `import.meta.dirname`).
- `connection` may be an async function returning the config (plus an optional
  `expirationChecker`) for rotating credentials.
- More config (snake_case mapping, logging, external pools): [QUERIES.md](QUERIES.md#config-extras).

## Rules that prevent silent failures

1. **A builder does nothing until awaited.** It is a lazy thenable: `await`,
   `.then()` or `.stream()` runs it, and every `await` of the same builder runs
   it again. A chain with no `await` (common in migrations and `forEach`) is
   never sent.
2. **Inside a transaction every query goes through `trx`.** A query on `knex`
   uses a different connection: it is not rolled back, cannot see the
   transaction's writes, and can deadlock until "Timeout acquiring a
   connection. The pool is probably full".
3. **`undefined` in a `where` throws** ("Undefined binding(s) detected"), by
   design, so a missing variable can't widen a query. Validate input first.
   `undefined` values in `insert`/`update` data are just skipped.
4. **`orWhere` is not grouped for you.** `.where(a).where(b).orWhere(c)` is
   `a AND b OR c`. Group with a callback:
   `.where(a).andWhere((q) => q.where(b).orWhere(c))`.
5. **A join's last argument is a column, not a value.**
   `.join('b', 'a.id', '=', 'x')` compares to a column named `x`. For a value
   use `.join('b', (j) => j.on('a.id', 'b.a_id').andOnVal('b.kind', 'x'))`.
6. **Never build SQL with template strings.** Values go in `?`, identifiers in
   `??` (or `:name` / `:name:`): `knex.raw('?? = ?', ['users.id', id])`.
7. **Check what the dialect returns** (next table). `.returning()` on MySQL only
   logs a warning and is ignored.
8. **Some clauses are dropped or rejected per dialect.** `update` with `.join()`
   compiles on PostgreSQL and SQLite with the join silently removed; `del` with
   `.limit()` throws everywhere except MySQL. See [QUERIES.md](QUERIES.md#dialect-differences).
9. **SQLite: altering a parent table inside a transaction deletes its child
   rows.** knex rebuilds the table, and with foreign keys on, `ON DELETE
   CASCADE` fires. Migrations run in a transaction by default. See
   [SCHEMA.md](SCHEMA.md#altering-tables) before altering or dropping columns
   on SQLite.

When unsure what a chain produces, print it: `query.toSQL().toNative()` gives
`{ sql, bindings }` without running anything, and it needs no database
connection (`require('knex')({ client: 'pg' })` is enough).

## What a query resolves to

| Call | PostgreSQL | MySQL / MariaDB | SQLite |
| --- | --- | --- | --- |
| `select` | rows `[]` | rows | rows |
| `.first()` | row or `undefined` | same | same |
| `.pluck('id')` | `[1, 2]` | same | same |
| `insert(...)` | driver result object: use `.returning()` | `[insertId]` (first row's id) | `[lastId]` |
| `insert(...).returning('id')` | `[{ id: 1 }]` | ignored (works with client `mariadb`, server 10.5+) | `[{ id: 1 }]` |
| `update` / `del` | affected count | affected count | affected count |
| `update(...).returning('*')` | rows | ignored | rows (`del` returns a count) |
| `.count()` | `[{ count: '5' }]` (string) | ``[{ 'count(*)': 5 }]`` | ``[{ 'count(*)': 5 }]`` |
| `knex.raw(sql)` | `{ rows, rowCount, ... }` | `[rows, fields]` | rows |

- `.returning()` always yields an array of objects, even for a single column,
  and `[]` when no row matched. After `onConflict().merge()` it returns the
  row on both the insert and the update path.
- Count safely: `const { n } = await knex('users').count('* as n').first(); Number(n)`.
- PostgreSQL returns `bigint` and `numeric` as strings. MSSQL uses
  `output inserted.*`, so `.returning()` works there too.
- SQLite returns booleans as `0`/`1` and JSON columns as text. It stores a JS
  `Date` as epoch milliseconds but `knex.fn.now()` / column defaults as text
  (`2026-11-26 08:48:54`), so one column can hold both: pick one (for example
  always write `new Date().toISOString()`).

## Queries in brief

```js
const user = await knex('users').where({ id }).first();                 // undefined if none
const rows = await knex('users as u')
  .leftJoin('orders as o', 'o.user_id', 'u.id')
  .select('u.id', 'u.email', { total: knex.raw('coalesce(sum(o.total), 0)') })
  .where('u.active', true)
  .whereIn('u.role', roles)                                             // [] compiles to 1 = 0, no error
  .groupBy('u.id')
  .orderBy([{ column: 'u.created_at', order: 'desc' }])
  .limit(20).offset(40);

const [{ id: newId }] = await knex('users').insert({ email }).returning('id'); // pg, sqlite, mssql
await knex('users').where({ id }).update({ name, updated_at: knex.fn.now() });
await knex('users').where({ id }).del();

await knex('users')                                                     // upsert: pg, mysql, sqlite
  .insert({ email, name })
  .onConflict('email')                                                  // needs a unique index on email
  .merge(['name']);                                                     // or .merge() for all columns, .ignore()
```

`where('col', null)` is `IS NULL`; `where('col', '=', null)` is the useless
`= NULL`. An `update` or `del` without a `where` hits every row.
Full reference: [QUERIES.md](QUERIES.md).

## Transactions

```js
const orderId = await knex.transaction(async (trx) => {
  const [{ id }] = await trx('orders').insert({ user_id }).returning('id');
  await trx('order_items').insert(items.map((i) => ({ ...i, order_id: id })));
  await trx('stock').where({ sku }).forUpdate().first();                // row locks need a transaction
  return id;                                                            // resolve = commit, throw = rollback
}, { isolationLevel: 'repeatable read' });                              // optional; also readOnly: true
```

- Pass `trx` to helpers (`function createOrder(data, db = knex)`), or bind an
  existing builder with `.transacting(trx)`.
- Await queries one after another. A transaction owns a single connection, so
  `Promise.all` gains nothing: the queries still run serially, and when one
  fails the others are still queued behind it.
- Don't call `trx.commit()` / `trx.rollback()` in an async handler: return or
  throw. Manual mode is `const trx = await knex.transaction()` followed by
  `try { ...; await trx.commit() } catch (e) { await trx.rollback(); throw e }`.
  A transaction that is never finished holds its connection forever.
- `trx.transaction(...)` nests with a savepoint. `isolationLevel` is ignored on
  SQLite. Valid levels: `read uncommitted`, `read committed`, `snapshot`,
  `repeatable read`, `serializable`.

## TypeScript

```ts
import knex, { type Knex } from 'knex';        // default import: works in CJS (esModuleInterop) and native ESM
                                               // `import { knex }` type-checks under nodenext but throws at runtime
type Writable<T> = { [K in keyof T]?: T[K] | Knex.Raw };

declare module 'knex/types/tables.js' {        // the .js suffix is required under moduleResolution nodenext
  interface Tables {
    posts: Post;                               // plain row type: the simple default
    users: Knex.CompositeTableType<            // <Select, Insert, Update> when they differ
      User, Pick<User, 'email'> & Writable<User>, Writable<Omit<User, 'id'>>>;
  }
}

const db: Knex = knex(config satisfies Knex.Config);
const u = await db('users').where('id', 1).first();           // User | undefined
const n = await db('users').count<{ n: string }[]>('* as n');  // aggregates need an explicit type
await db.transaction(async (trx: Knex.Transaction) => { /* ... */ });
```

- Composite insert/update types are taken literally, so without `| Knex.Raw`
  (the `Writable` helper) `update({ updated_at: db.fn.now() })` is a type
  error. Plain row types accept raw values already.
- Only insert/update payloads and object-style `where({ … })` are checked.
  String column names (`where('nope', 1)`, `select`, `orderBy`, `returning`)
  are not, and an aliased table (`'users as u'`) loses its typing.
- Untyped tables fall back to `any`: `db<User>('users')` types a single query;
  `db.batchInsert<Tables['posts']>('posts', rows)` needs the type spelled out.
- Types describe the builder, not the driver: a `bigint` column typed `number`
  still arrives as a string on PostgreSQL.

## Debugging

- `DEBUG=knex:query,knex:bindings,knex:tx node app.js`, or `debug: true` in the config.
- `knex.on('query' | 'query-response' | 'query-error', fn)` for logging and timing.
- `asyncStackTraces: true` (dev only) makes errors point at the calling line.
- Error messages contain the SQL with values inlined. Set
  `compileSqlOnError: false` to keep values out of logs.
- "Timeout acquiring a connection": a query on `knex` inside a transaction, a
  transaction that was never committed or rolled back, a pool too small for
  the load, or an unreachable database.

## References

- [QUERIES.md](QUERIES.md): every where/join/aggregate form, upserts, CTEs, JSON,
  locking, raw bindings, streaming, config extras and per-dialect differences.
  Read it before writing a non-trivial query.
- [SCHEMA.md](SCHEMA.md): column types and what they compile to, constraints,
  indexes, altering tables. Read it before writing `knex.schema` code.
- [MIGRATIONS.md](MIGRATIONS.md): knexfile, CLI, migration and seed rules,
  ESM/TypeScript setups, programmatic API. Read it before creating or running
  migrations.
- Upstream docs: https://knexjs.org/guide/ · https://github.com/knex/knex
