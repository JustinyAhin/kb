# SvelteKit + Cloudflare Workers conventions

Reusable patterns found across the SvelteKit projects deployed to Cloudflare
Workers. Confirm each repository’s SvelteKit version, adapter, package scripts,
and enabled experimental features before applying a rule.

## Baseline stack

- SvelteKit 2, commonly with Svelte 5 runes.
- `@sveltejs/adapter-cloudflare` and Wrangler for local preview, types, and
  deployment.
- TypeScript in strict mode where the project enables it.
- Tailwind CSS and, where already installed, shadcn-svelte.
- Bun is common in these repositories; use the package manager already declared
  by the repository.

When Svelte 5 is enabled, use runes (`$state`, `$derived`, `$effect`, `$props`)
and do not mix in Svelte 4 reactive syntax unless the project explicitly does
so for compatibility.

## SvelteKit routing and server boundaries

- Use direct `export const` declarations for route exports such as `load`,
  `entries`, and `prerender`. Avoid unnecessary explicit generated `$types`
  annotations; add them when they genuinely improve or are required by the
  code.
- Prefer remote functions in `src/lib/remote-functions/` for browser-to-server
  communication when remote functions are enabled:
  - `query` for reads;
  - `form` for progressively enhanced form submissions;
  - `command` for writes that are not form submissions.
- Reserve `+server.ts` endpoints for external webhooks, third-party callbacks,
  non-browser clients, and other cases that need a normal HTTP endpoint.
- Keep server-only code and imports out of shared/client modules. Do not leak
  Worker secrets, database clients, or private environment access into code the
  client can bundle.

### Authentication

Do not rely on `+layout.server.ts` as the only authorization boundary: layout
loads can be skipped during client-side navigation. Put route authorization in
each protected `+page.server.ts` load, or enforce it globally in
`hooks.server.ts`. Remote functions and `+server.ts` handlers must also verify
authorization at their own boundary.

## Cloudflare runtime and bindings

- Treat the Worker runtime as the source of truth for production bindings and
  secrets. Access request-scoped bindings through `event.platform.env` (or the
  project’s explicitly configured SvelteKit environment wrapper).
- Do not assume Node-only globals, filesystem access, long-lived process state,
  or module-level mutable singletons are available in a Worker.
- Do not use `$env/static/private` or `$env/dynamic/private` for runtime Worker
  bindings when the project’s adapter exposes them through `event.platform.env`.
  Follow an existing explicit environment wrapper if the project has one.
- Declare D1, KV, R2, Hyperdrive, and other bindings in `wrangler.toml`.
- After changing bindings, run the repository’s type-generation command,
  commonly `bun gen` / `wrangler types`.
- Never edit generated `worker-configuration.d.ts` by hand.
- Keep secrets out of source and version control. Use Wrangler/project tooling
  to configure production secrets; document the variable name and expected
  purpose without printing its value.

## Database pattern when using D1 + Drizzle

This is a recurring pattern, not a requirement for every Worker app:

- Define the Drizzle schema in `src/lib/server/db/schema.ts`.
- Create the Drizzle client from the request’s D1 binding; do not create a
  module-level database singleton.
- Pass the request-scoped `db` explicitly into database operations.
- Keep reusable queries in `src/lib/server/db/operations/`, commonly one file
  per table.
- Use the project’s established ID and timestamp conventions consistently.
- Treat the Drizzle schema as the source of truth for schema changes.

Several projects use Atlas to generate migration SQL from the Drizzle schema
and Wrangler to apply migrations to D1. In those projects:

1. edit the schema;
2. run the project’s `atlas:diff` command;
3. inspect the generated migration;
4. re-hash after manual migration edits with `atlas:hash`;
5. apply locally, then apply remotely with the project’s `db:migrate` scripts.

Do not run `drizzle-kit generate` in an Atlas-based project: it produces a
different migration format. Conversely, use `drizzle-kit` when the repository
explicitly chose it instead of Atlas.

## Verification checklist

Run from the app directory, using the names in `package.json`:

```bash
bun format <changed-files>
bun lint
bun check
bun build
```

The last command is especially useful for catching adapter, Worker, and
generated-type problems, but may be omitted when the repository has a narrower
documented check. If `check` or `build` regenerates Wrangler types, inspect the
resulting diff and do not hand-edit generated output.

## Useful documentation pointers

The projects consistently point agents to the LLM-oriented Svelte docs:

- https://svelte.dev/docs/svelte/llms.txt
- https://svelte.dev/docs/kit/llms.txt
- https://svelte.dev/docs/kit/remote-functions

