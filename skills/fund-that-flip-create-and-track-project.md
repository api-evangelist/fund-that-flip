---
name: create-and-track-project
description: Create a rehab/flip project in a FlipperForce workspace and keep its record current.
api: FlipperForce Public API
base_url: https://tools.flipperforce.com/api/v1
operations:
  - v1.user.account
  - v1.project.create
  - v1.project.list
  - v1.project.update
  - v1.project.delete
---

# Create and track a project

A **Project** is the root record for one property in FlipperForce. Everything else — expenses,
income, receipts, photos, updates — hangs off a project UUID.

## Before you start

- Authenticate with `Authorization: Bearer <api-key>`. The key is issued by request only, from
  Workspace Settings > Integrations (<https://tools.flipperforce.com/integrations>). Send it verbatim.
- Call `v1.user.account` (`GET /user/account`) first to confirm the key is live and to read the
  workspace the key belongs to. You need that workspace UUID for the create call.

## Steps

1. **Create the project** — `v1.project.create` (`POST /workspace/{workspace}/project`).
   Required fields: `name`, `address_1`, `city`, `state`, `postal_code`.
   Useful optional fields: `investment_strategy` (e.g. `fix_and_flip`, `wholesale`), `stage`,
   `type`, `style`, `description`, `address_2`, `country`, and `fetch_property_data` to have
   FlipperForce enrich the record from its property data provider.
   Expect **201 Created**. The response carries the `uuid` you will use everywhere else.
2. **Find existing projects** — `v1.project.list` (`GET /project/list`). Sort with `sort=updated`
   (the default) or `sort=nearby`, which additionally requires `latitude` and `longitude`
   (ranges -90..90 and -180..180).
3. **Update** — `v1.project.update` (`PATCH /project/{projectV1}`). Send only the fields you are
   changing.
4. **Delete** — `v1.project.delete` (`DELETE /project/{projectV1}`). Expect **204 No Content**.

## Rules

- **There is no idempotency key.** Retrying `v1.project.create` after a timeout creates a second
  project. Before retrying, call `v1.project.list` and match on address to see whether the first
  attempt landed.
- `lat`/`lng` come back `null` when the address cannot be geocoded — do not treat that as an error.
- `created_at`/`updated_at` are ISO 8601 Zulu **with microseconds**
  (`2025-04-30T21:21:10.123456Z`). Parse them as variable-precision; the microseconds were added
  in v0.0.9 and a fixed-width parser written against the old format will break.

## Errors

- **401** `{"message":"Unauthenticated."}` or `{"message":"Unauthorized: Invalid Public API key."}` —
  the key is missing or wrong.
- **403** — the key is valid but the workspace or project is outside its scope.
- **422** — validation. Read the `errors` object: each key is a request field, each value an array
  of reasons.
- **429** — rate limited. No `Retry-After` is sent; back off exponentially with jitter.

Quote the `x-app-invocation-id` response header when reporting a problem to support@flipperforce.com.
