---
name: ynab-api
description: Obsługuj YNAB API v1 — pobieraj plany, konta, kategorie i transakcje, synchronizuj zmiany oraz twórz i aktualizuj dane planu. Używaj przy integracjach z YNAB lub pracy z danymi tego API.
metadata:
  version: "1.0.0"
  api-version: "v1"
  api-release: "1.88.0"
  openapi-version: "3.1.1"
---

# YNAB API v1

Baza: `https://api.ynab.com/v1`. Snapshot dokumentacji: 2026-11-03.
`1.88.0` to wydanie API, `3.1.1` format OpenAPI, `1.0.0` wersja skilla.
Dokumentacja używa `/plans` i `plan_id`; nie zastępuj ich dawnym `/budgets`.

## Wybierz operację

W ścieżkach poniżej prefiks `P` oznacza `/plans/{plan_id}`.

| Potrzeba | Metoda i ścieżka |
| --- | --- |
| Użytkownik, lista i ustawienia planu | `GET /user`, `GET /plans`, `GET P/settings` |
| Plan, konta, kategorie, odbiorcy | `GET P`, `GET P/accounts`, `GET P/categories`, `GET P/payees` |
| Miesiąc i kategorie miesiąca | `GET P/months/{month}`, `GET P/months/{month}/categories/{category_id}` |
| Historia transakcji | `GET P/transactions` lub wariant dla konta, kategorii, odbiorcy, miesiąca |
| Tworzenie i aktualizacja transakcji | `POST P/transactions`, `PATCH P/transactions`, `PUT P/transactions/{transaction_id}` |
| Usunięcie transakcji | `DELETE P/transactions/{transaction_id}` |
| Import z połączonych instytucji | `POST P/transactions/import` |
| Transakcje cykliczne | `GET/POST P/scheduled_transactions`, `GET/PUT/DELETE P/scheduled_transactions/{scheduled_transaction_id}` |
| Przesunięcia środków — odczyt | `GET P/money_movements`, `GET P/money_movement_groups` i warianty miesiąca |

Przed budową żądania sprawdź metodę, parametry, body, odpowiedzi i schematy
w [references/openapi.yaml](references/openapi.yaml). Zawiera wszystkie 44 operacje,
w tym zarządzanie kontami, kategoriami, grupami kategorii i odbiorcami oraz lokalizacje.
Procedury i przykłady: [references/workflows.md](references/workflows.md).
Pochodzenie dokumentacji: [references/provenance.json](references/provenance.json).
Nie wymyślaj endpointów ani parametrów; nazwy interfejsu nie zawsze odpowiadają polom API
(np. przypisanie do kategorii nadal używa `budgeted`).

## Uwierzytelnianie i wykonanie

Dla własnego konta użyj Personal Access Token z Account Settings > Developer Settings,
przekazywanego z otoczenia jako `YNAB_ACCESS_TOKEN`. Nagłówek:
`Authorization: Bearer <ACCESS_TOKEN>`. Nie umieszczaj tokenu w URL, repo ani logach;
nie wyświetlaj całego `.env` podczas szukania sekretu.
Dla kont innych użytkowników używaj OAuth; procedurę i PKCE opisuje referencja workflow.
Scope `read-only` wybieraj, gdy integracja nie potrzebuje zapisu.

Sprawdzaj HTTP oraz obiekt `data`; błędy mają `error.id`, `error.name`, `error.detail`.
Nie traktuj brakujących pól jak zera. Nie drukuj całych odpowiedzi finansowych bez potrzeby.
Limit wynosi 200 żądań na godzinę na token w ruchomym oknie. Po 429 ogranicz żądania;
nie ponawiaj natychmiast. Używaj cache, precyzyjnych endpointów i delta requests.

## Odczyt i interpretacja

Jeśli plan nie jest ustalony, pobierz `GET /plans` i wybierz jednoznaczne ID.
`last-used` i `default` są aliasami, a `default` zależy od konfiguracji OAuth.
Do wieloetapowej synchronizacji i zapisów przypnij konkretne ID planu.

Kwoty to całkowite milliunits: 1000 = 1 jednostka waluty. Używaj liczb całkowitych
lub Decimal, nigdy binarnego float przy przeliczeniu. Wydatki są ujemne.
Nie sumuj różnych walut; walutę ustal z ustawień planu. Pola `*_formatted`
służą prezentacji, nie obliczeniom. Daty: `YYYY-MM-DD`, czasy według dokumentacji UTC.
Nie zmieniaj daty księgowej przez lokalne przeliczenie strefy.

Dla delta requests przechowuj `server_knowledge`, wysyłaj go jako
`last_knowledge_of_server` tylko tam, gdzie parametr jest udokumentowany.
Scalaj rekordy po ID, obsługuj `deleted=true`, utrwalaj kursor dopiero po scaleniu.
Stan synchronizacji rozdzielaj według planu, zasobu i filtrów. Delta nie jest pełną listą.

## Zapis zmienia plan

Wykonuj zapis, gdy użytkownik go zlecił; przygotowanie danych nie jest zleceniem wysłania.
Nie pytaj ponownie o udzieloną zgodę. Ustal plan, konta, kategorie, daty, walutę i kwoty;
przy niejednoznaczności przygotuj podgląd i zapytaj tylko o brakującą informację.
Nie ustawiaj zatwierdzenia, uzgodnienia ani nowego odbiorcy bez podstawy w zleceniu.

Przy imporcie pliku do transakcji użyj `POST /transactions` z body opisanym w schemacie;
`POST /transactions/import` uruchamia import z kont połączonych z instytucjami,
a nie upload pliku CSV. Przed wysłaniem sprawdź schemat odpowiedniego endpointu.

Dla tworzonych transakcji używaj stabilnego `import_id`, jeśli źródło zapewnia trwałe ID;
unikalność działa w obrębie konta. Nie generuj nowego ID przy ponowieniu ani nie zakładaj,
że sama kwota i data identyfikują transakcję. Raportuj `duplicate_import_ids`.
Po timeout zapisu sprawdź stan po ID/import_id; nie ponawiaj mutacji w ciemno.
Nie zakładaj atomowości operacji zbiorczych ani trybu dry-run.
Po zapisie podaj utworzone/zmienione/usunięte ID, pominięte duplikaty i błędy,
zgodnie z faktyczną odpowiedzią. Rozróżniaj podgląd, wysłanie i potwierdzony zapis.
