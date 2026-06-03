# Building an Executor

An **executor** is a small, stateless HTTP service you deploy to act on the outside world on Rekor's
behalf. Rekor is the system of record: it holds documents, brokers credentials, and dispatches signed
requests. The executor receives those requests, does the real work (call a third-party API, present a
client certificate, run logic Rekor can't), and writes the result back.

Use an executor when an agent needs a side effect Rekor itself can't perform — anything beyond storing
and querying data: calling an external API on a schedule or on data change, talking to a legacy system,
presenting a certificate, or running heavier processing.

```
Agent → Rekor (record + sign + dispatch) → Executor (verify + act) → result back to Rekor
```

Rekor reaches an executor two ways, both signed identically:
- **Triggers** — Rekor POSTs to the executor's URL when documents change (outbound webhooks).
- **External sources** — Rekor proxies a document read/write through to the executor's URL.

## The one rule: never hand-roll verification — use `rekor-sdk`

Every dispatched request is HMAC-signed and carries a timestamp and an idempotency key. Verifying that
correctly (constant-time compare, timestamp skew, signature-list rotation, exact body bytes) is easy to
get subtly wrong, and a mistake means anyone can forge a request to your executor. The `rekor-sdk`
package does it for you and is the only supported path:

```ts
import { createExecutor, toFetchHandler, ExecutorError } from 'rekor-sdk'

const handler = toFetchHandler(createExecutor({
  secret: process.env.REKOR_SIGNING_SECRET!, // the source/trigger signing secret
  handler: async (ctx) => {
    // ctx.body is the verified raw payload; ctx.idempotencyKey dedupes retries automatically.
    const event = JSON.parse(ctx.body)
    // ...do the work (call the upstream, transform, etc.)...
    // Return a result to send back, or nothing for a 204.
    return { status: 200, body: JSON.stringify({ ok: true }), contentType: 'application/json' }
    // Throw `new ExecutorError({ status, code, message, retriable })` to control retry behavior:
    // retriable → Rekor retries with backoff; non-retriable 4xx → Rekor stops.
  },
}))

// Wire `handler` to your framework's route:
//   Hono:       app.post('/rekor', (c) => handler(c.req.raw))
//   Cloudflare: export default { fetch: handler }
//   Next.js:    export const POST = (req: Request) => handler(req)
```

For Node/Express, use the Node adapter instead:

```ts
import express from 'express'
import { createExecutor, toNodeHandler } from 'rekor-sdk'

const rekor = toNodeHandler(createExecutor({ secret: process.env.REKOR_SIGNING_SECRET!, handler: async (ctx) => { /* ... */ } }))
const app = express()
app.post('/rekor', rekor) // mount BEFORE any JSON body parser — the adapter needs the raw bytes
```

The SDK gives you, for free: signature + timestamp verification, automatic dedupe on the idempotency
key (so an at-least-once delivery never double-acts), and a normalized error envelope. You write one
`handler` function.

## What the executor MUST do (the contract)

- **Verify** every request (the SDK does this — reject unsigned/invalid/stale).
- **Scope** all work to the signed request — never trust a tenant id from the request body.
- **Dedupe** on the idempotency key (the SDK does this with the default in-memory store; supply a shared
  store if you run multiple instances).
- **Stay stateless** — keep no durable per-tenant state. Credentials and records live in Rekor; the
  executor holds at most an ephemeral in-memory cache.
- **Return the normalized error envelope** `{ status, code, message, retriable }` (throw `ExecutorError`).
- **Keep secrets and PII out of logs.**
- **Write results back** — return them inline (sync), or POST to a Rekor hook (async).

## Where to run it

**Default to a Cloudflare Worker.** Most executors are just authenticated API calls, and a Worker is
cheap, fast, globally deployed, and uses the SDK's Fetch handler directly.

**Reach for Fly.io (or any container host) only when a Worker can't do the job:**
- **Mutual-TLS / client certificates** (e.g. a bank or government API that requires an A1 cert)
- **Long-running** work beyond a Worker's CPU/time budget
- **Raw TCP, SOAP, or other non-HTTP protocols**
- **Native dependencies** (heavy crypto, language runtimes a Worker can't host)

These are exactly the cases Rekor deliberately can't handle itself — so they live in the executor.

## Certificate-bearing integrations (the vault pattern)

When an upstream requires a client certificate, store it in Rekor's vault (base64-encoded) with a
`content_type` so it's a first-class, rotatable credential — not a file baked into the executor:

```bash
# Store the cert as a credential-grade secret. `--file` reads the .p12 / .pem and base64-encodes it;
# `--content-type` records the format so consumers know how to use it.
rekor secrets create --name partner-cert --file ./partner.p12 --content-type application/x-pkcs12
```

Rekor injects vault secrets into proxied requests, or the executor fetches the secret at startup and
presents it over mutual-TLS (on Fly.io, since a Worker can't). The cert stays in Rekor — the executor
remains stateless and the credential rotates centrally (rotate installs a new value you supply; it is
never auto-generated).

## Local development

Set `devBypass: true` to skip signature verification while developing locally (dedupe still applies if
the headers are present). **Never enable it in production** — it disables the only thing stopping a
forged request.

```ts
createExecutor({ secret, handler, devBypass: process.env.NODE_ENV !== 'production' })
```

## Idempotency and retries

Delivery is at-least-once: Rekor retries failed deliveries with backoff and surfaces status via
`rekor triggers deliveries`. Because the same event can arrive more than once, the SDK dedupes on the
idempotency key automatically — a replayed delivery returns the cached result without re-running your
handler. If you run more than one executor instance, pass a shared `store` (implement the
`IdempotencyStore` interface over your datastore) so dedupe holds across instances.
