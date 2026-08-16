---
name: record-expenses-and-receipts
description: Attach a receipt to a project and record the expense transaction and its line items for job costing.
api: FlipperForce Public API
base_url: https://tools.flipperforce.com/api/v1
operations:
  - v1.workspace.expense-accounts.list
  - v1.workspace.companies.list
  - v1.project.expense-categories.list
  - v1.workspace.upload-intent.create
  - v1.project.receipts.create
  - v1.project.receipts.list
  - v1.project.expense-transactions.create
  - v1.project.expense-transactions.list
  - v1.project.expense-line-items.list
---

# Record expenses and receipts

FlipperForce separates the **receipt** (the document) from the **expense transaction** (the money)
from the **line items** (what the money bought). Job costing needs all three.

## Steps

1. **Resolve the reference data you will cite.** These are workspace- or project-scoped lookups:
   - `v1.workspace.expense-accounts.list` (`GET /workspace/{workspace}/expense-accounts/list`) → `expense_account_uuid`
   - `v1.workspace.companies.list` (`GET /workspace/{workspace}/companies/list`) → `company_uuid` (the vendor)
   - `v1.project.expense-categories.list` (`GET /project/{projectV1}/expense-categories/list`) → `expense_category_uuid`
2. **Upload the receipt image, if you have one.** Attachments are a two-step flow:
   - `v1.workspace.upload-intent.create` (`POST /workspace/{workspace}/upload-intent`).
     Required: `filename`, `checksum`, `mime_type`, `size`. Also send `upload_type` naming the use
     case — it became required in the August 2025 release. Expect **201 Created**.
   - Then `v1.project.receipts.create` (`POST /project/{projectV1}/receipts/create`) to bind it to
     the project. `v1.project.receipts.list` reads them back.
3. **Record the expense transaction** — `v1.project.expense-transactions.create`
   (`POST /project/{projectV1}/expense-transactions/create`).
   Required: `total_tax` and `expenses` (the line items).
   Optional but strongly recommended: `receipt_uuid` (from step 2), `date`, `company_uuid`,
   `expense_account_uuid`, `expense_category_uuid`, `invoice_number`.
4. **Read it back** — `v1.project.expense-transactions.list` for transactions, or
   `v1.project.expense-line-items.list` for the flattened line-item view used in job-cost reports.

## Filtering the expense ledger

`v1.project.expense-transactions.list` and `v1.project.expense-line-items.list` support repeated
bracket arrays and range operators:

- `company_uuids[]`, `expense_account_uuids[]`, `expense_category_uuids[]`, `classes[]`
- `date[gte]`, `date[lte]`, `date[gt]`, `date[lt]`
- presence flags: `has_receipt`, `has_company`, `has_class`, `has_expense_account`, `has_expense_category`
- pagination: `cursor` and `per_page`

`has_receipt=false` is the query for "expenses missing documentation" — the usual reconciliation job.

## Rules

- **No idempotency key.** A retried `expense-transactions.create` books the expense twice. Before
  retrying, list transactions filtered by `date` and `invoice_number` and check.
- Delete/update by UUID: `v1.project.expense-transactions.update` / `.delete` and
  `v1.project.expense-line-items.update` / `.delete`.
- 422 on create almost always means a cited UUID belongs to a different workspace or project.
