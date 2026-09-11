# Client configuration snippets

One complete snippet per client. Replace every `<placeholder>`. Add only the headers you actually
need — all five settings are optional (see section 3 of `SKILL.md`).

## The values you need

| | Value |
|---|---|
| Server name | `timecockpit` (any name works; it becomes the tool prefix) |
| Production URL | `https://mcp.timecockpit.com` |
| Preview URL | `https://mcp-preview.timecockpit.com` |
| Transport | HTTP (streamable HTTP / "http"), **not** SSE, **not** stdio |
| **OAuth client id** | `<your-client-id>` — provided to you by your tenant administrator or time cockpit support; this package does not ship one |
| OAuth scope (only if a client demands it explicitly) | `https://mcp.timecockpit.com/mcp.access` (preview: `https://mcp-preview.timecockpit.com/mcp.access`) |

Every snippet below uses the `<your-client-id>` placeholder; replace it with the id you were given
before copying. Ask for it if you do not have it — nothing in this skill can stand in for it.

The client id is the one value every client needs and no client can discover. Microsoft Entra ID is
the authorization server behind the MCP endpoint and it does not support dynamic client
registration, so each client must be given a pre-registered public client id. The server does
publish OAuth metadata at `/.well-known/oauth-protected-resource` and
`/.well-known/oauth-authorization-server`, which is how clients find the authorize and token
endpoints and the scope — but not the client id.

Sign-in is a normal browser flow with a Microsoft work account. The account must have time cockpit
access, and your Entra tenant must have consented to the time cockpit MCP client application; if it
has not, an administrator has to consent once (see `troubleshooting.md`).

## Headers or route segments?

Every setting can travel as a header **or** as a route segment, never both. Clients that let you
set headers should use headers. Clients that do not (Claude Desktop, claude.ai) must put the
settings in the URL instead:

```
https://mcp.timecockpit.com/access/readonly/scope/owndata
https://mcp.timecockpit.com/tcTenantId/<tenant-id>/sandboxEnvironment/test
```

The two forms are equivalent. Mixing both for the *same* setting is rejected with `404`.

---

## Claude Code — remote

Project-scoped `.mcp.json` in the repository root (shared with the team), or the same object in
`~/.claude.json` for a user-scoped connection:

```json
{
  "mcpServers": {
    "timecockpit": {
      "type": "http",
      "url": "https://mcp.timecockpit.com",
      "headers": {
        "X-tc-tenant-id": "<tenant-id>",
        "X-access": "readonly",
        "X-scope": "owndata"
      },
      "oauth": {
        "clientId": "<your-client-id>",
        "callbackPort": 33418,
        "scopes": "https://mcp.timecockpit.com/mcp.access"
      }
    }
  }
}
```

* Drop `X-tc-tenant-id` unless your Entra tenant maps to several time cockpit tenants.
* Drop `X-access` / `X-scope` for a full-power administrator connection.
* `scopes` can usually be omitted — the server advertises it. Keep it if sign-in complains about
  the requested scope.
* `callbackPort` fixes the loopback redirect at `http://localhost:<port>/callback`. Pick a free
  port and keep it stable.

CLI equivalent:

```bash
claude mcp add --transport http timecockpit https://mcp.timecockpit.com \
  --header "X-access: readonly" --header "X-scope: owndata"
```

Authenticate with `/mcp` inside the session, then pick `timecockpit`. Docs:
<https://code.claude.com/docs/en/mcp>

## Claude Code — local stdio host

```json
{
  "mcpServers": {
    "timecockpit-local": {
      "command": "${OnCockpitAdminBinary}",
      "args": ["--mcp", "-c", "<configuration-name>", "--mcp-access", "readonly"]
    }
  }
}
```

No URL, no OAuth: the desktop client is already signed in. `-c` names an existing desktop
configuration, which carries the tenant, sandbox and profile. The `OnCockpitAdminBinary`
environment variable holds the full path to `OnCockpit.Admin.exe`; clients that do not expand
`${VAR}` in `command` need the literal value of that variable instead (`echo %OnCockpitAdminBinary%`).

## Claude Desktop / claude.ai custom connector

Settings → Connectors → **Add custom connector**:

| Field | Value |
|---|---|
| Name | `time cockpit` |
| Remote MCP server URL | `https://mcp.timecockpit.com` — or, to select modes, `https://mcp.timecockpit.com/access/readonly/scope/owndata` |
| Advanced settings → OAuth Client ID | `<your-client-id>` |
| Advanced settings → OAuth Client Secret | leave empty (public client) |

Custom headers cannot be set here, so **use the route-segment URL form** for tenant, sandbox,
access and scope.

This flow redirects to `https://claude.ai/api/mcp/auth_callback`, which must be registered as a
redirect URI on the client application. If sign-in ends with a redirect-URI error, that
registration is missing — ask your administrator or software architects; it is not something you can
fix in the connector dialog. Docs:
<https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp>

## VS Code (GitHub Copilot agent mode) — remote

`.vscode/mcp.json` in the workspace:

```json
{
  "servers": {
    "timecockpit": {
      "type": "http",
      "url": "https://mcp.timecockpit.com",
      "headers": {
        "X-access": "readonly",
        "X-scope": "owndata"
      }
    }
  }
}
```

VS Code drives the OAuth flow itself against the metadata the server publishes and prompts inside
the editor. **Unconfirmed:** whether the installed VS Code version lets you pin a pre-registered
client id — support for that is limited and version-dependent. If sign-in fails with a
dynamic-registration error there is no configuration workaround: use Claude Code or Claude Desktop
for the remote host, or the local stdio host below, and report your VS Code version. Docs:
<https://code.visualstudio.com/docs/agents/reference/mcp-configuration>

## VS Code — local stdio host

```json
{
  "servers": {
    "timecockpit-local": {
      "type": "stdio",
      "command": "${OnCockpitAdminBinary}",
      "args": ["--mcp", "-c", "<configuration-name>"]
    }
  }
}
```

## GitHub Copilot CLI

`~/.copilot/mcp-config.json`:

```json
{
  "mcpServers": {
    "timecockpit": {
      "type": "http",
      "url": "https://mcp.timecockpit.com",
      "headers": {
        "X-access": "readonly",
        "X-scope": "owndata"
      }
    }
  }
}
```

`copilot mcp add` walks you through the same fields interactively.

**Unconfirmed:** a pre-registered OAuth client id is reportedly ignored by this client (tracked as
an open issue upstream), so sign-in may fail with a dynamic-registration error that no
configuration can fix. Treat Copilot CLI against the remote host as unverified. Docs:
<https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-mcp-servers>

## Codex CLI

`~/.codex/config.toml`:

```toml
[mcp_servers.timecockpit]
url = "https://mcp.timecockpit.com"

[mcp_servers.timecockpit.http_headers]
"X-access" = "readonly"
"X-scope" = "owndata"
```

Then authenticate:

```bash
codex mcp login timecockpit
```

**Unconfirmed:** whether this client can be given the pre-registered client id. If `codex mcp
login` fails with a dynamic-registration error, that is the cause. Docs:
<https://developers.openai.com/codex/config-reference>

## Cursor

`.cursor/mcp.json` in the project (or `~/.cursor/mcp.json` globally):

```json
{
  "mcpServers": {
    "timecockpit": {
      "type": "http",
      "url": "https://mcp.timecockpit.com",
      "headers": {
        "X-access": "readonly",
        "X-scope": "owndata"
      },
      "auth": {
        "CLIENT_ID": "<your-client-id>"
      }
    }
  }
}
```

Cursor Desktop completes the flow on `http://localhost:8787/callback`. Docs:
<https://cursor.com/docs/mcp>

---

## After configuring any client

Run the four checks from section 6 of `SKILL.md`: `ping`, `get_environment`, `get_current_user`,
and `entra_whoami` on the remote host. `entra_whoami` echoes the resolved tenant, sandbox and
access/scope, which is the quickest way to prove the headers or route segments arrived.
