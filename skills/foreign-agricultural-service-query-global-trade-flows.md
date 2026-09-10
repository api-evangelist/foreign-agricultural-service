---
name: Query global agricultural trade flows (GATS)
description: Pull US Census or UN ComTrade import, export and re-export flows from USDA FAS GATS, choosing the right country-coding scheme and HS level for the source you are querying.
api: openapi/foreign-agricultural-service-gats-api-openapi.yml
operations:
  - GATSData_GetCountries
  - GATSData_GetRegions
  - GATSData_GetCommodities
  - GATSData_GetHS6Commodities
  - GATSData_GetUnitsOfMeasure
  - GATSData_GetCustomsDistricts
  - GATSData_GetCensusExportDataReleaseInfo
  - GATSData_GetCensusImportDataReleaseInfo
  - GATSData_GetUNTradeExportDataReleaseInfo
  - GATSData_GetUNTradeImportDataReleaseInfo
  - GATSData_GetCensusExports
  - GATSData_GetCensusImport
  - GATSData_GetCensusReExports
  - GATSData_GetCustomsDistrictsExports
  - GATSData_GetCustomsDistrictsImports
  - GATSData_GetCustomsDistrictsReExports
  - GATSData_GetUNTradeExports
  - GATSData_GetUNTradeImports
  - GATSData_GetUNTradeReExports
generated: '2026-09-10'
method: generated
source: openapi/_original/foreign-agricultural-service-fas-open-data-swagger.json
---

# Query global agricultural trade flows (GATS)

GATS carries two different trade sources under one API, and **the single most important thing to get right is which one you are in**, because they use different country coding, different HS levels and different time granularity.

| | US Census half | UN ComTrade half |
|---|---|---|
| Path parameter | `{partnerCode}` (numeric) | `{reporterCode}` (alphabetic, e.g. `IN` for India) |
| Time granularity | `{year}/{month}` | `{year}` only |
| HS level | HS10 (`GATSData_GetCommodities`) | HS6 (`GATSData_GetHS6Commodities`) |
| Operations | `GATSData_GetCensusExports`, `GATSData_GetCensusImport`, `GATSData_GetCensusReExports`, and the three `GATSData_GetCustomsDistricts*` | `GATSData_GetUNTradeExports`, `GATSData_GetUNTradeImports`, `GATSData_GetUNTradeReExports` |

The API does not warn you if you mix them. Passing a numeric partner code where a reporter code belongs is not a documented error.

## Auth

`API_KEY: <your key>` header on every request. Base URL `https://apps.fas.usda.gov/OpenData`. Missing key → `403 "Bad API Key"`; malformed key → `500 {"message":"An error has occurred."}`. Treat a 500 as a credential problem first.

## Steps

1. **Decide your source** using the table above, then stay in that column for the whole query.
2. **Resolve codes.** `GATSData_GetCountries` (`GET /api/gats/countries`) for the partner/reporter dimension, `GATSData_GetRegions` (`GET /api/gats/regions`) to group them.
3. **Resolve the commodity at the right level.** `GATSData_GetHS6Commodities` (`GET /api/gats/HS6Commodities`) to correlate UN ComTrade records — the operation says so directly. `GATSData_GetCommodities` (`GET /api/gats/commodities`) for the HS10 level, which also carries both the Census and the FAS unit-of-measure ids.
4. **Check release timing before you query.** Census: `GATSData_GetCensusExportDataReleaseInfo` (`GET /api/gats/census/data/exports/dataReleaseDates`) and `GATSData_GetCensusImportDataReleaseInfo`. UN ComTrade: `GATSData_GetUNTradeExportDataReleaseInfo` (`GET /api/gats/UNTrade/data/exports/dataReleaseDates`) and `GATSData_GetUNTradeImportDataReleaseInfo`.
5. **Pull the flow.**
   - Census, monthly: `GET /api/gats/censusExports/partnerCode/{partnerCode}/year/{year}/month/{month}` (and `censusImports`, `censusReExports`).
   - By US point of entry/exit: `GET /api/gats/customsDistrictExports/partnerCode/{partnerCode}/year/{year}/month/{month}` (and the imports and re-exports variants). Resolve district codes with `GATSData_GetCustomsDistricts` — one partner's trade in a year may arrive through several districts, so these rows sum to the national figure rather than duplicating it.
   - UN ComTrade, annual: `GET /api/gats/UNTradeExports/reporterCode/{reporterCode}/year/{year}` (and `UNTradeImports`, `UNTradeReExports`).
6. **Label quantities** with `GATSData_GetUnitsOfMeasure` (`GET /api/gats/unitsOfMeasure`). GATS commodity records carry two unit ids — a Census one and a FAS one. Pick the one matching your source and say which you used.

## Rules an agent must follow here

- **Re-exports are a third flow, not a subset.** Exports, imports and re-exports are separate operations. Do not add re-exports into exports unless you mean to, and say so when you do.
- **No pagination, no filters, no sorting.** Every parameter in this API is a path parameter. Month-by-month means one call per month.
- **Do not join across datasets casually.** GATS commodity/country codes are not ESR or PSD codes. There is no crosswalk published between the FAS codes used by ESR/PSD, the HS10/HS6 codes used by GATS, and the Census partner vs UN reporter country schemes — you must build and state your own.
- **Read-only.** No writes exist, so no idempotency key and nothing to reverse.
