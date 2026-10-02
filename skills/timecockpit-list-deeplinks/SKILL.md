---
name: timecockpit-list-deeplinks
description: Builds URLs that open a time cockpit web list with its filters pre-filled (project, user, date range and other list parameters), so a report row, chart bar or answer can link straight to the bookings or records behind it. Use when the user wants a link, wants to "open this in time cockpit", or when a report should be clickable. Not for running a list and reading its data (see timecockpit-lists-and-actions).
compatibility: Any agent. Discovering list parameters and resolving UUIDs needs a time cockpit MCP server with get_lists and execute_tcql_query; the link itself is plain string building.
metadata:
  version: "1.0"
---

# time cockpit list deeplinks

Build a URL that opens a time cockpit web list with its filters pre-filled. A typical use is linking
each bar or row of a report back to the bookings behind it. The same format is what the web app's
**Share List** action copies (release notes, September 2026). A link copied from there is a good
reference when you are unsure about a list.

## URL format

```
https://web.timecockpit.com/app/lists/<ListName>?parameters=<encoded>
```

- `<ListName>` is the technical list name from `get_lists`, for example `APP_DefaultTimesheetList`.
- `<encoded>` = `encodeURIComponent("Name1=value1&Name2=value2")`. Encode the joined string exactly
  once and do not encode the values a second time (see *Text values* below).
- `Name` is the list parameter without `@` (`@Project` → `Project`).
- `Name=` with an empty value sets the filter to null. Use this to clear a default, for example
  `UserDetail=` so the list shows all users instead of only the current one.
- Unknown names are ignored silently, so a typo only shows up as a missing filter.
- **Base URL.** It is `https://web.timecockpit.com` unless the user works on another time cockpit
  web address; then use that host. The tenant comes from the user's login, never from the URL, so
  a link works for every colleague who can see the list and shows them their own permitted data.

## Value formats

| Type | Format | Example |
| --- | --- | --- |
| Guid / relation | plain UUID | `79b82123-596e-4abe-bdf2-38a8c279f589` |
| Date / DateTime | `datetime(<UTC ISO>Z)` | `datetime(2025-12-31T23:00:00Z)` |
| Boolean | `true` / `false` | `IncludeClosed=true` |
| Text | plain | `Code=ALPHA` |

**Date filters must be local midnight in the tenant's time zone, converted to UTC.** A Date property
rejects any other time. For Europe/Vienna, 2026-01-01 is `2025-12-31T23:00:00Z` (UTC+1) and
2026-04-01 is `2026-03-31T22:00:00Z` (UTC+2). Compute the offset for each date and never hardcode
it, because daylight saving time changes it within a year. If you do not know the tenant's time
zone, ask the user.

**Text values.** A text value must not contain `&`, `=` or `%`. These characters would need their
own value-level encoding, and not every deployed web client decodes that. Prefer a Guid parameter,
or leave that filter out and tell the user.

## Steps

1. `get_lists(filter: '<topic>')`: pick the list and read its `parameters` (name, type,
   mandatory). Prefer a list whose relation filter is a `Guid`, because `Text` filters match codes,
   which are not unique across customers.
2. Resolve every relation value to its UUID with TCQL (for example `APP_Project.APP_ProjectUuid`).
   Never put a code or display name into a Guid parameter.
3. Convert each date to tenant-local midnight in UTC.
4. Join the pairs, encode once, and append the result as `?parameters=`.

```js
const midnightUtc = (y, m, d, tz) => {
  const u = Date.UTC(y, m - 1, d);
  const p = new Intl.DateTimeFormat('en-US', { timeZone: tz, hourCycle: 'h23', year: 'numeric',
    month: '2-digit', day: '2-digit', hour: '2-digit', minute: '2-digit', second: '2-digit' })
    .formatToParts(new Date(u));
  const g = (t) => +p.find((x) => x.type === t).value;
  const off = Date.UTC(g('year'), g('month') - 1, g('day'), g('hour'), g('minute'), g('second')) - u;
  return `datetime(${new Date(u - off).toISOString().slice(0, 19)}Z)`;
};
const link = (list, params, base = 'https://web.timecockpit.com') =>
  `${base}/app/lists/${list}?parameters=` +
  encodeURIComponent(Object.entries(params).map(([k, v]) => `${k}=${v ?? ''}`).join('&'));

link('APP_DefaultTimesheetList', {
  UserDetail: null,
  Project: '79b82123-596e-4abe-bdf2-38a8c279f589',
  BeginTime: midnightUtc(2026, 1, 1, 'Europe/Vienna'),
  EndTime: midnightUtc(2026, 12, 31, 'Europe/Vienna'),
});
```

## Caveats

- **Attribution differs.** A list's project filter usually matches the booking's own project, while
  reports count a booking for its task's project (see `timecockpit-tcql`, effective project).
  The list total can differ from the report figure when tasks were moved. Say so next to the link
  when it matters.
- **Last day unverified.** Whether `EndTime` includes the last day depends on the list definition.
  Check it once per list, for example by comparing the list total with your query.
- **Mandatory filters.** Mandatory Boolean filters (for example `IncludeClosed`) keep the list's
  default unless you set them.
- **Relation filters with a separate display value.** Some relation filters ignore a value passed
  in the URL. If a relation parameter has no effect, tell the user the filter has to be set by hand
  instead of trying variants.
- **Test before you hand it out.** When you can, open one generated link or ask the user to check
  one. A silently ignored parameter is only visible in the opened list.

## Related skills

`timecockpit-lists-and-actions` (list discovery and parameters), `timecockpit-tcql` (resolving
UUIDs), `timecockpit-reporting` (the figures a link typically backs up),
`timecockpit-documentation-lookup` (what a shipped list shows).
