# D1 + Drizzle + Atlas

This is a recurring pattern, not a requirement for every Cloudflare Worker
application. Confirm the repository’s database and migration tooling first.

## Database structure

- Define the Drizzle schema in `src/lib/server/db/schema.ts`.
- Create the Drizzle client from the request’s D1 binding; do not create a
  module-level database singleton.
- Pass the request-scoped `db` explicitly into database operations.
- Keep reusable queries in `src/lib/server/db/operations/`, commonly one file
  per table.
- Use the project’s established ID and timestamp conventions consistently.
- Treat the Drizzle schema as the source of truth for schema changes.

## Atlas migration workflow

Several projects use Atlas to generate migration SQL from the Drizzle schema
and Wrangler to apply migrations to D1:

1. Edit the schema.
2. Run the project’s `atlas:diff` command.
3. Inspect the generated migration.
4. Re-hash after manual migration edits with `atlas:hash`.
5. Apply locally, then apply remotely with the project’s `db:migrate` scripts.

Do not run `drizzle-kit generate` in an Atlas-based project: it produces a
different migration format. Conversely, use `drizzle-kit` when the repository
explicitly chose it instead of Atlas.

