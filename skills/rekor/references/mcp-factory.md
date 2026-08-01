# MCP Factory — full toolset config reference

Deep grammar for MCP Factory toolsets. The SKILL.md **MCP Factory** section covers the concept, the basic CLI, and the named menu of advanced knobs; read this file when you actually configure a curated toolset.

## The two building blocks: Actions and Toolsets

A toolset is composed from first-class **Actions**. The model is two steps:

1. **Author Actions** with `rekor actions upsert`. An Action is *one record_type + one operation* (`create` / `get` / `list` / `update` / `delete`) with an immutable slug `id`. The Action **owns** the guard/shaping config: `filterable_fields`, `expose_*`, `default_*` for reads (`get`/`list`); `writable_fields`, `precondition`, `binding` for writes (`create`/`update`/`delete`). The Action `id` is the agent-facing tool name by default.
2. **Compose a Toolset** with `rekor toolsets upsert`. Its `actions` are **references** to Actions by id — each entry is `{ "action": "<action_id>", "action_name"?: "...", "description_override"?: "..." }`, nothing more. Relationship tools, batch, and `sql_query` are declared on the toolset itself.

A toolset `actions[]` entry only accepts `action`, `action_name`, and `description_override`. Any other key (a stray `record_type`, `operations`, `filterable_fields`, …) is rejected at config-write — the shaping config lives on the Action, not the reference.

## Contents

- [Full authoring example](#full-authoring-example)
- [Tool names (`action_name` / `description_override`)](#tool-names)
- [Typed filter params (`filterable_fields`)](#typed-filter-params-filterable_fields)
- [Machinery params — hidden by default (`expose_*` / `default_*`)](#machinery-params--hidden-by-default)
- [Curated write surface (`writable_fields`)](#curated-write-surface-writable_fields)
- [Baked-in read predicates (`base_filter`)](#baked-in-read-predicates-base_filter)
- [Conditional writes (`precondition`)](#conditional-writes-precondition)
- [Guarded + unguarded writes on one record_type](#guarded--unguarded-writes-on-one-record_type)
- [Selecting a named write binding (`binding`)](#selecting-a-named-write-binding-binding)
- [Tool-design warnings at push](#tool-design-warnings-at-push)
- [Lenient list-tool arguments](#lenient-list-tool-arguments)
- [Which base serves the toolset](#which-base-serves-the-toolset)

## Full authoring example

**Step 1 — author the Actions.** One record_type + one operation each; the `id` becomes the agent-facing tool name.

A `list` Action with no `filterable_fields` generates a tool with NO parameters at all
(machinery is hidden by default) — right for a small catalog the agent reads whole, but
if the tool is meant to filter, declare the fields:

```bash
rekor actions upsert search_invoices --base my-ws --record_type invoices --operation list \
  --description "Find a customer's invoices, optionally narrowed by status" \
  --config '{ "filterable_fields": [
    { "field": "customer_id", "match": "exact" },
    { "field": "status", "optional": true }
  ] }'
rekor actions upsert get_invoice   --base my-ws --record_type invoices --operation get
rekor actions upsert create_payment --base my-ws --record_type payments --operation create
rekor actions upsert get_payment    --base my-ws --record_type payments --operation get
rekor actions upsert list_payments  --base my-ws --record_type payments --operation list \
  --description "List all payments"
```

`search_invoices` is shaped the way [the push-time warnings](#tool-design-warnings-at-push) want: `customer_id` is the tool's question so it stays required, and it takes an exact match safely because the invoices schema declares it `x-fk` into `customers` — a made-up id is rejected by the foreign-key probe rather than silently matching nothing. `status` is a genuine narrowing filter, so it is marked `optional`. An exact-match string with neither a closed set nor a foreign key behind it is the shape to avoid.

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
    { "field": "customer", "match": "text" },
    { "field": "status", "optional": true },
    { "field": "issued_at" }
  ]
}'
```

Each field becomes a parameter shaped by its type:

- an `enum` or boolean field → an exact-match param (the agent can only pick a valid value)
- a number or date/date-time field → a range pair (`<field>_min`/`<field>_max`, or `<field>_after`/`<field>_before`)
- a string field → a `<field>_contains` substring param

**`param` is a stem, not always the final param name.** Some matches suffix it:

| match | generated param(s) |
|---|---|
| `exact`, `any_of`, `member` | `<param>` (verbatim) |
| `text` | `<param>_contains` |
| `range` | `<param>_min` / `<param>_max` (`_after` / `_before` for dates) |
| `search` | `<param>_search` |

So `{"field": "plan_name", "match": "text"}` exposes `plan_name_contains`, not `plan_name`.

### `match: search` — the ranked param

`match: search` exposes a **ranked similarity** param (`<field>_search`) compiling to the `search` operator. It is the **only** match that reaches `search`, and therefore the only way a field's `x-search` tuning (`mode`, `threshold`) affects an agent-facing tool — `match: text` is a plain `%contains%` (`ilike`): case-insensitive, but accent-**sensitive** and typo-intolerant.

- **Plain string fields only**; `enum`/`pattern` don't apply (the value is a fuzzy query, not a value to pick), and a field whose `x-search` sets `searchable: false` is rejected at config-write.
- **Tuning is top-level-only.** `x-search` is read from top-level schema properties, so a `search` param on a nested path (`address.city`) works but uses the default blend — a nested `x-search` is not applied (see `references/querying.md`).
- **Ordering:** when a search param is filled, results come back best-match-first by default; an explicit `sort` overrides that.
- **Native record_types only.** Like every non-`eq` match, `search` isn't forwardable to a proxy-backed source — such a call fails loudly rather than silently ignoring the filter.
- **Rank is not existence — pick a `threshold` deliberately.** Everything above the cutoff is returned ranked, so a value that *doesn't exist* can still return a confident-looking near-match of something else. This bites hardest with `mode: name` on datasets sharing a prefix token (e.g. hundreds of plans all starting `AMIL`), where scores compress high and stop discriminating. If your workload needs "no match" to be a reliable signal, raise `threshold` for that field, or keep a `match: exact`/`text` param alongside for confirmation.

Per field you may set `param` (rename the generated param **stem** — see the table above), `match` (`exact` | `range` | `text` | `any_of` | `member` | `search` — `any_of` accepts a list and matches any; `member` exposes an **array** field as a single membership value; `search` is the ranked match above), an optional `enum` (constrain the param to a fixed set so the agent can only pick a valid value), an optional `pattern` (a regex declaring the valid format of a structured string field — e.g. an id or code shape like `^pat-[0-9]+$` — so a placeholder such as `"null"`/`"undefined"` or any wrong format is rejected with an actionable error instead of silently matching nothing; applies to an exact/any_of/member match on a plain string field, and cannot be combined with `enum`), `optional` (below), and `description`. An array field auto-exposes as a `member` param; object fields are rejected — expose a nested path (`address.city`) instead. The generic `filter` escape hatch (for OR / nesting) is **not** on the tool unless you set `expose_filter: true`; when it is, typed params and `filter` are combined with AND.

### Required by default (`optional`)

A declared field **is** the tool's question, so its generated param is **required**. `"optional": true` opts one out:

```json
{ "filterable_fields": [
  { "field": "patient_id", "match": "exact" },
  { "field": "status", "match": "exact", "optional": true }
] }
```

Why this direction: an optional parameter is one a weak model may fill with an invented placeholder (`null`, `none`, `sem_filtro`, `.*`). That compiles to a real condition matching zero rows — which, without the recovery hint below, reads as "nothing found" rather than an error, so the model retries with another guess instead of correcting, and can burn its whole iteration budget. Requiring the parameters that define the tool's question makes that failure unreachable.

- A **range** field's two bounds (`_min`/`_max`, `_after`/`_before`) are always optional — `"optional": false` on a range is rejected at config-write.
- A required param must carry a real value: blank or whitespace-only input counts as missing. On a **multi-select** (`any_of`) param an empty selection counts too — both the native `[]` and its JSON-encoded `"[]"` spelling — since either compiles to no condition at all. On a scalar param `"[]"` is just a value.
- **Several optional filters on one tool is a design smell** — it usually means the tool answers more than one question. Split it into per-intent Actions (`find_patient_by_phone` + `search_patient_by_name`) rather than widening one.

## Machinery params — hidden by default

Read-shape knobs on a **`list`** Action. The five machinery params — `filter`, `sort`, `limit`, `offset`, `fields` — are **not** on a generated tool unless you ask for them, so its inputSchema is pure typed semantics. Server-side translation of the typed params is unaffected, and so is the underlying REST API; this is about what the *agent* sees.

Opt one back in with `"expose_<param>": true`, and set the server-side value applied while a param stays hidden with `"default_sort"` / `"default_limit"` / `"default_fields"`:

```bash
rekor actions upsert search_invoices --base my-ws --record_type invoices --operation list --config '{
  "default_limit": 50,
  "default_fields": "external_id,data.status",
  "filterable_fields": [ { "field": "status" } ]
}'
```

- `"default_fields"` takes the same shape as the `fields` param — a comma-separated string (`"external_id,data.name"`) or an array (`["external_id", "data.name"]`).
- A hidden `limit` defaults to a generous page and surfaces a `"truncated"` flag in the response if it caps the result, so rows are never silently dropped. Set `"default_limit"` when you know the right page size, and `"default_fields"` to keep large records from filling the agent's context.
- An **empty result** where the agent itself narrowed the search carries an `"empty_result_hint"` naming the parameters it supplied (names only, never values) with the recovery that actually works: verify the value for a required parameter, drop it for an optional one, simplify a raw `filter`. This is what breaks the guess → zero rows → guess-again retry loop — zero rows used to read as "nothing found" with no cause. An empty page the agent did NOT cause gets no hint: no filters supplied, only a `base_filter` it can't see, or a page past the last row (its filters already returned rows, so nothing about them is in question).
- Expose a param only when the agent genuinely needs to drive it — an agent that paginates or sorts on purpose. `"expose_filter": true` restores the raw Filter-DSL escape hatch for OR / nesting the typed params can't express.
- `"agent_minimal": true` named the old opt-in preset for this behavior. It is still accepted (existing config keeps working) but does nothing, since hiding is now the default — omit it in new Actions.

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

## Baked-in read predicates (`base_filter`)

A read-shape knob on a **`list`** Action. `base_filter` is a Filter-DSL expression AND-merged into every call of that tool, **server-side and invisible to the agent** — it never appears in the generated `inputSchema`.

```bash
rekor actions upsert list_active_practitioners --base my-ws --record_type practitioners --operation list --config '{
  "base_filter": { "field": "data.status", "op": "eq", "value": "active" },
  "filterable_fields": [ { "field": "specialty", "match": "exact", "enum": ["cardiology", "dermatology", "pediatrics"] } ]
}'
```

The agent sees one parameter (`specialty`) and cannot see, set, or widen the status predicate. This is the read-side twin of `precondition`: the alternative — an optional `status` param the model is supposed to know to fill — is exactly the shape that invites an invented value.

Use it to carve a narrow tool out of a broad record_type: `list_active_practitioners` and `list_open_tickets` are the same `list` Action shape with a different predicate baked in, and each reads to the model as a single-purpose tool.

- **Combines flat.** The baked predicate, the typed params, and (if exposed) the raw `filter` are AND-combined into ONE group rather than nested pairwise, so the merge costs exactly **one** nesting level no matter how many parts are present. Config-write reserves that level (a `base_filter` is validated in its merged shape, so one that only fits un-merged is rejected up front). The level is not free, though: an agent filter already at the maximum depth will now be one over, so a `base_filter` costs the raw escape hatch one level of headroom.
- **`list` only.** Rejected on a `get`, which fetches one record by key and has no filter to merge into.
- **Native record_types only.** A source forwards only the filters it declares, so an invisible predicate on a proxy-backed record_type would either be dropped upstream (silently widening the result the tool promised to narrow) or fail every call with an error naming a filter the agent never sent. Rejected at config-write, and re-checked at promote in case a promote ADDS a source.
- **No `search`.** It would route every call off the read-after-write path; an invisible predicate must not silently change where the tool reads from.
- **Paths** address record data (`data.status`, or a bare `status`) and must resolve against the record_type schema — checked at config-write and again at promote, since a dangling predicate would surface to the agent as "no results" with no way to recover. Bare paths are stored canonicalized, so `rekor pull` always writes `data.status`.
- **The field must be comparable by the operator.** A value comparison needs a field the schema shows to be scalar — a declared `string`/`number`/`integer`/`boolean`, or (with no declared type) an `enum`/`const` of scalar values, which is how a workflow status field is spelled. **Array** fields take `has` and nothing else (any other operator compares the whole array and matches inconsistently); conversely `has` requires an array. Everything else — objects, maps, `$ref`/`oneOf`-composed fields, a bare `{}` — is rejected with a message naming the fix: address a nested path like `data.address.city`, declare the field's type, or use `is_null`/`is_not_null`, which are presence checks and stay valid against any shape.

  The rule is stated as what's *allowed* rather than what's blocked, deliberately: JSON Schema has open-ended ways to express structure, so an unrecognized shape is refused at config-write with an actionable error rather than becoming a predicate that silently matches nothing.
- **A filterless Action with a `base_filter` no longer forwards the raw `filter` verbatim** — merging requires parsing it first. Practically this means a malformed `filter` is rejected by the translator rather than the engine; the error is the same class either way.

**It is not a security boundary.** Like `precondition`, it shapes the generated tool, not the record_type: a caller with raw REST access to the same base still reads unfiltered rows, and a toolset-bound token's *surface* check is at record_type + operation granularity (filter shape has never been an access boundary). Scope data access with token grants; use `base_filter` to make the agent's job unambiguous, not to keep rows secret.

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

- Reference it from a toolset like any Action (`--action place_order`, or an `actions: [{ "action": "place_order" }]` entry). The generated tool takes one input object per step key — record steps take `data` (+ `external_id` for an idempotent upsert, or `id` for delete); relationship steps take `source_record_type`/`target_record_type` plus, per endpoint, exactly one of the internal id (`source_id`/`target_id`) or the external key (`source_external_id`/`target_external_id`, with optional `source_external_source`/`target_external_source`), an optional relationship-level `external_id`/`external_source` (the link's own key — makes a re-run of the whole action idempotent for that step) — and `id` for delete. The agent never sees the preconditions — they're injected server-side.
- Cross-step references work through external ids: set an `external_id` on a create step and address the link step's endpoint with `source_external_id`/`target_external_id` — steps run in order inside one transaction, so the link resolves the record the same action just created. (Referencing a same-action record by its *server-assigned* id remains impossible — the id doesn't exist until the step runs.) External addressing requires the record to exist by the time the link step runs; a miss rejects the whole action.

## Tool-design warnings at push

`rekor push` lints the `list` Actions in your config against the principles above and prints any suggestions to stderr. They are **advisory** — a rule encodes a heuristic about how a model behaves, and a heuristic that failed your push would be worse than the shape it objects to. Config validity is separate and still fails loudly.

| Rule | What it catches |
|---|---|
| `all-filters-required` | Every **value** filter is required and there is more than one, so the agent must supply all of them on every call. Usually means narrowing filters need `"optional": true`, or the Action is several intents fused together. This is the shape a config authored *before* required-by-default lands in. A `range` field is ignored here too, so declaring a date window neither triggers the rule nor silences it. |
| `many-optional-filters` | More than two optional filter **params** on one tool — each is a parameter a weaker model may fill with an invented value. A `range` field's two bounds don't count: they are optional by construction, so an anchor plus a date window stays clean. |
| `unconstrained-string-filter` | A value-picking string param with no `enum` and no `pattern`, so a made-up value compiles to a real condition that matches nothing. **Top-level** `x-fk` fields are exempt: their valid values are whatever rows exist in the target record_type, and the foreign-key probe already rejects a bad one. A *nested* `x-fk` (`insurance.plan_id`) is still flagged — the write-time probe only reads one level deep, so a nested annotation is inert config with nothing enforcing it. |
| `filter-escape-hatch` | `expose_filter` enabled alongside declared filter fields — the agent can bypass them with a raw expression, which is the untyped surface the typed params exist to replace. Fires on any truthy value, since `expose_filter: yes` is a *string* in YAML and still opens the hatch. |

Warnings appear on `rekor push`, on `--dry-run`, and on a push with no changes — that last case matters, because a config written before these rules existed is exactly the one that never diffs again.

## Lenient list-tool arguments

The generated `list` tools are lenient about how structured arguments arrive: `filter` is always a JSON-encoded Filter DSL string, while `sort` and any multi-value (`any_of`) parameter accept **either** the native array **or** a JSON-encoded string of it — so an agent that serializes array arguments as strings still works. (`sort` is the same JSON array of `{"field","direction"}` terms described in the Records section of SKILL.md.)

Generated tool schemas are otherwise **closed**: an argument not declared by the tool's `inputSchema` is rejected with the unknown name, a likely correction when one exists, and the valid parameter list. This includes stale typed-filter names after a rename and machinery parameters that aren't exposed; they are never silently discarded into a broader call. A missing **required** filter param is rejected the same way, naming every missing parameter at once so one round trip carries the whole fix.

## Which base serves the toolset

`mcp.rekor.pro/t/{slug}/mcp` resolves the toolset from the base your **token** is scoped to. So:

- **Production:** promote the toolset (and the Actions it references), then connect with a token scoped to the production base (`my-ws`).
- **Preview (sandbox testing):** connect with a token scoped to the **preview base id** (`my-ws--<preview-slug>`) to serve the not-yet-promoted toolset.

If the slug cannot be resolved for your token's base (unknown toolset, or one that only exists in a preview you are not scoped to), `initialize` returns a clear JSON-RPC error telling you to scope to the preview base id or promote — it will not silently hand back a session with zero tools.

**Promotion blocking.** Promotion is blocked if it would break a published toolset — removing a record_type or relationship type it exposes, dropping an Action a reference points at, or removing a field its `filterable_fields`, `writable_fields`, `base_filter`, or `precondition` depend on, or a named write binding an Action's `binding` selects — so promote the Actions and toolset together with the schema change (a dry run lists any such conflicts first).
