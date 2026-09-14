---
name: timecockpit-model-discovery
description: Explores a time cockpit tenant's customized data model over MCP with get_entities, describe_entity, get_environment and the local data model resources, and turns the result into technical names that actually exist. Use when a user asks which fields an entity has, how customers, projects, tasks and timesheets are related, what a tenant administrator customized, or before writing TCQL, a field projection, a list parameter, or any create, update or delete call.
compatibility: Any agent connected to a time cockpit MCP server (remote HTTP host or local stdio host). The timecockpit:// resources and the get_resource fallback exist on the local host only.
metadata:
  version: "1.1"
---

# time cockpit Model Discovery

No two time cockpit tenants share the same data model. Administrators add properties to
shipped entities, define entities of their own, and rename captions. So assume nothing:
**discover, then act**. A guessed technical name is the most common cause of a failed call
and of a confidently wrong answer.

Tool names here are bare. Clients prefix them with the server name, for example
`mcp__timecockpit__describe_entity`.

## 1. Discovery order

1. `get_environment` (or `get_current_user`) — confirms which environment and acting user the
   session is bound to. `get_environment` returns `currentUser` (the acting user's
   `APP_UserDetail` record), `globalSettings` (the tenant's `APP_GlobalSettings` record, or
   `null` when not readable) and `namedSets` (evaluated named sets keyed by name, e.g.
   `APP_CurrentUserRoles`; a tenant may define its own — read the keys rather than assume only
   that one). Records use the same shape as query rows: relations are reference objects,
   credential-like members are omitted. It reports identity, not permissions; never infer from it
   what the user may read or write. The server authorizes every call independently.
2. `get_entities` — the entity inventory. `nameContains` is a case-insensitive substring;
   `customOnly: true` returns only tenant-defined entities.
3. `describe_entity` — the full shape of one entity. Call it before you write a property name
   anywhere.

Metadata is stable within a session: discover an entity once and reuse the result.

## 2. Naming

| Prefix | Meaning |
| --- | --- |
| `APP_` | shipped with time cockpit — `APP_Timesheet`, `APP_Project`, `APP_Task`, `APP_Customer`, `APP_UserDetail` |
| `USR_` | added or defined by this tenant's administrator |
| `SYS_` | internal; never exposed on any MCP surface |

- Primary keys follow `<EntityName>Uuid`: `APP_Timesheet` gives `APP_TimesheetUuid`.
  `describe_entity` reports it as `primaryKeyName`, and record rows also carry it as
  `objectUuid`.
- `isCustom: true` marks tenant-specific entities and properties. It is the direct answer to
  "what did our administrator change": `get_entities` with `customOnly: true` for whole
  entities, plus the `isCustom` properties on shipped entities.
- Some model elements carry no prefix at all (action parameter entities in particular). Use the
  name exactly as the tool reported it; never normalize or "fix" it.
- `label` is the caption users see and may be renamed or localized. Match the user's wording
  against `label`, but always send `name` to the server.

## 3. Reading describe_entity

`properties` is one flat list holding stored properties *and* relations:

```jsonc
{
  "name": "APP_Timesheet", "primaryKeyName": "APP_TimesheetUuid",
  "displayPropertyName": "APP_Description",
  "properties": [
    { "name": "APP_BeginTime", "dataType": "DateTime", "isRequired": true },
    { "name": "APP_DurationInHours", "dataType": "Number", "isCalculated": true },
    { "name": "APP_Project", "dataType": "Relation", "isRelation": true, "relatedEntity": "APP_Project" },
    { "name": "USR_Billable", "dataType": "Boolean", "isCustom": true }
  ]
}
```

- `isRelation` / `relatedEntity` — a navigation to another entity. This is how the model is
  connected; there is no separate relations list.
- `isCalculated` — evaluated, not stored. Readable and queryable, never writable.
- `isBlob` — large binary or file content. Excluded from default projections; request it
  explicitly and only when the user actually needs it.
- `isRequired` — must be supplied on create. `isDisplayProperty` backs the record's
  `displayValue`. `isReadOnly` is populated by some hosts only, so treat a missing value as
  unknown rather than as writable.
- `dataType` is a small logical set (`String`, `Number`, `Integer`, `Boolean`, `DateTime`,
  `Date`, `Time`, `Duration`, `Uuid`, `Relation`, `Binary`, `Unknown`), not a storage type.

## 4. Relations and how the domains connect

Read `relatedEntity`; never assume a chain. In a stock model the business core links as
`APP_Timesheet.APP_Project`, `APP_Timesheet.APP_Task`, `APP_Timesheet.APP_UserDetail` and
`APP_Project.APP_Customer`, so a customer name reads as
`APP_Timesheet.APP_Project.APP_Customer.APP_CompanyName`. Confirm each hop with
`describe_entity` before relying on it: a tenant may add its own relation and expect that one to
be used instead. Present relationships as navigation paths, naming the entity each hop lands on.

## 5. Forms and lists are not part of describe_entity

`describe_entity` reports data shape only. Default forms and lists are separate model artifacts:
`get_lists` reports named lists with the `targetEntity` they read — that is how to find the
saved views belonging to an entity (see `timecockpit-lists-and-actions`) — and on the local host
a list's detail resource also names the form it opens for editing. Shipped artifacts follow
patterns like `APP_DefaultTimesheetList` and `APP_TimesheetForm`, but confirm any such name
through `get_lists` before using it.

## 6. Local host only: model resources

- `timecockpit://datamodel` — compact index of every visible entity with property, relation and
  trigger counts, plus visible list and action names. The fastest whole-model overview.
- `timecockpit://datamodel/lists/{listName}` and `.../actions/{actionName}` — full definitions,
  including query or script source.
- `timecockpit://datamodel/entities/{entityName}/triggers/{triggerName}` — an enabled trigger.
- `timecockpit://currentUser`, `timecockpit://globalSettings` — the same records as
  `get_environment`'s `currentUser`/`globalSettings`: relation members are `{objectUuid,
  displayValue}` objects (not reduced to bare UUIDs), and credential-like members are omitted.

The index omits property names and source text by design; expand through `describe_entity` or
the detail resources. Clients that cannot read MCP resources natively pass the same URI to
`get_resource`. The remote host has no resources — use the discovery tools there.

## 7. Recipes

**"Which fields does a timesheet have?"**

```
get_entities    { "nameContains": "timesheet" }
describe_entity { "entityName": "APP_Timesheet" }
```

Answer grouped — identity, time, assignment (relations), billing — and call out the tenant
additions (`isCustom`) separately, since those are what the admin added.

**"How are our customers, projects, tasks and timesheets related?"** Run `describe_entity` on
`APP_Timesheet`, then on each `relatedEntity` it names, and report the resulting navigation
paths. Flag any `isCustom` relation as a tenant-specific link.

**"What did our administrator customize?"** `get_entities { "customOnly": true }` for
tenant-defined entities, then `describe_entity` on each business entity and list the properties
where `isCustom` is true.

**"We don't seem to have an entity for X" (attendance, cost centre, tickets, ...):** absence from
the shipped `APP_` set is not absence from the model. Run `get_entities { "nameContains": "<keyword>" }`
and `get_entities { "customOnly": true }`, and check `get_lists { "filter": "<keyword>" }` for a
matching saved view, before telling the user the tenant does not track the concept.

## 8. Feeding discovery into the rest of the surface

Discovered names go straight into `timecockpit-tcql` (queries and filters),
`timecockpit-time-tracking` (typed timesheet reads and writes, `fields` and `additionalFields`),
`timecockpit-record-management` (generic create, update, delete),
`timecockpit-lists-and-actions` (saved views and model actions) and `timecockpit-reporting`
(turning results into figures). For connecting a client, access modes and scopes, see
`timecockpit-mcp-setup`.

## 9. When metadata is unavailable

- `describe_entity` on an unknown, internal or hidden entity returns a not-found error pointing
  back at `get_entities`. Do not retry with spelling variants and do not guess a technical
  name — hidden entities (`SYS_*` plus a fixed set of shipped telemetry and notification
  entities) are absent from every surface by design. Tell the user that data is not exposed
  over MCP.
- `get_entities` may report `truncated: true` when the response bound was reached; narrow with
  `nameContains` instead of assuming the model is that small.
- Optional tools (`get_lists`, `describe_list`, `execute_list`, `get_actions`,
  `execute_action`) may be absent for a deployment, tenant or session. Absence is a
  configuration fact, not an error.
- Never invent a property to make an answer fit. Report what the model does contain, and offer
  the closest discovered alternative.
