---
name: Ingest a transaction ledger in bulk
description: Push transaction ledger records into the 73 Strings platform from an external pipeline, handling reference data, partial success and the indeterminate-timeout case.
api: openapi/73-strings-transaction-api-openapi.yml
operations: [getTransactionTypes, getTransactionVersions, unifiedTransaction-v2, unifiedTransaction, GTN_FUND_TRANSACTION, getTransactionData]
generated: '2026-08-05'
method: generated
source: openapi/73-strings-transaction-api-openapi.yml + errors/73-strings-problem-types.yml
---

# Ingest a transaction ledger in bulk

This is the write-heaviest surface 73 Strings publishes. It is intended for client-side engineering teams running their own data pipelines. **The API does not orchestrate synchronization and does not enforce real-time guarantees** — that is stated in the API's own description. Everything below is about not corrupting a ledger.

## Before you start

- Auth: `subscription-key` header. The key is associated with a User ID.
- `orgId` and `userId` are mandatory in every payload and scope the data. The API is multi-tenant by design.
- All operations are POST. Base URL `https://api-accord-eut-73strings.azure-api.net/transactions`.
- **There is no idempotency key.** Plan for that before you send anything.

## Steps

1. **Fetch the reference data first.** Payload shape depends on it.
   - `getTransactionTypes` (`POST /api/v1/transactionType`) returns the supported transaction types and their `transactionTypeId`.
   - `getTransactionVersions` (`POST /api/v1/transactionVersion`) returns the supported transaction structure versions. Pin the version you build payloads against.
2. **Map your keys.** Transactions carry both platform ids and your own: `externalTransactionSourceReferenceId`, `externalInvestmentEntityId`, `externalOwnerEntityId`, `externalSecurityId`. Send your upstream keys in the `external*` fields so records stay reconcilable. The platform returns a generated `displayId` — persist it; it is the stable reference for downstream workflows.
3. **Submit the batch** — `unifiedTransaction-v2` (`POST /api/v2/unifiedTransaction`). Prefer v2; `unifiedTransaction` (`POST /api/v1/unifiedTransaction`) is the older shape and is still live but superseded. For fund-level records use `GTN_FUND_TRANSACTION` (`POST /api/v1/putfundTransactions`), whose payload nests `funds[]` each with `fundId`, `fundName` and `transactions[]`.
4. **Handle the outcome before sending the next batch** (see below).
5. **Verify** — `getTransactionData` (`POST /api/v1/transactionData`) reads back the processed records. It supports `Filter` and `SortCriteria` in the request body.

## Outcomes you must handle

| Status | Meaning | What to do |
|---|---|---|
| `200` | All submitted objects accepted | Persist the returned `displayId`s |
| `202` | Accepted, not yet complete | No polling endpoint is documented. Re-read with `getTransactionData` before assuming anything |
| `207` | **Partial success** — some objects persisted, some failed | Parse the per-object result. Resubmit **only** the failed objects |
| `422` | `VALIDATION_ERROR` — per-object validation failure | The response identifies which objects failed and why. Fix and resubmit those objects only |
| `400` | `EXCE00020` / `EXCE00028` — malformed request | A required field is null or empty, or the body does not match the pinned transaction version |
| `401` | `EXCE00028` — key rejected | Subscription key missing or has no active subscription |
| `429` | Throttled | No `Retry-After`. Exponential backoff with jitter |
| `504` | **Gateway timeout — processing state is indeterminate.** Some, all, or none of the submitted objects may have been persisted | **Do not blind-retry.** Call `getTransactionData` and reconcile on your `externalTransactionSourceReferenceId` first, then resubmit only what is genuinely missing |
| `500` | `EXCE00028` — server fault | Backoff, then contact support@73strings.com |

Remember the envelope: `{response, success, message, errorCode}` is returned on success *and* failure, and a business-level failure can ride on an HTTP 200 (`EXCE00021`, "Data not found."). Read `success` and `errorCode`.

## Do not

- Do not retry a 504 or a timed-out 429 without reconciling first. Without an idempotency key, a blind retry can double-post a ledger.
- Do not resubmit a whole batch after a 207. Resubmit the failed objects only.
- Do not mix transaction structure versions inside one batch.
