# Conversational BI w Power Apps — aplikacja samodzielna

Kompletna instrukcja budowy asystenta jako aplikacji canvas, poza raportem Power BI.

```
Power Apps (canvas)
      │  pytanie
      ▼
Power Automate  ──►  HTTP: Claude          (pytanie + schemat → DAX)
                ──►  Power BI: Run a query (DAX → wiersze)
                ──►  HTTP: Claude          (wiersze → odpowiedź po polsku)
      │  odpowiedź + DAX + dane
      ▼
Power Apps
```

**Czas:** 2–4 godziny przy pierwszym podejściu.
**Wymagane:** środowisko z konektorami premium (Developer Plan albo trial).

---

# Przygotowanie — zanim zaczniesz klikać

Miej pod ręką:

| Co | Skąd |
|---|---|
| `schema.txt` | masz w folderze projektu |
| Klucz Claude API + doładowane kredyty | console.anthropic.com → *API Keys* / *Billing* |
| Workspace ID, Dataset ID | z adresu w serwisie Power BI |
| 3 pytania kontrolne | takie, dla których znasz oczekiwany DAX |

### Zasada `@{ }`

Wszystkie wyrażenia poniżej wpisujesz przez zakładkę **Wyrażenie** (ikona **fx**)
**bez** `@{ }` — Power Automate zamieni je w kolorowy kafelek. Jedyny wyjątek to pole
**Treść** akcji HTTP: tam jesteś w środku JSON-a i `@{outputs('...')}` wpisujesz
dosłownie jako tekst.

### Uwaga o nazwach akcji HTTP

Akcje nazywają się `GeminiDAX` i `GeminiOdpowiedz`, choć wywołują Claude — tak powstały
w pierwszej wersji przepływu. Nie zmieniaj nazw w trakcie: odwołują się do nich inne kroki
(`body('GeminiDAX')`). Na końcu możesz je przemianować i poprawić wyrażenia za jednym razem.

### Dwie zasady nazewnictwa, które oszczędzą Ci godziny

**1. Nazywaj akcje krótko i bez polskich znaków.** Power Automate buduje z nazw akcji
wyrażenia (`outputs('NazwaAkcji')`), zamieniając spacje na podkreślenia. Akcja nazwana
*„Uruchom zapytanie względem zestawu danych"* daje wyrażenie, w którym łatwo o literówkę.
Nazwij ją `WynikPBI` i po sprawie.

**2. Nazwij przepływ bez myślników** — `ConvBIZapytaj`, nie `ConvBI-Zapytaj`.
W Power Fx nazwa z myślnikiem wymaga apostrofów: `'ConvBI-Zapytaj'.Run(...)`.

---

# CZĘŚĆ 1. Przepływ w Power Automate

To jest cały backend. Zbuduj go i **przetestuj w całości**, zanim otworzysz Power Apps.

**make.powerautomate.com** → sprawdź w prawym górnym rogu, czy jesteś w środowisku
z premium → *Moje przepływy* → **Nowy przepływ** → **Przepływ natychmiastowy** →
nazwa `ConvBIZapytaj` → wyzwalacz **Power Apps (V2)** → **Utwórz**.

## 1.1 Wyzwalacz

W akcji *Power Apps (V2)* kliknij **+ Dodaj dane wejściowe** → **Tekst** → nazwij `pytanie`.

## 1.1b `Redaguj` → **PYTANIE** (pierwsza akcja pod wyzwalaczem)

Dodaj akcję **Redaguj** o nazwie `PYTANIE`, a w polu *Dane wejściowe* wstaw wartość
**z zakładki Zawartość dynamiczna** (pozycja *pytanie* w sekcji wyzwalacza) — nie wpisuj
jej ręcznie.

**Po co osobna akcja:** `pytanie` to tylko etykieta, a wewnętrzna nazwa pola bywa różna
(`text`, `text_1`…) i zależy od tego, jak wyzwalacz został utworzony. Ręcznie wpisane
`triggerBody()?['pytanie']` albo `triggerBody()?['text']` potrafi zwracać pustkę.
Wstawienie z *Zawartości dynamicznej* gwarantuje właściwe odwołanie, a dalsze kroki
odwołują się już po prostu do `outputs('PYTANIE')`.

**Objaw pomyłki:** w `PROMPT` po napisie `PYTANIE UZYTKOWNIKA:` nie ma nic, a model
odpowiada, że nie dostał pytania.

## 1.2 `Redaguj` → **SCHEMAT**

Wklej całą zawartość pliku `schema.txt`.

To jedyne miejsce, które aktualizujesz po zmianie modelu.

## 1.3 `Redaguj` → **SYSTEM**

```
Jestes ekspertem DAX i modelowania tabelarycznego Power BI. Zamieniasz pytanie
biznesowe na JEDNO zapytanie DAX wykonywane przez Power BI executeQueries.
ZASADY: uzywaj wylacznie tabel, kolumn i miar ze SCHEMATU MODELU; nigdy nie
wymyslaj nazw; zapytanie zaczyna sie od EVALUATE lub DEFINE i jest tylko odczytem;
preferuj istniejace miary zamiast pisania SUM od zera; wynik ma byc maly
i zagregowany (SUMMARIZECOLUMNS, TOPN, ORDER BY), nigdy surowa tabela faktow;
nazwy kolumn wynikowych nadawaj po polsku. Zwracasz wylacznie JSON zgodny ze schematem.
```

Nie używaj w tym tekście cudzysłowów `"` — trafi on do treści JSON.

## 1.4 `Redaguj` → **PROMPT**

Tutaj sklejasz to, co zobaczy model. W polu *Dane wejściowe* wklej wyrażenie:

```
concat('SCHEMAT MODELU:', decodeUriComponent('%0A'), outputs('SCHEMAT'), decodeUriComponent('%0A%0A'), 'PYTANIE UZYTKOWNIKA:', decodeUriComponent('%0A'), outputs('PYTANIE'))
```

`decodeUriComponent('%0A')` to sposób na wstawienie znaku nowej linii — w wyrażeniach
Power Automate nie da się go wpisać wprost.

## 1.5 `Redaguj` → **PROMPT_JSON** ← to jest krok, który wszyscy pomijają

```
replace(replace(replace(outputs('PROMPT'), '\', '\\'), '"', '\"'), decodeUriComponent('%0A'), '\n')
```

**Po co:** Twoje miary zawierają cudzysłowy — np. `SWITCH(TRUE(), _Var >= 0, "Healthy", ...)`.
Wstawione surowo do treści JSON rozwalą ją i dostaniesz `400 Bad Request` albo
*„The template validation failed"*. Ta jedna akcja zamienia `"` na `\"`, backslash na
podwójny, a znaki nowej linii na dwuznak `\n`.

To jest najczęstsza przyczyna straconego wieczoru przy tej integracji.

## 1.6 `HTTP` → **GeminiDAX** (wywołuje Claude)

- **Metoda:** `POST`
- **URI:**
  ```
  https://api.anthropic.com/v1/messages
  ```
- **Nagłówki** — trzy wiersze:

| Klucz | Wartość |
|---|---|
| `x-api-key` | Twój klucz Claude, bez cudzysłowów |
| `anthropic-version` | `2023-06-01` |
| `content-type` | `application/json` |

- **Treść:**

```json
{
  "model": "claude-haiku-4-5-20251001",
  "max_tokens": 2048,
  "system": "@{replace(replace(outputs('SYSTEM'), decodeUriComponent('%0D'), ''), decodeUriComponent('%0A'), ' ')}",
  "messages": [ { "role": "user", "content": "@{outputs('PROMPT_JSON')}" } ],
  "output_config": {
    "format": {
      "type": "json_schema",
      "schema": {
        "type": "object",
        "properties": {
          "reasoning": { "type": "string" },
          "dax": { "type": "string" },
          "chart": { "type": "string" }
        },
        "required": ["reasoning", "dax", "chart"],
        "additionalProperties": false
      }
    }
  }
}
```

- `output_config` wymusza czysty JSON zgodny ze schematem — nie trzeba wycinać bloków
  ```` ``` ```` wyrażeniami tekstowymi.
- `replace` w polu `system` usuwa znaki nowej linii z akcji `SYSTEM`. Surowa nowa linia
  w środku tekstu JSON psuje całe żądanie.
- Gdy Haiku gubi się przy trudnych pytaniach, zmień `model` na `claude-sonnet-5`
  (lepszy w DAX, około dwa razy droższy).

**Ukryj klucz:** w akcji → *…* → **Ustawienia** → **Zabezpieczone dane wejściowe = Włączone**.
Klucz przestanie być widoczny w historii uruchomień.

## 1.7 `Inicjuj zmienną` → **varDax** (typ: Ciąg)

Wartość:

```
json(last(body('GeminiDAX')?['content'])?['text'])?['dax']
```

Zmienna, a nie `Redaguj`, bo w części 3 (pętla naprawcza) będziemy ją nadpisywać.

> **Dlaczego `last()`, a nie `[0]`.** Odpowiedź Claude to lista bloków. Przy prostych
> modelach pierwszy blok jest tym właściwym, ale mocniejsze modele potrafią zwrócić
> przed nim blok rozumowania — wtedy `[0]?['text']` daje `null` i `json()` wywala się
> komunikatem *„expects its parameter to be a string… provided value is of type Null"*.
> `last()` działa w obu przypadkach.

## 1.8 `Inicjuj zmienną` → **varWiersze** (typ: Ciąg), wartość pusta

## 1.9 Power BI → **WynikPBI**

Konektor **Power BI** → akcja **Uruchom zapytanie względem zestawu danych**.
Zmień nazwę akcji na `WynikPBI`.

| Pole | Wartość |
|---|---|
| Obszar roboczy | Twój workspace |
| Zestaw danych | Twój model semantyczny |
| Tekst zapytania | `variables('varDax')` |

To konektor **standardowy**. Premium potrzebujesz wyłącznie na akcje HTTP.

## 1.10 `Ustaw zmienną` → varWiersze

```
string(body('WynikPBI')?['firstTableRows'])
```

## 1.11 `Redaguj` → **PROMPT2_JSON**

```
replace(replace(replace(concat('PYTANIE:', decodeUriComponent('%0A'), outputs('PYTANIE'), decodeUriComponent('%0A%0A'), 'WYKONANY DAX:', decodeUriComponent('%0A'), variables('varDax'), decodeUriComponent('%0A%0A'), 'WYNIK:', decodeUriComponent('%0A'), variables('varWiersze')), '\', '\\'), '"', '\"'), decodeUriComponent('%0A'), '\n')
```

Ten sam zabieg co w 1.5 — wynik z Power BI to JSON pełen cudzysłowów.

## 1.12 `HTTP` → **GeminiOdpowiedz** (wywołuje Claude)

Ten sam URI i te same trzy nagłówki co w 1.6. Treść:

```json
{
  "model": "claude-haiku-4-5-20251001",
  "max_tokens": 1024,
  "system": "Jestes analitykiem BI. Na podstawie wyniku napisz zwiezla odpowiedz po polsku, 2-5 zdan, z konkretnymi liczbami. Opieraj sie WYLACZNIE na danych z wyniku, nigdy nie zmyslaj wartosci. Jesli wynik jest pusty, napisz o tym wprost i zasugeruj, jak doprecyzowac pytanie.",
  "messages": [ { "role": "user", "content": "@{outputs('PROMPT2_JSON')}" } ]
}
```

Tu nie ma `output_config` — chcemy zwykłego tekstu, nie JSON-a.
Pamiętaj o **Zabezpieczonych danych wejściowych**.

## 1.13 `Odpowiedz aplikacji PowerApp lub przepływowi`

Trzy wyjścia typu **Tekst**:

| Nazwa | Wartość |
|---|---|
| `odpowiedz` | `last(body('GeminiOdpowiedz')?['content'])?['text']` |
| `dax` | `variables('varDax')` |
| `wynik` | `variables('varWiersze')` |

## 1.14 Test — obowiązkowo przed Power Apps

*Zapisz* → **Testuj** → *Ręcznie* → wpisz pytanie kontrolne → **Uruchom przepływ**.

Sprawdź w historii uruchomień:

- **PROMPT_JSON** — czy cudzysłowy są zamienione na `\"`
- **GeminiDAX** — status 200 i sensowny DAX w odpowiedzi
- **WynikPBI** — czy wróciły wiersze
- **Odpowiedz** — czy `odpowiedz` to zdanie po polsku z liczbami

Jeśli któryś krok czerwony, napraw teraz. Diagnozowanie tego z poziomu Power Apps
jest wielokrotnie trudniejsze.

---

# CZĘŚĆ 1b. Walidacja DAX (zrób przed Power Apps)

Bez tego przepływ jest „zielony" nawet wtedy, gdy zamiast zapytania do Power BI
poszedł komentarz albo pusty tekst. Warunek przepuszcza do Power BI tylko prawdziwe
zapytania, a w pozostałych przypadkach przekazuje dalej wyjaśnienie modelu.

1. Najedź na strzałkę **między `varWiersze` (1.8) a `WynikPBI` (1.9)** → **+** →
   **Dodaj akcję** → wyszukaj **Warunek** (*Kontrolka*)
2. Zmień nazwę na `CzyDaxPoprawny`
3. Lewe pole → zakładka **Wyrażenie** →

```
or(startsWith(toUpper(trim(variables('varDax'))), 'EVALUATE'), startsWith(toUpper(trim(variables('varDax'))), 'DEFINE'))
```

4. Operator: **jest równe** → prawe pole → **Wyrażenie** → `true`
5. Przeciągnij **`WynikPBI`** i **`Ustaw zmienną varWiersze`** (1.10) do gałęzi **Prawda**
6. W gałęzi **Fałsz** dodaj **Ustaw zmienną** → `varWiersze` → Wyrażenie:

```
concat('BRAK ZAPYTANIA. Wyjasnienie modelu: ', json(last(body('GeminiDAX')?['content'])?['text'])?['reasoning'])
```

`PROMPT2_JSON` i dalsze kroki zostają **pod** warunkiem, poza gałęziami — wykonają się
w obu przypadkach. Gdy DAX był błędny, Claude w `GeminiOdpowiedz` dostanie wyjaśnienie
zamiast danych i przekaże je użytkownikowi, zamiast udawać, że zna odpowiedź.

**Test:** wpisz w wyzwalaczu pytanie spoza modelu, np. *„Jaka będzie jutro pogoda?"* —
przebieg powinien pójść gałęzią **Fałsz**, a odpowiedź wyjaśnić, że model tych danych nie zawiera.

---

# CZĘŚĆ 2. Aplikacja canvas

Współrzędne policzone dla ekranu **Tablet 16:9 — 1366 × 768 px**. `X` to odległość
od lewej krawędzi, `Y` od górnej. Wpisujesz je w panelu właściwości po prawej.

> **Separatory w formułach.** Wszystkie formuły poniżej są zapisane dla polskich ustawień
> regionalnych: **`;` między argumentami**, **`;;` między instrukcjami**. Jeśli Twoja sesja
> Power Apps używa wersji angielskiej, zamień `;` na `,` i `;;` na `;`.
> Sprawdzisz to, wpisując `If(` — podpowiedź pokaże właściwą sygnaturę.

> **Wklejaj formuły w całość.** Kliknij w pole, **Ctrl+A**, **Delete**, dopiero potem wklej.
> Przy ręcznym pisaniu edytor sam dopisuje zamykający cudzysłów i robią się `""teksty""`.

## 2.0 Ekran i ustawienia

1. **make.powerapps.com** → **+ Utwórz** → **Pusta aplikacja** → *Aplikacja kanwy* →
   nazwa `Conversational BI`, układ **Tablet** → **Utwórz**
2. Zębatka **Ustawienia** → **Wyświetlanie** → rozmiar **16:9**, orientacja **Pozioma**,
   **Skaluj, aby dopasować** = *Włączone*
3. Zaznacz ekran w *Drzewie widoku* → właściwość **Fill**:

```powerappsfl
RGBA(250; 250; 252; 1)
```

## 2.1 Podłącz przepływ

Lewy pasek ikon → ikona **Power Automate** → **+ Dodaj przepływ** → `ConvBIZapytaj`.

Jeśli przepływu nie ma na liście, sprawdź w prawym górnym rogu, czy aplikacja i przepływ
są w **tym samym środowisku**.

## 2.2 Mapa ekranu

```
 40                                    940  972                1326
  ┌──────────────────────────────────────┐  ┌───────────────────┐
  │ lblTytul            (Y 28,  H 44)    │  │                   │
  │ lblPodtytul         (Y 74,  H 24)    │  │                   │
  ├──────────────────────────────────────┴──┤                   │
  │ txtPytanie (Y 116, H 56)    btnZapytaj  │                   │
  ├─────────────────────────────────────────┤                   │
  │ btnPrzyklad1  btnPrzyklad2  btnPrzyklad3│                   │
  │ lblStatus           (Y 230, H 22)       │  lblHistoriaNagl. │
  ├──────────────────────────────────────┐  ├───────────────────┤
  │ recOdpowiedz + htmOdpowiedz (Y 262)  │  │ galHistoria       │
  │                            H 280     │  │ (Y 292, H 440)    │
  ├──────────────────────────────────────┤  │                   │
  │ lblDaxNaglowek      (Y 554, H 22)    │  │                   │
  │ lblDax              (Y 582, H 150)   │  │                   │
  └──────────────────────────────────────┘  └───────────────────┘
```

## 2.3 Kontrolki — pozycje i rozmiary

Wstawiasz przez **+ Wstaw**, nazwę zmieniasz dwuklikiem w *Drzewie widoku*.

| Nazwa | Typ kontrolki | X | Y | Width | Height |
|---|---|---|---|---|---|
| `lblTytul` | Etykieta tekstowa | 40 | 28 | 600 | 44 |
| `lblPodtytul` | Etykieta tekstowa | 40 | 74 | 900 | 24 |
| `txtPytanie` | Wprowadzanie tekstu | 40 | 116 | 1120 | 56 |
| `btnZapytaj` | Przycisk | 1176 | 116 | 150 | 56 |
| `btnPrzyklad1` | Przycisk | 40 | 184 | 264 | 36 |
| `btnPrzyklad2` | Przycisk | 320 | 184 | 264 | 36 |
| `btnPrzyklad3` | Przycisk | 600 | 184 | 264 | 36 |
| `lblStatus` | Etykieta tekstowa | 40 | 230 | 1286 | 22 |
| `recOdpowiedz` | Prostokąt (Ikony → Prostokąt) | 40 | 262 | 900 | 280 |
| `htmOdpowiedz` | Tekst HTML | 56 | 278 | 868 | 248 |
| `lblDaxNaglowek` | Etykieta tekstowa | 40 | 554 | 300 | 22 |
| `lblDax` | Etykieta tekstowa | 40 | 582 | 900 | 150 |
| `lblHistoriaNaglowek` | Etykieta tekstowa | 972 | 262 | 354 | 22 |
| `galHistoria` | Galeria pionowa (pusta) | 972 | 292 | 354 | 440 |

`recOdpowiedz` wstaw **przed** `htmOdpowiedz`, żeby prostokąt był pod spodem. Gdyby
przykrył tekst: prawy przycisk → **Zmień kolejność** → **Wyślij do tyłu**.

## 2.4 Właściwości — do wklejenia

> **Kolejność ma znaczenie.** Power Apps zna zmienną dopiero wtedy, gdy gdzieś pojawi się
> `Set(...)`. Wpisz najpierw `App.OnStart`, zaraz po nim `btnZapytaj.OnSelect` — inaczej
> przy `ladowanie`, `blad` czy `pytanieStartowe` zobaczysz *„nie został rozpoznany"*.

### `App` → `OnStart`
```powerappsfl
Set(ladowanie; false);;
Set(blad; "");;
Set(pytanieStartowe; "")
```

Po zapisaniu: ••• przy **App** w drzewie widoku → **Uruchom OnStart**.
Zmiennej `wynik` nie inicjujesz — to rekord z przepływu, a formuły mają zabezpieczenie
`IsBlank(wynik)`.

### `lblTytul`
```powerappsfl
Text:       "Conversational BI Assistant"
Size:       28
FontWeight: FontWeight.Bold
Color:      RGBA(32; 32; 40; 1)
```

### `lblPodtytul`
```powerappsfl
Text:  "Zapytaj o dane w jezyku naturalnym - asystent zamieni pytanie na DAX i odpyta model semantyczny."
Size:  12
Color: RGBA(110; 110; 120; 1)
```

### `txtPytanie`
```powerappsfl
Default:     pytanieStartowe
HintText:    "O co chcesz zapytać model?"
Size:        14
BorderColor: RGBA(200; 200; 210; 1)
```

### `btnZapytaj`
```powerappsfl
Text:
If(ladowanie; "Pracuję..."; "Zapytaj")

DisplayMode:
If(ladowanie || IsBlank(Trim(txtPytanie.Text)); DisplayMode.Disabled; DisplayMode.Edit)

Size: 15
Fill: RGBA(56; 96; 178; 1)

OnSelect:
Set(ladowanie; true);;
Set(blad; "");;
IfError(
    Set(wynik; ConvBIZapytaj.Run(txtPytanie.Text));
    Set(blad; FirstError.Message)
);;
If(
    IsBlank(blad);
    Collect(colHistoria;
        {
            pytanie:   txtPytanie.Text;
            odpowiedz: wynik.odpowiedz;
            dax:       wynik.dax;
            czas:      Now()
        }
    )
);;
Set(ladowanie; false)
```

Blokada przycisku nie jest kosmetyką: przepływ chodzi 5–15 sekund i bez niej użytkownik
kliknie trzy razy, uruchamiając trzy przebiegi i płacąc za trzy komplety wywołań Claude.

### `lblStatus`
```powerappsfl
Text:
If(ladowanie; "Generuję DAX i odpytuję model..."; blad)

Size: 12

Color:
If(IsBlank(blad); RGBA(110; 110; 120; 1); RGBA(200; 40; 40; 1))
```

### `recOdpowiedz`
```powerappsfl
Fill:            RGBA(255; 255; 255; 1)
BorderColor:     RGBA(225; 225; 232; 1)
BorderThickness: 1
```

### `htmOdpowiedz`

Wpisz **w jednej linii** — Power Apps gubi konkatenację przy łamaniu wiersza:

```powerappsfl
HtmlText:
If(IsBlank(wynik); ""; "<div style='font-family:Segoe UI; font-size:15px; line-height:1.6; color:#202028'>" & Substitute(wynik.odpowiedz; Char(10); "<br>") & "</div>")
```

`Substitute(...; Char(10); "<br>")` zamienia znaki nowej linii na znaczniki HTML —
bez tego akapity i wypunktowania z odpowiedzi skleją się w jeden blok.

> **Uwaga na kolejność warstw.** Kontrolka wstawiona przez **+ Wstaw** zawsze ląduje
> **na wierzchu**, niezależnie od tego, w jakiej kolejności ją dodajesz. Jeśli po
> wstawieniu `recOdpowiedz` odpowiedź przestanie być widoczna: prawy przycisk na
> prostokącie → **Zmień kolejność** → **Wyślij do tyłu**. Objaw jest mylący — wygląda
> jakby przepływ nie zwracał danych, a w rzeczywistości tekst jest zasłonięty.
> Szybki test: ustaw `recOdpowiedz.Visible` na `false`.

Etykieta HTML zamiast zwykłej, bo sama zawija długi tekst i pozwala ustawić interlinię.

### `lblDaxNaglowek`
```powerappsfl
Text:       "Wygenerowany DAX"
Size:       12
FontWeight: FontWeight.Semibold
Color:      RGBA(110; 110; 120; 1)
```

### `lblDax`
```powerappsfl
Text:  If(IsBlank(wynik); ""; wynik.dax)
Font:  Font.'Consolas'
Size:  11
Color: RGBA(60; 60; 70; 1)
Fill:  RGBA(246; 246; 249; 1)
VerticalAlign: VerticalAlign.Top
```

To pole robi największe wrażenie na pokazie — widać dokładnie, jakie zapytanie
powstało z pytania zadanego po polsku.

### `lblHistoriaNaglowek`
```powerappsfl
Text:       "Historia pytań"
Size:       12
FontWeight: FontWeight.Semibold
Color:      RGBA(110; 110; 120; 1)
```

### `galHistoria`
```powerappsfl
Items:
Sort(colHistoria; czas; SortOrder.Descending)

TemplateSize:  64
ShowScrollbar: true

OnSelect:
Set(wynik; {odpowiedz: ThisItem.odpowiedz; dax: ThisItem.dax; wynik: ""})
```

W szablonie galerii (pierwsza komórka) wstaw dwie etykiety:

| Nazwa | X | Y | Width | Height | Text | Size |
|---|---|---|---|---|---|---|
| `lblHistPytanie` | 12 | 8 | 320 | 32 | `ThisItem.pytanie` | 12 |
| `lblHistCzas` | 12 | 40 | 320 | 18 | `Text(ThisItem.czas; "hh:mm")` | 10 |

## 2.5 Przykładowe pytania

Dobierz je pod **swój** model — to one decydują, czy demo wygląda na przemyślane.
Dla modelu rentowności sprawdzą się takie:

```powerappsfl
btnPrzyklad1.Text:     "Przychód wg kategorii"
btnPrzyklad1.OnSelect:
Set(pytanieStartowe; "Pokaż przychód i marżę procentową według kategorii produktu");; Reset(txtPytanie)

btnPrzyklad2.Text:     "Marża poniżej celu"
btnPrzyklad2.OnSelect:
Set(pytanieStartowe; "Które produkty mają marżę poniżej celu i o ile?");; Reset(txtPytanie)

btnPrzyklad3.Text:     "Erozja marży"
btnPrzyklad3.OnSelect:
Set(pytanieStartowe; "Pokaż 5 produktów z największą erozją marży rok do roku");; Reset(txtPytanie)

Wszystkie trzy:
Size:  11
Fill:  RGBA(235; 238; 245; 1)
Color: RGBA(56; 96; 178; 1)
```

Kliknięcie przykładu wpisuje treść do pola (bo `txtPytanie.Default` = `pytanieStartowe`),
a użytkownik może ją poprawić przed wysłaniem.

> **Kolejność jest krytyczna:** najpierw `Set`, dopiero potem `Reset`. Odwrotnie pole
> odczyta *starą* wartość domyślną i klikanie przykładów nie da żadnego efektu —
> wygląda to tak, jakby przycisk w ogóle nie działał.

## 2.6 Test

1. **F5** (albo ▷ w prawym górnym rogu)
2. Kliknij przykład albo wpisz własne pytanie → **Zapytaj**
3. Po kilkunastu sekundach w białej karcie pojawi się odpowiedź, a pod nią DAX
4. **Esc** → **Zapisz** → **Opublikuj**

### Gdy coś nie gra

| Objaw | Przyczyna |
|---|---|
| Odpowiedź pusta, a przepływ zakończył się powodzeniem | Power Apps trzyma starą sygnaturę przepływu — usuń przepływ z aplikacji (ikona Power Automate → ⋯ → *Usuń*), dodaj ponownie i wklej `OnSelect` jeszcze raz |
| Model odpowiada „brak pytania" mimo wpisanego tekstu | pytanie nie dotarło do przepływu — sprawdź w historii przebiegu wyzwalacz → *Dane wyjściowe* → `body`; jeśli `text` jest puste, to ta sama przyczyna co wyżej |
| Kliknięcie przykładu nic nie robi | odwrócona kolejność `Set` i `Reset` w `OnSelect` (patrz 2.5) |
| Przycisk **Zapytaj** jest szary | pole pytania jest puste — tak ma być, `DisplayMode` go blokuje |

---

# CZĘŚĆ 3. Pętla naprawcza (opcjonalna, ale to ona ratuje demo)

Gdy model wygeneruje DAX z błędem, przepływ bez tego zwyczajnie się wywali.
W Pythonie to pięć linijek. Tutaj — kilkanaście kliknięć.

## 3.1 Zamknij zapytanie w zakresie

Kroki **1.9** i **1.10** przeciągnij do akcji **Zakres** nazwanej `Proba1`.

## 3.2 Dodaj `Zakres` → **Naprawa**

Po zakresie `Proba1`. W jego menu *…* → **Konfiguruj uruchamianie po** →
odznacz *powodzenie*, zaznacz **zakończyło się niepowodzeniem**.

Wewnątrz `Naprawa`:

**a)** `Redaguj` → **PROMPT_FIX_JSON**
```
replace(replace(replace(concat('Poprzednia proba zakonczyla sie bledem. Popraw zapytanie DAX.', decodeUriComponent('%0A%0A'), 'SCHEMAT MODELU:', decodeUriComponent('%0A'), outputs('SCHEMAT'), decodeUriComponent('%0A%0A'), 'PYTANIE:', decodeUriComponent('%0A'), outputs('PYTANIE'), decodeUriComponent('%0A%0A'), 'POPRZEDNI DAX:', decodeUriComponent('%0A'), variables('varDax'), decodeUriComponent('%0A%0A'), 'BLAD SILNIKA:', decodeUriComponent('%0A'), string(result('Proba1'))), '\', '\\'), '"', '\"'), decodeUriComponent('%0A'), '\n')
```

`result('Proba1')` zwraca wyniki wszystkich akcji z zakresu razem z treścią błędu.
Tę treść dostaje model i na jej podstawie poprawia zapytanie.

**b)** `HTTP` → **GeminiFix** — identycznie jak `GeminiDAX`, tylko z `outputs('PROMPT_FIX_JSON')`

**c)** `Ustaw zmienną` → varDax
```
json(last(body('GeminiFix')?['content'])?['text'])?['dax']
```

**d)** Power BI → **WynikPBI2** z `variables('varDax')`

**e)** `Ustaw zmienną` → varWiersze = `string(body('WynikPBI2')?['firstTableRows'])`

## 3.3 Popraw uruchamianie dalszych kroków

Akcja **PROMPT2_JSON** (krok 1.11) → *Konfiguruj uruchamianie po* → zaznacz
**powodzenie** *oraz* **pominięto**. Dzięki temu przepływ idzie dalej niezależnie
od tego, czy naprawa była potrzebna.

## 3.4 Pokaż to w aplikacji

W `Odpowiedz` dodaj czwarte wyjście `naprawiono`:
```
if(equals(length(body('WynikPBI')?['firstTableRows']), 0), 'tak', 'nie')
```

W Power Apps możesz wtedy dopisać pod odpowiedzią informację, że zapytanie wymagało
poprawki. Na pokazie to działa zaskakująco dobrze — widać, że system sam się naprawił.

---

# Rozwiązywanie problemów

| Objaw | Przyczyna |
|---|---|
| `The template validation failed` przy zapisie | błąd w treści JSON akcji HTTP — najczęściej brakujący przecinek albo nawias |
| Claude zwraca `400` + *invalid JSON* | nieescapowane cudzysłowy albo nowe linie — sprawdź, czy w treści jest `PROMPT_JSON` (nie `PROMPT`) i `replace` w polu `system` |
| Claude zwraca `401` | zły klucz w nagłówku `x-api-key` albo spacja na początku/końcu |
| Claude zwraca `400` + *credit balance too low* | brak kredytów — doładuj w console.anthropic.com → *Billing* |
| Claude zwraca `529` *overloaded* | chwilowe przeciążenie — przywróć *Zasady ponawiania* na *Domyślne*, ponowienie zwykle przechodzi |
| Claude `429` | limit zapytań na minutę dla nowego konta — odczekaj chwilę; limity rosną wraz z wydatkami |
| `WynikPBI` zwraca `BadRequest` | model wygenerował błędny DAX — po to jest część 3 |
| `WynikPBI` zwraca 401/403 | brak uprawnienia Build na modelu albo wyłączone ustawienie tenanta *„Interfejs API REST do wykonywania zapytań na modelu semantycznym"* |
| W Power Apps `wynik.odpowiedz` jest puste | zmieniłaś nazwy wyjść w akcji Odpowiedz — usuń przepływ z aplikacji i dodaj ponownie |
| `The requested operation is invalid` | to samo — Power Apps trzyma starą sygnaturę przepływu |
| Przepływ działa w teście, w aplikacji nie | sprawdź w historii uruchomień, co dokładnie przyszło w `pytanie` |

---

# Limity, o których warto pamiętać

- **750 uruchomień przepływu miesięcznie** na Developer Plan
- **100 akcji na przebieg**, maksymalnie **120 minut** na przebieg
- Odpowiedź do Power Apps ma ograniczony rozmiar — dlatego prompt systemowy wymusza
  małe, zagregowane wyniki
- Power BI: 100 000 wierszy / 15 MB / 120 zapytań na minutę
- Claude: płatność za tokeny — przy Haiku ok. 0,7 centa za pytanie (DAX + odpowiedź); limity zapytań na minutę rosną wraz z poziomem konta

---

# Kolejność, która działa

1. Przepływ w całości, przetestowany w Power Automate (część 1)
2. Trzy kontrolki w Power Apps, bez upiększania (część 2)
3. Wskaźnik ładowania i obsługa błędów
4. Pętla naprawcza (część 3)
5. Historia, przykładowe pytania, wygląd

Nie buduj aplikacji równolegle z przepływem. Przy pierwszym błędzie nie będziesz
wiedziała, po której stronie szukać.
