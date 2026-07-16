# SvelteKit conventions

Reusable patterns found across the SvelteKit projects. Confirm each
repository’s SvelteKit version, adapter, package scripts, and enabled
experimental features before applying a rule.

## Baseline stack

- SvelteKit 2, commonly with Svelte 5 runes.
- TypeScript in strict mode where the project enables it.
- Tailwind CSS and, where already installed, shadcn-svelte.
- Bun is common in these repositories; use the package manager already declared
  by the repository.

When Svelte 5 is enabled, use runes (`$state`, `$derived`, `$effect`, `$props`)
and do not mix in Svelte 4 reactive syntax unless the project explicitly does
so for compatibility.

For Tailwind IntelliSense diagnostics, see
[`../../tools/tailwind-intellisense.md`](../../tools/tailwind-intellisense.md).

## Routing and server boundaries

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
  secrets, database clients, or private environment access into code the client
  can bundle.

## Authentication

Do not rely on `+layout.server.ts` as the only authorization boundary: layout
loads can be skipped during client-side navigation. Put route authorization in
each protected `+page.server.ts` load, or enforce it globally in
`hooks.server.ts`. Remote functions and `+server.ts` handlers must also verify
authorization at their own boundary.

## Verification

Run from the app directory, using the names in `package.json`:

```bash
bun format <changed-files>
bun lint
bun check
bun build
```

The last command may be omitted when the repository has a narrower documented
check. If `check` or `build` regenerates adapter-specific types, inspect the
resulting diff and do not hand-edit generated output.

Run checks in the repository's documented order. Some adapters leave compiled
JavaScript under a generated directory that a later type check can discover. In
those projects, run the source/type check before the production build. If the
check must be repeated afterward, clear only disposable generated build output
with the repository's supported clean step, regenerate types, and then rerun
the check. Do not treat diagnostics from compiled adapter output as source
diagnostics.

## Documentation

- https://svelte.dev/docs/svelte/llms.txt
- https://svelte.dev/docs/kit/llms.txt
- https://svelte.dev/docs/kit/remote-functions

Cloudflare-specific runtime guidance is in
[`../cloudflare-workers/conventions.md`](../cloudflare-workers/conventions.md).
The recurring D1/Drizzle/Atlas pattern is in
[`../cloudflare-workers/d1-drizzle-atlas.md`](../cloudflare-workers/d1-drizzle-atlas.md).
