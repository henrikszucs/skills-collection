# knex migrations, seeds and CLI

For knex 3.3.0. Contents: [knexfile](#knexfile) · [CLI](#cli) ·
[Writing migrations](#writing-migrations) · [Rules](#rules) ·
[ESM and TypeScript](#esm-and-typescript) · [Seeds](#seeds) ·
[Programmatic API](#programmatic-api) · [Errors](#errors)

## knexfile

`knexfile.js` in the project root, keyed by environment. The CLI picks
`--env`, then `NODE_ENV`, then `development`.

```js
/** @type {Object.<string, import('knex').Knex.Config>} */
module.exports = {
  development: {
    client: 'pg',
    connection: process.env.DATABASE_URL,
    migrations: { directory: './db/migrations', tableName: 'knex_migrations' },
    seeds: { directory: './db/seeds' },
  },
  production: { /* ... */ },
};
```

Also valid: a single config object without environment keys, or an (async)
function returning either shape. Build the app's instance from the same file so
both agree: `require('knex')(require('./knexfile')[process.env.NODE_ENV || 'development'])`.
The CLI changes into the knexfile's folder before resolving relative paths;
your app resolves them against `process.cwd()`. Use absolute paths
(`path.join(__dirname, 'db/migrations')`) so both find the same files.

`migrations` options: `directory` (string or array, default `./migrations`),
`tableName` (`knex_migrations`), `schemaName`, `extension` (for generated files,
default `js`), `loadExtensions` (which files count as migrations), `stub` (your
own template), `disableTransactions`, `disableMigrationsListValidation`,
`sortDirsSeparately`, `migrationSource` (custom loader, e.g. for bundled apps).

## CLI

```bash
npx knex init                         # write a knexfile (-x ts for TypeScript)
npx knex migrate:make add_orders      # creates <timestamp>_add_orders.js (-x ts | mjs)
npx knex migrate:latest               # run every pending migration as one batch
npx knex migrate:rollback             # undo the last batch (--all: everything)
npx knex migrate:up [name]            # run the next (or the named) pending migration
npx knex migrate:down [name]          # undo the last (or the named) completed migration
npx knex migrate:to <name>            # 3.3+: run pending migrations up to and including <name>
npx knex migrate:before <name>        # 3.3+: run pending migrations before <name>
npx knex migrate:list                 # completed and pending (alias migrate:status)
npx knex migrate:currentVersion
npx knex migrate:unlock               # clear a stale migration lock
npx knex seed:make 01_users
npx knex seed:run [--specific=01_users.js]
```

Global flags: `--env production`, `--knexfile path`, `--cwd dir`, `--debug`,
`--migrations-directory`, `--client` and `--connection` (run without a
knexfile). `<name>` is the full file name including the extension.

## Writing migrations

```js
/** @param {import('knex').Knex} knex */
exports.up = async (knex) => {
  await knex.schema.createTable('orders', (t) => {
    t.increments('id');
    t.integer('user_id').unsigned().notNullable().references('users.id').onDelete('CASCADE').index();
    t.string('status', 20).notNullable().defaultTo('new');
    t.timestamps(true, true);
  });
  await knex('orders').insert({ user_id: 1 });        // data changes are fine too
};

exports.down = async (knex) => {
  await knex.schema.dropTable('orders');
};
```

Schema methods: [SCHEMA.md](SCHEMA.md).

A statement that cannot run in a transaction (PostgreSQL
`create index concurrently`, `alter type … add value` on old versions) opts out
per file:

```js
exports.up = (knex) => knex.raw('create index concurrently if not exists orders_status_idx on orders (status)');
exports.down = (knex) => knex.raw('drop index concurrently if exists orders_status_idx');
exports.config = { transaction: false };     // ESM / TypeScript: export const config = { transaction: false };
```

## Rules

1. **Await or return every call.** An un-awaited `knex.schema…` in `up` either
   never runs or runs after the migration is recorded as done.
2. **Use the `knex` argument**, not your app's instance. It is the migration's
   transaction; another instance runs outside it and can block on its locks.
3. **Never edit, rename or delete a migration that has run anywhere.** knex
   compares the files on disk with the `knex_migrations` table by file name
   (extension included) and refuses to run when a recorded file is missing.
   Fix forward with a new migration.
4. **`down` undoes `up` in reverse order** (drop the child table before the
   parent). Throw in `down` if the change is irreversible instead of leaving
   it empty.
5. **Don't import application code** (models, constants) into a migration: the
   migration must keep meaning the same thing after the app changes. Inline
   the values.
6. **One batch = one `migrate:latest` run.** By default all pending migrations
   share one transaction, so a failure rolls the whole batch back - but only
   where DDL is transactional (PostgreSQL, SQLite, MSSQL). MySQL commits each
   DDL statement implicitly, so keep MySQL migrations small. If any pending
   migration sets `config.transaction = false`, the batch is not wrapped and
   each other migration gets its own transaction.
7. File names sort the run order. Keep the generated timestamp prefix; a file
   added later with an older prefix than completed ones still runs, out of
   order.
8. **SQLite:** a migration that alters or drops columns on a table other
   tables reference must set `exports.config = { transaction: false }`, or the
   table rebuild cascade-deletes the child rows
   ([SCHEMA.md](SCHEMA.md#altering-tables)).
9. Concurrent deploys are safe: a row in `knex_migrations_lock` serialises
   runners. A crashed run can leave it set ("Migration table is already
   locked"): confirm nothing is running, then `knex migrate:unlock`.

## ESM and TypeScript

- **`"type": "module"` projects:** `knexfile.js` with `export default {…}` and
  `.js` migrations with `export const up` / `export const down` (seeds:
  `export const seed`) load fine, but `migrate:make` and `seed:make` still
  write CommonJS stubs (`exports.up`) into `.js` files, which fail with
  "exports is not defined in ES module scope". Rewrite them by hand, or point
  `migrations.stub` / `seeds.stub` at your own template.
- **`.mjs` migrations are ignored by default.** `migrate:make -x mjs` creates
  one and `migrate:latest` then reports nothing to run. Add
  `migrations: { extension: 'mjs', loadExtensions: ['.mjs', '.js'] }`.
- **TypeScript:** `knex init -x ts` and `migrate:make name -x ts` (or
  `migrations.extension: 'ts'`) generate:

  ```ts
  import type { Knex } from 'knex';
  export async function up(knex: Knex): Promise<void> {}
  export async function down(knex: Knex): Promise<void> {}
  ```

  Node versions with built-in type stripping (22.18+) run a `knexfile.ts` and
  `.ts` migrations as they are (checked on Node 26); the CLI may first print
  red "Failed to load external module ts-node/register" lines, which are
  harmless when the command then succeeds. Older versions need
  `ts-node` or `tsx` installed
  (`node --import tsx node_modules/knex/bin/cli.js migrate:latest`).
- **Don't mix `.ts` in development with compiled `.js` in production against
  the same database.** The recorded names differ
  (`20260101_x.ts` vs `20260101_x.js`) and validation fails with "migration
  directory is corrupt". Either run the same form everywhere, or set
  `loadExtensions: ['.js']` on the compiled side and migrate every database
  with it. When compiling, also keep `.d.ts` files out: `loadExtensions`
  matches by suffix, and `.ts` is in the default list.

## Seeds

```js
exports.seed = async (knex) => {
  await knex('users').del();
  await knex('users').insert([{ id: 1, email: 'a@example.com' }]);
};
```

Seeds are not tracked: `seed:run` executes **every** file in the directory, in
file-name order, every time. Number the files (`01_users.js`, `02_orders.js`),
delete children before parents, and make each seed idempotent. They are for
development and test data; ship required reference data in a migration.
On PostgreSQL, inserting explicit ids leaves the sequence behind: finish with
`select setval(pg_get_serial_sequence('users', 'id'), (select max(id) from users))`.

## Programmatic API

```js
const [batchNo, ran] = await knex.migrate.latest();          // ran: file names
await knex.migrate.rollback();                               // last batch
await knex.migrate.rollback(undefined, true);                // everything
await knex.migrate.up();          await knex.migrate.down({ name: '2026…_x.js' });
await knex.migrate.to({ name: '2026…_x.js' });               // 3.3+
await knex.migrate.before({ name: '2026…_x.js' });           // 3.3+
const [completed, pending] = await knex.migrate.list();      // [{ name }], [{ file, directory }]
await knex.migrate.currentVersion();                         // '20261126085120' or 'none'
await knex.migrate.status();                                 // 0 = up to date, negative = behind
await knex.migrate.forceFreeMigrationsLock();
await knex.seed.run({ specific: '01_users.js' });            // [[absolute paths of the files run]]
```

Every method takes the `migrations` config as its argument to override the
knexfile (`{ directory, tableName, … }`). Lifecycle hooks (3.2+) go in the same
object for `latest`, `rollback`, `up`, `down`, `to`, `before`:

```js
await knex.migrate.latest({
  beforeAll: async (trxOrKnex, migrations) => {},
  beforeEach: async (trxOrKnex, [migration]) => {},          // migration.file is the file name
  afterEach: async (trxOrKnex, [migration]) => {},
  afterAll: async (trxOrKnex, migrations) => {},
});
```

Tests: run `await knex.migrate.latest()` in global setup against a throwaway
database and `await knex.destroy()` in teardown. With `:memory:` SQLite every
connection is its own empty database, so leave the pool at the SQLite default
of one connection.

## Errors

| Message | Cause and fix |
| --- | --- |
| `The migration directory is corrupt, the following files are missing: …` | a recorded migration is gone or renamed (or `.ts` vs `.js`). Restore the file; last resort `disableMigrationsListValidation: true` |
| `Migration table is already locked` | another run is active or one crashed: `knex migrate:unlock` |
| `Already up to date` but changes are missing | wrong `--env`/database, wrong `directory`, or files with an extension not in `loadExtensions` |
| `Invalid migration: … must have both an up and down function` | a missing or misnamed export, or a stray non-migration file (`.d.ts`, helper) in the directory |
| `exports is not defined in ES module scope` | a generated CommonJS stub inside a `"type": "module"` project: rewrite as `export const up` |
| child rows vanished after a SQLite migration | table rebuild inside a transaction: see rule 8 |
| migration "succeeds", table missing | `up` did not await/return the schema call |
| `Knex: Timeout acquiring a connection` during a migration | a query on a global knex instance instead of the `knex` argument |
| `relation … already exists` after a failed MySQL/`transaction: false` run | the batch was half applied: fix the database by hand or make the migration re-runnable (`hasTable`, `if not exists`) |
