---
name: Run large USADATA counts asynchronously
description: Queue an audience or saturation count on the USADATA Leads Engine, poll it to a terminal state, and carry the CountID forward into pricing and ordering.
api: wsdl/usadata-inc-leads-engine.wsdl
endpoint: https://leadsengine.usadata.com/service.asmx
operations: [submitAsynchronousCount, getAsynchronousCountStatus, submitAsynchronousSaturationCount, getAsynchronousSaturationCountStatus, getSynchronousCounts, ping]
generated: '2026-08-13'
method: generated
source: wsdl/usadata-inc-leads-engine.wsdl
---

# Run large USADATA counts asynchronously

The Leads Engine offers a synchronous and an asynchronous form of every count.
The synchronous ones block until the count finishes, which is fine for a small
selection and wrong for a national file. Use this path when the count is large.

## The pattern

Submit-then-poll. There is **no callback, no webhook and no push notification**
on this service; polling is the only completion signal.

1. **Submit.** Call `submitAsynchronousCount` with a `Login` element and a
   `CountRequest`. It returns `AsynchronousCountResult {ReturnCode,
   ErrorMessage, CountID}`. Keep the `CountID` — it is the correlation id for
   everything downstream.

2. **Poll.** Call `getAsynchronousCountStatus` with the `CountID`. It returns
   `AsynchronousCountStatusResult`:
   - `Status` — a `CountStatus` enum: `Processing`, `Complete`, `Error`
   - `StatusPercent` — progress 0–100
   - `StatusMessage` — free text
   - `CountTotal` and `BreakdownResults` once complete

3. **Stop on a terminal state.** `Complete` and `Error` are terminal.
   `Processing` means keep polling. Use `StatusPercent` to space your polls —
   it is the only progress signal the contract gives you.

4. **Carry the CountID forward.** A `Complete` count's `CountID` is the input to
   `getMatrixReport`, `getOrderPrice` and `submitOrder`. See the
   count-and-order skill.

## Saturation (occupant / carrier-route) counts

Identical shape, different operations and a carrier-route result set:

- `submitAsynchronousSaturationCount` → `AsynchronousSaturationCountResult`
- `getAsynchronousSaturationCountStatus` → `AsynchronousSaturationCountStatusResult`,
  which adds `CarrierRoutes` — `CarrierRouteResult {ZipCode, Crrt, CountHomes,
  CountApartments, CountBusinesses}`
- Order it with `getSaturationOrderPrice` then `submitSaturationOrder`, passing
  `CarrierRouteRequest` entries that select homes / apartments / businesses per
  route.

## Batching many small counts

`getSynchronousCounts` takes an `ArrayOfCountRequest` and returns an
`ArrayOfSynchronousCountResult` in one call. Prefer it over N synchronous calls
when you are scoring several audience variants against each other.

## Rules an agent must not break

- **Check `ReturnCode` on every response.** HTTP 200 is not success on this
  service.
- Do not treat `Error` as retryable without reading `StatusMessage` — the
  contract publishes no code registry, so the English string is all you get.
- No rate limits are published. If you need a liveness check between polls,
  `ping` is anonymous and returns a boolean.
