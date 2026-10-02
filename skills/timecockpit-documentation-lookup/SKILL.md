---
name: timecockpit-documentation-lookup
description: Looks up the official time cockpit product documentation (docs.timecockpit.com) through Context7 or the site's llms.txt index, to answer how a feature works, what a shipped list, action, entity or setting does, how TCQL, scripting or the Web API behave, or where something is in the UI. Use when a question is about the product rather than about this tenant's data, or when an MCP result needs the documented meaning of a shipped artifact. Not a source for tenant-specific names or data (see timecockpit-model-discovery).
compatibility: Any agent. Uses the Context7 MCP server when the client has it, otherwise plain HTTP fetches of context7.com or docs.timecockpit.com.
metadata:
  version: "1.0"
---

# time cockpit documentation lookup

The product documentation is published at <https://docs.timecockpit.com> and indexed on Context7
as **`/websites/timecockpit`** (declared by the site's own `context7.json`). Look things up there
instead of answering product questions from memory. Names, defaults and UI paths change between
releases, and the documentation is the reference for how the product is meant to behave.

## What the documentation is for, and what it is not

- **Use it for product behavior.** That covers features and workflows, the UI labels and navigation
  paths the user sees, shipped `APP_` entities, lists and actions, TCQL syntax, Python scripting,
  the Web API/OData, configuration, permissions and roles, and the release notes.
- **Do not use it for tenant facts.** The documentation describes the shipped model. A tenant adds
  `USR_` artifacts, renames captions and hides shipped ones. Confirm every technical name against
  the live tenant (`timecockpit-model-discovery`) before you use it in a tool call.
- If the documentation and the live tenant disagree, the tenant is right about its own data. Tell
  the user about the difference instead of hiding it.
- The MCP tool contract is described by the tools themselves and by the other `timecockpit-*`
  skills. Where the documentation describes TCQL or the Web API in general, the MCP guardrails in
  `timecockpit-tcql` still apply.

## 1. Context7 (preferred)

With the Context7 MCP server connected, skip `resolve-library-id` because the ID is already known.
Call the docs tool directly:

```
query-docs { "libraryId": "/websites/timecockpit", "query": "<question in the user's words>" }
```

Older Context7 servers call the tool `get-library-docs` and take `context7CompatibleLibraryID`
plus a `topic`. Use whichever one the client lists. Put a concrete question in the query ("how do
I close a project", "TCQL Iif function") rather than a single word.

Without the MCP server, the same index answers over plain HTTP:

```
https://context7.com/api/v1/websites/timecockpit?type=txt&topic=<url-encoded topic>&tokens=3000
```

Each snippet carries a `Source:` URL on docs.timecockpit.com. Cite that URL when you answer.

## 2. Direct fallback: llms.txt

If Context7 is unreachable or returns nothing relevant, read the site itself:

- <https://docs.timecockpit.com/llms.txt>: one line per page with its purpose and link. Find the
  page there, then fetch it. Links point to the raw Markdown, and the rendered page has the same
  URL ending in `.html` instead of `.md`.
- <https://docs.timecockpit.com/llms-full.txt>: the entire documentation as one file. It is large,
  so fetch it only when you cannot find the page through the index.

## 3. Answering

- Link the page you used, and use its `.html` URL when you link it for a human reader.
- Quote UI labels exactly as the page writes them, and say which language the labels are in.
- Say "the documentation does not cover this" when it does not. Never fill the gap with a guessed
  label, path or default.
- For "since when" or "what changed" questions, check the monthly release notes
  (`doc/release-notes/<year>-<month>`).

## Related skills

`timecockpit-model-discovery` (the tenant's actual model), `timecockpit-tcql` (query syntax as the
MCP server accepts it), `timecockpit-lists-and-actions` (running lists),
`timecockpit-list-deeplinks` (links into the web app), `timecockpit-mcp-setup` (connecting a client).
