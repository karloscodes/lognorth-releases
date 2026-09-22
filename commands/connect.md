---
description: Connect this machine to a LogNorth server so the MCP tools work
argument-hint: [url] [agent-key]
allowed-tools: Bash(curl:*), Read, Edit, Write
---

Connect the user's LogNorth server to the `lognorth` MCP server.

The plugin's MCP config reads `${LOGNORTH_URL}` and `${LOGNORTH_AGENT_KEY}` from the environment. The URL goes in the `env` block of `~/.claude/settings.json`. The key never goes into a config file: many people keep `~/.claude/settings.json` in a dotfiles repo, and a key committed there is public.

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

**The URL.** Read `~/.claude/settings.json`, then add or update its `env` block, preserving everything else in the file:

```json
{
  "env": {
    "LOGNORTH_URL": "https://logs.yoursite.com"
  }
}
```

Create the file with just that object if it does not exist. Keep the existing indentation style.

**The key.** It goes in the environment as `LOGNORTH_AGENT_KEY`, never in `settings.json`. Before writing it into any file, check that git does not track that file:

```bash
f="$(readlink -f ~/.zshrc)"; git -C "$(dirname "$f")" ls-files --error-unmatch "$f" >/dev/null 2>&1 && echo tracked
```

- Not tracked: offer to add `export LOGNORTH_AGENT_KEY=...` to the user's shell profile.
- Tracked (a dotfiles repo): do not write it. Show the `export` line and tell the user to put it in an untracked file their profile sources (for example `~/.secrets`) or in their secret manager.

If an earlier version of this command left `LOGNORTH_AGENT_KEY` in the `env` block of `~/.claude/settings.json`, remove it from there, and tell the user to regenerate the key in LogNorth if that file was ever committed.

## 4. Confirm

Tell the user: connected to `<url>`, verified, and the tools appear after a restart because MCP config is read at startup. Then `/mcp` lists the server, and `/lognorth:investigate` with no argument shows what is wrong in production right now.

## Other clients

If the user is setting up Cursor, VS Code, Gemini CLI, Windsurf, Codex, or Zed instead, do not edit Claude Code's settings. Give them their client's config with the verified values filled in — the shapes are in the [plugin README](https://github.com/karloscodes/lognorth-releases#install) — and use the client's environment-variable syntax for the key where it has one (Codex: `--bearer-token-env-var LOGNORTH_AGENT_KEY`; Cursor: `${env:LOGNORTH_AGENT_KEY}`). Where a client can only hold the key in plain text, say so, and tell the user not to commit that file.
