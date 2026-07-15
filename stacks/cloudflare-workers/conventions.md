# Cloudflare Workers conventions

Reusable runtime and binding patterns for SvelteKit applications deployed to
Cloudflare Workers.

## Runtime and bindings

- Use `@sveltejs/adapter-cloudflare` and Wrangler for Worker preview, types,
  and deployment when the project targets Workers.
- Treat the Worker runtime as the source of truth for production bindings and
  secrets. Access request-scoped bindings through `event.platform.env` or the
  project’s explicitly configured SvelteKit environment wrapper.
- Do not assume Node-only globals, filesystem access, long-lived process state,
  or module-level mutable singletons are available in a Worker.
- Do not use `$env/static/private` or `$env/dynamic/private` for runtime Worker
  bindings when the project’s adapter exposes them through `event.platform.env`.
  Follow an existing explicit environment wrapper if the project has one.
- Declare D1, KV, R2, Hyperdrive, and other bindings in `wrangler.toml`.
- After changing bindings, run the repository’s type-generation command,
  commonly `bun gen` / `wrangler types`.
- Never edit generated `worker-configuration.d.ts` by hand.

## Secrets

Keep secrets out of source and version control. Use Wrangler/project tooling to
configure production secrets; document the variable name and expected purpose
without printing its value.

Do not read `.env` or `.env.keys` while working. Describe required changes and
let the user set the values.

## Related runbooks

- [D1 + Drizzle + Atlas](d1-drizzle-atlas.md)
- [Cloudflare Workers logs](../../operations/cloudflare-workers-logs.md)

