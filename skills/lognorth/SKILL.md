---
name: lognorth
description: Use when debugging production errors, checking logs, or the user mentions "lognorth". Reads a self-hosted LogNorth instance through its MCP tools.
---

# LogNorth

The user's production log is available through the `lognorth` MCP tools. Read it before you guess.

## The tools

| Tool | Arguments |
|------|-----------|
| `list_issues` | `app_id` |
| `search_logs` | `search`, `errors_only`, `since` (`15m`, `2h`, `7d`), `app_id`, `limit` (max 200) |
| `get_event` | `id` |
| `list_apps` | none |

All read-only. Nothing here writes, mutes, or deletes.

## How to use them

1. **`list_issues`** — what is broken, grouped, with counts and trend.
2. **`search_logs`** with `errors_only: true` — the actual failing requests.
3. **`get_event`** on one of them — full context plus every event in the same trace.

Three calls to a root cause. Start narrow: `search_logs` defaults to the last hour and 20 events, and that usually answers the question. Widen with `since` and `limit` only when it does not.

Call `list_apps` first when a tool needs an `app_id`. One app: use it silently. Several, and the user did not say which: ask.

## Rules

- **Lead with the answer**, not the JSON. "Three checkout failures in 12 minutes, all Stripe timeouts" beats a table the user has to read for themselves.
- **Check the volume before calling it urgent.** One error in a thousand requests is normal.
- **Read the trace before naming a cause.** What succeeded just before the failure rules out half the candidates.
- **Never claim you fixed, muted, or resolved anything.** You can only read.

## More detail

Read [reference.md](reference.md) when you need it: what each field means, how to triage by `count_24h` and `trend`, and the REST endpoints to fall back on if the MCP server is not connected.
