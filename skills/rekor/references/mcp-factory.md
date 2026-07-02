# MCP Factory — full toolset config reference

Deep grammar for MCP Factory toolsets. The SKILL.md **MCP Factory** section covers the concept, the basic CLI, the spec formats, and the named menu of advanced knobs; read this file when you actually configure a curated toolset.

## Contents

- [Full `--config` JSON](#full---config-json)
- [Tool names (`name` / `names`)](#tool-names-name--names)
- [Typed filter params (`filterable_fields`)](#typed-filter-params-filterable_fields)
- [Hiding machinery + server-side defaults (`expose_*` / `default_*` / `agent_minimal`)](#hiding-machinery)
- [Curated write surface (`writable_fields`)](#curated-write-surface-writable_fields)
- [Conditional writes (`precondition`)](#conditional-writes-precondition)
- [Guarded + unguarded writes on one collection](#guarded--unguarded-writes-on-one-collection)
- [Selecting a named write binding (`bindings`)](#selecting-a-named-write-binding-bindings)
- [Lenient list-tool arguments](#lenient-list-tool-arguments)
- [Which database serves the toolset](#which-database-serves-the-toolset)

## Full `--config` JSON

Use `--config` with full JSON (or `--config @toolset.json`) for advanced control over tool names, descriptions, filters, and writes:

```bash
rekor toolsets upsert invoicing-agent --database my-ws --config '{
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

## Tool names (`name` / `names`)

Each generated tool is named `<op>_<name>` — `name` is the base noun, defaulting to the collection id (so `invoices` → `get_invoices`, `list_invoices`). Set `name` to choose the base (e.g. `"name": "invoice"` → `get_invoice`). Override an individual tool with `names`, a per-operation map of literal names (`{ "list": "search_invoices" }`), so a tool reads as the job it does. Relationship ops map to `link_`/`list_`/`unlink_` over their base. Tool names must be unique across the toolset. Only `name` and `names` control naming — there is no `name_override` key, and every key inside `names` must be a valid operation. An unknown or deprecated key on a tool is rejected at config-write, so a stray key can never be silently accepted and then ignored.

## Typed filter params (`filterable_fields`)

Expose chosen fields of a collection as typed parameters on its generated `list` tool, derived from the collection schema — so the agent fills native arguments instead of writing a filter expression. Each field becomes a parameter shaped by its type:

- an `enum` or boolean field → an exact-match param (the agent can only pick a valid value)
- a number or date/date-time field → a range pair (`<field>_min`/`<field>_max`, or `<field>_after`/`<field>_before`)
- a string field → a `<field>_contains` substring param

Per field you may set `param` (rename the generated param), `match` (`exact` | `range` | `text` | `any_of` | `member` — `any_of` accepts a list and matches any; `member` exposes an **array** field as a single membership value), an optional `enum` (constrain the param to a fixed set so the agent can only pick a valid value), an optional `pattern` (a regex declaring the valid format of a structured string field — e.g. an id or code shape like `^pat-[0-9]+$` — so a placeholder such as `"null"`/`"undefined"` or any wrong format is rejected with an actionable error instead of silently matching nothing; applies to an exact/any_of/member match on a plain string field, and can't be combined with `enum`), and `description`. An array field auto-exposes as a `member` param; object fields are rejected — expose a nested path (`address.city`) instead. The generic `filter` parameter stays on the tool as the escape hatch for anything the typed params can't express (OR / nesting); typed params and `filter` are combined with AND.

## Hiding machinery

When the typed params cover everything an agent needs, set `"expose_filter": false` on the tool to drop the generic `filter` parameter entirely — keeping the agent-facing tool schema small. Server-side translation of the typed params is unaffected.

The list tool's machinery params (`sort`, `limit`, `offset`, `fields`) can each be hidden the same way — `"expose_sort"`/`"expose_limit"`/`"expose_offset"`/`"expose_fields": false` — and given a server-side default (`"default_sort"`/`"default_limit"`/`"default_fields"`) that still applies when the param is hidden. `"default_fields"` takes the same shape as the `fields` param — a comma-separated string (`"external_id,data.name"`) or an array (`["external_id", "data.name"]`). A hidden `limit` surfaces a `"truncated"` flag in the response if it caps the result, so rows are never silently dropped. `"agent_minimal": true` is a preset that hides all of them (plus `filter`) with a generous default limit, leaving an inputSchema that is pure typed semantics; any explicit `expose_*`/`default_*` on the same tool overrides the preset.

## Curated write surface (`writable_fields`)

The write-side mirror of `filterable_fields`. On a `create`/`update` tool, list exactly the fields the tool may set — an allowlist that does two things at once:

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

## Conditional writes (`precondition`)

Give a `create`/`update` tool a `precondition` — a Filter DSL expression checked against the document's **current** state before the write applies. If it doesn't hold, the write is rejected with a 409 conflict and nothing changes; if it holds, the write proceeds. This turns a fragile read-then-write into one correct, race-free call: model a bookable slot as a document and make "book it" a guarded update, so two agents can't both book the same slot.

```json
{
  "collection": "slots",
  "operations": ["update"],
  "names": { "update": "book_slot" },
  "precondition": { "field": "data.status", "op": "eq", "value": "free" }
}
```

The guard is **invisible to the agent** — it's part of the toolset config, never a tool parameter — so the agent just calls `book_slot(...)` and the booking integrity is enforced for it. Paths address the current document as `data.<field>`, `version`, or `id` (e.g. `{ "field": "version", "op": "is_null" }` = create-only-if-absent). One precondition per tool; the `search` operator isn't allowed. Idempotent writes (retries that shouldn't double-create) are already handled by upsert-by-`external_id` — the precondition is for cross-state guards, not retry safety.

## Guarded + unguarded writes on one collection

A `precondition` is one-per-tool, so to expose both a guarded write and a plain write on the same collection, declare two tool entries for that collection with the same operation but distinct `names`. Tool names only need to be unique across the toolset — a duplicate (collection, operation) pair is fine:

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

## Selecting a named write binding (`bindings`)

When a proxy-backed collection's source declares **named write bindings** (a `create`/`update`/`delete` op given as a map of named endpoint variants — see `references/external-sources.md`), a write tool picks which one it dispatches to with a `bindings` map: `{ "collection": "bookings", "operations": ["create"], "names": { "create": "book_trial" }, "bindings": { "create": "trial" } }`. This is what keeps one canonical collection (`bookings`) serving a backend whose writes fan out to several endpoints: declare two `create` tools with distinct `names` (`book_trial`, `book_makeup`) selecting different bindings (`trial`, `makeup`) of the same source. The agent never sees a `binding` parameter — it's injected server-side, exactly like `precondition`. Required when the bound op is a binding map with no `default`; rejected when the op is a single endpoint or names a binding the source doesn't declare (validated at config-write and re-checked at promotion).

## Lenient list-tool arguments

The generated `list` tools are lenient about how structured arguments arrive: `filter` is always a JSON-encoded Filter DSL string, while `sort` and any multi-value (`any_of`) parameter accept **either** the native array **or** a JSON-encoded string of it — so an agent that serializes array arguments as strings still works. (`sort` is the same JSON array of `{"field","direction"}` terms described in the Documents section of SKILL.md.)

## Which database serves the toolset

`mcp.rekor.pro/t/{slug}/mcp` resolves the toolset from the database your **token** is scoped to. So:

- **Production:** promote the toolset, then connect with a token scoped to the production database (`my-ws`).
- **Preview (sandbox testing):** connect with a token scoped to the **preview database id** (`my-ws--<preview-slug>`) to serve the not-yet-promoted toolset.

If the slug can't be resolved for your token's database (unknown toolset, or one that only exists in a preview you aren't scoped to), `initialize` returns a clear JSON-RPC error telling you to scope to the preview database id or promote — it won't silently hand back a session with zero tools.

**Promotion blocking.** Promotion is blocked if it would break a published toolset — removing a collection or relationship type the toolset exposes, a field its typed filters, `writable_fields`, or a `precondition` depend on, or a named write binding a tool's `bindings` selects — so promote the toolset together with the schema change (a dry run lists any such conflicts first).
