---
name: audit-workspace-activity
description: Read the FlipperForce workspace activity log to audit who changed what, and reconcile project income against expenses.
api: FlipperForce Public API
base_url: https://tools.flipperforce.com/api/v1
operations:
  - v1.user.account
  - v1.workspace.activity-log.filters
  - v1.workspace.activity-log.list
  - v1.project.income.list
  - v1.project.income.create
  - v1.project.income.show
  - v1.project.expense-line-items.list
---

# Audit workspace activity and reconcile a project

## Discover the filter vocabulary first

Do not guess activity types. Call `v1.workspace.activity-log.filters`
(`GET /workspace/{workspace}/activity-log/filters`) — it returns the valid values for the log's
filter parameters, including the project and user filters available to this workspace.

## Read the log

`v1.workspace.activity-log.list` (`GET /workspace/{workspace}/activity-log/list`) accepts:

- `activity_types[]` — types from the filters endpoint
- `project_uuids[]`, `user_uuids[]` — scope to specific projects or actors
- `performed_at[gte]`, `performed_at[lt]` — the time window
- `cursor`, `per_page` — pagination

Each entry carries an **actor**, a **target** (with links), and project context. This is the newest
part of the API — it shipped in v0.0.10 on 2026-07-25.

## Reconcile a project's P&L

1. **Income** — `v1.project.income.list` (`GET /project/{projectV1}/income/list`);
   `v1.project.income.show` for one record; `v1.project.income.create` to add one.
2. **Costs** — `v1.project.expense-line-items.list`
   (`GET /project/{projectV1}/expense-line-items/list`), filtered by
   `date[gte]`/`date[lte]` to match the income period.
3. Net the two per project to get job-cost profitability.

## Rules

- The activity log is an audit surface: treat it as read-mostly and page it with `cursor` rather
  than re-scanning from the beginning.
- Time filters on the activity log use `performed_at`, while the money endpoints use `date` — they
  are not interchangeable.
- Timestamps carry microseconds (`2026-06-14T15:30:45.123456Z`) as of v0.0.9. If you cache a
  high-water mark for incremental reads, store the full-precision string.
- **No idempotency key** on `income.create` — a retry double-books revenue.
