---
name: rekor
version: 1.20.0
description: |
  Set up and operate Rekor — a headless system of record for AI agents. Use when:
  installing the `rekor` CLI, authenticating, creating a database, defining the first
  collection, working in preview and promoting to production, querying documents via SQL,
  managing relationships, uploading file attachments, configuring inbound webhooks/triggers/batch
  operations, backing a collection with an external API (external sources), storing
  credentials in the secret vault, importing or exporting tool definitions across
  providers (OpenAI/Anthropic/Google/MCP), creating curated MCP Factory endpoints,
  scoping API tokens, modeling collections and MCP endpoints so LLM agents use them
  reliably, or interpreting Rekor concepts (database, collection, document,
  relationship, preview/production, external_id).
---

# Rekor — Data Layer for AI Agents

You have access to the `rekor` CLI — the builder interface for Rekor. Use it to set up schemas, configure collections, test in preview environments, and promote to production. Production agents use MCP tools to read/write documents; external systems integrate via REST API. The CLI is your tool for designing and managing the data model.

## Agent Guidelines

- Drive Rekor through the **`rekor` CLI** when you have shell access. If you also have `mcp__rekor__*` tools in your toolset, prefer the CLI for setup and configuration; reserve MCP tools for production read/write operations.
- Only provide information from this skill, tool descriptions, or reference documentation. Do not invent URLs, paths, commands, or flags.
- Schema work (collections, inbound webhooks, triggers, MCP Factory endpoints) only happens in **preview** databases. Always create or use a preview before changing schemas.
- **Identifiers are permanent.** The `id` of a database, collection, relationship type, and MCP Factory endpoint *is* its identity and is **immutable** — chosen once at creation and never renamed. Display names and descriptions stay editable, but never the `id` (a relationship type has no separate name — its `id` is also its label). There is no rename: to "rename" one of these you create a new entity and migrate its data (the old one is left untouched). So choose clear, stable, lowercase-slug ids up front (e.g. `patients`, `treated_by`).
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

After logging in with a user account, bind this repo to an organization so the CLI knows which one to operate in:

```bash
rekor init
```

This writes a committed `.rekor.yaml` at the repo root (auto-selecting your org if you have only one, otherwise prompting). It's required before creating databases, tokens, or secrets — and before `pull`/`push` — with a user login; if you have a single organization, `rekor init` selects it automatically. Override per command with `--org <id>` or `REKOR_ORG`. API keys (`rec_…`) are already scoped to one organization and skip this step.

### 3. Create a database

Databases are top-level data containers. One per app, domain, or tenant.

```bash
rekor databases create <id> --name "<Display Name>" [--tags <comma-separated>]
```

Set `REKOR_DATABASE=<id>` in the user's shell to avoid passing `--database` on every command.

### 4. Create a preview environment

Schema changes (collections, inbound webhooks, triggers, MCP Factory endpoints) are blocked in production databases. Create a preview to do schema work:

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
rekor sql "SELECT * FROM documents WHERE org_id = {org_id:String} AND database_id = {database_id:String} AND collection = '<slug>' AND deleted = false" --database <id>--<preview-slug>
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
rekor sql "SELECT data.invoice_number.:String as num, data.status.:String as status, arraySum(CAST(data.line_items[].amount, 'Array(Float64)')) as total FROM documents WHERE org_id = {org_id:String} AND database_id = {database_id:String} AND collection = 'invoices' AND deleted = false ORDER BY total DESC" --database my-ws
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
rekor databases promote <prod-id> --from <preview-id> [--dry-run] [--collections <ids>] [--triggers <ids>] [--inbound-webhooks <ids>]
rekor databases promotions <prod-id>
rekor databases rollback <prod-id> --promotion <promotion-id>
```

Tags let you group databases (e.g., `client:acme,billing`). Filter with `--tag`. Promotion is **human-only** (see Environments). Promote selectively with `--collections`/`--triggers`/`--inbound-webhooks` (omit to promote everything). `promotions` lists prior promotions; `rollback` reverts one by ID.

### Config as Code (pull / push)

Manage a database's whole config — collections, relationship types, inbound webhooks, triggers, MCP endpoints — as version-controlled YAML files instead of one-off commands. Files live under `rekor-ws/databases/<db>/`, one file per entity, so changes review cleanly in a pull request.

```bash
rekor pull <preview>                 # write the preview's config to rekor-ws/databases/<preview>/
rekor push <preview> [--dry-run]     # apply your local files to the preview (--dry-run shows the diff only)
rekor push <preview> --prune         # also delete entities that exist on the server but not in your files
```

- **Previews only.** `pull`/`push` operate on **preview** databases (config is never edited directly in production). `pull` refuses a production database. To ship, run `rekor databases promote` as usual.
- **Auto-create a preview.** Scaffold `rekor-ws/databases/<name>/database.yaml` with `origin_database_id: <prod-id>` (and no `database_id`); `rekor push` creates the preview from that production database, writes the new id back, and applies your files.
- **Secrets are never written to files.** Inbound-webhook/trigger and external-source secrets are stripped on `pull`. On `push`, a newly added inbound webhook/trigger gets a fresh secret (printed once — save it); existing secrets are left untouched. Manage secret values with `rekor inbound-webhooks` / `rekor triggers` / `rekor secrets`.
- **Deletions are opt-in.** Because deleting a collection also removes its documents, `push` is additive by default: entities missing from your files are reported but kept. Add `--prune` to delete them.
- **Renaming a collection is re-seed, not in-place.** A collection's `id` is its identity and must equal its filename (`collections/<id>.yaml`), so renaming means updating **both** the filename and the inner `id:` field together (changing only one errors; or delete the inner `id:` so it defaults to the filename). Either way it's a **create-new + orphan-old**, not an in-place rename: the diff shows `+ <new>` / `- <old>`, `push` creates a new **empty** collection, and the old one (with all its documents) is left untouched — documents do **not** migrate. To rename and keep the data: (1) `push` to create the new empty collection, (2) re-seed its documents (e.g. `rekor documents upsert`, or batch), then (3) `push --prune` to delete the orphaned old collection, which cascades its documents. The same filename-is-`id` rule applies to relationship types and the other config entities.

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
rekor documents query <collection> --database <ws> [--filter <json|@file>] [--sort <json>] [--fields <list>] [--limit <n>] [--offset <n>] [--external-source <src>]
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

**Sorting.** `--sort` takes a **JSON array** of terms, each `{"field":"data.<field>","direction":"asc"|"desc"}` — e.g. `--sort '[{"field":"data.created_at","direction":"desc"}]'`. Multiple terms sort lexically (first term primary). `direction` defaults to `asc` if omitted. The colon shorthand `data.name:asc` is **not** accepted — pass the JSON array form. Omit `--sort` when using `search` to keep the relevance ranking.

**Datetimes.** Declare datetime fields in the collection schema with `"format": "date-time"` (or `"format": "date"` for date-only). Rekor stores them in a single canonical form: a naive value (no offset) gets the collection's timezone attached, so `2026-06-11T13:00:00` is stored and returned as `2026-06-11T13:00:00-03:00` — the same shape from both `query` and a single-document read, with no UTC math required of you. Set the timezone with `"x-timezone": "America/Sao_Paulo"` on the collection schema, or set a database-wide default that every collection without its own `x-timezone` inherits — the resolution order is collection `x-timezone` → database `settings.timezone` → UTC. Set the database default from the CLI with `--timezone` (at create or update time), or via REST `PUT /v1/databases/<id>` with a `settings.timezone` body:

```bash
rekor databases create <database-id> --name "My DB" --timezone America/Sao_Paulo
rekor databases update <database-id> --timezone America/Sao_Paulo
# or via REST:
curl -X PUT https://api.rekor.pro/v1/databases/<database-id> \
  -H "Authorization: Bearer $REKOR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "settings": { "timezone": "America/Sao_Paulo" } }'
```

The value must be a valid IANA timezone (invalid names are rejected server-side). `rekor databases update --timezone` merges into the existing `settings` (it preserves other keys); the raw REST `PUT` and `--settings` flag **replace** the `settings` object wholesale, so send the full object. Omitted top-level fields (`name`, `description`, `tags`) are left unchanged on an existing database. **Filtering is instant-aware:** `eq`/`neq`/`in`/`not_in` and the range operators match datetimes by point-in-time, so the exact representation you pass (`T` vs space, with or without offset) doesn't change the result — `{ "field": "data.starts_at", "op": "eq", "value": "2026-06-11T13:00:00" }` matches the stored value regardless of how it was written. Use `eq` for an exact datetime, not `like` (which is plain substring matching). Raw `rekor sql` is **not** rewritten this way — match the canonical stored form (or prefer the Filter DSL `query` for datetime conditions).

**Partial vs full updates.** A full upsert (`manage_document` `upsert`, REST `PUT`) writes the **whole** document — send every field. To change just one field (flip a status, set a flag) without re-sending the rest, use a **partial update**: `manage_document` `patch`, the generated `update_<collection>` tool, or REST `PATCH .../documents/<collection>/<id>`. A partial update merges the provided fields over the stored document (top-level merge; nested objects are replaced wholesale), targets an existing document by id or `external_id`, and the merged result must still satisfy the collection schema. For a collection backed by an **external source**, the provided fields are forwarded to that upstream system, which decides how the update is applied — the merge-and-keep-the-rest guarantee applies to documents stored in Rekor.

**Cancellation & archival.** Documents are **updatable by default** — re-upserting the same `external_id` updates the existing document in place (idempotent upsert), and you can advance state freely (e.g. `scheduled → confirmed`). A collection can opt into stricter archival via its schema's `x-archival.mode`: `immutable` (write-once — any update is rejected) or `on_attribute` (updatable until a configured terminal value). A document can also be *cancelled* — a first-class state distinct from delete — via MCP (`manage_document` `cancel`) or REST (`POST .../documents/<collection>/<id>/cancel`); there is no CLI `cancel` subcommand. A cancelled document can no longer be updated. Documents may also be *archived* automatically (when they reach a terminal/immutable state, or after a long period of inactivity). An archived document stays readable and can still be cancelled, but it can't be deleted, and re-upserting by the same `external_id` creates a **new** document rather than updating the archived one. When querying with `rekor sql`, add `archived = false` to see only active documents (cancelled rows carry `cancelled = true`).

**Referential integrity (foreign keys).** A field can be declared a **foreign key** into another collection by adding an `x-fk` hint to that field in the schema. Every write then checks the value points at a real document — a missing reference is rejected with an actionable error instead of landing as silent bad data.

```jsonc
"patient_id":      { "type": "string", "x-fk": { "collection": "patients" } },                 // matches patients by external_id (default)
"professional_id": { "type": "string", "x-fk": { "collection": "professionals", "key": "code" } } // matches a named field on the target
```

- `key` is the field on the **target** that the value must match — `external_id` by default, or any top-level field (e.g. a `code`).
- Checked on every write (create, full upsert, partial update, batch). Empty/absent values are skipped — mark the field `required` if it must be present.
- In a **batch**, write the target before the document that references it (the batch is atomic — a bad reference rejects the whole batch).
- Use foreign keys for **reference data** (patients, plans, products), not high-volume event logs — a referenced collection is kept readily available so the check stays fast.

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

**Important**: Always include BOTH `org_id = {org_id:String}` AND `database_id = {database_id:String}`, plus `deleted = false`. Both scoping predicates are mandatory — a database id is unique per-org, not globally, so `org_id` is required to isolate your data. Both placeholders are bound server-side from your authenticated org and database. The `documents` table also carries `archived` and `cancelled` boolean columns — add `archived = false` to query only active documents. Queries always see the latest version of each row — the server handles deduplication.

**Accessing JSON fields**: Use `data.field.:Type` subcolumn syntax for the native JSON type. Use `CAST(data.field, 'Type')` when type-safe conversion is needed (e.g., integers stored as Int64 vs Float64).

**Examples**:

```bash
# Simple query
rekor sql "SELECT data.invoice_number.:String as num, data.status.:String as status FROM documents WHERE org_id = {org_id:String} AND database_id = {database_id:String} AND collection = 'invoices' AND deleted = false" --database my-ws

# Aggregation
rekor sql "SELECT data.status.:String as status, count() as cnt FROM documents WHERE org_id = {org_id:String} AND database_id = {database_id:String} AND collection = 'invoices' AND deleted = false GROUP BY status" --database my-ws

# Array aggregation (sum embedded line items)
rekor sql "SELECT data.invoice_number.:String as num, arraySum(CAST(data.line_items[].amount, 'Array(Float64)')) as total FROM documents WHERE org_id = {org_id:String} AND database_id = {database_id:String} AND collection = 'invoices' AND deleted = false" --database my-ws

# Explode array elements with ARRAY JOIN
rekor sql "SELECT item.description.:String as item, sum(CAST(item.amount, 'Float64')) as revenue FROM documents ARRAY JOIN data.line_items[] as item WHERE org_id = {org_id:String} AND database_id = {database_id:String} AND collection = 'invoices' AND deleted = false GROUP BY item ORDER BY revenue DESC" --database my-ws

# CTE joining documents with relationships
rekor sql "WITH inv AS (SELECT id, data.invoice_number.:String as num, arraySum(CAST(data.line_items[].amount, 'Array(Float64)')) as total FROM documents WHERE org_id = {org_id:String} AND database_id = {database_id:String} AND collection = 'invoices' AND deleted = false), pay AS (SELECT target_id, sum(CAST(data.allocated, 'Float64')) as paid FROM relationships WHERE org_id = {org_id:String} AND database_id = {database_id:String} AND rel_type = 'payment_for' AND deleted = false GROUP BY target_id) SELECT inv.num, inv.total, coalesce(pay.paid, 0) as paid, inv.total - coalesce(pay.paid, 0) as balance FROM inv LEFT JOIN pay ON pay.target_id = inv.id ORDER BY balance DESC" --database my-ws

# With parameters
rekor sql "SELECT * FROM documents WHERE org_id = {org_id:String} AND database_id = {database_id:String} AND data.status.:String = {status:String} AND deleted = false" --database my-ws --param status=issued

# Fuzzy / approximate text match, ranked by closeness (power-user form of `documents query --filter {op:search}`).
# Fold case + accents on BOTH sides so "São" matches "sao"; jaroWinklerSimilarity suits short names.
rekor sql "SELECT *, jaroWinklerSimilarity(lowerUTF8(data.car_model.:String), lowerUTF8({q:String})) AS score FROM documents WHERE org_id = {org_id:String} AND database_id = {database_id:String} AND collection = 'vehicles' AND deleted = false AND score >= 0.85 ORDER BY score DESC LIMIT 10" --database my-ws --param q='honda civic'
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

### Inbound Webhooks (data in)

External systems push data into Rekor via inbound webhooks. Each one provides a unique ingest URL.

```bash
rekor inbound-webhooks create --database <ws> --name <name> --secret <hmac-secret> [--id <id>] [--collection-scope <comma-separated>] [--field-mapping <json>] [--source-binding <json>] [--ingest-auth <json>]
rekor inbound-webhooks list --database <ws>
rekor inbound-webhooks get <id> --database <ws>
rekor inbound-webhooks delete <id> --database <ws>
```

`--secret` is the shared secret (required). By default the sender signs ingest requests and Rekor verifies an HMAC: either **Signing v1** — `X-Rekor-Signature: v1,<hex>` over id+timestamp+method+path+body, with an `X-Rekor-Timestamp` Rekor checks for freshness (the same scheme triggers and proxied calls use, so one signer works both directions) — or a legacy body-only HMAC for older senders. A v1 delivery is idempotent: a duplicate (a retry or replay carrying the same `X-Rekor-Id`) replays the first response instead of writing again. For senders that authenticate with a **static per-account header** instead of signing each request, `--ingest-auth '{"type":"static_header","header":"X-Account-Key"}'` verifies that header's value against `--secret` (constant-time) — Rekor compares the configured header rather than a signature. `--collection-scope` restricts which collections the inbound webhook may write to (omit for all). Inbound webhooks can only be created/deleted in preview databases. Promote to production when ready.

**Translate the received payload before write.** By default an inbound webhook stores the payload as-is. To store **canonical** documents straight from an external system's raw shape, attach a mapping — the same `field_mapping` contract external sources use (renames, value maps, date reformatting, computed/compose), applied in the inbound direction:
- `--source-binding '{"collection":"<id>","source":"<name>"}'` reuses an existing source's `field_mapping`, so the same translation that proxies that collection's reads/writes also canonicalizes inbound deliveries — one contract, both directions.
- `--field-mapping '{"to_external":{"status":"state"}}'` is an inline mapping for a purely-native collection with no source. `to_external` renames auto-invert on read (upstream `state` → Rekor `status`); or give `to_rekor`/`computed` explicitly.

The two are mutually exclusive. The mapping is validated when the webhook is created and again at promotion (a `source_binding` must still resolve to a source that has a `field_mapping`). This makes an executor-free **native-mirror sync** complete: pair an inbound webhook's `to_rekor` with an `external_write` trigger's `to_external` to keep a native collection in sync with a plain-HTTP upstream in both directions, reusing a single source contract.

### Triggers (data out)

Triggers fire automatically when documents change. A trigger's action is one of:
- **webhook** — an HMAC-signed HTTP POST to an external system (`--url`/`--secret`).
- **internal_write** — Rekor updates a *referenced* document for you, with no external service. On a matching write it locates a target document and applies a guarded patch (an optional compare-and-set `precondition` so the update only lands when the target is still in the expected state). Ideal for "when A changes, update related B" — e.g. booking an appointment flips its slot to busy only if it was free. Pass it via `--action` (see below); document.created/updated only. Two modes: **`async`** applies the patch just after the triggering write (eventual; a precondition miss is recorded but the triggering write still stands), and **`transactional`** applies it atomically *with* the triggering write — if the `precondition` misses, the whole write is rejected and rolled back (nothing is created). Use `transactional` when the two writes must succeed or fail together (e.g. don't create the appointment unless the slot was free).
- **external_write** — Rekor writes the changed document *out* to an external system by reusing an external source's write path — its field translation, endpoint, body shape, and auth — with no executor. Use it to keep a native collection mirrored to a legacy API: store records natively in Rekor and have each write pushed upstream in the upstream's own shape. The action names the source: `{type:"external_write","collection":"<source-owner>","source":"<name>","op":"create"}`. `collection` is the collection that *defines* the source (a native mirror points at a sibling source-backed collection; defaults to the triggering collection if omitted); `op` defaults from the event (created→create, updated→update, deleted→delete) and is overridable. On a **create**, Rekor reads the upstream-assigned id from the response and links it back onto the triggering document as its `external_id` (so the native record points at its upstream counterpart, and retries don't duplicate) — opt out with `link_external_id:false`. Updates/deletes are dispatched only for documents already linked to that source. Delivery is async, signed, retried, and dead-lettered like a webhook (inspect with `rekor triggers deliveries`); document.created/updated/deleted only.

```bash
rekor triggers create --database <ws> --name <name> --events <comma-separated> --url <url> --secret <hmac-secret> [--id <id>] [--collection-scope <comma-separated>] [--filter <json>]
# internal_write action (instead of --url/--secret):
rekor triggers create --database <ws> --name <name> --events document.created --collection-scope appointments \
  --action '{"type":"internal_write","target":"slots","match":{"source_field":"data.slot_id"},"patch":{"status":"busy"},"precondition":{"field":"data.status","op":"eq","value":"free"},"mode":"async"}'
# external_write action — mirror a native collection's writes out through a source's write path:
rekor triggers create --database <ws> --name <name> --events document.created,document.updated --collection-scope appointments \
  --action '{"type":"external_write","collection":"upstream_appointments","source":"legacy","op":"create"}'
rekor triggers list --database <ws>
rekor triggers get <id> --database <ws>
rekor triggers delete <id> --database <ws>
rekor triggers deliveries --database <ws> [--status <pending|delivered|failed|dead>] [--trigger-id <id>]
```

`--events` is a **comma-separated** list (not a JSON array). Valid events: `document.created`, `document.updated`, `document.deleted`, `document.cancelled`, `relationship.created`, `relationship.updated`, `relationship.deleted` — e.g. `--events document.created,document.updated`. (Bare names like `create`/`update` will silently never match.) For a webhook, `--secret` is the HMAC signing secret receivers verify with. `--collection-scope` limits firing to specific collections (omit for all). An `internal_write` target document stays available as long as a trigger references it. For `external_write`, the referenced source is validated at create time (the named collection, source, and write endpoint for each op must exist) — and referencing a source from a trigger never turns the triggering collection into a proxy; it stays native.

Add `--filter '<json>'` to fire only on documents matching a condition — the same filter DSL queries use, evaluated against the document (or relationship) being written, e.g. `--filter '{"field":"data.status","op":"eq","value":"paid"}'` fires only when `status` is `paid`. Combine conditions with `and`/`or` groups. A malformed filter is rejected at create time.

Triggers are HMAC-signed (`X-Rekor-Signature: v1,<hex>` over id+timestamp+method+path+body, with an `X-Rekor-Timestamp` the receiver checks for freshness — the same scheme proxied requests use) and carry `X-Rekor-Id`/`X-Rekor-Delivery-Id` for receiver dedupe. Delivery is reliable — failed attempts are retried with backoff and dead-lettered after repeated failure; inspect status with `rekor triggers deliveries`. By default, writes from inbound webhooks don't re-fire triggers (`skip_inbound_webhook_writes: true`). Triggers can only be created/deleted in preview databases.

### Executors (acting on the outside world)

Rekor records, signs, and dispatches — but the actual outside-world action (calling a third-party API, presenting a client certificate, running logic Rekor can't) happens in an **executor**: a small, stateless HTTP service you deploy and point a trigger or external source at. Rekor signs every dispatched request; the executor verifies it, does the work, and writes the result back.

**Do you even need one?** A plain REST/JSON API → just an external source (no executor) — and the same source's write path can be reused *outbound* by an `external_write` trigger, so mirroring a native collection to a plain-HTTP upstream needs no executor either. Build an executor only when a trigger runs custom logic, or a source call can't be a direct HTTP request (mTLS, SOAP, binary creds, heavy processing). Inbound webhooks go the other way — your executor calls an inbound webhook to write results back in.

**Always receive these requests with the `rekor-sdk` package — never hand-roll signature verification** (a mistake lets anyone forge a request to your executor). The SDK verifies the signature + timestamp, dedupes retries on the idempotency key, and normalizes errors — you write one handler:

```ts
import { createExecutor, toFetchHandler } from 'rekor-sdk'
const handler = toFetchHandler(createExecutor({
  secret: process.env.REKOR_SIGNING_SECRET!,
  handler: async (ctx) => { /* ctx.body is verified; do the work, return a result or nothing */ },
}))
// Hono: app.post('/rekor', (c) => handler(c.req.raw))  •  Cloudflare: export default { fetch: handler }
```

**Where to run it:** default to a **serverless platform** (Cloudflare Workers, Vercel, Deno Deploy, AWS Lambda — pick whichever you already use; the contract is platform-agnostic) — most executors are just authenticated API calls, and a serverless function is cheap, fast, and scales to zero. Step up to a **long-running container or server** (Fly.io, Railway, Render, a VM) only when a serverless function can't do the job: mutual-TLS client certificates, long-running work, raw TCP/SOAP, or native dependencies.

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
- `field_mapping` — **optional**; omit it for identity passthrough (the upstream object is stored as the document data verbatim, and writes send your data unchanged). When set, `to_external` / `to_rekor` map between your schema and the upstream's field names. A value is either a simple rename (`"status": "invoice_status"`) or a rich rule `{ path, values, default, transform, date_format, array_mode }`:
  - `transform` — `lowercase` / `uppercase` / `trim` / `to_string` / `to_number` / `to_boolean` / `iso_date`.
  - `values` — an enum map. **Always Rekor-keyed** (`{ "<your value>": "<upstream value>" }`, e.g. `{ "active": "ACTIVE" }`) — the *same* map serves both directions: forward (write) looks up by key, reverse (read) matches by value and returns the key. **The orientation does not flip in an explicit `to_rekor` rule** even though everything else in that rule reads upstream→Rekor (the rule key is the upstream field, `path` is your field): a `values` map stays Rekor-keyed and is reverse-matched, e.g. `to_rekor: { activity: { path: "activity_id", values: { "natacao-adulto": "8" } } }` maps an upstream `8` (or `"8"`) → `"natacao-adulto"`. As a convenience the reverse path *also* accepts the intuitive upstream-keyed form (`{ "8": "natacao-adulto" }`) when the Rekor-keyed match misses — but write the **Rekor-keyed** form to stay unambiguous (reverse-match wins on any key/value overlap). A fully backwards or mistyped map otherwise passes the raw upstream value through unchanged rather than erroring, so confirm by reading a document back after configuring.
  - `default` — value used when the field is missing.
  - `array_mode` — `first` / `last` / `flatten` for array-valued paths.
  - `date_format` (+ optional `rekor_format`, default ISO) — bidirectional date/time reshaping between the external wire format and Rekor's canonical. Tokens `yyyy MM dd HH mm ss` + separators, e.g. `date_format: "dd/MM/yyyy"` (⇄ ISO date) or `date_format: "HH:mm:ss"` with `rekor_format: "HH:mm"` (truncate seconds).
  - **Split one field into several upstream params** (`to_external` only) — give the rule `targets` (an array of rules) instead of a single `path` to fan one canonical field out to multiple upstream params, each with its own `path`/`transform`/`date_format`. The headline use: an upstream that wants **separate `date` + `time` params** fed from a single ISO datetime — `"starts_at": { "targets": [{ "path": "data", "date_format": "dd/MM/yyyy" }, { "path": "hora", "date_format": "HH:mm" }] }`. A split field can't be a `forward_filters` field (no single upstream name).
  - **Compose / computed fields** (read side) — `computed` builds a Rekor `data.*` field from a `{{token}}` template over the **raw upstream** record (the inverse of a split, and the same template grammar as a composite `id_path`). Use it to rebuild one field from several upstream fields — `"computed": { "starts_at": { "template": "{{data}}T{{hora}}", "sources": { "data": { "date_format": "dd/MM/yyyy", "rekor_format": "yyyy-MM-dd" }, "hora": { "date_format": "HH:mm", "rekor_format": "HH:mm:ss" } } } }` rebuilds an ISO datetime from upstream `data`+`hora` — or to derive a key from siblings (`"slot_id": { "template": "{{professional_id}}::{{data}}::{{hora}}" }`). `sources` reshapes each token (same vocabulary as a `to_rekor` rule); if any token is absent the whole field is omitted. Tokens reference the upstream's own field names (no chaining over other computed fields). A token may also read a **forwarded list filter** via the reserved `{{filter.<field>}}` namespace — see `forward_filters` below — to backfill a scope key/date the upstream omits from its list rows (e.g. `"starts_at": { "template": "{{filter.date}}T{{hora_atendimento}}" }` when a list-by-date upstream returns only the time). `{{filter.*}}` populates on `list` only and a composite `id_path` may use it too; each referenced field must be in `list.forward_filters.fields`.
  - Rich rules don't auto-invert — supply an explicit `to_rekor` entry (or a `computed` field) for the reverse direction. In that explicit `to_rekor` entry the rule key is the upstream field and `path` is your field, but a `values` map stays **Rekor-keyed** (see `values` above) — it's the one part that doesn't follow the rest of the rule's upstream→Rekor direction.
  - **Per-operation override**: the source-level `field_mapping` applies to every op, but an RPC/legacy upstream may use *different* param names for the same field across verbs (create takes `entity_id`, update takes `id_entity`). Put a `field_mapping` on an individual `get`/`list`/`create`/`update` endpoint to override the source-level one for that op (full replacement — set every field it needs); it falls back to the source-level mapping when unset, e.g. `"update": { …, "field_mapping": { "to_external": { "owner": "id_entity" } } }`. Not allowed on `delete` (no body — set its param names in the URL template).
- `get` / `list` / `create` / `update` / `delete` — per-operation endpoint templates. Each has its own `url` (with `{{external_id}}`, `{{query.*}}`, `{{data.*}}`, `{{current.*}}`/`{{prior.*}}` on update/delete — see below, `{{auth.org_id}}`, `{{auth.database_id}}` tokens) and `method` (any of GET/POST/PUT/PATCH/DELETE) — so **RPC-style verb paths and all-POST upstreams work directly**, e.g. `list: { url: ".../listar_x", method: "POST" }`, `create: { url: ".../incluir_x", method: "POST" }`. `response_path` / `total_path` extract records/count from an envelope (e.g. `response_path: "dados"` pulls the array out of `{ ..., dados: [...] }`).
- `id_path` — dot-path to the `external_id` within each **raw** upstream record, for upstreams keyed on a domain field rather than `id`/`_id` (e.g. `id_path: "codigo"`, `"identifiers.cpf"`). Applies to LIST (per row) and CREATE (from the response). When unset, the id is taken from the first present conventional key (`id`/`_id`/`Id`/`ID`/`uuid`/`key`). **Set this when your upstream has no conventional id field — otherwise a LIST whose rows all lack a resolvable `external_id` fails loud with a `422 EXTERNAL_CONFIG` naming the fix and the row keys it saw (a partial drop returns the resolvable rows plus a `meta.dropped_no_id` count), and CREATE fails to identify the new document.** For an upstream whose identity is composed of several fields with no single natural id, `id_path` may instead be a **template** joining fields, e.g. `id_path: "{{entity}}::{{date}}::{{time}}"`; a composite id is **list-only** (rejected at config-write when `get`/`create`/`update`/`delete` is also configured, since it can't address a single row by id). For an RPC-style upstream that returns the new record's server-assigned id on **CREATE** under a *different* field than the one keying LIST rows, set a **create-only** `id_path` on the `create` endpoint (e.g. `"create": { …, "id_path": "created_ref" }`) — it overrides the source-level `id_path` for the create read-back and falls back to it when unset. It must be a plain dot-path (no composite template), since the resolved id then addresses the record on later get/update/delete.
- `forward_filters` (on the `list` endpoint) — opt in to forward the agent's filter conditions to the upstream so a proxied `list`/`query` can pass a search key, a date, a parent id, etc. `fields` is the allowlist of Rekor field names that may be forwarded (each translated to the upstream's name/value via `field_mapping`); `target` (`query` for a GET, `body` for a POST/PUT/PATCH list — inferred from the method when omitted) chooses where they go. Without it, a proxied query still **rejects** any `--filter`/MCP `filter`. Currently forwards exact-match (`eq`) conditions only — a single `eq` or a flat AND of `eq`s; any other operator or shape is rejected with an actionable error rather than silently returning unfiltered data, e.g. `"list": { "url": ".../search", "method": "GET", "forward_filters": { "fields": ["email"] } }`. A forwarded value is also exposed to the read mapping as `{{filter.<field>}}` (a `computed` field or a composite `id_path`), so a row can carry the filter value the upstream leaves out of its list rows — e.g. forward `date`, then compose `starts_at` from `{{filter.date}}` + the row's time (see Compose / computed fields).
- **Static constants** need no special field: bake constant **query** params into the `url` (`".../listar?clinica=35"`), and put constant non-secret **headers** in `endpoint.headers` (`{ "X-Tenant": "35" }`). For a constant **body** param use `static_body` (below); for a secret use `injections`.
- `request_encoding` — `json` (default) or `form` to send `application/x-www-form-urlencoded` bodies for legacy form-post upstreams.
- `success_path` / `message_path` — for upstreams that return HTTP 200 even on logical failure (`{ "success": false, "message": "..." }`). When `success_path` resolves falsy, the call surfaces as an error carrying the `message_path` text instead of being mistaken for data. (Dot-paths, like `response_path` — not JSONPath.)
- `static_body` — constant, non-secret params merged into every body-bearing request (incl. POST-based reads), e.g. a fixed tenant id the agent never supplies. Agent data and `injections` win on a key collision.
- `body_template` (per `create`/`update`/`delete` endpoint) — map of body field → templated string, interpolated against the same per-op tokens as that endpoint's `url` (`{{external_id}}`, `{{data.*}}`, `{{query.*}}`, `{{auth.org_id}}`/`{{auth.database_id}}`) and merged into the request body. Use it to put the document id — or any addressable value — **into** the body for RPC/legacy upstreams that key the update/cancel verb by id in the body rather than the URL, e.g. `"update": { …, "body_template": { "agendamento_id": "{{external_id}}" } }`. Body-bearing ops only — create/update, and a `delete` whose method carries a body (POST/PUT/PATCH); rejected on get/list and on any GET/DELETE-method endpoint (no body to carry it). Unlike `static_body` (constants, never templated) its values are filled per request; on a key collision agent data is overlaid by `body_template`, and `injections` win over both. Values are filled as **strings** (the same as `url`/header tokens) — for a numeric/boolean field an upstream is strict about, send it through `field_mapping` or `static_body` instead.
- `{{current.*}}` / `{{prior.*}}` (in an `update`/`delete` `url`, headers, or `body_template`) — the document's **current** stored field values from the upstream, for upstreams that need the existing state on a write, not just the id + new values: optimistic concurrency (`?ifmatch={{prior.version}}`), "move"-style updates that take the OLD value plus the NEW (a reschedule taking the current `data_agendada` alongside `nova_data`), or a cancel keyed by the current field values. Rekor sources them with a pre-write read through the source's **`get`** endpoint, so a `get` is **required** when you reference one (rejected at config-write otherwise) — and the values are the **raw upstream record** (the upstream's own field names + wire format, echoed back verbatim). `prior` is an alias of `current` (a proxied write has no local merge). Update/delete only; referencing them costs one extra read per write (skipped entirely when unused). e.g. `"update": { …, "body_template": { "agendamento_id": "{{external_id}}", "data_agendada": "{{current.data_agendada}}", "nova_data": "{{data.data_agendada}}" } }`.
- `injections` — extra per-request vault secrets placed into a header or body field (for upstreams needing their own credential).
- `executor_secrets` — named vault credentials an executor pulls at dispatch, for a binary or large per-tenant credential it can't take inline (e.g. an mTLS client certificate). Each `{name, secret_ref}` declares a `vault:<name>` reference (templating only `{{auth.org_id}}`/`{{auth.database_id}}`); each pull is short-lived and single-use, scoped to the calling database.
- `cache_ttl` (seconds) + `stale_if_error` — read-through caching, optionally serving the last-known value on a transient upstream failure.
- `signing` — opt into HMAC request signing when the source points at an executor (adds `X-Rekor-Signature` + `X-Rekor-Timestamp`); omit for third-party APIs.
- `timeout_ms` — per-request upstream timeout (default 10s, max 30s); a timeout surfaces as a transient error so `stale_if_error` can serve a cached value.
- `breaker` — per-source circuit breaker: `{failure_threshold, cooldown_ms}`. After `failure_threshold` consecutive transient failures (default 5) the source opens and short-circuits calls for `cooldown_ms` (default 30s) before a single half-open probe. Always on; this only tunes it.

URLs must be absolute `https` (or `http` only when the source sets `allow_insecure_http: true`, for a legacy/on-prem upstream without TLS) and tokens may appear only in the path/query (never the host) — an SSRF guard rejects otherwise.

**Worked example — a non-REST / legacy upstream** (all-POST verb paths, a form body, a `{ success, dados }` envelope, a constant tenant id, and `dd/mm/yyyy` dates) configured as a **direct** source — no executor:

```json
{
  "name": "legacy",
  "auth": { "header": "Authorization", "value_template": "Bearer {{secret}}", "secret_ref": "vault:legacy-key" },
  "request_encoding": "form",
  "static_body": { "clinica": "35" },
  "success_path": "success",
  "message_path": "message",
  "id_path": "codigo",
  "list":   { "url": "https://api.legacy.example/listar_agendamentos", "method": "POST", "response_path": "dados", "total_path": "total" },
  "create": { "url": "https://api.legacy.example/incluir_agendamento",  "method": "POST", "response_path": "dados" },
  "field_mapping": {
    "to_external": { "date": { "path": "data", "date_format": "dd/MM/yyyy" }, "patient": "paciente" },
    "to_rekor":    { "data": { "path": "date", "date_format": "dd/MM/yyyy" }, "paciente": "patient" }
  }
}
```

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

**Reading results**: the default (table) output prints a one-line summary per operation (index, type, resulting id). Add `--output json` for the full per-operation payload — the complete document/relationship of every write. Because the batch is atomic, a single failing operation rolls the **whole** batch back and the command exits non-zero with `Operation <N> failed: <reason>` naming the offending index — no partial writes land.

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

# Get the MCP connection URL (Streamable HTTP)
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
      "name": "invoice",
      "names": { "list": "search_invoices" },
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
      "names": { "create": "assign_payment" },
      "description_override": "Link a payment to an invoice"
    }
  ],
  "sql_query": true
}'
```

Or from a file: `--config @endpoint.json`

**Tool names (`name` / `names`)**: each generated tool is named `<op>_<name>` — `name` is the base noun, defaulting to the collection id (so `invoices` → `get_invoices`, `list_invoices`). Set `name` to choose the base (e.g. `"name": "invoice"` → `get_invoice`). Override an individual tool with `names`, a per-operation map of literal names (`{ "list": "search_invoices" }`), so a tool reads as the job it does. Relationship ops map to `link_`/`list_`/`unlink_` over their base. Tool names must be unique across the endpoint. Only `name` and `names` control naming — there is no `name_override` key, and every key inside `names` must be a valid operation. An unknown or deprecated key on a tool is rejected at config-write, so a stray key can never be silently accepted and then ignored.

**Typed filter params (`filterable_fields`)**: expose chosen fields of a collection as typed parameters on its generated `list` tool, derived from the collection schema — so the agent fills native arguments instead of writing a filter expression. Each field becomes a parameter shaped by its type:

- an `enum` or boolean field → an exact-match param (the agent can only pick a valid value)
- a number or date/date-time field → a range pair (`<field>_min`/`<field>_max`, or `<field>_after`/`<field>_before`)
- a string field → a `<field>_contains` substring param

Per field you may set `param` (rename the generated param), `match` (`exact` | `range` | `text` | `any_of` | `member` — `any_of` accepts a list and matches any; `member` exposes an **array** field as a single membership value), an optional `enum` (constrain the param to a fixed set so the agent can only pick a valid value), an optional `pattern` (a regex declaring the valid format of a structured string field — e.g. an id or code shape like `^pat-[0-9]+$` — so a placeholder such as `"null"`/`"undefined"` or any wrong format is rejected with an actionable error instead of silently matching nothing; applies to an exact/any_of/member match on a plain string field, and can't be combined with `enum`), and `description`. An array field auto-exposes as a `member` param; object fields are rejected — expose a nested path (`address.city`) instead. The generic `filter` parameter stays on the tool as the escape hatch for anything the typed params can't express (OR / nesting); typed params and `filter` are combined with AND.

When the typed params cover everything an agent needs, set `"expose_filter": false` on the tool to drop the generic `filter` parameter entirely — keeping the agent-facing tool schema small. Server-side translation of the typed params is unaffected.

The list tool's machinery params (`sort`, `limit`, `offset`, `fields`) can each be hidden the same way — `"expose_sort"`/`"expose_limit"`/`"expose_offset"`/`"expose_fields": false` — and given a server-side default (`"default_sort"`/`"default_limit"`/`"default_fields"`) that still applies when the param is hidden. `"default_fields"` takes the same shape as the `fields` param — a comma-separated string (`"external_id,data.name"`) or an array (`["external_id", "data.name"]`). A hidden `limit` surfaces a `"truncated"` flag in the response if it caps the result, so rows are never silently dropped. `"agent_minimal": true` is a preset that hides all of them (plus `filter`) with a generous default limit, leaving an inputSchema that is pure typed semantics; any explicit `expose_*`/`default_*` on the same tool overrides the preset.

**Curated write surface (`writable_fields`)**: the write-side mirror of `filterable_fields`. On a `create`/`update` tool, list exactly the fields the tool may set — an allowlist that does two things at once:

- **Least-privilege / intent-scoped tools.** A field the agent sends that isn't on the list is rejected with an actionable error. So a front-desk tool can be barred from ever setting a price or internal field, and you can split one operation into intent tools — a `confirm` tool allowed to set only `status` and a `reschedule` tool allowed to set only the time — instead of relying on prose to fence them.
- **A rich, typed write schema.** The tool's `data` parameter is generated from the collection schema for just those fields — their types, enums, formats, `required`, and descriptions — instead of a generic "any object" slot. The agent gets the same typed, described, validated guidance on writes that `filterable_fields` gives on reads.

```json
{
  "collection": "appointments",
  "operations": ["update"],
  "names": { "update": "reschedule_appointment" },
  "writable_fields": [
    { "field": "start_time", "param": "when", "description": "New start time (ISO 8601)" }
  ]
}
```

Per field you may set `param` (rename the key the agent sets in `data` — the tool maps it back to the real field) and `description` (override the field's description). Fields are **top-level only** (writes merge at the top level — a partial update changes just the fields you send and leaves the rest untouched; nested objects are replaced whole). The collection schema stays the validation source of truth — `writable_fields` shapes *which* fields the tool exposes and *how they're described*, it doesn't re-validate values. Omit `writable_fields` to keep the generic any-field `data` slot. A tool with `writable_fields` but no `create`/`update` operation is rejected at config-write.

**Conditional writes (`precondition`)**: give a `create`/`update` tool a `precondition` — a Filter DSL expression checked against the document's **current** state before the write applies. If it doesn't hold, the write is rejected with a 409 conflict and nothing changes; if it holds, the write proceeds. This turns a fragile read-then-write into one correct, race-free call: model a bookable slot as a document and make "book it" a guarded update, so two agents can't both book the same slot.

```json
{
  "collection": "slots",
  "operations": ["update"],
  "names": { "update": "book_slot" },
  "precondition": { "field": "data.status", "op": "eq", "value": "free" }
}
```

The guard is **invisible to the agent** — it's part of the endpoint config, never a tool parameter — so the agent just calls `book_slot(...)` and the booking integrity is enforced for it. Paths address the current document as `data.<field>`, `version`, or `id` (e.g. `{ "field": "version", "op": "is_null" }` = create-only-if-absent). One precondition per tool; the `search` operator isn't allowed. Idempotent writes (retries that shouldn't double-create) are already handled by upsert-by-`external_id` — the precondition is for cross-state guards, not retry safety.

**Guarded + unguarded writes on one collection**: a `precondition` is one-per-tool, so to expose both a guarded write and a plain write on the same collection, declare two tool entries for that collection with the same operation but distinct `names`. Tool names only need to be unique across the endpoint — a duplicate (collection, operation) pair is fine:

```json
"tools": [
  {
    "collection": "slots",
    "operations": ["update"],
    "names": { "update": "book_slot" },
    "precondition": { "field": "data.status", "op": "eq", "value": "free" }
  },
  {
    "collection": "slots",
    "operations": ["update"],
    "names": { "update": "manage_slot" }
  }
]
```

`book_slot` succeeds only on a slot that's still `free`; `manage_slot` updates the same slot unconditionally (e.g. an operator correction). Without the distinct `names`, both would default to `update_slots` and collide.

The generated `list` tools are lenient about how structured arguments arrive: `filter` is always a JSON-encoded Filter DSL string, while `sort` and any multi-value (`any_of`) parameter accept **either** the native array **or** a JSON-encoded string of it — so an agent that serializes array arguments as strings still works. (`sort` is the same JSON array of `{"field","direction"}` terms described above.)

Connect agents to the endpoint URL with a token scoped to exactly one database. The agent sees only the tools you configured — fully domain-specific, no Rekor concepts.

For least-privilege, mint a token bound to the endpoint in one step: `rekor tokens create-for-endpoint <slug> --database <db>` (or pass `--mint-token` to `rekor endpoints upsert`). An endpoint-bound token's authorization IS the endpoint's tool surface — exactly those collections and operations, relationships, batch, and SQL only if you enabled it, nothing else — so a leaked token can't reach beyond the tools you exposed, and you can rotate or revoke one per agent.

Endpoints can only be created/modified in preview databases. Promote to production when ready. Promotion is blocked if it would break a published endpoint — removing a collection or relationship type the endpoint exposes, or a field its typed filters, `writable_fields`, or a `precondition` depend on — so promote the endpoint together with the schema change (a dry run lists any such conflicts first).

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

**Permissions**: `read:documents`, `write:documents`, `read:collections`, `write:collections`, `read:relationships`, `write:relationships`, `read:attachments`, `write:attachments`, `read:inbound_webhooks`, `write:inbound_webhooks`, `read:triggers`, `write:triggers`, `read:endpoints`, `write:endpoints`, `read:databases`, `write:databases`, `read:audit` (read-only; grants change-history access, admin-gated, not implied by other grants), or `*` for all.

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

The CLI checks for a newer published version in the background and prints a one-line notice when an update is available. When this skill is installed in your repo, it also checks whether a newer skill is published and nudges you to run `npx skills add wayai-pro/rekor-skill -y`. Disable both checks with `REKOR_NO_UPDATE_CHECK=1` (also disabled when `NO_UPDATE_NOTIFIER` or `CI` is set).

### Reporting

File a bug or issue to the Rekor team, then follow it through to a fix. Reports are deduplicated, so re-filing the same problem merges into the existing one rather than spamming.

```bash
# File a report (deduplicated). --dedup-key collapses repeat occurrences of the same issue.
rekor report create --title "Upsert 500s on large docs" --description "..." \
  [--severity high] [--steps "..."] [--error-message "..."] [--dedup-key "<stable-key>"] \
  [--database-id <id>] [--collection <name>] [--document-id <id>]   # entity pointers aid investigation

# Track and verify your own reports
rekor report list [--status shipped]      # your reports, newest first (--status shipped = awaiting your check)
rekor report get <report_id>              # status + the message thread (the team's notes to you)
rekor report accept <report_id>           # the shipped fix works → resolved
rekor report contest <report_id> --reason "still reproduces because ..."   # the fix didn't work, or a dismissal is wrong → back to the team
```

**The verification loop.** After the team escalates and a fix ships, your report moves to `shipped` — you'll get an email, and `rekor login`/`rekor status` remind you once (or find it with `rekor report list --status shipped`). Read `rekor report get <id>` (status + the team's note), then `accept` if it works or `contest --reason "..."` if it doesn't. You can also `contest` a `dismissed` report you believe is real — it routes back to the team. Contests are bounded (a cap, and the team may mark a dismissal final); past those, contact support. You can only list/read/act on reports **you** filed.

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
- **Modify schemas** (collections, triggers, inbound webhooks) only in preview databases

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

## Modeling Principles for Agent Consumption

The command reference above is the *mechanics*. This is how to **combine** them so an agent-backed app is reliable. The mechanics make almost any shape possible; these principles pick the shape that makes a fallible LLM agent succeed. They are domain-neutral — apply them whether you model invoices, appointments, or tickets.

**The one rule: model it so the right thing is the only easy thing to express, and the wrong thing is impossible.** The gap between an agent that systematically drops steps and one that runs a clean full lifecycle is usually the *shape* of the schema and tools — not the prompt or the model.

1. **One intent = one atomic write.** An action that takes N separate writes is N chances to drop one and leave the system half-updated. Collapse a single user intent into a single write; when one intent genuinely spans multiple documents, use `rekor batch` so they all commit or all roll back.
2. **State lives on the entity — don't duplicate it.** Keep each piece of state in exactly one place, on the entity it describes. Mirroring a status onto a second document forces the agent to update both in sync, recreating the multi-write problem. Link or query the source of truth instead of copying its fields.
3. **Expose the job, nothing more.** An agent-facing tool should express the task and nothing else. Push machinery — sort, limit, offset, field selection, the raw filter DSL — into endpoint config with `expose_*: false` plus server-side `default_*` (or the `agent_minimal` preset). The agent fills meaning via typed `filterable_fields` on reads and `writable_fields` on writes (the tool sees only the fields its job sets), not plumbing.
4. **Guards belong in config, not arguments.** Enforce invariants server-side where the agent can't forget them, not as instructions it must follow. A `precondition` makes a state-dependent write race-free and invisible to the agent; schema `required` + enums reject malformed input. The rejection message is the recovery channel — keep it legible and actionable so the agent self-corrects.
5. **Narrow the input space.** Every degree of freedom is a way to get it wrong. Constrain with enums (a closed set to pick from), typed `filterable_fields` (native params instead of free-form filter strings), stable `external_id` keys (idempotent upsert, no invented ids), and `x-fk` on writes (a reference must resolve to a real document).
6. **Model for the fallible agent, not the ideal one.** Agents skip steps, send empty strings for fields they didn't fill, and invent ids. `required` rejects the missing field, enums reject the made-up value, `x-fk` rejects the invented reference, `precondition` rejects the out-of-order write — each turns a silent corruption into a clear, correctable error.
7. **Separate the operator surface from the agent surface.** Keep the surface that *authors config* apart from the one that *invokes it*. Broad human/CI tokens design schemas and endpoints; endpoint-bound tokens (`rekor tokens create-for-endpoint`) only call the curated tools and can't reach past them. Name tools by action and cardinality (`book_slot`, `list_invoices`, `assign_payment`) so the name itself tells the agent what the tool does.

**Also:** keep data **canonical and self-describing** — declare datetime fields `format: date-time` and set the collection's `x-timezone` so every value carries its offset, and prefer readable enums over opaque codes, so a document can be reasoned about without outside context. And **optimize the common path deliberately** — make the single most frequent action one well-named tool call, accepting more friction on the rare one.

### Recipes: principle → the feature that realizes it

The principles above are agent-layer and backend-neutral; these are the specific endpoint knobs that implement them — the lookup from intent to implementation. Each knob is detailed in the endpoint section above.

- **One tool = one intent; curate the surface** — distinct `names` per operation (`names: { "list": "search_invoices" }`), even two tools on one collection; a curated `filterable_fields` list.
- **Hide machinery (filter / sort / limit / offset / fields)** — `expose_*: false` per param (plus a server-side `default_*`), or the `agent_minimal` preset that hides them all at once.
- **Constrain a closed set** — `filterable_fields[].enum`.
- **Constrain a structured id or code** — `filterable_fields[].pattern` (e.g. `^pat-[0-9]+$`); published to the tool schema and enforced at runtime.
- **Require a non-empty value** — automatic: discrete pick-a-value params reject empty and whitespace-only input (a placeholder like `""` is rejected, not silently matched).
- **Fail loud + actionable, never silent** — `x-fk` on write fields; the empty / `enum` / `pattern` rejections on read params (read/write parity).
- **Make a state-dependent write atomic + race-free** — a `create`/`update` tool with a `precondition` (compare-and-set; invisible to the agent; 409 on conflict).
- **Nudge the agent toward the right value** — `filterable_fields[].description`: name the field's kind, source, and when to use it; use placeholder shapes, never real data values.

**One gotcha:** an `enum` invites the agent to *pick* a value — right for a field it should always fill, wrong for an optional, usually-omitted one (a fixed set nudges it to choose rather than skip). For an optional filter, prefer a `pattern` (or a plain free param) so omitting it stays the natural default.

## Best Practices

- **Use external IDs** for idempotent upserts — retry-safe, no duplicates.
- **One database per context** — keeps data isolated (per test run, per agent, per environment).
- **Preview for schema changes** — modify collections, triggers, inbound webhooks in a preview database. Ask a human to promote.
- **Batch for atomicity** — when multiple writes must succeed or fail together.
- **Relationships over nested data** — link documents instead of embedding. Enables traversal and flexible queries.
- **Attachments for large or binary content** — document data and relationship metadata are for structured JSON and are capped at ~1 MiB. Store PDFs, images, and other files as attachments (`rekor attachments upload <collection> <id> --filename <name> --file <path>`) and reference them from the document; inlining base64 blobs is rejected.
- **Schema first** — define the collection schema before writing documents. Validation catches bad data early.
- **Query delay after writes** — single-document reads (`documents get`, `relationships get`) reflect writes immediately; `sql` and `query-relationships` may take a moment to catch up. If you need to query right after writing, wait briefly or use direct gets.
- **Set token expiration** — use `--expires-at` for short-lived tokens. Revoke unused tokens promptly.
- **Save tokens immediately** — the raw token is shown only once on creation. Store it securely before closing the terminal.
