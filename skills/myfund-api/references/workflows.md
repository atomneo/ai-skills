# Procedury myFund.pl API v1

## Odczyt

`getPortfele.php` przyjmuje `apiKey` i opcjonalne `format=json`. Lista `portfele`
zawiera m.in. `nazwa`, `id`, `grupa`, `subportfel`, opcjonalnie `udostepniony`.
Pusta lista jest poprawnym wynikiem. Do kolejnych żądań używaj `nazwa`.
`getPortfel.php` wymaga `apiKey`, `portfel`, `format=json`.

Przykład Python 3 (standardowa biblioteka, klucz już ustawiony w otoczeniu):

```python
import json
import os
import urllib.error
import urllib.parse
import urllib.request

params = urllib.parse.urlencode({
    "apiKey": os.environ["MYFUND_API_KEY"],
    "portfel": "Mój Portfel",  # zastąp nazwą wybraną przez użytkownika
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
    raise SystemExit("myFund: błąd połączenia; sprawdź dostępność usługi") from None
if str(data.get("status", {}).get("code")) != "0":
    raise SystemExit("myFund: błąd API lub brak statusu; sprawdź odpowiedź lokalnie")
# Przetwarzaj data lokalnie; nie drukuj całego portfela bez potrzeby.
```

Dla gzip wyślij `Accept-Encoding: gzip` i obsłuż dekompresję klientem;
powyższy przykład nie żąda gzip. Nie dodawaj nagłówka bez obsługi odpowiedzi.

Typowe zakresy `scope` (zawsze ze `status`):

- podsumowanie: `portfel,tickers,struktura,strukturaWalory`;
- wyniki: `zyskWCzasie,wartoscWCzasie,wkladWCzasie,benchWCzasie,stopaZwrotuWCzasie`;
- historia: `historiaOperacji,historiaOperacjiLiczba` (ISIN może być pusty);
- dywidendy: `dywidendyWCzasie`;
- konta: `udzialKontInwestycyjnychWCzasie`;
- zobowiązania: `zobowiazania,zobowiazaniaLiczba,zobowiazaniaPodsumowanie`.

Historyczne udziały kont odnoszą się do sumy dodatnich wycen, dzisiejsze do sumy
netto. Brak konta w dacie oznacza zero według opisu tego raportu. Nie przenoś tej
reguły na inne brakujące pola. Nie sumuj kwot różnych walut bez przeliczenia.

## Import JSON

Wyślij JSON do `importOperacji.php`, `Content-Type: application/json`.
Wymagane: `portfel`, `typImportu`, `operacje`, a także klucz `apiKey` w body
lub query. Poniższy przykład jest podglądem danych, nie poleceniem wykonania:

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

Dla `typImportu="transakcji"` wymagane pola operacji to `data`, `nazwa`,
`operacja`, `ilosc`, `cena`. Rozpoznawanie waloru: ticker myFund, dokładna nazwa,
ISIN, nazwa zwyczajowa, następnie podobieństwo nazwy/data/cena. Preferuj jednoznaczny
identyfikator potwierdzony przez użytkownika lub dane źródłowe.

Dla `typImportu="przepływów"` wymagane są `data`, `operacja`, `kwota`;
przykładowe typy to `Wpłata`, `Wypłata`, `Koszt`, `Odsetki`.
Pozostałe pola i ich typy sprawdź w schematach `ImportTransaction` i `ImportCashflow`.
Data: `YYYY-MM-DD`. Przykład czasu w specyfikacji to `13:45:00`, choć etykieta
`format` mówi `HH-MM-SS`; stosuj przykład i odnotuj rozbieżność przy integracji.
`sprawdzCene` jest stringiem `"true"` lub `"false"`, a nie booleanem.

## Import pliku brokera

`POST /importOperacjiPlik.php`, multipart: `apiKey`, `portfel`, `plik`.
Formaty: csv, xls, xlsx, pdf, html, htm. `bank` musi odpowiadać nazwie w interfejsie
importu, np. `DM BDM (csv)`. Aliasy: `source`, `sourcePlugin`.
Bez źródła serwer używa ostatniego ustawienia portfela; nie zakładaj, że pasuje ono
do pliku. Opcjonalnie: `kontoGotowkowe`, `kontoInwestycyjne`, `duplikaty`, `sprawdzCene`.
Zachowaj oryginalny plik i jego nazwę; nie przerabiaj danych bez potrzeby.

## Import PPK

`POST /importOperacjiPPKPlik.php`, multipart: `apiKey`, `portfel`, `kontoPPK`, `plik`.
Użyj nazwy konta bez dopisków Pracownik/Pracodawca/Państwo; alias to `konto`.
Konto musi mieć aktywne wszystkie trzy subkonta w PLN. Plik ma być **mniejszy niż
5 000 000 bajtów**, w formacie csv/xls/xlsx/html/htm zgodnym ze źródłem.
Aliasy pliku: `file`, `imagefile`. `bank`: nazwa lub kod źródła PPK,
np. `PKO PPK (csv/xlsx)` lub `PKOPPK`; aliasy `source`, `sourcePlugin`.

Procenty składek (`skladkaPodPracownik`, `skladkaPodPracodawca`,
`skladkaDodPracownik`, `skladkaDodPracodawca`) są w zakresie 0–100; suma
podstawowych musi być dodatnia. Serwer używa ustawień portfela, a przy ich braku
2 / 1.5 / 2 / 1.5. Nadpisuj tylko według danych użytkownika. `force2` i `force3`
(0/1) wykluczają się. Endpoint nie zmienia zapamiętanych ustawień portfela.

W PPK `duplikaty=0` zgłasza błąd przy duplikacie, `1` pomija, `2` dodaje wszystkie.
Dla pozostałych importów dokumentacja opisuje `0` jako „nie dodawaj”, bez tej samej
gwarancji błędu. Nie ujednolicaj tych znaczeń. Wszystkie wiersze pominięte jako
duplikaty w PPK mogą dać sukces, `liczbaOperacji=0`, `minData=null`.

## Błędy

Sprawdzaj odpowiedzi opisane dla konkretnego endpointu. `getPortfele.php` opisuje
401, 403 i 502; importy 400 i 401, a PPK także 403 (blokada edycji), 405 (metoda).
Nie próbuj naprawiać 401 przez regenerację klucza bez zlecenia użytkownika.
Przy niepewnym wyniku importu zatrzymaj ponawianie i ustal, czy operacje zapisano.
Nie traktuj późniejszego odczytu z cache jako dowodu, że import nie nastąpił.
