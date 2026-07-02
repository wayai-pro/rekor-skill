---
name: rekor
version: 1.34.0
description: |
  Set up and operate Rekor — a headless system of record for AI agents. Use when:
  installing the `rekor` CLI, authenticating, creating a database, defining the first
  collection, working in preview and promoting to production, querying documents via SQL,
  managing relationships, uploading file attachments, configuring inbound webhooks/triggers/batch
  operations, backing a collection with an external API (external sources), storing
  credentials in the secret vault, importing or exporting tool definitions across
  providers (OpenAI/Anthropic/Google/MCP), creating curated MCP Factory toolsets,
  scoping API tokens, modeling collections and MCP toolsets so LLM agents use them
  reliably, or interpreting Rekor concepts (database, collection, document,
  relationship, preview/production, external_id).
---

# Rekor — Data Layer for AI Agents

You have access to the `rekor` CLI — the builder interface for Rekor. Use it to set up schemas, configure collections, test in preview environments, and promote to production. Production agents use MCP tools to read/write documents; external systems integrate via REST API. The CLI is your tool for designing and managing the data model.

## Agent Guidelines

- Drive Rekor through the **`rekor` CLI** when you have shell access. If you also have `mcp__rekor__*` tools in your toolset, prefer the CLI for setup and configuration; reserve MCP tools for production read/write operations.
- Only provide information from this skill, tool descriptions, or reference documentation. Do not invent URLs, paths, commands, or flags.
- Schema work (collections, inbound webhooks, triggers, MCP Factory toolsets) only happens in **preview** databases. Always create or use a preview before changing schemas.
- **Identifiers are permanent.** The `id` of a database, collection, relationship type, Tool, and MCP Factory toolset *is* its identity and is **immutable** — chosen once at creation and never renamed. Display names and descriptions stay editable, but never the `id` (a relationship type has no separate name — its `id` is also its label). There is no rename: to "rename" one of these you create a new entity and migrate its data (the old one is left untouched). So choose clear, stable, lowercase-slug ids up front (e.g. `patients`, `treated_by`).
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

Schema changes (collections, inbound webhooks, triggers, MCP Factory toolsets) are blocked in production databases. Create a preview to do schema work:

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

**MCP Factory toolset** — for purpose-built MCP tools per agent (recommended for LLM agents):

See the [MCP Factory](#mcp-factory-custom-toolsets) section below.

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
rekor databases create-preview <prod-id> --name <preview-slug> [--description <desc>] [--integrations <enabled|disabled>]
rekor databases update <preview-id> --integrations <enabled|disabled>   # preview-only eval toggle
rekor databases list-previews <prod-id>
rekor databases promote <prod-id> --from <preview-id> [--dry-run] [--collections <ids>] [--triggers <ids>] [--inbound-webhooks <ids>]
rekor databases promotions <prod-id>
rekor databases rollback <prod-id> --promotion <promotion-id>
```

Tags let you group databases (e.g., `client:acme,billing`). Filter with `--tag`. Promotion is **human-only** (see Environments). Promote selectively with `--collections`/`--triggers`/`--inbound-webhooks` (omit to promote everything). `promotions` lists prior promotions; `rollback` reverts one by ID.

**Eval mode (`--integrations disabled`).** A preview can run with its **external integrations disabled** (default `enabled`). When disabled, every external edge goes inert — collection sources, outbound `external_write` triggers, and inbound-webhook hydration — and the preview serves its own **seeded** data with the exact same schema, tools, and field mappings, **without calling the external systems**. That makes a preview a deterministic, prod-safe target for **agent evals**: exercise your agent's policy against the same canonical surface without polluting a production-only upstream or hitting live rate limits/PII. Seed it by writing documents while disabled (writes to source-backed collections land locally), flip to `enabled` to test the real integration, and re-clone from production to stay drift-free. **Production databases are always `enabled`** — the toggle is preview-only.

### Config as Code (pull / push)

Manage a database's whole config — collections, relationship types, inbound webhooks, triggers, MCP toolsets — as version-controlled YAML files instead of one-off commands. Files live under `rekor-ws/databases/<db>/`, one file per entity, so changes review cleanly in a pull request.

```bash
rekor pull <preview>                 # write the preview's config + a read-only mirror of linked production
rekor push <preview> [--dry-run]     # apply your local files to the preview (--dry-run shows the diff only)
rekor push <preview> --prune         # also delete entities that exist on the server but not in your files
rekor use <preview>                  # bind THIS git worktree to a database (push/pull refuse a different one)
rekor unbind                         # clear this worktree's database binding
```

- **Previews are the only edit target.** `push` only writes **preview** databases (config is never edited directly in production). When you `pull` a preview, the linked production config is also written beside it as a **read-only reference** folder (`rekor-ws/databases/<prod>/`) so you can see the live production config while iterating — it's marked read-only (`push` refuses it and bare `pull`/`push` auto-selection ignores it). `pull <production>` directly writes only that read-only reference. To ship, run `rekor databases promote` as usual.
- **Auto-create a preview.** Scaffold `rekor-ws/databases/<name>/database.yaml` with `origin_database_id: <prod-id>` (and no `database_id`); `rekor push` creates the preview from that production database, writes the new id back, and applies your files.
- **Worktree binding (routing guard).** Each git checkout (main or linked worktree) can be bound to one database. `pull`/`push` refuse to run against a different database once bound — catching the common mistake of a prompt landing in the wrong terminal/worktree and clobbering another preview's config. The binding is set automatically on the first successful `pull`/`push` into an unbound checkout (and on creating a new preview), is per-worktree and never committed, and complements `.rekor.yaml` (which pins the org repo-wide). `rekor status` shows the current binding. **If `pull`/`push` errors with a binding mismatch, stop and ask the user before doing anything else** — it usually means the prompt was meant for a different worktree. Do **not** run `rekor unbind` / `rekor use` without explicit user instruction in the current session; changing the binding is a routing decision, like switching which database the user thinks you're working on.
- **Secrets are never written to files.** Inbound-webhook/trigger and external-source secrets are stripped on `pull`. On `push`, a newly added inbound webhook/trigger gets a fresh secret (printed once — save it); existing secrets are left untouched. Manage secret values with `rekor inbound-webhooks` / `rekor triggers` / `rekor secrets`.
- **Deletions are opt-in.** Because deleting a collection also removes its documents, `push` is additive by default: entities missing from your files are reported but kept. Add `--prune` to delete them.
- **Renaming a collection is re-seed, not in-place.** A collection's `id` is its identity and must equal its filename (`collections/<id>.yaml`), so renaming means updating **both** the filename and the inner `id:` field together (changing only one errors; or delete the inner `id:` so it defaults to the filename). Either way it's a **create-new + orphan-old**, not an in-place rename: the diff shows `+ <new>` / `- <old>`, `push` creates a new **empty** collection, and the old one (with all its documents) is left untouched — documents do **not** migrate. To rename and keep the data: (1) `push` to create the new empty collection, (2) re-seed its documents (e.g. `rekor documents upsert`, or batch), then (3) `push --prune` to delete the orphaned old collection, which cascades its documents. The same filename-is-`id` rule applies to relationship types and the other config entities.

### Templates

Ready-made data layers you can pull and stand up in minutes — each bundles a database's collections, relationship types, and a custom toolset for an AI agent.

```bash
rekor template list                                       # browse available templates
rekor template pull <slug> [--lang en|pt|es] [--dry-run]  # write a template's data layer into rekor-ws/databases/<slug>/
rekor template pull <slug> --force                        # overwrite existing files in the target folder
```

- **`pull` seeds a config-as-code folder.** It writes the template's collections, relationship types, and MCP toolset into `rekor-ws/databases/<slug>/` (id defaults to the slug; override with `--database <id>`), pre-wired with an `origin_database_id` so the normal flow stands it up: `rekor databases create <id> --name "…"`, then `rekor push <id>`, then `rekor databases promote <id> --from <preview>`. Once promoted, the template's MCP toolset is live for your agents.
- **`pull` won't overwrite your edits.** If the target folder already has files the template would write, `pull` refuses and lists them (your other files are left untouched) — pass `--force` to overwrite, `--database <new-id>` to write elsewhere, or `--dry-run` to preview the file set first.
- **`--lang`** selects a localized variant; an untranslated language falls back to the default (with a note).

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

Every field is searchable by default — `search` needs no setup. Exact filters and `search` compose in one query: put exact conditions alongside the `search` leg so the exact ones narrow the set and `search` ranks within it. To **tune** how a field is matched — `x-search` modes (`name`/`text`/`fuzzy`), `threshold`, and `searchable: false` — see **`references/querying.md`**. Search runs against the latest synced data, so a document written a moment earlier may take a brief moment to appear.

**Sorting.** `--sort` takes a **JSON array** of terms, each `{"field":"data.<field>","direction":"asc"|"desc"}` — e.g. `--sort '[{"field":"data.created_at","direction":"desc"}]'`. Multiple terms sort lexically (first term primary). `direction` defaults to `asc` if omitted. The colon shorthand `data.name:asc` is **not** accepted — pass the JSON array form. Omit `--sort` when using `search` to keep the relevance ranking.

**Datetimes.** Declare datetime fields in the collection schema with `"format": "date-time"` (or `"format": "date"` for date-only). Rekor stores them in a single canonical form: a naive value (no offset) gets the collection's timezone attached, so `2026-06-11T13:00:00` is stored and returned as `2026-06-11T13:00:00-03:00` — the same shape from both `query` and a single-document read, with no UTC math required of you. The timezone resolves collection `x-timezone` → database `settings.timezone` → UTC. **Filtering is instant-aware:** `eq`/`neq`/`in`/`not_in` and the range operators match datetimes by point-in-time, so the exact representation you pass (`T` vs space, with or without offset) doesn't change the result. Use `eq` for an exact datetime, not `like` (plain substring). Raw `rekor sql` is **not** rewritten this way — prefer the Filter DSL `query` for datetime conditions. Setting the timezone (CLI `--timezone`, REST, merge-vs-replace `settings` semantics) is covered in **`references/querying.md`**.

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
```

For the full example gallery — array aggregation, `ARRAY JOIN` to explode embedded arrays, CTEs joining documents with relationships, `--param` binding, and fuzzy/`jaroWinklerSimilarity` ranked text match — see **`references/querying.md`**.

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

### Integration Modeling (canonical-first)

The integration edges below — inbound webhooks, triggers, and external sources — connect a collection to an external system. This is how to decide *which* edge to use and how to shape the entity behind it. Companion to the **Modeling Principles for Agent Consumption** section (near the end), which shapes the agent-facing tool surface; this shapes how the data is backed.

**The one rule: model the entity canonically first — agnostic to any integration — then decide, per operation, how it's backed.** The collection IS the entity: a clean schema with business-named properties, sensible enums and defaults — never the upstream payload's shape. Integration lives at declarative edges, never smeared into the schema or tool surface. A consumer (an agent, a query) should not be able to tell whether a field is rekor-native or mirrored from a legacy system.

- **Model the canonical first.** Derive the schema and tool surface from the domain and the agents' real journeys, not from any backend's payload. Put each business rule at the most reliable layer it fits — schema (enums, `required`, defaults) > trigger (reactive invariant) > tool constraint (`writable_fields`/`precondition`) > prompt — and never encode an enforceable data invariant as a prompt rule. Right-grain it: model only what the journeys touch and extend additively (speculative schema is harder to remove than to add).
- **Swap-test.** If re-backing an entity (legacy → native → a different system) would force a schema or tool-surface change, the model leaked the integration — fix the model. Only an integration-agnostic model is swappable: the same canonical contract can be served natively in one database and proxied to a legacy API in another by swapping only the source's `field_mapping`. Backfill catalog gaps the legacy system lacks with native collections in the same database, exposed through the same toolset.
- **Offline evals are a free payoff — the swap-test from the other side.** The preview-only `--integrations disabled` eval toggle (see **Databases** above) works *precisely because* the model is integration-agnostic: where the swap-test *re-backs* an entity without touching its surface, the eval toggle *un-backs* it without touching its surface — disable the source and the same canonical schema, tools, and field mappings keep serving, now from seeded local data. So model canonical-first and you get deterministic, prod-safe agent evals for free: it's one principle, not a separate feature.
- **Native vs proxy is a per-COLLECTION choice; a source's operation blocks declare which operations the upstream exposes.** A collection with no sources is native (all CRUD local); a collection with a source is proxy-backed — every write goes upstream and reads resolve against the upstream (it stores nothing locally). The source's `list`/`get`/`create`/`update`/`delete` blocks pick *which* operations the upstream supports (read-only `get`/`list`, or full read-write); an operation the source omits is **unsupported**, not silently native. You don't split native and proxy across the operations of one collection — you mix them across collections: **one toolset freely composes native-backed and proxy-backed tools** (a proxied `invoices` beside native `plans`/`activities` you backfilled).
- **Don't route native-vs-proxy per tool or per field.** Native-vs-proxy is a per-collection choice, never per-tool; `writable_fields` restricts *which* fields a tool writes, never *where*. (Two proxy tools on the same op MAY select different **named write bindings** of the source — see External Sources — but that only picks which upstream *endpoint* a write hits; it stays all-proxy and never makes one tool native and another proxy.) A "native read + proxy write on one collection" mix is a read-consistency hole (the native read won't reflect the proxy write until something syncs it) — and once you add that sync, native-write + an `external_write` trigger is strictly better. The two coherent shapes are **all-proxy** or **all-native + sync**; the per-tool mix is the incoherent middle.
- **Prefer the native mirror; reach for proxy only when forced.** A native collection kept in sync gives local query/join/filter, speed, transactional writes, audit, and availability decoupled from the upstream. Choose proxy-on-read only when staleness is unacceptable, a write must be synchronously confirmed by the upstream system of record, or the dataset is too large/cold to mirror.
- **Hybrid entities (mirrored fields + a rekor-owned overlay) live in ONE collection.** A field the upstream lacks (a workflow `status`, a tag, a score) belongs on the same document as the mirrored fields — integrated query/filter demands it (you can't `list where status=X` if `status` lives in another collection). Splitting an entity to separate ownership trades a real query capability for bookkeeping; don't. Exception: an overlay that is itself a richer entity with its own lifecycle/history → a related collection + relationship. A scalar attribute is never a reason to split.
- **`field_mapping` is one bidirectional contract, reused at every edge.** The same source contract powers proxy reads, write-through, inbound-webhook hydration (`source_binding`), and `external_write` out. Define the translation once; the edges reuse it.
- **Reference webhooks: hydrate via the source's `get`, never follow the payload's callback.** Most mature webhooks are reference-style ("record X changed, go fetch it") — small payload, no stale snapshot, self-correcting ordering. Resolve them by extracting the id and fetching through the bound source's `get`. A payload-supplied callback URL is an SSRF vector and is ignored.

**Pattern catalog** — pick the edge by what the upstream offers and what you need:

| Pattern | Source / edge shape | Reads | Writes | Use when |
|---|---|---|---|---|
| **Native** | `sources: []` | local | local | rekor owns the data |
| **Proxy-read** | source `list`/`get` | live upstream | — | freshness-critical or too-big-to-mirror reference data |
| **Write-through** | source `create`/`update` (+ `get`/`list` to read) | live upstream | → upstream (id returned) | upstream is system of record; you want synchronous confirmation, no local mirror |
| **Push-synced mirror** | hydrating inbound webhook + bound source `get` | local | local | upstream pushes change notifications; you want local query of a fresh mirror |
| **Bidirectional mirror** | hydration in (`merge`) + `external_write` trigger out (`watched_fields`) | local | local (→ upstream via trigger) | a hybrid entity synced both ways, all-native tools |

The **bidirectional mirror** is the idiomatic home for a hybrid entity: one native collection, all-native tools, sync as declarative edges. Use hydration `merge` so a re-sync refreshes only the mapping-owned fields and preserves your overlay, and `external_write` `watched_fields: "mapped"` so an overlay-only change doesn't push a no-op upstream. The hydration write target must be **native** (no source) — the out-edge is the `external_write` trigger, not a proxy write on the same collection — and `skip_inbound_webhook_writes` keeps a re-sync from re-firing the trigger (no loop).

**Identity keys.** Catalog / reference entities (plans, activities, locations) → key by a stable **slug** that doubles as `external_id`, so cross-collection `x-fk` references resolve. People / relations (members, bookings, enrollments) → key by rekor's internal **uuid**.

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

**Hydrating webhooks (notification + fetch).** Many systems send *reference-style* webhooks — "record X changed, here is its id" — instead of the full record (Stripe recommends re-fetching by id; Shopify, Salesforce, HubSpot, legacy CRMs do the same). Point such a webhook at a collection's read source and Rekor will **fetch the full record itself** on each delivery, then store the canonical document:

```bash
rekor inbound-webhooks create --database <ws> --name orders-ref --secret <token> \
  --collection-scope orders \
  --source-binding '{"collection":"orders","source":"shop_api"}' \
  --ingest-auth '{"type":"static_header","header":"X-Account-Key"}' \
  --hydration '{"id_path":"record_id","event_path":"event","event_map":{"ResourceDeleted":"delete"}}'
```

On a delivery, Rekor reads the record id from the thin body (`id_path`), calls the bound source's read endpoint to pull the full record, canonicalizes it through the source's `field_mapping`, and upserts it by `external_id` — the inbound twin of a proxied read. The event field (`event_path` + `event_map`) routes to upsert or delete (a delete skips the fetch and removes the mirrored record); everything else defaults to upsert. The fetch + write happen **asynchronously** with automatic retry/backoff, so the sender gets an immediate accept; check progress with `rekor inbound-webhooks deliveries --database <ws>`. Requires `--source-binding` (it provides the read endpoint) and a single native `--collection-scope`. **Security:** Rekor only ever fetches the *source-configured* URL with the extracted id — a callback URL inside the payload is ignored, never followed. Reference-style senders typically pair with `--ingest-auth` (a static header), since they don't sign each delivery.

**Keep your own fields on re-sync — `merge`.** By default each re-sync replaces the whole document, so any field you added locally (a workflow status, tag, score, assignment, internal note) is overwritten. Add `"merge":true` to `--hydration` and a re-sync refreshes **only** the fields the source's mapping owns and **preserves** everything else you put on the document — so one collection can hold both the upstream mirror *and* your local enrichment, and you can still list/filter the entity by your own field (e.g. `status=qualified`). The first sync still creates the record from the mapped fields; a field the upstream later clears is reflected; the merged document is still schema-validated. Example: `--hydration '{"id_path":"record_id","merge":true}'`.

### Triggers (data out)

Triggers fire automatically when documents change. A trigger's action is one of:
- **webhook** — an HMAC-signed HTTP POST to an external system (`--url`/`--secret`).
- **internal_write** — Rekor updates a *referenced* document for you, with no external service. On a matching write it locates a target document and applies a guarded patch (an optional compare-and-set `precondition` so the update only lands when the target is still in the expected state). Ideal for "when A changes, update related B" — e.g. booking an appointment flips its slot to busy only if it was free. Pass it via `--action` (see below); document.created/updated only. Two modes: **`async`** applies the patch just after the triggering write (eventual; a precondition miss is recorded but the triggering write still stands), and **`transactional`** applies it atomically *with* the triggering write — if the `precondition` misses, the whole write is rejected and rolled back (nothing is created). Use `transactional` when the two writes must succeed or fail together (e.g. don't create the appointment unless the slot was free).
- **external_write** — Rekor writes the changed document *out* to an external system by reusing an external source's write path — its field translation, endpoint, body shape, and auth — with no executor. Use it to keep a native collection mirrored to a legacy API: store records natively in Rekor and have each write pushed upstream in the upstream's own shape. The action names the source: `{type:"external_write","collection":"<source-owner>","source":"<name>","op":"create"}`. `collection` is the collection that *defines* the source (a native mirror points at a sibling source-backed collection; defaults to the triggering collection if omitted); `op` defaults from the event (created→create, updated→update, deleted→delete) and is overridable. On a **create**, Rekor reads the upstream-assigned id from the response and links it back onto the triggering document as its `external_id` (so the native record points at its upstream counterpart, and retries don't duplicate) — opt out with `link_external_id:false`. Updates/deletes are dispatched only for documents already linked to that source. By default every matching update is pushed upstream; add `watched_fields` to scope **update** dispatch to writes that actually changed an upstream-mapped field — `"mapped"` watches exactly the fields the source maps upstream (so a write touching only a Rekor-owned field — one the source doesn't map out — skips the redundant no-op upstream push), or pass an explicit array of data-field names. Creates and deletes always dispatch. Delivery is async, signed, retried, and dead-lettered like a webhook (inspect with `rekor triggers deliveries`); document.created/updated/deleted only.

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

Each source defines the menu below (full grammar, every option's rules, and a worked legacy-upstream example in **`references/external-sources.md`**):

- `name` — must equal the document's `external_source`.
- `auth` — header + value template; secret inline or `vault:<name>`. An **inline** secret is environment-local — promotion never copies it, so set it on production directly (a newly promoted source stays unconfigured, requests fail with a clear error, until you do). A `vault:<name>` reference travels by reference.
- `field_mapping` — **optional** (omit for identity passthrough). `to_external`/`to_rekor` map between your schema and the upstream's field names — a simple rename or a rich rule (`transform`, `values` [always Rekor-keyed], `default`, `date_format`, `array_mode`, `target`). Advanced shapes: **split** one field to several params (`targets`), **destructure** a composite value (`separator`+`parts`), **compose** a Rekor field from several upstream fields (`computed` templates), and **per-operation overrides**.
- `get` / `list` / `create` / `update` / `delete` — per-operation endpoint templates, each with its own `url` (tokens `{{external_id}}`, `{{query.*}}`, `{{data.*}}`, `{{current.*}}`/`{{prior.*}}`, `{{auth.*}}`) and `method` — so RPC-style verb paths and all-POST upstreams work directly. `response_path`/`total_path`/`success_path`/`message_path` unwrap envelopes.
- **Named write bindings** (`create`/`update`/`delete`) — declare a write op as a map of named endpoint variants when one canonical entity's writes fan out to several backend endpoints; the caller picks one by name (via a tool's `bindings`), keeping ONE collection.
- `id_path` — where the `external_id` lives in each raw upstream record (dot-path, composite template, or create-only/request-templated variants) for upstreams not keyed on `id`.
- `forward_filters` (on `list`) — forward the agent's `eq` filter conditions upstream (allowlisted `fields`, `query`/`body` target); also exposed to read mappings as `{{filter.<field>}}`.
- `static_body` / `body_template` / `request_encoding` — constant, templated, and form-encoded request-body shaping (e.g. put the id in the body for legacy verbs).
- `injections` / `executor_secrets` — extra per-request vault secrets in a header/body, and named credentials an executor pulls at dispatch (e.g. an mTLS cert).
- `cache_ttl`+`stale_if_error` / `signing` / `timeout_ms` / `breaker` — read-through caching, HMAC signing (for executors), per-request timeout, and the per-source circuit breaker.

URLs must be absolute `https` (or `http` with `allow_insecure_http: true`) and tokens may appear only in the path/query (never the host) — an SSRF guard rejects otherwise.

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

Import tool definitions from any LLM provider as collections (`rekor providers import <provider>`), export collections as tool definitions (`rekor providers export <provider>`), or turn a provider-native tool call into a document (`rekor providers import-call`). Supported providers: `openai`, `anthropic`, `google`, `mcp`. Command forms and payload shapes in **`references/providers.md`**.

### MCP Factory (custom toolsets)

Create purpose-built MCP servers from your database collections. Each toolset serves domain-specific tools — agents see `create_invoice`, `list_payments`, not generic Rekor operations.

```bash
# Create a curated toolset
rekor toolsets upsert invoicing-agent --database my-ws \
  --name "Invoicing Agent" \
  --tool "invoices:get,list" \
  --tool "payments:create,get,list" \
  --relationship "invoice_payment:list" \
  --batch "invoices:create,update" --batch "payments:create" --batch "invoice_payment:create" \
  --sql-query

# Get the MCP connection URL (Streamable HTTP)
rekor toolsets url invoicing-agent
# → https://mcp.rekor.pro/t/invoicing-agent/mcp

# List all toolsets
rekor toolsets list --database my-ws

# Get toolset config (with resolved schemas)
rekor toolsets get invoicing-agent --database my-ws --resolved

# Delete a toolset
rekor toolsets delete invoicing-agent --database my-ws
```

**Tool spec format**: `collection:op1,op2` (operations: `create`, `get`, `list`, `update`, `delete`)
**Relationship spec format**: `rel_type:op1,op2` (operations: `create`, `list`, `delete`)
**Batch spec format**: `collection_or_rel:op1,op2`

**Advanced control** comes through `--config` with full JSON (or `--config @toolset.json`) — custom tool names/descriptions, typed params, curated writes, and guards. The knobs, one line each (full grammar + JSON examples in **`references/mcp-factory.md`**):

- **`name` / `names`** — a tool is named `<op>_<name>` (base defaults to the collection id); override per-operation so a tool reads as the job it does (`{ "list": "search_invoices" }`). Names must be unique across the toolset.
- **`filterable_fields`** — expose chosen fields as typed `list` params (native arguments instead of a raw filter). Per field: `param`, `match` (`exact`/`range`/`text`/`any_of`/`member`), `enum`, `pattern`, `description`.
- **`expose_*: false` + `default_*`** — hide machinery params (`filter`/`sort`/`limit`/`offset`/`fields`) and set server-side defaults; `agent_minimal: true` hides them all at once.
- **`writable_fields`** — the write-side mirror: an allowlist of exactly the fields a `create`/`update` tool may set (least-privilege intent tools) that also generates a rich typed `data` schema from the collection schema.
- **`precondition`** — a Filter DSL compare-and-set on a `create`/`update` tool, checked against the document's current state; a miss is a 409 and nothing changes. Invisible to the agent — turns a fragile read-then-write into one race-free call (e.g. `book_slot` only if the slot is still `free`).
- **`bindings`** — pick which **named write binding** of a proxy source's op a write tool dispatches to (see External Sources), keeping one canonical collection whose writes fan out to several endpoints.

Connect agents to the toolset URL with a token scoped to exactly one database. The agent sees only the tools you configured — fully domain-specific, no Rekor concepts. For least-privilege, mint a token bound to the toolset in one step: `rekor tokens create-for-toolset <slug> --database <db>` (or pass `--mint-token` to `rekor toolsets upsert`). A toolset-bound token's authorization IS the toolset's tool surface — exactly those collections and operations, relationships, batch, and SQL only if you enabled it, nothing else — so a leaked token can't reach beyond the tools you exposed, and you can rotate or revoke one per agent.

Toolsets can only be created/modified in preview databases. Promote to production when ready; promotion is blocked if it would break a published toolset (a removed collection/rel-type, or a field its typed filters/`writable_fields`/`precondition`/`bindings` depend on) — a dry run lists conflicts first. `mcp.rekor.pro/t/{slug}/mcp` resolves the toolset from the database your **token** is scoped to, so connect with a production token for the promoted toolset or a preview-database token to sandbox-test the not-yet-promoted one (`references/mcp-factory.md` covers the resolution rules).

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

**Permissions**: `read:documents`, `write:documents`, `read:collections`, `write:collections`, `read:relationships`, `write:relationships`, `read:attachments`, `write:attachments`, `read:inbound_webhooks`, `write:inbound_webhooks`, `read:triggers`, `write:triggers`, `read:toolsets`, `write:toolsets`, `read:databases`, `write:databases`, `read:audit` (read-only; grants change-history access, admin-gated, not implied by other grants), or `*` for all.

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

The CLI checks for a newer published version in the background and prints a one-line notice when an update is available. When this skill is installed in your repo, it also checks whether a newer skill is published and nudges you to run `mkdir -p .claude && npx skills add wayai-pro/rekor-skill -y`. Disable both checks with `REKOR_NO_UPDATE_CHECK=1` (also disabled when `NO_UPDATE_NOTIFIER` or `CI` is set).

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

The command reference above is the *mechanics*. This is how to **combine** them so an agent-backed app is reliable. The mechanics make almost any shape possible; these principles pick the shape that makes a fallible LLM agent succeed. They are domain-neutral — apply them whether you model invoices, appointments, or tickets. These shape the agent-facing tool surface; for how to **back** an entity (native, proxied, mirrored) and shape the integration edges, see **Integration Modeling (canonical-first)** above.

**The one rule: model it so the right thing is the only easy thing to express, and the wrong thing is impossible.** The gap between an agent that systematically drops steps and one that runs a clean full lifecycle is usually the *shape* of the schema and tools — not the prompt or the model.

1. **One intent = one atomic write.** An action that takes N separate writes is N chances to drop one and leave the system half-updated. Collapse a single user intent into a single write; when one intent genuinely spans multiple documents, use `rekor batch` so they all commit or all roll back.
2. **State lives on the entity — don't duplicate it.** Keep each piece of state in exactly one place, on the entity it describes. Mirroring a status onto a second document forces the agent to update both in sync, recreating the multi-write problem. Link or query the source of truth instead of copying its fields.
3. **Expose the job, nothing more.** An agent-facing tool should express the task and nothing else. Push machinery — sort, limit, offset, field selection, the raw filter DSL — into toolset config with `expose_*: false` plus server-side `default_*` (or the `agent_minimal` preset). The agent fills meaning via typed `filterable_fields` on reads and `writable_fields` on writes (the tool sees only the fields its job sets), not plumbing.
4. **Guards belong in config, not arguments.** Enforce invariants server-side where the agent can't forget them, not as instructions it must follow. A `precondition` makes a state-dependent write race-free and invisible to the agent; schema `required` + enums reject malformed input. The rejection message is the recovery channel — keep it legible and actionable so the agent self-corrects.
5. **Narrow the input space.** Every degree of freedom is a way to get it wrong. Constrain with enums (a closed set to pick from), typed `filterable_fields` (native params instead of free-form filter strings), stable `external_id` keys (idempotent upsert, no invented ids), and `x-fk` on writes (a reference must resolve to a real document).
6. **Model for the fallible agent, not the ideal one.** Agents skip steps, send empty strings for fields they didn't fill, and invent ids. `required` rejects the missing field, enums reject the made-up value, `x-fk` rejects the invented reference, `precondition` rejects the out-of-order write — each turns a silent corruption into a clear, correctable error.
7. **Separate the operator surface from the agent surface.** Keep the surface that *authors config* apart from the one that *invokes it*. Broad human/CI tokens design schemas and toolsets; toolset-bound tokens (`rekor tokens create-for-toolset`) only call the curated tools and can't reach past them. Name tools by action and cardinality (`book_slot`, `list_invoices`, `assign_payment`) so the name itself tells the agent what the tool does.

**Also:** keep data **canonical and self-describing** — declare datetime fields `format: date-time` and set the collection's `x-timezone` so every value carries its offset, and prefer readable enums over opaque codes, so a document can be reasoned about without outside context. And **optimize the common path deliberately** — make the single most frequent action one well-named tool call, accepting more friction on the rare one.

### Recipes: principle → the feature that realizes it

The principles above are agent-layer and backend-neutral; these are the specific toolset knobs that implement them — the lookup from intent to implementation. Each knob is detailed in the toolset section above.

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
