# Statamic with Infisical on Dokploy

Use this pattern when a containerized Statamic CMS runs on Dokploy and should
load its runtime configuration directly from Infisical. Dokploy stores only the
credentials needed to bootstrap Infisical; application secrets remain in
Infisical and are injected into PHP-FPM when the container starts.

## Architecture

```text
Dokploy Compose environment
  -> Universal Auth client ID and client secret
  -> PHP container authenticates to Infisical
  -> infisical run injects the selected environment and path
  -> Statamic startup and PHP-FPM

Internet
  -> Dokploy/Traefik terminates HTTPS
  -> Caddy serves public files and forwards PHP to PHP-FPM
```

Do not give the Caddy container application secrets. It needs the built
`public` tree, while the PHP container needs the Infisical bootstrap values.

## Configure Infisical

1. Check existing organization machine identities and their project grants.
2. Select an identity whose grants match the CMS workload.
   If access requirements differ, create a separate identity.
   Use **No Access** at organization level.
3. Add the identity to the Infisical project with a read-only project role.
   Prefer a role restricted to production and the CMS folder when the plan permits it.
4. Confirm Universal Auth is configured.
5. Retrieve the existing client ID and client secret from their protected credential store.
   If no client secret exists, create one.
   The Universal Auth client ID differs from the machine identity ID.
6. When IP allowlisting is available, restrict client secret use to the VPS public IP as a `/32`.
   Apply the same restriction to issued access tokens.
   This optional paid feature is not required for deployment.
7. Store the client ID and client secret in the team password manager.
   Keep them separate from the Dokploy API credential.

Create a dedicated folder such as `prod:/cms`. At minimum, provide stable and
production-safe values for:

```text
APP_NAME
APP_ENV=production
APP_KEY
APP_DEBUG=false
APP_URL=https://cms.example.com
LOG_CHANNEL
LOG_LEVEL
DB_CONNECTION
SESSION_DRIVER
CACHE_STORE
QUEUE_CONNECTION
FILESYSTEM_DISK
STATAMIC_PRO_ENABLED
STATAMIC_GRAPHQL_ENABLED
STATAMIC_GRAPHQL_AUTH_TOKEN
STATAMIC_API_ENABLED
STATAMIC_GIT_ENABLED
DEBUGBAR_ENABLED=false
```

Add mail, database, storage, license, and provider values when the application
uses them. Keep `APP_KEY` stable across deployments. Do not generate a new key
inside a running or replacement production container.

## Authenticate at container startup

Install a pinned Infisical CLI release in the PHP runtime image.
The former Cloudsmith package repository stopped serving the CLI on September 16, 2026.
Use the [current installation instructions](https://infisical.com/docs/cli/install) or an official release archive.

For an Alpine image, set the version and architecture-specific checksum through build arguments.
Get both values from the [official CLI release](https://github.com/Infisical/cli/releases).
Pin them in the project's build configuration.

```dockerfile
ARG TARGETARCH
ARG INFISICAL_CLI_VERSION
ARG INFISICAL_CLI_SHA256

RUN apk add --no-cache curl \
    && test -n "$INFISICAL_CLI_VERSION" \
    && test -n "$INFISICAL_CLI_SHA256" \
    && curl -fsSL "https://github.com/Infisical/cli/releases/download/v${INFISICAL_CLI_VERSION}/cli_${INFISICAL_CLI_VERSION}_linux_${TARGETARCH}.tar.gz" \
        -o /tmp/infisical.tar.gz \
    && printf '%s  /tmp/infisical.tar.gz\n' "$INFISICAL_CLI_SHA256" | sha256sum -c - \
    && tar -xzf /tmp/infisical.tar.gz -C /usr/local/bin infisical \
    && chmod 755 /usr/local/bin/infisical \
    && rm /tmp/infisical.tar.gz

ENV INFISICAL_DISABLE_UPDATE_CHECK=true
```

Use an entrypoint that rejects missing bootstrap credentials.
Exchange Universal Auth credentials for an access token.
Remove authentication credentials from the application environment.
Fetch the selected secrets before calling the Statamic startup script:

```sh
#!/bin/sh
set -eu

: "${INFISICAL_CLIENT_ID:?INFISICAL_CLIENT_ID must be set}"
: "${INFISICAL_CLIENT_SECRET:?INFISICAL_CLIENT_SECRET must be set}"
: "${INFISICAL_PROJECT_ID:?INFISICAL_PROJECT_ID must be set}"

INFISICAL_ENVIRONMENT="${INFISICAL_ENVIRONMENT:-prod}"
INFISICAL_SECRET_PATH="${INFISICAL_SECRET_PATH:-/cms}"
INFISICAL_DOMAIN="${INFISICAL_DOMAIN:-https://app.infisical.com/api}"
export INFISICAL_DOMAIN

INFISICAL_TOKEN="$(
    infisical login \
        --method=universal-auth \
        --client-id="$INFISICAL_CLIENT_ID" \
        --client-secret="$INFISICAL_CLIENT_SECRET" \
        --plain \
        --silent
)"
: "${INFISICAL_TOKEN:?Infisical authentication returned no token}"
export INFISICAL_TOKEN

unset INFISICAL_CLIENT_ID INFISICAL_CLIENT_SECRET

exec infisical run \
    --projectId="$INFISICAL_PROJECT_ID" \
    --env="$INFISICAL_ENVIRONMENT" \
    --path="$INFISICAL_SECRET_PATH" \
    --silent \
    -- \
    env -u INFISICAL_TOKEN -u INFISICAL_CLIENT_ID -u INFISICAL_CLIENT_SECRET \
    /var/www/html/docker-start.sh "$@"
```

Replace the example startup path with the project's existing Statamic startup script.
That script prepares writable directories, clears Laravel caches, refreshes Statamic's Stache, and starts PHP-FPM.
Keep secret values out of command output. Keep shell tracing disabled.

## Map only bootstrap values in Compose

Reference specific Dokploy variables instead of loading its entire `.env` into
the container:

```yaml
services:
  cms-app:
    environment:
      INFISICAL_CLIENT_ID: ${INFISICAL_CLIENT_ID}
      INFISICAL_CLIENT_SECRET: ${INFISICAL_CLIENT_SECRET}
      INFISICAL_PROJECT_ID: ${INFISICAL_PROJECT_ID}
      INFISICAL_ENVIRONMENT: ${INFISICAL_ENVIRONMENT:-prod}
      INFISICAL_SECRET_PATH: ${INFISICAL_SECRET_PATH:-/cms}
      INFISICAL_DOMAIN: ${INFISICAL_DOMAIN:-https://app.infisical.com/api}
```

Set these in the Dokploy Compose environment:

```text
INFISICAL_CLIENT_ID=<universal-auth-client-id>
INFISICAL_CLIENT_SECRET=<universal-auth-client-secret>
INFISICAL_PROJECT_ID=<infisical-project-id>
INFISICAL_ENVIRONMENT=prod
INFISICAL_SECRET_PATH=/cms
INFISICAL_DOMAIN=https://app.infisical.com/api
```

Do not put `APP_KEY`, GraphQL tokens, provider credentials, or other Statamic
runtime values in Dokploy. Dokploy writes its Compose environment beside the
Compose file and protects the bootstrap credential only as well as the Dokploy
installation and its backups are protected.

An agent runner's token service does not supply these values to the CMS container automatically.
Configure bootstrap access in the CMS service's Environment tab, even when its identity already works on another server.

## Migrate an existing deployment

1. Confirm the CMS server and Dokploy Compose service.
2. Move the existing production application settings into the selected Infisical folder.
3. Preserve the existing `APP_KEY` and GraphQL token.
4. Verify the identity can fetch that folder without printing values.
5. Add the bootstrap settings in **CMS → Environment**.
6. Save the settings.
7. Deploy the updated image and Compose configuration.
8. Complete the [deployment checks](#verify-the-deployment).
9. After verification passes, remove the old application values from Dokploy.
10. Redeploy with only bootstrap settings.
11. Repeat the deployment checks.

Saving environment settings does not replace running containers.
Retain the existing named volumes during migration.
If startup fails, restore the previous deployment image and its verified environment.
Use [Worker sync and build setup](environment-variables-with-infisical.md#cloudflare-workers-and-hosted-builds) for a Cloudflare frontend.

## Serve assets and trust the HTTPS proxy

Build frontend assets in a separate image stage and copy the complete
application `public` tree into both the PHP image and the Caddy image. Otherwise
Caddy can return the HTML response while CSS, JavaScript, Statamic vendor files,
or uploaded assets return `404`.

Publish the Statamic control panel assets in the final PHP stage. Composer runs
`statamic:install` after `composer install`. If `composer install` runs only in
an earlier stage, that step does not reach the final image:

```dockerfile
RUN php artisan package:discover --ansi \
    && php artisan vendor:publish --tag=statamic-cp --force \
    && php artisan view:cache
```

Without this step, each `/cp` page returns `500` with
`Vite manifest not found at: .../public/vendor/statamic/cp/build/manifest.json`.
Copy `public` into the Caddy image after this step.

Dokploy normally terminates public HTTPS before forwarding the request through
the private Docker network. Configure Laravel to trust those forwarded headers
so Vite and Statamic generate `https://` URLs:

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->trustProxies(at: '*');
})
```

Use the broad proxy setting only when the PHP service is reachable exclusively
through the trusted private proxy network. Also set the Infisical `APP_URL` to
the canonical HTTPS CMS URL. An unstyled page with `http://` stylesheet or
script URLs on an HTTPS document usually means the proxy headers are not being
trusted or `APP_URL` is wrong.

In Dokploy, attach the public hostname to the Caddy service on container port
`80`, enable HTTPS, and use the Let's Encrypt certificate resolver.

## Choose a Statamic persistence model

Named volumes for `storage` and an SQLite `database` do not preserve everything
edited through the Control Panel. Decide explicitly how production changes are
owned:

- Use Statamic's Git workflow when content, blueprints, users, and related flat
  files should be committed and deployed from the repository.
- Use persistent volumes plus tested backups when the VPS is the source of
  truth for `content`, `users`, and `public/assets`.

Do not combine the models accidentally. Confirm which paths a replacement image
overwrites before allowing production edits. When `public/assets` is a local
volume, mount that same volume into both PHP and Caddy at the same path so files
uploaded by Statamic are visible to the web server. Object storage is another
option for deployments that should not share a local asset volume.

## Start a prerendered frontend build after each CMS deploy

A frontend that prerenders from the CMS reads it during the build.
When one push redeploys both, the frontend build can run while the CMS restarts.
The build then fails with HTTP 404 or 502, or it reads old content.

Add a one-shot Compose service that starts the frontend build after the CMS is ready:

```yaml
  # The frontend prerenders from this CMS. A build during a CMS deploy fails or reads old content.
  web-deploy-hook:
    image: curlimages/curl:8.17.0
    restart: "no"
    depends_on:
      cms-web:
        condition: service_healthy
    env_file:
      - .env
    entrypoint: ["/bin/sh", "-c"]
    command:
      - |
        if [ -z "$$CF_DEPLOY_HOOK_URL" ]; then echo "CF_DEPLOY_HOOK_URL is empty. No web build."; exit 0; fi
        for attempt in $$(seq 1 30); do
          if curl -fsS -o /dev/null "$${CMS_PUBLIC_URL:-https://cms.example.com}/up"; then
            curl -fsS -X POST "$$CF_DEPLOY_HOOK_URL" -o /dev/null && echo "Web build started." && exit 0
          fi
          sleep 5
        done
        echo "The public CMS did not answer. No web build."
        exit 1
```

1. In Cloudflare, open **Workers & Pages**, select the Worker, and open **Settings**.
2. In **Builds**, find **Deploy Hooks** and select **Add**.
3. Enter a hook name such as `cms-deploy`. Select the production branch. Select **Save**.
4. Copy the hook URL that Cloudflare shows.
5. In Dokploy, open the CMS Compose service and select **Environment**.
6. Add `CF_DEPLOY_HOOK_URL=<hook URL>` on its own line. Select **Save**.
7. Do not put the URL in Infisical. The curl container does not authenticate to Infisical.
8. Set `CMS_PUBLIC_URL` when the default host is wrong.
9. On the **General** tab, select **Deploy**.
10. In the `web-deploy-hook` container log, confirm that the build started.
11. In Cloudflare, confirm that **Last triggered** changed and a new build ran.

Keep **Autodeploy** on in Dokploy. Each push then redeploys the CMS and starts the hook build.

The hook URL starts a build without other authentication. Treat it as a secret.
An empty value skips the build, so the service is safe before the hook exists.
The check uses the public URL, so it also confirms the proxy route.
A push-triggered frontend build can still fail. The hook build then publishes the current content.

## Verify the deployment

Perform these checks without printing secret values:

1. Authenticate with the machine identity and list secret names from the
   configured project, environment, and path.
2. Deploy the Compose application and confirm the PHP service exits on missing
   or invalid bootstrap credentials instead of starting with empty config.
3. Confirm HTTP redirects to HTTPS and HTTPS returns a valid certificate.
4. Inspect the HTML response and confirm stylesheet and script URLs use
   `https://`; request representative `/build`, `/vendor/statamic`, and
   `/assets` URLs directly.
5. Open the Statamic Control Panel and exercise login, redirects, and passkey
   endpoints to catch forwarded-scheme errors.
6. Send an authenticated GraphQL probe when GraphQL is enabled. Confirm the
   unauthenticated behavior matches the intended policy.
   Use the frontend's HTTPS endpoint and HTTP client for the probe.
7. Restart both services and, separately, redeploy from a clean image to verify
   volume recovery and backups.

Changing a secret in Infisical does not update an already-running PHP process.
Restart or redeploy the PHP service to fetch the new value.

## Rotate Universal Auth credentials

1. Create a replacement client secret without revoking the active one.
2. Update `INFISICAL_CLIENT_SECRET` in Dokploy.
3. Redeploy and complete the authentication and application smoke checks.
4. Revoke the old client secret.

This order avoids making the CMS unavailable during rotation.

## References

- [Infisical Universal Auth](https://infisical.com/docs/documentation/platform/identities/universal-auth)
- [Infisical CLI reference](https://infisical.com/docs/cli/reference)
- [Dokploy Docker Compose environment](https://docs.dokploy.com/docs/core/docker-compose#environment)
- [Laravel trusted proxies](https://laravel.com/docs/13.x/requests#configuring-trusted-proxies)
