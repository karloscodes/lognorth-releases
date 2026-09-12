# LogNorth reference

Field meanings, triage rules, and the REST fallback. Read this when the tool output needs interpreting; the workflow itself is in SKILL.md.

## Data model

One event is one thing that happened: a request, a job, a log line.

| Field | Description |
|-------|-------------|
| `message` | What happened, human readable |
| `timestamp` | When |
| `duration_ms` | How long it took |
| `trace_id` | Correlation id. Everything from one request shares it |
| `context` | Arbitrary metadata: `method`, `path`, `status`, `error`, plus whatever the app sent |
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
| `active` | Had occurrences in the last 24h |
| `hash` | Stable id for this error type |

**Triage order**

1. `is_new` — new since the last deploy. Look here first.
2. `trend: "up"` with a high `count_24h` — actively getting worse.
3. High `count_7d`, low `count_24h`, `trend: "down"` — an old problem already fading. Say so instead of raising an alarm.

## Traces

`get_event` returns `relatedEvents`: every event sharing the `trace_id`, in order. That is the whole request — the HTTP entry point, the jobs it triggered, and where it broke.

## Presenting results

| View | Format |
|------|--------|
| Issues | Table: error, `count_24h`, `count_7d`, trend, last seen |
| Events | Table: message, time, path, status, duration |
| One event | The context object, with the error and stack trace called out |
| Trace | Chronological list, marking where it broke |

## REST fallback

When the MCP server is not connected, the same data is available over HTTP. Needs `LOGNORTH_URL` and `LOGNORTH_AGENT_KEY` (read-only, starts with `lgn-agent-`, from Settings > Developer). `LOGNORTH_API_KEY` is the older name for the same key.

```bash
curl -s "$LOGNORTH_URL/api/v1/agent/issues" \
  -H "Authorization: Bearer $LOGNORTH_AGENT_KEY" | jq '.issues'
```

| Endpoint | Returns |
|----------|---------|
| `GET /api/v1/agent/apps` | Apps and their ids |
| `GET /api/v1/agent/issues` | Grouped errors with counts |
| `GET /api/v1/agent/events` | Events, filtered |
| `GET /api/v1/agent/events/:id` | One event plus `relatedEvents` |

Filters for `/events`: `app_id`, `is_error`, `search`, `limit`, `offset`, `start_time`, `end_time` (RFC3339).

The agent key is separate from the app keys (`lgn-`) that SDKs use to send events, and it cannot ingest or sign in to the web UI.

MCP needs LogNorth v0.16.0 or later. These REST endpoints work on earlier versions.
