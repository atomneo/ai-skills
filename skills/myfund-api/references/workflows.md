# myFund.pl API v1 workflows

## Reading

`getPortfele.php` accepts `apiKey` and optional `format=json`. The `portfele` list
includes `nazwa`, `id`, `grupa`, `subportfel`, and optionally `udostepniony`.
An empty list is valid. Use `nazwa` for subsequent requests.
`getPortfel.php` requires `apiKey`, `portfel`, and `format=json`.

Python 3 example (standard library, key already set in the environment):

```python
import json
import os
import urllib.error
import urllib.parse
import urllib.request

params = urllib.parse.urlencode({
    "apiKey": os.environ["MYFUND_API_KEY"],
    "portfel": "Mój Portfel",  # replace with the name selected by the user
    "format": "json",
    "scope": "status,portfel,tickers",
})
request = urllib.request.Request(
    "https://myfund.pl/API/v1/getPortfel.php?" + params
)
try:
    with urllib.request.urlopen(request, timeout=30) as response:
        data = json.load(response)
except urllib.error.HTTPError as error:
    raise SystemExit(f"myFund: HTTP {error.code}") from None
except (urllib.error.URLError, TimeoutError):
    raise SystemExit("myFund: connection error; check service availability") from None
if str(data.get("status", {}).get("code")) != "0":
    raise SystemExit("myFund: API error or missing status; inspect the response locally")
# Process data locally; do not print the entire portfolio unnecessarily.
```

For gzip, send `Accept-Encoding: gzip` and handle decompression in the client;
the example above does not request gzip. Do not add the header without response handling.

Typical `scope` selections (always include `status`):

- summary: `portfel,tickers,struktura,strukturaWalory`;
- performance: `zyskWCzasie,wartoscWCzasie,wkladWCzasie,benchWCzasie,stopaZwrotuWCzasie`;
- history: `historiaOperacji,historiaOperacjiLiczba` (ISIN may be empty);
- dividends: `dywidendyWCzasie`;
- accounts: `udzialKontInwestycyjnychWCzasie`;
- liabilities: `zobowiazania,zobowiazaniaLiczba,zobowiazaniaPodsumowanie`.

Historical account shares refer to the sum of positive valuations; today's shares
refer to the net sum. According to this report's description, an absent account on
a date means zero. Do not extend this rule to other missing fields.
Do not sum amounts in different currencies without conversion.

## JSON import

Send JSON to `importOperacji.php`, with `Content-Type: application/json`.
Required: `portfel`, `typImportu`, `operacje`, and the `apiKey` key in the body
or query. The following example is a data preview, not an instruction to execute:

```json
{
  "apiKey": "{apiKey}",
  "portfel": "Mój Portfel",
  "typImportu": "transakcji",
  "kontoGotowkowe": "Gotówka",
  "kontoInwestycyjne": "Bez konta",
  "duplikaty": 1,
  "sprawdzCene": "true",
  "operacje": [
    {"data": "2026-05-10", "nazwa": "NASDAQ_AAPL", "operacja": "Kupno", "ilosc": 1, "cena": 200}
  ]
}
```

For `typImportu="transakcji"`, required operation fields are `data`, `nazwa`,
`operacja`, `ilosc`, and `cena`. Instrument matching order: myFund ticker, exact name,
ISIN, common name, then similarity of name/date/price. Prefer an unambiguous
identifier confirmed by the user or source data.

For `typImportu="przepływów"`, required fields are `data`, `operacja`, and `kwota`;
example types are `Wpłata`, `Wypłata`, `Koszt`, and `Odsetki`.
Check other fields and their types in the `ImportTransaction` and `ImportCashflow` schemas.
Date: `YYYY-MM-DD`. The specification's time example is `13:45:00`, although its
`format` label says `HH-MM-SS`; follow the example and note the discrepancy when integrating.
`sprawdzCene` is the string `"true"` or `"false"`, not a boolean.

## Broker file import

`POST /importOperacjiPlik.php`, multipart: `apiKey`, `portfel`, `plik`.
Formats: csv, xls, xlsx, pdf, html, htm. `bank` must match the name in the import UI,
e.g. `DM BDM (csv)`. Aliases: `source`, `sourcePlugin`.
Without a source, the server uses the portfolio's last setting; do not assume it
matches the file. Optional: `kontoGotowkowe`, `kontoInwestycyjne`, `duplikaty`, `sprawdzCene`.
Preserve the original file and filename; do not transform the data unnecessarily.

## PPK import

`POST /importOperacjiPPKPlik.php`, multipart: `apiKey`, `portfel`, `kontoPPK`, `plik`.
Use the account name without the Pracownik/Pracodawca/Państwo suffixes; the alias is `konto`.
The account must have all three PLN subaccounts enabled. The file must be **smaller than
5,000,000 bytes**, in csv/xls/xlsx/html/htm format matching the source.
File aliases: `file`, `imagefile`. `bank`: PPK source name or code,
e.g. `PKO PPK (csv/xlsx)` or `PKOPPK`; aliases: `source`, `sourcePlugin`.

Contribution percentages (`skladkaPodPracownik`, `skladkaPodPracodawca`,
`skladkaDodPracownik`, `skladkaDodPracodawca`) range from 0 to 100; the sum of
basic contributions must be positive. The server uses portfolio settings, or
2 / 1.5 / 2 / 1.5 if absent. Override only according to user data. `force2` and `force3`
(0/1) are mutually exclusive. The endpoint does not change saved portfolio settings.

For PPK, `duplikaty=0` raises an error on duplicates, `1` skips them, and `2` adds all.
For other imports, the documentation describes `0` as “do not add,” without the same
error guarantee. Do not conflate these meanings. When all PPK rows are skipped as
duplicates, the result may be success with `liczbaOperacji=0` and `minData=null`.

## Errors

Check responses documented for the specific endpoint. `getPortfele.php` documents
401, 403, and 502; imports document 400 and 401, with PPK also documenting
403 (editing blocked) and 405 (method).
Do not try to fix 401 by regenerating the key without a user instruction.
When an import outcome is uncertain, stop retrying and determine whether operations were saved.
Do not treat a later cached read as evidence that the import did not occur.
