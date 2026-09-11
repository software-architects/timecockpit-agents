---
name: timecockpit-lists-and-actions
description: Discovers and runs a time cockpit tenant's named lists (saved views) with get_lists, describe_list and execute_list, and its model actions with get_actions and execute_action, including the confirmation handshake for state-changing actions. Use when a user asks for a saved view or "the list we use for X", wants report-like data that already exists in the app, or wants to run a model action such as creating an invoice or closing a project.
compatibility: Any agent connected to a time cockpit MCP server (remote or local host). All five tools depend on optional deployment capabilities and may be absent; execute_action is additionally unavailable in read-only sessions and execute_list and execute_action in own-data sessions.
metadata:
  version: "1.0"
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
name — not a regular expression. Each entry reports `name`, `friendlyName`, `targetEntity` and
a `parameters` map keyed by parameter name (`@BeginTime`) with a `type` and `mandatory` flag.

An empty `parameters` map means none were discovered, not proof that the list takes none. Only
lists the acting user is permitted to see are returned.

### describe_list

Columns only, derived from a **5-row sample** — it never performs a full run. So a column whose
sampled value was null comes back as `unknown`; an empty sample yields no columns plus a warning
telling you to supply parameters; and `rows` is always empty, because `describe_list` never
returns data. Pass the parameters you intend to use, so the sample is representative.

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
- The row cap (100 by default) is applied *before* execution. Check `truncated` and narrow with
  parameters or `condition` rather than asking for more rows.

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
Prefer **TCQL** (`timecockpit-tcql`) when `get_lists` turns up nothing matching, when you need a
projection, aggregation or join the list does not produce, or when you need a shape that stays
stable regardless of later edits to the list. Do not re-implement a list you found — run it. And
never try to change a list definition here; lists are signed artifacts owned by the model editor.

## 4. Interpreting the tabular result

`execute_list`, `describe_list` and every other record-returning tool share one result envelope:

```jsonc
{
  "columns": [ { "name": "APP_BeginTime", "label": "Begin", "dataType": "DateTime", "isCustom": false } ],
  "rows":    [ { "objectUuid": "…", "displayValue": "Review", "APP_BeginTime": "2026-01-05T08:30:00" } ],
  "count": 1, "truncated": false, "warnings": []
}
```

- `columns` describes **this** response — the tenant's model decides the shape, so read it from
  `columns` instead of assuming. Each column carries `name`, `label`, `dataType`, `isCustom`,
  and `relatedEntity` for relations.
- In `rows`, only `objectUuid` and `displayValue` are guaranteed. Everything else, shipped
  `APP_` members included, appears flat under its technical name. A relation column carries the
  referenced record's display value; follow up with `describe_entity` or a targeted query when
  you need more of that record.
- `count` is the number of rows returned; `truncated: true` means data was cut off. Never
  present a truncated result as a total or as a complete list. `warnings` is informational —
  surface anything that changes how the numbers should be read.

When reporting to a user, use the column `label` and the list's `friendlyName`; keep technical
names for follow-up calls. `objectUuid` values are what you feed into an action's `inputSet`.

## 5. Actions: discover, then execute

### get_actions

`entityName` is required — actions are discovered per entity, so establish the entity first
(`timecockpit-model-discovery`). Each entry reports `name`, `displayName`, `parameterEntity`,
`minimumInputSetSize` and `executeWithoutPrompt`. Only actions permitted for the acting user and
visible in lists are returned.

- `minimumInputSetSize` greater than zero means the action needs at least that many records.
- `parameterEntity` names a typed parameter entity; run `describe_entity` on it to learn the
  parameter fields. Those names may carry no `APP_`/`USR_` prefix — use them verbatim.
- `executeWithoutPrompt: false` means the action normally opens a dialog in the desktop client.
  Some shipped actions also report progress through message boxes, which have nowhere to go in
  an agent session. If such an action stalls or returns no clear result, report that instead of
  retrying.

### execute_action

Inputs: `actionName`, `inputSet` (an array of record UUID strings, capped at around 50),
`parameter` (a JSON object matching the parameter entity), `entityName` (the entity context),
and `confirmed`.

1. `get_actions` on the entity.
2. `describe_entity` on `parameterEntity`, if the action declares one.
3. Collect the record UUIDs — the `objectUuid` values from `execute_list`,
   `execute_tcql_query` or `get_timesheets`.
4. Call `execute_action` **without** `confirmed`. It returns
   `status: "confirmation_required"` plus a message; nothing has run.
5. Restate the operation to the user in plain language: which action, how many records, which
   parameter values, and what will change. Wait for their answer.
6. Re-invoke the identical call with `confirmed: true`. Success returns `status: "completed"`
   plus the action's result.

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
| `describe_list` returns no columns | the 5-row sample was empty; supply parameters that select data |
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
