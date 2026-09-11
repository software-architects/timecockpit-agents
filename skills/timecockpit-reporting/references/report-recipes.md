# Report recipes

Concrete starting points for the questions users ask most. Every recipe assumes the rules in
`SKILL.md`: check for a named list first, verify property names with `describe_entity`, pass values
as parameters, and report the data window.

Property names below are the **shipped** ones. A tenant may have renamed nothing but added a lot --
`USR_` properties are frequently the dimension the user actually means.

## Verified shipped names used here

| Entity | Properties used |
| --- | --- |
| `APP_Timesheet` | `APP_BeginTime`, `APP_EndTime`, `APP_DurationInHours`, `APP_Description`, `APP_Project`, `APP_Task`, `APP_UserDetail`, `APP_Billable`, `APP_Billed`, `APP_Revenue` |
| `APP_Project` | `APP_Code`, `APP_ProjectName`, `APP_Customer`, `APP_Budget`, `APP_BudgetInHours`, `APP_Billable`, `APP_Closed`, `APP_StartDate`, `APP_EndDate`, `APP_Manager` |
| `APP_Task` | `APP_BudgetInHours` |
| `APP_Customer` | `APP_Code`, `APP_CompanyName` |
| `APP_UserDetail` | `APP_Firstname`, `APP_Lastname`, `APP_Fullname`, `APP_Department`, `APP_IsEnabled` |
| `APP_Vacation` | `APP_BeginTime`, `APP_EndTime`, `APP_Description`, `APP_IsWholeDay`, `APP_IsApproved`, `APP_SpecialLeave`, `APP_UserDetail` |
| `APP_VacationEntitlement` | `APP_VacationEntitlementDate`, `APP_NumberOfWeeks`, `APP_UserDetail` |

Shipped named lists worth checking with `get_lists` before writing any query:
`APP_BudgetaryControlOfProjectsList`, `APP_BudgetaryControlOfTasksList`,
`APP_UnbilledTimesheetsList`, `APP_VacationPerEffectiveDateList`,
`APP_OvertimePerEffectiveDateList`, `APP_DefaultWorkingTimeViolationList`.

## Hours per customer and project

```
From T In APP_Timesheet
Where T.APP_BeginTime >= @FromDate And T.APP_BeginTime < @ToDate
Select New With {
  .Customer = T.APP_Project.APP_Customer.APP_CompanyName,
  .Project  = T.APP_Project.APP_ProjectName,
  .Hours    = Sum(T.APP_DurationInHours),
  .Bookings = Count(T.APP_TimesheetUuid) }
```

Drop `.Project` for customer totals only; drop both grouping keys and keep the aggregates for a
single grand-total row.

## Hours per user for a period

```
From T In APP_Timesheet
Where T.APP_BeginTime >= @FromDate And T.APP_BeginTime < @ToDate
Select New With {
  .User  = T.APP_UserDetail.APP_Fullname,
  .Hours = Sum(T.APP_DurationInHours) }
```

For one user only, prefer `get_timesheets(from, to, userDetailId)` and total the rows -- but check
`truncated` before you do.

## Billable vs. non-billable split

```
From T In APP_Timesheet
Where T.APP_BeginTime >= @FromDate And T.APP_BeginTime < @ToDate
Select New With {
  .Project  = T.APP_Project.APP_ProjectName,
  .Billable = T.APP_Billable,
  .Hours    = Sum(T.APP_DurationInHours) }
```

Two rows per project (`True`/`False`). Present it as a table with a billable share per project.
`APP_Billable` is the flag on the booking; `APP_Billed` says whether it has been invoiced -- they
answer different questions.

## Budget vs. actual

TCQL aggregates one entity's rows at a time, so run two small queries and join client-side.

```
From P In APP_Project
Where P.APP_Closed = False
Select Top(100) New With { P.APP_Code, P.APP_ProjectName, P.APP_BudgetInHours }
```

```
From T In APP_Timesheet
Where T.APP_BeginTime < @ToDate
Select New With {
  .Code  = T.APP_Project.APP_Code,
  .Hours = Sum(T.APP_DurationInHours) }
```

Match on `APP_Code`, then report booked, budget, remaining and percent consumed. Say whether the
actual figure covers the whole project life or only the queried window -- booked-to-date needs no
lower bound. Check `APP_BudgetaryControlOfProjectsList` first; where it exists, it already answers
this question with the tenant's own definition.

## Utilization

Utilization needs a denominator, and the denominator is a tenant policy decision. Ask which one
applies before computing anything:

- target hours derived from `APP_WorkingTimeWeight` / `APP_WorkingTimeLimit`,
- a fixed contracted number of hours per week that the user gives you, or
- billable share of booked hours (no target hours needed).

The last one is the only variant computable from timesheets alone:

```
From T In APP_Timesheet
Where T.APP_BeginTime >= @FromDate And T.APP_BeginTime < @ToDate
Select New With {
  .User     = T.APP_UserDetail.APP_Fullname,
  .Billable = T.APP_Billable,
  .Hours    = Sum(T.APP_DurationInHours) }
```

Label that result "billable share of booked hours", never plain "utilization", unless the user
defined the target.

## Missing bookings

There is no "missing booking" entity -- a gap is the absence of rows, so ask the server for what
*is* there per person and period, then compare against the working calendar you were given.

```
From T In APP_Timesheet
Where T.APP_BeginTime >= @FromDate And T.APP_BeginTime < @ToDate
  And T.APP_UserDetail.APP_UserDetailUuid = @User
Select New With { .Begin = T.APP_BeginTime, .Hours = T.APP_DurationInHours }
```

Keep the window short enough to stay under the row cap (a week per call is usually safe), bucket
the begin times into days yourself, and only then subtract absences (`APP_Vacation`) and public
holidays -- a vacation day with no booking is not a gap. State the working-day assumption you used.
If the tenant has a shipped or custom list for booking completeness, use it instead.

## Absence overview

```
From V In APP_Vacation
Where V.APP_BeginTime < @ToDate And V.APP_EndTime >= @FromDate
Select New With {
  .User     = V.APP_UserDetail.APP_Fullname,
  .Approved = V.APP_IsApproved,
  .Records  = Count(V.APP_VacationUuid) }
```

Day counting is not a row count: records span ranges and may be partial days
(`APP_IsWholeDay = False`). For anything the user will act on, list the individual records instead
of a computed day figure, or use `APP_VacationPerEffectiveDateList`. Entitlement lives in
`APP_VacationEntitlement` (`APP_NumberOfWeeks` as of `APP_VacationEntitlementDate`) and needs the
tenant's own weeks-to-days conversion -- do not guess a multiplier.

## Presentation checklist

- Markdown table, total row, two decimals, unit on the number.
- Data window as an explicit half-open range.
- Scope: which users, which projects, and which flags were included or excluded.
- `truncated` and `warnings` from any result, relayed on the spot.
- Source named: which list or which query produced the figure.
