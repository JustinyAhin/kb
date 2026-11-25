# PostHog in a SvelteKit app

Use this runbook to add PostHog product analytics to a SvelteKit app on Cloudflare Workers.
It covers the browser SDK, the ingestion proxy, custom events, error reports, and privacy filters.
To query the data after setup, use [PostHog MCP](../../operations/posthog-mcp.md).

It was written for SvelteKit 2 and 3, Svelte 5, and `posthog-js` 1.x.
All names below are placeholders.

## Prerequisites

- A PostHog project for the app. Record its name, ID, and timezone in the app's own KB.
- The project key (`<posthog-project-key>`, starts with `phc_`). It is public, but keep it out of this KB.
- The app's domain on Cloudflare.

## 1. Add the ingestion proxy

Send events through a subdomain of the app. Ad blockers block the default PostHog hosts.

1. In PostHog, open **Settings → Managed reverse proxy**. Add `<proxy-subdomain>.<app-domain>`.
2. Add the CNAME record that PostHog shows. Use DNS only (grey cloud) in Cloudflare.
3. Wait until PostHog shows the proxy as live.

Use a short neutral subdomain, for example `t`. Names such as `analytics` or `posthog` are on block lists.

Expected result: `https://<proxy-host>/flags/?v=2` returns HTTP 200.

## 2. Add the variables

| Variable              | Value                   |
| --------------------- | ----------------------- |
| `PUBLIC_POSTHOG_KEY`  | `<posthog-project-key>` |
| `PUBLIC_POSTHOG_HOST` | `https://<proxy-host>`  |

- Read them through `$env/static/public`. The build puts them into the browser bundle.
- Put both in the build variables, not only in the Worker runtime variables. See [Workers deploy](../cloudflare-workers/deploy.md).
- A change to a value needs a rebuild and a deploy.
- Add both names to `.env.example` with empty values.
- Use one name for the key. A different name in the build settings disables PostHog with no error.

Load values with [Varlock and 1Password](environment-variables-with-varlock-and-1password.md) or [Infisical](../../operations/environment-variables-with-infisical.md#cloudflare-workers-and-hosted-builds).

## 3. Install the SDK

```sh
bun add posthog-js
```

## 4. Make one analytics module

Keep all PostHog calls in `src/lib/client/analytics/index.ts`. Other code imports only its helpers.

```ts
import posthog from "posthog-js";

import { browser, dev } from "$app/environment";

import { normalizeEventUrls } from "./normalize-event-urls";

let initialized = false;

const initPostHog = ({
  apiKey,
  apiHost,
}: {
  apiKey: string;
  apiHost: string;
}): void => {
  if (!browser || dev || initialized) return;

  posthog.init(apiKey, {
    api_host: apiHost,
    ui_host: "https://us.posthog.com",
    defaults: "<latest-defaults-date>",
    capture_pageleave: true,
    disable_capture_url_hashes: true,
    before_send: normalizeEventUrls,
  });

  initialized = true;
};

const capture = (event: string, properties?: Record<string, unknown>): void => {
  if (!browser || !initialized) return;

  posthog.capture(event, properties);
};

const captureException = (opts: {
  error: unknown;
  status?: number;
  message?: string;
}): void => {
  if (!browser || !initialized) return;

  const err =
    opts.error instanceof Error ? opts.error : new Error(String(opts.error));

  posthog.captureException(err, { status: opts.status, message: opts.message });
};

export { initPostHog, capture, captureException };
```

Configuration rules:

- `defaults`: use the latest date in the [PostHog JS configuration docs](https://posthog.com/docs/libraries/js/config). It sets `capture_pageview: 'history_change'`.
- Do not set `capture_pageview: true`. It records only the first page load. SvelteKit navigations do not reload the page.
- `ui_host`: use the PostHog app host for the project region, `https://us.posthog.com` or `https://eu.posthog.com`. It is not the ingestion host.
- `dev` stops collection in `bun run dev`. A missing key also stops collection.
- Autocapture, persistence, and session recording use SDK defaults and project settings. Change them in PostHog first.

## 5. Start the SDK and report errors

Start the SDK in `src/hooks.client.ts`. This file runs only in the browser. The `init` hook needs SvelteKit 2.10 or later.

```ts
import type { ClientInit, HandleClientError } from "@sveltejs/kit";
import { PUBLIC_POSTHOG_HOST, PUBLIC_POSTHOG_KEY } from "$env/static/public";

import { captureException, initPostHog } from "$lib/client/analytics";

export const init: ClientInit = () => {
  if (PUBLIC_POSTHOG_KEY)
    initPostHog({ apiKey: PUBLIC_POSTHOG_KEY, apiHost: PUBLIC_POSTHOG_HOST });
};

export const handleError: HandleClientError = ({ error, status, message }) => {
  captureException({ error, status, message });

  return { message };
};
```

`$env/static/public` fails the build when a name is not declared. Keep both names in `.env.example` and the build variables.

## 6. Remove private data from URLs

Query strings and hashes can hold user input. Strip them from every URL field before PostHog sends the event.
Put this in `src/lib/client/analytics/normalize-event-urls.ts`:

```ts
import type { BeforeSendFn } from "posthog-js";

const urlFields = [
  "$current_url",
  "$referrer",
  "$session_entry_url",
  "$session_entry_referrer",
  "$initial_current_url",
  "$initial_referrer",
];

const normalizeUrlFields = (properties: unknown): void => {
  if (
    !properties ||
    typeof properties !== "object" ||
    Array.isArray(properties)
  )
    return;

  const values = properties as Record<string, unknown>;

  for (const field of urlFields) {
    const value = values[field];
    if (typeof value === "string") values[field] = value.split(/[?#]/, 1)[0];
  }
};

const normalizeEventUrls: BeforeSendFn = (event) => {
  if (!event) return event;

  for (const properties of [event.properties, event.$set, event.$set_once]) {
    normalizeUrlFields(properties);
  }

  return event;
};

export { normalizeEventUrls };
```

- The filter removes UTM parameters from URLs. PostHog still keeps them as separate `utm_*` properties.
- `$web_vitals` events keep URLs inside nested metric objects. Clean those fields too when web vitals are on.
- Skip this filter only when URLs never hold user data.

For session recordings:

- Use `session_recording.maskCapturedNetworkRequestFn` to drop request bodies that hold personal data.
- Set `disable_session_recording: true` on pages that show private content.
- Call `posthog.stopSessionRecording()` in `beforeNavigate` before the app goes to such a page.

## 7. Send custom events

Name custom events `<app>:<action>`. Put the variant in properties, for example `action`, `placement`, and `kind`.

Components do not import PostHog. They send a browser event:

```ts
window.dispatchEvent(
  new CustomEvent("<app>:share", {
    detail: { action: "copy", placement: "result", kind: "result" },
  }),
);
```

The analytics module listens for the event. It checks each field against an allowlist, then calls `capture`.
Fields that are not on the allowlist do not go to PostHog. This keeps user input out of events.

```ts
const actions = new Set(["copy", "save", "native"]);

const listenForShareEvents = (): (() => void) => {
  if (!browser || !initialized) return () => {};

  const onShare = (event: Event) => {
    if (!(event instanceof CustomEvent)) return;

    const { action } = (event.detail ?? {}) as Record<string, unknown>;
    if (typeof action !== "string" || !actions.has(action)) return;

    capture("<app>:share", { action });
  };

  window.addEventListener("<app>:share", onShare);

  return () => window.removeEventListener("<app>:share", onShare);
};
```

Start the listener in the root `+layout.svelte` with `onDestroy(listenForShareEvents())`.
The root layout mounts before its children, so it catches events from child mount hooks.

Record each custom event, its emission point, and its meaning in the app's KB. [PostHog MCP](../../operations/posthog-mcp.md#understand-event-semantics-first) needs this record.

## 8. Server events (optional)

The steps above send events only from the browser. Use `posthog-node` only when an event must come from the server, for example a payment webhook.

- Make one client per request with `flushAt: 1` and `flushInterval: 0`.
- Pass the shutdown promise to `event.platform.context.waitUntil()`. A Worker can stop before a flush that is not awaited.
- Read the key from `event.platform.env`. See [Cloudflare Workers conventions](../cloudflare-workers/conventions.md).

## 9. Update the privacy page

Update the privacy page in the same release. Name PostHog, the data it collects, and its use of cookies and browser storage.

## 10. Verify

1. Build with the real key and host. Serve the build, not `bun run dev`.
2. Open a page. Go to a second page with a link.
3. In the network panel, find requests to `<proxy-host>`. They must return HTTP 200.
4. Check that each payload has no query string, no hash, and no user input.
5. After deploy, find the `$pageview` events with [PostHog MCP](../../operations/posthog-mcp.md).

Expected result: two `$pageview` events, one for each page.

PostHog drops events from automated browsers. A headless browser test can send requests but show no events in PostHog.
Use a normal browser for the final check.

If no request goes to `<proxy-host>`:

- Look for `PUBLIC_POSTHOG_KEY` in the deployed bundle. If it is absent, add it to the build variables and rebuild.
- Check the browser extensions. Ad blockers can block the proxy host too.
