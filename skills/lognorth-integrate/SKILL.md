---
name: lognorth-integrate
description: Use when setting up LogNorth in a project — installs the SDK, adds the middleware, and confirms events arrive. For a language without an SDK (Python, PHP, Elixir, Java, .NET, and others), writes a small client that follows the LogNorth protocol. Trigger when the user says "add lognorth", "set up logging", or "integrate lognorth".
---

# LogNorth Integration

Add LogNorth to the user's project, then prove it works by reading the events back. The job is done when you see the event, not when the code compiles.

## 1. Detect the stack

| File | Stack |
|------|-------|
| `go.mod` | Go |
| `package.json` | Node.js / Bun |
| `Gemfile` | Rails |
| Anything else | No SDK: write a client, see "Any other language" below |

If the project already uses OpenTelemetry (`@opentelemetry` in package.json, otel imports), use the OpenTelemetry setup below instead of an SDK.

## 2. Get the URL and one key per environment

- **URL**, for example `https://logs.yoursite.com`.
- **An app key per environment.** In LogNorth, each environment is its own app: `shop-prod`, `shop-staging`. Each app has its own key (`lgn-`, from the app's page). Do not share one key across environments.

Read them from the environment: `LOGNORTH_URL` and `LOGNORTH_API_KEY`. Never write a key into source code. The agent key (`lgn-agent-`) is for reading and cannot send events.

The SDKs send events in every environment except `development` and `test`. Staging and preview report too. Do not add a production-only check.

## 3. Install and configure

### Go

```bash
go get github.com/karloscodes/lognorth-sdk-go
```

```go
lognorth.Configure(lognorth.Options{
    URL:         os.Getenv("LOGNORTH_URL"),
    APIKey:      os.Getenv("LOGNORTH_API_KEY"),
    Environment: os.Getenv("APP_ENV"), // "development" or "test" turns sending off
})
lognorth.IgnorePaths("/healthz", "/metrics")

http.ListenAndServe(":8080", lognorth.Middleware(mux))
```

If the project logs with slog: `slog.SetDefault(slog.New(lognorth.NewHandler()))`.

### Node.js / Bun

```bash
npm install github:karloscodes/lognorth-sdk-ts
```

```typescript
import LogNorth from '@karloscodes/lognorth-sdk'
import { middleware } from '@karloscodes/lognorth-sdk/express' // or /hono

LogNorth.config(process.env.LOGNORTH_URL!, process.env.LOGNORTH_API_KEY!) // environment defaults to NODE_ENV
app.use(middleware({ ignorePaths: ['/healthz', '/metrics'] }))
```

Next.js route handlers: `export const GET = withLogger()(handler)` from `@karloscodes/lognorth-sdk/next`. Pino: see the SDK README.

### Rails

```ruby
# Gemfile
gem "lognorth", github: "karloscodes/lognorth-sdk-rails"
```

Put the URL and key in credentials (`bin/rails credentials:edit`):

```yaml
lognorth:
  url: https://logs.yoursite.com
  api_key: lgn-...
```

The Railtie reads them and adds the middleware and the error subscriber. No initializer. It ignores `/up` by default. Only add what differs, for example `config.lognorth.ignored_paths = %w[/up /healthz]`.

### OpenTelemetry (any language)

Set the standard exporter variables. LogNorth accepts OTLP over HTTP/protobuf, the OTel default:

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=https://logs.yoursite.com/otel
OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer lgn-..."
```

### Any other language

No SDK and no OpenTelemetry: write a small client in the project's language. The protocol is at https://lognorth.com/docs/integrations/protocol/. Read it if you can; the rules below are the part that must not be skipped.

**Send.** `POST {LOGNORTH_URL}/api/v1/events/batch` with `Authorization: Bearer {LOGNORTH_API_KEY}`, `Content-Type: application/json`, and `{"events": [...]}`. At most 1,000 events and 4 MB per request. Each event: `message` (required), `timestamp` (RFC 3339 UTC, set when it happens), `duration_ms`, `trace_id`, `context`.

**Capture.** A middleware sends one event per request after the response: message `"METHOD /path → status"`, and `method`, `path` (no query string), `status` (a number), and `environment` in `context`. Unhandled errors add `error`, `error_class`, `error_file`, `error_line`, and `stack_trace` (about 20 frames). Jobs send one event when they end. Send nothing in `development` and `test`.

**Buffer.** Never send on the request path and never raise into the app. Keep events in memory; flush at 10 events or 5 seconds after the first; send error events at once; cap the buffer at 1,000 and drop the oldest; send one request at a time; flush on SIGTERM, SIGINT, and exit with a deadline of a few seconds.

**Retry.**

| Answer | Do |
|---|---|
| `201` | Done. Do not resend events listed in `"errors"`. |
| `400`, `401`, `404` | Do not retry. Drop the batch and write one line to stderr. |
| `413` | Split the batch in half and resend each half. |
| `429` | Keep the events, wait `Retry-After` seconds. |
| `5xx`, network error, timeout | Keep the events at the front of the buffer, back off 1s, 2s, 4s up to 60s with 20% jitter. |

Use a 5-second connect timeout and a 10-second request timeout.

Put the client in one small module the project owns, with no new dependency when the standard library can do HTTP and JSON. Test the parts that decide: what gets buffered, what is retried, and what is dropped, against a local HTTP server that answers 201, 429, 500, and 401.

## 4. Log what the middleware cannot see

Only where it earns its place: a payment, a signup, a background job.

```go
lognorth.Log("User signed up", map[string]any{"user_id": 123})
lognorth.Error("Checkout failed", err, map[string]any{"order_id": 42})
```

```typescript
LogNorth.log('User signed up', { user_id: 123 })
LogNorth.error('Checkout failed', err, { order_id: 42 })
```

```ruby
LogNorth.log("User signed up", user_id: 123)
LogNorth.error("Checkout failed", e, order_id: 42)
```

`Log` is batched (10 events or 5 seconds). `Error` is sent at once. The SDKs flush on SIGINT and SIGTERM.

## 5. Verify

1. Run the app with a non-development environment (for example `APP_ENV=staging`), because development does not send.
2. Make a request that reaches the middleware.
3. If the `lognorth` MCP tools are connected, call `search_logs` with `since: 5m` and the request's path. Seeing the event is the proof.
4. No MCP tools: ask the user to look at the app in LogNorth. Events appear within seconds.

Nothing arrived? Check, in order: the environment name, the URL and key variables, and whether the request path is in the ignore list.

Full options are in each SDK's README: [Go](https://github.com/karloscodes/lognorth-sdk-go), [Node](https://github.com/karloscodes/lognorth-sdk-ts), [Rails](https://github.com/karloscodes/lognorth-sdk-rails). For any other language, the [protocol](https://lognorth.com/docs/integrations/protocol/).
