---
name: timecockpit-record-management
description: Creates, updates and deletes time cockpit records through the MCP tools create_entity, update_entity and delete_entity, including the mandatory describe_entity discovery step, the confirmed=true handshake and a verification re-read. Use when the user asks to add, change, correct, close, archive or delete a project, customer, task, user, absence, timesheet or any tenant-defined record, or when a mutation was rejected as invalid, read-only or unconfirmed. Not for building analytical reports (see timecockpit-reporting).
compatibility: Requires an AI client connected to a time cockpit MCP server (remote or local host) whose session is not in ReadOnly access or OwnData scope. Tool names may carry a client prefix.
metadata:
  version: "1.0"
---

# Time Cockpit Record Management

Write operations against a live tenant. Every create, update and delete is real data in a
production database, so the sequence below is not optional.

Related skills: `timecockpit-model-discovery` (entities and properties),
`timecockpit-time-tracking` (timesheets), `timecockpit-lists-and-actions` (named lists and model
actions), `timecockpit-tcql` (query syntax), `timecockpit-mcp-setup` (connection, access modes).

Tool names are written bare here. Most clients prefix them, e.g. `mcp__<server>__create_entity`.

## 1. Pick the right tool first

Generic CRUD is the escape hatch, not the default:

1. **Typed tool** — `create_timesheet` for time bookings. It validates begin/end, resolves the
   acting user and accepts tenant properties via `additionalFields`.
2. **Model action** — `get_actions('<Entity>')` then `execute_action`. Actions carry the tenant's
   own business logic (approvals, invoicing, status changes). Prefer one when it matches the intent.
3. **Generic CRUD** — `create_entity` / `update_entity` / `delete_entity` for everything else.

## 2. Discover before you write

Always call `describe_entity('<EntityName>')` before the first write to an entity in a session.
Use `get_entities(nameContains: '...')` if you do not know the exact name. Never guess a name.

From the metadata, read off:

- `isRequired` — must be present in a create payload.
- `isRelation` + `relatedEntity` — the value is a **UUID of a record in that entity**, never a
  code, name or label. Look the UUID up first with `execute_tcql_query`.
- `isCalculated`, `isReadOnly`, `isPrimaryKey` — rejected by the server; never send them.
- `isCustom` (`USR_` prefix) — tenant-specific extensions. They may be required and often carry
  the tenant's real business meaning; treat them as first-class, not optional extras.
- `dataType` — drives the JSON value type (see below).
- `displayPropertyName` — the property whose value you quote back to the user.

## 3. Build the `data` payload

`data` is a **native JSON object**, not a string containing JSON.

```json
{ "APP_Code": "PRJ-042", "APP_ProjectName": "Website relaunch",
  "APP_Customer": "8f14e45f-ea0a-4c9b-9c4a-2b2f9d7c1a01",
  "APP_StartDate": "2026-04-01", "APP_Billable": true, "APP_BudgetInHours": 120 }
```

Rules:

- Relation properties take a UUID string, or `null` to clear. Both the relation name
  (`APP_Customer`) and its `...Uuid` form (`APP_CustomerUuid`) are accepted.
- Dates and times are ISO 8601: `2026-04-01`, `2026-04-01T08:30:00`.
- Booleans are JSON `true` / `false` — not `"true"`, `"Yes"` or `1`.
- Numbers are JSON numbers; do not localise the decimal separator.
- On `update_entity`, send **only the properties that change**. Omitted properties keep their
  values; an unnecessary key is a silent extra change on the record.

## 4. The confirmation handshake

Generic mutations are confirmation-gated by deployment policy. The first call returns

```json
{ "status": "confirmation_required", "entity": "APP_Project",
  "id": "...", "message": "Confirm the generic update operation, ..." }
```

and **nothing has happened yet**. Then:

1. Read the record back (`execute_tcql_query`, or `get_timesheets` for timesheets) so you can name
   its `displayValue` — the confirmation payload does not include it.
2. Restate to the user in plain language: entity, UUID, the record's display value, and each field
   with its old and new value. For a create, restate every property you are about to set.
3. Wait for the user's explicit yes. A previous yes never covers a new record.
4. Re-invoke the *same* tool with identical arguments plus `confirmed: true`.

Some sandbox deployments run with confirmation disabled, so a mutation can succeed on the first
call. Ask the user anyway — the handshake is the user's control, not the server's.

## 5. Deletes

- Irreversible. There is no undo, no recycle bin and no MCP restore path.
- **One record per call, one confirmation per record.** `delete_entity` takes a single UUID.
- Never loop over a result set and delete it. If the user wants several records gone, list them as
  a numbered table with display values, get explicit approval for that exact batch, then delete
  them one at a time and report each result.
- Prefer a soft alternative when the model offers one: `APP_Project.APP_Closed`, an `APP_IsHidden`
  flag, or a tenant status property. Suggest it before deleting.
- Related records may block the delete or be cascaded away by the model. Say so before confirming.

## 6. Read the `MutationResult`

| Field | Meaning |
| --- | --- |
| `status` | `completed`, or `confirmation_required` (nothing was written) |
| `entity` | the entity that was written |
| `id` | UUID of the record; on a create this is the new record's key |
| `displayName` | display text where the host supplies one |
| `changedFields` | echo of the property names **you sent**, not a server-side diff |
| `warnings` | non-fatal remarks — always relay them |

Because `changedFields` is only an echo, finish with a verification read: query the record by its
UUID, compare the values that matter, and report what the tenant actually stores now.

```
describe_entity('APP_Project')
execute_tcql_query("From C In APP_Customer Where C.APP_Code = @Code Select Top(1) New With { C.APP_CustomerUuid, C.APP_CompanyName }", { "@Code": "ACME" })
create_entity('APP_Project', { ... })                 -> status: confirmation_required
create_entity('APP_Project', { ... }, confirmed=true) -> status: completed, id: ...
execute_tcql_query("From P In APP_Project Where P.APP_ProjectUuid = @Id Select Top(1) P", { "@Id": "..." })
```

Correcting one timesheet description, after `get_timesheets` produced the `objectUuid`:

```
update_entity('APP_Timesheet', '<uuid>', { "APP_Description": "Sprint review" })                 -> confirmation_required
update_entity('APP_Timesheet', '<uuid>', { "APP_Description": "Sprint review" }, confirmed=true) -> completed
get_timesheets(from: '2026-04-01', to: '2026-04-02')
```

## 7. When it fails

| Symptom | Cause and fix |
| --- | --- |
| `has no member '...'` | Wrong or invented property name. Re-run `describe_entity`. |
| `is calculated or read-only` / `is the primary key` | Remove that key from `data`. |
| `Relation member '...' must be a non-empty UUID or null` | You sent a code or name. Resolve the UUID first. |
| `does not exist in this tenant's data model` | Typo, or the entity is hidden from MCP (system and internal entities are never writable). |
| Validation or trigger error from the tenant | The model's own rule rejected the payload. Report the message; do not retry with altered values to get around it. |
| Permission denied | The acting user's role lacks write permission. Not fixable from the agent side. |
| `create_entity` / `update_entity` / `delete_entity` not offered at all | The session is `Access=ReadOnly`, or `Scope=OwnData` (generic CRUD is hidden there — use `create_timesheet`). Only the operator can change this; see `timecockpit-mcp-setup`. |
| `not available in read-only mode` | Same cause, tool invoked directly. |

## Never do

- Never write to an entity you have not described in this session.
- Never invent an entity, property or relation name, and never carry a name over from another tenant.
- Never pass `data` as a JSON string, and never quote values to make them fit.
- Never send `confirmed: true` on the first call, or on a payload the user has not seen.
- Never treat one approval as covering several records, a retry with changed values, or a follow-up
  mutation.
- Never delete in bulk, and never delete to clean up something the user did not ask about.
- Never guess a UUID or reuse one from an example, another tenant or an unverified lookup.
- Never write to system (`SYS_`) or internal entities, or try to reach them through a relation.
- Never suppress a warning, error or truncation notice from a result.
- Never report success without a verification read.
