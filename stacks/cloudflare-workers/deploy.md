# Deploy a SvelteKit app to Cloudflare Workers

Use Workers Builds to deploy each push. Use Wrangler from a laptop only for the first deploy or a fix.
To start the app, use [New SvelteKit project](../sveltekit/new-project.md).
For secrets from Infisical, use [Worker sync and build setup](../../operations/environment-variables-with-infisical.md#cloudflare-workers-and-hosted-builds).

## First deploy from a laptop

1. Log in with `bunx wrangler login`.
2. Apply D1 migrations with `bunx wrangler d1 migrations apply <database> --remote`.
3. Build with `bun run build`.
4. Deploy with `bunx wrangler deploy`.
5. Add each runtime secret with `bunx wrangler secret put <NAME>`.

Expected result: the Worker answers on its `workers.dev` URL.

## Connect Workers Builds

1. Open **Workers & Pages**, select the Worker, and open **Settings → Builds**.
2. Connect the GitHub repository and select the production branch.
3. Set the values in the table.
4. Push a commit and confirm that a build runs and deploys.

| Setting               | Value                                                |
| --------------------- | ---------------------------------------------------- |
| Root directory        | The app folder, for example `web`                    |
| Build command         | `bun run build`                                      |
| Deploy command        | `bunx wrangler deploy`                               |
| Build watch paths     | `web/*`                                              |
| Non-production deploy | `bunx wrangler versions upload`                      |
| Build variables       | Each `PUBLIC_*` value and each value the build reads |

- Set a build command. Without it, the deploy fails because `.svelte-kit/cloudflare/_worker.js` is missing.
- `PUBLIC_*` values go into the bundle at build time. Runtime secrets do not supply them.
- `wrangler versions upload` does not change production. Use `wrangler deploy`, or promote the version under **Deployments**.
- For a staging preview, use `bunx wrangler versions upload --env staging --name <worker> --keep-vars`. A preview uses the D1 database that its version's binding names.
- When one repository deploys more than one Worker, use `bun --no-env-file run build` and a deploy script that deploys the dependent Worker first.

## Custom domain

Add the route to `wrangler.toml`:

```toml
routes = [{ pattern = "example.com", custom_domain = true }]
```

1. Rebuild before you deploy. Old build output does not include the new route.
2. Deploy.
3. Confirm that the domain answers over HTTPS.

The zone must be in the same Cloudflare account.
To attach a domain without a deploy, open the Worker and select **Settings → Domains & Routes → Add**.

## D1 migrations

Apply additive migrations before the deploy:

```sh
bunx wrangler d1 migrations apply <database> --remote
bun run build
bunx wrangler deploy
```

For a breaking migration, deploy the new code and apply the migration with no time between them.
Old code that runs against a new schema fails.

## Rebuild when CMS content changes

A prerendered site reads the CMS at build time. To rebuild after each CMS deploy, use a deploy hook.
See [Start a prerendered frontend build after each CMS deploy](../../operations/statamic-infisical-dokploy.md#start-a-prerendered-frontend-build-after-each-cms-deploy).
Cloudflare merges hook calls that arrive before the last build starts.

To find a missed build, read `GET /accounts/<account-id>/builds/workers/<worker-tag>/builds`.

## Troubleshooting

| Symptom                                                      | Cause and fix                                                                                                 |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| `wrangler secret put` fails with error 10215                 | The latest uploaded version is not deployed. Run `wrangler versions secret put <NAME>`, then deploy.          |
| A secret is empty after `op read ... \| wrangler secret put` | The `op read` call failed and Wrangler stored an empty value. Check the reference. Run `op read` alone first. |
| An old preview URL returns 500 after you add secrets         | The alias keeps its old bindings. Upload a new version with the same alias.                                   |
| The deploy is "successful" but production did not change     | The deploy command was `wrangler versions upload`. Use `wrangler deploy`.                                     |
| The custom domain is gone after a deploy                     | The deploy used old build output. Rebuild, then deploy.                                                       |

For runtime logs, see [Cloudflare Workers logs](../../operations/cloudflare-workers-logs.md).
