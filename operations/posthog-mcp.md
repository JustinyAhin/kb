# PostHog MCP

Use this when investigating product behavior, traffic changes, alerts, and time-series trends through the PostHog MCP.

## Confirm project context

Confirm or switch the active organization and project before querying. The MCP keeps project state between calls, so a previous task may have left another project active. Record the expected project name, ID, and timezone in the consuming project's local instructions.

## Efficient MCP workflow

PostHog MCP exposes one command wrapper. Use the smallest necessary sequence:

1. `search <term>` only when the tool name is unknown.
2. `info <tool>` only when its input schema is not already known.
3. `schema <tool> <path>` only when `info` returns a hint for a complex field.
4. `read-data-schema` to verify each event or property used in a query.
5. `call <tool> <json>` to run the query.

Do not call `tools` or repeat `info` after the schema is known. In particular, `info execute-sql` returns a very large reference document. Its commonly used input is only:

```json
{
  "query": "<HogQL>",
  "truncate": true
}
```

Known command shapes:

```text
call read-data-schema {"query":{"kind":"events","limit":100}}
call read-data-schema {"query":{"kind":"event_properties","event_name":"$pageview"}}
call read-data-schema {"query":{"kind":"event_property_values","event_name":"$pageview","property_name":"$search_engine"}}
call execute-sql {"query":"SELECT ..."}
```

The MCP requires schema verification even for PostHog system events. Verify only what the analysis needs; broad schema dumps waste time and tokens.

## Understand event semantics first

An event name rarely proves where it is emitted or what prerequisites it represents. Before treating an application event as an attempt, success, or failure:

1. Find its emission point in the application.
2. Note early returns, redirects, deduplication, and writes before the event.
3. Record the semantics in the consuming project's local knowledge base.

A zero count for a completion event does not prove an endpoint outage. It can also mean no traffic, rejected inputs, reused records, an earlier failure, or changed product behavior.

## Time-window rules

- Use explicit timestamps in the PostHog project's timezone.
- Compare only completed hours.
- Compare both the previous day and the same weekday one week earlier.
- Put the timestamp bound in `WHERE`; a condition inside `countIf` does not limit the scan.
- Keep every `events` query tightly time-bounded.

At 05:37 in the project timezone, compare hours `00:00–04:59` with:

```sql
AND toHour(timestamp) < 5
```

Do not include the partially completed 05:00 hour.

## Standard traffic and conversion comparison

Use one scan with conditional aggregates:

```sql
SELECT
  toDate(timestamp) AS day,
  countIf(event = '$pageview') AS pageviews,
  uniqIf(person_id, event = '$pageview') AS unique_visitors,
  countIf(event = '<conversion_event>') AS conversions
FROM events
WHERE timestamp >= toDateTime('<start>')
  AND timestamp < toDateTime('<end>')
  AND toHour(timestamp) < <latest_completed_hour>
  AND event IN ('$pageview', '<conversion_event>')
GROUP BY day
ORDER BY day
```

Use `person_id`, not `distinct_id`, when counting unique people.

## Hourly incident timeline

Find the first bad hour and the start of recovery:

```sql
SELECT
  toDate(timestamp) AS day,
  toHour(timestamp) AS hour,
  countIf(event = '$pageview') AS pageviews,
  uniqIf(person_id, event = '$pageview') AS unique_visitors,
  countIf(event = '<conversion_event>') AS conversions
FROM events
WHERE timestamp >= toDateTime('<start>')
  AND timestamp < toDateTime('<end>')
  AND event IN ('$pageview', '<conversion_event>')
GROUP BY day, hour
ORDER BY day, hour
```

Exclude the current incomplete hour before interpreting the output.

## Acquisition-source breakdown

Verify `$search_engine`, `$referring_domain`, and their current values before using them:

```sql
SELECT
  toDate(timestamp) AS day,
  coalesce(properties.$search_engine, 'direct_or_other') AS source,
  count() AS pageviews
FROM events
WHERE timestamp >= toDateTime('<start>')
  AND timestamp < toDateTime('<end>')
  AND toHour(timestamp) < <latest_completed_hour>
  AND event = '$pageview'
GROUP BY day, source
ORDER BY day, pageviews DESC
```

`direct_or_other` means PostHog did not classify a search engine. It can include direct visits and unclassified referrals; do not label it strictly as direct traffic.

Use `properties.$virt_traffic_type` when separating regular visitors from bots or automation, after verifying the property and its current values.

## Alert investigation checklist

For a low-conversion or low-activity alert, check:

1. The alerted completion event.
2. `$pageview` volume and unique visitors.
3. Hourly timing of the decline.
4. Acquisition-source breakdown.
5. Attempt or upstream activity, if its relationship to completion is verified.
6. Server request status and duration for the exact route.
7. Infrastructure request counts for the same time window.

Interpret the combinations:

- Pageviews down and completion rate stable: acquisition or traffic problem.
- Pageviews stable and completions down: product conversion or behavior problem.
- Attempts stable and completions down: application processing changed.
- Server failures up: application or dependency incident.
- PostHog down but infrastructure stable: tracking or client-side delivery problem.

## HogQL efficiency and safety

- Use `countIf` and `uniqIf` to compare several events in one scan.
- Never select the full `properties` object; select individual keys.
- Avoid separate queries for metrics sharing the same timeframe and grouping.
- Keep high-cardinality URL and query-string groupings narrowly bounded.
- Add `LIMIT` to exploratory result sets.
- Personal API keys do not support `OFFSET`; use timestamp-based keyset pagination.
- Prefer typed `query-*` tools for standard PostHog insights.
- Use `execute-sql` for aligned multi-event comparisons and custom aggregations.
- Do not create or save an insight unless persistent PostHog changes were requested.

## Cross-check with infrastructure

PostHog reports captured behavior; it is not the source of truth for whether an application received a request. Cross-check traffic incidents with server or edge request logs. A tracking drop with stable infrastructure traffic points to instrumentation or client-side delivery rather than acquisition.
