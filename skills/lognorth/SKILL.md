---
name: lognorth
description: Use when debugging production errors, checking logs, or user mentions "lognorth". Reads a self-hosted LogNorth instance through its MCP tools, or the REST API when MCP is not connected.
---

# LogNorth

Read production logs, errors, and traces from a self-hosted LogNorth instance.

**Your role:** The user is debugging. They ask "what's breaking in production?" or "find events for trace abc-123". Get the data, then present it clearly: tables for lists, full context for single events, a chronological flow for traces.

Everything here is read-only. There is no way to change or delete anything, so never tell the user you have muted, resolved, or cleaned up an issue.

## Use the MCP tools

If this plugin's MCP server is connected, four tools are available. Prefer them over `curl`: they need no shell, no `jq`, and no credentials in the command line.

| Tool | When to use it |
|------|----------------|
| `list_apps` | First, whenever a tool needs an `app_id`. One app? Use it silently. Several, and the user did not say which? Ask. |
| `list_issues` | Start here for "what is broken?". Grouped errors with counts and trend. |
| `search_logs` | Find the failing requests: text, `errors_only`, `since`, `app_id`, `limit`. |
| `get_event` | One event with its parsed context and every event in the same trace. |

`search_logs` defaults to the last hour and 20 events. Widen deliberately with `since` (`15m`, `2h`, `7d`) and `limit` (max 200) rather than asking for everything up front — a narrow first query is cheaper and usually answers the question.

**A typical investigation**

1. `list_issues` — what is broken, and how often.
2. `search_logs` with `errors_only: true` — the actual failing requests.
3. `get_event` on one of them — the full context plus the trace around it.

That is three calls to a root cause. Do not dump raw JSON at the user; read it and say what it means.

## When MCP is not connected

Fall back to the REST API. It returns the same data.

```bash
curl -s "$LOGNORTH_URL/api/v1/agent/issues" \
  -H "Authorization: Bearer $LOGNORTH_AGENT_KEY" | jq '.issues'
```

| Endpoint | Returns |
|----------|---------|
| `GET /api/v1/agent/apps` | Apps and their ids |
| `GET /api/v1/agent/issues` | Grouped errors with counts |
| `GET /api/v1/agent/events` | Events, filtered |
| `GET /api/v1/agent/events/:id` | One event plus `relatedEvents` from its trace |

Filters for `/events`: `app_id`, `is_error`, `search`, `limit`, `offset`, `start_time`, `end_time` (RFC3339).

Needs `LOGNORTH_URL` and `LOGNORTH_AGENT_KEY` (read-only, starts with `lgn-agent-`, from Settings > Developer). It is separate from the app keys (`lgn-`) that SDKs use to send events. `LOGNORTH_API_KEY` is the older name for the same key: if `LOGNORTH_AGENT_KEY` is unset, use it.

Requires LogNorth v0.16.0 or later for MCP. The REST endpoints work on earlier versions.

## Data model

One event is one thing that happened: a request, a job, a log line.

| Field | Description |
|-------|-------------|
| `message` | What happened, human readable |
| `timestamp` | When |
| `duration_ms` | How long it took |
| `trace_id` | Correlation id: everything from one request shares it |
| `context` | Arbitrary metadata: `method`, `path`, `status`, `error`, and whatever the app sent |
| `is_error` | Derived on ingest: `context.error` exists, or `context.status >= 500` |

There are no severity levels. An event is an error or it is not.

## Reading issues

Issues group identical errors. The counts are the story.

| Field | What it tells you |
|-------|-------------------|
| `count` | Total ever |
| `count_24h` | Is it happening right now? |
| `count_7d` | Weekly volume: how big is this? |
| `trend` | "up" = getting worse, "down" = improving |
| `is_new` | First seen under 24h ago: likely a fresh regression |
| `first_seen` / `last_seen` | When it started, and whether it has stopped |

**Triage order**

1. `is_new` — a new error since the last deploy. Look here first.
2. `trend: "up"` with a high `count_24h` — actively getting worse.
3. High `count_7d`, low `count_24h`, `trend: "down"` — an old problem that is already fading. Say so instead of raising an alarm.

One error in a thousand requests is usually normal. Check the volume before calling something urgent.

## Following a trace

Every event from one request shares a `trace_id`. `get_event` returns `relatedEvents`, which is the whole request in order: the HTTP entry point, the jobs it triggered, and where it failed.

Read the trace before guessing at a cause. It shows what succeeded right before the failure, which is usually what rules out half the candidates.

## Presenting results

| View | Format |
|------|--------|
| Issues | Table: error, `count_24h`, `count_7d`, trend, last seen |
| Events | Table: message, time, path, status, duration |
| One event | The context object, with the error and stack trace called out |
| Trace | Chronological list, marking where it broke |

Lead with the answer. "Three checkout failures in the last 12 minutes, all Stripe timeouts" beats a table the user has to read for themselves.

## Related skills

- `lognorth-integrate` — add the LogNorth SDK to a project.
- `lognorth-deploy` — stand up a LogNorth server.
