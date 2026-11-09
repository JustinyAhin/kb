# Dokploy private services

Use this runbook to keep databases and internal services on a Dokploy server
off the public internet. Each client uses the shortest private path to the
service. No database needs a public port.

All names, addresses, and identifiers below are examples. Replace them before use.

## Choose the access path

| Client | Path | Example host and port |
| --- | --- | --- |
| A container on the same server | Docker network `dokploy-network` | `<db-app-name>:5432` |
| A Cloudflare Worker | Hyperdrive, Cloudflare Access, and Cloudflare Tunnel | Tunnel ingress to the server |
| Your computer or a script | Tailscale or an SSH tunnel | `runner-1:<port>` or `127.0.0.1:<local-port>` |

Do not use the server's public address from a container on the same server.
That traffic goes out of the container and back in through Docker. It skips
TLS on most setups and needs a published port.

## Prerequisites

- A Dokploy API key, read from a password manager. See
  [Dokploy volume backups](dokploy-volume-backups.md#prerequisites) for the
  read pattern.
- A private path to Dokploy, for example Tailscale.
- A provider firewall. See [Ubuntu VPS setup](vps-tailscale-setup.md#8-add-a-provider-firewall).

```bash
export DOKPLOY_URL="http://runner-1:3000"
export T="$(op read "op://<vault>/<item>/<field>")"
```

## Find the public ports

List the ports that listen on all addresses:

```bash
ssh admin@runner-1 "sudo ss -tlnpH | awk '{print \$4}' | grep -E '^(0\.0\.0\.0|\[::\]|\*):' | sort -u"
```

List the Dokploy databases that publish an external port:

```bash
curl -s -H "x-api-key: $T" "$DOKPLOY_URL/api/project.all" \
  | jq -r '.[] | .name as $p | .environments[] | .postgres[]? | "\($p) \(.postgresId)"' \
  | while read -r project id; do
      curl -s -H "x-api-key: $T" "$DOKPLOY_URL/api/postgres.one?postgresId=$id" \
        | jq -r --arg p "$project" '"\($p)  \(.appName)  external=\(.externalPort)"'
    done
```

The same pattern works for the `mysql`, `mariadb`, `mongo`, and `redis` keys.

Compose files can also publish ports. A line such as `- '3000'` publishes the
container port on a random public host port. Traefik does not need it. Traefik
reaches the container through `dokploy-network`.

## Find the clients before closing a port

A connection through a published port shows the Docker gateway as its source,
for example `172.18.0.1`. It does not show the real client. Use these sources:

- The application configuration: find the database host and port in each
  environment file. Print the host and port only. Never print the full URL.
- The live sockets on the server:

  ```bash
  sudo ss -tnpH state established '( sport = :5432 )'
  ```

- The database sessions:

  ```sql
  select client_addr, application_name, count(*)
  from pg_stat_activity
  where backend_type = 'client backend'
  group by 1, 2;
  ```

Live checks show only current connections. Scripts that run from time to time
do not show. Check the repositories and the scheduled jobs too.

## Move a same-server client to the internal name

1. Find the database `appName` in Dokploy. The service name on
   `dokploy-network` is the same as the `appName`.
2. Check that the client container can resolve it:

   ```bash
   docker exec <client-container> getent hosts <db-app-name>
   ```

3. In the client's database URL, change only the host and the port. Use
   `<db-app-name>:5432`. Keep the user, password, database, and parameters.
4. Deploy the client. If the URL is part of the image, build the image again.
5. Check that the database sessions now come from a `dokploy-network` address,
   not from the Docker gateway.

## Remove the external port

Do this only after all clients use a private path.

```bash
curl -s -X POST -H "x-api-key: $T" -H "content-type: application/json" \
  "$DOKPLOY_URL/api/postgres.saveExternalPort" \
  -d '{"postgresId": "<postgres-id>", "externalPort": null}'

curl -s -X POST -H "x-api-key: $T" -H "content-type: application/json" \
  "$DOKPLOY_URL/api/postgres.deploy" \
  -d '{"postgresId": "<postgres-id>"}'
```

The deploy restarts the database. The data stays in its named volume. Check
that the port no longer listens:

```bash
ssh admin@runner-1 "sudo ss -tlnH | grep ':<port> ' || echo closed"
```

For a database in a Compose service, remove its `ports:` entry. Then deploy
the Compose service again.

## Cloudflare Tunnel origins

A tunnel that runs as a host service reaches a database through a published
port on the host. Publish that port on the loopback address only:

```yaml
ports:
  - "127.0.0.1:5432:5432"
```

Alternatively, run `cloudflared` as a container on `dokploy-network`. Then set
the tunnel ingress to `tcp://<db-app-name>:5432` and remove the published port.

## Dokploy panel

The Dokploy panel listens on port `3000`. If you give the panel a domain,
Traefik also serves it on port `443`. The provider firewall does not block it.

Use one of these options:

- Remove the panel domain. Open the panel only through Tailscale.
- Put Cloudflare Access in front of the panel domain. Add a bypass rule for
  the deploy webhook paths if Git providers send webhooks to that domain.

## Verify

- Each public site responds through `80` and `443`.
- Each database port is closed from the internet.
- Each client application has no new connection errors in its logs.
- The database sessions come from the expected private paths.
