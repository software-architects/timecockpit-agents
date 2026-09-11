# TCQL query cookbook

Starting points for common questions. Every entity and property name below is a **shipped**
(`APP_`) name verified against a live tenant, and every query was executed successfully through
`execute_tcql_query`.

Before you reuse one:

- A tenant may have removed a shipped entity, renamed nothing but added `USR_` properties, or
  captured the concept you need in a `USR_` entity instead. Run `describe_entity` on every entity
  the query touches when the answer matters.
- Replace the `#…#` date bounds with the period actually asked for: **begin inclusive, end
  exclusive**.
- Adjust `Top(N)` to what you will display. The server caps it (default 100) whatever you write.
- Remember the renaming: `T.APP_DurationInHours` comes back as `USR_DurationInHours`,
  `.Hours = Sum(…)` as `USR_Hours`. Read `columns` before building a table.

---

## Shipped entities you will use most

| Entity | Key | Display | Useful members |
|---|---|---|---|
| `APP_Timesheet` | `APP_TimesheetUuid` | `APP_Description` | `APP_BeginTime`, `APP_EndTime`, `APP_DurationInHours`, `APP_DurationInDays`, `APP_Billable`*, `APP_Billed`*, `APP_NoBilling`, `APP_Revenue`*, `APP_DateActual`*, `APP_ProjectCodeActual`*, `APP_CompanyNameActual`*, `APP_IsDurationTimesheet`, `APP_IsHomeOffice`, `APP_Location`; relations `APP_Project`, `APP_Task`, `APP_UserDetail`, `APP_Invoice`, `APP_WorkingTimeWeight` |
| `APP_Project` | `APP_ProjectUuid` | `APP_Fullname`* | `APP_Code`, `APP_ProjectName`, `APP_Closed`, `APP_Billable`, `APP_FixedPrice`, `APP_Budget`, `APP_BudgetInHours`, `APP_StartDate`, `APP_EndDate`, `APP_HourlyRate`, `APP_HourlyRateActual`*; relation `APP_Customer` |
| `APP_Task` | `APP_TaskUuid` | `APP_Fullname`* | `APP_Code`, `APP_Description`, `APP_CodeAndDescription`*, `APP_Closed`, `APP_Budget`, `APP_BudgetInHours`, `APP_Billable`*; relation `APP_Project` |
| `APP_Customer` | `APP_CustomerUuid` | `APP_CompanyName` | `APP_Code`, `APP_Town`, `APP_ZipCode`, `APP_Email`, `APP_HourlyRate`, `APP_VatID`; relation `APP_Country` |
| `APP_UserDetail` | `APP_UserDetailUuid` | `APP_Fullname`* | `APP_Username`, `APP_Firstname`, `APP_Lastname`, `APP_IsEnabled`, `APP_IsHidden`, `APP_IsTenantAdmin`, `APP_WeeklyHoursOfWork`, `APP_DateOfJoining`, `APP_DateOfSeparation`; relations `APP_Department`, `APP_LegalHolidayCalendar` |
| `APP_Department` | `APP_DepartmentUuid` | `APP_Code` | `APP_Code`, `APP_Description` — **that is all**; there is no `APP_Fullname` |
| `APP_Vacation` | `APP_VacationUuid` | `APP_Description` | `APP_BeginTime` (first day), `APP_EndTime` (last day), `APP_IsWholeDay`, `APP_SpecialLeave`, `APP_DurationInHours`*; relation `APP_UserDetail` |
| `APP_Invoice` | `APP_InvoiceUuid` | `APP_InvoiceNumber` | `APP_InvoiceDate`, `APP_InvoiceIsSent`, `APP_PaymentReceivedOn`, `APP_ServiceStartDate`, `APP_ServiceEndDate`, `APP_ReadOnlyRevenue`*, `APP_ReadOnlyGrossRevenueWithDiscount`*, `APP_Period`*; relation `APP_Project` |

`*` = calculated property: readable in queries, not writable.

---

## 1. Hours per project in a period

```
From T In APP_Timesheet
Where T.APP_BeginTime >= #2026-09-01# And T.APP_BeginTime < #2026-10-01#
Select Top(100) New With {
  .Project  = T.APP_Project.APP_Code,
  .Hours    = Sum(T.APP_DurationInHours),
  .Bookings = Count()
}
```

One row per project. Add `.Customer = T.APP_Project.APP_Customer.APP_CompanyName` to also group by
customer. Sort the returned rows yourself — ordering by an aggregate is not supported.

## 2. My own hours per project this month

```
From T In APP_Timesheet
Where T.APP_UserDetail.APP_UserDetailUuid = Environment.CurrentUser.APP_UserDetailUuid
  And T.APP_BeginTime >= #2026-09-01# And T.APP_BeginTime < #2026-10-01#
Select Top(100) New With {
  .Project = T.APP_Project.APP_Code,
  .Hours   = Sum(T.APP_DurationInHours)
}
```

For a plain list of one user's own entries, `get_timesheets` is cheaper and works in own-data mode.

## 3. Hours per user and month

```
From T In APP_Timesheet
Where T.APP_BeginTime >= #2026-01-01# And T.APP_BeginTime < #2027-01-01#
Select Top(100) New With {
  .User    = T.APP_UserDetail.APP_Fullname,
  .Period  = :FormatDateAsPeriod(T.APP_BeginTime),
  .Hours   = Sum(T.APP_DurationInHours),
  .Entries = Count()
}
```

`:FormatDateAsPeriod` returns `"YYYY/MM"`. Users × months can exceed the row cap quickly — query
one quarter at a time, or drop `.User` for a company total per month.

## 4. Billable hours and revenue per customer

```
From T In APP_Timesheet
Where T.APP_BeginTime >= #2026-07-01# And T.APP_BeginTime < #2026-10-01#
  And T.APP_Billable = True
Select Top(100) New With {
  .Customer = T.APP_Project.APP_Customer.APP_CompanyName,
  .Hours    = Sum(T.APP_DurationInHours),
  .Revenue  = Sum(T.APP_Revenue)
}
```

`APP_Billable` and `APP_Revenue` are calculated from the project/task setup and rates. State that
the figures are time-cockpit-calculated, not invoiced amounts — invoiced amounts live on
`APP_Invoice`.

## 5. Recent bookings of one project, with row ids

```
From T In APP_Timesheet
Where T.APP_Project.APP_Code = @Code
  And T.APP_BeginTime >= #2026-09-01#
Order By T.APP_BeginTime Desc
Select Top(50) New With {
  T.APP_BeginTime, T.APP_EndTime, T.APP_Description, T.APP_DurationInHours,
  .User = T.APP_UserDetail.APP_Fullname,
  .Id   = :ObjectUuid(T)
}
```

`parameters: {"@Code": "PROJ-001"}`. `.Id` is the real timesheet key — the envelope's `objectUuid`
is a generated id for projected rows and must not be used to identify a record.

## 6. Projects of a customer

```
From P In APP_Project
Where P.APP_Customer.APP_CompanyName Like @Name And P.APP_Closed = False
Order By P.APP_Code
Select Top(50) New With {
  P.APP_Code, P.APP_ProjectName, P.APP_StartDate, P.APP_EndDate, P.APP_BudgetInHours,
  .Customer = P.APP_Customer.APP_CompanyName,
  .Id       = :ObjectUuid(P)
}
```

`parameters: {"@Name": "%Contoso%"}`. Escape `%` and `_` in the user's text if they should match
literally, or use `=` on `APP_Customer.APP_Code` for an exact lookup.

## 7. Open tasks

```
From Ta In APP_Task
Where Ta.APP_Closed = False And Ta.APP_Project.APP_Closed = False
Order By Ta.APP_Project.APP_Code, Ta.APP_Code
Select Top(100) New With {
  .Project = Ta.APP_Project.APP_Code,
  Ta.APP_Code, Ta.APP_Description, Ta.APP_BudgetInHours,
  .Id = :ObjectUuid(Ta)
}
```

`APP_Task.APP_Closed` is nullable in the shipped model: if tasks are missing from the result, use
`Where (Ta.APP_Closed = False Or Ta.APP_Closed = Null)`.

## 8. Resolve a UUID from a code (before a mutation)

```
From P In APP_Project Where P.APP_Code = @Code
Select Top(5) New With { P.APP_ProjectUuid, P.APP_Code, P.APP_ProjectName, P.APP_Closed }
```

```
From Ta In APP_Task Where Ta.APP_Code = @Code And Ta.APP_Project.APP_Code = @Project
Select Top(5) New With { Ta.APP_TaskUuid, Ta.APP_Code, Ta.APP_Description }
```

Ask for `Top(5)`, not `Top(1)`: if more than one record matches, or none does, let the user choose
instead of picking one. The key comes back as `USR_ProjectUuid` / `USR_TaskUuid`.

## 9. Budget burn-down per project

```
From P In APP_Project
Where P.APP_Closed = False And P.APP_BudgetInHours <> Null
Order By P.APP_Code
Select Top(50) New With {
  P.APP_Code, P.APP_BudgetInHours,
  .BookedHours = ( From T In P.Timesheets Select New With { .H = Sum(T.APP_DurationInHours) } )
}
```

A project with no bookings returns `null`, not `0` — treat null as zero when computing the
remaining budget, and do the subtraction and percentage yourself when presenting.

## 10. Project counts and totals per customer

```
From P In APP_Project
Order By P.APP_Customer.APP_CompanyName
Select Top(100) New With {
  .Customer = P.APP_Customer.APP_CompanyName,
  .Projects = Count(),
  .Budget   = Sum(P.APP_BudgetInHours)
}
```

## 11. Active users

```
From U In APP_UserDetail
Where U.APP_IsHidden = False
Order By U.APP_Lastname, U.APP_Firstname
Select Top(100) New With {
  U.APP_Fullname, U.APP_Username, U.APP_WeeklyHoursOfWork,
  .Department = U.APP_Department.APP_Code
}
```

`APP_IsEnabled` is nullable and is often Null in practice, so `Where U.APP_IsEnabled = True`
silently returns nothing. Filter on `APP_IsHidden` (required) or write
`(U.APP_IsEnabled = True Or U.APP_IsEnabled = Null)`.

## 12. Bookings per user (via back reference)

```
From U In APP_UserDetail
Where U.APP_IsHidden = False
Select Top(100) New With {
  U.APP_Fullname,
  .Bookings = ( From T In U.Timesheets Select New With { .N = Count() } )
}
```

## 13. Absences in a period

```
From V In APP_Vacation
Where V.APP_BeginTime < #2026-10-01# And V.APP_EndTime >= #2026-09-01#
Order By V.APP_BeginTime
Select Top(100) New With {
  .User = V.APP_UserDetail.APP_Fullname,
  V.APP_BeginTime, V.APP_EndTime, V.APP_IsWholeDay, V.APP_SpecialLeave, V.APP_Description
}
```

Note the overlap test: `APP_EndTime` is the **last day** of the absence, so a range overlap needs
both bounds, unlike a timesheet query which filters `APP_BeginTime` only. `APP_SickLeave` follows
the same shape — run `describe_entity` on it first.

## 14. Invoices of a period

```
From I In APP_Invoice
Where I.APP_InvoiceDate >= #2026-07-01# And I.APP_InvoiceDate < #2026-10-01#
Order By I.APP_InvoiceDate Desc
Select Top(50) New With {
  I.APP_InvoiceNumber, I.APP_InvoiceDate, I.APP_ReadOnlyRevenue, I.APP_InvoiceIsSent,
  I.APP_PaymentReceivedOn,
  .Customer = I.APP_Project.APP_Customer.APP_CompanyName
}
```

Unpaid invoices: add `And I.APP_PaymentReceivedOn = Null`.

## 15. Planned vs. actual working time (per user)

```
From U In APP_UserDetail
Where U.APP_IsHidden = False
Select Top(50) New With {
  U.APP_Fullname,
  .Planned  = :PlannedHoursOfWork(U.APP_UserDetailUuid, #2026-01-01#, #2026-12-31#, True),
  .Actual   = :ActualHoursOfWork(U.APP_UserDetailUuid, #2026-01-01#, #2026-12-31#, True),
  .Overtime = :Overtime(U.APP_UserDetailUuid, #2026-12-31#, True, True),
  .RemainingVacationWeeks = :RemainingVacationWeeks(U.APP_UserDetailUuid, #2026-12-31#)
}
```

These functions take an **inclusive** end date, unlike a `Where` range. They depend on the
tenant's working-time and vacation-entitlement configuration; if the numbers look wrong, say that
the configuration is the likely cause rather than adjusting the query. `RemainingVacationWeeks`
is in weeks, not days.

## 16. How big is the answer, before asking for rows

```
From T In APP_Timesheet
Where T.APP_BeginTime >= #2026-01-01# And T.APP_BeginTime < #2027-01-01#
Select Top(1) New With { .Entries = Count(), .Hours = Sum(T.APP_DurationInHours) }
```

One row, no grouping keys. Use this when a row-level query came back `truncated: true`, so you can
tell the user the true total and then decide how to slice the detail.
