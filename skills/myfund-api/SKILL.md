---
name: myfund-api
description: Obsługuj myFund.pl API v1 — pobieraj portfele i ich skład, analizuj historię oraz importuj transakcje, przepływy i pliki PPK. Używaj przy zadaniach dotyczących integracji z myFund.pl lub danych z tego API.
metadata:
  version: "1.0.0"
  api-version: "v1"
  openapi-version: "3.1.0"
---

# myFund.pl API v1

Kompatybilność: **myFund.pl API v1**, baza `https://myfund.pl/API/v1`.
Źródło: dołączony snapshot dokumentacji z 2026-11-03. `3.1.0` oznacza
format OpenAPI, a `1.0.0` wersję skilla; żadna z tych liczb nie jest wersją API.
API jest opisane jako beta. Nie zakładaj zgodności z innymi wersjami.

## Wybierz operację

| Potrzeba | Metoda i ścieżka |
| --- | --- |
| Lista portfeli | `GET /getPortfele.php` |
| Skład, wyniki, historia, dywidendy, konta, zobowiązania | `GET /getPortfel.php` |
| Transakcje lub przepływy jako JSON | `POST /importOperacji.php` |
| Eksport brokera jako plik | `POST /importOperacjiPlik.php` |
| Eksport PPK z podziałem składek | `POST /importOperacjiPPKPlik.php` |

Przed konstruowaniem żądania przeczytaj odpowiedni endpoint i jego schematy
w [references/openapi.yaml](references/openapi.yaml). Praktyczne zasady i przykłady
znajdziesz w [references/workflows.md](references/workflows.md).
Nie wymyślaj endpointów, nazw źródeł importu ani nieudokumentowanych parametrów.

## Uwierzytelnianie i wykonanie

Klucz pochodzi z **Menu > Konto > Ustawienia konta > Klucz API**.
Regeneracja unieważnia poprzedni klucz. Używaj sekretu z otoczenia
(np. `MYFUND_API_KEY`), nie zapisuj go w skillu ani przykładach.
Nie wypisuj pełnych URL-i z kluczem, nagłówków diagnostycznych ani danych prywatnych
w logach. Nie odczytuj i nie wyświetlaj całego `.env`, aby znaleźć jeden sekret.

Odczyt wymaga `apiKey` w query. W importach używaj body JSON lub pól multipart.
Koduj parametry URL, zachowuj polskie znaki i dokładną nazwę portfela, nie jego ID.
Dla multipart pozwól klientowi wygenerować `Content-Type` z boundary.

Sprawdzaj zarówno HTTP, jak i `status.code`: `"0"` to sukces, `"1"` błąd,
`"7"` brak portfela. HTTP 200 sam nie potwierdza powodzenia.
Jeżeli odpowiedź nie zawiera oczekiwanego statusu lub danych, zgłoś niepełny wynik.
Odczyty są cache'owane przez 5 minut, importy nie. Nie obiecuj natychmiastowej
aktualizacji danych po imporcie.

## Odczyt i interpretacja

Najpierw pobierz listę portfeli, jeśli nazwa nie jest znana. Przy ograniczaniu
`scope` zawsze dodawaj `status`; wybieraj wyłącznie potrzebne pola.
`scope` działa tylko na najwyższym poziomie i rozróżnia wielkość liter.
Nieznane pola są pomijane; brak pola nie oznacza wartości zero.

Zachowuj walutę, datę i jednostki. Wartości liczbowe często są stringami,
czasem z `+`; puste wartości, `---` i `&nbsp;` traktuj jako brak danych.
Nie konwertuj całego obiektu ślepo na liczby. Do kwot używaj arytmetyki dziesiętnej.
`tickers` i historia są mapami, nie tablicami; szeregi czasowe są mapami dat.
Nie mieszaj wartości pieniężnych `struktura` z udziałami `strukturaWalory`.

## Import zmienia portfel

Samo przygotowanie danych nie upoważnia do wysłania importu. Wykonuj import,
gdy użytkownik zlecił tę operację; nie pytaj ponownie o już udzieloną zgodę.
Przed wysłaniem ustal portfel, dane, konta, źródło pliku i politykę duplikatów.
Jeśli brakuje informacji wpływającej na zapis, przygotuj podgląd i zapytaj o nią.
Nie dopisuj nieznanych cen, ilości, dat ani podatków.

Domyślnie zachowuj `duplikaty=1` i `sprawdzCene="true"`.
`duplikaty=2` może powielić operacje; używaj tylko na wyraźne zlecenie.
Nie zakładaj atomowości ani idempotencji importu. Po timeout lub zerwaniu połączenia
nie ponawiaj POST automatycznie: sprawdź historię i stan importu, uwzględniając cache.
Nie ma udokumentowanego trybu dry-run; lokalny podgląd nie jest walidacją serwera.
Po zakończeniu podaj status, liczbę operacji, pominięcia i najwcześniejszą datę,
jeżeli odpowiedź je zawiera. Rozróżniaj przygotowanie żądania od wykonanego importu.
