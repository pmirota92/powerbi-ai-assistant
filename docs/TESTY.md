[TESTY.md](https://github.com/user-attachments/files/33034120/TESTY.md)
# Zestaw testowy — Conversational BI

Pytania dobrane pod model **Strategic_Profitability_Command_Center**. Kolejność od
najprostszych do najtrudniejszych — jeśli coś padnie, od razu wiadomo, na jakim poziomie
złożoności kończą się możliwości.

**Jak testować:** sprawdzaj **wygenerowany DAX**, nie tylko odpowiedź. Ładne zdanie
zbudowane na złym zapytaniu jest groźniejsze niż widoczny błąd.

---

## A. Rozgrzewka — jedna miara, bez wymiarów

Sprawdza, czy cała ścieżka działa.

| # | Pytanie | Czego oczekiwać w DAX |
|---|---|---|
| 1 | Jaki jest łączny przychód? | `[Total Revenue]` w `ROW` albo `SUMMARIZECOLUMNS` bez grupowania |
| 2 | Ile wynosi marża procentowa brutto? | `[Gross Margin %]` |
| 3 | Ile sztuk sprzedaliśmy łącznie? | `[Total Quantity]` |

---

## B. Jeden wymiar — grupowanie

Sprawdza, czy model trafia w relacje i używa kolumn z wymiarów.

| # | Pytanie | Czego oczekiwać |
|---|---|---|
| 4 | Pokaż przychód i marżę procentową według kategorii produktu | `'Dim_Product'[Category]` |
| 5 | Jaki przychód generuje każdy region? | `'Dim_Customer'[Region]` |
| 6 | Porównaj marżę procentową między segmentami klientów | `'Dim_Customer'[Segment]` |
| 7 | Pokaż przychód według poziomu produktu (Tier) | `'Dim_Product'[Tier]` |

---

## C. Ranking — TOPN i sortowanie

| # | Pytanie | Czego oczekiwać |
|---|---|---|
| 8 | Pokaż 5 produktów z najwyższym przychodem | `TOPN` albo `ORDER BY` + `TOPN` |
| 9 | Którzy klienci generują największą marżę? Pokaż 10 najlepszych | `'Dim_Customer'[Customer_Name]`, `TOPN(10, ...)` |
| 10 | Pokaż 5 produktów z najniższą marżą procentową | `TOPN` z `ASC` |

---

## D. Czas — time intelligence

Najczęstsze źródło subtelnych błędów. Sprawdzaj okresy, nie tylko składnię.

| # | Pytanie | Czego oczekiwać |
|---|---|---|
| 11 | Porównaj przychód rok do roku według kategorii | `[Revenue LY]`, `[Revenue Growth YoY %]` |
| 12 | Jak zmieniał się przychód miesiąc po miesiącu w ostatnich 12 miesiącach? | kolumny z `'Date'`, np. `[Month-Year]` lub `[Month Offset]` |
| 13 | Jaka była marża procentowa w tym i poprzednim roku? | `[Gross Margin %]` i `[Margin % LY]` albo `[Gross Margin % PY]` |
| 14 | Pokaż przychód narastająco w bieżącym kwartale | kolumny kwartału z `'Date'` |

---

## E. Filtrowanie po wyniku miary

Tu model najczęściej się wykłada — `FILTER` musi obejmować tabelę, nie stać obok niej.

| # | Pytanie | Czego oczekiwać |
|---|---|---|
| 15 | Które produkty mają marżę poniżej celu i o ile? | `[Margin Variance to Target]`, `VAR` + `FILTER` |
| 16 | Pokaż produkty, których marża spadła o więcej niż 100 punktów bazowych | `[Margin Erosion (bps)]` |
| 17 | Którzy klienci mają ujemną marżę? | `FILTER` po `[Gross Margin]` |

---

## F. Dekompozycja — pytania „dlaczego"

To jest test, czy asystent rozumie **Twój** model, a nie tylko składnię DAX.
Najlepszy materiał na demo.

| # | Pytanie | Czego oczekiwać |
|---|---|---|
| 18 | Dlaczego marża spadła? Rozbij zmianę na wpływ ceny, kosztu i wolumenu | `[Price Impact]`, `[Cost Impact]`, `[Volume Impact]` |
| 19 | Które kategorie najbardziej obniżyły marżę rok do roku? | `[Margin Erosion (bps)]` wg kategorii |
| 20 | Gdzie tracimy najwięcej na rabatach? | `[Total Discount]` albo `[Excessive Discount Impact]` |
| 21 | Gdzie widzisz największy potencjał sprzedaży dodatkowej? | `[Upsell Opportunity Potential]` |

---

## G. Testy odporności — tu ma *nie* zadziałać

Równie ważne jak poprzednie. Sprawdzasz, czy asystent przyzna się do niewiedzy,
zamiast zmyślić liczbę.

| # | Pytanie | Poprawna reakcja |
|---|---|---|
| 22 | Jaka będzie jutro pogoda? | odmowa — brak takich danych w modelu |
| 23 | Ilu mamy pracowników? | odmowa — brak tabeli z pracownikami |
| 24 | Pokaż wszystkie transakcje sprzedaży | zamiast surowej tabeli faktów: agregat albo pytanie o doprecyzowanie |
| 25 | Pokaż marżę według koloru produktu | odmowa — nie ma kolumny z kolorem |

Punkty 22–25 powinny przejść gałęzią **Fałsz** w warunku walidacji DAX (część 1b).
Jeśli którekolwiek z nich zwróci liczbę — to najpoważniejszy problem do naprawienia,
bo taka odpowiedź wygląda wiarygodnie, a jest zmyślona.

---

## Arkusz wyników

Przy każdej zmianie promptu albo modelu przelatuj tę listę i notuj:

| # | DAX poprawny | Liczba zgodna z raportem | Odpowiedź sensowna | Uwagi |
|---|---|---|---|---|
| 1 | | | | |
| 4 | | | | |
| 8 | | | | |
| 11 | | | | |
| 15 | | | | |
| 18 | | | | |
| 22 | | | | |

Jeśli spieszysz się, przetestuj po jednym pytaniu z każdej sekcji — te siedem wyżej.
Pełne 25 rób przed pokazem albo po większej zmianie w modelu semantycznym.

---

## Co zrobić z wynikami

- **Błąd składni DAX** → dopisz regułę do akcji `SYSTEM`, tak jak przy `FILTER`
- **Zła miara przy dobrej składni** → uzupełnij **opis (Description)** tej miary
  w Power BI Desktop i wygeneruj `schema.json` na nowo
- **Model myli okresy** → sprawdź, czy tabela `Date` jest oznaczona jako tabela dat
- **Sekcja G zwraca liczby** → zaostrz prompt: model ma zwracać puste `dax`,
  gdy pytanie wykracza poza schemat
