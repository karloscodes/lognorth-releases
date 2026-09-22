---
name: lognorth-integrate
description: Use when setting up LogNorth in a project — installs the SDK, adds the middleware, and confirms events arrive. Trigger when the user says "add lognorth", "set up logging", or "integrate lognorth".
---

# LogNorth Integration

Add LogNorth to the user's project, then prove it works by reading the events back. The job is done when you see the event, not when the code compiles.

## 1. Detect the stack

| File | Stack |
|------|-------|
| `go.mod` | Go |
| `package.json` | Node.js / Bun |
| `Gemfile` | Rails |

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

Full options are in each SDK's README: [Go](https://github.com/karloscodes/lognorth-sdk-go), [Node](https://github.com/karloscodes/lognorth-sdk-ts), [Rails](https://github.com/karloscodes/lognorth-sdk-rails).
