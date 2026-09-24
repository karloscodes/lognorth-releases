---
description: Connect this machine to a LogNorth server so the MCP tools work
argument-hint: [url] [agent-key]
allowed-tools: Bash(north:*), Bash(command -v north), Bash(curl -fsSL https\://lognorth.com/cli:*)
---

Connect the user's LogNorth server. The plugin's MCP server is `north mcp`, a small local command that reads the connection from `~/.config/lognorth/remote.json`. `north connect` checks the URL and key against the server and writes that file. No agent config holds the key.

Arguments, if given: `$1` is the URL, `$2` is the agent key.

## 1. Collect the two values

Use the arguments when present. Ask only for what is missing:

- **URL**: their LogNorth address, for example `https://logs.yoursite.com`.
- **Agent key**: from **Settings > Developer** in LogNorth. It starts with `lgn-agent-`. A key that starts with `lgn-` but not `lgn-agent-` is an app key for sending events; tell them and ask for the agent key.

Never echo the key back in full. Show the last four characters at most.

## 2. Install north if it is missing

```bash
command -v north
```

If that prints nothing, install it. The installer downloads one binary, checks its checksum, and needs no sudo:

```bash
curl -fsSL https://lognorth.com/cli | sh
```

If the installer says to add a folder to the `PATH`, tell the user: the agent starts `north mcp` by name, so it must be on the `PATH`.

## 3. Connect

```bash
north connect <url> <key>
```

It verifies before it saves. When it fails, it says why: a rejected key, an app key, a server it cannot reach, or a LogNorth too old for MCP (`lognorth update` on the server fixes that one). Relay the message and fix it with the user.

## 4. Confirm

Tell the user: connected to `<url>`. The tools appear after a restart, because MCP config is read at startup. After that, `north connect` with a new key takes effect at once, with no restart. `/lognorth:investigate` with no argument shows what is wrong in production now.

If they also use Codex, Gemini CLI, or Cursor, `north agents` in a terminal adds LogNorth to those too.
