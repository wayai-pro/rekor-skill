# External Sources — full config reference

Deep grammar for the external sources that back a record_type with an external API. The SKILL.md **External Sources** section covers the concept and the menu of what a source can declare; read this file when you actually configure one.

## Contents

- [What a source defines](#what-a-source-defines)
- [`field_mapping` rules](#field_mapping-rules)
  - [Split one field into several upstream params (`targets`)](#split-one-field-into-several-upstream-params)
  - [Destructure a composite field (`separator` + `parts`)](#destructure-a-composite-field)
  - [Compose / computed fields (read side)](#compose--computed-fields-read-side)
  - [Per-operation override](#per-operation-override)
- [Per-operation endpoints](#per-operation-endpoints)
- [Named write bindings](#named-write-bindings)
- [`id_path` (and its variants)](#id_path)
- [`forward_filters`](#forward_filters)
- [Static constants, `static_body`, `body_template`](#static-constants-static_body-body_template)
- [`{{current.*}}` / `{{prior.*}}`](#current--prior)
- [`injections`, `executor_secrets`, caching, signing, timeout, breaker](#other-source-options)
- [URL / SSRF rules](#url--ssrf-rules)
- [Worked example — a non-REST / legacy upstream](#worked-example)

## What a source defines

Configure sources as part of the record_type schema (preview only, promoted like any schema change):

```bash
rekor record-types upsert invoices --base my-ws--preview --name "Invoices" \
  --schema @schema.json --sources @sources.json
```

Each source defines:
- `name` — must equal the record's `external_source`.
- `auth` — header + value template; the secret can be inline or a `vault:<name>` reference. An **inline** secret is environment-local — promotion never copies it to production, so set it on the production base directly (a newly promoted source stays unconfigured and requests fail with a clear "no usable credential configured" error until you do). A `vault:<name>` reference travels by reference and needs no extra step.
- `field_mapping` — optional; omit for identity passthrough. See [below](#field_mapping-rules).
- `get` / `list` / `create` / `update` / `delete` — per-operation endpoints. See [below](#per-operation-endpoints).
- `id_path` — how to find the `external_id` in each upstream record. See [below](#id_path).
- `forward_filters` — forward the agent's filter conditions to the upstream. See [below](#forward_filters).
- `static_body`, `body_template`, `request_encoding` — request-body shaping. See [below](#static-constants-static_body-body_template).
- `injections`, `executor_secrets`, `cache_ttl` + `stale_if_error`, `signing`, `timeout_ms`, `breaker` — see [below](#other-source-options).

## `field_mapping` rules

**Optional**; omit it for identity passthrough (the upstream object is stored as the record data verbatim, and writes send your data unchanged). When set, `to_external` / `to_rekor` map between your schema and the upstream's field names. A value is either a simple rename (`"status": "invoice_status"`) or a rich rule `{ path, values, default, transform, date_format, array_mode, target }`:

- `transform` — `lowercase` / `uppercase` / `trim` / `to_string` / `to_number` / `to_boolean` / `iso_date`.
- `values` — an enum map. **Always Rekor-keyed** (`{ "<your value>": "<upstream value>" }`, e.g. `{ "active": "ACTIVE" }`) — the *same* map serves both directions: forward (write) looks up by key, reverse (read) matches by value and returns the key. **The orientation does not flip in an explicit `to_rekor` rule** even though everything else in that rule reads upstream→Rekor (the rule key is the upstream field, `path` is your field): a `values` map stays Rekor-keyed and is reverse-matched, e.g. `to_rekor: { activity: { path: "activity_id", values: { "natacao-adulto": "8" } } }` maps an upstream `8` (or `"8"`) → `"natacao-adulto"`. As a convenience the reverse path *also* accepts the intuitive upstream-keyed form (`{ "8": "natacao-adulto" }`) when the Rekor-keyed match misses — but write the **Rekor-keyed** form to stay unambiguous (reverse-match wins on any key/value overlap). A fully backwards or mistyped map otherwise passes the raw upstream value through unchanged rather than erroring, so confirm by reading a record back after configuring.
- `default` — value used when the field is missing.
- `array_mode` — `first` / `last` / `flatten` for array-valued paths.
- `date_format` (+ optional `rekor_format`, default ISO) — bidirectional date/time reshaping between the external wire format and Rekor's canonical. Tokens `yyyy MM dd HH mm ss` + separators, e.g. `date_format: "dd/MM/yyyy"` (⇄ ISO date) or `date_format: "HH:mm:ss"` with `rekor_format: "HH:mm"` (truncate seconds).
- `target` (`to_external`, write side; default `body`) — route a mapped param to the request **query string** (`target: "query"`) instead of the body on `create`/`update`. The value map / transform / default still apply — the write-direction counterpart of `forward_filters.target`. Use it for an upstream whose create/update params are **query** (not a JSON body) and need a value translated to an upstream code, e.g. `"activity_id": { "path": "idActivity", "values": { "lash-lift": "42" }, "target": "query" }` sends `?idActivity=42`. A query-target name that collides with a param already in the endpoint's `url` is rejected at config time; each split `targets` entry can pick its own `target`.
- Rich rules don't auto-invert — supply an explicit `to_rekor` entry (or a `computed` field) for the reverse direction. In that explicit `to_rekor` entry the rule key is the upstream field and `path` is your field, but a `values` map stays **Rekor-keyed** (see `values` above) — it's the one part that doesn't follow the rest of the rule's upstream→Rekor direction.

### Split one field into several upstream params

(`to_external` only) — give the rule `targets` (an array of rules) instead of a single `path` to fan one canonical field out to multiple upstream params, each with its own `path`/`transform`/`date_format`/`target`. The headline use: an upstream that wants **separate `date` + `time` params** fed from a single ISO datetime — `"starts_at": { "targets": [{ "path": "data", "date_format": "dd/MM/yyyy" }, { "path": "hora", "date_format": "HH:mm" }] }`. A split field can't be a `forward_filters` field (no single upstream name).

### Destructure a composite field

(`to_external` only) — give the rule a `separator` plus a `parts` array (instead of a single `path` or a split `targets`) to split ONE composite canonical value on the separator and route each part to its own upstream param, each independently value-mapped / `date_format`'d / `target`'d. This is the **write-side inverse of a composite `id_path`** (and of `computed` compose): when a write tool references an entity by its composite id — e.g. `session_id = "<activity>::<date>"` — but the upstream action endpoint needs the decomposed parts as separate, independently-translated params: `"session_id": { "separator": "::", "parts": [{ "path": "idActivity", "values": { "yoga": "A1" }, "target": "query" }, { "path": "activityDate", "date_format": "dd/MM/yyyy", "target": "query" }] }` sends `?idActivity=A1&activityDate=...`. Parts are **positional by default** (segment `i` → `parts[i]`, so declare them in the same order the read-side composite id concatenates); a part with no matching segment emits its `default` or is omitted, and extra trailing segments are dropped. **Selective extraction**: give each part an explicit `segment` index to consume a chosen — possibly **non-contiguous** — segment, declaring only the parts you need. So ONE canonical id carrying a superset of identifiers (`activityCode::configId::date::time`) can serve N write actions that each need a different subset — e.g. a trial-booking endpoint keyed by `(activityCode, date)` = `[{ segment: 0, path: "idActivity" }, { segment: 2, path: "activityDate", date_format: "yyyy-MM-dd" }]`, and a makeup-booking endpoint keyed by `(configId, date)` = segments 1+2 — one resource record_type, no ignored params, no per-action duplication. Per rule it's all-or-nothing: declare `segment` on **every** part or none (no mixing); declared indices are unique. Unlike a split (which routes the *same* value to every target), a destructure routes a *different* part to each. This keeps the agent-facing tool integration-agnostic: it references the entity by the same canonical composite id whether the backend is native or a proxied legacy endpoint. A destructure field can't be a `forward_filters` field (no single upstream name).

### Compose / computed fields (read side)

`computed` builds a Rekor `data.*` field from a `{{token}}` template over the **raw upstream** record (the inverse of a split, and the same template grammar as a composite `id_path`). Use it to rebuild one field from several upstream fields — `"computed": { "starts_at": { "template": "{{data}}T{{hora}}", "sources": { "data": { "date_format": "dd/MM/yyyy", "rekor_format": "yyyy-MM-dd" }, "hora": { "date_format": "HH:mm", "rekor_format": "HH:mm:ss" } } } }` rebuilds an ISO datetime from upstream `data`+`hora` — or to derive a key from siblings (`"slot_id": { "template": "{{professional_id}}::{{data}}::{{hora}}" }`). `sources` reshapes each token (same vocabulary as a `to_rekor` rule); if any token is absent the whole field is omitted. Tokens reference the upstream's own field names (no chaining over other computed fields). A token may also read a **forwarded list filter** via the reserved `{{filter.<field>}}` namespace — see `forward_filters` below — to backfill a scope key/date the upstream omits from its list rows (e.g. `"starts_at": { "template": "{{filter.date}}T{{hora_atendimento}}" }` when a list-by-date upstream returns only the time). `{{filter.*}}` populates on `list` only and a composite `id_path` may use it too; each referenced field must be in `list.forward_filters.fields`.

### Per-operation override

The source-level `field_mapping` applies to every op, but an RPC/legacy upstream may use *different* param names for the same field across verbs (create takes `entity_id`, update takes `id_entity`). Put a `field_mapping` on an individual `get`/`list`/`create`/`update` endpoint to override the source-level one for that op (full replacement — set every field it needs); it falls back to the source-level mapping when unset, e.g. `"update": { …, "field_mapping": { "to_external": { "owner": "id_entity" } } }`. Not allowed on `delete` (no body — set its param names in the URL template).

## Per-operation endpoints

`get` / `list` / `create` / `update` / `delete` — per-operation endpoint templates. Each has its own `url` (with `{{external_id}}`, `{{query.*}}`, `{{data.*}}`, `{{current.*}}`/`{{prior.*}}` on update/delete — see below, `{{auth.org_id}}`, `{{auth.base_id}}` tokens) and `method` (any of GET/POST/PUT/PATCH/DELETE) — so **RPC-style verb paths and all-POST upstreams work directly**, e.g. `list: { url: ".../listar_x", method: "POST" }`, `create: { url: ".../incluir_x", method: "POST" }`. `response_path` / `total_path` extract records/count from an envelope (e.g. `response_path: "dados"` pulls the array out of `{ ..., dados: [...] }`).

- `success_path` / `message_path` — for upstreams that return HTTP 200 even on logical failure (`{ "success": false, "message": "..." }`). When `success_path` resolves falsy, the call surfaces as an error carrying the `message_path` text instead of being mistaken for data. (Dot-paths, like `response_path` — not JSONPath.)

## Named write bindings

(`create` / `update` / `delete` only) — when one canonical entity's writes fan out, in the backend, to **several** endpoints with different URLs/params/id-semantics (a "book trial" vs a "book makeup" that are both *creates* of the same booking), declare the write op as a **map of named variants** instead of a single endpoint: `"create": { "trial": { "url": ".../experimental-class", "method": "POST", "id_path": "idActivitySession" }, "makeup": { "url": ".../enroll", "method": "POST", "id_path": "{{data.member_id}}::{{data.session_id}}" } }`. Each binding is a full endpoint (its own `url`/`method`/`field_mapping`/`id_path`/`body_template`/`response_path`/`headers`) and **shares the source's transport** (`auth`/`injections`/`signing`/`breaker`/`cache`/`request_encoding`/`static_body`). The caller selects one **by name** per write — at the MCP Factory layer via a tool's `bindings` (see `references/mcp-factory.md`), so the canonical record_type stays ONE record_type instead of shattering into one-per-endpoint. A `default` binding is used when no name is selected (e.g. an `external_write` trigger's dispatch, which selects none — so a source used as an `external_write` target must give the targeted op a single endpoint or a `default` binding). Reads (`get`/`list`) don't fan out — they stay a single endpoint. The single-endpoint form is the default; selecting a binding name that the op doesn't declare (or naming one on a single-endpoint op) is rejected at config-write and again at promotion.

## `id_path`

Dot-path to the `external_id` within each **raw** upstream record, for upstreams keyed on a domain field rather than `id`/`_id` (e.g. `id_path: "codigo"`, `"identifiers.cpf"`). Applies to LIST (per row) and CREATE (from the response). When unset, the id is taken from the first present conventional key (`id`/`_id`/`Id`/`ID`/`uuid`/`key`). **Set this when your upstream has no conventional id field — otherwise a LIST whose rows all lack a resolvable `external_id` fails loud with a `422 EXTERNAL_CONFIG` naming the fix and the row keys it saw (a partial drop returns the resolvable rows plus a `meta.dropped_no_id` count), and CREATE fails to identify the new record.**

- **Composite (list-only).** For an upstream whose identity is composed of several fields with no single natural id, `id_path` may instead be a **template** joining fields, e.g. `id_path: "{{entity}}::{{date}}::{{time}}"`; a composite id is **list-only** (rejected at config-write when `get`/`create`/`update`/`delete` is also configured, since it can't address a single row by id).
- **Create-only override.** For an RPC-style upstream that returns the new record's server-assigned id on **CREATE** under a *different* field than the one keying LIST rows, set a **create-only** `id_path` on the `create` endpoint (e.g. `"create": { …, "id_path": "created_ref" }`) — it overrides the source-level `id_path` for the create read-back and falls back to it when unset.
- **Request-templated create id.** For an **action/RPC write endpoint that returns no addressable entity** (an empty body, or a bare envelope like `{ "result": "ok" }`), make the create-only `id_path` a **request template** over the create inputs instead — `"create": { …, "id_path": "{{data.member_id}}::{{data.enrollment_id}}::{{data.date}}" }` — and Rekor derives the `external_id` from the request data. Identical inputs map to the same id, so a repeated identical action upserts the same record rather than duplicating. Its tokens must be `{{data.<field>}}` naming declared record_type fields, and the source must be create-only (a list is fine) — a request-templated create id can't address a single row, so `get`/`update`/`delete` are rejected at config-write.

## `forward_filters`

(on the `list` endpoint) — opt in to forward the agent's filter conditions to the upstream so a proxied `list`/`query` can pass a search key, a date, a parent id, etc. `fields` is the allowlist of Rekor field names that may be forwarded (each translated to the upstream's name/value via `field_mapping`); `target` (`query` for a GET, `body` for a POST/PUT/PATCH list — inferred from the method when omitted) chooses where they go. Without it, a proxied query still **rejects** any `--filter`/MCP `filter`. Currently forwards exact-match (`eq`) conditions only — a single `eq` or a flat AND of `eq`s; any other operator or shape is rejected with an actionable error rather than silently returning unfiltered data, e.g. `"list": { "url": ".../search", "method": "GET", "forward_filters": { "fields": ["email"] } }`. A forwarded value is also exposed to the read mapping as `{{filter.<field>}}` (a `computed` field or a composite `id_path`), so a row can carry the filter value the upstream leaves out of its list rows — e.g. forward `date`, then compose `starts_at` from `{{filter.date}}` + the row's time (see Compose / computed fields).

## Static constants, `static_body`, `body_template`

- **Static constants** need no special field: bake constant **query** params into the `url` (`".../listar?clinica=35"`), and put constant non-secret **headers** in `endpoint.headers` (`{ "X-Tenant": "35" }`). For a constant **body** param use `static_body`; for a secret use `injections`.
- `request_encoding` — `json` (default) or `form` to send `application/x-www-form-urlencoded` bodies for legacy form-post upstreams.
- `static_body` — constant, non-secret params merged into every body-bearing request (incl. POST-based reads), e.g. a fixed tenant id the agent never supplies. Agent data and `injections` win on a key collision.
- `body_template` (per `create`/`update`/`delete` endpoint) — map of body field → templated string, interpolated against the same per-op tokens as that endpoint's `url` (`{{external_id}}`, `{{data.*}}`, `{{query.*}}`, `{{auth.org_id}}`/`{{auth.base_id}}`) and merged into the request body. Use it to put the record id — or any addressable value — **into** the body for RPC/legacy upstreams that key the update/cancel verb by id in the body rather than the URL, e.g. `"update": { …, "body_template": { "agendamento_id": "{{external_id}}" } }`. Body-bearing ops only — create/update, and a `delete` whose method carries a body (POST/PUT/PATCH); rejected on get/list and on any GET/DELETE-method endpoint (no body to carry it). Unlike `static_body` (constants, never templated) its values are filled per request; on a key collision agent data is overlaid by `body_template`, and `injections` win over both. Values are filled as **strings** (the same as `url`/header tokens) — for a numeric/boolean field an upstream is strict about, send it through `field_mapping` or `static_body` instead.

## `current` / `prior`

`{{current.*}}` / `{{prior.*}}` (in an `update`/`delete` `url`, headers, or `body_template`) — the record's **current** stored field values from the upstream, for upstreams that need the existing state on a write, not just the id + new values: optimistic concurrency (`?ifmatch={{prior.version}}`), "move"-style updates that take the OLD value plus the NEW (a reschedule taking the current `data_agendada` alongside `nova_data`), or a cancel keyed by the current field values. Rekor sources them with a pre-write read through the source's **`get`** endpoint, so a `get` is **required** when you reference one (rejected at config-write otherwise) — and the values are the **raw upstream record** (the upstream's own field names + wire format, echoed back verbatim). `prior` is an alias of `current` (a proxied write has no local merge). Update/delete only; referencing them costs one extra read per write (skipped entirely when unused). e.g. `"update": { …, "body_template": { "agendamento_id": "{{external_id}}", "data_agendada": "{{current.data_agendada}}", "nova_data": "{{data.data_agendada}}" } }`.

## Other source options

- `injections` — extra per-request vault secrets placed into a header or body field (for upstreams needing their own credential).
- `executor_secrets` — named vault credentials an executor pulls at dispatch, for a binary or large per-tenant credential it can't take inline (e.g. an mTLS client certificate). Each `{name, secret_ref}` declares a `vault:<name>` reference (templating only `{{auth.org_id}}`/`{{auth.base_id}}`); each pull is short-lived and single-use, scoped to the calling base. See `references/executors.md`.
- `cache_ttl` (seconds) + `stale_if_error` — read-through caching, optionally serving the last-known value on a transient upstream failure.
- `signing` — opt into HMAC request signing when the source points at an executor (adds `X-Rekor-Signature` + `X-Rekor-Timestamp`); omit for third-party APIs.
- `timeout_ms` — per-request upstream timeout (default 10s, max 30s); a timeout surfaces as a transient error so `stale_if_error` can serve a cached value.
- `breaker` — per-source circuit breaker: `{failure_threshold, cooldown_ms}`. After `failure_threshold` consecutive transient failures (default 5) the source opens and short-circuits calls for `cooldown_ms` (default 30s) before a single half-open probe. Always on; this only tunes it.

## URL / SSRF rules

URLs must be absolute `https` (or `http` only when the source sets `allow_insecure_http: true`, for a legacy/on-prem upstream without TLS) and tokens may appear only in the path/query (never the host) — an SSRF guard rejects otherwise.

## Worked example

A non-REST / legacy upstream (all-POST verb paths, a form body, a `{ success, dados }` envelope, a constant tenant id, and `dd/mm/yyyy` dates) configured as a **direct** source — no executor:

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
