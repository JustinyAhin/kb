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

1. Create a dedicated organization machine identity for the CMS. Give it no
   organization-level access.
2. Add the identity to the Infisical project with a read-only project role.
   Prefer a custom role restricted to the production environment and CMS
   folder when the plan supports that scope.
3. Enable Universal Auth and create a client secret. Record the Universal Auth
   client ID and the new client secret; the machine identity ID is not the
   client ID.
4. When IP allowlisting is available, restrict both the client secret and the
   issued access token to the VPS public IP as a `/32`. This is an optional paid
   feature and should not block deployment on plans that do not include it.
5. Store the client ID and client secret in the team password manager. Keep
   them separate from the Dokploy API credential so each can be rotated and
   shared independently.

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

Install the Infisical CLI in the PHP runtime image. Disable its update check so
container startup remains quiet and deterministic:

```dockerfile
RUN apk add --no-cache bash curl \
    && curl -1sLf 'https://dl.cloudsmith.io/public/infisical/infisical-cli/setup.alpine.sh' | bash \
    && apk add --no-cache infisical

ENV INFISICAL_DISABLE_UPDATE_CHECK=true
```

Use an entrypoint that fails when required bootstrap values are missing,
exchanges Universal Auth credentials for a short-lived token, removes the
client credentials from the child-process environment, and starts Statamic
through `infisical run`:

```sh
#!/bin/sh
set -eu

: "${INFISICAL_CLIENT_ID:?INFISICAL_CLIENT_ID must be set}"
: "${INFISICAL_CLIENT_SECRET:?INFISICAL_CLIENT_SECRET must be set}"
: "${INFISICAL_PROJECT_ID:?INFISICAL_PROJECT_ID must be set}"

INFISICAL_ENVIRONMENT="${INFISICAL_ENVIRONMENT:-prod}"
INFISICAL_SECRET_PATH="${INFISICAL_SECRET_PATH:-/cms}"
INFISICAL_DOMAIN="${INFISICAL_DOMAIN:-https://app.infisical.com/api}"

export INFISICAL_TOKEN
INFISICAL_TOKEN="$(
    infisical login \
        --method=universal-auth \
        --client-id="$INFISICAL_CLIENT_ID" \
        --client-secret="$INFISICAL_CLIENT_SECRET" \
        --plain \
        --silent
)"

unset INFISICAL_CLIENT_ID INFISICAL_CLIENT_SECRET

exec infisical run \
    --projectId="$INFISICAL_PROJECT_ID" \
    --env="$INFISICAL_ENVIRONMENT" \
    --path="$INFISICAL_SECRET_PATH" \
    --silent \
    -- \
    /var/www/html/docker-start.sh "$@"
```

The final startup script should prepare writable directories, clear stale
Laravel configuration and view caches, warm Statamic's Stache, and then `exec`
PHP-FPM. Never print the injected environment or enable shell tracing in these
scripts.

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

## Serve assets and trust the HTTPS proxy

Build frontend assets in a separate image stage and copy the complete
application `public` tree into both the PHP image and the Caddy image. Otherwise
Caddy can return the HTML response while CSS, JavaScript, Statamic vendor files,
or uploaded assets return `404`.

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
- [Infisical CLI `run`](https://infisical.com/docs/cli/commands/run)
- [Dokploy Docker Compose environment](https://docs.dokploy.com/docs/core/docker-compose#environment)
- [Laravel trusted proxies](https://laravel.com/docs/13.x/requests#configuring-trusted-proxies)
