---
name: timecockpit-lists-and-actions
description: Discovers and runs a time cockpit tenant's named lists (saved views) with get_lists, describe_list and execute_list, and its model actions with get_actions and execute_action, including the confirmation handshake for state-changing actions. Use when a user asks for a saved view or "the list we use for X", wants report-like data that already exists in the app, or wants to run a model action such as creating an invoice or closing a project.
compatibility: Any agent connected to a time cockpit MCP server (remote or local host). All five tools depend on optional deployment capabilities and may be absent; execute_action is additionally unavailable in read-only sessions and execute_list and execute_action in own-data sessions.
metadata:
  version: "1.3"
---

# time cockpit Lists and Actions

**Named lists** are the saved views a tenant authored in its own model: a stored query plus
filter parameters, columns, grouping and sorting. They encode business rules the tenant already
agreed on ("which timesheets count as unbilled"), so a matching list usually beats a query you
write yourself.

**Model actions** are named operations in the model — scripts that create, change or delete
records. They are the only way to trigger tenant business logic over MCP, and they are the most
dangerous surface here.

Tool names below are bare; clients prefix them with the server name, e.g.
`mcp__timecockpit__get_lists`.

## 1. Check availability before promising anything

These five tools are capability-gated and may simply not exist for a session: `get_lists`
(`named_lists`), `describe_list` and `execute_list` (`execute_lists`), `get_actions`
(`model_actions`), `execute_action` (`execute_actions`). When a capability is unavailable the
tool is removed from the tool list *and* rejected if called directly. On top of that, a
**read-only** session hides and rejects `execute_action`, and an **own-data** session hides
`execute_list` and `execute_action` because their results cannot be constrained to the acting
user (`get_lists`, `describe_list` and `get_actions` remain).

So if you cannot see a tool, the capability is off for this deployment, tenant or session. Say
that plainly and fall back: reads can go through `timecockpit-tcql`, but nothing substitutes for
an action. Never guess a technical list or action name to call a tool you cannot see.

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

Reports the list's result `columns` and its declared `parameters` (mandatory/optional) without
running the list — `rows` is always empty, because `describe_list` never returns data.

- On a **local** host, columns come straight from the list's own TCQL/script definition (parsed
  and validated, never executed), so they are correctly typed even when the list currently
  matches zero rows, and `parameters` is read from the list's filter form — the same source
  `get_lists` uses, so the two tools never disagree about a list's parameters.
- A **remote** host has no in-process access to a list's query/script text, so it falls back to a
  small **5-row sample** execution: a column whose sampled value was null still comes back as
  `Unknown`, an empty sample yields only the two stable columns (`objectUuid`, `displayValue`)
  plus a warning, and `parameters` is always an empty map there — pass the parameters you intend
  to use so the sample is representative. Check `warnings` either way; a sample-derived result
  says so explicitly.

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
- `rows` normally carry `objectUuid` (feed it into an action's `inputSet`), but `displayValue` is
  typically `null` or absent — anonymous projections have no display property. A relation column
  reads as `{ "objectUuid": …, "displayValue": … }` or `null` — never a bare UUID or a bare
  string; its own `displayValue` is likewise `null` or absent when the related record was not
  loaded. Use `.objectUuid` for lookups/`inputSet` entries and `.displayValue` for presentation,
  following up with `describe_entity` or a targeted query if the user needs a name that is missing.
- `count` is the number of rows returned; `truncated: true` means data was cut off. Never
  present a truncated result as a total or as a complete list. `warnings` is informational —
  surface anything that changes how the numbers should be read.

When reporting to a user, prefer the list's `friendlyName`, and a human-friendly heading with the
`USR_` prefix dropped for each column; keep technical names for follow-up calls.

## 5. Actions: discover, then execute

### get_actions

`entityName` is optional. Supply it — the normal case — and you get that entity's actions, so
establish the entity first (`timecockpit-model-discovery`). Omit it and you get the **standalone**
actions instead: the ones bound to no entity, which `execute_action` runs without an `entityName`.
That is the only way to discover a standalone action; an entity's listing never includes one you
could run entity-free, and the standalone listing never includes an entity-bound action. Each entry
reports `name`, `displayName`, `parameterEntity`, `minimumInputSetSize` and `executeWithoutPrompt`.
Only actions the acting user holds execute permission for are returned. Desktop UI placement is
not a filter: an action the app shows no button for is still listed and runnable here if the user
may execute it.

- Looking for a tenant-wide maintenance or import action and not finding it under any entity? Call
  `get_actions` with no arguments.
- If a call with no `entityName` reports that the standalone actions could not be determined, that
  is a deployment limitation, not an empty result: do not report "there are no standalone actions".
  Fall back to `get_actions` with an `entityName`.

- `minimumInputSetSize` greater than zero means the action needs at least that many records.
- `parameterEntity` names a typed parameter entity; run `describe_entity` on it to learn the
  parameter fields. Those names may carry no `APP_`/`USR_` prefix — use them verbatim.
- `executeWithoutPrompt: false` means the action normally opens a dialog in the desktop client.
  Some shipped actions also report progress through message boxes, which have nowhere to go in
  an agent session. If such an action stalls or returns no clear result, report that instead of
  retrying.

### execute_action

Inputs: `actionName` — **always required** — plus `entityName`, `inputSet` (an array of record
UUID strings, capped at around 50), `parameter` (a JSON object matching the parameter entity),
and `confirmed`.

`entityName` is **required whenever `inputSet` is non-empty, and optional otherwise**. The records
of an input set belong to an entity and actions are authorized per entity, so a call that passes an
input set without `entityName` is rejected with a message telling you to run `get_actions` first. A
standalone action — one with no input set at all — has no entity to name and is authorized against
the standalone catalog instead (the one `get_actions` lists with no arguments), so omit `entityName`
there rather than guessing one. Either way the
action name is resolved against exactly the catalog `get_actions` publishes — guessing a name you
did not see there fails, it does not reach the tenant. `parameter` is validated against the live parameter
entity before anything runs: unknown members, calculated or read-only members, the parameter's own
identifier, `SYS_*` members, and values of the wrong type are all rejected. That is why step 2
below matters — take the member names from `describe_entity`, do not invent them.

1. `get_actions` on the entity — or with no arguments when you are after a standalone action.
2. `describe_entity` on `parameterEntity`, if the action declares one.
3. Collect the record UUIDs — the `objectUuid` values from `execute_list`,
   `execute_tcql_query` or `get_timesheets`.
4. Call `execute_action` **without** `confirmed`. It returns
   `status: "confirmation_required"` plus a message; nothing has run.
5. Restate the operation to the user in plain language: which action, how many records, which
   parameter values, and what will change. Wait for their answer.
6. Re-invoke the identical call with `confirmed: true`. Success returns `status: "completed"`
   plus the action's result: `action`, `succeeded`, `output`, `modifiedInputSet` (the UUIDs the
   action changed — re-read them to verify) and, when the action produced one, `actionResult`.

`actionResult` is one of three shapes and never a payload: `type: "uri"` (a link, with `uri`,
`fileName`, `displayUri`, `open`), `type: "file"` (`contentType`, `fileName`, `byteCount`,
`inline` — metadata only, the file's content is *not* returned over MCP), or `type: "other"` with
a fixed summary sentence meaning "the action produced something this tool does not expose". Report
`other` as exactly that; do not tell the user the action failed, and do not guess what it made.

```
get_actions { "entityName": "APP_Timesheet" }
// -> APP_CreateInvoiceAction, parameterEntity "CreateInvoiceParameter", minimumInputSetSize 1

execute_action {
  "actionName": "APP_CreateInvoiceAction",
  "entityName": "APP_Timesheet",
  "inputSet": ["…uuid…", "…uuid…"],
  "parameter": { "InvoiceDate": "2026-02-01", "InvoiceNumber": "2026-0042",
                 "InvoiceDescription": "January consulting" }
}
// -> status "confirmation_required" -> restate to the user -> same call plus "confirmed": true
```

Never send `confirmed: true` on the first call, and never treat the handshake as a formality —
it exists so a human sees the scope before records change. An operator may have disabled
confirmation for a controlled sandbox, in which case the first call executes immediately, so
explain the operation *before* you call it rather than relying on the server to stop you.

## 6. Actions change state and cannot be stopped

- Assume every action is destructive until you have read what it does. Actions run tenant
  logic: they create and delete records, flag work as billed, or push data onward.
- Once an action has started it cannot be interrupted. Cancelling the request, stopping the
  agent, or a client timeout does not roll it back — the work may complete anyway. Do not retry
  an action that timed out; read the affected records first to find out what happened.
- There is no undo. When the effect is uncertain, run it against a single record, verify, then
  widen the `inputSet`.
- On the local host, action and list definitions (query and script source) are readable through
  the model resources — read the definition when the effect is not obvious
  (`timecockpit-model-discovery`).
- A failed action surfaces as an error rather than a partial result. Re-read the records to
  establish what actually changed before telling the user anything.

## 7. Failure modes

| Symptom | Meaning |
| --- | --- |
| tool missing from the tool list | capability off for this deployment/tenant, or a read-only / own-data session |
| list or action not found | wrong name, or not permitted for this user — re-run `get_lists` / `get_actions`; do not try name variants |
| "an entity context is required" on `execute_action` | `entityName` was omitted or blank on a call that passed an `inputSet` — run `get_actions` for the entity and pass its name; a standalone action with no input set needs no `entityName` |
| "cannot be executed without an entity context" on `execute_action` | the action was called with no `inputSet` and no `entityName`, but it is entity-scoped — pass the entity, or pick one from `get_actions` with no arguments |
| "could not be determined for this tenant" on `get_actions` / `execute_action` with no `entityName` | this deployment cannot resolve the standalone catalog — not a statement that none exist; use `get_actions` with an `entityName` |
| action "is not available for entity" | the name is not in that entity's catalog — re-run `get_actions`, do not retry with a variant |
| a parameter member rejected | it is unknown, calculated, read-only, the identifier, or the wrong type — re-read `describe_entity` on `parameterEntity` |
| `describe_list` returns no columns | remote host only: the 5-row sample was empty; supply parameters that select data |
| parameter rejected | key missing its `@`, or the value is an object or array |
| `condition` rejected | it references a hidden entity or breaks a guardrail; narrow with parameters instead |
| `truncated: true` | the row cap applied; narrow the request, do not stitch pages |
| `status: "confirmation_required"` | expected on a first action call; restate, then re-invoke with `confirmed: true` |

## Related skills

`timecockpit-model-discovery` (entities, properties, relations), `timecockpit-tcql` (queries
when no list fits), `timecockpit-time-tracking` (typed timesheet reads and writes),
`timecockpit-record-management` (generic create, update, delete), `timecockpit-reporting`
(turning results into figures), `timecockpit-mcp-setup` (connecting a client, access modes and
scopes).
