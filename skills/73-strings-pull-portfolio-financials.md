---
name: Pull financials for a portfolio company
description: Discover the portfolio companies in a 73 Strings organization, resolve the reporting version, then pull entity-, business-unit- and security-level financial data.
api: openapi/73-strings-asset-info-openapi.yml, openapi/73-strings-financial-data-openapi.yml
operations: [getVersionList, getEntityList, getEntityBusinessUnits, getEntitySecuritiesList, getAttributeIDs, entityFinancialData, entityBuFinancialData, entitySecurityFinancialData, streamEntityFinancialData]
generated: '2026-08-05'
method: generated
source: openapi/*.yml + conventions/73-strings-conventions.yml + errors/73-strings-problem-types.yml
---

# Pull financials for a portfolio company

## Before you start

- Auth: send `subscription-key: <key>` as a request **header** on every call. The key comes from an Azure API Management product subscription; three of the four products require administrator approval.
- Every request body must carry `orgId` and `userId`. They are mandatory and they determine which tenant's data you see. Never guess them — take them from the caller.
- Every operation here is **POST**, including the reads. Nothing goes in the path or query string.
- There is no pagination. Bound results with the request filters (`entityIds`, `versionId`, `startDate`/`endDate`, `periodType`), not with paging.

## Steps

1. **List reporting versions** — `getVersionList` (`POST /api/v1/version-list`, Asset Info API). Body: `{orgId, userId}`. Returns `versionsData[]` with `versionId` and version name. Pick the version the caller asked for; do not default silently.
2. **List the portfolio companies** — `getEntityList` (`POST /api/v1/ems/getEntityList`, Asset Info API). Body includes `orgId` and `userId`. Match the company the caller named against the returned entities and keep its id.
3. **Resolve the metrics** — `getAttributeIDs` (`POST /api/v1/ems/getEntityAttributeIds`, Asset Info API) returns each `attributeId` with `attributeName`, a categorical `tag` (e.g. "Ratio Analysis"), `attributeType` (e.g. `NUMERICAL`) and `displayProperty` (e.g. `ABSOLUTE_NUMBER`). Financial-data requests take `attributeNames`, so use this call to confirm the exact spelling before you ask for a metric.
4. **Pull company-level financials** — `entityFinancialData` (`POST /api/v1/ems/entityFinancialData`). Request carries `entityIds`, `attributeNames`, `versionId`/`versionName`, `startDate`, `endDate`, `periodType`, `currency`, `units`, `asOfDate`, `includeDrillDown`, plus `orgId` and `userId`. The response rows carry `companyId`, `attributeId`, `attributeName`, `period`, `cashFlowDate`, `currency`, `units` and `trueUpValue`.
5. **Go one level down, if asked**:
   - By segment: `getEntityBusinessUnits` (`POST /api/v1/ems/getEntityBusinessUnits`) to list business units for the company, then `entityBuFinancialData` (`POST /api/v1/ems/entityBuFinancialData`) with `businessUnitIds`. Responses add `segmentId`, `geography`, `sector` and `fiscalYearEnd`.
   - By instrument: `getEntitySecuritiesList` (`POST /api/v1/ems/getEntitySecuritiesList`) to list securities, then `entitySecurityFinancialData` (`POST /api/v1/ems/entitySecurityFinancialData`) with `securityIds`. Responses add `securityId`.
6. **Large pulls** — use `streamEntityFinancialData` (`POST /api/v2/ems/entityFinancialData`) instead of step 4. It returns `text/event-stream`; consume it as server-sent events rather than buffering a single JSON body.
7. **Narrative text** — `getAllFinancialTexts` (`POST /api/v1/data/getAllFinancialTexts`) returns the financial commentary attached to the entity.

## Reading the response

Every call returns the same envelope:

```json
{ "response": ..., "success": true, "message": "Success", "errorCode": null }
```

- **Check `success` and `errorCode` — not the HTTP status.** A 200 with `errorCode: "EXCE00021"` and `message: "Data not found."` means the query matched nothing. Treat it as an empty result, not as success with data.
- `400` + `EXCE00020` "Fields cannot be null or empty." — a required field is missing. `orgId` and `userId` are the usual culprits.
- `401` + `EXCE00028` — the subscription key is missing, wrong, or has no active subscription.
- `405` — you used a method other than POST.
- `429` — throttled. There is no `Retry-After` header; back off exponentially with jitter. The Starter product allows 5 calls/minute and 100 calls/week.
- `202` — accepted but not complete. No polling endpoint or callback is documented, so surface this to the caller rather than looping.
- `500` + `EXCE00028` — retry with backoff, then contact support@73strings.com.

## Do not

- Do not call the same company by two different keys: the platform uses `entityId` in Asset Info and Financial Data and `companyId` in Documents, Qualitative Data and Captable. They are the same object.
- Do not retry a write blindly. There is no idempotency key on this API.
