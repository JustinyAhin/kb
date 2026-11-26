# Cloudflare Workers Logs

Use this when debugging production Worker failures that already happened or comparing Worker request volume across historical windows. For live debugging, `wrangler tail` is usually simpler.

## Prerequisites

```bash
cf auth whoami
```

Confirm the token is valid and has Workers Observability scopes. Never print tokens or read project environment files to obtain credentials.

## Find fields

List telemetry fields around the incident window:

```bash
cf o11y telemetry keys \
  --from <from-ms> \
  --to <to-ms> \
  --limit 200
```

With `cf` v1.0.0-rc.3, use `cf o11y`. `cf observability` has only `tail` and
`usage-limit`, so `cf observability telemetry ...` fails with "Unknown commands".
(History: `cf` v0.6.0 used `cf observability telemetry ...`.)

Common fields include:

- `$workers.scriptName`
- `$workers.event.request.method`
- `$workers.event.request.path`
- `$workers.event.request.url`
- `$workers.event.response.status`
- `$metadata.service`
- `$metadata.message`
- `$metadata.error`
- `$metadata.requestId`
- `$metadata.trigger`

## Errors that the app catches

Errors that the app catches and saves do not show in Workers logs. An example is a Cloudflare Workflow step error stored on a task row. Read them from the app's own task or activity record.

## Time windows

Use UTC and compare equivalent completed windows. Convert RFC 3339 timestamps to the milliseconds expected by Workers Observability with Node:

```bash
node -e "console.log(Date.parse('2026-07-15T05:00:00Z'))"
```

After every query, verify that `run.timeframe.from` and `run.timeframe.to` match the requested values. Cloudflare can clamp a query at the retention boundary instead of returning an explicit error. Also inspect `sampleInterval`; a value greater than `1` means the result was sampled.

## One-off query workflow

Use an inline query for a single check. It saves nothing, so it needs no cleanup:

```bash
cf o11y telemetry query --body '{
  "queryId": "tmp-worker-debug",
  "view": "events",
  "limit": 100,
  "dry": true,
  "timeframe": { "from": <from-ms>, "to": <to-ms> },
  "parameters": {
    "filterCombination": "and",
    "filters": [
      {
        "key": "$workers.scriptName",
        "operation": "eq",
        "type": "string",
        "value": "<worker-name>"
      }
    ]
  }
}'
```

Read `events.events[]`. Each event has `timestamp`, `$workers.event.request`, `$workers.event.response.status` and `$metadata`. The response can omit `sampleInterval`. In that case, the response does not show whether Cloudflare sampled the result.

On 2026-11-17, with `cf` v0.6.0, saved-query creation (`cf o11y queries create`) returned HTTP 400 with no reason, while this inline query worked. Use the inline query first.

## Historical query workflow

For repeated comparisons, create a temporary saved query, pass its `id` to
`cf o11y telemetry query`, and delete it afterward.

Create the query through `cf` so the CLI handles authentication:

```bash
cf o11y queries create --body '{
  "name": "tmp-worker-debug",
  "description": "temporary query for Worker debugging",
  "parameters": {
    "limit": 50,
    "filterCombination": "and",
    "filters": [
      {
        "key": "$workers.scriptName",
        "operation": "eq",
        "type": "string",
        "value": "<worker-name>"
      },
      {
        "key": "$workers.event.request.path",
        "operation": "eq",
        "type": "string",
        "value": "<request-path>"
      }
    ]
  }
}'
```

Copy `id`, then query events or invocations:

```bash
cf o11y telemetry query \
  --query-id <query-id> \
  --view events \
  --timeframe-from <from-ms> \
  --timeframe-to <to-ms> \
  --limit 50
```

```bash
cf o11y telemetry query \
  --query-id <query-id> \
  --view invocations \
  --timeframe-from <from-ms> \
  --timeframe-to <to-ms> \
  --limit 20
```

`events` is best for scanning errors. `invocations` groups request logs by request ID and is better when a request has multiple console or error entries.

Keep the Worker filter even when the route appears unique, so a broad account query cannot pull logs from another Worker with a similar path.

## Compare request volume

Use `calculations` for equivalent-window comparisons:

```bash
cf o11y telemetry query \
  --query-id <query-id> \
  --view calculations \
  --timeframe-from <from-ms> \
  --timeframe-to <to-ms> \
  --ignore-series
```

The total count is in `calculations[0].aggregates[0].value`. Run the same saved query with different timeframes for an apples-to-apples comparison.

For broad activity checks, filter by Worker and request method first:

```js
filters: [
  {
    key: '$workers.scriptName',
    operation: 'eq',
    type: 'string',
    value: '<worker-name>'
  },
  {
    key: '$workers.event.request.method',
    operation: 'eq',
    type: 'string',
    value: 'POST'
  }
]
```

A broad POST count corroborates activity but is not automatically a feature-specific count. Add the exact route filter before attributing requests to a feature.

## Cleanup

Always delete the temporary query:

```bash
cf o11y queries delete <query-id> --force
```

Verify cleanup:

```bash
cf o11y queries list
```

Use a recognizable `tmp-` prefix. For repeated investigations, wrap creation and execution in a script with deletion in a `finally` block so a failed query cannot leave saved queries behind.

## Broader traffic incidents

Workers Observability, zone analytics, and product analytics answer different questions:

- Workers Observability shows Worker routes, statuses, invocations, and console errors.
- Cloudflare zone analytics shows all edge requests, pageviews, threats, and status/path groups.
- Product analytics shows visitors, referral sources, and application events.

Use at least two layers before concluding that traffic recovered.

### Zone analytics with GraphQL

Do not use `cf analytics dashboard get`; Cloudflare returns error `1015` because that API has been retired. Query the GraphQL API instead:

```bash
node - <<'NODE'
(async () => {
const fs = require('fs');
const raw = fs.readFileSync(
  `${process.env.HOME}/Library/Preferences/cloudflare/config/default.json`,
  'utf8'
);
const { oauth_token } = JSON.parse(raw);
const zone = '<zone-id>';
const query = `
  query {
    viewer {
      zones(filter: { zoneTag: "${zone}" }) {
        httpRequests1hGroups(
          limit: 1000
          filter: {
            datetime_geq: "<from-rfc3339>"
            datetime_lt: "<to-rfc3339>"
          }
        ) {
          dimensions { datetime }
          sum { requests pageViews threats }
        }
      }
    }
  }
`;
const res = await fetch('https://api.cloudflare.com/client/v4/graphql', {
  method: 'POST',
  headers: {
    authorization: `Bearer ${oauth_token}`,
    'content-type': 'application/json'
  },
  body: JSON.stringify({ query })
});
console.log(await res.text());
})();
NODE
```

Change only the two UTC timestamps when comparing equivalent windows. Zone totals include bots, static assets, and third-party traffic, so expect them to differ from human pageviews in product analytics.

### Audit configuration changes

Check deployments and Cloudflare configuration changes around the incident:

```bash
cf accounts logs audit list \
  --since <from-rfc3339> \
  --before <to-rfc3339> \
  --zone-id <zone-id> \
  --direction asc \
  --limit 1000
```

Compare the first bad interval with deployment, DNS, ruleset, cache purge, and tag-manager changes. A nearby change is correlation only; confirm it against request and error timing.
