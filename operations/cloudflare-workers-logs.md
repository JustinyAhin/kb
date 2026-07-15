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
cf workers observability telemetry keys \
  --from <from-ms> \
  --to <to-ms> \
  --limit 200
```

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

## Time windows

Use UTC and compare equivalent completed windows. Convert RFC 3339 timestamps to the milliseconds expected by Workers Observability with Node:

```bash
node -e "console.log(Date.parse('2026-07-15T05:00:00Z'))"
```

After every query, verify that `run.timeframe.from` and `run.timeframe.to` match the requested values. Cloudflare can clamp a query at the retention boundary instead of returning an explicit error. Also inspect `sampleInterval`; a value greater than `1` means the result was sampled.

## Historical query workflow

`cf workers observability telemetry query` requires a saved `queryId`. Create a temporary saved query, run it, and delete it afterward.

Create the query with the OAuth token already managed by `cf`. This macOS example reads the CLI credential store without printing the token:

```bash
node - <<'NODE'
(async () => {
const fs = require('fs');
const raw = fs
  .readFileSync(`${process.env.HOME}/Library/Preferences/.cf/auth.jsonc`, 'utf8')
  .replace(/\/\/.*$/gm, '');
const { oauth_token } = JSON.parse(raw);
const account = '<account-id>';
const body = {
  name: `tmp-worker-debug-${Date.now()}`,
  description: 'temporary query for Worker debugging',
  parameters: {
    limit: 50,
    filterCombination: 'and',
    filters: [
      {
        key: '$workers.scriptName',
        operation: 'eq',
        type: 'string',
        value: '<worker-name>'
      },
      {
        key: '$workers.event.request.path',
        operation: 'eq',
        type: 'string',
        value: '<request-path>'
      }
    ]
  }
};
const res = await fetch(
  `https://api.cloudflare.com/client/v4/accounts/${account}/workers/observability/queries`,
  {
    method: 'POST',
    headers: {
      authorization: `Bearer ${oauth_token}`,
      'content-type': 'application/json'
    },
    body: JSON.stringify(body)
  }
);
console.log(await res.text());
})();
NODE
```

Copy `result.id`, then query events or invocations:

```bash
cf workers observability telemetry query \
  --query-id <query-id> \
  --view events \
  --timeframe-from <from-ms> \
  --timeframe-to <to-ms> \
  --limit 50
```

```bash
cf workers observability telemetry query \
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
cf workers observability telemetry query \
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
node - <<'NODE'
(async () => {
const fs = require('fs');
const raw = fs
  .readFileSync(`${process.env.HOME}/Library/Preferences/.cf/auth.jsonc`, 'utf8')
  .replace(/\/\/.*$/gm, '');
const { oauth_token } = JSON.parse(raw);
const account = '<account-id>';
const id = '<query-id>';
const res = await fetch(
  `https://api.cloudflare.com/client/v4/accounts/${account}/workers/observability/queries/${id}`,
  {
    method: 'DELETE',
    headers: { authorization: `Bearer ${oauth_token}` }
  }
);
console.log(await res.text());
})();
NODE
```

Verify cleanup:

```bash
cf workers observability queries list
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
const raw = fs
  .readFileSync(`${process.env.HOME}/Library/Preferences/.cf/auth.jsonc`, 'utf8')
  .replace(/\/\/.*$/gm, '');
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
