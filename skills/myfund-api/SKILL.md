---
name: myfund-api
description: Work with myFund.pl API v1 — retrieve portfolios and holdings, analyze history, and import transactions, cash flows, and PPK files. Use for myFund.pl integrations or tasks involving data from this API.
metadata:
  version: "1.0.0"
  api-version: "v1"
  openapi-version: "3.1.0"
---

# myFund.pl API v1

Compatibility: **myFund.pl API v1**, base URL `https://myfund.pl/API/v1`.
Source: bundled documentation snapshot dated 2026-11-03. `3.1.0` is the
OpenAPI format version and `1.0.0` is the skill version; neither is the API version.
The API is described as beta. Do not assume compatibility with other versions.

## Choose an operation

| Need | Method and path |
| --- | --- |
| List portfolios | `GET /getPortfele.php` |
| Holdings, performance, history, dividends, accounts, liabilities | `GET /getPortfel.php` |
| Transactions or cash flows as JSON | `POST /importOperacji.php` |
| Broker export file | `POST /importOperacjiPlik.php` |
| PPK export with contribution breakdown | `POST /importOperacjiPPKPlik.php` |

Before constructing a request, read the relevant endpoint and its schemas
in [references/openapi.yaml](references/openapi.yaml). Practical guidance and examples
are in [references/workflows.md](references/workflows.md).
Do not invent endpoints, import source names, or undocumented parameters.

## Authentication and execution

Obtain the key from **Menu > Konto > Ustawienia konta > Klucz API**.
Regenerating it invalidates the previous key. Use a secret from the environment
(e.g. `MYFUND_API_KEY`); do not store it in the skill or examples.
Do not log full URLs containing the key, diagnostic headers, or private data.
Do not read and display an entire `.env` file to find a single secret.

Reads require `apiKey` in the query. For imports, use the JSON body or multipart fields.
URL-encode parameters, preserving Polish characters and the exact portfolio name, not its ID.
For multipart requests, let the client generate `Content-Type` with its boundary.

Check both HTTP and `status.code`: `"0"` means success, `"1"` an error,
and `"7"` a missing portfolio. HTTP 200 alone does not confirm success.
If the response lacks the expected status or data, report an incomplete result.
Reads are cached for 5 minutes; imports are not. Do not promise immediate
updates to retrieved data after an import.

## Reading and interpretation

Retrieve the portfolio list first if the name is unknown. When restricting
`scope`, always include `status`; select only the fields needed.
`scope` applies only at the top level and is case-sensitive.
Unknown fields are omitted; a missing field does not mean zero.

Preserve currency, dates, and units. Numeric values are often strings,
sometimes with `+`; treat empty values, `---`, and `&nbsp;` as missing data.
Do not blindly convert the entire object to numbers. Use decimal arithmetic for money.
`tickers` and history are maps, not arrays; time series are maps keyed by date.
Do not confuse monetary values in `struktura` with shares in `strukturaWalory`.

## Imports change the portfolio

Preparing data alone does not authorize submitting an import. Execute an import
when the user requests it; do not ask again for authorization already given.
Before submitting, establish the portfolio, data, accounts, file source, and duplicate policy.
If information affecting the write is missing, prepare a preview and ask for it.
Do not fill in unknown prices, quantities, dates, or taxes.

Keep `duplikaty=1` and `sprawdzCene="true"` by default.
`duplikaty=2` may duplicate operations; use it only on explicit instruction.
Do not assume imports are atomic or idempotent. After a timeout or lost connection,
do not automatically retry POST: check history and import status, accounting for the cache.
There is no documented dry-run mode; a local preview is not server validation.
After completion, report the status, operation count, omissions, and earliest date
when present in the response. Distinguish request preparation from an executed import.
