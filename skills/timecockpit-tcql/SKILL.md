---
name: timecockpit-tcql
description: Writes and runs read-only TCQL (Time Cockpit Query Language) queries through the MCP tool execute_tcql_query and interprets the TabularResult it returns. Use for ad-hoc data questions, reporting and totals across customers, projects, tasks, users or periods, for cross-entity lookups and resolving a record UUID from a code or name, and whenever a TCQL query was rejected, returned no rows, or came back with unexpected column names.
compatibility: Requires a connected time cockpit MCP server (remote or local stdio host) exposing execute_tcql_query, get_entities and describe_entity. execute_tcql_query is hidden and rejected when the session is restricted to own data (Scope=OwnData) — use get_timesheets there instead. Tool names may carry a client prefix.
metadata:
  version: "1.0"
---

# Time Cockpit TCQL

TCQL is time cockpit's query language: LINQ-shaped, read-only, executed server-side against the
tenant's own data model. `execute_tcql_query` is the **escape hatch** for questions no typed tool
answers. Tool names are bare here; most clients prefix them (`mcp__time_cockpit__execute_tcql_query`).

Related skills: `timecockpit-time-tracking` (own hours, booking), `timecockpit-record-management`
(create/update/delete), `timecockpit-lists-and-actions` (named lists, model actions).

## 1. Pick the cheapest tool that answers the question

| Question | Use |
|---|---|
| My / a user's own timesheets in a period | `get_timesheets` — typed filters, no TCQL needed |
| A report the tenant already defined | `get_lists` → `describe_list` → `execute_list` |
| What properties does an entity have | `describe_entity` — **not** a probe query |
| Who am I | `get_current_user` |
| Anything else: joins, grouping, totals, cross-entity filters, UUID lookup | `execute_tcql_query` |

## 2. Workflow — never guess a name

1. **Discover the shape.** `get_entities` (optionally `nameContains`) then
   `describe_entity('APP_Timesheet')`. Read `primaryKeyName`, `displayPropertyName`, and each
   property's `isRelation`, `relatedEntity`, `isCalculated`, `isBlob`, `isCustom`, `isRequired`.
   Tenants add `USR_` properties and whole `USR_` entities; a property that exists in one tenant
   may not exist in another. If you did not read it from metadata, do not write it into a query.
2. **Write one narrow query.** Always a `Where` clause that bounds the result by date, user,
   project, customer or another business scope, and always an explicit `Select Top(N)` with the
   smallest N that answers the question.
3. **Project only the columns you will actually show.** `Select T` returns every scalar property
   of the entity — a lot of tokens per row, and it silently drops the related values you probably
   wanted anyway.
4. **Run it, then read the envelope**: `count`, `truncated`, `warnings`, `columns`.
5. **If truncated, do not page.** Narrow the filter or aggregate instead (see §7).

## 3. Query skeleton

```
From <Alias> In <Entity>[.Include("<Relation>")]
[Where <condition>]
[Order By <expr> [Asc|Desc] [, <expr> [Asc|Desc]]]
Select [Top(N)] <Alias> | New With { <members> }
```

Clause order is fixed: `From` → `Where` → `Order By` → `Select`. `Top(N)` goes **after** the
`Select` keyword, before the alias or the `New With { … }` block.

```
From T In APP_Timesheet
Where T.APP_BeginTime >= #2026-09-01# And T.APP_BeginTime < #2026-10-01#
Order By T.APP_BeginTime Desc
Select Top(50) New With {
  T.APP_BeginTime, T.APP_EndTime, T.APP_Description, T.APP_DurationInHours,
  .Project = T.APP_Project.APP_Code,
  .Id      = :ObjectUuid(T)
}
```

Relations need no join: navigate with dots (`T.APP_Project.APP_Customer.APP_CompanyName`) and
TCQL includes what it needs. `.Include(…)` is optional and only worth it when you select the whole
related record.

## 4. Six rules that break queries

1. **Case sensitivity.** Keywords are exact: `From`, `In`, `Where`, `Order By`, `Select`, `Top`,
   `New With`, `And`, `Or`, `Not`, `Null`, `True`, `False`, `Like`, `Asc`, `Desc`. The range
   variable must start with a capital letter — `From T In …`, never `From t In …`.
2. **`New With { }` renames every column.** The ownership prefix is stripped and `USR_` is added:
   `T.APP_Description` comes back as **`USR_Description`**, `.Project = …` as **`USR_Project`**.
   Never write an `APP_` prefix on an alias (`.APP_Foo = …` is a semantic error). Read the response's
   `columns` to learn the real names before you build a table from the rows.
3. **`Top(N)` is enforced, not suggested.** The server inserts `Top(<cap>)` when you omit it and
   silently lowers a larger one to the cap (default 100 rows). Ask for a total with an aggregate,
   not a bigger `Top`.
4. **Comments are rejected.** A query containing `//` or `/* */` fails outright. Keep the query
   text pure TCQL and put your explanation in the chat message.
5. **`SYS_*` and internal entities are invisible.** Referencing one — even inside a string literal
   — is rejected. There is no system-metadata query path through MCP; `get_entities` and
   `describe_entity` are the only model discovery surface.
6. **Parameters, never interpolation.** Put user-supplied values in the `parameters` object as
   `@Name` and reference `@Name` in the query. Names must match `@` + identifier, values must be
   JSON scalars (no arrays, no objects).

## 5. Filtering

- **Literals:** date `#2026-01-01#`, date+time `#2026-01-01 08:30:00#`, time `#08:30:00#` (these
  three formats only), string `'text'` or `"text"` (double a single quote to escape it), `True`,
  `False`, `Null`, GUID `{3F860049-3A74-4D54-9ACA-80B2BC0152FC}`, plain numbers.
- **Operators:** `= <> < <= > >=`, `And`, `Or`, `Not`, `+ - * / %`, `Like` / `Not Like`
  (`%` = any, `_` = one character), `In (a, b, c)` / `Not In (…)`.
- **Null is SQL null.** `= Null` and `<> Null` mean IS / IS NOT NULL. Every other comparison
  against Null matches nothing — so on a nullable Boolean, `Where U.APP_IsEnabled = True` **and**
  `Where U.APP_IsEnabled <> False` both drop the Null rows. Write
  `Where U.APP_IsEnabled = True Or U.APP_IsEnabled = Null`, or filter on a required property
  instead. Check `isRequired` in `describe_entity` before comparing a Boolean.
- **Date ranges: begin inclusive, end exclusive.** September 2026 is
  `T.APP_BeginTime >= #2026-09-01# And T.APP_BeginTime < #2026-10-01#`. Both bounds test
  `APP_BeginTime` only, so an entry that starts before the range but ends inside it is not returned.
- **The acting user:** `Where T.APP_UserDetail.APP_UserDetailUuid = Environment.CurrentUser.APP_UserDetailUuid`.
  `Environment.CurrentUser.<property>` reads any property of `APP_UserDetail`.
- **Functions** start with a colon: `:Iif(cond, a, b)`, `:Today()`, `:Now()`,
  `:Substring(s, 1, 3)` (1-based), `:Date(x)`, `:Year(x)`, `:Month(x)`, `:IsoWeek(x)`,
  `:FormatDateAsPeriod(x)`, `:ObjectUuid(alias)`, `:DisplayValue(alias)`, and more — full list in
  the reference file.

## 6. Aggregation — there is no Group By

`Sum(…)`, `Min(…)`, `Max(…)`, `Avg(…)`, `CountDistinct(…)`, `Count()`. Grouping is **implicit**:
every projected expression that is *not* inside an aggregate becomes a grouping key.

```
From T In APP_Timesheet
Where T.APP_BeginTime >= #2026-09-01# And T.APP_BeginTime < #2026-10-01#
Select Top(100) New With {
  .Project  = T.APP_Project.APP_Code,
  .Hours    = Sum(T.APP_DurationInHours),
  .Bookings = Count()
}
```

Add one non-aggregated member and you add a grouping level — that is the usual cause of "why did
I get 400 rows instead of 12". Order by a grouped expression, never by an aggregate. An aggregate
in a sub-query over an empty relation returns **Null**, not 0.

## 7. Aggregate instead of paging

`Skip` / `Take` do not work through `execute_tcql_query` — the enforced `Top` clause conflicts with
them. There is no cursor either. When a result is truncated:

- tighten the period, or split it into several smaller queries whose results you add up;
- or replace the row-level query with an aggregate that returns one row per group;
- or use `Count()` first to find out how big the answer is before asking for rows.

Never report a total computed from a `truncated: true` result.

## 8. Reading the `TabularResult`

```jsonc
{ "columns": [ { "name": "USR_Hours", "dataType": "Number", "isCustom": true } ],
  "rows":    [ { "objectUuid": "…", "displayValue": "…", "USR_Hours": 12.5 } ],
  "count": 1, "truncated": false, "warnings": [ "…" ] }
```

- `columns` describes **this** response: `name` (exactly as it appears on every row), `dataType`,
  `isCustom` (true for `USR_` names — which for `New With` projections means *all* of them, not
  just tenant extensions). For TCQL results `dataType` is inferred from the returned values, so an
  all-null column reports `Unknown` and a string that happens to parse as a date reports
  `DateTime`. Do not treat it as model metadata; `describe_entity` is the authority.
- Zero rows means **`columns` is empty too**. That is "no matching data", not a broken query.
- `objectUuid` is the real record key only for `Select <Alias>`. For a `New With { }` projection it
  is a generated per-row id that changes between identical runs — useless as a key. Project the
  primary key explicitly (`:ObjectUuid(T)`, or `T.APP_TimesheetUuid`) whenever the user may want to
  act on a row afterwards.
- `displayValue` is likewise the record's display value only for `Select <Alias>`; for a projection
  it is derived and can even echo the first projected member. Treat it as informational.
- `count` is the number of rows returned, **not** the number of matches. `truncated: true` plus a
  warning naming the row limit means data was left behind; a warning saying the total is unknown
  means the result exactly filled the cap and may or may not be complete.
- Date/times are tenant-local wall clock with no offset (`2026-09-10T22:53:35`). Do not convert
  them, do not append `Z`. Durations are decimal hours.

**Presenting results:** render a markdown table with human column headings (drop the `USR_`
prefix in the heading, keep the real name if the user might re-query), state the filter that
produced the numbers, give totals with one or two decimals, and say the row count. If `truncated`
is true, say so before the numbers.

## 9. When a query fails

Errors are deliberately terse and never echo the query. Walk this list:

1. A rejected keyword or prefix — comment (`//`, `/* */`), `SYS_` token, `Skip`/`Take`,
   an `APP_`-prefixed alias in `New With`, unbalanced parentheses or quotes.
2. A property or entity that does not exist in **this** tenant — re-run `describe_entity` for both
   the source entity and every navigated relation. (`APP_Department` has only `APP_Code` and
   `APP_Description`; `APP_Department.APP_Fullname` fails.)
3. A parameter typing problem. Strings, booleans and decimals are reliable. If a comparison
   against a `@parameter` fails, fall back to a literal in the query text: a date range is safe as
   `#2026-09-01#` because you construct it, and write numbers with a decimal point (`10.0`).
4. Response too large: "exceeds the configured MCP response limit" is a byte/token limit, not the
   row cap. Project fewer columns (never a blob property — `isBlob` in `describe_entity`) or fewer
   rows.
5. Tool missing entirely: the session is restricted to own data. Use `get_timesheets`.

## 10. Reference files

- **[references/tcql-reference.md](references/tcql-reference.md)** — full grammar, every operator,
  the complete documented function list, sub-queries, back-reference navigation, `In Set`,
  `Environment`, `.Include(…)`, and the exact server-side guardrails. Open it when you need a
  function signature or a construct not shown above.
- **[references/query-cookbook.md](references/query-cookbook.md)** — ready-made queries (hours per
  project or user, billable revenue, projects of a customer, open tasks, UUID lookup, budget
  burn-down) with the shipped property names verified against a live tenant. Open it before
  composing a reporting query from scratch.
