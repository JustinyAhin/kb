# New SvelteKit project on Cloudflare Workers

Use this runbook to start a SvelteKit app that deploys to Cloudflare Workers with Bun.
It was written for SvelteKit 3, Svelte 5, and Wrangler 4.
To deploy the app, use [Workers deploy](../cloudflare-workers/deploy.md).

## 1. Scaffold the app

Run the command from the repository root. The target folder must be empty or absent.

```sh
bun x sv create web --template minimal --types ts \
  --add prettier eslint "sveltekit-adapter=adapter:cloudflare+cfTarget:workers" \
  --install bun
```

Check the SvelteKit major version in `web/package.json`.
The SvelteKit 3 rules below do not apply to SvelteKit 2.

## 2. Replace `wrangler.jsonc` with `wrangler.toml`

`sv` writes `wrangler.jsonc`. Delete it and write `wrangler.toml`. TOML is the preferred format.
Turn on logs and traces at the start:

```toml
name = "example-app"
compatibility_date = "2026-09-15"
compatibility_flags = ["nodejs_als"]
main = ".svelte-kit/cloudflare/_worker.js"
workers_dev = true
preview_urls = true

[assets]
binding = "ASSETS"
directory = ".svelte-kit/cloudflare"

[observability]
enabled = true

[observability.logs]
enabled = true
head_sampling_rate = 1
invocation_logs = true
persist = true

[observability.traces]
enabled = true
head_sampling_rate = 1
persist = true
```

Add D1, KV, R2, and other bindings to this file.
For a custom domain, see [Workers deploy](../cloudflare-workers/deploy.md#custom-domain).

## 3. Generate `Env` types from `.env.example`

Make the tracked `.env.example` the only input for type generation.
It holds variable names and safe defaults. It never holds secret values.

```sh
# Copy to .env and add the values. Wrangler also reads this file to generate Env types.
CMS_GRAPHQL_ENDPOINT=https://example-cms.test/graphql
CMS_GRAPHQL_AUTH_TOKEN=
RESEND_API_KEY=
```

Reason: without `--env-file`, Wrangler reads the git-ignored `.env`.
CI has no `.env`, so its `Env` type has no secret names and the type check fails.
With `.env.example`, local and CI types are the same.

Use these `package.json` scripts:

```json
{
  "dev": "env NODE_OPTIONS=--use-system-ca vite dev",
  "build": "wrangler types --env-file .env.example && env NODE_OPTIONS=--use-system-ca vite build",
  "preview": "wrangler dev .svelte-kit/cloudflare/_worker.js --port 4173",
  "check": "wrangler types --env-file .env.example --check && svelte-kit sync && svelte-check --tsconfig ./tsconfig.json",
  "gen": "wrangler types --env-file .env.example"
}
```

- `NODE_OPTIONS=--use-system-ca` lets Node trust the local Herd certificate. Add it only when dev or build calls a local HTTPS service.
- `wrangler types --check` also reads `.svelte-kit/cloudflare/_worker.js`. After a build, run `bun run gen` before `bun run check`. Otherwise the check reports changed types.
- Commit `worker-configuration.d.ts` or ignore it. Keep one choice per repository. Never edit it by hand.

Add the generated file to `tsconfig.json` and type `App.Platform` in `src/app.d.ts`:

```ts
interface Platform {
  env: Env;
  ctx: ExecutionContext;
  caches: CacheStorage;
  cf?: IncomingRequestCfProperties;
}
```

## 4. Stop automatic `.env` loading

Load environment values only on purpose.
Then local secrets do not go into the browser bundle or into a different tool.

- In `bunfig.toml`, set `[run] env = false`. Bun then does not load `.env` into each script.
- In `vite.config.ts`, set `envDir: false`. Vite then does not read `.env` files.
- Run scripts that need `.env` with an explicit flag, for example `bun --env-file=.env scripts/x.ts`.

## 5. Declare environment variables in `src/env.ts` (SvelteKit 3)

SvelteKit 3 replaces `$env/*` with `defineEnvVars`. Validate each value with Valibot.

```ts
import { defineEnvVars } from "@sveltejs/kit/env";
import * as v from "valibot";

const variables = defineEnvVars({
  RESEND_API_KEY: {
    description: "Resend API key that sends the request notice",
    schema: v.optional(v.pipe(v.string(), v.minLength(1))),
  },
});

export { variables };
```

- Import values from `$app/env/private` or `$app/env/public`.
- Validation runs at startup. A malformed value stops the Worker.
- Make a value `v.optional` when the deployed Worker does not need it. Prerender-only CMS values are an example.
- Keep local secrets in `.env`, not `.dev.vars`. Wrangler falls back to `.env`.
- Read D1 and other bindings from `event.platform.env` or `cloudflare:workers`, not from `src/env.ts`.

Other SvelteKit 3 changes:

- Import app modules with `#lib/...` and a file extension. Add `"imports": { "#lib": "./src/lib/index.js", "#lib/*": "./src/lib/*" }` to `package.json`. `$lib` is removed.
- Import `dev`, `browser`, and `building` from `$app/env`.
- Import hook types such as `Handle` from `@sveltejs/kit/hooks`.

## 6. Configure the adapter in `vite.config.ts`

```ts
const config = defineConfig(({ command }) => {
  const cloudflare = adapter();

  return {
    envDir: false as const,
    plugins: [
      sveltekit({
        experimental: { remoteFunctions: true },
        // Prerender does not use Worker bindings.
        adapter:
          command === "build"
            ? { ...cloudflare, emulate: undefined }
            : cloudflare,
      }),
    ],
  };
});
```

## 7. Add D1 (optional)

1. Create the database with `bunx wrangler d1 create example-app`.
2. Add the `[[d1_databases]]` block that Wrangler prints to `wrangler.toml`. Set `migrations_dir = "migrations"`.
3. Follow [D1 + Drizzle + Atlas](../cloudflare-workers/d1-drizzle-atlas.md) for the schema and migrations.

## 8. Verify

```sh
cp .env.example .env
bun run dev
bun run check
bun run lint
bun run build
bun run preview
```

Expected result: every command passes and `http://localhost:4173` serves the built Worker.
