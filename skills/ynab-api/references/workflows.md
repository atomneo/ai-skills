# YNAB API v1 workflows

## Reading and resource selection

Start with `GET /plans`, then retrieve only the needed resource from a specific plan.
Responses use `data.plans`, `data.accounts`, `data.transactions`, etc.; individual
resources use singular names. Empty arrays are valid. Address resources by ID,
not name. Find the path in OpenAPI and follow its `$ref` references.

Python 3 example without external dependencies; the token is already in the environment:

```python
import json
import os
import urllib.error
import urllib.request

request = urllib.request.Request(
    "https://api.ynab.com/v1/plans",
    headers={"Authorization": "Bearer " + os.environ["YNAB_ACCESS_TOKEN"]},
)
try:
    with urllib.request.urlopen(request, timeout=30) as response:
        result = json.load(response)
except urllib.error.HTTPError as error:
    raise SystemExit(f"YNAB: HTTP {error.code}; inspect the API error locally") from None
except (urllib.error.URLError, TimeoutError):
    raise SystemExit("YNAB: connection error") from None
if not isinstance(result.get("data", {}).get("plans"), list):
    raise SystemExit("YNAB: expected plan list is missing")
plans = result["data"]["plans"]
# Select a plan according to the request; do not print the full response unnecessarily.
```

Transactions can be filtered using parameters such as `since_date` and `type`, but
parameter availability depends on the endpoint. Do not assume pagination or filters
beyond the specification. Supply the month according to the OpenAPI `month` parameter;
do not use a daily date as a month without checking the format and allowed aliases.

## Amounts, categories, and transfers

```python
from decimal import Decimal

amount = Decimal("-123.45") * 1000
if amount != amount.to_integral_value():
    raise ValueError("Amount requires more than three decimal places")
milliunits = int(amount)  # -123450
```

Respect the plan currency's precision. Account balances and category balances have
different meanings; do not calculate spending by summing both. Categories contain
`budgeted`, `activity`, and `balance`. Updating monthly `budgeted` sets a target amount,
not an increment. Check `SaveMonthCategoryWrapper` and the field schema before updating.
`hidden`, `closed`, `deleted`, a missing field, and `null` are not equivalent.

Create a transfer using the `payee_id` matching the destination account's
`transfer_payee_id`, not two independent expenses. Do not double-count transfers in
spending analysis. For splits, check `subtransactions`, the sum of amounts, and
account/category restrictions. Scheduled transactions are a separate resource;
do not count them as already posted.

## Creating and importing transactions

Below is a local preview of the body for `POST /plans/{plan_id}/transactions`;
IDs are placeholders, and the example is not an instruction to submit:

```json
{
  "transaction": {
    "account_id": "{account_id}",
    "date": "2026-11-03",
    "amount": -123450,
    "payee_id": "{payee_id}",
    "category_id": "{category_id}",
    "memo": "Example expense",
    "import_id": "source:example-0001"
  }
}
```

Choose either `transaction` or `transactions`, not both. Check `NewTransaction`,
`SaveTransactionWithOptionalFields`, and the endpoint wrapper. `import_id` is at most
36 characters and unique per account; it must be stable for a given source record.
The documentation describes the format `YNAB:[milliunit_amount]:[iso_date]:[occurrence]`
for compatibility with Direct Import/File Based Import. Preserve the distinction
between multiple real transactions on the same day for the same amount.
An import may match an existing manually entered transaction on the same account
with the same amount and a date within +/-10 days. Do not equate every confirmed
import with the creation of a new row.

`POST /transactions/import` triggers Direct Import on linked accounts;
it does not accept a CSV export. When submitting file data, first transform it locally
into `transactions`, verifying currency, accounts, signs, dates, and duplicates.
Batch PATCH updates use `id` or `import_id`; `id` takes precedence,
and an existing `import_id` cannot be changed. Check PUT/PATCH restrictions separately.

After a write, check `transaction_ids`, `transaction`/`transactions`,
`duplicate_import_ids`, and `server_knowledge` when present in the response.
When the outcome is uncertain, read the state first. Do not apply blanket retries to POST/DELETE.
Do not transfer a cursor received after a write to an independent synchronization without merging data.

## Delta synchronization

1. Retrieve the full resource with its parameters and save the records and `server_knowledge`.
2. For the same plan, resource, and filters, add `last_knowledge_of_server`.
3. Merge changes by ID; `deleted=true` removes a record from the active local state.
4. Persist the data and new cursor together; retain the previous cursor if merging fails.

The parameter is available only for selected operations. An empty delta does not
mean an empty plan. After changing the filter scope, perform a full read of the new scope.
The limit is 200 requests per hour per token, shared across resources. Do not retrieve
an entire plan if one account or category is sufficient. For 429, honor `Retry-After`
if the server returns it; the documentation does not guarantee this header.

## OAuth — only for integrations requiring user authorization

A personal PAT does not expire, but can be revoked. Do not ask other users for PATs;
register an OAuth Application in Developer Settings for them.

Authorization Code: `https://app.ynab.com/oauth/authorize`, parameters `client_id`,
`redirect_uri`, `response_type=code`; add a unique `state` and validate it on return.
PKCE: a random `code_verifier` of 43–128 characters, a SHA-256 challenge encoded as
base64url without padding, and `code_challenge_method=S256`. Retain the verifier for code exchange.
Exchange the code with POST to `https://app.ynab.com/oauth/token`, form-urlencoded:
`client_id`, `client_secret`, `redirect_uri`, `grant_type=authorization_code`,
`code`, plus `code_verifier` for PKCE. Do not expose client_secret to the client.

The OAuth token expires after 7200 seconds. Refresh using the same token endpoint,
`grant_type=refresh_token`, `client_id`, `client_secret`, and `refresh_token`.
Securely save the new token pair returned by the server. The `read-only` scope
prohibits mutations (403). The snapshot also describes Implicit Grant without a refresh token;
check current official requirements before choosing a flow for a new application.
A new application runs in Restricted Mode (25 authorizations by other users)
and requires YNAB review before this limit is lifted.

## Errors and results

Check response codes for the specific operation. 401 indicates an authentication
problem; do not regenerate a PAT without instruction. 403 may result from scope.
For 404, check the plan and resource IDs. Analyze 400/409 validation errors using
`error` and the endpoint description. 429 requires reducing traffic. Reads after
transient errors may be retried a limited number of times; writes require checking state.
Do not expose tokens or unnecessary financial data in messages.
