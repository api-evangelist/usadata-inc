---
name: Resolve USADATA geography and addresses
description: Use the anonymous diagnostics and the geography lookup operations on the USADATA Leads Engine to resolve states, counties, cities, ZIPs and street addresses into the codes an audience count needs.
api: wsdl/usadata-inc-leads-engine.wsdl
endpoint: https://leadsengine.usadata.com/service.asmx
operations: [ping, getVersion, getCountiesInState, getCitiesInState, getZipsInState, getZipsInCity, lookupAddress, getAvailablePriorOrders]
generated: '2026-08-13'
method: generated
source: wsdl/usadata-inc-leads-engine.wsdl
---

# Resolve USADATA geography and addresses

Geography is where most Leads Engine integrations go wrong: a `CountRequest`
wants codes, not place names. These operations turn user-facing geography into
the identifiers the count engine accepts.

## Check the service first (no credentials needed)

Two operations take **no `Login` element at all** and are callable anonymously:

- `ping` → boolean. Liveness. Verified live 2026-08-13, returned `true`.
- `getVersion` → string. The running build. Verified live 2026-08-13, returned
  `3.0.9719.28036`.

There is no status page for this service, so `ping` is your availability check.

## Resolve geography

Every one of these takes a `Login` element plus one argument and returns a
`GeographyElementResult {Elements, ErrorMessage, ReturnCode}`, where `Elements`
is an `ArrayOfColumnValue` — `ColumnValue {ID, Description}` pairs. **Use the
`ID`, show the `Description`.**

- `getCountiesInState(State)` → FIPS county codes for a state.
- `getCitiesInState(State)` → cities in a state. The `ID` you get back is the
  `CityID` the next call wants.
- `getZipsInState(State)` → ZIP codes in a state.
- `getZipsInCity(CityID)` → ZIP codes in a city. Feed it a `CityID` from
  `getCitiesInState`, not a city name.

## Resolve a street address

`lookupAddress` takes an `AddressRequest {Address, City, State, Zip}` and
returns `FindAddressResult.Candidates` — an array of `AddressDefinition`
carrying `Address`, `City`, `State`, `Zip`, a `Location` `Point {X, Y}`, a
`Radius`, a `Quantity` and a `KeyCode`.

That geocoded `Point` is what makes radius and drive-time targeting work:
drop the resolved `AddressDefinition` into `CountRequest.AreaCriteria`
(`AreaRequest.Address` or `AreaRequest.Addresses`), or into
`AreaRequest.DriveTimeAddress` as a `DriveTimeDefinition {Location,
LocationType, DriveTime, Centroid}` for drive-time rings.

Address resolution is ambiguous by design — it returns *candidates*. Never
silently pick the first one for a user-facing order; surface the list.

## Suppress people you already mailed

`getAvailablePriorOrders(AppID, AccountID, DataSource, Interval)` returns a
`SuppressionPriorOrdersSelection` listing prior orders you can suppress
against. Put the selected order ids in `CountRequest.SuppressionPriorOrder`
before counting, not after ordering.

> Note: this is the one operation that does **not** take the `Login` element —
> it takes `AppID` and `AccountID` as plain strings instead. That is an
> inconsistency in the published contract, recorded in
> `authentication/usadata-inc-authentication.yml`. Treat those values as
> credentials anyway.

## Rules an agent must not break

- Read `ReturnCode` / `ErrorMessage` on every result. HTTP 200 is not success.
- Never invent a county, city or ZIP code. Every value must come back from one
  of these operations or from `getTableDefinition`.
