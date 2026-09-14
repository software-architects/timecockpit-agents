# Booking and review patterns

Worked procedures for the `timecockpit-time-tracking` skill. Tool names are bare; clients prefix
them with the server name.

## Verified `APP_Timesheet` members worth knowing

Shipped members of the timesheet entity. A tenant may add `USR_` properties and may hide or extend
these — always reconcile against `describe_entity('APP_Timesheet')` and against the `columns` array
of the actual response.

| Member | Kind | Use |
|---|---|---|
| `APP_BeginTime` | writable date/time | Start of the entry ("Start"). The only member the `from`/`to` filters look at. |
| `APP_EndTime` | writable date/time | End of the entry ("End"). Must be after begin, same calendar day. |
| `APP_Description` | writable text | What was done. Required in the standard configuration. |
| `APP_Project` | relation → `APP_Project` | Set on create via `projectId`, in `additionalFields` it would be `APP_ProjectUuid`. |
| `APP_Task` | relation → `APP_Task` | Set on create via `taskId`. |
| `APP_UserDetail` | relation → `APP_UserDetail` | Set on create via `userDetailId`. |
| `APP_DurationInHours` | calculated | Duration in hours — read this instead of computing. |
| `APP_DurationInDays` | calculated | Duration in person-days. |
| `APP_DateActual` | calculated date | Day of the entry; convenient for grouping. |
| `APP_ProjectCodeActual` | calculated | Effective project code; convenient for per-project grouping. |
| `APP_UserDetailName` | calculated | User name, useful when reading another user's data. |
| `APP_IsDurationTimesheet` | writable flag | True for duration-only bookings; begin/end are then bookkeeping values. |
| `APP_BeginTimeActual`, `APP_EndTimeActual` | calculated | Begin/end for from-to bookings only. |
| `APP_Location`, `APP_IsHomeOffice` | writable | Location, telework — settable through `additionalFields`. |
| `APP_Billable`, `APP_Billed`, `APP_NoBilling` | writable / calculated mix | Billing state; check `isReadOnly` before writing. |
| `APP_Revenue`, `APP_HourlyRateActual` | calculated | Never writable. |

Related entities: `APP_Project` has `APP_Code` ("Project Code"), `APP_ProjectName`, `APP_Closed`,
`APP_Billable`, and a `APP_Customer` relation. `APP_Task` has `APP_Code` ("Task Code"),
`APP_Description`, `APP_Closed`, `APP_CodeAndDescription`.

## Model validation rules the server enforces

This is the **shipped** rule set. A tenant may have added its own validation rules or triggers on
`APP_Timesheet`, so treat a rejection you cannot map to items 1-7 as one of those, not as a bug,
and relay the server's message verbatim.

Anticipate these instead of letting the create fail:

1. End time must be after begin time.
2. Begin and end time must be on the **same day**.
3. If a task is selected, a project must be selected as well.
4. The selected task must belong to the selected project.
5. A closed project must not be selected.
6. Timesheets marked "no billing" must belong to a billable project or task.
7. No write permission for other users' timesheets, or for timesheets before the tenant's booking
   completion date.

## Reviewing one week

1. `get_current_user` (only if an id is needed later).
2. `get_timesheets` with `from` = Monday, `to` = the **following** Monday, `top: 100`, and
   `fields: ["APP_BeginTime","APP_EndTime","APP_Description","APP_DurationInHours","APP_ProjectCodeActual"]`.
3. If `truncated` is true, repeat per day or per half-week and merge.
4. Sort ascending by `APP_BeginTime` yourself — the tool returns descending.
5. Present a per-day table (day, entries, hours) plus a per-project table, then the week total and
   the number of entries behind it.

## Gap and overlap pass for one day

```
entries = rows for that day, APP_IsDurationTimesheet not true, sorted ascending by APP_BeginTime
cursor  = begin of the first entry
for each entry:
    if entry.APP_BeginTime > cursor      -> gap    from cursor to entry.APP_BeginTime
    if entry.APP_BeginTime < cursor      -> overlap between the previous entry and this one
    cursor = max(cursor, entry.APP_EndTime)
```

Report gaps longer than a few minutes only; rounding produces noise. Report every overlap with both
`objectUuid` values. Never create an entry to close a gap without being asked for that specific
entry.

## Resolving a project by code or name

Own-data scope (no `execute_tcql_query`): look at the user's own history.

```
get_timesheets { descriptionContains: "<keyword>", top: 20,
                 fields: ["APP_BeginTime","APP_Description","APP_ProjectCodeActual"] }
```

Then reuse the project/task ids from the row the user recognizes:
`row.APP_Project.objectUuid` / `row.APP_Task.objectUuid` (each is `{objectUuid, displayValue}` or
`null`).

Default scope, exact code:

```
execute_tcql_query {
  query: "From P In APP_Project Where P.APP_Code = @Code And P.APP_Closed <> True
          Select Top(5) New With { P.APP_ProjectUuid, P.APP_Code, P.APP_ProjectName }",
  parameters: { "@Code": "ACME-2026" }
}
```

Fuzzy by name — keep `Top` small and let the user pick:

```
execute_tcql_query {
  query: "From P In APP_Project Where P.APP_ProjectName Like @Name
          Select Top(10) New With { P.APP_ProjectUuid, P.APP_Code, P.APP_ProjectName }",
  parameters: { "@Name": "%website%" }
}
```

Tasks of one project:

```
execute_tcql_query {
  query: "From T In APP_Task Where T.APP_Project.APP_Code = @Code
          Select Top(25) New With { T.APP_TaskUuid, T.APP_Code, T.APP_Description }",
  parameters: { "@Code": "ACME-2026" }
}
```

Pass values as parameters (names must start with `@`, scalars only). Never build the query text
from user input.

## A booking, end to end

1. Resolve project and task ids (above).
2. `get_timesheets` for that single day to see what is already booked.
3. Restate and ask:

   > Monday 2026-03-02, 09:00–11:30 (2.5 h), project ACME-2026 "Website Relaunch", task DEV
   > "Development", description "Implemented the search filter", booked for you. Create it?

4. On a clear yes:

   ```
   create_timesheet {
     beginTime: "2026-03-02T09:00:00",
     endTime:   "2026-03-02T11:30:00",
     description: "Implemented the search filter",
     projectId: "…", taskId: "…"
   }
   ```

5. Report `status`, `id`, and any `warnings`; re-read the day and confirm the duration.

## Several entries at once

List every proposed entry in one table (date, begin, end, duration, project, task, description),
get one explicit approval for the whole list, then create them one at a time in chronological
order. Stop at the first failure, report which entries were created and which were not, and ask
before retrying. Do not treat "book my week as usual" as approval for anything.
