# Querying & search reference

Deep how-to for reading records: the SQL example gallery, `search`-field tuning (`x-search`), and datetime configuration. The SKILL.md **Records** and **SQL Query** sections cover the concepts, the Filter DSL operators, and the mandatory rules; read this file for worked examples and tuning knobs.

## Contents

- [SQL query examples](#sql-query-examples)
- [Fuzzy / approximate `search` tuning (`x-search`)](#search-tuning)
- [Datetime configuration](#datetime-configuration)

## SQL query examples

All queries must include BOTH `org_id = {org_id:String}` AND `base_id = {base_id:String}`, plus `deleted = false` (and `archived = false` for active-only). Access JSON fields with `data.field.:Type` subcolumn syntax, or `CAST(data.field, 'Type')` for type-safe conversion.

Note the contrast with Filter-DSL list queries (`rekor records list` / `rekor relationships list` / the query tools): those return **active records by default**, and an explicit filter condition on `archived` overrides the default (your predicate decides which tier you see). Raw SQL applies no such default — you state the tiers yourself, and it's also the only way to read archived rows through record traversal (`related`), which has no filter override.

```bash
# Simple query
rekor sql "SELECT data.invoice_number.:String as num, data.status.:String as status FROM records WHERE org_id = {org_id:String} AND base_id = {base_id:String} AND record_type = 'invoices' AND deleted = false" --base my-ws

# Aggregation
rekor sql "SELECT data.status.:String as status, count() as cnt FROM records WHERE org_id = {org_id:String} AND base_id = {base_id:String} AND record_type = 'invoices' AND deleted = false GROUP BY status" --base my-ws

# Array aggregation (sum embedded line items)
rekor sql "SELECT data.invoice_number.:String as num, arraySum(CAST(data.line_items[].amount, 'Array(Float64)')) as total FROM records WHERE org_id = {org_id:String} AND base_id = {base_id:String} AND record_type = 'invoices' AND deleted = false" --base my-ws

# Explode array elements with ARRAY JOIN
rekor sql "SELECT item.description.:String as item, sum(CAST(item.amount, 'Float64')) as revenue FROM records ARRAY JOIN data.line_items[] as item WHERE org_id = {org_id:String} AND base_id = {base_id:String} AND record_type = 'invoices' AND deleted = false GROUP BY item ORDER BY revenue DESC" --base my-ws

# CTE joining records with relationships
rekor sql "WITH inv AS (SELECT id, data.invoice_number.:String as num, arraySum(CAST(data.line_items[].amount, 'Array(Float64)')) as total FROM records WHERE org_id = {org_id:String} AND base_id = {base_id:String} AND record_type = 'invoices' AND deleted = false), pay AS (SELECT target_id, sum(CAST(data.allocated, 'Float64')) as paid FROM relationships WHERE org_id = {org_id:String} AND base_id = {base_id:String} AND rel_type = 'payment_for' AND deleted = false GROUP BY target_id) SELECT inv.num, inv.total, coalesce(pay.paid, 0) as paid, inv.total - coalesce(pay.paid, 0) as balance FROM inv LEFT JOIN pay ON pay.target_id = inv.id ORDER BY balance DESC" --base my-ws

# With parameters
rekor sql "SELECT * FROM records WHERE org_id = {org_id:String} AND base_id = {base_id:String} AND data.status.:String = {status:String} AND deleted = false" --base my-ws --param status=issued

# Fuzzy / approximate text match, ranked by closeness (power-user form of `records query --filter {op:search}`).
# Fold case + accents on BOTH sides so "São" matches "sao"; jaroWinklerSimilarity suits short names.
rekor sql "SELECT *, jaroWinklerSimilarity(lowerUTF8(data.car_model.:String), lowerUTF8({q:String})) AS score FROM records WHERE org_id = {org_id:String} AND base_id = {base_id:String} AND record_type = 'vehicles' AND deleted = false AND score >= 0.85 ORDER BY score DESC LIMIT 10" --base my-ws --param q='honda civic'
```

## Search tuning

The Filter DSL `search` operator matches a field by **approximate** value — use it when you have a near-correct string but not the exact stored one (e.g. a model name, a place, a plan name, a SKU with a possible typo). Results come back **ranked by closeness**, each carrying a `_search_score` (0–1, higher = closer):

```jsonc
// "find vehicles whose model is close to 'honda civic', only active ones"
{ "and": [
  { "field": "data.status",    "op": "eq",     "value": "active" },
  { "field": "data.car_model", "op": "search", "value": "honda civic" }
] }
```

Every field is searchable by default — `search` needs no setup. Exact filters and `search` compose in one query: put exact conditions alongside the `search` leg so the exact ones narrow the set and `search` ranks within it. To **tune** how a field is matched, set an optional `x-search` hint on that field in the record_type schema:

```jsonc
"car_model": { "type": "string", "x-search": { "mode": "name" } }
```

- `mode: "name"` — short proper nouns (people, brands, plan/model names); favors close, prefix-aligned matches.
- `mode: "text"` — free-text / multi-word fields; matches on shared word fragments.
- `mode: "fuzzy"` — codes / SKUs / IDs where typos dominate.
- `threshold` (0–1) — minimum closeness to return; `searchable: false` — forbid searching a field (e.g. a large free-text blob you never want scanned).

Tuning is optional — unset fields use a sensible default. `x-search` hints apply to **top-level fields**; `search` still works on nested paths (e.g. `data.address.city`) using the default behavior, they just aren't individually tuned. Search runs against the latest synced data, so a record written a moment earlier may take a brief moment to appear.

**Pick `threshold` deliberately.** Everything above the cutoff is returned, ranked — so a value that doesn't exist can still come back as a confident near-match of something else. `mode: "name"` compresses scores high when many values share a prefix token (hundreds of plans all starting `AMIL`), which is exactly when a default cutoff under-discriminates. If "no match" needs to be a trustworthy signal in your workload, raise `threshold` on that field.

**Exposing search to an agent:** these hints only affect a toolset's generated tool if the `list` Action declares `match: search` on that field in `filterable_fields` — the only match that compiles to the `search` operator (`references/mcp-factory.md`). A `searchable: false` field is rejected there at config-write.

Omit `--sort` when using `search` to keep the relevance ranking.

## Datetime configuration

Declare datetime fields in the record_type schema with `"format": "date-time"` (or `"format": "date"` for date-only). Rekor stores them in a single canonical form: a naive value (no offset) gets the record_type's timezone attached, so `2026-06-11T13:00:00` is stored and returned as `2026-06-11T13:00:00-03:00` — the same shape from both `query` and a single-record read, with no UTC math required of you.

Set the timezone with `"x-timezone": "America/Sao_Paulo"` on the record_type schema, or set a base-wide default that every record_type without its own `x-timezone` inherits — the resolution order is record_type `x-timezone` → base `settings.timezone` → UTC. Set the base default from the CLI with `--timezone` (at create or update time), or via REST `PUT /v1/bases/<id>` with a `settings.timezone` body:

```bash
rekor bases create <base-id> --name "My DB" --timezone America/Sao_Paulo
rekor bases update <base-id> --timezone America/Sao_Paulo
# or via REST:
curl -X PUT https://api.rekor.pro/v1/bases/<base-id> \
  -H "Authorization: Bearer $REKOR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "settings": { "timezone": "America/Sao_Paulo" } }'
```

The value must be a valid IANA timezone (invalid names are rejected server-side). `rekor bases update --timezone` merges into the existing `settings` (it preserves other keys); the raw REST `PUT` and `--settings` flag **replace** the `settings` object wholesale, so send the full object. Omitted top-level fields (`name`, `description`, `tags`) are left unchanged on an existing base.

**Filtering is instant-aware:** `eq`/`neq`/`in`/`not_in` and the range operators match datetimes by point-in-time, so the exact representation you pass (`T` vs space, with or without offset) doesn't change the result — `{ "field": "data.starts_at", "op": "eq", "value": "2026-06-11T13:00:00" }` matches the stored value regardless of how it was written. Use `eq` for an exact datetime, not `like` (which is plain substring matching). Raw `rekor sql` is **not** rewritten this way — match the canonical stored form (or prefer the Filter DSL `query` for datetime conditions).
