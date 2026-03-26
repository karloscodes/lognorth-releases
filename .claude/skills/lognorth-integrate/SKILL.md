---
name: lognorth-integrate
description: Use when setting up LogNorth in a project — installs SDK, configures middleware, and wires up error reporting. Trigger when user says "add lognorth", "set up logging", or "integrate lognorth".
---

# LogNorth Integration

Set up LogNorth error tracking and logging in the user's project. Detect the stack, install the right SDK, configure middleware, and verify it works.

## Step 1: Detect the Stack

Look at the project files to determine the language and framework:

| File | Stack |
|------|-------|
| `go.mod` | Go |
| `package.json` | Node.js / Bun |
| `Gemfile` | Ruby on Rails |

If the project uses OpenTelemetry already (look for `@opentelemetry` in package.json or otel imports), suggest the OTel integration instead of an SDK.

## Step 2: Get Connection Details

Ask the user for:
- **LogNorth URL** (e.g., `https://logs.yoursite.com`)
- **API key** (starts with `lgn-`, created in Settings)

If these are already in environment variables or config files, use those.

## Step 3: Install and Configure

### Go

```bash
go get github.com/karloscodes/lognorth-sdk-go
```

Add to `main.go` (or wherever the app starts):

```go
import lognorth "github.com/karloscodes/lognorth-sdk-go"

func main() {
    lognorth.Config("https://logs.yoursite.com", "your-api-key")

    // ... rest of app
}
```

**Add middleware** (wrap the HTTP handler):

```go
http.ListenAndServe(":8080", lognorth.Middleware(http.DefaultServeMux))
```

**With slog** (optional, if the project uses slog):

```go
slog.SetDefault(slog.New(lognorth.NewHandler()))
```

**Ignore noisy routes:**

```go
lognorth.IgnorePaths("/healthz", "/_health", "/metrics")
```

### Bun / Node.js

```bash
npm install github:karloscodes/lognorth-sdk-ts
```

Add configuration early in app startup:

```typescript
import LogNorth from '@karloscodes/lognorth-sdk'

LogNorth.config('https://logs.yoursite.com', 'your-api-key')
```

**Add middleware** based on framework:

```typescript
// Express
import { middleware } from '@karloscodes/lognorth-sdk/express'
app.use(middleware())

// Hono
import { middleware } from '@karloscodes/lognorth-sdk/hono'
app.use(middleware())

// Next.js (wrap route handlers)
import { withLogger } from '@karloscodes/lognorth-sdk/next'
export const GET = withLogger()(handler)
```

**Ignore noisy routes:**

```typescript
app.use(middleware({ ignorePaths: ['/healthz', '/_health', '/metrics'] }))
```

**With Pino** (optional, if the project uses Pino):

```typescript
import pino from 'pino'
import { transport } from '@karloscodes/lognorth-sdk/pino'

const logger = pino({ level: 'info' }, transport())
app.use(middleware(logger))
```

### Rails

```ruby
# Gemfile
gem "lognorth", github: "karloscodes/lognorth-sdk-rails"
```

```bash
bundle install
```

**Option A — Rails credentials (recommended):**

```bash
bin/rails credentials:edit
```

```yaml
lognorth:
  url: https://logs.yoursite.com
  api_key: your-api-key
```

The Railtie auto-configures from credentials. No initializer needed.

**Option B — Environment variables:**

```ruby
# config/initializers/lognorth.rb
LogNorth.config(
  ENV["LOGNORTH_URL"],
  ENV["LOGNORTH_API_KEY"]
)
```

**Configure in `config/application.rb`:**

```ruby
config.lognorth.enabled = Rails.env.production?
config.lognorth.middleware = true
config.lognorth.error_subscriber = true
config.lognorth.ignored_paths = %w[/healthz /_health /up]
```

The middleware and error subscriber are enabled by default.

### OpenTelemetry (any language)

If the project already uses OpenTelemetry, just point the OTLP log exporter at LogNorth:

```
POST https://logs.yoursite.com/api/v1/otel/logs
Authorization: Bearer your-api-key
Content-Type: application/json
```

OTLP JSON format only. No protobuf. See the docs for language-specific OTel setup.

## Step 4: Add Manual Logging (Optional)

Show the user how to add custom log calls beyond automatic middleware logging:

**Go:**
```go
lognorth.Log("User signed up", map[string]any{"user_id": 123})
lognorth.Error("Checkout failed", err, map[string]any{"order_id": 42})
```

**TypeScript:**
```typescript
LogNorth.log('User signed up', { user_id: 123 })
LogNorth.error('Checkout failed', err, { order_id: 42 })
```

**Rails:**
```ruby
LogNorth.log("User signed up", user_id: 123)
LogNorth.error("Checkout failed", error: e, order_id: 42)
```

## Step 5: Verify

After setup, suggest the user:
1. Start the app
2. Make a few requests
3. Check LogNorth — events should appear within seconds

## Behavior Notes

- `Log()` calls are batched (10 events or 5 seconds). `Error()` calls are sent immediately.
- Middleware auto-generates a `trace_id` per request (or uses `X-Trace-ID` from incoming headers).
- All logs within the same request automatically share the same `trace_id`.
- SDKs auto-flush on SIGINT/SIGTERM — no manual shutdown needed.
- Error detection is automatic on the server: `context.error`, `context.error_class`, or `context.status >= 500` marks an event as an error.
- Don't store the API key in source code. Use environment variables or credentials.
