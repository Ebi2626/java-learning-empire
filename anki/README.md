# Fiszki Anki

Talie trzymamy jako pliki **CSV** (jeden plik = jedna faza/talia). Prosty format, łatwy do edycji
ręcznej i z notatek, działa z natywnym importem Anki.

## Format
- Standardowy **CSV (RFC 4180)**: separator **przecinek `,`**, każde pole w **cudzysłowach** `"..."`.
- Cudzysłowy chronią treść zawierającą `,` i `;` (a tych w odpowiedziach jest sporo).
- Pola: `"Przód","Tył"`. Jedna fiszka = jedna linia. Bez nagłówka.
- Wewnętrzny `"` jest escapowany jako `""` (standard CSV).
- HTML działa (np. `<code>...</code>`, `<br>`).

> **Źródłowy format w notatkach** to `Pytanie;Odpowiedź` (czytelny) — skrypt eksportujący tnie na
> **pierwszym** `;` i zamienia na cytowany CSV. Dzięki temu `;` w treści odpowiedzi nie psuje importu.

## Import do Anki (desktop)
1. `File → Import…` → wybierz plik `.csv`.
2. **Type:** Basic. **Deck:** np. `Java::00 Fundament`.
3. **Fields separated by:** Comma.
4. Zaznacz „Allow HTML in fields".
5. **Field mapping:** Field 1 → Front, Field 2 → Back.
6. Wybór duplikatów: „Update" jeśli reimportujesz po edycji.

> Wskazówka: użyj notacji talii z `::`, by mieć drzewo `Java::01 Język`, `Java::05 Spring` itd.

## Talie
| Plik | Talia | Status |
|---|---|---|
| `00-fundament.csv` | Java::00 Fundament | gotowa — 51 fiszek |
| `01-jezyk.csv` | Java::01 Język | gotowa — 152 fiszki |
| `02-jvm.csv` | Java::02 JVM | gotowa — 97 fiszek |
| `03-wspolbieznosc.csv` | Java::03 Współbieżność | gotowa — 107 fiszek |
| `04-narzedzia.csv` | Java::04 Narzędzia | gotowa — 95 fiszek |
| `05-spring.csv` | Java::05 Spring | gotowa — 148 fiszek |
| `06-persystencja.csv` | Java::06 Persystencja | gotowa — 150 fiszek |
| `07-api-design.csv` | Java::07 API design | gotowa — 97 fiszek |
| `08-bezpieczenstwo.csv` | Java::08 Bezpieczeństwo | gotowa — 87 fiszek |
| `09-architektura.csv` | Java::09 Architektura | gotowa — 105 fiszek |
| `10-cloud-native.csv` | Java::10 Cloud-native | gotowa — 125 fiszek |
| `11-flowable-bpmn.csv` | Java::11 Flowable | gotowa — 79 fiszek |
| `12-wzorce.csv` | Java::12 Wzorce | gotowa — 88 fiszek |

## Konwencja tworzenia fiszek (żeby działały)
- **Atomowość** — jedna fiszka = jeden fakt. Nie „wymień 7 rzeczy".
- **Aktywne przywołanie** — pytanie wymusza odpowiedź z głowy, nie rozpoznanie.
- **Z notatki do talii** — sekcja `🃏 Fiszki Anki` w każdej notatce to źródło; przeklej do CSV.
- **Cloze opcjonalnie** — dla definicji można użyć typu Cloze (`{{c1::...}}`), ale trzymajmy się Basic dla prostoty.
