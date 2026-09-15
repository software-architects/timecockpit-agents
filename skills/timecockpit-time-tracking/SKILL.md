---
name: timecockpit-time-tracking
description: Reviews, summarizes, and books a user's own working time in time cockpit over the connected MCP server. Use when someone asks what they worked on, wants a daily, weekly, or per-project summary of their hours, wants to find gaps, overlaps, or duplicate bookings in their timesheet, wants to book hours on a project or task, or wants to correct their recorded time.
compatibility: Requires a connected time cockpit MCP server (remote or local stdio host) exposing get_current_user, get_timesheets and describe_entity. create_timesheet is hidden and rejected when the session runs with read-only access; execute_tcql_query is hidden when the session is restricted to own data.
metadata:
  version: "1.2"
---

# Time Cockpit Time Tracking

Help a user review and record **their own** working time. Tool names below are bare; most clients
prefix them with the server name (`mcp__timecockpit__get_timesheets`). If a tool is missing from
the session, it is not available for this user — say so instead of guessing a workaround.

| Purpose | Tool |
|---|---|
| Who am I acting as | `get_current_user` |
| Read timesheets | `get_timesheets` |
| Book time | `create_timesheet` |
| Learn tenant-specific timesheet fields | `describe_entity` (`APP_Timesheet`) |
| Resolve a project/task UUID from a code or name | `execute_tcql_query` (narrow, parameterized) |

Time cockpit organizes work as **customer → project → task**. A timesheet entry belongs to one
user, has a begin and an end time, a description, and usually a project and a task.

## Session modes: check before promising anything

- **Read-only access** — `create_timesheet` is absent from the tool list and rejected if called.
  Tell the user their session cannot book time and offer to prepare the entries for review instead.
- **Own-data scope** — `get_timesheets` and `create_timesheet` are forced to the acting user
  server-side, and `execute_tcql_query` is *not* available at all. Resolve projects and tasks from
  the user's own past bookings in that mode (see below).
- **Other users' data** requires passing `userDetailId` *and* permission on the tenant. In own-data
  scope, a `userDetailId` that is not the acting user is rejected.

## 1. Identify the acting user

Call `get_current_user` first for any user-scoped work. It returns the acting user's full
`APP_UserDetail` record (`objectUuid`, `displayValue`, and every readable property). Use
`objectUuid` only where an id is needed — omit `userDetailId` on `get_timesheets`/`create_timesheet`
to mean "me", which is clearer and works in every mode.

## 2. Review a period

`get_timesheets` filters (all optional, combined with AND):

- `from` — **inclusive** ISO 8601, `to` — **exclusive**. So one week is `from: 2026-03-02`,
  `to: 2026-03-09`. A single day is `from` that day, `to` the next day.
- Both bounds filter **`APP_BeginTime` only**. An entry that starts before `from` is not returned
  even if it ends inside the range.
- `projectCode`, `taskCode` — exact match on the project's / task's `APP_Code`.
- `descriptionContains` — case-insensitive substring, matched literally (wildcards are escaped).
- `fields` — narrow the projection. Without it, *every* property of `APP_Timesheet` comes back,
  including tenant extensions; that is a lot of tokens per row. For review and summaries request
  only what you use, e.g. `["APP_BeginTime","APP_EndTime","APP_Description","APP_DurationInHours","APP_ProjectCodeActual"]`.
  `objectUuid` and `displayValue` are always included.
- `top` — default 50, capped by the server (typically 100).

Results are ordered by begin time **descending**. Always check `truncated` in the response: if it
is true, split the period into smaller ranges instead of reporting an incomplete total. Read the
returned `columns` array to see what you actually got — the property set is tenant-specific, so
never assume a column exists.

Relations (`APP_Project`, `APP_Task`, `APP_UserDetail`, …) come back as reference objects,
`{ "objectUuid": …, "displayValue": … }`, or `null` — never a bare UUID. Use `.objectUuid` for
lookups or when passing an id to another tool, and `.displayValue` for presentation; it is `null`
when the related record was not loaded.

## 3. Summarize

- Prefer the shipped calculated property **`APP_DurationInHours`** over doing arithmetic. Fall back
  to `APP_EndTime − APP_BeginTime` only when that column is absent from `columns`.
- **Times are tenant-local wall-clock values with no UTC offset.** Treat `2026-03-02T08:30:00` as
  08:30 in the user's own working day. Do not convert to UTC, do not add `Z` or an offset, and do
  not restate times in another zone.
- Group by day using the date part of `APP_BeginTime` (or `APP_DateActual`, a shipped calculated
  date). Group per project using `APP_ProjectCodeActual`, or `APP_Project.displayValue` from the
  relation object.
- State totals in hours with one or two decimals *and* say which period and which filters produced
  them. Report the count of entries alongside the total so the user can sanity-check.
- Entries with `APP_IsDurationTimesheet = true` are duration-only bookings; their begin/end times
  are bookkeeping values, not real clock times. Count their duration, but do not present them as
  a time span or use them for gap detection.

## 4. Find gaps, overlaps, and duplicates

Work from one sorted-ascending day at a time (see
[references/booking-patterns.md](references/booking-patterns.md) for the procedure):

- **Gap** — an uncovered span between the end of one entry and the begin of the next. Report gaps,
  do not fill them: only the user knows what happened in that time.
- **Overlap** — one entry begins before the previous one ends. Report both entries with their
  UUIDs so the user can decide which to correct.
- **Duplicate** — same day, same project/task, same or near-identical description, overlapping or
  adjacent times. Ask before assuming one is wrong.
- Only compare within one user and one day; the model forbids an entry spanning midnight anyway.

## 5. Resolve the project and task to book on

`create_timesheet` takes **UUIDs**, not codes. Two ways to get them:

- **Preferred, and the only way in own-data scope:** read the user's recent bookings for that
  project with `get_timesheets` (`projectCode` / `descriptionContains`) and reuse the project and
  task ids from a matching row: `row.APP_Project.objectUuid` and `row.APP_Task.objectUuid`.
- **Otherwise** use one narrow parameterized `execute_tcql_query`, e.g.
  `From P In APP_Project Where P.APP_Code = @Code Select Top(5) New With { P.APP_ProjectUuid, P.APP_Code, P.APP_ProjectName }`
  with `{"@Code": "…"}`. Never interpolate user input into the query text. Tasks work the same way
  over `APP_Task` (`APP_TaskUuid`, `APP_Code`, `APP_Description`).

If a code or name matches more than one record, or nothing, ask the user to choose — do not pick.

## 6. Book time — confirm first, always

`create_timesheet` parameters: `beginTime`, `endTime` (both ISO 8601, required), `description`,
`projectId`, `taskId`, `userDetailId`, `additionalFields`.

Before calling it, restate in one block and wait for an explicit yes:

> Date, begin–end time, resulting duration, project (code and name), task, description, and the
> user the entry is booked for.

Rules that are not negotiable:

- One confirmation per entry set. Never create speculative entries, never back-fill a gap, and
  never bulk-book a week from a rough description without listing every entry first.
- Run a `get_timesheets` read for that day first and warn about overlaps *before* creating.
- Write times as local wall clock **without** `Z` or an offset (`2026-03-02T08:30:00`). A supplied
  offset is reinterpreted against the server's own time zone and will silently shift the entry.
- `endTime` must be after `beginTime`, and both must fall on the **same calendar day** — split an
  overnight session into two entries.
- If you pass `taskId`, pass the matching `projectId` too: the model requires a project whenever a
  task is set, and the task must belong to that project.
- A **closed** project is rejected. So is writing into a period already closed for booking, or
  another user's timesheet without permission — surface those errors verbatim in plain words.
- `additionalFields` carries tenant `USR_` properties and shipped fields without a dedicated
  parameter (e.g. `APP_Location`, `APP_IsHomeOffice`). Learn what exists and what is required with
  `describe_entity('APP_Timesheet')`; look at `isCustom`, `isRequired`, `isCalculated`. Calculated
  and read-only members (`APP_DurationInHours`, `APP_DateActual`, `APP_ProjectCodeActual`,
  `APP_Revenue`, …) cannot be written. Begin/end time, description, project, task, and user must go
  through their own parameters, not `additionalFields`.

## 7. Interpret the result

`create_timesheet` returns a `MutationResult`: `status`, `entity`, `id`, `displayName`,
`changedFields`, `warnings`.

- `status: "completed"` plus an `id` means the entry exists. Report the id and the booked values.
- Read `warnings` out loud; they are the tenant's own remarks about the record.
- Verify by re-reading that day with `get_timesheets` before telling the user everything is fine,
  and confirm the duration matches what they approved.
- On an error, nothing was created. Repeat the message, name the likely cause (closed project,
  same-day rule, missing project for a task, permission), and propose a corrected call.

## Out of scope — use the neighbouring skills

- **Changing or removing an existing entry** goes through the generic `update_entity` /
  `delete_entity` tools, which require explicit confirmation. Use the
  `timecockpit-record-management` skill; do not improvise CRUD here.
- **Reporting across users, teams, customers, budgets, or billing** and anything needing free
  queries or named lists: `timecockpit-reporting` and `timecockpit-tcql`.
