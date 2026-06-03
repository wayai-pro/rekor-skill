---
name: rekor
description: |
  Set up and operate Rekor — a headless system of record for AI agents. Use when:
  installing the `rekor` CLI, authenticating, creating a database, defining the first
  collection, working in preview and promoting to production, querying documents via SQL,
  managing relationships, configuring hooks/triggers/batch operations, importing or
  exporting tool definitions across providers (OpenAI/Anthropic/Google/MCP), creating
  curated MCP Factory endpoints, scoping API tokens, or interpreting Rekor concepts
  (database, collection, document, relationship, preview/production, external_id).
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
- **Relationship**: A typed, directed link between two documents with optional metadata.

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

### 4. Link documents

```bash
rekor relationships upsert --database my-ws \
  --source invoices/rec_abc --target customers/rec_xyz \
  --type belongs_to
```

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
rekor databases tag <id> --tags <comma-separated>
rekor databases delete <id>
```

Tags let you group databases (e.g., `client:acme,billing`). Filter with `--tag`.

### Collections

```bash
rekor collections list --database <ws>
rekor collections get <id> --database <ws>
rekor collections upsert <id> --database <ws> --name <name> --schema <json|@file>
rekor collections delete <id> --database <ws>
```

### Documents

```bash
rekor documents upsert <collection> --database <ws> --data <json|@file> [--id <uuid>] [--external-id <id>] [--external-source <src>]
rekor documents get <collection> <id> --database <ws>
rekor documents delete <collection> <id> --database <ws>
```

### SQL Query

Execute read-only SQL queries directly against database data. Supports filtering, aggregation, JOINs, CTEs, and array operations.

```bash
rekor sql "<query>" --database <ws> [--param key=value ...] [--file query.sql]
```

**Tables**: `documents`, `relationships`, `collections`, `databases`, `operations_log`

**Important**: Always include `database_id = {database_id:String}` and `deleted = false`. Queries always see the latest version of each row — the server handles deduplication.

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
```

### Relationships

```bash
rekor relationships upsert --database <ws> --source <col/id> --target <col/id> --type <type> [--id <id>] [--data <json>]
rekor relationships get <id> --database <ws>
rekor relationships delete <id> --database <ws>
rekor query-relationships <collection> <id> --database <ws> [--type <type>] [--direction outgoing|incoming|both]
```

### Hooks (inbound webhooks)

External systems push data into Rekor via hooks. Each hook provides a unique ingest URL.

```bash
rekor hooks create --database <ws> --name <name> --collection <collection>
rekor hooks list --database <ws>
rekor hooks get <id> --database <ws>
rekor hooks delete <id> --database <ws>
```

Hooks can only be created/deleted in preview databases. Promote to production when ready.

### Triggers (outbound webhooks)

Triggers fire automatically when documents change, notifying external systems via HTTP POST.

```bash
rekor triggers create --database <ws> --name <name> --collection <collection> --url <url> --events '["create","update","delete"]'
rekor triggers list --database <ws>
rekor triggers get <id> --database <ws>
rekor triggers delete <id> --database <ws>
rekor triggers deliveries --database <ws> [--status <pending|delivered|failed|dead>] [--trigger-id <id>]
```

Triggers are HMAC-signed (`X-Rekor-Signature`) and carry an `X-Rekor-Delivery-Id` for receiver dedupe. Delivery is reliable — failed attempts are retried with backoff and dead-lettered after repeated failure; inspect status with `rekor triggers deliveries`. By default, writes from hooks don't re-fire triggers (`skip_hook_writes: true`). Triggers can only be created/deleted in preview databases.

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

# Get the MCP connection URL
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
      "description_override": "Search invoices by customer, status, or date range"
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

Connect agents to the endpoint URL with a token scoped to exactly one database. The agent sees only the tools you configured — fully domain-specific, no Rekor concepts.

Endpoints can only be created/modified in preview databases. Promote to production when ready.

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

**Permissions**: `read:documents`, `write:documents`, `read:collections`, `write:collections`, `read:relationships`, `write:relationships`, `read:attachments`, `write:attachments`, `read:hooks`, `write:hooks`, `read:triggers`, `write:triggers`, `read:endpoints`, `write:endpoints`, `read:databases`, `write:databases`, or `*` for all.

**Scope fields**: `databases` (required), `collections` (optional — omit for all), `environments` (optional — `production`, `preview`, or omit for both).

Tokens enforce a privilege ceiling — you can only create tokens with equal or narrower scope than your own.

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
