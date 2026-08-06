---
name: Assemble a cap table and diligence pack for a portfolio company
description: Pull the capitalization table, qualitative profile, custom attributes and attached documents for one portfolio company held in 73 Strings.
api: openapi/73-strings-captable-openapi.yml, openapi/73-strings-qualitative-data-openapi.yml, openapi/73-strings-documents-openapi.yml, openapi/73-strings-asset-info-openapi.yml
operations: [getEntityList, getCaptableData, getGeneralDetails, getQualitativeAnalysisData, getCustomAttributeDetails, getDocumentIdForCompanyID, getDocumentDetailsByDocumentIDAndCompanyID]
generated: '2026-08-05'
method: generated
source: openapi/*.yml + conventions/73-strings-conventions.yml + data-model/73-strings-data-model.yml
---

# Assemble a cap table and diligence pack

## Before you start

- Auth: `subscription-key` header. `orgId` and `userId` are mandatory in every body.
- These three APIs address the portfolio company as **`companyId`**, while Asset Info and Financial Data call the same object **`entityId`**. Carry one value, name it correctly per call.
- All operations are POST.

## Steps

1. **Resolve the company** — `getEntityList` (`POST /api/v1/ems/getEntityList`, Asset Info API). Match the caller's company name to an entity and keep its id.
2. **Cap table** — `getCaptableData` (`POST /api/v1/ems/captable/data`, Captable API). The response is grouped: `GroupedEquityEntityResponse` (equity lines) and `GroupedCashEntityResponse` (cash lines), plus `GroupedSecurityData`/`SecurityDataDetail` per instrument, `KeyMetric`/`KeyMetricData`, `RegimeDetail`, and `ReportingDate`/`ReportingDateAggregate`. Pick the reporting date the caller asked for rather than assuming the latest.
3. **Company profile** — `getGeneralDetails` (`POST /api/v1/ems/getEntityGeneralDetails`, Qualitative Data API) returns the general descriptive record for the company.
4. **Qualitative analysis** — `getQualitativeAnalysisData` (`POST /api/v1/ems/getEntityQualitativeAnalysisData`) returns the qualitative analysis attached to the entity.
5. **Custom attributes** — `getCustomAttributeDetails` (`POST /api/v1/ems/customAttributeData`) returns the organization's own attribute definitions and their values. Note this operation reports partially-invalid input via an `InvalidRequests` block in the response rather than by failing the whole call — check it.
6. **Documents**:
   - `getDocumentIdForCompanyID` (`POST /api/v1/ems/getEntityDocumentList`, Documents API) lists the documents attached to the company, each as a `DocumentDto` with a `documentId`.
   - `getDocumentDetailsByDocumentIDAndCompanyID` (`POST /api/v1/ems/getEntityDocumentDownloadFIle`) returns the download detail for one document, given the document id and the company id. The operation path's spelling (`DownloadFIle`) is the provider's own — send it exactly.

## Reading the response

Same envelope everywhere: `{response, success, message, errorCode}`.

- `200` + `success: false` + `errorCode: "EXCE00021"` ("Data not found.") means no cap table, no documents, or no qualitative record exists for that company — an empty pack, not an error. Say so plainly rather than retrying.
- `400`/`EXCE00020` — a required field is null or empty.
- `401`/`EXCE00028` — subscription key rejected.
- `404` — the Captable and Documents APIs are among the six operations that document a real `404 Resource not found`.
- `429` — throttled, no `Retry-After`; back off.

## Do not

- Do not fabricate a cap table position, ownership percentage or document from partial data. If `getCaptableData` returns `EXCE00021`, report the gap.
- Do not page — there is no pagination on any of these operations. Narrow with request filters instead.
