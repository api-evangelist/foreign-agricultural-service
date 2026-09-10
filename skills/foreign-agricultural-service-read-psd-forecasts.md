---
name: Read world production and supply forecasts (PSD)
description: Pull USDA FAS Production, Supply and Distribution forecasts for a commodity — world, all countries, or one country — for a market year, and resolve the attribute ids that label each measure.
api: openapi/foreign-agricultural-service-psd-api-openapi.yml
operations:
  - PSDData_GetCommodities
  - PSDData_GetCommodityAttributes
  - PSDData_GetCountries
  - PSDData_GetRegions
  - PSDData_GetUnitsOfMeasure
  - PSDData_GetDataReleaseInfo
  - PSDData_GetWorldCommodityDataByYear
  - PSDData_GetCommodityDataByYear
  - PSDData_GetCountryCommodityDataByYear
generated: '2026-09-10'
method: generated
source: openapi/_original/foreign-agricultural-service-fas-open-data-swagger.json
---

# Read world production and supply forecasts (PSD)

PSD is USDA's forecast of world agricultural commodity supply and use — production, exports, imports, ending stocks and the rest — by country and market year. These are **forecasts that get revised**, not settled history. Every operation is a GET. Base URL `https://apps.fas.usda.gov/OpenData`.

## Auth

`API_KEY: <your key>` header on every request. Missing key → `403 "Bad API Key"`; malformed key → `500 {"message":"An error has occurred."}`.

## Steps

1. **Resolve the commodity.** `PSDData_GetCommodities` — `GET /api/psd/commodities`. PSD uses FAS commodity codes, the same family as ESR and *not* the HS codes GATS uses.
2. **Resolve the attribute ids — do this before you read any figure.** `PSDData_GetCommodityAttributes` — `GET /api/psd/commodityAttributes`. PSD forecast rows carry an attribute id, not an attribute name. Without this lookup a row of numbers is unlabelled and you cannot tell production from ending stocks. This is the step most likely to be skipped and the one most likely to produce a wrong answer.
3. **Check what has been released for that commodity.** `PSDData_GetDataReleaseInfo` — `GET /api/psd/commodity/{commodityCode}/dataReleaseDates`. Note that this release lookup is scoped *per commodity*, unlike ESR's, so it takes a `{commodityCode}` in the path.
4. **Pull the forecast at the scope you need.**
   - World roll-up: `PSDData_GetWorldCommodityDataByYear` — `GET /api/psd/commodity/{commodityCode}/world/year/{marketYear}`
   - Every country: `PSDData_GetCommodityDataByYear` — `GET /api/psd/commodity/{commodityCode}/country/all/year/{marketYear}`
   - One country: `PSDData_GetCountryCommodityDataByYear` — `GET /api/psd/commodity/{commodityCode}/country/{countryCode}/year/{marketYear}`

   Resolve `{countryCode}` with `PSDData_GetCountries` (`GET /api/psd/countries`) and group with `PSDData_GetRegions` (`GET /api/psd/regions`).
5. **Label the units.** `PSDData_GetUnitsOfMeasure` — `GET /api/psd/unitsOfMeasure`. PSD quantities are meaningless without the unit, and units differ by commodity.

## Rules an agent must follow here

- **Say it is a forecast, and say which market year.** A PSD figure without its market year and its release date is not a fact anyone can check. Carry both through to whatever you report.
- **Prefer the world roll-up over summing countries.** `world` is published as its own operation; adding up `country/all` is a different number and invites a rounding or coverage error.
- **No pagination.** `country/all` returns the whole set for that commodity and market year in one response.
- **Read-only.** No writes exist in this API, so there is no idempotency contract and nothing to undo.
- **No rate-limit signal.** Nothing tells you how much budget is left; pace conservatively.
