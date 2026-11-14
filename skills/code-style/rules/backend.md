# Backend rules

These rules apply to API routes, middleware, background jobs, and database writes.
Examples use Hono and Zod. Use the framework and validator that the repo already has.

## Validation

### BE1. Validate every input with a schema validator

Validate the body, query, params, and headers of each route. Use the validator in the repo (Zod, ArkType, Valibot). If the repo has none, ask before you add one. Do not validate with hand-written `if` checks.

Make schemas strict: set types, lengths, ranges, and formats. Reject unknown fields when they have no use.

```ts
// ✅
const createOrderSchema = z.strictObject({
  productID: z.uuid(),
  quantity: z.int().min(1).max(MAX_ORDER_QUANTITY),
  email: z.email(),
})

// ❌
if (!body.productID || typeof body.quantity !== "number") {
  return c.json({ error: "Bad input" }, 400)
}
```

### BE2. Validate in middleware, in the same process

Run validation in middleware before the handler. The handler then gets typed, valid data. Do not call another service to validate. A network hop adds latency to each request and increases TTFB.

```ts
// ✅
app.post("/orders", zValidator("json", createOrderSchema, sendValidationError), createOrder)

// ❌
app.post("/orders", async (c) => {
  const isValid = await fetch(VALIDATION_SERVICE_URL, { method: "POST", body: await c.req.text() })
  // ...
})
```

### BE3. Return 400 with each field and its reason

When validation fails, return `400`. List each failed field and the reason. Use the validator's own error formatter. Do not write your own loop over issues.

| Validator | Formatter |
|---|---|
| Zod 4 | `z.flattenError(error)` |
| Zod 3 | `error.flatten()` |
| ArkType | `errors.summary`, or `errors.byPath` |

```ts
// ✅
function sendValidationError(result: ValidationResult, c: Context) {
  if (result.success) {
    return
  }

  const { fieldErrors } = z.flattenError(result.error)

  return c.json({ error: "Invalid request", fields: fieldErrors }, 400)
}
// -> { "error": "Invalid request", "fields": { "quantity": ["Too small: expected number to be >=1"] } }

// ❌
return c.json({ error: "Bad request" }, 400)
```

## Responses and security

### BE4. Return the correct status code

| Case | Status |
|---|---|
| Invalid input, including business rule errors. Do not use `422` | `400` |
| No auth or bad auth | `401` |
| Auth is valid, access is not allowed | `403` |
| Resource does not exist | `404` |
| Conflict, for example a duplicate or a stale version | `409` |
| Rate limit | `429` |
| Unexpected error | `500` |

Do not return `200` with an error in the body.

### BE5. Do not leak internals in errors

For a `500`, return a generic message and the request ID. Log the details on the server. Do not send stack traces, SQL, or internal IDs to the client.

```ts
// ✅
return c.json({ error: "Internal error", requestID }, 500)

// ❌
return c.json({ error: error.message, stack: error.stack }, 500)
```

### BE6. Check access on each route

- Authenticate before validation work that costs time or money.
- Check that the user owns or can access the resource. A valid ID is not permission.
- Use parameterized queries or the ORM. Do not build SQL from strings.
- Apply a rate limit to routes that write, send messages, or cost money.

## Data safety

### BE7. Use a transaction for related writes

When one request writes more than one row or table, put the writes in one transaction. Then a failure leaves no partial data.

```ts
// ✅
const order = await db.transaction(async (tx) => {
  const createdOrder = await tx.insert(orders).values(newOrder).returning()
  await tx.update(stock).set({ quantity: sql`${stock.quantity} - ${newOrder.quantity}` })

  return createdOrder
})

// ❌
const order = await db.insert(orders).values(newOrder).returning()
await db.update(stock).set({ quantity: sql`${stock.quantity} - ${newOrder.quantity}` })
```

Do not call external services inside a transaction. A slow call holds the database lock.

### BE8. Make write routes idempotent when a retry can cause harm

When a client can send the same request again (payments, orders, emails), accept an `Idempotency-Key` header. Store the key with the result. On a repeat key, return the stored result and do not run the work again.

```ts
// ✅
const idempotencyKey = c.req.header("Idempotency-Key")
const storedResponse = await getStoredResponse(idempotencyKey)

if (storedResponse) {
  return c.json(storedResponse.body, storedResponse.status)
}
```

A unique constraint in the database is often enough. Use it before you build a key store.

## Failure

### BE9. Plan for failure at each step

Before you finish a route, list each step that can fail. For each step, decide:

- What state is left if it fails here?
- Can the client retry safely? See [BE8](#be8-make-write-routes-idempotent-when-a-retry-can-cause-harm).
- Does slow work belong in a queue instead of the request?
- Does each external call have a timeout?
- Does a retry use backoff and a maximum attempt count?

```ts
// ✅
const order = await createOrder(input)
await queue.send("order.created", { orderID: order.id })

return c.json(order, 201)

// ❌
const order = await createOrder(input)
await sendConfirmationEmail(order)
await syncToERP(order)

return c.json(order, 201)
```

To send a message only when the write succeeds, use an outbox table in the same transaction.

### BE10. Handle errors in one place

Use one error handler for the app. Map known errors to status codes there. Do not wrap each handler in `try/catch` only to log and return `500`.

```ts
// ✅
app.onError((error, c) => {
  if (error instanceof NotFoundError) {
    return c.json({ error: error.message }, 404)
  }

  logger.error({ error, requestID: c.get("requestID") })

  return c.json({ error: "Internal error", requestID: c.get("requestID") }, 500)
})
```

## Observability

### BE11. Log one canonical event per request

Emit one wide, structured log event at the end of each request. Add fields to it during the request. Do not scatter many small log lines.

Minimum fields: `requestID`, `traceID`, `method`, `route`, `status`, `durationMs`, `userID` when known, and `error` when there is one.

```ts
// ✅
logger.info({
  requestID,
  traceID,
  method: "POST",
  route: "/orders",
  status: 201,
  durationMs: 84,
  userID,
  orderID: order.id,
})

// ❌
console.log("creating order")
console.log("order created", order.id)
```

Do not log secrets, tokens, passwords, or full payment data.

### BE12. Trace calls across services

Propagate the trace context to each outgoing call. Add a span around slow or external work: database queries, HTTP calls, and queue sends. Use OpenTelemetry or the tracing tool in the repo. If the repo has none, suggest one and do not add it without approval.
