# Dokploy Volume Backups

Use this when a service keeps its state in Docker named volumes rather than a managed database: SQLite, a flat-file CMS, uploaded assets, or any application whose data the *Backups* tab cannot reach. That tab produces SQL dumps for managed database services and does not apply here. **Volume Backups** is a separate tab and a separate API.

Only named volumes are supported. Bind mounts cannot be backed up this way, so a service that needs backups must use named volumes.

## Prerequisites

An S3-compatible destination configured under `/dashboard/settings/destinations`. Cloudflare R2 works; use region `auto`, endpoint `https://<account-id>.r2.cloudflarestorage.com`, and an **Object Read & Write** token scoped to the single bucket. Press **Test** until it passes before configuring anything else.

Scope the token to objects rather than admin: backups need writes and restores need reads, but neither needs the ability to create or delete buckets. Account tokens never expire unless revoked; a token with a TTL will silently stop backups when it lapses.

Read the API token from a password manager and never print it:

```bash
export DOKPLOY_URL="https://<dokploy-host>"
export T="$(op read "op://<vault>/<item>/token")"
```

## Prefer the API over the CLI

`dokploy volume-backups list` has returned `400` against instances where the equivalent API call succeeds. Use the API for anything operational.

The CLI is still the best reference for field names, because the API returns no schema:

```bash
dokploy volume-backups create --help
```

## Find the identifiers

Backups attach to a service and a destination, so collect three ids first.

```bash
curl -s -H "x-api-key: $T" "$DOKPLOY_URL/api/project.all" \
  | jq -r '.[] | "\(.projectId)  \(.name)"'

curl -s -H "x-api-key: $T" "$DOKPLOY_URL/api/project.one?projectId=<project-id>" \
  | jq -r '.environments[] | (.compose[]? | "compose \(.composeId) \(.appName)"),
                             (.applications[]? | "app \(.applicationId) \(.appName)")'

curl -s -H "x-api-key: $T" "$DOKPLOY_URL/api/destination.all" \
  | jq -r '.[] | "\(.destinationId)  \(.name)  \(.bucket)"'
```

Services are nested under `environments`, not directly on the project.

## Create a backup

One record per volume:

```bash
curl -s -X POST -H "x-api-key: $T" -H 'content-type: application/json' -d '{
  "name": "content",
  "volumeName": "<appName>_<volume>",
  "prefix": "<service>/<volume>/",
  "serviceType": "compose",
  "appName": "<appName>",
  "serviceName": "<compose-service>",
  "composeId": "<compose-id>",
  "destinationId": "<destination-id>",
  "cronExpression": "0 3 * * *",
  "keepLatestCount": 30,
  "turnOff": true,
  "enabled": true
}' "$DOKPLOY_URL/api/volumeBackups.create"
```

The fields that are easy to get wrong:

| Field | Notes |
| --- | --- |
| `volumeName` | The **full Docker volume name**. Compose volumes are `{appName}_{volumeName}`, so a compose volume declared as `content` is `myapp-ab12cd_content`. Passing the compose-level name produces a backup that fails at run time. |
| `serviceType` | One of `application`, `compose`, `postgres`, `mysql`, `mariadb`, `mongo`, `redis`, `libsql`. Set the matching id field (`composeId`, `applicationId`, …). |
| `serviceName` | For compose, the service to stop when `turnOff` is set. Pick the container that writes the volume. |
| `keepLatestCount` | Retention. Dokploy prunes to this many archives, so a bucket lifecycle rule is optional rather than required. |
| `turnOff` | Stops the container for the duration of the run. |

## Verify before trusting it

Trigger a run immediately rather than waiting for the schedule:

```bash
curl -s -X POST -H "x-api-key: $T" -H 'content-type: application/json' \
  -d '{"volumeBackupId":"<id>"}' "$DOKPLOY_URL/api/volumeBackups.runManually"
```

A `200` means the run was accepted, not that it succeeded. The API records no run status, so confirm in the bucket. Objects land at:

```
{appName}_{serviceName}/{prefix}/{volumeName}-{timestamp}.tar
```

Then download one archive and list it. A file of plausible size is not proof of contents — a misconfigured volume name can still produce an archive of the wrong tree.

When configuring several volumes for one service, create and verify **one** first. A wrong `volumeName` is then a single delete instead of several.

## Schedule around the container restarts

With `turnOff` enabled, each run stops and restarts the service. Backing up several volumes on the same cron expression means several restarts at once, so stagger them:

```
0 3 * * *    first volume
15 3 * * *   second volume
30 3 * * *   third volume
```

Leaving `turnOff` off avoids the downtime but risks a torn write if the service writes during the archive. For small archives, prefer the brief downtime at a quiet hour.

## Restore

Restores are destructive and fail if the target volume already exists or is in use. The sequence is: stop the service, remove the volume, then restore into `{appName}_{volumeName}` so the service picks it up on restart.

Run one drill into a throwaway volume name while nothing is at stake. An untested backup is not a backup, and the failure modes here — wrong volume name, wrong tree, unreadable archive — are all invisible until a restore.

## What this does not cover

Dokploy's own system backup saves only its configuration: projects, domains, environment variables, and Traefik settings. It does **not** include application volumes. Losing the host without volume backups loses the data regardless of how recent the system backup is.

Volume backups also do not replace version history. They capture state at a point in time with no diff and no attribution, so a service whose data should be reviewable needs a separate mechanism.
