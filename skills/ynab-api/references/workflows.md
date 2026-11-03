# Procedury YNAB API v1

## Odczyt i wybór zasobu

Najpierw `GET /plans`, następnie pobierz tylko potrzebny zasób konkretnego planu.
Odpowiedź to `data.plans`, `data.accounts`, `data.transactions` itd.; pojedyncze zasoby
używają liczby pojedynczej. Puste tablice są poprawne. Do adresowania używaj ID,
nie nazw. Wyszukaj w OpenAPI ścieżkę i wskazane przez nią `$ref`.

Przykład Python 3, bez zewnętrznych zależności; token jest już w otoczeniu:

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
    raise SystemExit(f"YNAB: HTTP {error.code}; sprawdź błąd API lokalnie") from None
except (urllib.error.URLError, TimeoutError):
    raise SystemExit("YNAB: błąd połączenia") from None
if not isinstance(result.get("data", {}).get("plans"), list):
    raise SystemExit("YNAB: brak oczekiwanej listy planów")
plans = result["data"]["plans"]
# Wybierz plan zgodnie ze zleceniem; nie drukuj pełnej odpowiedzi bez potrzeby.
```

Transakcje można ograniczyć m.in. przez `since_date` i `type`, ale dostępność
parametrów zależy od endpointu. Nie zakładaj paginacji ani filtrów spoza specyfikacji.
Miesiąc podawaj według parametru `month` w OpenAPI; nie używaj daty dziennej
jako miesiąca bez sprawdzenia formatu i dozwolonych aliasów.

## Kwoty, kategorie i przelewy

```python
from decimal import Decimal

amount = Decimal("-123.45") * 1000
if amount != amount.to_integral_value():
    raise ValueError("Kwota wymaga więcej niż trzech miejsc dziesiętnych")
milliunits = int(amount)  # -123450
```

Uwzględnij precyzję waluty planu. Salda kont i kategorii mają inne znaczenie;
nie licz wydatków przez sumowanie obu. Kategorie zawierają `budgeted`, `activity`,
`balance`. Zmiana miesięcznego `budgeted` ustawia kwotę docelową, nie przyrost.
Sprawdź `SaveMonthCategoryWrapper` oraz schemat pola przed aktualizacją.
`hidden`, `closed`, `deleted`, brak pola i `null` nie są równoważne.

Przelew twórz przez `payee_id` odpowiadający `transfer_payee_id` konta docelowego,
nie przez dwa niezależne wydatki. Nie podwajaj przelewów w analizie wydatków.
Dla splitów sprawdź `subtransactions`, sumę kwot i ograniczenia kont/kategorii.
Transakcje cykliczne są osobnym zasobem; nie licz ich jako już zaksięgowanych.

## Tworzenie i import transakcji

Poniżej lokalny podgląd body `POST /plans/{plan_id}/transactions`;
identyfikatory są placeholderami, a przykład nie jest poleceniem wysłania:

```json
{
  "transaction": {
    "account_id": "{account_id}",
    "date": "2026-11-03",
    "amount": -123450,
    "payee_id": "{payee_id}",
    "category_id": "{category_id}",
    "memo": "Przykładowy wydatek",
    "import_id": "source:example-0001"
  }
}
```

Wybierz `transaction` albo `transactions`, nie oba. Sprawdź `NewTransaction`,
`SaveTransactionWithOptionalFields` i wrapper endpointu. `import_id` ma maks. 36 znaków,
unikalność w obrębie konta; musi być stabilny dla danego rekordu źródłowego.
Dokumentacja opisuje format `YNAB:[milliunit_amount]:[iso_date]:[occurrence]`
dla zgodności z Direct Import/File Based Import. Zachowaj rozróżnienie kilku
rzeczywistych transakcji tego samego dnia i tej samej kwoty.
Import może dopasować istniejącą transakcję wprowadzoną ręcznie na tym samym koncie
z tą samą kwotą i datą w zakresie +/-10 dni. Nie utożsamiaj każdego potwierdzonego
importu z utworzeniem nowego wiersza.

`POST /transactions/import` uruchamia Direct Import na połączonych kontach;
nie przyjmuje eksportu CSV. Wysyłając dane z pliku, najpierw przekształć je lokalnie
do `transactions`, zweryfikuj walutę, konta, znaki, daty i duplikaty.
Aktualizacje zbiorcze PATCH używają `id` lub `import_id`; `id` ma pierwszeństwo,
a istniejącego `import_id` nie można zmienić. Ograniczenia PUT/PATCH sprawdź oddzielnie.

Po zapisie sprawdź `transaction_ids`, `transaction`/`transactions`,
`duplicate_import_ids` i `server_knowledge`, o ile odpowiedź je zawiera.
Przy niepewnym wyniku najpierw odczytaj stan. Nie stosuj ogólnego retry dla POST/DELETE.
Nie przenoś kursora otrzymanego po zapisie do niezależnej synchronizacji bez scalenia danych.

## Synchronizacja delta

1. Pobierz pełny zasób z jego parametrami i zapisz rekordy oraz `server_knowledge`.
2. Dla tego samego planu, zasobu i filtrów dodaj `last_knowledge_of_server`.
3. Scal zmiany po ID; `deleted=true` usuwa rekord z aktywnego lokalnego stanu.
4. Utrwal dane i nowy kursor razem; po nieudanym scaleniu zachowaj poprzedni kursor.

Parametr jest dostępny tylko w wybranych operacjach. Pusta delta nie oznacza pustego
planu. Po zmianie zakresu filtrów wykonaj pełny odczyt nowego zakresu.
Limit to 200 żądań na godzinę na token, wspólny dla zasobów. Nie pobieraj całego
planu, jeśli wystarczy jedno konto lub kategoria. Dla 429 uwzględnij `Retry-After`,
jeśli serwer go zwróci; dokumentacja nie gwarantuje tego nagłówka.

## OAuth — tylko dla integracji wymagającej autoryzacji użytkowników

Własny PAT nie wygasa, ale może być odwołany. Nie proś innych użytkowników o PAT;
dla nich zarejestruj OAuth Application w Developer Settings.

Authorization Code: `https://app.ynab.com/oauth/authorize`, parametry `client_id`,
`redirect_uri`, `response_type=code`; dodaj unikalny `state` i zweryfikuj zwrot.
PKCE: losowy `code_verifier` 43–128 znaków, challenge SHA-256 zakodowany base64url
bez paddingu, `code_challenge_method=S256`. Zachowaj verifier do wymiany kodu.
Kod wymień POST do `https://app.ynab.com/oauth/token`, form-urlencoded:
`client_id`, `client_secret`, `redirect_uri`, `grant_type=authorization_code`,
`code`, a przy PKCE także `code_verifier`. Nie ujawniaj client_secret klientowi.

Token OAuth wygasa po 7200 sekundach. Odświeżanie: ten sam endpoint token,
`grant_type=refresh_token`, `client_id`, `client_secret`, `refresh_token`.
Bezpiecznie zapisz nową parę tokenów zwróconą przez serwer. Scope `read-only`
zabrania mutacji (403). Snapshot opisuje też Implicit Grant bez refresh tokenu;
przed wyborem przepływu dla nowej aplikacji sprawdź aktualne oficjalne wymagania.
Nowa aplikacja działa w Restricted Mode (25 autoryzacji innych użytkowników)
i wymaga przeglądu YNAB przed zniesieniem tego limitu.

## Błędy i wynik

Sprawdzaj kody odpowiedzi dla konkretnej operacji. 401 oznacza problem
uwierzytelnienia; nie regeneruj PAT bez zlecenia. 403 może wynikać ze scope.
404 sprawdź pod kątem planu i ID zasobu. Walidację 400/409 analizuj według
`error` i opisu endpointu. 429 wymaga ograniczenia ruchu. Odczyty po przejściowym
błędzie można ponawiać w ograniczony sposób; zapisy wymagają sprawdzenia stanu.
Nie ujawniaj w komunikatach tokenu ani niepotrzebnych danych finansowych.
