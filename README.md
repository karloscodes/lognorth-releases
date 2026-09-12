# LogNorth plugin

Your agent should not have to ask you what production is doing. This plugin connects it straight to your [LogNorth](https://lognorth.com) server: it reads the failing requests, follows the trace, and tells you what broke, in the same pane as the code.

It bundles two things. The **MCP server** gives your agent four read-only tools over your logs. The **skills** teach it what the data means and how to triage an issue.

Everything stays on your box. Your agent asks your instance, and only the answer reaches your AI provider.

Needs LogNorth **v0.16.0 or later** for MCP. Run `lognorth update` if you are behind. The skills fall back to the REST API, which works on older versions.

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

There is no tool that writes, mutes, or deletes. The agent can look, never touch.

## First, two values

1. **Your URL**, for example `https://logs.yoursite.com`.
2. **An agent key** from **Settings > Developer** in LogNorth. It is read-only and starts with `lgn-agent-`. It is not the app key your SDKs use to send events.

Export them and every install below works as written:

```bash
export LOGNORTH_URL="https://logs.yoursite.com"
export LOGNORTH_AGENT_KEY="lgn-agent-..."
```

## Install

### Claude Code — the plugin

```
/plugin marketplace add karloscodes/lognorth-releases
/plugin install lognorth@lognorth
```

That is the whole thing: MCP server and both skills, wired to the two environment variables above. Check it with `/mcp`.

### Claude Code — MCP only

```bash
claude mcp add --transport http lognorth "$LOGNORTH_URL/mcp" \
  --header "Authorization: Bearer $LOGNORTH_AGENT_KEY"
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

### Skills without MCP

For an agent that reads skills but has no MCP support, install the skills alone. They call the REST API with `curl`:

```bash
npx skills add karloscodes/lognorth-releases --skill lognorth
```

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
- **Client not listed above** — every client takes either a URL with headers or a stdio command. Use the JSON shape from Cursor for the first, and the `mcp-remote` command for the second. Paths for these config files move between releases, so check your client's docs if the one above is missing.

## Docs

- [MCP setup](https://lognorth.com/docs/integrations/mcp/)
- [Agent API reference](https://lognorth.com/docs/integrations/ai-agents/)
- [All docs](https://lognorth.com/docs/)

## Releases

This repo also carries the LogNorth binary releases the installer downloads. See [Releases](https://github.com/karloscodes/lognorth-releases/releases).
