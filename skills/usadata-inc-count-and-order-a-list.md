---
name: Count and order a USADATA mailing list
description: Discover a data table, build an audience, run a count, price it, place the order and collect the delivered file from the USADATA Leads Engine SOAP service.
api: wsdl/usadata-inc-leads-engine.wsdl
endpoint: https://leadsengine.usadata.com/service.asmx
operations: [getTableList, getTableDefinition, getSynchronousCount, getMatrixReport, getOrderPrice, submitOrder, getOrderStatus, cancelOrder]
generated: '2026-08-13'
method: generated
source: wsdl/usadata-inc-leads-engine.wsdl
---

# Count and order a USADATA mailing list

This is the primary money path on the Leads Engine service. Every operation
named here exists verbatim in the published WSDL at
`https://leadsengine.usadata.com/service.asmx?WSDL`.

## Before you start

- Transport is **SOAP 1.1 or 1.2 over HTTPS** to a single endpoint,
  `https://leadsengine.usadata.com/service.asmx`. There is no REST projection.
- `SOAPAction` is `http://Usadata/Services/Service/<operation>`.
- **Credentials go in the body, not a header.** Put a `Login` element
  (`UserID`, `Password`, `ClientID`, namespace
  `http://Usadata.com/Services/LeadsEngine/Login`) as the first child of every
  request except `ping` and `getVersion`. Credentials are issued by USADATA;
  there is no self-service sign-up.
- **HTTP 200 does not mean success.** Read `ReturnCode` and `ErrorMessage` off
  every result object before trusting it. See
  `errors/usadata-inc-error-codes.yml`.

## Steps

1. **Pick a data source.** Call `getTableList`. It returns `TableListResult`
   with `TableList` — `Table {ID, Description}` entries such as consumer,
   business and occupant files. Choose the `ID`.

2. **Learn what you can filter on.** Call `getTableDefinition` with that
   `TableName`. It returns `ArrayOfColumn`; each `Column {ID, Description, Type,
   Values}` carries its enumerated `ColumnValue` list. Only build selections
   from values that appear here — do not guess codes.

3. **Narrow the geography if you need to.** `getCountiesInState`,
   `getCitiesInState`, `getZipsInState` and `getZipsInCity` return
   `GeographyElementResult.Elements` you can drop straight into the request.
   `lookupAddress` resolves a loose address to `AddressDefinition` candidates
   with a `Point {X, Y}` for radius or drive-time targeting.

4. **Count the audience.** Build a `CountRequest` (`TableName`,
   `AreaCriteria`, `GeographicCriteria`, `SelectionCriteria`, optional
   `Suppression` / `SuppressionPriorOrder`, optional polygon geometry) and call
   `getSynchronousCount`. It blocks until the count finishes and returns
   `SynchronousCountResult {ReturnCode, ErrorMessage, CountID, CountTotal,
   BreakdownResults}`.
   *Use `submitAsynchronousCount` + `getAsynchronousCountStatus` instead for
   large or slow counts — see the async-counts skill.*

5. **Optional: cross-tab it.** `getMatrixReport` takes the `CountID` plus a
   `RowName` and `ColumnName` and returns the matrix report.

6. **Price it before you buy.** Build an `OrderRequest` (`CountID`,
   `InvoiceReferenceID`, `DesiredQuantity`, `AvailableQuantity`,
   `SelectionMethod`, `FileFormat`, `Usage`, `OutputColumns`) and call
   `getOrderPrice`. It returns `PriceRequestResult.OrderPrice` as a double.
   `Usage` is a licence term — `Single`, `Multiple` or `Analytical` — not a
   formatting flag. Pick the one you are actually licensed for.

7. **Place the order.** Call `submitOrder` with the same `OrderRequest`. It
   returns `OrderRequestResult {OrderID, ErrorMessage, ReturnCode}`.

   > **This operation is not idempotent.** There is no idempotency key in the
   > contract. `InvoiceReferenceID` is a client-supplied reference, and the
   > contract does not promise it is enforced unique — so a blind retry after a
   > timeout can place a second order. On a timeout, do **not** resend: poll
   > `getOrderStatus` first, or reconcile with USADATA.

8. **Poll to delivery.** Call `getOrderStatus` with the `OrderID`. Read
   `CurrentStatus` — `Queued`, `Processing`, `Complete`, `Cancelled`, `OnHold`
   or `Error`. On `Complete`, `OrderStatusResult.URL` is the download. `OnHold`
   and `Error` need a human at USADATA.

9. **Back out if you must.** `cancelOrder` with the `OrderID` cancels a pending
   order and returns `OrderCancelResult`.

## Rules an agent must not break

- Never call `submitOrder` without a successful `getOrderPrice` on the same
  `OrderRequest` first — the price is quantity- and filter-dependent and is not
  published anywhere else.
- Never retry `submitOrder` or `submitSaturationOrder` automatically.
- No rate limits are published (`rate-limits/usadata-inc-rate-limits.yml`).
  Pace polling conservatively; there is no `Retry-After` and no backoff signal
  to read.
