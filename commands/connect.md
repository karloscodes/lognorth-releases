---
description: Connect this machine to a LogNorth server so the MCP tools work
argument-hint: [url] [agent-key]
allowed-tools: Bash(curl:*), Read, Edit, Write
---

Connect the user's LogNorth server to the `lognorth` MCP server.

The plugin's MCP config reads `${LOGNORTH_URL}` and `${LOGNORTH_AGENT_KEY}`. Shell exports do not reach a GUI-launched client, so store them where Claude Code itself will apply them: the `env` block of `~/.claude/settings.json`.

Arguments, if given: `$1` is the URL, `$2` is the agent key.

## 1. Collect the two values

Use the arguments when present. Otherwise check, in order, and only ask for what is still missing:

- `env | grep -E '^LOGNORTH_(URL|AGENT_KEY|API_KEY)='`
- the `env` block already in `~/.claude/settings.json`

When you have to ask:

- **URL** — their LogNorth address, for example `https://logs.yoursite.com`. Strip any trailing slash and any trailing `/mcp`.
- **Agent key** — from **Settings > Developer** in LogNorth. It starts with `lgn-agent-`. If they paste something starting with `lgn-` but not `lgn-agent-`, that is an app key for sending events; tell them and ask for the agent key instead.

Never echo the key back in full. Show the last four characters at most.

## 2. Verify before writing anything

```bash
curl -sS -o /dev/null -w '%{http_code}' -X POST "<url>/mcp" \
  -H "Authorization: Bearer <key>" \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

| Code | Meaning | Say this |
|------|---------|----------|
| `200` | Works | Continue to step 3. |
| `401` | Key rejected | Wrong key, or an app key. Get the agent key from Settings > Developer. |
| `404` | No MCP endpoint | The server predates v0.16.0. Run `lognorth update` on it. |
| `000` | Unreachable | Check the URL, and whether the host is reachable from here. |

Do not write config for a setup that does not answer. Fix it with the user first.

## 3. Store them

Read `~/.claude/settings.json`, then add or update its `env` block, preserving everything else in the file:

```json
{
  "env": {
    "LOGNORTH_URL": "https://logs.yoursite.com",
    "LOGNORTH_AGENT_KEY": "lgn-agent-..."
  }
}
```

Create the file with just that object if it does not exist. Keep the existing indentation style.

## 4. Confirm

Tell the user: connected to `<url>`, verified, and the tools appear after a restart because MCP config is read at startup. Then `/mcp` lists the server, and `/lognorth:investigate` with no argument shows what is wrong in production right now.

## Other clients

If the user is setting up Cursor, VS Code, Gemini CLI, Windsurf, Codex, or Zed instead, do not edit Claude Code's settings. Give them their client's config with the verified values filled in — the shapes are in the [plugin README](https://github.com/karloscodes/lognorth-releases#install) — and point out that those files hold the key in plain text, so it should not be committed.
