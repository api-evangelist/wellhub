---
name: Sync employee eligibility to Wellhub
description: Authenticate, create an eligibility job, add employee items in batches, submit it, then poll status and read any errors.
api: openapi/wellhub-integrations-openapi-original.json
operations:
  - createAccessToken
  - createEligibilityJob
  - addJobItems
  - submitJob
  - getJobStatus
  - listJobErrors
---

# Sync employee eligibility to Wellhub

Keep a client's employee roster in sync with Wellhub so onboarding and offboarding stay current. All calls go over HTTPS to `https://api.clients.wellhub.com` (use `https://pilot-api.clients.wellhub.com` for the Sandbox, which returns synthetic data and mutates nothing).

## 1. Get an access token — `POST /oauth/token`
Send `client_id`, `client_secret`, and `grant_type=client_credentials` as **`application/x-www-form-urlencoded`** (not JSON, or you get `400 invalid grant type`). Use the returned Bearer JWT in `Authorization: Bearer <token>` on every subsequent call. Manage credentials in the Wellhub for Companies portal (clients.wellhub.com → Settings).

## 2. Create a job — `POST /v1/eligibility/jobs`
Returns `201` with the new `job_id`. A job is a batch of eligibility updates.

## 3. Add items — `POST /v1/eligibility/jobs/{job-id}/items`
Add up to **500 items per request**; each item needs an `operation` and a `company_tax_id`. Returns `204`. Oversized/invalid batches return `400`; too-large payloads return `413`.

## 4. Submit — `POST /v1/eligibility/jobs/{job-id}/submit`
Returns `202 Accepted` with an **empty body — do not parse it**. Submitting a job that was already submitted returns `409 Conflict`; a job with no items or in a bad state returns `400`/`422`. This 409 guard is what makes the flow safely re-entrant (there is no idempotency-key header).

## 5. Poll status — `GET /v1/eligibility/jobs/{job-id}/status`
Poll until processing completes; the response carries job statistics.

## 6. Read errors — `GET /v1/eligibility/jobs/{job-id}/errors`
List per-record processing errors. This endpoint is cursor-paginated: follow the opaque `cursor` field until it is `null`.

## Conventions to respect
- **Pagination**: opaque `cursor` query param; stop when the response `cursor` is `null`. Never construct cursor values.
- **Rate limits** (per client): job endpoints 25–50 RPM, read endpoints up to 300 RPM, token endpoint 5 RPM, each with a burst/sec cap. On `429`, honor `RateLimit-Reset` and back off. See `rate-limits/wellhub-rate-limits.yml`.
- **Errors**: JSON `{ error, trace_id }`; quote the `trace_id` to support. Validation failures return a list of `{ field, message }`. See `errors/wellhub-problem-types.yml`.
