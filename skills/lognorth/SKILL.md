---
name: lognorth
description: Use when debugging production errors, investigating an alert, checking logs, or the user mentions "lognorth". Reads a self-hosted LogNorth instance through its MCP tools, then finds the cause in the code.
---

# LogNorth

The user's production log is available through the `lognorth` MCP tools. You also have their code and its git history. A hosted log tool has only the first. Use both: the log says what broke and when, the repo says why.

All tools are read-only. Nothing here writes, mutes, or deletes.

## Investigate

Go in this order. Stop as soon as you know the cause.

1. **What is wrong now.** `list_alerts` for rate alerts (spike, drift, silence) and apps that are down. `list_issues` for grouped errors. When the user names a path or an issue, start from it.
2. **When it started.** `endpoint_timeline` for the path. Find the first step where errors rose or traffic fell, and compare it with the normal level in the response.
3. **What failed.** `search_logs` with `errors_only: true`, `issue: <hash>` when you have one, and `since` / `until` around the start time.
4. **The whole request.** `get_event` on one failure. Read the trace: what succeeded just before the failure rules out half the causes.
5. **The code.** Open `error_file:error_line` from the event. Then run `git log --since=<start minus 1 hour> --until=<start>` and look for the change that shipped just before the problem began.
6. **The fix.** Name the cause, cite the event id and the commit, and propose the change. Do not apply it unless the user asks.

Call `list_apps` when a tool needs an `app_id`. One app: use it silently. Several, and the user did not say which: ask.

## Rules

- **Lead with the answer**, not the JSON. "Checkout fails 8% of the time since 14:05, normally 1%. All Stripe timeouts, starting with a3f9c1e, which cut the timeout to 2s" beats a table.
- **Trust the alert's baseline.** An alert compares against the same time of day on earlier days, so it is already more than "errors went up". Quote its normal level.
- **Check the volume before calling it urgent.** One error in a thousand requests is normal. An issue with a high 7-day count and a falling trend is fading, so say that.
- **Say when you are guessing.** No trace, no `error_file`, or no commit near the start time means the cause is a hypothesis. Label it.
- **Never claim you fixed, muted, or resolved anything in LogNorth.** You can only read.

## Without the MCP tools

Some agents cannot connect the `lognorth` MCP server from a plugin. The same tools answer over plain HTTPS, so use the shell instead:

```bash
lognorth_mcp() {  # usage: lognorth_mcp <tool> ['<json arguments>']
  curl -sS "$LOGNORTH_URL/mcp" \
    -H "Authorization: Bearer $LOGNORTH_AGENT_KEY" -H 'Content-Type: application/json' \
    -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"'"$1"'","arguments":'"${2:-"{}"}"'}}'
}
lognorth_mcp list_alerts
lognorth_mcp search_logs '{"errors_only": true, "since": "2h"}'
```

- The tool's answer is JSON inside `result.content[0].text`. When `result.isError` is true, that text is the error message.
- The tools are the ones the steps above name, with the same arguments.
- If `LOGNORTH_URL` or `LOGNORTH_AGENT_KEY` is empty, ask the user to add both to their shell profile. The URL is their LogNorth address; the agent key starts with `lgn-agent-` and comes from **Settings > Developer**.

## When a tool is missing

`list_alerts`, `endpoint_timeline`, and the `issue` and `until` arguments of `search_logs` need a newer LogNorth. If they are not in the tool list, tell the user to run `lognorth update` on the server, then continue with the tools you have. In Claude Code, if no `lognorth` tools appear at all, run `/lognorth:connect`.

## More detail

Read [reference.md](reference.md) when you need it: what each field means, how alerts decide, and how traces fit together.
