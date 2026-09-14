---
name: timecockpit-mcp-setup
description: Connects an AI client (Claude Code, Claude Desktop, VS Code Copilot, Copilot CLI, Codex CLI, Cursor) to the time cockpit MCP server at mcp.timecockpit.com, or to the local stdio host started with OnCockpit.Admin.exe --mcp, and selects the right tenant, sandbox, read-only and own-data settings. Use when setting up, changing or sharing an MCP connection, when sign-in fails ("does not support dynamic client registration", 401, 403, consent), when expected tools are missing from the tool list, when the agent talks to the wrong tenant or to the test sandbox instead of production, and when installing the timecockpit-agents skills with apm.
compatibility: The remote host needs a Microsoft Entra ID work account with time cockpit access and an MCP client that speaks HTTP transport with a pre-registered OAuth client id (Entra does not support dynamic client registration). The local stdio host needs an installed time cockpit desktop client and is Windows only.
metadata:
  version: "1.0"
---

# time cockpit MCP setup

Getting an AI agent onto a time cockpit tenant is three decisions and one verification sequence:
**which host**, **which tenant/sandbox**, **how much power the agent gets**. Everything else is
client-specific JSON, which lives in `references/client-configs.md`.

Tool names in these skills are bare (`ping`, `get_timesheets`). Most clients prefix them at
runtime — `mcp__timecockpit__ping`, `mcp_timecockpit_ping`. Same tools.

## 1. Remote host or local host?

| | Remote HTTP host | Local stdio host |
|---|---|---|
| Endpoint | `https://mcp.timecockpit.com` (preview: `https://mcp-preview.timecockpit.com`) | `OnCockpit.Admin.exe --mcp -c <configuration>` |
| Sign-in | Microsoft Entra ID work account, browser flow | already signed in through the desktop client |
| Platform | any client, any OS | Windows only |
| Tenant choice | per request, header or URL | fixed by the desktop configuration |
| Extra tools | `entra_whoami` | `execute_script_literal`, `execute_script_file`, `get_resource`, model resources |
| Use it for | everyday work: time tracking, reporting, record management | administrative work that needs scripting against the model |

Default to the remote host. Reach for the local host only for the trusted administrative work
described in section 8.

## 2. Decide access and scope before you connect

These are the two settings that matter, because they are part of the connection and cannot be
tightened later in the conversation.

| Who is connecting | Access | Scope | Effect |
|---|---|---|---|
| Someone wiring up a personal assistant for the first time | `readonly` | `owndata` | Can only look, and only at their own data. Safest starting point. |
| Someone who books and corrects their own time | default | `owndata` | `get_timesheets` and `create_timesheet` are forced to the acting user. |
| A tenant administrator doing reporting or maintenance | default | default | Full tool surface, still bounded by that user's time cockpit permissions. |
| Anyone exploring or demoing on live data | `readonly` | default | Read-only across everything the user may see. |

`readonly` and `owndata` only ever *narrow* a connection, which is why callers are allowed to set
them at all. Neither replaces time cockpit's own permissions: an agent can never exceed what the
signed-in user is allowed to do.

## 3. The five settings

Each is optional, and each may be supplied **either** as a header **or** as a URL route segment —
never both for the same setting, or the request is rejected with `404` and a message naming it.
Use headers where the client lets you set them, route segments where it does not (Claude Desktop,
for example).

| Setting | Header | Route segment | Values | When you need it |
|---|---|---|---|---|
| Tenant | `X-tc-tenant-id` | `/tcTenantId/{id}` | tenant id (GUID) | Only when your Entra tenant maps to more than one time cockpit tenant. Otherwise omit. |
| Sandbox | `X-sandbox-environment` | `/sandboxEnvironment/test` | `test` | To work against the test sandbox. Omitted means **production**. |
| Profile | `X-profile` | `/profile/{profile}` | profile name | Only if your tenant uses named profiles and you were told to set one. |
| Access | `X-access` | `/access/readonly` | `readonly`, `default` | To hide every mutating tool. |
| Scope | `X-scope` | `/scope/owndata` | `owndata`, `default` | To restrict the connection to the acting user's own data. |

Values are case-insensitive (`readonly` = `ReadOnly`). There is no `readwrite` value — omit the
setting, or say `default`. Route segments are key/value pairs in any order:
`https://mcp.timecockpit.com/access/readonly/scope/owndata`.

**Confirmation is not yours to set.** Whether mutating tools demand an explicit `confirmed=true`
is fixed by the operator: deployment configuration remotely, the `--mcp-confirmation` launch
argument locally. A request that tries to send `X-confirmation` or a `/confirmation/...` segment is
rejected outright. When a mutation answers `status: confirmation_required`, that is the normal
handshake — describe what will change, then re-issue the same call with `confirmed=true`.

## 4. Claude Code, remote, minimal

Project `.mcp.json`:

```json
{
  "mcpServers": {
    "timecockpit": {
      "type": "http",
      "url": "https://mcp.timecockpit.com",
      "oauth": { "clientId": "<your-client-id>", "callbackPort": 33418 }
    }
  }
}
```

Then run `/mcp` in the session and pick `timecockpit` to sign in with your work account.

The client id is required and user-specific: you receive it from your tenant administrator or time
cockpit support, and this package does not ship one. Entra does not support dynamic client
registration, so a client that cannot be handed a pre-registered client id fails at sign-in (see
`references/troubleshooting.md`).

For read-only own-data mode add `"headers": { "X-access": "readonly", "X-scope": "owndata" }`.

## 5. Every other client

`references/client-configs.md` carries one complete, copy-paste snippet per client — Claude Code
(remote and local), Claude Desktop / claude.ai custom connector, VS Code Copilot (remote and
local), GitHub Copilot CLI, Codex CLI, Cursor — plus which clients accept headers, which need the
route-segment URL form instead, and which clients' pre-registered-client-id support is unconfirmed.

## 6. Verify the connection, in this order

1. `ping` — the host is reachable and the request is authenticated. If this fails, nothing else
   will.
2. `get_environment` — you are attached to a real tenant. Check that the environment and version
   look like the system the user expected.
3. `get_current_user` — the agent is acting as the right person.
4. `entra_whoami` (remote only) — the Entra identity plus the tenant, sandbox and access/scope the
   server actually resolved for this connection. This is the tool that settles "why is it showing
   the wrong data".

Then read the tool list once and tell the user what they actually got, especially if something
they expect is absent.

## 7. "A tool is missing" is usually configuration

Three reasons a tool legitimately disappears — check them before reporting a bug:

| Cause | What vanishes |
|---|---|
| `access=readonly` | Every mutating tool: `create_timesheet`, `create_entity`, `update_entity`, `delete_entity`, `execute_action`, and the local script tools. |
| `scope=owndata` | Every tool whose result cannot be pinned to the acting user: `execute_tcql_query`, `execute_list`, `execute_action`, `create_entity`, `update_entity`, `delete_entity`. `get_timesheets` and `create_timesheet` remain, filtered to that user. |
| Optional server capability | `get_lists`, `describe_list`, `execute_list`, `get_actions`, `execute_action` sit on a backing capability that is checked per deployment and tenant. They are genuinely absent when it is unavailable — not unconditionally available anywhere. |

Hosts differ as well: `entra_whoami` is remote-only, while `execute_script_literal`,
`execute_script_file`, `get_resource` and the `timecockpit://` model resources are local-only.
A hidden tool is also rejected when called by name, so guessing gets you nowhere.

## 8. The local stdio host

Launched through the desktop client, selecting an existing desktop configuration with `-c`. That
configuration carries the tenant, sandbox and profile, so there are no separate arguments for them:

```powershell
OnCockpit.Admin.exe --mcp -c "<configuration-name>" --mcp-access readonly --mcp-scope owndata
```

`--mcp-access` takes `Default` or `ReadOnly`, `--mcp-scope` takes `Default` or `OwnData`,
`--mcp-confirmation` takes `Required` or `Disabled`. `--mcp` needs the configured data context, so
it cannot be combined with `--no-data-context`, `--file`, `--action` or `--force-update`.

`OnCockpit.Admin.exe` ships with the time cockpit desktop client. Its full path is in the
`OnCockpitAdminBinary` environment variable; use `${OnCockpitAdminBinary}` (or `%OnCockpitAdminBinary%`
in a shell) as the command in client configurations rather than a hard-coded path, because the
updater changes the versioned install folder.

This host is a **trusted administrative** surface. `execute_script_literal` and
`execute_script_file` run arbitrary IronPython against the live tenant with the desktop user's full
rights, and `get_resource` plus the native resources expose the whole visible model. They are
mutating, so `--mcp-access readonly` hides the script tools. Point an agent here only when the work
genuinely needs it, and follow the `time-cockpit-local-mcp` skill (from the internal `software-architects/tc-agents-internal` package) for using those tools safely.

## 9. Installing these skills with apm

The skills live in the public `software-architects/timecockpit-agents` package, installed with
[Microsoft APM](https://microsoft.github.io/apm/):

```bash
apm install software-architects/timecockpit-agents                              # the whole package
apm install software-architects/timecockpit-agents/skills/timecockpit-mcp-setup # a single skill
```

Or add `software-architects/timecockpit-agents` under `dependencies.apm` in your own `apm.yml` and run
`apm install`. The package also declares the remote MCP server (name `timecockpit`, HTTP transport,
`https://mcp.timecockpit.com`), so `apm install` can write it into
`.mcp.json` (Claude Code), `.github/mcp.json` (Copilot CLI), `.vscode/mcp.json` (VS Code) or `.codex/config.toml` (Codex), depending on your configured targets, once you accept
the trust prompt. Then add your own client id to that generated entry (section 4); apm cannot know
it.

## 10. Where to go next

* Something is broken → `references/troubleshooting.md` (symptom, cause, fix).
* Client-specific JSON → `references/client-configs.md`.
* Now that it works: `timecockpit-time-tracking` (book and correct your own hours),
  `timecockpit-reporting` (totals, budgets, utilization), `timecockpit-tcql` (ad-hoc queries),
  `timecockpit-model-discovery` (what this tenant's model actually looks like),
  `timecockpit-record-management` (create, update, delete), `timecockpit-lists-and-actions`
  (named lists and model actions), and `time-cockpit-local-mcp` from the internal `software-architects/tc-agents-internal` package (administrative script execution).
