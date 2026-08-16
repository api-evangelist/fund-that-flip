---
name: publish-progress-update
description: Publish a dated progress update with photos to a FlipperForce project, and maintain the project photo log.
api: FlipperForce Public API
base_url: https://tools.flipperforce.com/api/v1
operations:
  - v1.workspace.upload-intent.create
  - v1.project.updates.create
  - v1.project.updates.photos.create
  - v1.project.updates.list
  - v1.project.updates.update
  - v1.project.updates.destroy
  - v1.project.updates.photos.destroy
  - v1.project.photo-log.list
  - v1.project.photo-log.create
  - v1.project.photo-log.update
  - v1.project.photo-log.destroy
---

# Publish a progress update

Progress updates are the investor- and client-facing narrative on a rehab. The **photo log** is the
raw chronological record; an **update** is a curated dated post that can carry photos of its own.

## Steps

1. **Create the update** — `v1.project.updates.create` (`POST /project/{projectV1}/updates/create`).
   Required: `title`, `description`, `posted_at`. Expect **201 Created**; keep the returned
   `uuid` as `{updateUuid}`.
2. **Upload each photo** — `v1.workspace.upload-intent.create`
   (`POST /workspace/{workspace}/upload-intent`) with `filename`, `checksum`, `mime_type`, `size`
   and an `upload_type` naming the use case.
3. **Attach photos to the update** — `v1.project.updates.photos.create`
   (`POST /project/{projectV1}/updates/{updateUuid}/photos`).
4. **Verify** — `v1.project.updates.list` (`GET /project/{projectV1}/updates/list`).

Edit with `v1.project.updates.update` (`PUT /project/{projectV1}/updates/{updateUuid}` — note this
one is PUT, not PATCH, unlike most updates in this API). Remove a single photo with
`v1.project.updates.photos.destroy`; remove the whole update with `v1.project.updates.destroy`.

## The photo log

The photo log is a separate, project-wide surface:

- `v1.project.photo-log.list` (`GET /project/{projectV1}/photo-log`) returns photos grouped by date.
- `v1.project.photo-log.create` (`POST /project/{projectV1}/photo-log/create`) accepts
  `photo_timestamp` to order a photo by its **EXIF** capture time rather than its upload time —
  use this when backfilling photos taken earlier on site.
- `v1.project.photo-log.update` and `v1.project.photo-log.destroy` maintain individual photos.

## Rules

- **No idempotency key.** A retried `updates.create` publishes a duplicate post. Check
  `v1.project.updates.list` filtered by `posted_at` before retrying.
- `posted_at` drives ordering — set it to the date the work happened, not the time you called the API.
- Upload intent returns **201**, not 200. Treat 200 as unexpected.
