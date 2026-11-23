# Statamic as a headless CMS for SvelteKit

Use this runbook to add a Statamic CMS that a SvelteKit app reads through GraphQL.
It was written for Statamic 6, Laravel 13, and PHP 8.5.
To deploy the CMS, use [Dokploy deploy](../../operations/dokploy-deploy.md) and [Statamic with Infisical on Dokploy](../../operations/statamic-infisical-dokploy.md).

## Repository layout

```text
example-project/
  web/          SvelteKit app on Cloudflare Workers
  example-cms/  Statamic CMS on Dokploy
  kb/           project notes
```

## 1. Create the CMS

1. Create the project with the Statamic CLI or Herd in `example-cms/`.
2. Link it in Herd with SSL. Herd then serves `https://example-cms.test`.
3. Add the typed GraphQL addon with `composer require segbedji/graphql-typed`.
4. Make the first user with `php please make:user`.

Use `php please` for Statamic commands. Use `php artisan` for Laravel commands.

## 2. Turn on GraphQL

1. In `.env`, set `STATAMIC_PRO_ENABLED=true` and `STATAMIC_GRAPHQL_ENABLED=true`.
2. Set `STATAMIC_GRAPHQL_AUTH_TOKEN` to a long random value.
3. In `config/statamic/graphql.php`, list each collection, nav, taxonomy, and global that the app reads.
4. Give each collection its `allowed_filters`, for example `['id', 'published', 'slug', 'status', 'title', 'uri']`.

Without `allowed_filters`, Statamic refuses the filter, even when the typed addon shows the field.
Without a token, the GraphQL API is public. The container start script must stop when GraphQL is on and the token is empty.

## 3. Read the CMS from SvelteKit with gqty

1. Add `CMS_GRAPHQL_ENDPOINT` and `CMS_GRAPHQL_AUTH_TOKEN` to `web/.env.example` and `web/src/env.ts`. Make both `v.optional`.
2. Add `@gqty/cli`, `graphql`, and `gqty`.
3. Write `web/scripts/cms-generate.ts`. It sends a GraphQL introspection query with the token, passes the result to `generate` from `@gqty/cli`, and writes `src/lib/server/cms/schema.generated.ts`.
4. Add the script `"cms:generate": "bun --env-file=.env scripts/cms-generate.ts"`.
5. Run `bun run cms:generate` after each blueprint change. Never edit the generated file.
6. Keep CMS code in `src/lib/server/cms/`. Map gqty results to plain objects there.

Prerender all CMS pages. The deployed Worker then does not need the CMS values.
A catch-all `[...path]` route lists the page URLs in its `entries` export.
The dev server shows drafts. Builds publish only published entries.

## 4. Container layout

The image has three stages in one `Dockerfile`:

| Stage      | Base image           | Job                                                     |
| ---------- | -------------------- | ------------------------------------------------------- |
| `frontend` | `node:24-alpine`     | Build the control panel Vite assets into `public/build` |
| `app`      | `php:8.5-fpm-alpine` | Install Composer packages and run PHP-FPM on port 9000  |
| `web`      | `caddy:2-alpine`     | Serve `public/` and send PHP requests to `cms-app:9000` |

`compose.yaml` runs `cms-app` and `cms-web`. Keep `storage` and `users` in named volumes.
Mount `storage` read-only into `cms-web`.

Required details:

- PHP 8.5 includes OPcache. Do not install it as an extension.
- Copy the complete `public` tree into the Caddy image, after `vendor:publish --tag=statamic-cp`.
- Use `docker/start.sh` as the entrypoint. It stops when `APP_KEY` is empty, then clears caches, links storage, and runs `php please stache:refresh`. The command `php please statamic:stache:refresh` fails, and the first deploy fails with it.
- Set `clear_env = no` in the PHP-FPM pool. PHP then sees the container environment.
- In `bootstrap/app.php`, call `$middleware->trustProxies(at: '*')`. Statamic then builds `https://` URLs behind Dokploy.
- In the Caddyfile, send `X-Robots-Tag: noindex` on every path except `/assets/*`. Only the web app must appear in search.
- Exclude `.env*`, `vendor`, `node_modules`, `storage`, and `public/build` in `.dockerignore`.

## 5. Choose who owns content

When Git owns `content/`, do not mount a volume on `content/`.
A volume hides the content in the new image. Use `php please stache:refresh`, not `stache:warm`.
Edit content in Git only. The next deploy deletes control panel edits on the server.
For the other model, see [Choose a Statamic persistence model](../../operations/statamic-infisical-dokploy.md#choose-a-statamic-persistence-model).

## 6. Make the first production user

Use one address for each site, for example `me+<site>@example.com`. Do not use a personal address.
Make the user with no known password. Then set the password so that no person or agent sees it.

1. After the first deploy, make the password in 1Password. Print only the item ID:

   ```sh
   op item create --category login --vault <vault> --title "<site> CMS" \
     --generate-password=letters,digits,symbols,32 username=me+<site>@example.com \
     --format json | jq -r .id
   ```

2. Copy a small PHP script into the PHP container. It boots Laravel, reads the password from stdin, and saves it:

   ```php
   <?php
   require __DIR__.'/vendor/autoload.php';
   $app = require __DIR__.'/bootstrap/app.php';
   $app->make(Illuminate\Contracts\Console\Kernel::class)->bootstrap();
   $p = rtrim(stream_get_contents(STDIN), "\n");
   Statamic\Facades\User::findByEmail('me+<site>@example.com')->password($p)->save();
   ```

3. Send the password over SSH into the container:

   ```sh
   op read "op://<vault>/<item-id>/password" \
     | ssh root@<server-ip> docker exec -i <cms-app-container> php /var/www/html/set-password.php
   ```

4. Check it the same way. Use a second script that prints only the result of `Hash::check($p, $user->password())`.
5. Delete both scripts from the container.

A named volume holds `users/`, so a redeploy keeps the user.

## Verify

1. Open `https://example-cms.test/cp` and log in.
2. Run `bun run cms:generate` in `web/`. Expected result: the script writes the schema file.
3. Run `bun run build` in `web/`. Expected result: the build prerenders the CMS pages.
