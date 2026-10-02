---
name: timecockpit-lists-and-actions
description: Discovers and runs a time cockpit tenant's named lists (saved views) with get_lists, describe_list and execute_list. Use when a user asks for a saved view or "the list we use for X", or wants report-like data that already exists in the app. Model actions are not available over MCP; when a user wants one run (for example creating an invoice or closing a project), direct them to the time cockpit UI.
compatibility: Any agent connected to a time cockpit MCP server (remote or local host). All three tools depend on optional deployment capabilities and may be absent; execute_list is additionally unavailable in own-data sessions.
metadata:
  version: "2.1"
---

# time cockpit Lists

**Named lists** are the saved views a tenant authored in its own model: a stored query plus
filter parameters, columns, grouping and sorting. They encode business rules the tenant already
agreed on ("which timesheets count as unbilled"), so a matching list usually beats a query you
write yourself.

**Model actions** — named operations in the model such as creating an invoice or closing a
project — are **not available over MCP** in either host. No tool lists or runs them. When a user
wants an action run, tell them to run it in the time cockpit UI; afterwards you can read the
affected records to confirm the outcome. Do not emulate an action with `create_object`,
`update_object` or `delete_object`: an action carries tenant logic, approvals and side effects that
a sequence of object edits does not reproduce. (This skill keeps its historical name so existing
installations and references keep working; it covers lists only.)

Tool names below are bare; clients prefix them with the server name, e.g.
`mcp__timecockpit__get_lists`.

## 1. Check availability before promising anything

These three tools are capability-gated and may simply not exist for a session: `get_lists`
(`named_lists`), `describe_list` and `execute_list` (`execute_lists`). When a capability is
unavailable the tool is removed from the tool list *and* rejected if called directly. On top of
that, an **own-data** session hides `execute_list` because its results cannot be constrained to the
acting user (`get_lists` and `describe_list` remain).

So if you cannot see a tool, the capability is off for this deployment, tenant or session. Say
that plainly and fall back: reads can go through `timecockpit-tcql`. Never guess a technical list
name to call a tool you cannot see.

## 2. Lists: discover, describe, execute

### get_lists

`filter` is a case-insensitive substring matched against the technical name *and* the friendly
name — not a regular expression. It is applied **before** the server-side row limit, so a match
far down a large catalog is found rather than silently dropped. Each entry reports `name`,
`friendlyName`, `targetEntity` and a `parameters` map keyed by parameter name (`@BeginTime`) with
a `type` and `mandatory` flag.

The result also carries `truncated`. `truncated: true` means the catalog held further lists that
did not fit the limit — narrow with `filter` and look again before telling a user a list does not
exist. An empty `parameters` map means none were discovered, not proof that the list takes none.
Only lists the acting user is permitted to see are returned.

### describe_list

Reports the list's result `columns` and its declared `parameters` (mandatory/optional). `rows` is
always empty — `describe_list` never returns data. It does **not**, however, promise to leave the
list unexecuted: deriving columns may require running the list under a 5-row cap.

- On a **local** host, a TCQL-backed list's columns come straight from its definition (parsed and
  validated, never dispatched), so they are correctly typed even when the list currently matches
  zero rows. A script-backed list's shape is taken from the script itself; if that fails, the host
  falls back to a **5-row sample** execution and says so in `warnings`. `parameters` is read from
  the list's filter form — the same source `get_lists` uses, so the two tools never disagree.
- A **remote** host has no in-process access to a list's query/script text, so it **always** runs a
  **5-row sample** execution: a column whose sampled value was null still comes back as `Unknown`,
  an empty sample yields only the two stable columns (`objectUuid`, `displayValue`) plus a warning,
  and `parameters` is always an empty map there — pass the parameters you intend to use so the
  sample is representative.

Check `warnings` either way; a sample-derived result says so explicitly. Treat `describe_list` as a
cheap, capped read rather than a guaranteed no-op: on an unfamiliar script-backed list it may run
the list's own signed script, so it can be slower than its "metadata" framing suggests.

### execute_list

Inputs: `listName`, an optional `parameters` object, an optional `condition`.

- Parameter keys must start with `@` and match the declared names. Values are JSON scalars:
  string, number, boolean or null — ISO 8601 for dates and timestamps, GUID strings for
  relation and Guid parameters. Objects and arrays are rejected.
- Omitting an optional parameter usually means "do not filter on it"; shipped lists commonly
  guard their filters with an `Or @X = Null` clause. A `mandatory` parameter must be supplied.
- `condition` is an extra TCQL Where fragment combined with the list's own query. It is caller
  input, so it is screened for hidden-entity references and capped like raw TCQL. Use it to
  narrow a list further, never to reshape it.
- The row cap (100 by default) is applied *before* execution. `truncated` is exact here: the host
  reads one row past the cap to find out, so a result that exactly fills the cap reports the truth
  either way. Check it and narrow with parameters or `condition` rather than asking for more rows.

```
get_lists     { "filter": "unbilled" }
describe_list { "listName": "APP_UnbilledTimesheetsList",
                "parameters": { "@BeginTime": "2026-01-01", "@EndTime": "2026-01-31" } }
execute_list  { "listName": "APP_UnbilledTimesheetsList",
                "parameters": { "@BeginTime": "2026-01-01", "@EndTime": "2026-01-31",
                                "@Project": "00000000-0000-0000-0000-000000000000" } }
```

## 3. A list or raw TCQL?

Prefer a **list** when the user names one or names a business view; when the tenant's own
definition already encodes rules you would otherwise have to reinvent (what counts as unbilled,
how it is grouped and sorted); or when the answer must match what the user sees in the app.
Also run `get_lists` with a filter for the user's own wording even when no shipped entity seems to
cover the concept: a tenant-defined (`USR_`) list is exactly how such gaps usually get closed (see
`timecockpit-model-discovery`, section 7).
Prefer **TCQL** (`timecockpit-tcql`) when `get_lists` turns up nothing matching, when you need a
projection, aggregation or join the list does not produce, or when you need a shape that stays
stable regardless of later edits to the list. Do not re-implement a list you found — run it. And
never try to change a list definition here; lists are signed artifacts owned by the model editor.

## 4. Interpreting the tabular result

`execute_list`, `describe_list` and every other record-returning tool share one result envelope:

```jsonc
{
  "columns": [ { "name": "USR_BeginTime", "dataType": "DateTime", "isCustom": true } ],
  "rows":    [ { "objectUuid": "…", "USR_BeginTime": "2026-01-05T08:30:00" } ],
  "count": 1, "truncated": false, "warnings": []
}
```

`describe_list`'s response additionally carries a `parameters` map (same shape as `get_lists`'
`parameters`: `{ "@BeginTime": { "type": "DateTime", "mandatory": true } }`); no other tool
populates that field.

- A named list runs as a `New With` query internally, so `columns` are **inferred from the
  returned values**, not typed from the model: no `label`, `dataType` guessed from the data
  (`Unknown` for an all-null column, `Relation` for a nested object), and every column name gets
  the tenant `USR_` prefix regardless of whether it is a real tenant extension (see
  `timecockpit-tcql` for why). Read `columns` instead of assuming a name or type.
- `rows` normally carry `objectUuid` (the key for a follow-up read or an `update_object` call), but
  `displayValue` is typically `null` or absent — anonymous projections have no display property. A
  relation column reads as `{ "objectUuid": …, "displayValue": … }` or `null` — never a bare UUID
  or a bare string; its own `displayValue` is likewise `null` or absent when the related record was
  not loaded. Use `.objectUuid` for lookups and `.displayValue` for presentation, following up with
  `describe_entity` or a targeted query if the user needs a name that is missing.
- `count` is the number of rows returned; `truncated: true` means data was cut off. Never
  present a truncated result as a total or as a complete list. `warnings` is informational —
  surface anything that changes how the numbers should be read.

When reporting to a user, prefer the list's `friendlyName`, and a human-friendly heading with the
`USR_` prefix dropped for each column; keep technical names for follow-up calls.

On the local host, a list's definition (query and script source) is also readable through the
model resources — read it when what the list counts is not obvious (`timecockpit-model-discovery`).

## 5. Failure modes

| Symptom | Meaning |
| --- | --- |
| tool missing from the tool list | capability off for this deployment/tenant, or an own-data session (`execute_list`) |
| list not found | wrong name, or not permitted for this user — re-run `get_lists`; do not try name variants |
| `describe_list` returns no columns | the 5-row sample was empty (remote host always samples; local host samples only as a script-list fallback); supply parameters that select data |
| parameter rejected | key missing its `@`, or the value is an object or array |
| `condition` rejected | it references a hidden entity or breaks a guardrail; narrow with parameters instead |
| `truncated: true` | the row cap applied; narrow the request, do not stitch pages |
| user asks to run a model action | not available over MCP in either host — the user runs it in the time cockpit UI |

## Related skills

`timecockpit-model-discovery` (entities, properties, relations), `timecockpit-tcql` (queries
when no list fits), `timecockpit-time-tracking` (typed timesheet reads and writes),
`timecockpit-record-management` (generic create, update, delete), `timecockpit-reporting`
(turning results into figures), `timecockpit-mcp-setup` (connecting a client, access modes and
scopes), `timecockpit-list-deeplinks` (a link that opens a list in the web app with its filters
pre-filled), `timecockpit-documentation-lookup` (what a shipped list is documented to show).
