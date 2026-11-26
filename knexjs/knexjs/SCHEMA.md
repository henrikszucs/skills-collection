# knex schema builder

`knex.schema` reference for knex 3.3.0. Contents: [Tables](#tables) ·
[Column types](#column-types) · [Column modifiers](#column-modifiers) ·
[Keys, indexes, constraints](#keys-indexes-constraints) ·
[Altering tables](#altering-tables) · [Views, schemas, extensions](#views-schemas-extensions) ·
[Gotchas](#gotchas)

Preview any schema call without running it: `knex.schema.createTable(...).toSQL()`
returns the list of statements (works with no connection, except SQLite
alterations, which need to read the existing table).

## Tables

```js
await knex.schema.createTable('orders', (t) => {
  t.increments('id');                                             // auto-increment primary key
  t.uuid('public_id').notNullable().defaultTo(knex.fn.uuid()).unique();
  t.integer('user_id').unsigned().notNullable()
    .references('id').inTable('users').onDelete('CASCADE').index();
  t.string('status', 20).notNullable().defaultTo('new');
  t.decimal('total', 12, 2).notNullable();
  t.jsonb('meta');
  t.timestamps(true, true);                                       // created_at, updated_at: not null default now
  t.unique(['user_id', 'status']);
  t.check('?? >= 0', ['total'], 'orders_total_nonneg');
});

await knex.schema.dropTableIfExists('orders');
await knex.schema.renameTable('orders', 'purchases');
if (!(await knex.schema.hasTable('orders'))) { /* ... */ }
await knex.schema.hasColumn('orders', 'status');                  // boolean
await knex.schema.withSchema('app').createTable('t', (t) => { /* "app"."t" */ });
await knex.schema.createTableLike('orders_archive', 'orders');
```

Calls chained on `knex.schema` run in order as one awaited unit:
`knex.schema.dropTableIfExists('b').dropTableIfExists('a')`.
Don't use `createTableIfNotExists` (it logs a deprecation warning and breaks
when the table needs follow-up `alter` statements): check `hasTable` first.

## Column types

| Method | PostgreSQL | MySQL | SQLite |
| --- | --- | --- | --- |
| `increments('id')` | `serial primary key` | `int unsigned not null auto_increment primary key` | `integer not null primary key autoincrement` |
| `bigIncrements('id')` | `bigserial primary key` | `bigint unsigned … primary key` | `integer … autoincrement` |
| `integer` / `bigInteger` / `smallint` / `tinyint` | `integer` / `bigint` / `smallint` / `smallint` | `int` / `bigint` / `smallint` / `tinyint` | `integer` / `bigint` / `integer` / `tinyint` |
| `string('s')` / `string('s', 50)` | `varchar(255)` / `varchar(50)` | same | same |
| `text('t')` / `text('t', 'longtext')` | `text` | `text` / `longtext` | `text` |
| `boolean` | `boolean` | `boolean` (tinyint(1)) | `boolean` (0/1) |
| `decimal('d')` / `decimal('d', 12, 2)` | `decimal(8, 2)` / `decimal(12, 2)` | same | `float` |
| `float` / `double` | `real` / `double precision` | `float(8, 2)` / `double` | `float` |
| `date` / `time` | `date` / `time` | same | same |
| `timestamp('at')` | `timestamptz` | `timestamp` | `datetime` |
| `datetime('at')` | `timestamptz` | `datetime` | `datetime` |
| `timestamp('at', { useTz: false, precision: 3 })` | `timestamp(3)` | `timestamp(3)` | `datetime` |
| `timestamps()` | two nullable `timestamptz`, no default | two nullable `datetime` | two nullable `datetime` |
| `timestamps(true, true)` | `timestamptz not null default CURRENT_TIMESTAMP` | `timestamp not null default CURRENT_TIMESTAMP` | `datetime not null default CURRENT_TIMESTAMP` |
| `json` / `jsonb` | `json` / `jsonb` | `json` | `json` (stored as text) |
| `uuid('u')` | `uuid` | `char(36)` (`{ useBinaryUuid: true }` -> `binary(16)`) | `char(36)` |
| `enu('k', ['a', 'b'])` | `text check (k in (…))` | `enum('a', 'b')` | `text check (…)` |
| `binary('b')` | `bytea` | `blob` | `blob` |
| `specificType('tags', 'text[]')` | emitted verbatim | verbatim | verbatim |

- `timestamps(true, true)` arguments are `(useTimestamps, defaultToNow)`. The
  object form adds camelCase names:
  `t.timestamps({ useTimestamps: true, defaultToNow: true, useCamelCase: true })`.
- `increments(name, { primaryKey: false })` and `uuid(name, { primaryKey: true })`
  control the inline primary key.
- Native PostgreSQL enum: `t.enu('k', ['a', 'b'], { useNative: true, enumName: 'kind_t' })`
  (creates the type first; `existingType: true` reuses one). Changing the
  allowed values later needs raw SQL either way.
- `geometry`, `geography`, `point` exist; anything else goes through
  `specificType`, including PostgreSQL identity columns:
  `t.specificType('id', 'bigint generated always as identity').primary()`.
- MSSQL: `string` is `nvarchar`, `json`/`text` are `nvarchar(max)`, `boolean`
  is `bit`, timestamps are `datetime2`, defaults become named constraints.

## Column modifiers

```js
t.string('email').notNullable()              // columns are nullable unless you say otherwise
t.integer('n').defaultTo(0)
t.timestamp('at').defaultTo(knex.fn.now())   // knex.fn.now(6) for precision
t.timestamp('exp').defaultTo(knex.raw("now() + interval '1 day'"))
t.jsonb('meta').defaultTo('{}')              // JSON defaults are strings
t.integer('user_id').unsigned()              // MySQL only; ignored on pg and sqlite
t.string('email').unique()                   // or { indexName: 'users_email_uq' }
t.string('email').index()
t.string('code').primary()
t.string('a').comment('shown in the catalog')
t.string('a').collate('utf8mb4_unicode_ci')  // mysql
t.integer('pos').after('name') / .first()    // mysql column order
t.integer('age').checkPositive().checkBetween([0, 150])
t.string('s').checkIn(['a', 'b']).checkLength('<=', 5).checkRegex('^[a-z]+$')
```

## Keys, indexes, constraints

```js
t.primary(['tenant_id', 'id']);                                   // composite
t.primary(['tenant_id', 'id'], 'pk_orders');                      // named (the { constraintName } form breaks on mysql)
t.integer('org_id').references('orgs.id');                        // shorthand for references('id').inTable('orgs')
t.foreign('org_id').references('id').inTable('orgs').onDelete('SET NULL').onUpdate('CASCADE');
t.foreign(['a', 'b'], 'fk_ab').references(['x', 'y']).inTable('other');
t.integer('org_id').references('orgs.id').withKeyName('fk_users_org').deferrable('deferred'); // pg

t.index(['status', 'created_at']);                                // name: orders_status_created_at_index
t.index(['meta'], 'orders_meta_gin', 'gin');                      // index type (pg)
t.index('title', null, { indexType: 'FULLTEXT' });                // mysql
t.index(['email'], 'users_email_live', { predicate: knex.whereNull('deleted_at') });  // partial: pg, sqlite, mssql
t.unique(['email'], { indexName: 'users_email_live_uq', predicate: knex.whereNull('deleted_at') });
t.unique(['a'], { deferrable: 'deferred' });                      // pg
```

A `predicate` must not contain bound values: `knex.where('status', 'queued')`
emits `where "status" = ?` with no binding, which is broken DDL and only fails
when it runs. Use `whereNull` / `whereNotNull`, or inline the literal:
`predicate: knex.whereRaw("?? = 'queued'", ['status'])`.

Default names are `<table>_<cols>_index`, `_unique`, `_foreign` and
`<table>_pkey`. The `drop*` methods rebuild the name from the columns, so pass
the same columns, or the explicit name as the second argument:

```js
t.dropIndex(['status', 'created_at']);   t.dropIndex([], 'orders_meta_gin');
t.dropUnique(['email']);                 t.dropUniqueIfExists(['email']);    // IfExists: 3.2+, not mysql
t.dropForeign('org_id');                 t.dropForeign('org_id', 'fk_users_org');
t.dropPrimary();
```

## Altering tables

```js
await knex.schema.alterTable('users', (t) => {     // .table() is an alias
  t.string('nick');                                // add column
  t.string('name', 500).notNullable().alter();     // change type / nullability / default
  t.renameColumn('fullname', 'name');
  t.dropColumn('legacy');   t.dropColumns('a', 'b');
  t.setNullable('bio');                            // drop not null
  t.dropNullable('email');                         // set not null
  t.dropTimestamps();
});
```

- **`.alter()` redefines the whole column.** Anything you leave out is removed:
  on PostgreSQL it first drops the default and `not null`, then sets the type,
  then re-adds only what the new definition states. Repeat `notNullable()` and
  `defaultTo()` every time. PostgreSQL only: `.alter({ alterNullable: false })`
  leaves nullability alone and `.alter({ alterType: false })` leaves the type
  alone (both default to `true`; the default is always dropped and re-set).
  To change only a default, use raw SQL with the literal inlined (DDL cannot
  take `?` value bindings; `??` identifiers are fine):
  `knex.raw("alter table ?? alter column ?? set default 'free'", ['accounts', 'plan'])`.
- `.alter()` on an `increments` or non-native `enu` column emits invalid SQL on
  PostgreSQL (`type serial primary key`, `type text check (…)`). Change those
  with raw SQL: drop and re-add the check constraint, or alter the sequence.
- Adding a `notNullable()` column to a table with rows needs a `defaultTo`, or
  three steps: add nullable, backfill, `dropNullable`.
- On PostgreSQL a type change is emitted as `type integer using ("col"::integer)`.
  When a plain cast is not enough (text to timestamp with a format, text to
  enum), write the `alter table … using …` yourself with `knex.raw`.
- SQLite cannot alter or drop most things in place, so `.alter()`,
  `dropColumn`, `setNullable`/`dropNullable`, `foreign`, `dropForeign` and
  `dropPrimary` make knex **rebuild the table** (create copy, move rows, drop
  the old one, rename). Columns, defaults, checks, foreign keys and indexes
  carry over, and several changes can share one `alterTable`.
- **The rebuild deletes child rows when it runs inside a transaction.** With
  foreign keys enforced, dropping the old table fires `ON DELETE CASCADE` on
  every table that references it, silently. Outside a transaction knex
  switches `foreign_keys` off for the rebuild; inside one SQLite ignores that
  pragma. Migrations are transactional by default, so a migration that
  rebuilds a referenced table needs `exports.config = { transaction: false }`
  (verified on 3.3.0 with both SQLite drivers). Adding a column or an index
  does not rebuild and is safe.

## Views, schemas, extensions

```js
await knex.schema.createView('active_users', (v) => {
  v.columns(['id', 'email']);
  v.as(knex('users').select('id', 'email').where('active', true));
});
await knex.schema.createViewOrReplace(...);  await knex.schema.dropViewIfExists('active_users');
await knex.schema.createMaterializedView('stats', (v) => v.as(query));   // pg
await knex.schema.refreshMaterializedView('stats');
await knex.schema.createSchemaIfNotExists('app');                        // pg only
await knex.schema.createExtensionIfNotExists('pgcrypto');                // pg
await knex.schema.raw('create index concurrently …');                    // anything else
```

Triggers, functions, generated columns, exclusion constraints and enum changes
have no builder: use `knex.raw` / `knex.schema.raw`.

## Gotchas

- **MySQL foreign keys must match types exactly.** `increments()` is
  `int unsigned`, so the referencing column needs `.integer('x').unsigned()`
  (`bigInteger().unsigned()` for `bigIncrements`), or MySQL fails with errno
  150 / "incompatible".
- **PostgreSQL does not index foreign key columns.** Add `.index()` on them.
- `knex.fn.uuid()` compiles to `gen_random_uuid()` (PostgreSQL 13+, or the
  `pgcrypto` extension before that) and `UUID()` on MySQL (default
  expressions need MySQL 8.0.13+).
- Literal defaults are emitted quoted (`defaultTo(false)` and `defaultTo(0)`
  both become `default '0'`), which every supported database accepts.
- `timestamp()` is `timestamptz` on PostgreSQL but a timezone-less type
  elsewhere. Keep the connection in UTC.
- MSSQL `unique()` creates a filtered unique index (`where col is not null`),
  not a unique constraint, so multiple NULLs are allowed.
- MySQL ignores `predicate` (no partial indexes) and has no transactional DDL:
  a migration that fails halfway leaves the earlier statements applied.
- Table-level `engine()`, `charset()` and `collate()` are MySQL-only and throw
  on other dialects. `comment()` works on PostgreSQL, MySQL and MSSQL.
