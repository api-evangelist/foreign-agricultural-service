---
name: Track weekly US export sales for a commodity
description: Pull USDA FAS Export Sales Reporting (ESR) data for one commodity — to all destinations or to one country — for a market year, resolving the commodity, country and region codes first.
api: openapi/foreign-agricultural-service-esr-api-openapi.yml
operations:
  - ESRData_GetCommodities
  - ESRData_GetCountries
  - ESRData_GetRegions
  - ESRData_GetUnitsOfMeasure
  - ESRData_GetDataReleaseInfo
  - ESRData_GetAllCountriesData
  - ESRData_GetCountryData
generated: '2026-09-10'
method: generated
source: openapi/_original/foreign-agricultural-service-fas-open-data-swagger.json
---

# Track weekly US export sales for a commodity

ESR is USDA FAS's weekly record of US agricultural export sales: what was sold, to whom, and against which market year. Every operation is a GET. Base URL `https://apps.fas.usda.gov/OpenData`.

## Before you call anything

Send your key on **every** request as the `API_KEY` header — not a query parameter, not a bearer token:

```
API_KEY: <your key>
```

Get one free at https://apps.fas.usda.gov/opendatawebv2/#/signup. If you skip it you get `403` with the body `"Bad API Key"` — a bare JSON string, so do not call `.json()["message"]` on it. If the key is malformed you get `500 {"message":"An error has occurred."}`. **A 500 from this API is more often a bad credential than a server fault: check the key before you retry or back off.**

## Steps

1. **Resolve the commodity code.** `ESRData_GetCommodities` — `GET /api/esr/commodities`. ESR uses FAS commodity codes, not HS codes; the contract's own example is `104` for White Wheat. Do not reuse a code from the GATS dataset here, it is a different vocabulary.
2. **Resolve the destination, if you want one country.** `ESRData_GetCountries` — `GET /api/esr/countries`. Also FAS codes; the contract's example is `1220` for Canada. To group destinations, call `ESRData_GetRegions` (`GET /api/esr/regions`) and join region name onto the country records — that is exactly what the operation says it is for.
3. **Check what has actually been released.** `ESRData_GetDataReleaseInfo` — `GET /api/esr/datareleasedates`. This returns which commodity / market-year combinations have data and when they were last released. **Call it first, not after an empty result:** there is no documented error for asking about an unreleased slice, and the operation exists specifically so you do not have to guess. The same response is how you learn that export numbers get revised across multiple years for a commodity — cached ESR figures go stale, they do not just get appended to.
4. **Pull the data.**
   - All destinations: `ESRData_GetAllCountriesData` — `GET /api/esr/exports/commodityCode/{commodityCode}/allCountries/marketYear/{marketYear}`
   - One destination: `ESRData_GetCountryData` — `GET /api/esr/exports/commodityCode/{commodityCode}/countryCode/{countryCode}/marketYear/{marketYear}`
5. **Label the quantities.** `ESRData_GetUnitsOfMeasure` — `GET /api/esr/unitsOfMeasure`. Quantities come back as numbers against a unit id; resolve it before you present or compare anything.

## Rules an agent must follow here

- **There is no pagination and no filtering.** No `limit`, `offset`, `page` or query parameter exists anywhere in this API. `allCountries` returns the entire slice in one response — size it for that.
- **Cache the reference lookups, not the data.** Commodities, countries, regions and units change rarely; export figures are revised.
- **Ask for JSON or XML deliberately.** Every operation declares `application/json`, `text/json`, `application/xml` and `text/xml`. Without an `Accept` header you get JSON.
- **Nothing here writes.** There is no create, update or delete operation in the whole API, so there is no idempotency key to send and nothing to undo. Retrying a GET is always safe once the credential is right.
- **You have no rate-limit signal.** No `RateLimit-*` or `Retry-After` header is returned and no limit is published. Pace yourself conservatively rather than relying on the API to tell you when to stop.
