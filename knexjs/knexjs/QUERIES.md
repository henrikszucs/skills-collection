# knex queries

Query builder reference for knex 3.3.0. SQL in comments is PostgreSQL output
unless noted. Contents: [Select](#select) · [Where](#where) · [Joins](#joins) ·
[Aggregates](#aggregates-grouping-windows) · [Insert, update, delete](#insert-update-delete) ·
[Upsert](#upsert) · [Subqueries, CTEs, unions](#subqueries-ctes-unions) ·
[Raw SQL](#raw-sql) · [JSON](#json) · [Locking](#locking) ·
[Reuse and composition](#reuse-and-composition) · [Streaming, timeouts, batches](#streaming-timeouts-batches) ·
[Config extras](#config-extras) · [Dialect differences](#dialect-differences)

## Select

```js
knex('users')                                   // select * from "users"
knex('users as u')  /  knex({ u: 'users' })     // table alias
knex.select('id').from('users')
knex.withSchema('app').from('users')            // "app"."users"

.select('id', 'u.email as mail')                // "u"."email" as "mail"
.select({ mail: 'u.email' })                    // same, object form
.select(knex.ref('u.id').as('uid'))
.select(knex.raw('lower(??) as lower_email', ['u.email']))
.distinct('role')
.distinctOn('user_id')                          // pg only
.first('id', 'email')                           // adds limit 1, resolves to one row or undefined
.pluck('id')                                    // resolves to a flat array of one column

.orderBy('created_at', 'desc')
.orderBy([{ column: 'rank', order: 'desc', nulls: 'last' }, 'id'])
.orderByRaw('random()')
.limit(20).offset(40)
```

Dotted strings are split into identifiers, and `as` is recognised in column and
table strings. Anything else (functions, operators, casts) needs `knex.raw`.

## Where

```js
.where('id', 1)                                 // "id" = ?
.where('age', '>=', 18)
.where({ role: 'admin', active: true })         // AND of equalities
.whereNot('status', 'banned')                   // not "status" = ?
.where('deleted_at', null) / .whereNull('deleted_at')        // is null
.whereNotNull('email')
.whereIn('id', [1, 2, 3])                       // [] -> "1 = 0" (matches nothing)
.whereNotIn('id', [])                           // [] -> "1 = 1" (matches everything)
.whereIn(['a', 'b'], [[1, 2], [3, 4]])          // composite: ("a", "b") in (values ...)
.whereIn('id', knex('admins').select('user_id'))            // subquery
.whereBetween('age', [18, 65]) / .whereNotBetween(...)
.whereLike('name', 'a%')                        // case-sensitive on pg, mysql, mssql
.whereILike('name', '%ann%')                    // case-insensitive (pg ilike)
.whereExists(knex('orders').whereRaw('orders.user_id = users.id'))
.whereRaw('lower(email) = ?', [email])
.where('a.updated_at', '>', knex.ref('b.updated_at'))       // column to column
.where((q) => q.where('a', 1).orWhere('b', 2))  // parenthesised group
```

Every form has `or…` and `and…` variants (`orWhereIn`, `orWhereNull`,
`andWhereLike`…). `orWhere({ a: 1, b: 2 })` is `or (a = 1 and b = 2)`.

Gotchas, all verified against 3.3.0:

- A value of `undefined` throws; `where({ id: undefined })` too.
- `where('id', [1, 2])` does not mean IN: it compiles to broken SQL. Use `whereIn`.
- `whereIn('id', [1, null])` never matches the null. Add `.orWhereNull('id')` in a group.
- `where('name', 'similar to', pattern)` is allowed since 3.2 (PostgreSQL syntax).
- On SQLite `whereLike` and `whereILike` both compile to plain `like`, which
  is case-insensitive for ASCII.
- knex does not escape `%` and `_` in a pattern. For user input, escape them
  yourself and say so in SQL:
  `.whereRaw("?? ilike ? escape '\\'", ['name', '%' + term.replace(/[\\%_]/g, '\\$&') + '%'])`.
- Optional filters: build conditionally instead of passing `undefined`.

```js
const q = knex('users');
if (role) q.where({ role });
if (search) q.whereILike('name', `%${search}%`);
const rows = await q;
```

## Joins

```js
.join('orders', 'orders.user_id', 'users.id')              // inner join
.leftJoin('orders as o', 'o.user_id', 'users.id')
.join('orders', { 'orders.user_id': 'users.id' })
.join('orders', (j) => {
  j.on('orders.user_id', 'users.id')
   .andOn('orders.tenant_id', 'users.tenant_id')
   .andOnVal('orders.status', 'paid')                      // value, bound as ?
   .onIn('orders.kind', ['a', 'b'])
   .orOn('orders.alt_user_id', 'users.id');
})
.joinRaw('natural full join other')
.join(knex('orders').select('user_id').sum('total as t').groupBy('user_id').as('s'), 's.user_id', 'users.id')
```

Also `rightJoin`, `fullOuterJoin`, `crossJoin`, `leftOuterJoin`. Join callbacks
offer `on`, `onVal`, `onIn`, `onNotIn`, `onNull`, `onNotNull`, `onExists`,
`onBetween` and their `and`/`or` forms. After a join two tables often share
column names such as `id`: select explicit aliased columns, or the later column
overwrites the earlier one in the row object.

## Aggregates, grouping, windows

```js
.count('* as n')  .countDistinct('user_id as n')  .count({ n: '*' })
.sum('total as revenue')  .avg('age as avg_age')  .min('age as lo')  .max('age as hi')
.groupBy('role').having('n', '>', 5).havingRaw('count(*) > ?', [5])
.select('*').rowNumber('rn', 'created_at', 'user_id')   // row_number() over (partition by "user_id" order by "created_at")
.rank('r', { column: 'revenue', order: 'desc' }, 'region')   // (alias, orderBy, partitionBy); also denseRank
.rank('r', (w) => w.orderBy('revenue', 'desc').partitionBy('region'))
.rank('r', knex.raw('order by ?? desc', ['revenue']))
```

Aggregates resolve to an array of one row; chain `.first()` to get the object.
The key is the alias, so always give one (unaliased keys are `count` on
PostgreSQL but `count(*)` elsewhere). PostgreSQL returns `count` and `sum` as
strings.

Counting a paginated list: clone the filtered builder before adding limit and
order (see [Reuse and composition](#reuse-and-composition)).

## Insert, update, delete

```js
await knex('users').insert({ email, name });
await knex('users').insert([{ email: 'a' }, { email: 'b' }]);          // one multi-row statement
await knex('users').insert({ email }).returning(['id', 'created_at']); // or insert(data, ['id'])
await knex('users').where({ id }).update({ name });
await knex('users').where({ id }).update({ name }).returning('*');     // pg, mssql, sqlite
await knex('users').where({ id }).increment('logins', 1);
await knex('users').where({ id }).increment({ logins: 1 }).update({ last_seen: knex.fn.now() });
await knex('users').where({ id }).del();                               // alias: .delete()
await knex('logs').truncate();
```

- `knex.fn.now()` is `CURRENT_TIMESTAMP`; `knex.fn.uuid()` generates a UUID in SQL.
- Multi-row insert rows should share the same keys. On SQLite a missing key
  throws unless `useNullAsDefault: true`; elsewhere it becomes `DEFAULT`.
- `insert([])` rejects with "The query is empty" and `update({})` throws: guard
  empty input.
- Plain objects are `JSON.stringify`-ed for you; **arrays are not**. The pg
  driver sends a JS array as a PostgreSQL array and SQLite fails with a bind
  error, so for a JSON column always write `JSON.stringify(arr)`. Reading:
  pg and mysql2 parse JSON columns for you, SQLite hands back the string.
- Nothing updates `updated_at` automatically. Set it in the update or add a
  database trigger.
- Many rows: `knex.batchInsert('users', rows, 500)` chunks inside a transaction
  (`.transacting(trx)` and `.returning('id')` are supported).

## Upsert

```js
await knex('users').insert(row).onConflict('email').merge();              // update every inserted column
await knex('users').insert(row).onConflict('email').merge(['name']);      // update listed columns
await knex('users').insert(row).onConflict('email').merge({ name, updated_at: knex.fn.now() });
await knex('users').insert(row).onConflict(['tenant_id', 'email']).ignore();
await knex('users').insert(row).onConflict('email').merge().where('users.locked', false); // pg, sqlite only
await knex('invoices').insert({ account_id, period, total })                              // add to the stored value
  .onConflict(['account_id', 'period'])
  .merge({ total: knex.raw('?? + excluded.??', ['invoices.total', 'total']) })            // pg, sqlite; qualify the target table
  .returning('*');
```

| Dialect | `merge()` | `ignore()` | Notes |
| --- | --- | --- | --- |
| PostgreSQL, SQLite | `on conflict (cols) do update set …` | `do nothing` | columns must match a unique index or primary key |
| MySQL, MariaDB | `on duplicate key update` | `insert ignore` | column list is ignored: any unique key triggers; `insert ignore` also hides other errors |
| MSSQL | throws | throws | write a `MERGE` with `knex.raw` |

An ignored row returns `[]` with `.returning()`; without it SQLite still
resolves to a last-insert id, so don't use the result to detect a conflict.
`.upsert()` is a different method (`REPLACE INTO`, MySQL and CockroachDB only).

## Subqueries, CTEs, unions

```js
knex('users').whereIn('id', (q) => q.select('user_id').from('orders'))
knex(knex('orders').where('total', '>', 100).as('big')).select('big.*')          // subquery in FROM needs .as()
knex('users').select('*', knex('orders').count('*').whereRaw('orders.user_id = users.id').as('order_count'))

knex.with('recent', knex('orders').where('created_at', '>', since))
  .select('*').from('recent')
knex.withRecursive('tree', (q) => {
  q.select('id', 'parent_id').from('nodes').where('id', rootId)
   .unionAll((u) => u.select('n.id', 'n.parent_id').from('nodes as n').join('tree', 'tree.id', 'n.parent_id'));
}).select('*').from('tree')

knex('a').select('x').union(knex('b').select('x'))       // also unionAll, intersect, except
```

## Raw SQL

| Placeholder | Meaning |
| --- | --- |
| `?` | value binding |
| `??` | identifier (quoted, dots split): `knex.raw('??', ['users.id'])` -> `"users"."id"` |
| `:name` / `:name:` | named value / named identifier, with an object of bindings |
| `\\?` (in a JS string) | a literal `?`, e.g. the PostgreSQL jsonb `?` operator |

```js
await knex.raw('select * from users where id = ?', [id]);          // result shape differs per driver
knex('users').whereRaw('?? = any(?)', ['id', ids])                 // pg: one array binding
knex('users').select(knex.raw('count(*) filter (where active) as active_count'))
knex('t').whereRaw("meta \\? 'key' and tenant_id = ?", [tenantId])
```

- `knex.raw` works anywhere a value or column is accepted (select, where
  values, `defaultTo`, insert/update values, orderBy).
- A list in raw SQL: prefer `whereIn`. With raw, PostgreSQL takes `= any(?)`
  with the array; for other databases generate one `?` per item
  (`ids.map(() => '?').join(',')`). MySQL rejects an array binding inside
  `whereRaw`.
- `.wrap('(', ') as t')` wraps a raw fragment.

## JSON

```js
.select(knex.jsonExtract('meta', '$.address.city', 'city'))
.whereJsonPath('meta', '$.age', '>', 18)
.whereJsonObject('meta', { a: 1 })            // whole-document equality
.whereJsonSupersetOf('meta', { a: 1 })        // contains; pg (@>) and mysql only
.update({ meta: knex.jsonSet('meta', '$.seen', true) })   // also jsonInsert, jsonRemove
```

`whereJsonPath` casts by the value's JS type on PostgreSQL
(`jsonb_path_query_first(...)::int > ?`). For anything beyond these, native
operators through `whereRaw` are clearer (`meta->>'city' = ?`).

## Locking

```js
await trx('jobs').where({ status: 'queued' }).orderBy('id').forUpdate().skipLocked().first();
```

`forUpdate()`, `forShare()`, `forNoKeyUpdate()`, `forKeyShare()` (last two
PostgreSQL), plus `.skipLocked()` or `.noWait()` (PostgreSQL and MySQL 8+;
they throw on SQLite, are ignored on MSSQL). `forUpdate('table')` limits the
lock to one table of a join. Locks only hold inside a transaction.

Claiming a queue row in one statement (PostgreSQL), safe with many workers:

```js
const [job] = await knex('jobs')
  .update({ status: 'running', locked_at: knex.fn.now() })
  .where('id', knex('jobs').select('id').where({ status: 'queued' })
    .orderBy('run_after').limit(1).forUpdate().skipLocked())
  .returning('*');                                       // undefined when the queue is empty
```

## Reuse and composition

```js
const active = (q) => q.where('active', true).whereNull('deleted_at');
const paginate = (q, page, size) => q.limit(size).offset((page - 1) * size);
knex('users').modify(active).modify(paginate, 2, 20);

const base = knex('users').where({ role });
const total = await base.clone().count('* as n').first();   // clone: builders are mutable
const page = await base.clone().orderBy('id').limit(20);

base.clearSelect().clearWhere().clearOrder().clear('limit');
knex.queryBuilder()                                         // empty builder to pass around

require('knex').QueryBuilder.extend('paginate', function (page, size) {
  return this.limit(size).offset((page - 1) * size);        // adds a method to every builder
});
```

Every chained call mutates and returns the same builder. Don't store a builder
in a module-level constant and keep chaining onto it: conditions accumulate
across requests. Return a fresh one from a function instead.

## Streaming, timeouts, batches

```js
for await (const row of knex('events').select('*').stream()) { /* one row at a time */ }
await knex('big').select().timeout(5000, { cancel: true });  // cancel: pg and mysql only (throws on sqlite)
```

- PostgreSQL streaming needs the `pg-query-stream` package installed.
- `.timeout(ms)` without `cancel` only rejects the promise with
  `KnexTimeoutError`; the query keeps running on the server.
- `.toSQL().toNative()` -> `{ sql, bindings }`; `.toString()` / `.toQuery()` ->
  SQL with values inlined, for logs only (never execute it).

## Config extras

```js
const knex = require('knex')({
  client: 'pg',
  connection: { connectionString: process.env.DATABASE_URL, ssl: { rejectUnauthorized: false } },
  searchPath: ['app', 'public'],                 // pg schemas
  pool: {
    min: 0, max: 10,
    afterCreate: (conn, done) => conn.query("set time zone 'UTC'", (err) => done(err, conn)),
    validate: (conn) => true,                    // 3.2+: return false to discard a connection
  },
  // snake_case columns <-> camelCase code
  wrapIdentifier: (value, origImpl) => origImpl(value === '*' ? value : value.replace(/[A-Z]/g, (c) => '_' + c.toLowerCase())),
  postProcessResponse: (result) => /* map row keys back to camelCase */ result,
  log: { warn() {}, error: console.error, deprecate() {}, debug: console.debug },
  asyncStackTraces: process.env.NODE_ENV !== 'production',
  compileSqlOnError: false,
  defaultDateTimePrecision: 3,                   // 3.2+, pg and cockroachdb
});
```

- `postProcessResponse` also receives non-row results (counts, raw result
  objects): only transform arrays and plain row objects, e.g.
  `Array.isArray(r) ? r.map(camelKeys) : r && typeof r === 'object' ? camelKeys(r) : r`.
- `wrapIdentifier` also applies inside your migrations (columns come out
  snake_case), but generated index names use the name as written
  (`t.index(['runAfter'])` -> `jobs_runafter_index`): name indexes explicitly.
- In TypeScript the `afterCreate` parameters need explicit types (the option
  is typed `Function`).
- `connectionPool: existingPool` (3.3+) reuses a `pg.Pool`, a
  `mysql`/`mysql2` pool, an `oracledb` pool or a tarn pool instead of
  `connection`. knex will not close it: end it yourself.
- `.queryContext(obj)` on a builder is passed to `wrapIdentifier` and
  `postProcessResponse`, for per-query opt-outs.
- `knex.withUserParams({ ... })` returns a copy of the instance carrying
  `knex.userParams`.
- MySQL: use the `mysql2` driver; set `charset: 'utf8mb4'`, and
  `decimalNumbers: true` if you want DECIMAL as numbers. Client `mariadb`
  (3.3+, `mariadb` driver) adds `RETURNING` for insert and delete.
- PostgreSQL number parsing is the driver's job:
  `require('pg').types.setTypeParser(20, Number)` for `bigint` (OID 20; loses
  precision past 2^53), `1700` for `numeric`.

## Dialect differences

| Feature | PostgreSQL | MySQL | SQLite | MSSQL |
| --- | --- | --- | --- | --- |
| `.returning()` | yes | no (warning; `mariadb` client: insert/delete) | insert, update | yes (`output`) |
| `onConflict` | yes | yes (no `.where`) | yes | throws |
| `update` + `join` | join dropped: use `.updateFrom('b')` | yes | join dropped | yes |
| `del` + `join` | yes (`using`) | yes | invalid SQL | yes |
| `del` + `limit` | throws | yes (3.2.9+) | throws | throws |
| `skipLocked` / `noWait` | yes | 8.0+ | throws | ignored |
| `distinctOn` | yes | throws | throws | throws |
| `whereILike` | `ilike` | plain `like` | plain `like` | CI collation |
| `truncate` | `truncate … restart identity` | `truncate` | `delete from` | `truncate table` |
| `offset` without `limit` | yes | yes | yes | needs `orderBy` |
| identifier quoting | `"x"` | `` `x` `` | `` `x` `` | `[x]` |

Since 3.2 knex throws "Aborted query compilation" for clause combinations that
would have been silently dropped (for example `limit` on a delete). The
`update` + `join` case is still silent on PostgreSQL and SQLite; use
`updateFrom` (PostgreSQL only) or a `whereIn` subquery:

```js
await knex('orders').update({ status: knex.ref('s.status') }).updateFrom('staging as s').where('orders.id', knex.ref('s.order_id'));
await knex('orders').whereIn('user_id', knex('users').select('id').where({ banned: true })).update({ status: 'void' });
```
