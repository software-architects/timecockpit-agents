# Troubleshooting a time cockpit MCP connection

Work top to bottom: sign-in problems first, then routing, then tools, then results. Run
`entra_whoami` (remote host) before guessing — it reports the Entra identity, the resolved time
cockpit tenant and user, the sandbox, and the access/scope settings the server actually applied.

## Sign-in and authorization

| Symptom | Cause | Fix |
|---|---|---|
| `Incompatible auth server: does not support dynamic client registration` | The client never received the pre-registered OAuth client id. Microsoft Entra ID is the authorization server and does not offer dynamic registration. | Set your client id (obtained from your tenant administrator or time cockpit support) as shown in `client-configs.md` (`oauth.clientId`, `auth.CLIENT_ID`, or the connector's Advanced settings). If the client accepts the setting but still reports this, its client-id support is broken or unimplemented — see the per-client notes and switch clients. |
| Browser flow ends on an Entra error about the redirect URI | The redirect URI your client uses is not registered on the time cockpit MCP client application (for example `https://claude.ai/api/mcp/auth_callback`, or a loopback port). | Not fixable from your side: ask your administrator or software architects to register that exact redirect URI. Keeping `callbackPort` stable avoids re-registering per session. |
| Entra says the application needs admin approval / consent | Your Entra tenant has not consented to the time cockpit MCP client application and your tenant policy blocks user consent. | An Entra administrator consents once for the tenant. |
| Sign-in succeeds, every tool call returns `401` | The token was not sent, expired, or was issued for a different endpoint (for example a preview token against production). | Re-authenticate (`/mcp` in Claude Code, `codex mcp login timecockpit`). Check that the URL in the config is the endpoint you signed in to. |
| Sign-in succeeds, every tool call returns `403` | The account authenticated but is not permitted: no time cockpit access for that user, or the requested scope was not granted. | Confirm the user can sign in to time cockpit normally. Then check the granted scope; if a client demands one explicitly, use the scope value in `client-configs.md`. |
| Wrong person's data | You are signed in with a different work account than expected. | `get_current_user` and `entra_whoami`, then re-authenticate with the right account. |

## Tenant, sandbox and settings routing

| Symptom | Cause | Fix |
|---|---|---|
| Sign-in works but requests fail because the tenant is not uniquely mapped | Your Entra tenant maps to more than one time cockpit tenant, so the server cannot pick one. | Set `X-tc-tenant-id: <tenant-id>`, or use `/tcTenantId/<tenant-id>` in the URL. Your administrator has the tenant id. |
| `404` with a message naming a setting | The setting was supplied twice — once as a header and once as a route segment. | Pick one form. Header *or* route, per setting. |
| `404` `Unknown route parameter '...'` | A typo in a route segment key, or an odd number of segments (route settings are key/value pairs). | Use exactly `tcTenantId`, `sandboxEnvironment`, `profile`, `access`, `scope`, each followed by a value. |
| `404` `Invalid value '...' for header 'X-access'` (or `X-scope`) | An unsupported value. `Access` accepts only `Default` / `ReadOnly`; `Scope` only `Default` / `OwnData`. | There is no `readwrite` and no `all`: omit the setting, or send `default`. Casing does not matter. |
| The agent reads or writes production when you meant the sandbox | Sandbox is opt-in; omitting it means production. | `X-sandbox-environment: test`, or `/sandboxEnvironment/test`. Verify with `entra_whoami`. |
| The agent hit the sandbox when you meant production | A leftover sandbox header or `/sandboxEnvironment/test` segment. | Remove it. Verify with `entra_whoami` and `get_environment`. |
| Settings appear to be ignored | They were set on a different profile/scope of the config file than the one in use, or the client was not restarted. | Reload the client, then confirm with `entra_whoami`, which echoes the requested settings back. |

## Missing or rejected tools

| Symptom | Cause | Fix |
|---|---|---|
| No mutating tools (`create_timesheet`, `create_entity`, `update_entity`, `delete_entity`, `execute_action`) | The connection runs with `access=readonly`. | Remove `X-access` / `/access/readonly` and reconnect — deliberately, after considering section 2 of `SKILL.md`. |
| `execute_tcql_query`, `execute_list` and generic CRUD are missing, but `get_timesheets` works | The connection runs with `scope=owndata`, which hides everything that cannot be limited to the acting user. | Use the typed tools (`get_timesheets`, `create_timesheet`), or reconnect without `owndata` if broader access is genuinely intended. |
| `get_lists` / `describe_list` / `execute_list` / `get_actions` / `execute_action` are absent | These depend on an optional server capability that is checked per deployment and tenant. Absence is expected, not a bug. | Use `execute_tcql_query` (outside `owndata`) or the typed tools instead. |
| `entra_whoami` not found | It is remote-only. | Expected on the local stdio host; use `get_current_user` there. |
| `execute_script_literal`, `execute_script_file`, `get_resource` or `timecockpit://` resources not found | They are local-only, trusted administrative capabilities and are never registered by the remote host. | Use the local stdio host and the `time-cockpit-local-mcp` skill from the team-internal `software-architects/tc-agents-internal` package. |
| A tool is called by name and rejected even though it is not in the list | Hidden tools are also rejected at call time — hiding and authorization use the same rules. | Fix the configuration; do not retry the call. |

## Results and confirmations

| Symptom | Cause | Fix |
|---|---|---|
| A mutation returns `status: confirmation_required` | The default policy demands an explicit acknowledgement before anything is written. Nothing was changed. | Tell the user exactly which record and which fields will change, then re-issue the identical call with `confirmed=true`. |
| A result says it is truncated, or `truncated: true` | Row and response-size caps are enforced before the query runs; large results are cut on purpose. | Narrow the request: a date range, a user or project filter, a smaller `Top(N)`, fewer projected columns. Aggregate on the server rather than paging raw rows — see `timecockpit-reporting`. |
| A TCQL query is rejected for containing comments | Comments could hide the real `Select` / `Top` clause from the guardrail, so they are refused. | Remove `//` and `/* */` from the query. |
| A field or entity "does not exist" although you saw it in the UI | Every tenant's model differs, and system plus internal entities are never exposed over MCP. | Discover names with `get_entities` / `describe_entity` (see `timecockpit-model-discovery`) instead of guessing. |

## Local stdio host

| Symptom | Cause | Fix |
|---|---|---|
| The host exits immediately with an invalid-option message | `--mcp` was combined with `--no-data-context`, `--file`, `--action` or `--force-update`, or an `--mcp-*` value was invalid. | Run `--mcp` with only `-c <configuration>` and the `--mcp-access` / `--mcp-scope` / `--mcp-confirmation` values. `--mcp-access`: `Default`/`ReadOnly`. `--mcp-scope`: `Default`/`OwnData`. `--mcp-confirmation`: `Required`/`Disabled`. |
| An `--mcp-*` setting was passed without `--mcp` | Those settings only apply to the MCP host. | Add `--mcp`. |
| The host cannot get a data context | `-c` names a configuration that does not exist, or that configuration has never been set up / signed in on this machine. | Start the desktop client normally, confirm the configuration name and that it connects, then reuse that exact name. |
| The client reports the server as failed to start | `OnCockpitAdminBinary` is not set or the client did not expand it, or the desktop client is mid-update. | Check `echo %OnCockpitAdminBinary%` in a new shell and use its literal value if the client does not expand `${VAR}`. Start the desktop client once, let any update finish, then retry. |
| The wrong tenant or sandbox | Tenant, sandbox and profile come from the `-c` configuration; there are no separate arguments. | Select or create the desktop configuration that points at the intended tenant and sandbox. |

## Still stuck

Collect, in this order: the client and its version, the URL or command line in use (with the tenant
id redacted), whether headers or route segments were used, the exact error text, and the output of
`ping`, `get_environment`, `get_current_user` and `entra_whoami`. Never paste tokens or
credentials. Then contact your tenant administrator, or time cockpit support at
<support@timecockpit.com>.
