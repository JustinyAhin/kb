# SvelteKit environment variables with Varlock and 1Password

Use this optional approach when a project selects Varlock with 1Password.
Put secret loading in project startup commands. Keep normal commands such as `bun run dev`.
Use this guide for new projects and small changes to existing projects.

Reviewed against official documentation on 2026-10-12.
Confirm the installed SvelteKit major version, Vite version, and deployment adapter before changes.
All names and identifiers below are examples.

The [Varlock and 1Password runbook](../../operations/environment-variables-with-varlock-and-1password.md) owns account scope, machine authentication, token storage, and recovery.
Use its procedures on both development machines. This guide owns application startup and variable access.

## 1. Identify the consumers

List variable names from public examples, source references, package scripts, and binding declarations.
Do not open secret `.env` files or run a tool that automatically reads them during this inventory.
Record whether each variable is required, public, private, or optional with a defined fallback.

| Consumer | Needed change |
| --- | --- |
| Standalone Bun or Node script reading `process.env` | Wrap its startup command. Existing lookups can remain. |
| SvelteKit environment module | Preserve the installed version's module and adapter contract. Supply values at the correct build or runtime phase. |
| Script parsing an environment file directly | Add a small environment lookup adapter, or provide a generated file with an explicit lifecycle. |
| Worker request binding | Preserve `event.platform.env`, or the existing project wrapper. Configure local and production bindings separately. |

Do not change every application import to `varlock/env` merely to load secrets.
That migration is needed only when selecting Varlock's code-level integration features.

For a direct file reader, prefer an environment lookup for the required names.
Generated files are an alternative when a tool requires a file.
Define private permissions, cleanup, refresh, and failure behavior before using that alternative.

## 2. Add the project schema and dependencies

Create a project-specific development Environment through the shared runbook.
Keep its ID and bootstrap settings in `~/.config/varlock/example-project/.env.local` on each machine.
Use desktop authentication locally and a scoped service account on the server.

Install pinned dependencies with the existing package manager. For a Bun project, these are example version pins:

```bash
bun add --exact varlock@1.22.0 @varlock/1password-plugin@2.0.5
```

Commit the lockfile. Keep the dependency group available during each wrapped command's installation phase.
Review [Varlock installation](https://varlock.dev/getting-started/installation/) before selecting another version.

Start `.env.schema` from the runbook's [schema template](../../operations/environment-variables-with-varlock-and-1password.md#3-define-a-shared-cli-schema).
Use `@plugin(@varlock/1password-plugin)` because the project installs the plugin as a dependency.
Change the import path to the project's machine configuration.
Import only `OP_TOKEN` and `OP_ENVIRONMENT_ID` from that file.
Keep both internal. Also keep the legacy `OP_SERVICE_ACCOUNT_TOKEN` name internal.

Declare the application's variables explicitly. For example:

```dotenv
# @required @sensitive @type=string
EXAMPLE_API_KEY=

# @required @public @type=url
PUBLIC_SITE_URL=
```

Choose types that match actual application contracts.
Use `@optional` only for an implemented fallback.
Keep unknown bulk keys disabled. Restrict bulk loading with `pick` when the Environment serves additional consumers.
Never put secret values in the schema, examples, README, or tracked configuration.

Keep secret environment files ignored. Make the public schema an explicit exception when necessary:

```gitignore
.env*
!.env.schema
```

Review existing ignore rules before changing them.

## 3. Keep the ordinary startup commands

For a standard Vite development server, adapt `package.json`:

```json
{
  "scripts": {
    "dev": "varlock run --path .env.schema --inject vars -- vite dev",
    "env:check": "varlock load --path .env.schema --agent",
    "assets:generate": "varlock run --path .env.schema --inject vars -- bun --no-env-file scripts/create-assets.ts"
  }
}
```

Preserve existing server arguments and script names.
Wrap standalone scripts only when they need those variables.
The explicit schema path avoids Varlock's automatic project environment-file discovery.
Internal bootstrap values are excluded from child processes.
`--inject vars` excludes the serialized configuration blob.

In the package's `bunfig.toml`, disable Bun's automatic dotenv loading:

```toml
env = false
```

This prevents old project files from entering `process.env` before Varlock starts.
Use `--no-env-file` for child Bun commands when necessary.
Files selected explicitly with `--env-file` still load.
See [Bun environment loading](https://bun.com/docs/runtime/environment-variables).

For **standalone Vite**, disable its environment-file discovery in `vite.config.ts`:

```ts
export default defineConfig({
  envDir: false
});
```

Merge the setting into the existing configuration. Keep its existing imports and plugins.
See [Vite `envDir`](https://vite.dev/config/shared-options.html#envdir).

SvelteKit 2 also loads environment files through its own `kit.env.dir` setting.
Do not assume the standalone Vite setting disables this separate loader.
For this approach, select a dedicated empty source in `svelte.config.js`:

```js
const config = {
  kit: {
    env: { dir: './.no-dotenv' }
  }
};

export default config;
```

Merge this setting into the existing configuration. Keep the adapter and other settings.
Keep `.no-dotenv` absent or empty. Do not put dotenv files there.
Ignore `.no-dotenv/` so accidental contents cannot enter source control.
Check that the installed loader accepts an absent directory during fixture tests.
An existing-project migration can also remove legacy files through an authorized private workflow.
Check adapter-specific loaders, including Wrangler, separately.
New projects can avoid legacy secret files from the start.

Intentional shell exports retain Varlock's normal override precedence.
Remove obsolete exports once during migration instead of repeating variable filters in every script.
The schema remains the application's variable inventory.

## 4. Preserve the application's access contract

Standalone scripts can continue reading injected values:

```ts
const apiKey = process.env.EXAMPLE_API_KEY;
if (!apiKey) throw new Error('EXAMPLE_API_KEY is required');
```

Keep private lookups in server-only code. Do not log values.

For existing **SvelteKit 2** projects:

| Module | Value phase and boundary |
| --- | --- |
| `$env/static/private` | Replaced at build time. Server-only imports. The value can enter the server bundle. |
| `$env/static/public` | Replaced at build time. Browser-visible variables use the configured public prefix, normally `PUBLIC_`. |
| `$env/dynamic/private` | Runtime values depend on the adapter. Server-only imports. |
| `$env/dynamic/public` | Runtime public values are sent to the browser. |

A changed static value needs a rebuild. A changed injected runtime value needs a new process load.
Do not inject development credentials into production builds that embed static values.
Use the existing build configuration and deployment scope.

The [SvelteKit 2 module definitions](https://github.com/sveltejs/kit/tree/%40sveltejs/kit%402.48.4/packages/kit/src/types/synthetic) describe these contracts.
Current SvelteKit 3 documentation uses explicit definitions and `$app/env/private` or `$app/env/public`.
The older `$env/*` modules are deprecated in version 3.
For a new project, follow the [environment API for its installed major version](https://svelte.dev/docs/kit/environment-variables).
Do not upgrade SvelteKit as a side effect of secret loading.

Varlock sensitivity and framework exposure are separate rules in this CLI-only pattern.
Never give a secret a browser-visible `PUBLIC_` or `VITE_` name.
Do not change public prefixes to expose all variables.

For Cloudflare, preserve the [Worker binding contract](../cloudflare-workers/conventions.md#runtime-and-bindings).
Use request-scoped `event.platform.env`, or the project's explicit wrapper, for runtime Worker secrets.
Development injection does not configure production secrets.
The deployed Worker must not depend on a server token file or a local 1Password CLI.

## Optional framework integration

The [official Varlock SvelteKit integration](https://varlock.dev/integrations/sveltekit/) adds Vite integration, generated types, and code-level checks.
Select it when those features have a defined benefit.
Follow its plugin order and application lookup requirements.
Avoid running duplicate loaders with different schemas or authentication paths.
Keep a CLI startup wrapper when ordinary scripts also need injected `process.env` values.

## Verification and project checklist

Use fake data in a disposable fixture for configuration tests.
Do not read existing secret files or call paid providers by default.

1. Check the public variable inventory against source consumers.
2. Run the repository's existing format and type checks.
3. Validate the schema with a mock `op` executable and fake bootstrap token.
4. Confirm only declared application variables are loaded from the provider.
5. Confirm bootstrap names and `__VARLOCK_ENV` do not reach the child.
6. Confirm missing required values and provider errors block startup.
7. Add fake stale `.env` and `.env.local` values in the fixture.
8. Run the actual package command with those fixtures.
9. Confirm values follow the selected loader contract without unexpected file fallback.
10. Check public and private bundle boundaries with the selected adapter.
11. Check the Mac and server launch paths with presence-only output when machine configuration is ready.
12. Document the normal commands, optional feature fallbacks, and production configuration owner in the project README.

Use the shared runbook's [safe verification procedure](../../operations/environment-variables-with-varlock-and-1password.md#verification) for real authentication checks.
`--agent` output still shows public values. Keep private metadata sensitive.
Test a feature with provider access only when that test is required and authorized.

## Recovery

If startup fails, use the shared runbook's [recovery table](../../operations/environment-variables-with-varlock-and-1password.md#rotation-and-recovery).
If an application lookup fails, check its build phase, runtime adapter, and variable name.
Restore only the affected startup configuration or lookup adapter when reverting this change.
Keep the previous private configuration until the replacement passes checks.
