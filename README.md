# LogNorth plugin

Your agent should not have to ask you what production is doing. This plugin connects it straight to your [LogNorth](https://lognorth.com) server: it reads the failing requests, follows the trace, and tells you what broke, in the same pane as the code.

It bundles two things. The **MCP server** gives your agent six read-only tools over your logs and alerts. The **skills** teach it how to go from an alert to the line of code and the commit that caused it.

Everything stays on your box. Your agent asks your instance, and only the answer reaches your AI provider.

It installs as a plugin in Claude Code, Codex, GitHub Copilot CLI, VS Code, and Gemini CLI, from this one repo.

Needs LogNorth **v0.16.0 or later**. `list_alerts`, `endpoint_timeline`, and `/lognorth:investigate` need the release after v0.16.2. Run `lognorth update` if you are behind.

## What you get

| Tool | What it does |
|------|--------------|
| `list_apps` | Your apps and their ids |
| `list_alerts` | What is alerting now: error rate spikes, traffic drops, apps down, each against its normal level |
| `endpoint_timeline` | Requests and errors for one endpoint over time, to see when a problem started |
| `list_issues` | Grouped errors with occurrence counts and trend |
| `search_logs` | Find requests by text, issue, errors only, and time window |
| `get_event` | One event with its full context and every event in its trace |

| Skill | What it does |
|-------|--------------|
| `lognorth` | Investigate production: from an alert to the failing request, the code, and the commit |
| `lognorth-integrate` | Add the LogNorth SDK to a Go, Node, or Rails project |

| Command | What it does |
|---------|--------------|
| `/lognorth:connect` | Point this machine at your server: asks, verifies, saves |
| `/lognorth:investigate` | Investigate an alert or issue. Every LogNorth alert email ends with this command, ready to paste |

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

### Codex

```
codex plugin marketplace add karloscodes/lognorth-releases
codex plugin add lognorth@karloscodes
```

That installs the skills. For the native tools too, add the server once. Codex reads the key from your environment each session:

```bash
codex mcp add lognorth --url https://logs.yoursite.com/mcp --bearer-token-env-var LOGNORTH_AGENT_KEY
```

### GitHub Copilot CLI

```
copilot plugin marketplace add karloscodes/lognorth-releases
copilot plugin install lognorth@karloscodes
```

### VS Code, with Copilot

Add the marketplace to your settings, then install **lognorth** from the Extensions view (search `@agentPlugins`):

```json
"chat.plugins.marketplaces": ["karloscodes/lognorth-releases"]
```

For the native tools, add the server:

```bash
code --add-mcp '{"name":"lognorth","type":"http","url":"https://logs.yoursite.com/mcp","headers":{"Authorization":"Bearer lgn-agent-..."}}'
```

### Gemini CLI

```
gemini extensions install https://github.com/karloscodes/lognorth-releases
```

It asks for your URL and agent key, and keeps the key in your system keychain. Then `/lognorth:investigate` works as in Claude Code.

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

### Zed, and other stdio-only clients

Bridge them with `mcp-remote`, which turns a stdio client into an HTTP one. Zed, in `settings.json`:

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

### Any agent: the skills and two variables

The skills work even where the MCP tools are not connected. They call the same endpoint over HTTPS with `curl`, using two variables from your shell profile:

```bash
export LOGNORTH_URL=https://logs.yoursite.com
export LOGNORTH_AGENT_KEY=lgn-agent-...
```

For an agent without plugins, copy `skills/lognorth` and `skills/lognorth-integrate` into its skills folder.

## Then ask

An alert email ends with a line like this. Paste it into your agent:

```
/lognorth:investigate /api/checkout

Agent: [list_alerts, endpoint_timeline, search_logs, get_event]

/api/checkout fails 8% of the time since 14:05, normally 1%. All 212
failures are "Stripe timeout after 2s" at app/payments/charge.rb:41.

[git log around 14:05]

a3f9c1e, deployed 14:02, cut the Stripe timeout from 30s to 2s.
Stripe's p99 today is 3.1s. Restore the 30s timeout; the diff is below.
```

Or just ask: "is anything broken in production?"

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
