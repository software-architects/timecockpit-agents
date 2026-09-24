# TCQL reference

Complete syntax reference for queries sent to `execute_tcql_query`. Load this when you need an
exact function signature, a construct not covered in SKILL.md, or the precise guardrail rules.

**Contents:** 1. Statement grammar · 2. Expressions (literals, Null semantics, Booleans, date
arithmetic, Like) · 3. Parameters · 4. Functions · 5. Aggregation and implicit grouping ·
6. Relations, back references and sub-queries · 7. Include · 8. In Set and Environment ·
9. Anonymous vs typed projections · 10. Server-side guardrails · 11. Result envelope

---

## 1. Statement grammar

```
<statement> ::=
  <fromClause> [<whereClause>] [<orderByClause>] <selectClause>

<fromClause> ::=
  From <alias> In <source>

<source> ::=
  <entityName>[.Include("<relationPath>")[.Include("<relationPath>")...]]
  <entityName>.Include(*)

<whereClause> ::=
  Where <searchCondition>

<orderByClause> ::=
  Order By <expression> [Asc|Desc] [, <expression> [Asc|Desc]]...

<selectClause> ::=
  Select [Top(N)] <alias>
  Select [Top(N)] <alias> With(VersionInfoOnly)
  Select [Top(N)] New [<TargetEntity>] With { <member> [, <member>]... }

<member> ::=
  <alias>.<property>[.<property>...]
  .<aliasName> = <expression>
```

- Clause order is fixed. `Order By` comes before `Select`.
- `Top(N)` and the bare `Top N` form are both accepted; it sits immediately after `Select`.
- Keywords are **case sensitive** (`From`, not `from`). The range-variable alias must start with a
  capital letter.
- Identifiers are the technical names from the data model: `APP_` for shipped members, `USR_` for
  tenant extensions.

### Skip / Take

The language has `Skip N` / `Take N` between `Order By` and `Select`, but they are **not usable
through `execute_tcql_query`**: the server always enforces a `Top` clause, which conflicts with
them. Aggregate or narrow instead of paging.

### With(VersionInfoOnly)

Reads only row version information, for sync scenarios. Not useful for reporting.

---

## 2. Expressions

```
<expression> ::=
  ( <expression> )
  | <expression> [Or | And] <expression>
  | <expression> [= | <> | < | <= | > | >=] <expression>
  | <expression> In ( <expression> [, <expression>...] )
  | <expression> In Set( "<expressionName>" [, "<propertyName>"] )
  | <expression> In @<parameter>
  | <expression> [Not] Like <expression>
  | <expression> [+ | - | * | / | %] <expression>
  | Not <expression>
  | :<FunctionName>( [<expression>[, <expression>...]] )
  | Sum(<expression>) | Min(…) | Max(…) | Avg(…) | CountDistinct(…) | Count()
  | <alias>.<member>[.<member>...]
  | <literal>
  | @<parameter>
  | ( <fromClause> [<whereClause>] <selectClause> )      -- sub-query
  | Environment.CurrentUser.<propertyOfAPP_UserDetail>
```

`Not In` and `Not Like` are written as two words.

### Literals

| Kind | Form | Notes |
|---|---|---|
| Null | `Null` | `= Null` / `<> Null` mean IS / IS NOT NULL |
| Boolean | `True`, `False` | |
| String | `'text'` or `"text"` | double a single quote inside a single-quoted string |
| Number | `42`, `10.5`, `-3` | decimal point, never a thousands separator |
| Date | `#2026-01-01#` | `YYYY-MM-DD` only |
| Time | `#08:30:00#` | `HH:MM:SS`, 24-hour |
| Date + time | `#2026-01-01 08:30:00#` | `YYYY-MM-DD HH:MM:SS` |
| GUID | `{3F860049-3A74-4D54-9ACA-80B2BC0152FC}` | registry braces form |

### Null semantics

SQL three-valued logic. Verified against a live tenant on a nullable Boolean whose value was Null:

| Filter | Matches a Null row |
|---|---|
| `X = True` | no |
| `X = False` | no |
| `X <> False` | **no** |
| `X = Null` | yes |
| `X = True Or X = Null` | yes |

So a nullable flag needs the explicit `Or … = Null` branch. `describe_entity` reports
`isRequired`; only a required property is safe to compare directly.

### Booleans need an explicit comparison

A bare Boolean member is not accepted as a condition. Verified against a live tenant:

| Expression | Result |
|---|---|
| `Where … And T.APP_Billable` (calculated member) | rejected |
| `Where … And T.APP_NoBilling` (stored member) | rejected |
| `:Iif(T.APP_Billable, T.APP_DurationInHours, 0)` | rejected (inside or outside an aggregate) |
| `Where … And T.APP_Billable = True` | works |
| `:Iif(T.APP_Billable = True, T.APP_DurationInHours, 0)` | works |
| `Sum(:Iif(T.APP_Billable = True, T.APP_DurationInHours, 0))` | works |

The server's error for the rejected forms is a generic "An error occurred invoking" with no hint,
so check for a bare Boolean first when a query that looks right fails.

### Arithmetic on date/time values

Subtracting two date+time values yields **days** as a decimal, Excel-style:
`#2010-01-02# - #2010-01-01#` = `1.0`, and `#2010-01-01 14:00:00# - #2010-01-01 12:00:00#` =
`0.0833…` (2/24). Multiply by 24 for hours. For timesheets prefer the shipped calculated property
`APP_DurationInHours` over doing this arithmetic.

`+` concatenates strings: `P.APP_Code + ': ' + P.APP_ProjectName`.

### Like

`%` matches 0..n characters, `_` matches exactly one. To match user text literally, escape the
wildcards (`[%]`, `[_]`, `[[]`) or use `=`.

---

## 3. Parameters

Pass values in the tool's `parameters` object and reference them as `@Name`:

```
query:      From P In APP_Project Where P.APP_Code = @Code Select Top(5) New With { P.APP_Code, P.APP_ProjectName }
parameters: { "@Code": "PROJ-001" }
```

Rules enforced by the server:

- every key must be `@` + a plain identifier (letters, digits, underscore; first char a letter);
- the identifier must not be a SQL/TCQL **reserved word**. `@From`, `@To`, `@Date`, `@User`,
  `@Count`, `@Min`, `@Max`, `@Key`, `@Value`, `@Start`, `@End`, `@Year`, `@Month`, `@Limit` are all
  rejected (the error names the reserved word). Pick a specific name instead: `@FromDate`,
  `@ToDate`, `@UserUuid`, `@MaxRows`, ...;
- values must be JSON **scalars** — string, number, boolean, or null. Arrays and objects are
  rejected, so `In (…)` cannot be filled from one list parameter;
- every number, whole or fractional, is bound as decimal (TCQL's only numeric type), so `10` and
  `10.0` are equivalent;
- strings that look like an ISO 8601 date/time (`2026-01-01`, `2026-01-01T08:00:00`) or a GUID are
  converted to those types and work in comparisons and projections alike.

Never build a query by string-concatenating user input: use a parameter for every user-supplied
value. Reject or quote-escape anything that came from the user before it touches query text.

---

## 4. Functions

Functions start with a colon. Argument counts and types depend on the function.

### Date and time

| Function | Notes |
|---|---|
| `:AddDays(dt, n)` / `:AddHours` / `:AddMinutes` / `:AddSeconds` / `:AddMonths` / `:AddYears` | |
| `:Date(dt)` | date part |
| `:Day(dt)` / `:Month(dt)` / `:Year(dt)` / `:Quarter(dt)` | |
| `:DayOfWeek(dt)` | 0 = Sunday, 1..6 = Monday..Saturday |
| `:FirstOfMonth(dt)` / `:LastOfMonth(dt)` | |
| `:LastOfInterval(dt, 'Day'\|'Month'\|'Quarter'\|'Year')` | last value of the interval |
| `:IsoWeek(dt)` | ISO 8601 week number |
| `:NewDate(year, month, day)` | |
| `:Now()` / `:Today()` | tenant-local |
| `:RoundDateTime(dt, minutes)` | rounds to the given minute step |
| `:FormatDate(dt, format)` | .NET date format string |
| `:FormatDateAsPeriod(dt)` | returns `YYYY/MM` |
| `:FormatDateCanonical(dt[, withTime])` | `2026-10-07` / `2026-10-07 00:00:00` |

### Strings and numbers

| Function | Notes |
|---|---|
| `:Substring(s, start, length)` | **start is 1-based** |
| `:Len(s)` | |
| `:Replace(s, find, replaceWith)` | |
| `:IsNullOrEmpty(s)` | |
| `:Soundex(s)` | server database only |
| `:Round(n, digits)` | |
| `:FormatNumber(n, format)` | .NET numeric format string |
| `:Iif(condition, trueValue, falseValue)` | usable in `Where` and `Select`, also inside `Sum(…)`; `condition` must be a comparison (`X = True`), not a bare Boolean |

### Records and model

| Function | Notes |
|---|---|
| `:ObjectUuid(alias)` | the record's real primary key — use this in projections |
| `:DisplayValue(alias)` | the entity's display property value |
| `:FriendlyName('Entity.Property')` | label of a property |
| `:DefaultDisplayPropertyFriendlyName('Entity')` | |
| `:BackreferenceFriendlyName('Entity.Relation')` | e.g. `'APP_Timesheet.APP_Project'` → `Timesheets` |
| `:Translate(genericKey, fallback)` | localized string |
| `:HasPermission(alias, 'Read'\|'Insert'\|'Update'\|'Delete'\|'Write')` | |
| `:GetDataContextType()` / `:GetIsInSystemMode()` | diagnostics |

### Working time and absences

Only available in TCQL queries (not in calculated properties or in-memory expressions):

| Function | Parameters |
|---|---|
| `:PlannedHoursOfWork(userUuid, beginTime, endTime, includeLumpSumOvertime)` | both bounds **inclusive** |
| `:ActualHoursOfWork(userUuid, beginTime, endTime, includeWeights)` | both bounds **inclusive** |
| `:Overtime(userUuid, effectiveDate, includeWeights, includeLumpSumOvertime)` | |
| `:RemainingVacationWeeks(userUuid, effectiveDate)` | result in **weeks** |
| `:AverageHoursOfWorkPerDay(userUuid, effectiveDate)` | planned hours per working day |

Note the bound convention differs from a plain `Where` range: these functions take an inclusive end.
Weights come from the `APP_Timesheet.APP_WorkingTimeWeight` relation.

---

## 5. Aggregation and implicit grouping

`Sum(x)`, `Min(x)`, `Max(x)`, `Avg(x)`, `CountDistinct(x)`, `Count()` (no argument).

There is **no `Group By` clause**. Every projected expression that is not inside an aggregate
becomes a grouping key:

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

groups by user **and** period. Removing `.Period` collapses to one row per user.

Sorting by an aggregated value is not supported against the local data store — order by a grouped
expression, or sort the returned rows yourself when presenting them.

A scalar function may sit inside an aggregate. The common case is a conditional total:
`Sum(:Iif(T.APP_Billable = True, T.APP_DurationInHours, 0))` sums only billable hours without
adding a grouping level. The alternative — projecting `.IsBillable = T.APP_Billable` as a grouping
key — also works, but doubles the row count and leaves the pivoting to you.

---

## 6. Relations, back references and sub-queries

**Forward navigation** (n:1) uses the relation property name:
`T.APP_Project.APP_Customer.APP_CompanyName`. No join, no `Include` required — TCQL adds what the
`Where` / `Select` clauses reference.

**Back references** (1:n) use the relation's back-reference name, which is model metadata and is
**not** reported by `describe_entity`. Shipped back references verified against a live tenant:

| From | Back reference | Target |
|---|---|---|
| `APP_Project` | `P.Timesheets` | `APP_Timesheet` (via `APP_Timesheet.APP_Project`) |
| `APP_Project` | `P.Tasks` | `APP_Task` |
| `APP_Task` | `Ta.Timesheets` | `APP_Timesheet` |
| `APP_UserDetail` | `U.Timesheets` | `APP_Timesheet` |
| `APP_Customer` | `C.Projects` | `APP_Project` |
| `APP_Customer` | `C.Projects.Timesheets` | chained over two levels |

`P.Timesheets` and `C.Projects.Timesheets` follow `APP_Timesheet.APP_Project` only. time cockpit
itself attributes a booking with a task to the task's project, and the two can differ when a task
was moved to another project after booking. Use these back references for quick scans; for
figures that must match time cockpit, aggregate `APP_Timesheet` by the effective project instead
(query cookbook, "The effective project and customer of a booking").

Back-reference names carry **no** `APP_`/`USR_` prefix. For a tenant-defined relation, discover its
back-reference name with `:BackreferenceFriendlyName('<Entity>.<Relation>')`, or ask the tenant
administrator — do not guess.

**Sub-queries** are allowed in `Select` and `Where`, wrapped in parentheses, and return a single
scalar:

```
From P In APP_Project
Select Top(50) New With {
  P.APP_Code,
  .TotalHours = ( From T In P.Timesheets Select New With { .H = Sum(T.APP_DurationInHours) } )
}
```

```
From P In APP_Project
Where ( From T In P.Timesheets Select New With { .H = Sum(T.APP_DurationInHours) } ) > 5
Select Top(50) New With { P.APP_Code }
```

A sub-query aggregate over an empty relation yields **Null**, not 0 — a project with no bookings
comes back with `null`, so treat null as zero when you total the column, and say so if it matters.
Sub-queries cannot carry `Order By`, `Top` or `Include`.

---

## 7. Include

```
From T In APP_Timesheet.Include("APP_Project.APP_Customer") Select Top(20) T
```

Loads the related records with the source. Only worth it when you select the whole record; for a
projection, plain dotted navigation is cheaper. **Never use `.Include(*)`** — it expands every
relation recursively and will blow the response limit.

---

## 8. In Set and Environment

`In Set("<expressionName>"[, "<propertyName>"])` filters against a named set defined in the
tenant's model:

```
From C In APP_Customer Where C.APP_Code In Set("MyCustomers", "APP_Code") Select Top(50) New With { C.APP_CompanyName }
```

The set must already exist in the tenant; there is no MCP surface to list sets, so only use this
when the user or an administrator names one.

`Environment.CurrentUser.<property>` reads any property of `APP_UserDetail` for the acting user:

```
From T In APP_Timesheet
Where T.APP_UserDetail.APP_UserDetailUuid = Environment.CurrentUser.APP_UserDetailUuid
Select Top(50) New With { T.APP_BeginTime, T.APP_Description }
```

`Environment.CurrentUser.APP_UserDetailUuid` and `Environment.CurrentUser.APP_Username` are the
two you will normally need. This is preferable to hard-coding a UUID: it keeps the query correct
for whoever runs it.

---

## 9. Anonymous vs typed projections

| Form | Column names | `objectUuid` | `displayValue` |
|---|---|---|---|
| `Select T` | the entity's own scalar **and relation** property names (`APP_*`, `USR_*`); the primary key itself is not repeated as a member | the record's real primary key | the record's default display value |
| `Select New With { … }` | `USR_` + name with the ownership prefix stripped (`T.APP_Description` → `USR_Description`, `.Hours = …` → `USR_Hours`) | a **generated** per-row id, different on every run — not a record key | `null` or absent — an anonymous projection has no display property |
| `Select New APP_Project With { … }` | member names exactly as declared | typed result; rarely useful read-only | typed result; rarely useful read-only |

A relation member of a `Select T` row (e.g. `T.APP_Project`) is a reference object,
`{ "objectUuid": …, "displayValue": … }`, or `null` when the relation is empty — never a bare
UUID. Its `displayValue` is `null` or absent when the related record was not loaded (no
`.Include(...)` / navigation into it).

Consequences:

- You cannot control the `USR_` prefix. Pick alias names that read well after it is added, and map
  them to human headings when presenting.
- Writing an ownership prefix on an alias (`.APP_Total = …`) is a semantic error: *"You cannot use
  ownership prefixes in a select new with clause."*
- In a `.Alias = <identifier>` member, the identifier must be the source alias itself.
- The `objectUuid` of a `Select New With { … }` row is a generated per-row id, not the record's
  key; to hand the user something they can act on later, project the key explicitly:
  `.Id = :ObjectUuid(T)` or `T.APP_TimesheetUuid`.

---

## 10. Server-side guardrails

Applied by the MCP server before the query is dispatched. All of them fail closed.

| Guardrail | Behavior |
|---|---|
| Row cap | The outer `Top` is inserted when missing and lowered when larger than the configured cap (default **100** rows). `Top(0)` or a non-numeric `Top` is an error. |
| Comments | A query containing `//` or `/* */` is rejected outright — either form could hide the real `Select`/`Top` from the lexical scan. |
| System entities | Any `SYS_*` token, **including inside a string literal**, is rejected. So is any entity on the internal denylist (shipped telemetry/notification entities such as signal, notification, feature-flag and formatting-profile entities). |
| System members | `SYS_*` properties are stripped from returned rows and columns. |
| Lexical structure | Unbalanced parentheses or an unterminated string literal are rejected before execution. |
| Parameters | Names must be `@` + identifier and must not be a reserved word (`@From`, `@To`, `@Date`, ...); values must be JSON scalars. Numbers bind as decimal. |
| Response size | A response over the configured byte (~128 KB) or token (~8 000) budget is refused with *"exceeds the configured MCP response limit. Narrow the request and try again."* — distinct from the row cap. |
| Access mode | The tool is hidden and rejected when the session is restricted to own data (`Scope=OwnData`). Read-only mode does not affect it: TCQL is read-only by construction. |
| Permissions | The tenant's own model permissions still apply to every row and member. A query can legitimately return fewer rows than the data contains. |

Only `Select` queries are executable; TCQL has no INSERT/UPDATE/DELETE form. Mutations go through
`create_entity` / `update_entity` / `delete_entity` / `execute_action`.

---

## 11. Result envelope (`TabularResult`)

`Select T` (whole record) — columns are typed from the model, including the relation member:

```jsonc
{
  "columns": [
    { "name": "APP_Code", "label": "Code", "dataType": "String", "isCustom": false },
    { "name": "APP_Customer", "label": "Customer", "dataType": "Relation", "relatedEntity": "APP_Customer", "isCustom": false }
  ],
  "rows": [
    { "objectUuid": "…", "displayValue": "PROJ-001", "APP_Code": "PROJ-001", "APP_Customer": { "objectUuid": "…", "displayValue": "Demo Customer" } }
  ],
  "count": 1,
  "truncated": false,
  "warnings": [ "…" ]
}
```

`Select New With { … }` (anonymous projection) — columns are inferred from the returned values:

```jsonc
{
  "columns": [
    { "name": "USR_Project", "dataType": "String",  "isCustom": true },
    { "name": "USR_Hours",   "dataType": "Number",  "isCustom": true }
  ],
  "rows": [
    { "USR_Project": "PROJ-001", "USR_Hours": 12.5 }
  ],
  "count": 1,
  "truncated": false,
  "warnings": [ "…" ]
}
```

| Field | Meaning |
|---|---|
| `columns[].name` | property name exactly as it appears on every row object |
| `columns[].label` | human caption — present when the result entity is known (`Select T`), **absent** for `New With` results |
| `columns[].dataType` | `Unknown`, `String`, `Number`, `Integer`, `Boolean`, `DateTime`, `Date`, `Time`, `Duration`, `Uuid`, `Relation`, `Binary`. Typed from the model for `Select T`. For `New With` it is **inferred from the values**: an all-null column reports `Unknown`, a nested object reports `Relation`, and a string that parses as a date (e.g. `:FormatDateAsPeriod` → `"2026/09"`) reports `DateTime`. |
| `columns[].isCustom` | true for tenant (`USR_`) members. For `New With` projections every column name gets the `USR_` prefix, so it does *not* distinguish a real tenant extension from a renamed shipped property there. |
| `columns[].relatedEntity` | referenced entity for relation columns, typed or inferred |
| `rows[]` | open objects: `objectUuid` (a generated per-row id for `New With`), `displayValue` (`null` or absent for `New With`), plus whatever was projected. A relation member is `{ "objectUuid": …, "displayValue": … }` or `null`, never a bare UUID; its `displayValue` is `null` or absent when not loaded. |
| `count` | rows returned — not the number of matches |
| `truncated` | true when a row or payload limit cut the result. Exact for the row cap: the server reads one row past it, so a result that exactly fills the cap is reported complete, not truncated |
| `warnings[]` | non-fatal remarks; a warning names the row limit whenever the cap bit. A result the cap did not cut carries no truncation warning |

An empty result carries only the two stable columns, `objectUuid` and `displayValue` — the
rest of the shape is derived from the returned records.

Values: date/times are tenant-local wall clock with no offset; durations are decimal hours;
missing values are JSON `null`; blob properties should never be projected.
