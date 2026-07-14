---
name: logging-best-practices
description: >
  Implements structured logging using wide events (canonical log lines) with high-cardinality context.
  Designs canonical log line schemas, configures single-logger patterns, builds wide event middleware,
  and adds business and environment context to log output.
  Use when writing logging code, reviewing log statements, designing a logging strategy,
  setting up log infrastructure, adding observability, structuring logs for analytics,
  or replacing scattered console.log calls with queryable structured events.
  Not for application-level error handling or monitoring/alerting configuration.
license: MIT
metadata:
  author: boristane
  version: "1.1.0"
  category: observability
---

# Logging Best Practices

## Output Format

Produces structured JSON log events containing request context, business context, environment metadata, and timing — one wide event per request per service.

## Workflow

1. **Configure single logger** — Create one logger instance at startup and import it everywhere; put environment characteristics (commit hash, service version, region, instance ID) in its base fields so every event carries them automatically ([full setup](references/structure.md))
2. **Add wide event middleware** — Wrap all request handlers to initialise the event, capture timing and status, and emit exactly once in a `finally` block ([full middleware](references/wide-events.md))
3. **Enrich with business context** — In each handler, add user, cart, feature flag, and domain-specific fields to the event ([field catalogue](references/context.md))
4. **Validate** — Confirm each request produces exactly one structured JSON event with 20+ fields including high-cardinality IDs; check against [common pitfalls](references/pitfalls.md)

### Steps 1–2: logger and middleware skeleton

```typescript
// lib/logger.ts — one logger, environment attached to every event
import pino from 'pino';

export const logger = pino({
  base: {
    service: process.env.SERVICE_NAME,
    version: process.env.SERVICE_VERSION,
    commit_hash: process.env.COMMIT_SHA,
    region: process.env.AWS_REGION,
    instance_id: process.env.HOSTNAME,
  },
});

// middleware/wideEvent.ts — one event per request, emitted in finally
app.use('*', async (c, next) => {
  const startTime = Date.now();
  const wideEvent: Record<string, unknown> = {
    request_id: c.get('requestId') ?? crypto.randomUUID(),
    method: c.req.method,
    path: c.req.path,
  };
  c.set('wideEvent', wideEvent);
  try {
    await next();
    wideEvent.status_code = c.res.status;
    wideEvent.outcome = c.res.status < 400 ? 'success' : 'error';
  } catch (error) {
    wideEvent.status_code = 500;
    wideEvent.outcome = 'error';
    wideEvent.error = { message: error.message, type: error.name };
    throw error;
  } finally {
    wideEvent.duration_ms = Date.now() - startTime;
    logger.info(wideEvent);
  }
});
```

### Step 3: handler enrichment

```typescript
app.post('/checkout', async (c) => {
  const wideEvent = c.get('wideEvent');

  const user = await getUser(c.get('userId'));
  wideEvent.user = {
    id: user.id,
    subscription: user.subscription,
    account_age_days: user.accountAgeDays,
  };

  const cart = await getCart(user.id);
  wideEvent.cart = { total_cents: cart.total, item_count: cart.items.length };
  wideEvent.feature_flags = { new_checkout_flow: flags.newCheckoutFlow };

  return c.json({ success: true });
});
// Middleware owns timing, status, environment, and emission — handlers own business context only.
```

## Core Rules

### 1. Wide Events (CRITICAL)

Emit **one context-rich event per request per service**, built up through the request lifecycle and emitted once in a `finally` block. Consolidate scattered per-request `console.log()` calls into this single structured event. ([references/wide-events.md](references/wide-events.md))

### 2. High Cardinality & Dimensionality (CRITICAL)

Include fields with high cardinality (user IDs, request IDs — millions of unique values) and high dimensionality (20+ fields per event). This enables querying by specific users and answering questions you haven't anticipated yet. ([references/context.md](references/context.md))

### 3. Business Context (CRITICAL)

Include business context in every event: user subscription tier, cart value, feature flags, account age. The goal is "a premium customer couldn't complete a $2,499 purchase", not just "checkout failed". ([references/context.md](references/context.md))

### 4. Request Correlation (HIGH)

Propagate a `request_id` across every service hop (e.g. an `x-request-id` header) so one query reconstructs a request's full journey through the system. ([references/wide-events.md](references/wide-events.md))

### 5. Structure & Consistency (HIGH)

- JSON format for every log line, each field queryable — a message tempting you toward `console.log('something happened')` becomes fields on the wide event instead
- Consistent field names across services (always `user_id`, never `userId`)
- Two log levels only: `info` and `error` — context that would go in a debug log goes in the wide event instead

([references/structure.md](references/structure.md))

## References

- [Logging Sucks](https://loggingsucks.com)
- [Observability Wide Events 101](https://boristane.com/blog/observability-wide-events-101/)
- [Stripe — Canonical Log Lines](https://stripe.com/blog/canonical-log-lines)
