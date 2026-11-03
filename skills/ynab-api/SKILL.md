---
name: ynab-api
description: Work with YNAB API v1 — retrieve plans, accounts, categories, and transactions, synchronize changes, and create or update plan data. Use for YNAB integrations or tasks involving data from this API.
metadata:
  version: "1.0.0"
  api-version: "v1"
  api-release: "1.88.0"
  openapi-version: "3.1.1"
---

# YNAB API v1

Base URL: `https://api.ynab.com/v1`. Documentation snapshot: 2026-11-03.
`1.88.0` is the API release, `3.1.1` the OpenAPI format, and `1.0.0` the skill version.
The documentation uses `/plans` and `plan_id`; do not replace them with the older `/budgets`.

## Choose an operation

In the paths below, prefix `P` stands for `/plans/{plan_id}`.

| Need | Method and path |
| --- | --- |
| User, plan list, and plan settings | `GET /user`, `GET /plans`, `GET P/settings` |
| Plan, accounts, categories, payees | `GET P`, `GET P/accounts`, `GET P/categories`, `GET P/payees` |
| Month and monthly categories | `GET P/months/{month}`, `GET P/months/{month}/categories/{category_id}` |
| Transaction history | `GET P/transactions` or the account, category, payee, or month variant |
| Create and update transactions | `POST P/transactions`, `PATCH P/transactions`, `PUT P/transactions/{transaction_id}` |
| Delete a transaction | `DELETE P/transactions/{transaction_id}` |
| Import from linked institutions | `POST P/transactions/import` |
| Scheduled transactions | `GET/POST P/scheduled_transactions`, `GET/PUT/DELETE P/scheduled_transactions/{scheduled_transaction_id}` |
| Money movements — read | `GET P/money_movements`, `GET P/money_movement_groups` and their month variants |

Before building a request, check the method, parameters, body, responses, and schemas
in [references/openapi.yaml](references/openapi.yaml). It contains all 44 operations,
including management of accounts, categories, category groups, and payees, plus locations.
Procedures and examples: [references/workflows.md](references/workflows.md).
Documentation provenance: [references/provenance.json](references/provenance.json).
Do not invent endpoints or parameters; UI names do not always match API fields
(e.g. assigning money to a category still uses `budgeted`).

## Authentication and execution

For your own account, use a Personal Access Token from Account Settings > Developer Settings,
provided through the environment as `YNAB_ACCESS_TOKEN`. Header:
`Authorization: Bearer <ACCESS_TOKEN>`. Do not put the token in URLs, the repository, or logs;
do not display an entire `.env` file when looking for a secret.
For other users' accounts, use OAuth; the workflow reference covers the procedure and PKCE.
Choose the `read-only` scope when the integration does not need writes.

Check HTTP and the `data` object; errors contain `error.id`, `error.name`, and `error.detail`.
Do not treat missing fields as zero. Do not print complete financial responses unnecessarily.
The limit is 200 requests per hour per token in a rolling window. After 429, reduce requests;
do not retry immediately. Use caching, specific endpoints, and delta requests.

## Reading and interpretation

If the plan is not established, retrieve `GET /plans` and select an unambiguous ID.
`last-used` and `default` are aliases, and `default` depends on OAuth configuration.
Pin a specific plan ID for multi-step synchronization and writes.

Amounts are integer milliunits: 1000 = 1 currency unit. Use integers
or Decimal, never binary floating point for conversion. Expenses are negative.
Do not sum different currencies; determine the currency from plan settings. Fields ending
in `*_formatted` are for presentation, not calculations. Dates use `YYYY-MM-DD`;
times are UTC according to the documentation. Do not shift an accounting date through
local timezone conversion.

For delta requests, retain `server_knowledge` and send it as
`last_knowledge_of_server` only where the parameter is documented.
Merge records by ID, handle `deleted=true`, and persist the cursor only after merging.
Separate synchronization state by plan, resource, and filters. A delta is not a full list.

## Writes change the plan

Execute writes when the user requests them; preparing data is not an instruction to submit it.
Do not ask again for authorization already given. Establish the plan, accounts, categories,
dates, currency, and amounts; if ambiguous, prepare a preview and ask only for missing information.
Do not set approval or reconciliation status, or create a payee, without support in the request.

To import a file as transactions, use `POST /transactions` with the body defined in the schema;
`POST /transactions/import` triggers import from accounts linked to institutions,
not a CSV file upload. Check the relevant endpoint's schema before submitting.

For new transactions, use a stable `import_id` if the source provides persistent IDs;
uniqueness is per account. Do not generate a new ID on retry or assume that amount
and date alone identify a transaction. Report `duplicate_import_ids`.
After a write timeout, check state by ID/import_id; do not blindly retry mutations.
Do not assume batch operations are atomic or that a dry-run mode exists.
After a write, report created/updated/deleted IDs, skipped duplicates, and errors
according to the actual response. Distinguish a preview, submission, and confirmed write.
