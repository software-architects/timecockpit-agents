---
name: timecockpit-reporting
description: Answers analytical questions about time cockpit data with the read-only MCP tools — hours per project, customer, user or period, budget versus actual, utilization, missing bookings and absence overviews — using named lists where the tenant has them and aggregating TCQL queries otherwise. Use when the user asks how many hours, how much was booked or billed, who is over or under budget, what is still missing, or asks for a summary, breakdown, totals or an export of time data. Not for creating or changing records (see timecockpit-record-management).
compatibility: Requires an AI client connected to a time cockpit MCP server (remote or local host). Named-list tools depend on an optional server capability and may be absent. Tool names may carry a client prefix.
metadata:
  version: "1.0"
---

# Time Cockpit Reporting

Read-only analysis of a live tenant. The whole job is to get the right *aggregate* out of the
server, because the server will not hand you the raw rows behind a large report.

Related skills: `timecockpit-tcql` (query syntax — all of it),
`timecockpit-lists-and-actions` (named lists), `timecockpit-model-discovery` (entity and property
discovery), `timecockpit-time-tracking` (booking time), `timecockpit-mcp-setup` (connection).

Tool names are written bare here. Most clients prefix them, e.g. `mcp__<server>__execute_list`.

## 1. Choose the data source in this order

1. **Named list** — `get_lists(filter: 'budget')`, then `describe_list('<name>')` to see the
   columns, then `execute_list('<name>', parameters, condition)`. A list is the tenant's own,
   reviewed definition of a report, so it beats anything you can assemble yourself. Shipped
   examples include `APP_BudgetaryControlOfProjectsList`, `APP_BudgetaryControlOfTasksList`,
   `APP_UnbilledTimesheetsList` and `APP_VacationPerEffectiveDateList`. `get_lists` and
   `execute_list` may be absent from the tool list entirely — that means the capability is not
   available for this deployment, not that the tenant has no lists.
2. **`get_timesheets`** — one user, one date range, typed filters (`from`, `to`, `userDetailId`,
   `projectCode`, `taskCode`, `descriptionContains`, `fields`, `top`). Best for "what did I book
   last week". It returns raw rows, so totals are yours to compute.
3. **`execute_tcql_query`** — cross-entity and multi-user analysis, and the only way to get the
   server to aggregate for you.

Confirm what "the user" means with `get_current_user` before reporting on "my" anything.

## 2. Respect the row cap — aggregate, do not page

Every record-returning tool is capped server-side (default **100 rows**; `get_timesheets`
defaults to 50). There is no paging cursor, and `Top(N)` is inserted or lowered to the cap
automatically. Results carry `truncated` plus a warning; a result that exactly fills the cap may be
truncated even when the server cannot prove it.

So a month of bookings for a team will never come back as raw rows. Three legitimate ways out:

- **Aggregate in the query.** One row per project or per user fits easily inside 100 rows.
- **Slice the window.** Loop day by day, week by week, or project by project, and total the slices
  yourself — but only when aggregation cannot express the question.
- **Narrow the question.** Ask the user for the period, the projects or the people that matter.

Never claim a total from a `truncated` result. Either aggregate, slice, or say the window is too
wide and name what you need.

## 3. TCQL aggregation (verified)

TCQL supports the aggregate functions `Sum`, `Min`, `Max`, `Avg`, `Count` and `CountDistinct`.
Note the spelling `Avg`, and that aggregates take no leading colon (unlike scalar functions such
as `:Iif`).

**There is no `Group By` clause.** Grouping is implicit: every expression in the `Select` that is
*not* inside an aggregate becomes a grouping key. So this returns one row per project:

```
From T In APP_Timesheet
Where T.APP_BeginTime >= @FromDate And T.APP_BeginTime < @ToDate
Select New With { .Project = T.APP_Project.APP_ProjectName, .Hours = Sum(T.APP_DurationInHours) }
```

Add `.Customer = T.APP_Project.APP_Customer.APP_CompanyName` and you group by customer *and*
project. Remove `.Project` and you get one row per customer. That is the whole mechanism.

Two limits to plan around:

- Sorting by an aggregated value is documented as unsupported against the local data store — sort
  the returned rows yourself.
- `Top(N)` caps the number of **result rows**, i.e. groups. 100 groups is usually plenty; 100 raw
  bookings is not. If a grouped result is still truncated, the grouping is too fine.

Pass user-supplied values as parameters (`@FromDate`, `@ToDate`, ...), never interpolated into the query
text. Comments (`//`, `/* */`) are rejected outright. Full syntax lives in `timecockpit-tcql`.

## 4. Durations

`APP_Timesheet.APP_DurationInHours` is the shipped decimal hours property — use it and `Sum()` it.
Only compute `T.APP_EndTime - T.APP_BeginTime` when you deliberately need the raw interval, and
say which one you used. Booked hours are not attendance hours and not billable hours: `APP_Billable`
and `APP_Billed` are separate flags, and `APP_Revenue` is money, not time.

Verify with `describe_entity('APP_Timesheet')` before relying on any property — tenants add `USR_`
properties (cost centre, activity type, billing state) that often *are* the dimension the user
means.

## 5. Project time vs. attendance time

The shipped model tracks **project time** in `APP_Timesheet`. Absences live in `APP_Vacation`
(`APP_BeginTime`, `APP_EndTime`, `APP_IsWholeDay`, `APP_IsApproved`, `APP_SpecialLeave`,
`APP_UserDetail`), with entitlements in `APP_VacationEntitlement` and target working time in
`APP_WorkingTimeWeight` / `APP_WorkingTimeLimit`.

There is **no shipped attendance-time entity exposed over MCP** — the raw clock/attendance signal
entities are internal and hidden. A tenant that tracks attendance does so through its own `USR_`
entity or a `USR_` flag on the timesheet. So: run `get_entities(nameContains: 'attend')` and
`get_entities(customOnly: true)` before answering an attendance question, and if nothing turns up,
tell the user this tenant does not expose attendance data to the agent rather than silently
reporting project time as attendance.

## 6. Present the answer

- A markdown table, with a **total row**, and hours rounded to two decimals with a unit.
- State the data window explicitly ("bookings with begin time from 2026-04-01 up to, not
  including, 2026-05-01").
- State the scope: which users, which projects, whether unapproved or non-billable rows are in.
- If any result was `truncated`, say so on the same line as the number, or do not give the number.
- Relay `warnings` from the result verbatim in a short note.
- Say where the figure came from — named list, `get_timesheets`, or an aggregating query.

## 7. Two worked sequences

Hours per project for one month:

```
get_lists(filter: 'budget')                      -> nothing usable? fall through to TCQL
describe_entity('APP_Timesheet')                 -> confirm APP_DurationInHours + USR_ dimensions
execute_tcql_query(
  "From T In APP_Timesheet Where T.APP_BeginTime >= @FromDate And T.APP_BeginTime < @ToDate
   Select New With { .Project = T.APP_Project.APP_ProjectName, .Hours = Sum(T.APP_DurationInHours) }",
  { "@FromDate": "2026-04-01", "@ToDate": "2026-05-01" })
-> sort by hours client-side, table + total, note the window
```

Budget vs. actual:

```
get_lists(filter: 'budget')                      -> APP_BudgetaryControlOfProjectsList?
describe_list('APP_BudgetaryControlOfProjectsList')
execute_list('APP_BudgetaryControlOfProjectsList')
-> otherwise: query APP_Project (APP_Code, APP_ProjectName, APP_BudgetInHours) and the
   Sum(APP_DurationInHours) per project separately, then join the two small results yourself
```

More patterns, including per-user utilization, missing bookings and absence overviews:
[references/report-recipes.md](references/report-recipes.md).

## Never do

- Never total a `truncated` result, and never page a wide report by repeating the same query.
- Never invent an entity, property or list name; discover it, or say you could not find it.
- Never present project time as attendance time, booked hours as billable hours, or unapproved
  absences as approved.
- Never interpolate user values into TCQL instead of using parameters.
- Never report a figure without its data window and scope.
- Never use a mutating tool while answering an analytical question.
