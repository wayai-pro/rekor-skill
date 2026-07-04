# MCP Factory — full toolset config reference

Deep grammar for MCP Factory toolsets. The SKILL.md **MCP Factory** section covers the concept, the basic CLI, and the named menu of advanced knobs; read this file when you actually configure a curated toolset.

## The two building blocks: Actions and Toolsets

A toolset is composed from first-class **Actions**. The model is two steps:

1. **Author Actions** with `rekor actions upsert`. An Action is *one record_type + one operation* (`create` / `get` / `list` / `update` / `delete`) with an immutable slug `id`. The Action **owns** the guard/shaping config: `filterable_fields`, `expose_*`, `default_*`, `agent_minimal` for reads (`get`/`list`); `writable_fields`, `precondition`, `binding` for writes (`create`/`update`/`delete`). The Action `id` is the agent-facing tool name by default.
2. **Compose a Toolset** with `rekor toolsets upsert`. Its `actions` are **references** to Actions by id — each entry is `{ "action": "<action_id>", "action_name"?: "...", "description_override"?: "..." }`, nothing more. Relationship tools, batch, and `sql_query` are declared on the toolset itself.

A toolset `actions[]` entry only accepts `action`, `action_name`, and `description_override`. Any other key (a stray `record_type`, `operations`, `filterable_fields`, …) is rejected at config-write — the shaping config lives on the Action, not the reference.

## Contents

- [Full authoring example](#full-authoring-example)
- [Tool names (`action_name` / `description_override`)](#tool-names)
- [Typed filter params (`filterable_fields`)](#typed-filter-params-filterable_fields)
- [Hiding machinery + server-side defaults (`expose_*` / `default_*` / `agent_minimal`)](#hiding-machinery)
- [Curated write surface (`writable_fields`)](#curated-write-surface-writable_fields)
- [Conditional writes (`precondition`)](#conditional-writes-precondition)
- [Guarded + unguarded writes on one record_type](#guarded--unguarded-writes-on-one-record_type)
- [Selecting a named write binding (`binding`)](#selecting-a-named-write-binding-binding)
- [Lenient list-tool arguments](#lenient-list-tool-arguments)
- [Which base serves the toolset](#which-base-serves-the-toolset)

## Full authoring example

**Step 1 — author the Actions.** One record_type + one operation each; the `id` becomes the agent-facing tool name.

```bash
rekor actions upsert search_invoices --base my-ws --record_type invoices --operation list \
  --description "Search invoices by customer, status, or date range"
rekor actions upsert get_invoice   --base my-ws --record_type invoices --operation get
rekor actions upsert create_payment --base my-ws --record_type payments --operation create
rekor actions upsert get_payment    --base my-ws --record_type payments --operation get
rekor actions upsert list_payments  --base my-ws --record_type payments --operation list
```

**Step 2 — compose the toolset by reference.** Use `--config` with full JSON (or `--config @toolset.json`) for advanced control:

```bash
rekor toolsets upsert invoicing-agent --base my-ws --config '{
  "name": "Invoicing Agent",
  "actions": [
    { "action": "search_invoices" },
    { "action": "get_invoice" },
    { "action": "create_payment" },
    { "action": "get_payment" },
    { "action": "list_payments", "description_override": "List the payments recorded against an invoice" }
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

For a quick toolset without per-Action shaping, skip the JSON and reference Actions by id on the flags: `rekor toolsets upsert invoicing-agent --base my-ws --name "Invoicing Agent" --action search_invoices --action get_invoice --action create_payment --relationship invoice_payment:create,list --sql-query`. A `--action <id>=<surface_name>` form renames the surface tool (see below).

Relationship tools and `batch` are still declared inline on the toolset (they are not first-class Actions): a relationship entry is `{ rel_type, operations, name?, names?, description_override? }`; `batch` is `{ "enabled": true, "operations": { "invoices": ["create","update"] } }`.

## Tool names

Each Action's generated MCP tool name defaults to its **`id`** — so an Action with id `search_invoices` surfaces as `search_invoices`, and `get_invoice` as `get_invoice`. Name an Action for the job it does when you author it (`rekor actions upsert book_slot …`), rather than accepting a generic `update_slots`.

A toolset reference can override two things per surface without touching the Action:

- **`action_name`** — rename this Action *in this toolset* (e.g. `{ "action": "search_invoices", "action_name": "find_invoices" }`). Tool names must be unique across the toolset.
- **`description_override`** — replace the Action's default description for this toolset.

Tool names must be unique across the toolset, so if you reference the same Action twice (or two Actions that would collide) give one a distinct `action_name`. **Relationship** tools are named by the inline `name`/`names` map on their entry — `name` sets the base noun (link/list/unlink over it), `names` overrides a specific operation (`{ "create": "assign_payment" }`); every key inside `names` must be a valid relationship operation (`create`/`list`/`delete`), or the override is rejected at config-write.

## Typed filter params (`filterable_fields`)

A read-shape knob on a **`list`** Action. Expose chosen fields of a record_type as typed parameters on the generated `list` tool, derived from the record_type schema — so the agent fills native arguments instead of writing a filter expression.

```bash
rekor actions upsert search_invoices --base my-ws --record_type invoices --operation list --config '{
  "filterable_fields": [
    { "field": "status" },
    { "field": "issued_at" },
    { "field": "customer", "match": "text" }
  ]
}'
```

Each field becomes a parameter shaped by its type:

- an `enum` or boolean field → an exact-match param (the agent can only pick a valid value)
- a number or date/date-time field → a range pair (`<field>_min`/`<field>_max`, or `<field>_after`/`<field>_before`)
- a string field → a `<field>_contains` substring param

Per field you may set `param` (rename the generated param), `match` (`exact` | `range` | `text` | `any_of` | `member` — `any_of` accepts a list and matches any; `member` exposes an **array** field as a single membership value), an optional `enum` (constrain the param to a fixed set so the agent can only pick a valid value), an optional `pattern` (a regex declaring the valid format of a structured string field — e.g. an id or code shape like `^pat-[0-9]+$` — so a placeholder such as `"null"`/`"undefined"` or any wrong format is rejected with an actionable error instead of silently matching nothing; applies to an exact/any_of/member match on a plain string field, and cannot be combined with `enum`), and `description`. An array field auto-exposes as a `member` param; object fields are rejected — expose a nested path (`address.city`) instead. The generic `filter` parameter stays on the tool as the escape hatch for anything the typed params cannot express (OR / nesting); typed params and `filter` are combined with AND.

## Hiding machinery

Read-shape knobs on a **`list`** Action. When the typed params cover everything an agent needs, set `"expose_filter": false` on the Action to drop the generic `filter` parameter entirely — keeping the agent-facing tool schema small. Server-side translation of the typed params is unaffected.

```bash
rekor actions upsert search_invoices --base my-ws --record_type invoices --operation list --config '{
  "expose_filter": false,
  "default_limit": 50,
  "filterable_fields": [ { "field": "status" } ]
}'
```

The list tool's machinery params (`sort`, `limit`, `offset`, `fields`) can each be hidden the same way — `"expose_sort"`/`"expose_limit"`/`"expose_offset"`/`"expose_fields": false` — and given a server-side default (`"default_sort"`/`"default_limit"`/`"default_fields"`) that still applies when the param is hidden. `"default_fields"` takes the same shape as the `fields` param — a comma-separated string (`"external_id,data.name"`) or an array (`["external_id", "data.name"]`). A hidden `limit` surfaces a `"truncated"` flag in the response if it caps the result, so rows are never silently dropped. `"agent_minimal": true` is a preset that hides all of them (plus `filter`) with a generous default limit, leaving an inputSchema that is pure typed semantics; any explicit `expose_*`/`default_*` on the same Action overrides the preset.

## Curated write surface (`writable_fields`)

A write-shape knob on a **`create`**/**`update`** Action — the write-side mirror of `filterable_fields`. List exactly the fields the tool may set — an allowlist that does two things at once:

- **Least-privilege / intent-scoped tools.** A field the agent sends that is not on the list is rejected with an actionable error. So a front-desk tool can be barred from ever setting a price or internal field, and you can split one operation into intent tools — a `confirm` tool allowed to set only `status` and a `reschedule` tool allowed to set only the time — instead of relying on prose to fence them.
- **A rich, typed write schema.** The tool's `data` parameter is generated from the record_type schema for just those fields — their types, enums, formats, `required`, and descriptions — instead of a generic "any object" slot. The agent gets the same typed, described, validated guidance on writes that `filterable_fields` gives on reads.

```bash
rekor actions upsert reschedule_appointment --base my-ws --record_type appointments --operation update --config '{
  "writable_fields": [
    { "field": "start_time", "param": "when", "description": "New start time (ISO 8601)" }
  ]
}'
```

Per field you may set `param` (rename the key the agent sets in `data` — the tool maps it back to the real field) and `description` (override the field's description). Fields are **top-level only** (writes merge at the top level — a partial update changes just the fields you send and leaves the rest untouched; nested objects are replaced whole). The record_type schema stays the validation source of truth — `writable_fields` shapes *which* fields the tool exposes and *how they are described*, it does not re-validate values. Omit `writable_fields` to keep the generic any-field `data` slot.

## Conditional writes (`precondition`)

A write-shape knob on a **`create`**/**`update`** Action — a Filter DSL expression checked against the record's **current** state before the write applies. If it does not hold, the write is rejected with a 409 conflict and nothing changes; if it holds, the write proceeds. This turns a fragile read-then-write into one correct, race-free call: model a bookable slot as a record and make "book it" a guarded update, so two agents cannot both book the same slot.

```bash
rekor actions upsert book_slot --base my-ws --record_type slots --operation update --config '{
  "precondition": { "field": "data.status", "op": "eq", "value": "free" }
}'
```

The guard is **invisible to the agent** — it is part of the Action config, never a tool parameter — so the agent just calls `book_slot(...)` and the booking integrity is enforced for it. Paths address the current record as `data.<field>`, `version`, or `id` (e.g. `{ "field": "version", "op": "is_null" }` = create-only-if-absent). One precondition per Action; the `search` operator is not allowed. Idempotent writes (retries that should not double-create) are already handled by upsert-by-`external_id` — the precondition is for cross-state guards, not retry safety.

## Guarded + unguarded writes on one record_type

A `precondition` is one-per-Action, so to expose both a guarded write and a plain write on the same record_type, author **two Actions** for that record_type with the same operation but distinct ids, then reference both:

```bash
rekor actions upsert book_slot   --base my-ws --record_type slots --operation update --config '{
  "precondition": { "field": "data.status", "op": "eq", "value": "free" }
}'
rekor actions upsert manage_slot --base my-ws --record_type slots --operation update
```

```json
{
  "name": "Booking Agent",
  "actions": [
    { "action": "book_slot" },
    { "action": "manage_slot" }
  ]
}
```

`book_slot` succeeds only on a slot that is still `free`; `manage_slot` updates the same slot unconditionally (e.g. an operator correction). Because each Action has its own id, both surface as distinct tools with no name collision.

## Selecting a named write binding (`binding`)

A write-shape knob on a **`create`**/**`update`**/**`delete`** Action. When a proxy-backed record_type's source declares **named write bindings** (a `create`/`update`/`delete` op given as a map of named endpoint variants — see `references/external-sources.md`), a write Action picks which one it dispatches to with a `binding`:

```bash
rekor actions upsert book_trial  --base my-ws --record_type bookings --operation create --config '{ "binding": "trial" }'
rekor actions upsert book_makeup --base my-ws --record_type bookings --operation create --config '{ "binding": "makeup" }'
```

This is what keeps one canonical record_type (`bookings`) serving a backend whose writes fan out to several endpoints: author two `create` Actions with distinct ids (`book_trial`, `book_makeup`) selecting different bindings (`trial`, `makeup`) of the same source, then reference both from the toolset. The agent never sees a `binding` parameter — it is injected server-side, exactly like `precondition`. Required when the bound op is a binding map with no `default`; rejected when the op is a single endpoint or names a binding the source does not declare (validated at config-write and re-checked at promotion).

## Composite Actions (atomic multi-entity writes)

An Action can instead be a **composite** — an ordered list of write `steps[]` executed **atomically** as one MCP tool. All steps commit together or none do, so an agent gets a single semantic call (e.g. `place_order`) instead of a fragile create-then-link-then-update sequence it could leave half-applied.

- Each step is one native write: a record `create`/`update`/`delete`, or a relationship `create`/`delete`. Give each a unique `key` (it namespaces that step's inputs on the generated tool). A composite has `steps` **instead of** the top-level `record_type`/`operation`.
- **Per-step guards, AND-combined.** A step may carry a `precondition` (the same compare-and-set Filter DSL as a single write). If *any* step's precondition fails, the whole action is rejected (409) and nothing is written — so you can gate a multi-record write on the current state of several records at once.
- Steps must be **native** record_types (a proxy/external-backed record_type can't join the atomic write — rejected when you save the action).
- Author it with `--config` (the flags cover single-op only):

```
rekor actions upsert place_order --base my-ws --config '{
  "description": "Create an order and its first line atomically",
  "steps": [
    { "key": "order", "record_type": "orders",      "operation": "create", "writable_fields": [{ "field": "status" }, { "field": "total" }] },
    { "key": "line",  "record_type": "order_lines", "operation": "create" }
  ]
}'
```

- Reference it from a toolset like any Action (`--action place_order`, or an `actions: [{ "action": "place_order" }]` entry). The generated tool takes one input object per step key — record steps take `data` (+ `external_id` for an idempotent upsert, or `id` for delete); relationship steps take `source_record_type`/`source_id`/`target_record_type`/`target_id` (+ `id` for delete). The agent never sees the preconditions — they're injected server-side.
- Cross-step references (linking to a record the *same* action just created by its server-assigned id) aren't supported yet — address each step with values you already hold (e.g. an `external_id` you set on the create step and reuse on a link step).

## Lenient list-tool arguments

The generated `list` tools are lenient about how structured arguments arrive: `filter` is always a JSON-encoded Filter DSL string, while `sort` and any multi-value (`any_of`) parameter accept **either** the native array **or** a JSON-encoded string of it — so an agent that serializes array arguments as strings still works. (`sort` is the same JSON array of `{"field","direction"}` terms described in the Records section of SKILL.md.)

## Which base serves the toolset

`mcp.rekor.pro/t/{slug}/mcp` resolves the toolset from the base your **token** is scoped to. So:

- **Production:** promote the toolset (and the Actions it references), then connect with a token scoped to the production base (`my-ws`).
- **Preview (sandbox testing):** connect with a token scoped to the **preview base id** (`my-ws--<preview-slug>`) to serve the not-yet-promoted toolset.

If the slug cannot be resolved for your token's base (unknown toolset, or one that only exists in a preview you are not scoped to), `initialize` returns a clear JSON-RPC error telling you to scope to the preview base id or promote — it will not silently hand back a session with zero tools.

**Promotion blocking.** Promotion is blocked if it would break a published toolset — removing a record_type or relationship type it exposes, dropping an Action a reference points at, or removing a field its `filterable_fields`, `writable_fields`, or `precondition` depend on, or a named write binding an Action's `binding` selects — so promote the Actions and toolset together with the schema change (a dry run lists any such conflicts first).
