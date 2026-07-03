---
temat: "DMN — reguły decyzyjne we Flowable"
faza: 11
status: nieopanowany
priorytet: 🟢
tags: [dmn, flowable, reguly, procesy]
powiazane: ["[[wiedza/11-flowable-bpmn/bpmn-podstawy]]", "[[wiedza/11-flowable-bpmn/flowable-spring]]", "[[wiedza/11-flowable-bpmn/multitenant-procesy]]"]
---

# DMN — reguły decyzyjne we Flowable

> **TL;DR:** **DMN** (Decision Model and Notation, standard OMG) to sposób modelowania **decyzji biznesowych** w postaci
> czytelnych **decision tables** (wejścia → wyjścia → reguły), rozstrzyganych przez **hit policy** i wyrażanych w języku
> **FEEL**. Oddziela logikę reguł od kodu i od przepływu **BPMN** — analityk/admin edytuje `.dmn` bez rekompilacji.
> W procesie **business rule task** wywołuje decyzję przez **DmnDecisionService**, przekazuje zmienne procesowe jako input
> i zapisuje wynik do zmiennej. Reguły często zmieniane + czytelność dla biznesu → DMN; złożony algorytm → kod.

## 1. Co — definicja i API

**DMN** to standard OMG (obok BPMN i CMMN) do **modelowania decyzji** — odpowiada na pytanie *„jak z zestawu danych
wejściowych wyprowadzić rozstrzygnięcie"*. Kluczowa idea: **oddzielenie trzech warstw**:

```
┌─────────────── BPMN (proces: JAK płynie praca) ───────────────┐
│  ...  →  [Business Rule Task: „Zakwalifikuj kandydata"]  → ...  │
│                         │ wywołuje                              │
│                         ▼                                       │
│            ┌──────── DMN (decyzja: CO postanowić) ────────┐     │
│            │  decision table + hit policy + FEEL          │     │
│            └──────────────────────────────────────────────┘     │
│            (kod Javy: tylko to, czego DMN nie ogarnia)          │
└────────────────────────────────────────────────────────────────┘
```

- **BPMN** mówi *kiedy* i *w jakiej kolejności* — [[wiedza/11-flowable-bpmn/bpmn-podstawy]].
- **DMN** mówi *co postanowić* dla danego wejścia — bez `if/else` rozsianego po serwisach.
- **Kod** robi resztę (I/O, integracje, algorytmy).

**Główne artefakty DMN:**

- **Decision Table** — tabela: kolumny **input** (wejścia) i **output** (wyjścia), a każdy **wiersz = reguła**.
- **Hit Policy** — reguła rozstrzygania, gdy pasuje wiele wierszy (patrz sekcja 2).
- **FEEL** (Friendly Enough Expression Language) — język wyrażeń: warunki, zakresy, porównania.
- **DRD** (Decision Requirements Diagram) — graf zależności: która decyzja korzysta z wyników innej decyzji
  i z jakich danych wejściowych (input data). Pozwala rozbić dużą decyzję na mniejsze, powiązane.

**Po co to:** reguły biznesowe **zmieniają się często** (progi wieku, wagi, kryteria), a zmiana kodu = commit + build +
deploy + testy regresyjne. DMN pozwala **analitykowi/adminowi edytować tablicę** i wdrożyć nowy `.dmn` bez ruszania kodu.
To spójne z ideą konfigurowalności **per tenant** — każdy tenant może mieć własną wersję reguł ([[wiedza/11-flowable-bpmn/multitenant-procesy]]).

## 2. Jak — tablica decyzyjna, hit policy, FEEL pod spodem

### Tablica decyzyjna (decision table)

Anatomia — przykład rekrutacyjny „Kwalifikacja kandydata":

```
Decision: kwalifikacjaKandydata            HIT POLICY: FIRST (F)
┌──────┬─────────────┬──────────────┬─────────────┬───────────────────┐
│      │  INPUT       │  INPUT       │  INPUT      │  OUTPUT           │
│ Rule │  wiek        │ doswiadczenie│ wynikTestu  │  decyzja          │
│      │ (number)     │ (number lat) │ (number %)  │ (string)          │
├──────┼─────────────┼──────────────┼─────────────┼───────────────────┤
│  1   │ < 18         │ -            │ -           │ "odrzucony"       │
│  2   │ [18..65]     │ >= 5         │ >= 80       │ "zakwalifikowany" │
│  3   │ [18..65]     │ [2..5)       │ >= 60       │ "rozmowa"         │
│  4   │ [18..65]     │ < 2          │ >= 90       │ "rozmowa"         │
│  5   │ -            │ -            │ -           │ "odrzucony"       │
└──────┴─────────────┴──────────────┴─────────────┴───────────────────┘
```

- **Input columns** — nazwa (`wiek`) + **input expression** (zwykle sama zmienna) + typ (`number`, `string`, `boolean`, `date`).
- **Output column** — nazwa (`decyzja`) + typ; opcjonalnie lista dozwolonych wartości.
- **Reguła (wiersz)** — komórki input to **warunki FEEL**; pusta komórka lub `-` oznacza „nie sprawdzaj" (zawsze pasuje).
  Wiersz „strzela" (matches), gdy **wszystkie** jego warunki input są prawdziwe. Komórka output to **wartość/wyrażenie FEEL**.

### Hit Policy — jak rozstrzygnąć wiele pasujących reguł

Klucz: co, gdy pasuje więcej niż jeden wiersz. Silnik **waliduje** to zgodnie z polityką.

| Hit policy | Skrót | Zwraca | Sens |
|---|---|---|---|
| **UNIQUE** | U | jeden wynik | Reguły **muszą być rozłączne** — może pasować co najwyżej jedna. Nakładanie się = błąd walidacji. Najbezpieczniejsze, wymusza kompletność myślenia. |
| **FIRST** | F | pierwszy pasujący | Bierze **pierwszy** wiersz od góry, resztę ignoruje. **Kolejność ma znaczenie** — świetne dla „drabinki" warunków od najostrzejszego do fallback. |
| **PRIORITY** | P | najwyższy priorytet | Może pasować wiele, ale wygrywa output o **najwyższym priorytecie** wg zadeklarowanej listy wartości output (nie wg kolejności wierszy). |
| **ANY** | A | jeden wynik | Wiele może pasować, ale **wszystkie muszą dać ten sam output** — inaczej błąd. „Kilka dróg, ten sam werdykt". |
| **COLLECT** | C | **lista** wyników | Zbiera outputy wszystkich pasujących. Warianty agregacji: `C+` suma, `C<` min, `C>` max, `C#` liczność. |
| **RULE ORDER** | R | lista w kolejności wierszy | Jak COLLECT, ale zachowuje kolejność wierszy. |

W przykładzie użyliśmy **FIRST**: reguła 1 (odsiew < 18) przed regułami [18..65], a reguła 5 to **fallback** „odrzucony".
Gdyby zależało nam na weryfikowalnej rozłączności — **UNIQUE** i jawne warunki bez luk. **PRIORITY** gdy kilka reguł
może pasować, ale mamy hierarchię werdyktów (np. „odrzucony" > „rozmowa" > „zakwalifikowany" jako priorytet decyzji).

### FEEL — język wyrażeń

**FEEL** to standardowy język DMN do zapisu warunków i wartości. Kilka konstrukcji, które musisz znać:

```feel
< 18                          // porównanie
>= 80
[18..65]                      // zakres domknięty:  18 <= x <= 65
(18..65]                      // otwarty z lewej:   18 <  x <= 65   (nawias = wyłączony kraniec)
[2..5)                        // domknięty/otwarty: 2  <= x <  5
"zakwalifikowany"             // string w cudzysłowie
not(0), not("X")              // negacja
1, 2, 3                       // lista wartości (OR): pasuje, gdy x in {1,2,3}
wiek >= 18 and wynikTestu > 70 // wyrażenie boolowskie (gdy input expression to np. `true`)
date("2026-07-03")            // typy dat, czasu, duration
```

Ważne: w komórce input pisze się **warunek** względem input expression (kolumna „wiek", komórka `[18..65]` = `wiek in [18..65]`).
W komórce output pisze się **wartość/wyrażenie**. Puste = brak ograniczenia (input) lub brak wartości (output).

### Integracja z Flowable (mechanika pod spodem)

- **Deployment `.dmn`** — plik z tablicą wdrażasz do repozytorium Flowable. Jest **wersjonowany** (kolejne deploymenty →
  nowa wersja, `latest` wygrywa) i może być **per tenant** (`tenantId`), tak samo jak definicje BPMN.
- **Business Rule Task** w BPMN wskazuje `flowable:decisionRef="kwalifikacjaKandydata"`. Silnik:
  1. bierze **zmienne procesowe** o nazwach zgodnych z input expressions (`wiek`, `doswiadczenie`, `wynikTestu`) jako input,
  2. wykonuje decyzję (ewaluacja reguł + hit policy),
  3. zapisuje output (`decyzja`) **z powrotem do zmiennej procesowej** o nazwie kolumny output.
- **DmnDecisionService** — programowe API Flowable do ewaluacji decyzji poza kontekstem procesu (np. z REST controllera).
  W Spring Boot 3 / Flowable 7 wstrzykujesz je jak inne serwisy (`RuntimeService`, `TaskService` — [[wiedza/11-flowable-bpmn/flowable-spring]]).

## 3. Dlaczego / kiedy — DMN vs kod, pułapki

**Kiedy DMN:**
- reguły **często się zmieniają** i muszą być zmieniane **bez deployu** (progi, wagi, kryteria kwalifikacji);
- reguły muszą być **czytelne dla biznesu** — analityk/audytor patrzy na tabelę, nie na kod;
- potrzebna **konfigurowalność per tenant** (różne progi dla różnych klientów);
- decyzja to głównie **mapowanie warunków na werdykt** (klasyfikacja, scoring progowy, routing).

**Kiedy zwykły kod (a nie DMN):**
- **złożona logika algorytmiczna** (pętle, rekurencja, wołanie zewnętrznych systemów, ML) — tego nie zamkniesz w tabeli;
- reguła jest stabilna i czysto techniczna (walidacja formatu, mapowanie ID);
- wydajność krytyczna w tight-loopie — narzut ewaluatora DMN bywa nieuzasadniony;
- logika wymaga stanu / efektów ubocznych — DMN ma być **czysty** (input → output, bez side effects).

**Pułapki:**
- **Rozstrzelenie logiki** — część reguł w DMN, część w kodzie, część w warunkach BPMN (sequence flow conditions). Trudno
  wtedy odpowiedzieć „gdzie zapada decyzja X". Trzymaj decyzje biznesowe w DMN, przepływ w BPMN, resztę w kodzie — świadomie.
- **Testowanie reguł** — tablica bez testów to bomba. Pisz testy jednostkowe wołające `DmnDecisionService` z zestawem
  wejść i asercją na output; pokryj granice zakresów (`18`, `65`) i przypadek **fallback**. Waliduj kompletność/rozłączność.
- **Wersjonowanie** — po zmianie `.dmn` powstaje nowa wersja; procesy w toku vs nowe instancje mogą używać różnych wersji.
  Ustal politykę (latest vs pinned) i pamiętaj o rollbacku. Per tenant komplikuje to jeszcze bardziej ([[wiedza/11-flowable-bpmn/multitenant-procesy]]).
- **Zła hit policy** — UNIQUE tam, gdzie reguły się nakładają → błędy runtime; FIRST tam, gdzie kolejność przypadkowa →
  niedeterministyczne werdykty przy edycji. Wybieraj politykę świadomie i dokumentuj.
- **Typy i null** — brak zmiennej procesowej dla input expression lub zły typ → NPE/pusty wynik. Zadbaj o defaulty.

## Przykład w praktyce (rekrutacja: automatyczna kwalifikacja kandydata)

Proces „Rekrutacja" ma **business rule task** „Zakwalifikuj kandydata", który woła decyzję DMN z sekcji 2 (hit policy FIRST).
Fragment BPMN:

```xml
<businessRuleTask id="kwalifikacja" name="Zakwalifikuj kandydata"
                  flowable:decisionRef="kwalifikacjaKandydata" />
<!-- input: zmienne procesowe wiek, doswiadczenie, wynikTestu
     output: zapisany do zmiennej procesowej `decyzja` -->
<sequenceFlow sourceRef="kwalifikacja" targetRef="bramkaDecyzja"/>
<exclusiveGateway id="bramkaDecyzja"/>
<sequenceFlow sourceRef="bramkaDecyzja" targetRef="taskRozmowa">
  <conditionExpression xsi:type="tFormalExpression">${decyzja == 'rozmowa'}</conditionExpression>
</sequenceFlow>
```

Wywołanie z poziomu procesu — po prostu startujesz proces z odpowiednimi zmiennymi:

```java
@Service
public class RekrutacjaService {

    private final RuntimeService runtimeService;
    private final DmnDecisionService dmnDecisionService; // Flowable 7 DMN

    public RekrutacjaService(RuntimeService runtimeService,
                             DmnDecisionService dmnDecisionService) {
        this.runtimeService = runtimeService;
        this.dmnDecisionService = dmnDecisionService;
    }

    /** Wariant 1: przez proces — business rule task sam wywoła DMN. */
    public void przyjmijAplikacje(int wiek, int doswiadczenie, int wynikTestu) {
        Map<String, Object> vars = Map.of(
                "wiek", wiek,
                "doswiadczenie", doswiadczenie,
                "wynikTestu", wynikTestu);
        runtimeService.startProcessInstanceByKey("rekrutacja", vars);
        // po business rule task zmienna `decyzja` = "zakwalifikowany"/"rozmowa"/"odrzucony"
    }

    /** Wariant 2: bezpośrednio przez DmnDecisionService (bez procesu, np. z REST). */
    public String oceń(int wiek, int doswiadczenie, int wynikTestu, String tenantId) {
        Map<String, Object> result = dmnDecisionService.createExecuteDecisionBuilder()
                .decisionKey("kwalifikacjaKandydata")
                .tenantId(tenantId)                 // wersja reguł per tenant
                .variable("wiek", wiek)
                .variable("doswiadczenie", doswiadczenie)
                .variable("wynikTestu", wynikTestu)
                .executeWithSingleResult();          // FIRST/UNIQUE → pojedynczy wynik
        return (String) result.get("decyzja");
    }
}
```

Analityk zmienia próg (`>= 80` → `>= 75`) w edytorze, wdraża nowy `.dmn` — **kod się nie zmienia**, a nowe instancje
procesu dostają nowe reguły. Test regresyjny (poniżej) chroni przed przypadkowym rozszczelnieniem tablicy:

```java
@Test
void granicaWieku_18_nieOdrzuca() {
    var r = dmnDecisionService.createExecuteDecisionBuilder()
            .decisionKey("kwalifikacjaKandydata")
            .variable("wiek", 18).variable("doswiadczenie", 6).variable("wynikTestu", 85)
            .executeWithSingleResult();
    assertThat(r.get("decyzja")).isEqualTo("zakwalifikowany"); // 18 wchodzi w [18..65]
}
```

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię jednym zdaniem, czym jest DMN i co oddziela (reguły od kodu i od przepływu BPMN).
- [ ] Narysuję decision table (input/output/reguły) i wskażę, kiedy wiersz „strzela".
- [ ] Wymienię hit policies i wyjaśnię różnicę UNIQUE vs FIRST vs COLLECT.
- [ ] Zapiszę warunki w FEEL, w tym zakres `[18..65]` i granice otwarte/domknięte.
- [ ] Wywołam decyzję z business rule task oraz z DmnDecisionService.
- [ ] Uzasadnię wybór DMN vs kod i wskażę pułapki (wersjonowanie, testy, hit policy).

### 🔲 Black-box check
- [ ] Jak business rule task mapuje zmienne procesowe na input i wynik na zmienną?
- [ ] Co robi silnik, gdy pasuje wiele reguł, przy UNIQUE, FIRST, COLLECT, PRIORITY?
- [ ] Czym różni się warunek w komórce input od wartości w komórce output?
- [ ] Jak działa wersjonowanie `.dmn` i co to zmienia dla procesów w toku?
- [ ] Czym jest DRD i po co dzielić decyzję na powiązane decyzje?
- [ ] Jak wygląda ewaluacja per tenant (różne wersje reguł)?

### 🎤 Pytania rekrutacyjne
- [ ] „Po co DMN, skoro to samo napiszę w if/else?"
- [ ] „Czym różni się FIRST od UNIQUE i kiedy którego użyć?"
- [ ] „Jak przetestujesz reguły decyzyjne i jak je wersjonujesz?"
- [ ] „Kiedy reguła NIE nadaje się do DMN?"
- [ ] „Jak wywołać decyzję z procesu BPMN we Flowable?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/11-flowable-bpmn.csv`.*

```
Co to DMN?;Decision Model and Notation — standard OMG do modelowania decyzji/reguł biznesowych, oddzielony od kodu i od przepływu BPMN.
Co DMN oddziela?;Logikę decyzyjną (reguły) od kodu Javy i od przepływu procesu (BPMN) — reguły w edytowalnej tablicy.
Z czego składa się decision table?;Kolumny input (wejścia), kolumny output (wyjścia) i wiersze = reguły, plus hit policy.
Kiedy reguła (wiersz) "strzela"?;Gdy WSZYSTKIE jej warunki input są prawdziwe; pusta komórka lub "-" = brak warunku (zawsze pasuje).
Co to hit policy?;Reguła rozstrzygania, gdy pasuje wiele wierszy decision table.
Hit policy UNIQUE?;Może pasować co najwyżej jedna reguła (muszą być rozłączne); nakładanie = błąd walidacji.
Hit policy FIRST?;Zwraca pierwszy pasujący wiersz od góry; kolejność wierszy ma znaczenie (dobre dla drabinki + fallback).
Hit policy PRIORITY?;Może pasować wiele, wygrywa output o najwyższym priorytecie wg listy wartości output (nie wg kolejności wierszy).
Hit policy ANY?;Wiele może pasować, ale wszystkie muszą dać ten sam output — inaczej błąd.
Hit policy COLLECT?;Zbiera outputy wszystkich pasujących w listę; warianty agregacji: C+ suma, C< min, C> max, C# liczność.
Co to FEEL?;Friendly Enough Expression Language — język wyrażeń DMN: warunki, zakresy, porównania, listy, daty.
Jak w FEEL zapisać zakres domknięty 18-65?;[18..65] (18 <= x <= 65); nawias ( ) wyłącza kraniec, np. [2..5) to 2 <= x < 5.
Co to DRD?;Decision Requirements Diagram — graf zależności między decyzjami i danymi wejściowymi; dzieli dużą decyzję na powiązane.
Jak proces BPMN wywołuje DMN we Flowable?;Business rule task z flowable:decisionRef; zmienne procesowe → input, output zapisany do zmiennej procesowej.
Które API Flowable wykonuje decyzję programowo?;DmnDecisionService (createExecuteDecisionBuilder → decisionKey, variable, tenantId, executeWithSingleResult).
Kiedy DMN, a kiedy kod?;DMN: reguły często zmieniane, czytelne dla biznesu, konfigurowalne per tenant. Kod: złożona logika algorytmiczna, side effects, wydajność krytyczna.
Jak wersjonuje się .dmn we Flowable?;Kolejny deployment tworzy nową wersję (latest wygrywa), możliwe per tenant (tenantId); uważaj na procesy w toku.
Główne pułapki DMN?;Rozstrzelenie logiki (DMN/BPMN/kod), brak testów reguł (granice + fallback), złe wersjonowanie, źle dobrana hit policy, null/typy input.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/11-flowable-bpmn/bpmn-podstawy]] · [[wiedza/11-flowable-bpmn/flowable-spring]] · [[wiedza/11-flowable-bpmn/multitenant-procesy]]
