# `@hatched/sdk-js`

[![npm](https://img.shields.io/npm/v/@hatched/sdk-js.svg)](https://www.npmjs.com/package/@hatched/sdk-js)
[![Types](https://img.shields.io/badge/types-TypeScript-blue.svg)](https://www.typescriptlang.org/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](./LICENSE)

Official JavaScript/TypeScript SDK for the [Hatched](https://hatched.live) gamification API. Server-first, edge-runtime friendly, dual ESM + CJS, zero runtime dependencies.

## About this repository

This repo is the public home of the `@hatched/sdk-js` npm package — issue tracker, changelog, and reference docs. The SDK source ships to npm; you don't need to clone this repo to use it. Run `npm install @hatched/sdk-js` and follow the snippets below.

- **Found a bug or want a feature?** [Open an issue](https://github.com/hatched-live/hatched-sdk-js/issues).
- **Reference docs**: <https://docs.hatched.live/reference/sdk-js>
- **Changelog**: [CHANGELOG.md](./CHANGELOG.md)

## Install

```bash
pnpm add @hatched/sdk-js
# or
npm install @hatched/sdk-js
# or
yarn add @hatched/sdk-js
```

Node 18+ (native `fetch` + `AbortSignal.timeout`). Runs on Next.js App Router, Vercel Edge, Cloudflare Workers, Deno, and Bun.

## Two ways to authenticate

The SDK is **server-only by default** — secret keys (`hatch_live_*`, `hatch_test_*`) must never ship to a browser bundle. The SDK throws at construction if it detects a DOM environment. For browser use, there's a browser-safe variant: **publishable keys** (`hatch_pk_*`).

| Key             | Prefix                         | Where        | Can do                                                       |
| --------------- | ------------------------------ | ------------ | ------------------------------------------------------------ |
| **Secret**      | `hatch_live_*`, `hatch_test_*` | Server only  | Everything — events, coin earn/spend, webhooks, admin writes |
| **Publishable** | `hatch_pk_*`                   | Browser-safe | Read buddies, read operations, mint read-only embed tokens   |

Full decision tree + security model: [docs.hatched.live/concepts/auth-model](https://docs.hatched.live/concepts/auth-model).

### Server (secret key)

```ts
import { HatchedClient } from '@hatched/sdk-js';

const hatched = new HatchedClient({
  apiKey: process.env.HATCHED_API_KEY!, // never hard-code this
});

// full surface available
const egg = await hatched.eggs.create({ userId: 'user_42' });
await hatched.eggs.updateStatus(egg.eggId, 'ready');
const op = await hatched.eggs.hatch(egg.eggId);
const result = await hatched.operations.wait(op.operationId);

await hatched.events.send({
  eventId: 'evt_lesson_42_user_123',
  userId: 'user_123',
  type: 'lesson_completed',
  properties: { score: 94 },
});
```

### Browser (publishable key)

```ts
import { HatchedClient, PublishableKeyScopeError } from '@hatched/sdk-js';

const hatched = new HatchedClient({
  publishableKey: process.env.NEXT_PUBLIC_HATCHED_PK!, // safe in bundles
  baseUrl: process.env.NEXT_PUBLIC_HATCHED_API_BASE_URL, // set for staging
});

// reads work
const buddy = await hatched.buddies.get('bdy_abc');

// read-only embed token mint works (scoped, short-lived)
const embed = await hatched.embedTokens.create({
  buddyId: 'bdy_abc',
  userId: 'user_42',
  ttlSeconds: 900,
});

// mutations fail fast, no network round-trip
try {
  await hatched.events.send({ ... });
} catch (err) {
  if (err instanceof PublishableKeyScopeError) {
    // move this call to your backend with HATCHED_API_KEY
  }
}
```

Create publishable keys in the Hatched dashboard → **Developers → API keys → Create publishable key**. They're scoped: by default `read:operations`, `write:embed-tokens`.

## Error handling

Every error is a typed subclass of `HatchedError` and carries `code`, `statusCode`, `requestId`, and optional `details`.

```ts
import {
  HatchedError,
  RateLimitError,
  ValidationError,
  PublishableKeyScopeError,
} from '@hatched/sdk-js';

try {
  await hatched.events.send({ ... });
} catch (err) {
  if (err instanceof RateLimitError) {
    await sleep(err.retryAfter * 1000);
  } else if (err instanceof ValidationError) {
    console.error('validation failed:', err.details);
  } else if (err instanceof PublishableKeyScopeError) {
    console.error('endpoint requires a secret key');
  } else if (err instanceof HatchedError) {
    console.error(err.code, err.requestId, err.message);
  } else {
    throw err;
  }
}
```

Full [error code catalogue](https://docs.hatched.live/reference/error-codes).

## Retries and idempotency

- `GET` and `idempotent: true` `POST`s retry on network errors, `408`, `429`, and `5xx` with exponential backoff + jitter, up to `maxRetries` (default 3).
- `events.send` is idempotent on `eventId` — re-sending the same id returns the cached effect without re-applying rules.
- `4xx` (except the retryable set above) and `ValidationError`s surface immediately.

## Rate limits and request ids

```ts
await hatched.events.send({ ... });

console.log(hatched.getRateLimitInfo());
// { limit: 1000, remaining: 986, reset: 1735689600, retryAfter: undefined }

console.log(hatched.getLastRequestId());
// 'req_abc_123'
```

Paste the request id into any support ticket and we can look up the full trace.

## Cancellation

Every resource method accepts an optional `AbortSignal`:

```ts
const controller = new AbortController();
setTimeout(() => controller.abort(), 500);
await hatched.buddies.get(buddyId, controller.signal);
```

Combined with the internal timeout via `AbortSignal.any` — either source can cancel.

## Configuration

```ts
new HatchedClient({
  apiKey: process.env.HATCHED_API_KEY!, // or publishableKey
  // optional: hatch_test_* keys default to staging; live keys default to prod
  baseUrl: 'https://api.staging.hatched.live/api/v1',
  timeoutMs: 15_000,
  maxRetries: 3,
  fetch: customFetch, // edge-runtime override
  debug: false,
});
```

## Pairing with the browser widget

The SDK mints the short-lived browser tokens that the hosted widget consumes.
Use `widgetSessions.create()` for interactive widgets and `embedTokens.create()`
for read-only displays, then load `https://cdn.hatched.live/widget.js` and
mount widgets with `data-hatched-mount`.

```html
<script src="https://cdn.hatched.live/widget.js" data-session-token="SESSION_TOKEN" defer></script>
<div data-hatched-mount="buddy"></div>
```

Full widget install guide: <https://docs.hatched.live/guides/widget-integration>.

## Resources

| Resource                 | Methods                                                                                  | Secret key | Publishable key |
| ------------------------ | ---------------------------------------------------------------------------------------- | ---------- | --------------- |
| `hatched.eggs`           | `create`, `get`, `list`, `updateStatus`, `hatch`                                         | ✅         | ❌              |
| `hatched.buddies`        | `get`, `list`, `earn`, `spend`, `equip`, `evolve`, `rerenderAppearance`, `archive`, etc. | ✅         | ✅ read only    |
| `hatched.events`         | `send`, `sendBatch`                                                                      | ✅         | ❌              |
| `hatched.operations`     | `get`, `wait` (alias: `waitForCompletion`)                                               | ✅         | ✅              |
| `hatched.widgetSessions` | `create`, `revoke`                                                                       | ✅         | ✅ `create`     |
| `hatched.embedTokens`    | `create`                                                                                 | ✅         | ✅              |
| `hatched.webhooks`       | `list`, `create`, `delete`, `deliveries`, `replay`, `verifySignature`                    | ✅         | ❌              |

## Webhook signature verification

```ts
import { WebhooksResource } from '@hatched/sdk-js';

// node runtimes — uses node:crypto
const valid = WebhooksResource.verifySignature(
  rawBody,
  request.headers.get('hatched-signature')!,
  process.env.HATCHED_WEBHOOK_SECRET!,
);
if (!valid) return new Response('invalid signature', { status: 400 });
```

Header format: `t=<unix_ts>,v1=<hex HMAC-SHA256 over "${t}.${rawBody}">`. Sign over the **raw body bytes** — a JSON parse/stringify round-trip reorders keys and breaks the signature.

For Cloudflare Workers (no `node:crypto`), verify with Web Crypto manually — see [edge runtimes](https://docs.hatched.live/guides/edge-runtimes).

## Framework guides

- [Next.js App Router](https://docs.hatched.live/guides/nextjs-integration)
- [Express](https://docs.hatched.live/guides/express-integration)
- [Edge runtimes (CF Workers, Vercel Edge, Deno, Bun)](https://docs.hatched.live/guides/edge-runtimes)
- [Browser usage with publishable keys](https://docs.hatched.live/guides/browser-usage)
- [Troubleshooting](https://docs.hatched.live/guides/troubleshooting)

## Reporting issues

Please file bugs and feature requests at <https://github.com/hatched-live/hatched-sdk-js/issues>. Include:

- SDK version (`npm ls @hatched/sdk-js`)
- Runtime (Node version, edge runtime, browser)
- The `requestId` from the failing call (`hatched.getLastRequestId()`)
- A minimal repro

For security-sensitive disclosures, email **security@hatched.live** instead of opening a public issue.

## Docs

Full documentation: <https://docs.hatched.live>

- [Getting started](https://docs.hatched.live/guides/getting-started)
- [SDK reference](https://docs.hatched.live/reference/sdk-js)
- [HTTP API reference](https://docs.hatched.live/reference/http-api)
- [Error codes](https://docs.hatched.live/reference/error-codes)
- [Auth model (secret vs publishable keys)](https://docs.hatched.live/concepts/auth-model)
- [Changelog](./CHANGELOG.md)

## License

[MIT](./LICENSE)
