---
name: rekor
description: |
  Set up and operate Rekor — a headless system of record for AI agents. Use when:
  installing the `rekor` CLI, authenticating, creating a database, defining the first
  collection, working in preview and promoting to production, querying documents via SQL,
  managing relationships, uploading file attachments, configuring hooks/triggers/batch
  operations, backing a collection with an external API (external sources), storing
  credentials in the secret vault, importing or exporting tool definitions across
  providers (OpenAI/Anthropic/Google/MCP), creating curated MCP Factory endpoints,
  scoping API tokens, or interpreting Rekor concepts (database, collection, document,
  relationship, preview/production, external_id).
---

# Rekor — Data Layer for AI Agents

You have access to the `rekor` CLI — the builder interface for Rekor. Use it to set up schemas, configure collections, test in preview environments, and promote to production. Production agents use MCP tools to read/write documents; external systems integrate via REST API. The CLI is your tool for designing and managing the data model.

## Agent Guidelines

- Drive Rekor through the **`rekor` CLI** when you have shell access. If you also have `mcp__rekor__*` tools in your toolset, prefer the CLI for setup and configuration; reserve MCP tools for production read/write operations.
- Only provide information from this skill, tool descriptions, or reference documentation. Do not invent URLs, paths, commands, or flags.
- Schema work (collections, hooks, triggers, MCP Factory endpoints) only happens in **preview** databases. Always create or use a preview before changing schemas.
- **Promotion is human-only.** When schema is ready, surface the exact `rekor databases promote` command and wait for the user to run it.
- Tokens are shown only **once on creation**. Always tell the user to copy and store it before doing anything else.
- Never auto-commit. Show the user `git diff` (when working with schema files) and wait for approval.

---

## First-Time Setup Walkthrough

When the user is starting from scratch, walk them through these steps. Skip any that are already done — check with `rekor databases list` before assuming.

### 1. Install the CLI

```bash
npm install -g rekor-cli
```

Verify with `rekor --version`. The CLI is published to npm as `rekor-cli`; the binary is `rekor`.

### 2. Authenticate

If the user has a Rekor account:

```bash
rekor login
```

Opens a browser to complete OAuth. For headless or CI use, pass an existing API key:

```bash
rekor login --token rec_xxx
```

If the user does **not** have an account yet, direct them to [rekor.pro](https://rekor.pro) — free plan includes 1,000 operations per month, no credit card required.

### 3. Create a database

Databases are top-level data containers. One per app, domain, or tenant.

```bash
rekor databases create <id> --name "<Display Name>" [--tags <comma-separated>]
```

Set `REKOR_DATABASE=<id>` in the user's shell to avoid passing `--database` on every command.

### 4. Create a preview environment

Schema changes (collections, hooks, triggers, MCP Factory endpoints) are blocked in production databases. Create a preview to do schema work:

```bash
rekor databases create-preview <id> --name "<preview-slug>"
```

The resulting preview database ID is `<id>--<preview-slug>`. Use that as `--database` for all schema commands.

### 5. Define the first collection

A collection is a JSON Schema that defines a document type. Define it in the preview database:

```bash
rekor collections upsert <slug> --database <id>--<preview-slug> \
  --name "<Display Name>" \
  --schema '{"type":"object","properties":{...},"required":[...]}'
```

For schemas longer than a line, write them to a file and pass `--schema @path/to/schema.json`.

### 6. Test in preview, then ask the user to promote

Write some test documents to the preview, query them, iterate until the schema is right:

```bash
rekor documents upsert <slug> --database <id>--<preview-slug> --data '{...}'
rekor sql "SELECT * FROM documents WHERE database_id = {database_id:String} AND collection = '<slug>' AND deleted = false" --database <id>--<preview-slug>
```

When ready, surface the promotion command for the user to run (you cannot promote yourself):

```bash
rekor databases promote <id> --from <id>--<preview-slug>
```

Recommend `--dry-run` first to preview the diff.

### 7. Connect production agents

Two ways to give production agents access:

**Scoped API token** — for direct REST/CLI access:

```bash
rekor tokens create --name "<agent-name>" \
  --grants '[{"scope":{"databases":["<id>"],"environments":["production"]},"permissions":["read:documents","write:documents"]}]'
```

The raw token (`rec_...`) is shown **once**. Copy it immediately.

**MCP Factory endpoint** — for purpose-built MCP tools per agent (recommended for LLM agents):

See the [MCP Factory](#mcp-factory-custom-mcp-endpoints) section below.

---

## Core Concepts

- **Collection**: A schema (JSON Schema) that defines a document type. No migrations — create at runtime.
- **Document**: A JSON document conforming to a collection's schema.
- **Relationship type**: A schema (like a collection, but for links) that defines a `rel_type`. It optionally validates a relationship's metadata against a JSON Schema and optionally restricts which collections it may connect. Must exist before any relationship of that type can be created.
- **Relationship**: A typed, directed link between two documents. Its `rel_type` must be a declared relationship type; its metadata is validated against that type's schema.

## Quick Start

### 1. Create a collection

```bash
rekor collections upsert invoices --database my-ws \
  --name "Invoices" \
  --schema '{"type":"object","properties":{"customer":{"type":"string"},"amount":{"type":"number"},"status":{"type":"string","enum":["draft","issued","paid"]}},"required":["customer","amount"]}'
```

### 2. Create a document

```bash
rekor documents upsert invoices --database my-ws \
  --data '{"customer":"Acme Corp","amount":5000,"status":"draft"}'
```

With external ID for idempotent upsert:

```bash
rekor documents upsert invoices --database my-ws \
  --data '{"customer":"Acme Corp","amount":5000}' \
  --external-id inv_123 --external-source billing
```

### 3. Query documents

```bash
rekor sql "SELECT data.invoice_number.:String as num, data.status.:String as status, arraySum(CAST(data.line_items[].amount, 'Array(Float64)')) as total FROM documents WHERE database_id = {database_id:String} AND collection = 'invoices' AND deleted = false ORDER BY total DESC" --database my-ws
```

### 4. Declare a relationship type, then link documents

A relationship type must exist before linking. Declare it once (a config write — do this in a preview environment, then promote):

```bash
rekor relationship-types upsert belongs_to --database my-ws \
  --description "Invoice belongs to a customer" \
  --source-collections '["invoices"]' --target-collections '["customers"]'
```

Then link:

```bash
rekor relationships upsert --database my-ws \
  --source invoices/rec_abc --target customers/rec_xyz \
  --type belongs_to
```

Omit `--schema` to allow any metadata; pass `--schema` to validate the relationship's `--data` against a JSON Schema.

### 5. Traverse relationships

```bash
rekor query-relationships invoices rec_abc --database my-ws \
  --type belongs_to --direction outgoing
```

---

## Full Command Reference

> Destructive `delete` commands prompt for confirmation in an interactive terminal. Pass `-y`/`--yes` to skip the prompt (it is also auto-skipped in non-interactive/CI contexts).

### Databases

```bash
rekor databases list [--tag <tag>]
rekor databases get <id>
rekor databases create <id> --name <name> [--description <desc>] [--tags <comma-separated>]
rekor databases rename <id> --name <new-name>   # display name only — the id/slug is immutable
rekor databases tag <id> --tags <comma-separated>
rekor databases delete <id>
rekor databases create-preview <prod-id> --name <preview-slug> [--description <desc>]
rekor databases list-previews <prod-id>
rekor databases promote <prod-id> --from <preview-id> [--dry-run] [--collections <ids>] [--triggers <ids>] [--hooks <ids>]
rekor databases promotions <prod-id>
rekor databases rollback <prod-id> --promotion <promotion-id>
```

Tags let you group databases (e.g., `client:acme,billing`). Filter with `--tag`. Promotion is **human-only** (see Environments). Promote selectively with `--collections`/`--triggers`/`--hooks` (omit to promote everything). `promotions` lists prior promotions; `rollback` reverts one by ID.

### Collections

```bash
rekor collections list --database <ws>
rekor collections get <id> --database <ws>
rekor collections upsert <id> --database <ws> --name <name> --schema <json|@file> [--description <desc>] [--icon <name>] [--color <hex>] [--sources <json|@file>]
rekor collections delete <id> --database <ws>
rekor collections history <id> --database <ws> [--limit <n>] [--offset <n>] [--diff]
```

`--icon`/`--color` are display hints for the inspection UI. `--sources` declares **external sources** — see the External Sources section below.

Deleting a collection also removes all of its documents (and any relationships pointing at them); a collection recreated with the same id starts empty.

### Documents

```bash
rekor documents upsert <collection> --database <ws> --data <json|@file> [--id <uuid>] [--external-id <id>] [--external-source <src>]
rekor documents get <collection> <id> --database <ws>
rekor documents query <collection> --database <ws> [--filter <json|@file>] [--sort <json>] [--fields <list>] [--limit <n>] [--offset <n>]
rekor documents delete <collection> <id> --database <ws>
rekor documents history <id> --database <ws> [--limit <n>] [--offset <n>] [--diff]
```

`history` returns the full change history for an entity — every version as a snapshot, who changed it, the operation, and when. Admin-only: available to organization owners/admins or a token granted `read:audit`; ordinary agent tokens are rejected. (Same `history` subcommand exists on `relationships`, `collections`, and `relationship-types`.)

**Filtering & search.** `query` (REST `GET /documents/<collection>`, MCP `query_documents`) takes a Filter DSL expression — a condition `{field, op, value}` or an `and`/`or` group of them. Operators: `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `in`, `not_in`, `like`, `ilike`, `is_null`, `is_not_null`, `has`, and **`search`**. This DSL is the only accepted form — Mongo-style shorthand (`{"data.field":"value"}`, `{"field":{"$in":[...]}}`) is **not** supported.

`search` matches a field by **approximate** value — use it when you have a near-correct string but not the exact stored one (e.g. a model name, a place, a plan name, a SKU with a possible typo). Results come back **ranked by closeness**, each carrying a `_search_score` (0–1, higher = closer):

```jsonc
// "find vehicles whose model is close to 'honda civic', only active ones"
{ "and": [
  { "field": "data.status",    "op": "eq",     "value": "active" },
  { "field": "data.car_model", "op": "search", "value": "honda civic" }
] }
```

Every field is searchable by default — `search` needs no setup. Exact filters and `search` compose in one query: put exact conditions alongside the `search` leg so the exact ones narrow the set and `search` ranks within it. To **tune** how a field is matched, set an optional `x-search` hint on that field in the collection schema:

```jsonc
"car_model": { "type": "string", "x-search": { "mode": "name" } }
```

- `mode: "name"` — short proper nouns (people, brands, plan/model names); favors close, prefix-aligned matches.
- `mode: "text"` — free-text / multi-word fields; matches on shared word fragments.
- `mode: "fuzzy"` — codes / SKUs / IDs where typos dominate.
- `threshold` (0–1) — minimum closeness to return; `searchable: false` — forbid searching a field (e.g. a large free-text blob you never want scanned).

Tuning is optional — unset fields use a sensible default. `x-search` hints apply to **top-level fields**; `search` still works on nested paths (e.g. `data.address.city`) using the default behavior, they just aren't individually tuned. Search runs against the latest synced data, so a document written a moment earlier may take a brief moment to appear.

**Sorting.** `--sort` takes a **JSON array** of terms, each `{"field":"data.<field>","direction":"asc"|"desc"}` — e.g. `--sort '[{"field":"data.created_at","direction":"desc"}]'`. Multiple terms sort lexically (first term primary). `direction` defaults to `asc` if omitted. A bare string like `data.name:asc` is **not** accepted. Omit `--sort` when using `search` to keep the relevance ranking.

**Partial vs full updates.** A full upsert (`manage_document` `upsert`, REST `PUT`) writes the **whole** document — send every field. To change just one field (flip a status, set a flag) without re-sending the rest, use a **partial update**: `manage_document` `patch`, the generated `update_<collection>` tool, or REST `PATCH .../documents/<collection>/<id>`. A partial update merges the provided fields over the stored document (top-level merge; nested objects are replaced wholesale), targets an existing document by id or `external_id`, and the merged result must still satisfy the collection schema. For a collection backed by an **external source**, the provided fields are forwarded to that upstream system, which decides how the update is applied — the merge-and-keep-the-rest guarantee applies to documents stored in Rekor.

**Cancellation & archival.** Documents are **updatable by default** — re-upserting the same `external_id` updates the existing document in place (idempotent upsert), and you can advance state freely (e.g. `scheduled → confirmed`). A collection can opt into stricter archival via its schema's `x-archival.mode`: `immutable` (write-once — any update is rejected) or `on_attribute` (updatable until a configured terminal value). A document can also be *cancelled* — a first-class state distinct from delete — via MCP (`manage_document` `cancel`) or REST (`POST .../documents/<collection>/<id>/cancel`); there is no CLI `cancel` subcommand. A cancelled document can no longer be updated. Documents may also be *archived* automatically (when they reach a terminal/immutable state, or after a long period of inactivity). An archived document stays readable and can still be cancelled, but it can't be deleted, and re-upserting by the same `external_id` creates a **new** document rather than updating the archived one. When querying with `rekor sql`, add `archived = false` to see only active documents (cancelled rows carry `cancelled = true`).

### Attachments

Store large or binary content (PDFs, images, files) as attachments on a document and reference them from the document data — document/relationship JSON is for structured data and is capped at ~1 MiB, so inlining base64 blobs is rejected. Filenames may contain paths for folder structure (e.g. `docs/guide.md`).

```bash
rekor attachments upload <collection> <id> --database <ws> --filename <name> [--content-type <mime>] [--file <path>]
rekor attachments url <collection> <id> --database <ws> --filename <name>
rekor attachments list <collection> <id> --database <ws> [--prefix <path>]
rekor attachments delete <collection> <id> <attachment-id> --database <ws>
```

`upload` with `--file` uploads the local file directly; without `--file`, `upload` returns a presigned URL you `PUT` the bytes to. `url` returns a **download** URL for fetching an existing attachment. `--prefix` filters `list` to a path prefix (e.g. `docs/`).

### SQL Query

Execute read-only SQL queries directly against database data. Supports filtering, aggregation, JOINs, CTEs, and array operations.

```bash
rekor sql "<query>" --database <ws> [--param key=value ...] [--file query.sql]
```

**Tables**: `documents`, `relationships`, `collections`, `relationship_types`, `databases`, `operations_log`, `organization`

The `organization` table exposes org-level metadata (e.g. plan/status) and requires an `{org_id:String}` predicate instead of `{database_id:String}`.

**Important**: Always include `database_id = {database_id:String}` and `deleted = false`. The `documents` table also carries `archived` and `cancelled` boolean columns — add `archived = false` to query only active documents. Queries always see the latest version of each row — the server handles deduplication.

**Accessing JSON fields**: Use `data.field.:Type` subcolumn syntax for the native JSON type. Use `CAST(data.field, 'Type')` when type-safe conversion is needed (e.g., integers stored as Int64 vs Float64).

**Examples**:

```bash
# Simple query
rekor sql "SELECT data.invoice_number.:String as num, data.status.:String as status FROM documents WHERE database_id = {database_id:String} AND collection = 'invoices' AND deleted = false" --database my-ws

# Aggregation
rekor sql "SELECT data.status.:String as status, count() as cnt FROM documents WHERE database_id = {database_id:String} AND collection = 'invoices' AND deleted = false GROUP BY status" --database my-ws

# Array aggregation (sum embedded line items)
rekor sql "SELECT data.invoice_number.:String as num, arraySum(CAST(data.line_items[].amount, 'Array(Float64)')) as total FROM documents WHERE database_id = {database_id:String} AND collection = 'invoices' AND deleted = false" --database my-ws

# Explode array elements with ARRAY JOIN
rekor sql "SELECT item.description.:String as item, sum(CAST(item.amount, 'Float64')) as revenue FROM documents ARRAY JOIN data.line_items[] as item WHERE database_id = {database_id:String} AND collection = 'invoices' AND deleted = false GROUP BY item ORDER BY revenue DESC" --database my-ws

# CTE joining documents with relationships
rekor sql "WITH inv AS (SELECT id, data.invoice_number.:String as num, arraySum(CAST(data.line_items[].amount, 'Array(Float64)')) as total FROM documents WHERE database_id = {database_id:String} AND collection = 'invoices' AND deleted = false), pay AS (SELECT target_id, sum(CAST(data.allocated, 'Float64')) as paid FROM relationships WHERE database_id = {database_id:String} AND rel_type = 'payment_for' AND deleted = false GROUP BY target_id) SELECT inv.num, inv.total, coalesce(pay.paid, 0) as paid, inv.total - coalesce(pay.paid, 0) as balance FROM inv LEFT JOIN pay ON pay.target_id = inv.id ORDER BY balance DESC" --database my-ws

# With parameters
rekor sql "SELECT * FROM documents WHERE database_id = {database_id:String} AND data.status.:String = {status:String} AND deleted = false" --database my-ws --param status=issued

# Fuzzy / approximate text match, ranked by closeness (power-user form of `documents query --filter {op:search}`).
# Fold case + accents on BOTH sides so "São" matches "sao"; jaroWinklerSimilarity suits short names.
rekor sql "SELECT *, jaroWinklerSimilarity(lowerUTF8(data.car_model.:String), lowerUTF8({q:String})) AS score FROM documents WHERE database_id = {database_id:String} AND collection = 'vehicles' AND deleted = false AND score >= 0.85 ORDER BY score DESC LIMIT 10" --database my-ws --param q='honda civic'
```

### Relationship Types

A relationship type defines a `rel_type` and is required before any relationship of that type can be created (a config operation — preview then promote). `--schema` (optional) validates relationship metadata; `--source-collections`/`--target-collections` (optional) restrict which collections it may connect.

```bash
rekor relationship-types upsert <rel_type> --database <ws> [--description <text>] [--schema <json>] [--source-collections <json>] [--target-collections <json>]
rekor relationship-types list --database <ws>
rekor relationship-types get <rel_type> --database <ws>
rekor relationship-types delete <rel_type> --database <ws>
rekor relationship-types history <rel_type> --database <ws> [--limit <n>] [--offset <n>] [--diff]
```

Deleting a relationship type also removes all relationships of that type.

### Relationships

```bash
rekor relationships upsert --database <ws> --source <col/id> --target <col/id> --type <type> [--id <id>] [--data <json>]
rekor relationships get <id> --database <ws>
rekor relationships delete <id> --database <ws>
rekor relationships history <id> --database <ws> [--limit <n>] [--offset <n>] [--diff]
rekor query-relationships <collection> <id> --database <ws> [--type <type>] [--direction outgoing|incoming|both] [--limit <n>] [--offset <n>]
```

The `--type` must be a declared relationship type. `--data` is validated against that type's schema.

### Hooks (inbound webhooks)

External systems push data into Rekor via hooks. Each hook provides a unique ingest URL.

```bash
rekor hooks create --database <ws> --name <name> --secret <hmac-secret> [--id <id>] [--collection-scope <comma-separated>]
rekor hooks list --database <ws>
rekor hooks get <id> --database <ws>
rekor hooks delete <id> --database <ws>
```

`--secret` is the HMAC shared secret the sender signs ingest requests with (required). Ingest accepts either **Signing v1** — `X-Rekor-Signature: v1,<hex>` over id+timestamp+method+path+body, with an `X-Rekor-Timestamp` Rekor checks for freshness (the same scheme triggers and proxied calls use, so one signer works both directions) — or a legacy body-only HMAC for older senders. A v1 delivery is idempotent: a duplicate (a retry or replay carrying the same `X-Rekor-Id`) replays the first response instead of writing again. `--collection-scope` restricts which collections the hook may write to (omit for all). Hooks can only be created/deleted in preview databases. Promote to production when ready.

### Triggers (outbound webhooks)

Triggers fire automatically when documents change, notifying external systems via HTTP POST.

```bash
rekor triggers create --database <ws> --name <name> --url <url> --secret <hmac-secret> --events <comma-separated> [--id <id>] [--collection-scope <comma-separated>] [--filter <json>]
rekor triggers list --database <ws>
rekor triggers get <id> --database <ws>
rekor triggers delete <id> --database <ws>
rekor triggers deliveries --database <ws> [--status <pending|delivered|failed|dead>] [--trigger-id <id>]
```

`--events` is a **comma-separated** list (not a JSON array). Valid events: `document.created`, `document.updated`, `document.deleted`, `document.cancelled`, `relationship.created`, `relationship.updated`, `relationship.deleted` — e.g. `--events document.created,document.updated`. (Bare names like `create`/`update` will silently never match.) `--secret` (required) is the HMAC signing secret receivers verify with. `--collection-scope` limits firing to specific collections (omit for all).

Add `--filter '<json>'` to fire only on documents matching a condition — the same filter DSL queries use, evaluated against the document (or relationship) being written, e.g. `--filter '{"field":"data.status","op":"eq","value":"paid"}'` fires only when `status` is `paid`. Combine conditions with `and`/`or` groups. A malformed filter is rejected at create time.

Triggers are HMAC-signed (`X-Rekor-Signature: v1,<hex>` over id+timestamp+method+path+body, with an `X-Rekor-Timestamp` the receiver checks for freshness — the same scheme proxied requests use) and carry `X-Rekor-Id`/`X-Rekor-Delivery-Id` for receiver dedupe. Delivery is reliable — failed attempts are retried with backoff and dead-lettered after repeated failure; inspect status with `rekor triggers deliveries`. By default, writes from hooks don't re-fire triggers (`skip_hook_writes: true`). Triggers can only be created/deleted in preview databases.

### Executors (acting on the outside world)

Rekor records, signs, and dispatches — but the actual outside-world action (calling a third-party API, presenting a client certificate, running logic Rekor can't) happens in an **executor**: a small, stateless HTTP service you deploy and point a trigger or external source at. Rekor signs every dispatched request; the executor verifies it, does the work, and writes the result back.

**Always receive these requests with the `rekor-sdk` package — never hand-roll signature verification** (a mistake lets anyone forge a request to your executor). The SDK verifies the signature + timestamp, dedupes retries on the idempotency key, and normalizes errors — you write one handler:

```ts
import { createExecutor, toFetchHandler } from 'rekor-sdk'
const handler = toFetchHandler(createExecutor({
  secret: process.env.REKOR_SIGNING_SECRET!,
  handler: async (ctx) => { /* ctx.body is verified; do the work, return a result or nothing */ },
}))
// Hono: app.post('/rekor', (c) => handler(c.req.raw))  •  Cloudflare: export default { fetch: handler }
```

**Where to run it:** default to a **Cloudflare Worker** — most executors are just authenticated API calls, and a Worker is cheap, fast, and globally deployed. Reach for **Fly.io (or any container)** only when a Worker can't do the job: mutual-TLS client certificates, long-running work, raw TCP/SOAP, or native dependencies.

For the full contract, framework variants, the certificate-via-vault pattern, and local dev, read `references/executors.md` (bundled alongside this skill).

### External Sources (back a collection with an external API)

A collection can declare one or more **external sources**. When a document operation carries an `external_source` that matches a configured source name, Rekor proxies that read/write to the external API (or an executor) instead of its own store — so an agent reads and writes `stripe`-backed invoices through the same `rekor documents`/MCP calls, with no separate integration code.

Configure sources as part of the collection schema (preview only, promoted like any schema change):

```bash
rekor collections upsert invoices --database my-ws--preview --name "Invoices" \
  --schema @schema.json --sources @sources.json
```

Each source defines:
- `name` — must equal the document's `external_source`.
- `auth` — header + value template; the secret can be inline or a `vault:<name>` reference.
- `field_mapping` — `to_external` / `to_rekor` maps between your schema and the upstream's field names (simple renames or rich rules with value maps, transforms, defaults).
- `get` / `list` / `create` / `update` / `delete` — per-operation endpoint templates (`url` with `{{external_id}}`, `{{query.*}}`, `{{data.*}}` tokens; `method`; `response_path` / `total_path` to extract the body).
- `injections` — extra per-request vault secrets placed into a header or body field (for upstreams needing their own credential).
- `executor_secrets` — named vault credentials an executor pulls at dispatch, for a binary or large per-tenant credential it can't take inline (e.g. an mTLS client certificate). Each `{name, secret_ref}` declares a `vault:<name>` reference (templating only `{{auth.org_id}}`/`{{auth.database_id}}`); each pull is short-lived and single-use, scoped to the calling database.
- `cache_ttl` (seconds) + `stale_if_error` — read-through caching, optionally serving the last-known value on a transient upstream failure.
- `signing` — opt into HMAC request signing when the source points at an executor (adds `X-Rekor-Signature` + `X-Rekor-Timestamp`); omit for third-party APIs.
- `timeout_ms` — per-request upstream timeout (default 10s, max 30s); a timeout surfaces as a transient error so `stale_if_error` can serve a cached value.
- `breaker` — per-source circuit breaker: `{failure_threshold, cooldown_ms}`. After `failure_threshold` consecutive transient failures (default 5) the source opens and short-circuits calls for `cooldown_ms` (default 30s) before a single half-open probe. Always on; this only tunes it.

URLs must be absolute `https` and tokens may appear only in the path/query (never the host) — an SSRF guard rejects otherwise.

### Batch (atomic)

Execute up to 1,000 operations atomically — all succeed or all fail:

```bash
rekor batch --database <ws> --operations '[
  {"type":"upsert_document","collection":"invoices","data":{"customer":"A","amount":100}},
  {"type":"upsert_document","collection":"invoices","data":{"customer":"B","amount":200}},
  {"type":"upsert_relationship","rel_type":"related_to","source_collection":"invoices","source_id":"id1","target_collection":"invoices","target_id":"id2"}
]'
```

Operation types: `upsert_document`, `delete_document`, `upsert_relationship`, `delete_relationship`, `upsert_collection`, `delete_collection`

### Provider Adapters

Import tool definitions from any LLM provider as collections, or export collections as tool definitions.

**Import** (creates collections from tool definitions):

```bash
# OpenAI
rekor providers import openai --database <ws> --tools '[{"type":"function","function":{"name":"create_invoice","parameters":{"type":"object","properties":{"customer":{"type":"string"},"amount":{"type":"number"}},"required":["customer"]}}}]'

# Anthropic
rekor providers import anthropic --database <ws> --tools '[{"name":"create_invoice","input_schema":{"type":"object","properties":{"customer":{"type":"string"}}}}]'

# MCP
rekor providers import mcp --database <ws> --tools '[{"name":"create_invoice","inputSchema":{"type":"object","properties":{"customer":{"type":"string"}}}}]'

# From file
rekor providers import openai --database <ws> --tools @tools.json
```

**Export** (get collections as tool definitions):

```bash
rekor providers export openai --database <ws>
rekor providers export anthropic --database <ws> --collections invoices,customers
rekor providers export mcp --database <ws> --output tools.json
```

**Import tool call** (create a document from provider-native format):

```bash
# OpenAI tool call → document
rekor providers import-call openai invoices --database <ws> \
  --data '{"arguments":{"customer":"Acme","amount":5000}}' \
  --external-id call_abc123 --external-source openai

# Anthropic tool call → document
rekor providers import-call anthropic invoices --database <ws> \
  --data '{"input":{"customer":"Acme","amount":5000}}'
```

Supported providers: `openai`, `anthropic`, `google`, `mcp`

### MCP Factory (custom MCP endpoints)

Create purpose-built MCP servers from your database collections. Each endpoint serves domain-specific tools — agents see `create_invoice`, `list_payments`, not generic Rekor operations.

```bash
# Create a curated MCP endpoint
rekor endpoints upsert invoicing-agent --database my-ws \
  --name "Invoicing Agent" \
  --tool "invoices:get,list" \
  --tool "payments:create,get,list" \
  --relationship "invoice_payment:list" \
  --batch "invoices:create,update" --batch "payments:create" --batch "invoice_payment:create" \
  --sql-query

# Get the MCP connection URL (add --transport sse for the SSE endpoint; default mcp)
rekor endpoints url invoicing-agent
# → https://mcp.rekor.pro/e/invoicing-agent/mcp

# List all endpoints
rekor endpoints list --database my-ws

# Get endpoint config (with resolved schemas)
rekor endpoints get invoicing-agent --database my-ws --resolved

# Delete an endpoint
rekor endpoints delete invoicing-agent --database my-ws
```

**Tool spec format**: `collection:op1,op2` (operations: `create`, `get`, `list`, `update`, `delete`)
**Relationship spec format**: `rel_type:op1,op2` (operations: `create`, `list`, `delete`)
**Batch spec format**: `collection_or_rel:op1,op2`

**Custom tool names and descriptions**: Use `--config` with full JSON for advanced control:

```bash
rekor endpoints upsert invoicing-agent --database my-ws --config '{
  "name": "Invoicing Agent",
  "tools": [
    {
      "collection": "invoices",
      "operations": ["get", "list"],
      "name_override": "search_invoices",
      "description_override": "Search invoices by customer, status, or date range",
      "filterable_fields": [
        { "field": "status" },
        { "field": "issued_at" },
        { "field": "customer", "match": "text" }
      ]
    },
    {
      "collection": "payments",
      "operations": ["create", "get", "list"],
      "description_override": "Manage payment documents for invoices"
    }
  ],
  "relationships": [
    {
      "rel_type": "invoice_payment",
      "operations": ["create", "list"],
      "name_override": "assign_payment",
      "description_override": "Link a payment to an invoice"
    }
  ],
  "sql_query": true
}'
```

Or from a file: `--config @endpoint.json`

**Typed filter params (`filterable_fields`)**: expose chosen fields of a collection as typed parameters on its generated `list` tool, derived from the collection schema — so the agent fills native arguments instead of writing a filter expression. Each field becomes a parameter shaped by its type:

- an `enum` or boolean field → an exact-match param (the agent can only pick a valid value)
- a number or date/date-time field → a range pair (`<field>_min`/`<field>_max`, or `<field>_after`/`<field>_before`)
- a string field → a `<field>_contains` substring param

Per field you may set `param` (rename the generated param), `match` (`exact` | `range` | `text` | `any_of` — `any_of` accepts a list, matching any) to override the inferred behavior, and `description`. Invalid fields (unknown, array- or object-typed, or name-clashing) are rejected when you save the endpoint — for an object field, expose a nested path (`address.city`) instead. The generic `filter` parameter stays on the tool as the escape hatch for anything the typed params can't express (OR / nesting); typed params and `filter` are combined with AND.

Connect agents to the endpoint URL with a token scoped to exactly one database. The agent sees only the tools you configured — fully domain-specific, no Rekor concepts.

Endpoints can only be created/modified in preview databases. Promote to production when ready.

**Which database serves the endpoint:** `mcp.rekor.pro/e/{slug}/mcp` resolves the endpoint from the database your **token** is scoped to. So:

- **Production:** promote the endpoint, then connect with a token scoped to the production database (`my-ws`).
- **Preview (sandbox testing):** connect with a token scoped to the **preview database id** (`my-ws--<preview-slug>`) to serve the not-yet-promoted endpoint.

If the slug can't be resolved for your token's database (unknown endpoint, or one that only exists in a preview you aren't scoped to), `initialize` returns a clear JSON-RPC error telling you to scope to the preview database id or promote — it won't silently hand back a session with zero tools.

### API Tokens

Create scoped tokens for agents, integrations, and CI/CD. Tokens can be restricted to specific databases, collections, environments, and permissions. Tokens are hashed before storage — the raw value is shown only once on creation.

```bash
# Create a full-access token
rekor tokens create --name "my-key" --grants '[{"scope":{"databases":["*"]},"permissions":["*"]}]'

# Create a scoped token (read-only on one database)
rekor tokens create --name "client-a-reader" \
  --grants '[{"scope":{"databases":["client-a"],"environments":["production"]},"permissions":["read:documents","read:collections"]}]'

# Create a token with expiration
rekor tokens create --name "temp-key" \
  --grants '[{"scope":{"databases":["*"]},"permissions":["*"]}]' \
  --expires-at 2026-12-31T23:59:59Z

# List tokens (shows status, last_used_at, expires_at)
rekor tokens list

# Revoke a token
rekor tokens revoke <token_id>
```

**Permissions**: `read:documents`, `write:documents`, `read:collections`, `write:collections`, `read:relationships`, `write:relationships`, `read:attachments`, `write:attachments`, `read:hooks`, `write:hooks`, `read:triggers`, `write:triggers`, `read:endpoints`, `write:endpoints`, `read:databases`, `write:databases`, `read:audit` (read-only; grants change-history access, admin-gated, not implied by other grants), or `*` for all.

**Scope fields**: `databases` (required), `collections` (optional — omit for all), `environments` (optional — `production`, `preview`, or omit for both).

Tokens enforce a privilege ceiling — you can only create tokens with equal or narrower scope than your own.

### Secrets (vault)

Store credentials at the organization level — API keys, signing secrets, or credential-grade blobs (a client certificate, keystore, or service-account JSON). Secrets are encrypted at rest and always masked in responses. Reference them from source/trigger config (e.g. an executor's signing secret) or have an executor read one at runtime.

```bash
# Store a string secret
rekor secrets create --name stripe-key --value sk_live_xxx --tags billing

# Store a credential file (base64-encoded) with its type, and an expiry — e.g. a client certificate
rekor secrets create --name partner-cert --file ./partner.p12 \
  --content-type application/x-pkcs12 --expires-at 2027-01-01T00:00:00Z

rekor secrets list                         # List (values masked)
rekor secrets list --expiring [--days 30]  # Only secrets expiring/expired within the window
rekor secrets get <id>                      # Metadata (value masked)
rekor secrets rotate <id> --value <new> [--expires-at <iso>]  # Install a new value (never auto-generates)
rekor secrets delete <id>
```

Set `--expires-at` (ISO-8601) on credentials that lapse — yearly certificates, time-bound tokens — then `rekor secrets list --expiring` surfaces what needs renewing before it breaks an integration. Expiry is informational; Rekor never auto-disables a secret.

### Account & Diagnostics

```bash
rekor whoami                 # Show the authenticated identity
rekor status                 # Auth, connectivity, and CLI version diagnostics
rekor update                 # Update the CLI to the latest published version
```

The CLI checks for a newer published version in the background and prints a one-line notice when an update is available. Disable the check with `REKOR_NO_UPDATE_CHECK=1` (also disabled when `NO_UPDATE_NOTIFIER` or `CI` is set).

---

## Output Format

Add `--output json` for machine-readable JSON output (default is `table`).

```bash
rekor documents get invoices rec_abc --database my-ws --output json
```

## Data Input

Any `--data`, `--schema`, `--tools`, or `--operations` flag accepts:
- Inline JSON: `'{"key":"value"}'`
- File reference: `@path/to/file.json`

## Environments

Databases are either **production** or **preview**. As an agent, you can:

- **Read** from any database (production or preview)
- **Write documents and relationships** to any database
- **Modify schemas** (collections, triggers, hooks) only in preview databases

To modify schemas, create a preview database first:

```bash
rekor databases create-preview my-database --name "add-invoices"
```

Then work in the preview database:

```bash
rekor collections upsert invoices --database my-database--add-invoices \
  --schema '{"type":"object","properties":{"amount":{"type":"number"}}}'
```

When you're done, ask a human operator to promote your changes:

```
# Human runs: rekor databases promote my-database --from my-database--add-invoices
```

**Promotion is a human-only operation.** You cannot promote directly. Always work in preview for schema changes.

## Best Practices

- **Use external IDs** for idempotent upserts — retry-safe, no duplicates.
- **One database per context** — keeps data isolated (per test run, per agent, per environment).
- **Preview for schema changes** — modify collections, triggers, hooks in a preview database. Ask a human to promote.
- **Batch for atomicity** — when multiple writes must succeed or fail together.
- **Relationships over nested data** — link documents instead of embedding. Enables traversal and flexible queries.
- **Attachments for large or binary content** — document data and relationship metadata are for structured JSON and are capped at ~1 MiB. Store PDFs, images, and other files as attachments (`rekor attachments upload <collection> <id> --filename <name> --file <path>`) and reference them from the document; inlining base64 blobs is rejected.
- **Schema first** — define the collection schema before writing documents. Validation catches bad data early.
- **Query delay after writes** — single-document reads (`documents get`, `relationships get`) reflect writes immediately; `sql` and `query-relationships` may take a moment to catch up. If you need to query right after writing, wait briefly or use direct gets.
- **Set token expiration** — use `--expires-at` for short-lived tokens. Revoke unused tokens promptly.
- **Save tokens immediately** — the raw token is shown only once on creation. Store it securely before closing the terminal.
