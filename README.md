# LogNorth plugin

Your agent should not have to ask you what production is doing. This plugin connects it straight to your [LogNorth](https://lognorth.com) server: it reads the failing requests, follows the trace, and tells you what broke, in the same pane as the code.

It bundles two things. The **MCP server** gives your agent four read-only tools over your logs. The **skills** teach it what the data means and how to triage an issue.

Everything stays on your box. Your agent asks your instance, and only the answer reaches your AI provider.

Needs LogNorth **v0.16.0 or later**. Run `lognorth update` if you are behind.

## What you get

| Tool | What it does |
|------|--------------|
| `list_apps` | Your apps and their ids |
| `search_logs` | Find requests by text, errors only, and time window |
| `get_event` | One event with its full context and every event in its trace |
| `list_issues` | Grouped errors with occurrence counts and trend |

| Skill | What it does |
|-------|--------------|
| `lognorth` | Debug production: triage issues, follow traces, read context |
| `lognorth-integrate` | Add the LogNorth SDK to a Go, Node, or Rails project |

| Command | What it does |
|---------|--------------|
| `/lognorth:connect` | Point this machine at your server: asks, verifies, saves |

There is no tool that writes, mutes, or deletes. The agent can look, never touch.

## First, two values

1. **Your URL**, for example `https://logs.yoursite.com`.
2. **An agent key** from **Settings > Developer** in LogNorth. It is read-only and starts with `lgn-agent-`. It is not the app key your SDKs use to send events.

## Install

### Claude Code

Three commands, **typed one at a time**, each followed by Enter. They are not a block to paste together.

```
/plugin marketplace add karloscodes/lognorth-releases
```

```
/plugin install lognorth
```

```
/lognorth:connect
```

If you see a prompt asking you to "Enter marketplace source", the first command ran without its argument. Type just `karloscodes/lognorth-releases` there, nothing else.

`/lognorth:connect` asks for the URL and the key, checks them against your server before saving anything, and stores them where Claude Code picks them up in every project and session. Restart, then `/mcp` shows the server.

Pass them on the same line if you prefer: `/lognorth:connect https://logs.yoursite.com lgn-agent-...`

### Claude Code, without the plugin

```bash
claude mcp add --transport http lognorth https://logs.yoursite.com/mcp \
  --header "Authorization: Bearer lgn-agent-..."
```

### Cursor

`~/.cursor/mcp.json` for every project, or `.cursor/mcp.json` for one:

```json
{
  "mcpServers": {
    "lognorth": {
      "url": "https://logs.yoursite.com/mcp",
      "headers": { "Authorization": "Bearer lgn-agent-..." }
    }
  }
}
```

### VS Code, with Copilot

```bash
code --add-mcp '{"name":"lognorth","type":"http","url":"https://logs.yoursite.com/mcp","headers":{"Authorization":"Bearer lgn-agent-..."}}'
```

Or commit `.vscode/mcp.json` so the team gets it:

```json
{
  "servers": {
    "lognorth": {
      "type": "http",
      "url": "https://logs.yoursite.com/mcp",
      "headers": { "Authorization": "Bearer lgn-agent-..." }
    }
  }
}
```

### Gemini CLI

`~/.gemini/settings.json`:

```json
{
  "mcpServers": {
    "lognorth": {
      "httpUrl": "https://logs.yoursite.com/mcp",
      "headers": { "Authorization": "Bearer lgn-agent-..." }
    }
  }
}
```

### Windsurf

`~/.codeium/windsurf/mcp_config.json`:

```json
{
  "mcpServers": {
    "lognorth": {
      "serverUrl": "https://logs.yoursite.com/mcp",
      "headers": { "Authorization": "Bearer lgn-agent-..." }
    }
  }
}
```

### Codex CLI, Zed, and other stdio-only clients

Some clients still speak only stdio. Bridge them with `mcp-remote`, which turns a stdio client into an HTTP one.

Codex CLI, in `~/.codex/config.toml`:

```toml
[mcp_servers.lognorth]
command = "npx"
args = [
  "-y", "mcp-remote",
  "https://logs.yoursite.com/mcp",
  "--header", "Authorization: Bearer lgn-agent-..."
]
```

Zed, in `settings.json`:

```json
{
  "context_servers": {
    "lognorth": {
      "source": "custom",
      "command": {
        "path": "npx",
        "args": ["-y", "mcp-remote", "https://logs.yoursite.com/mcp", "--header", "Authorization: Bearer lgn-agent-..."]
      }
    }
  }
}
```

The same `npx -y mcp-remote <url> --header "Authorization: Bearer <key>"` command works for any client that accepts a stdio command.

## Then ask

```
You: the checkout endpoint is throwing 500s, what's happening?

Agent: [list_issues, then search_logs errors_only]

Three failures on POST /api/checkout in the last 12 minutes, all the
same error: "Stripe timeout after 30s", 121-123ms each.

[get_event on the first one]

The trace shows GET /api/health succeeded in the same window, so the
box is up. The timeout is Stripe-side.
```

## If it does not connect

- **404 on `/mcp`** — the server predates v0.16.0. Run `lognorth update`.
- **401** — wrong key, or an app key instead of an agent key. Agent keys start with `lgn-agent-` and come from Settings > Developer.
- **Nothing in the tool list** — most clients only read MCP config at startup. Restart it.
- **Claude Code shows the server but no tools** — the URL or key never resolved. Run `/lognorth:connect`, which verifies both before saving.
- **Client not listed above** — every client takes either a URL with headers or a stdio command. Use the JSON shape from Cursor for the first, and the `mcp-remote` command for the second. Paths for these config files move between releases, so check your client's docs if the one above is missing.

## Docs

- [MCP setup and every client's config](https://lognorth.com/docs/integrations/mcp/)
- [All docs](https://lognorth.com/docs/)

## Releases

This repo also carries the LogNorth binary releases the installer downloads. See [Releases](https://github.com/karloscodes/lognorth-releases/releases).
