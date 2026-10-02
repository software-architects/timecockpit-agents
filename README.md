# timecockpit-agents

Agent Skills and agent definitions for working with **[time cockpit](https://www.timecockpit.com)** through AI agents.

This package teaches an MCP-capable AI agent how to talk to a time cockpit tenant: how to explore a
customized data model, write correct TCQL, review and book time, run the tenant's own saved lists,
change records safely, and build defensible reports.

No two tenants share the same model. The shipped `APP_` entities, lists and actions are normally
present, but administrators add `USR_` entities, properties, relations, lists and actions, and may
rename captions or hide shipped artifacts. Every skill here therefore discovers names against the
live tenant before using them; see `timecockpit-model-discovery`.

Everything here is prompt material — Markdown skills and agent definitions plus one MCP server
declaration. There is no code to run and no service to host.

## Prerequisites

- A **time cockpit account**. The skills operate on your own tenant's live data; permissions are
  enforced by the server, not by the agent.
- An **MCP-capable AI agent**: Claude Code, VS Code with GitHub Copilot, OpenAI Codex, Cursor, or
  Claude Desktop.
- Access to a time cockpit MCP server — the hosted remote server, or the local Windows host that
  ships with the desktop administration tool.
- Optionally **Microsoft APM** (`apm`) to install and update this package, or **Node.js** to install
  the skills only with the [skills.sh](https://skills.sh) CLI (`npx skills`). Without either you can
  copy the `skills/` and `agents/` folders into your project by hand.
- Optionally the [Context7](https://context7.com) MCP server, which the
  `timecockpit-documentation-lookup` skill uses to search the product documentation. Without it the
  skill falls back to plain HTTP.

## Quick start

Install the whole package into the current project:

```bash
apm install software-architects/timecockpit-agents
```

Or declare it as a dependency in your own `apm.yml` so every checkout gets the same version:

```yaml
dependencies:
  apm:
    - software-architects/timecockpit-agents
```

You can also depend on a **single skill** through its virtual path, which is the better choice when
you only need one capability:

```yaml
dependencies:
  apm:
    - software-architects/timecockpit-agents/skills/timecockpit-tcql
    - software-architects/timecockpit-agents/skills/timecockpit-reporting
```

Then run `apm install` and restart your agent so it picks up the new skills, agents, and MCP entry.

## Alternative: skills only, with skills.sh

If you do not use apm, the [skills.sh](https://skills.sh) CLI installs the skills from `skills/`
into a wide range of agents, including Claude Code, GitHub Copilot, Codex, and Cursor. It needs Node.js
and nothing else; `npx` fetches the CLI on first use.

```bash
npx skills add software-architects/timecockpit-agents
```

The CLI asks which skills and which agents to install into, and whether to install into the current
project or globally. To pick a single skill up front:

```bash
npx skills add software-architects/timecockpit-agents --skill timecockpit-tcql
```

skills.sh installs **skills only**. The agents from `agents/` and the `timecockpit` MCP server entry
are not deployed, so configure the MCP server in your client yourself — the `timecockpit-mcp-setup`
skill walks you through it, and `skills/timecockpit-mcp-setup/references/client-configs.md` has the
configuration for each client. There is no version pinning either; each run installs the current
default branch.

## What apm deploys where

`apm` writes into the client directories your project targets:

| Content | Deployed to |
| --- | --- |
| Skills (`skills/<name>/SKILL.md` plus `references/`) | `.agents/skills/`, `.claude/skills/` |
| Agents (`agents/*.agent.md`) | `.github/agents/`, `.claude/agents/`, `.codex/agents/` |
| The `timecockpit` remote MCP server entry | per target: `.mcp.json` (Claude Code), `.github/mcp.json` (Copilot CLI), `.vscode/mcp.json` (VS Code), `.codex/config.toml` (Codex) |

The MCP entry is added only after you accept the trust prompt — `apm` asks before it writes a server
declaration into your local client configuration. The entry carries no OAuth client id: that id is
user-specific and comes from your tenant administrator or time cockpit support, so add it to the
generated entry yourself (see the `timecockpit-mcp-setup` skill). Skills are loaded by name, so an agent picks the right
one from its description; you can also name one explicitly ("use the timecockpit-tcql skill").

## Skills

| Skill | What it is for |
| --- | --- |
| `timecockpit-mcp-setup` | Connect an AI client to the time cockpit MCP server (remote or local), choose the tenant, sandbox, read-only and own-data settings, and troubleshoot sign-in problems or missing tools. Start here. |
| `timecockpit-model-discovery` | Explore the tenant's customized data model — entities, properties, relations, and what the administrator added — and turn it into technical names that actually exist. |
| `timecockpit-tcql` | Write and run read-only TCQL queries and interpret the tabular result, including grouping, parameters, row caps, and the common failure modes. |
| `timecockpit-time-tracking` | Review, summarize, and book a user's own working time: daily and weekly summaries, gaps, overlaps, duplicates, and corrections. |
| `timecockpit-reporting` | Answer analytical questions — hours per project, customer, user or period, budget versus actual, utilization, missing bookings, absences — with the read-only tools. |
| `timecockpit-lists-and-actions` | Discover and run the tenant's named lists (saved views). Model actions are not available over MCP; they are run in the time cockpit UI. The skill keeps its historical name. |
| `timecockpit-record-management` | Create, update, and delete records through the generic object tools (`create_object`, `update_object`, `delete_object`), with the discovery step, the confirmation handshake, and a verification re-read. |
| `timecockpit-list-deeplinks` | Build links that open a time cockpit web list with its filters pre-filled, for example from a report row to the bookings behind it. |
| `timecockpit-documentation-lookup` | Look up the official product documentation (docs.timecockpit.com) through Context7 (`/websites/timecockpit`) or its `llms.txt` index, instead of answering product questions from memory. |

The administrative scripting skill `time-cockpit-local-mcp` (IronPython migrations and model
introspection through the local host, Windows only) and the internal customization, documentation and
content skills live in the team-internal `software-architects/tc-agents-internal` package.

## Agents

| Agent | For whom | What it does |
| --- | --- | --- |
| `timecockpit-time-assistant` | Everyone who books time | Reviews and summarizes your own hours, finds gaps and overlaps, and books entries — always confirming before it writes. Never touches other people's data. |
| `timecockpit-reporting-analyst` | Controllers, project leads | Answers hours, budget, utilization, billing, and absence questions read-only, through named lists and aggregating queries, always stating the data window and scope. |
| `timecockpit-tenant-admin` | Tenant administrators | Discovers the customized model, runs named lists, and performs careful generic create, update, and delete operations one change at a time, verifying afterwards. |

Agents are entry points, not replacements for the skills: each one loads the skills it needs.

## MCP server at a glance

**Availability:** the time cockpit MCP server is currently available to selected customers only.
If your tenant is not enabled for it, ask your tenant administrator or time cockpit support at
<support@timecockpit.com>.

Model actions are not exposed through MCP on either host; run them in the time cockpit UI.

**Remote (hosted).** `https://mcp.timecockpit.com`, HTTP transport, sign-in with your Microsoft
Entra account. Per-session settings are passed either as a request header or as a route segment, one
or the other, never both:

| Setting | Header | Route | Effect |
| --- | --- | --- | --- |
| Tenant | `X-tc-tenant-id` | `/tcTenantId/{id}` | Needed only when your Entra tenant maps to more than one time cockpit tenant. |
| Sandbox | `X-sandbox-environment` | `/sandboxEnvironment/{value}` | `test` targets the sandbox; empty means production. |
| Read-only | `X-access` | `/access/{value}` | `readonly` hides and rejects every mutating tool. |
| Own data | `X-scope` | `/scope/{value}` | `owndata` hides the tools that cannot be constrained to you, and forces the current-user filter on timesheet reads and writes. |

**Local (Windows).** `OnCockpit.Admin.exe --mcp -c <configuration>` (path in the
`OnCockpitAdminBinary` environment variable) starts a stdio host against the
selected desktop configuration, with `--mcp-access`, `--mcp-scope`, and `--mcp-confirmation` as
overrides. The local host additionally exposes model resources and the trusted IronPython tools;
tenant, sandbox, and profile routing come from the chosen configuration.

For the actual client configuration files, the sign-in flow, and troubleshooting, use the
`timecockpit-mcp-setup` skill.

## Safety notes

- **Mutations are confirmation gated.** Generic object create, update, and delete calls
  return `confirmation_required` on the first call and only run when the caller re-invokes with
  `confirmed=true`. Confirmation is a deployment setting: it cannot be set per request, and an
  operator may disable it for a disposable sandbox. The skills confirm with the user regardless.
- **Read-only and own-data sessions really are restricted.** Restricted tools are removed from tool
  discovery *and* rejected if called. A missing tool is a configuration fact, not a bug.
- **Row caps, not paging.** Record-returning tools are capped server-side (100 rows by default, 50
  for timesheet reads) and there is no paging cursor. Results carry a `truncated` flag; the skills
  aggregate or narrow instead of stitching pages, and never total a truncated result.
- **No system data.** `SYS_` and internal entities are invisible on every MCP surface, and
  referencing one is rejected. There is no metadata back door.
- **Deletes have no undo.** One record per call, one confirmation per record, and a
  verification read afterwards.

## Versioning

The package carries a semantic version in `apm.yml`, and each release is tagged `v<version>` in git.
Pin a version when you need reproducible behaviour:

```yaml
dependencies:
  apm:
    - software-architects/timecockpit-agents#v0.1.0
```

Without a pin you track the default branch and get updates on the next `apm install`.

## Contributing

- One skill per task. A skill answers "how do I do X", not "everything about Y".
- Skills follow the Agent Skills specification: a folder under `skills/` containing `SKILL.md` with
  YAML frontmatter (`name`, `description`, optionally `compatibility` and `metadata`).
- The `description` is what makes a skill trigger. Say what it does *and* when to use it, in the
  user's vocabulary, and say what it is not for.
- Keep `SKILL.md` short and operational — the rules an agent needs while working. Put grammar
  references, cookbooks, and long tables in `references/` and link to them from `SKILL.md`.
- Agents live in `agents/<name>.agent.md` with `name` and `description` in the frontmatter and the
  system prompt in the body. Kebab-case filenames. An agent should point at skills rather than
  duplicate them.
- Use bare MCP tool names in prose, and note that clients add a server prefix.
- Verify names against a real tenant before writing them down. Never invent an entity, property,
  list, or action name, and never include tenant ids, credentials, or internal URLs.

## Support

Product website: <https://www.timecockpit.com>

Documentation and contact: <https://docs.timecockpit.com>
