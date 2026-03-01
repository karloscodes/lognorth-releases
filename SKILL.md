---
name: lognorth
description: Use when debugging production errors, checking logs, or user mentions "lognorth". Queries LogNorth error tracking API.
---

# LogNorth Agent API

Query your LogNorth instance to see errors, logs, and traces.

**Your role:** The user is debugging. They might ask "What's breaking in production?" or "Find events for trace abc-123". Query the API and present results clearly — tables for lists, details for single events, summaries for issues.

## Environment Variables

Requires:
- `LOGNORTH_URL` - Your LogNorth instance (e.g., `https://logs.yoursite.com`)
- `LOGNORTH_API_KEY` - Agent API key from Settings page (starts with `lgn-agent-`)

Note: Agent keys are read-only and separate from app keys (`lgn-`) used for ingestion.

## Data Model

| Concept | What it means |
|---------|---------------|
| **Event** | A single log entry (message + context metadata) |
| **Error** | An event where `is_error=true` (has `context.error` or status >= 500) |
| **Issue** | Unique error type, grouped by error message. Has occurrence count. |
| **Trace** | Events sharing a `trace_id` — typically one request's full journey |

### Event Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | int | Unique identifier |
| `message` | string | What happened (human readable) |
| `timestamp` | ISO 8601 | When it happened |
| `duration_ms` | int | How long it took (optional) |
| `trace_id` | string | Correlation ID for related events (optional) |
| `is_error` | bool | True if error detected |
| `context` | object | All metadata — method, path, status, error, user, etc. |

### Issue Fields

| Field | Type | Description |
|-------|------|-------------|
| `hash` | string | Unique identifier for this error type |
| `error` | string | The error text |
| `count` | int | Total occurrences (all time) |
| `count_24h` | int | Occurrences in last 24 hours |
| `count_7d` | int | Occurrences in last 7 days |
| `first_seen` | timestamp | When first occurred |
| `last_seen` | timestamp | Most recent occurrence |
| `trend` | string | "up", "down", or "stable" (comparing recent vs previous period) |
| `active` | bool | Had occurrences in last 24h |
| `is_new` | bool | First seen less than 24h ago |

**Assessing criticality:** Use `count_24h`, `count_7d`, and `trend` to gauge severity:
- High `count_24h` + trend "up" = urgent, getting worse
- High `count_7d` but low `count_24h` + trend "down" = improving
- `is_new` = true = new error, investigate immediately

## API Endpoints

### List Apps

**Always call this first** to see available apps:
- If only 1 app exists, use it automatically (no need to ask)
- If multiple apps and user didn't specify, ask which app they mean

```bash
curl -s "$LOGNORTH_URL/api/v1/agent/apps" \
  -H "Authorization: Bearer $LOGNORTH_API_KEY" | jq '.apps'
```

Returns: `[{"id": 1, "name": "Admin Dashboard"}, {"id": 2, "name": "API Server"}]`

### List Events

```bash
curl -s "$LOGNORTH_URL/api/v1/agent/events" \
  -H "Authorization: Bearer $LOGNORTH_API_KEY" | jq
```

**Filters:**

| Parameter | Example | Description |
|-----------|---------|-------------|
| `app_id` | `3` | Filter by app (get ID from /apps) |
| `is_error` | `true` or `false` | Only errors or only successes |
| `search` | `connection+refused` | Text search in message/context |
| `limit` | `50` | Max results (default 50) |
| `offset` | `100` | Pagination offset |
| `start_time` | `2024-01-01T00:00:00Z` | Filter by time range |
| `end_time` | `2024-01-02T00:00:00Z` | Filter by time range |

**Examples:**

Recent errors:
```bash
curl -s "$LOGNORTH_URL/api/v1/agent/events?is_error=true&limit=10" \
  -H "Authorization: Bearer $LOGNORTH_API_KEY" | jq '.events[] | {id, message, timestamp}'
```

Search by trace ID:
```bash
curl -s "$LOGNORTH_URL/api/v1/agent/events?search=trace-abc-123" \
  -H "Authorization: Bearer $LOGNORTH_API_KEY" | jq '.events'
```

Events from last hour:
```bash
curl -s "$LOGNORTH_URL/api/v1/agent/events?start_time=$(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ)" \
  -H "Authorization: Bearer $LOGNORTH_API_KEY" | jq '.events'
```

### Get Event Detail

```bash
curl -s "$LOGNORTH_URL/api/v1/agent/events/123" \
  -H "Authorization: Bearer $LOGNORTH_API_KEY" | jq
```

Returns:
- `event` — Full event with all context
- `eventContext` — Parsed context object
- `relatedEvents` — Other events with same trace_id

### List Issues

Grouped errors with occurrence counts:

```bash
curl -s "$LOGNORTH_URL/api/v1/agent/issues" \
  -H "Authorization: Bearer $LOGNORTH_API_KEY" | jq '.issues[] | {hash, error_message, count, last_seen}'
```

## Debugging Workflows

### Starting Point: What's breaking?

1. **List apps** (if multiple exist):
   ```bash
   curl -s "$LOGNORTH_URL/api/v1/agent/apps" -H "Authorization: Bearer $LOGNORTH_API_KEY" | jq '.apps'
   ```

2. **Check issues first** — See grouped errors with occurrence counts:
   ```bash
   curl -s "$LOGNORTH_URL/api/v1/agent/issues?app_id=3" -H "Authorization: Bearer $LOGNORTH_API_KEY" | jq '.issues'
   ```
   Issues show you patterns: which errors happen most, which are new, which are trending up.

3. **Get recent errors** — See the latest failures:
   ```bash
   curl -s "$LOGNORTH_URL/api/v1/agent/events?app_id=3&is_error=true&limit=10" -H "Authorization: Bearer $LOGNORTH_API_KEY" | jq '.events'
   ```

### Following a Trace

When an error has a `trace_id`, you can see the full request journey — what happened before and after the error.

1. **Get event detail** — Returns the event plus all related events in the same trace:
   ```bash
   curl -s "$LOGNORTH_URL/api/v1/agent/events/123" -H "Authorization: Bearer $LOGNORTH_API_KEY" | jq
   ```
   The response includes `relatedEvents` — all events sharing the same `trace_id`.

2. **Or search by trace_id directly**:
   ```bash
   curl -s "$LOGNORTH_URL/api/v1/agent/events?search=abc-123-trace-id" -H "Authorization: Bearer $LOGNORTH_API_KEY" | jq '.events'
   ```

**What traces reveal:**
- The HTTP request that started the trace (method, path, status)
- Background jobs or child operations triggered by the request
- Where in the flow the error occurred
- What happened before and after the failure

### Understanding Issues (Grouped Errors)

Issues group identical errors together. Use them to understand patterns:

| Field | What it tells you |
|-------|-------------------|
| `count` | Total occurrences ever |
| `count_24h` | How many today — is it actively happening? |
| `count_7d` | Weekly volume — is this a big problem? |
| `trend` | "up" = getting worse, "down" = improving |
| `is_new` | First seen < 24h ago — new regression? |
| `first_seen` / `last_seen` | When did this start? Still happening? |

**Triage priority:**
1. `is_new` = true → New error, investigate immediately
2. `trend` = "up" + high `count_24h` → Getting worse, urgent
3. High `count_7d` but low `count_24h` → Old problem, maybe improving

### Finding Related Errors

To find all occurrences of a specific error type:
```bash
curl -s "$LOGNORTH_URL/api/v1/agent/events?search=ConnectionRefused&is_error=true" -H "Authorization: Bearer $LOGNORTH_API_KEY" | jq '.events'
```

To see errors on a specific endpoint:
```bash
curl -s "$LOGNORTH_URL/api/v1/agent/events?search=/api/checkout&is_error=true" -H "Authorization: Bearer $LOGNORTH_API_KEY" | jq '.events'
```

## Presenting Results

| View | Format |
|------|--------|
| **Issues list** | Table: error message, count_24h, count_7d, trend, last_seen |
| **Errors list** | Table: message, timestamp, path, status |
| **Single event** | Full context object, highlight the error and stack trace |
| **Trace view** | Chronological list showing the request flow, mark where error occurred |

**Always do:**
- When showing an error, mention its `trace_id` so user can ask for the full trace
- When showing issues, highlight `is_new` and `trend` = "up" as urgent
- Link related events — "This error has 47 occurrences in the last 24h"
