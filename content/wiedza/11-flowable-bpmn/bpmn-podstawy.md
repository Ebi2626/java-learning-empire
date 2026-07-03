---
temat: "BPMN 2.0 — modelowanie procesów biznesowych"
faza: 11
status: nieopanowany
priorytet: 🟢
tags: [bpmn, flowable, procesy]
powiazane: ["[[wiedza/11-flowable-bpmn/flowable-spring]]", "[[wiedza/11-flowable-bpmn/dmn-reguly]]", "[[projekt-rekrutacja/docs/ARCHITEKTURA]]"]
---

# BPMN 2.0 — modelowanie procesów biznesowych

> **TL;DR:** **BPMN** (Business Process Model and Notation) to standard **OMG** do *graficznego* modelowania procesów
> biznesowych, który jednocześnie jest **wykonywalnym XML-em**. Ten sam diagram jest wspólnym językiem biznesu i IT
> oraz artefaktem, który silnik (**Flowable**) *uruchamia*. Proces staje się **konfigurowalnym modelem**, a nie
> rozproszonym po kodzie łańcuchem `if`-ów — admin tenantu dopracowuje przepływ bez zmiany kodu.

## 1. Co — definicja i API

**BPMN 2.0** = notacja (co narysować: kółka, prostokąty, romby) **plus** serializacja do XML o zdefiniowanej
semantyce wykonania. Kluczowa różnica względu starszych diagramów: BPMN 2.0 jest **executable** — ten sam plik
`.bpmn20.xml` jest zarówno obrazkiem dla analityka, jak i wsadem dla silnika procesowego.

Dwa poziomy, które trzeba rozdzielić:
- **Process definition** (definicja) — *szablon* procesu: sama struktura z pliku BPMN. Zdeployowana raz, wersjonowana.
- **Process instance** (instancja) — konkretne *wykonanie* definicji (jedna rekrutacja jednego kandydata).
  Z jednej definicji powstają tysiące instancji, każda z własnym stanem.

```
┌──────────────────────────────── BPMN 2.0 diagram ────────────────────────────────┐
│                                                                                   │
│   (○)start ──▶ [ user task ] ──▶ ◇ XOR ──[tak]──▶ [ service task ] ──▶ (◉)end     │
│                                    │                                              │
│                                    └──[nie]──────────────────────▶ (◉)end reject  │
│                                                                                   │
│   ○ event   □ task   ◇ gateway   → sequence flow   ● token (znacznik sterowania)  │
└───────────────────────────────────────────────────────────────────────────────────┘
```

Podstawowe **elementy** (kategorie BPMN):

- **Events (zdarzenia)** — okręgi. *Coś się dzieje.*
  - **Start event** (cienki okrąg) — wejście do procesu. Warianty: `none` (ręczny start), **message** (przyjście
    wiadomości), **timer** (o czasie/cyklicznie), **signal**, **error**.
  - **End event** (gruby okrąg) — zakończenie ścieżki.
  - **Intermediate event** (podwójny okrąg) — w trakcie: `message` (czekaj na wiadomość), `timer` (odczekaj/deadline),
    **error** (rzuć/złap błąd biznesowy), **signal** (broadcast do wielu procesów).
- **Tasks (zadania)** — zaokrąglone prostokąty. *Jednostka pracy.*
  - **User task** — czeka na *człowieka* (formularz, decyzja); trafia do listy zadań (task list), ma `assignee`/`candidateGroups`.
  - **Service task** — *automatyka*: wywołanie kodu (Java delegate / Spring bean / expression) lub REST. Bez człowieka.
  - **Script task** — inline skrypt (np. Groovy/JS).
  - **Send / Receive task** — wysłanie / oczekiwanie na komunikat.
  - **Business rule task** — deleguje decyzję do **DMN** (tablica decyzyjna) → [[wiedza/11-flowable-bpmn/dmn-reguly]].
- **Gateways (bramki)** — romby. *Rozgałęzienie/scalenie sterowania.*
  - **Exclusive / XOR** (romb z „X") — wybiera **dokładnie jedną** ścieżkę wg warunków; pierwszy prawdziwy warunek wygrywa.
  - **Parallel / AND** (romb z „+") — uruchamia **wszystkie** ścieżki równolegle; scalając, **czeka na wszystkie**.
  - **Inclusive / OR** (romb z „O") — uruchamia **każdą**, której warunek jest prawdziwy (0..n); scala czekając na aktywne.
  - **Event-based gateway** — czeka, aż nastąpi *jedno z* konkurujących zdarzeń (np. wiadomość vs timeout).
- **Flows (przepływy)** — strzałki łączące elementy.
  - **Sequence flow** — zwykłe następstwo.
  - **Conditional flow** — ma warunek (`conditionExpression`); przechodzimy tylko gdy prawdziwy.
  - **Default flow** — „w przeciwnym razie" (przekreślona strzałka) na bramce, gdy żaden warunek nie zaszedł.
- **Pools / Lanes (pule i tory)** — *kto* odpowiada. **Pool** = uczestnik/organizacja; **lane** = rola/dział wewnątrz puli
  (np. lane „Rekruter", lane „Manager"). Nie zmieniają wykonania — porządkują *odpowiedzialność*.
- **Artifacts (artefakty)** — adnotacje tekstowe, grupy, data objects: dokumentacja diagramu, nie wpływają na wykonanie.

## 2. Jak — elementy, token, zmienne pod spodem

**Token** — centralna koncepcja wykonania. Wyobraź sobie *znacznik* („żeton"), który wchodzi w start event i wędruje
strzałkami przez diagram. Element „aktywny" to ten, na którym stoi token:
- Na **user task** token *zatrzymuje się* aż człowiek ukończy zadanie (stan **wait state** — trwały punkt w bazie).
- Na **service task** token przelatuje synchronicznie (wykonaj kod → dalej), chyba że zadanie jest asynchroniczne.
- **Parallel gateway** rozdzielając *rozmnaża* token na N (fork), a scalając czeka aż wszystkie N wrócą (join).
- **XOR gateway** nie rozmnaża — kieruje jeden token w jedną gałąź.
- Instancja procesu **żyje**, dopóki istnieje choć jeden aktywny token; znika, gdy wszystkie dojdą do end eventów.

To wyjaśnia, dlaczego proces potrafi *czekać dniami*: na wait state silnik zapisuje stan tokenu i zmiennych do bazy
i „zwalnia" wątek — nic nie blokuje. To fundamentalna różnica względem zwykłego kodu, który trzymałby stos wątku.

**Process variables (zmienne procesowe)** — kontekst (mapa `nazwa → wartość`) przypięty do *instancji* procesu,
przepływający przez cały diagram. To one niosą dane biznesowe: `candidateId`, `score`, `accepted`, `stanowisko`.
- Warunki na bramkach i flow to **wyrażenia** (UEL/JUEL) czytające te zmienne: `${accepted == true}`.
- User task zapisuje do zmiennych wynik formularza; service task je czyta i modyfikuje.
- Zmienne są **serializowane do bazy** przy każdym wait state (stąd ostrożnie z dużymi obiektami — patrz dobre praktyki).

Cykl życia (na przykładzie XOR z warunkiem):
1. Deploy definicji (`.bpmn20.xml`) → silnik parsuje XML, tworzy **process definition** (wersjonowaną).
2. `startProcessInstanceByKey("rekrutacja", vars)` → nowa **instancja**, token w start event.
3. Token dociera do **exclusive gateway** → silnik ewaluuje `conditionExpression` po kolei; pierwszy `true` wyznacza gałąź;
   jeśli żaden — używa **default flow**, a gdy i tego brak → błąd wykonania.

## 3. Dlaczego / kiedy — konfigurowalność, dobre praktyki

**Po co to wszystko** — sedno dla naszego projektu:
- **Wspólny język biznes ↔ IT.** Analityk/PO i developer patrzą na *ten sam* diagram. Znika „głuchy telefon" specyfikacji.
- **Proces jako model, nie rozsiane `if`-y.** Zamiast logiki przepływu rozproszonej po serwisach i kontrolerach —
  jeden czytelny, wersjonowany artefakt. Zmiana kolejności kroków = zmiana diagramu, nie refaktor kodu.
- **Konfigurowalność bez zmian w kodzie** — **kluczowe** dla wielotenantowej rekrutacji: **admin tenantu**
  dopracowuje własny wariant procesu (dodaje etap ankiety, zmienia próg akceptacji w DMN, dokłada approvera),
  a my nie dotykamy Javy ani nie robimy deploymentu aplikacji — deployujemy nową *wersję definicji*.
- **Audyt i historia** za darmo — silnik zapisuje, kto/kiedy ukończył które zadanie (history tables).

**Dobre praktyki modelowania:**
- **Czytelność ponad wszystko** — diagram ma się mieścić „na jednym ekranie"; głębokie zagnieżdżenia wynoś do
  **call activity** (subprocesów). Jeden diagram = jeden poziom abstrakcji.
- **Nie modeluj za drobno** — nie każda linijka kodu to service task. Task = sensowny krok biznesowy, nie mikro-operacja.
- **Obsługa błędów świadomie** — używaj **error boundary events** i **timer boundary events** (np. eskalacja, gdy
  user task nie zostanie ukończony w SLA). Nie zostawiaj „martwych" gałęzi bez end eventu.
- **Bramki: warunki wyczerpujące + default flow** na XOR/inclusive, żeby token zawsze miał dokąd pójść.
- **Reguły decyzyjne → DMN**, nie do bramek. XOR mówi *którędy*; **DMN** (tablica decyzyjna) mówi *dlaczego* — logikę
  „score ≥ 80 → zaproś na rozmowę" trzymaj w [[wiedza/11-flowable-bpmn/dmn-reguly]], wołaną przez business rule task.
- **Uważaj na process variables** — trzymaj małe, prymitywne dane; duże obiekty/pliki referencjonuj przez ID, nie serializuj.
- **Nazywaj elementy** czasownikami z perspektywy biznesu („Przegląd aplikacji"), nie technicznie („callService3").

**Kiedy NIE BPMN:** dla prostego, krótkiego, czysto-technicznego przepływu bez wait states i bez udziału człowieka
narzut silnika i XML-a bywa większy niż zysk — zwykły serwis wystarczy. BPMN błyszczy przy **długo żyjących**,
**human-in-the-loop**, **konfigurowalnych** procesach — dokładnie jak rekrutacja.

## Przykład w praktyce (proces rekrutacji)

Słownie, przepływ definicji `rekrutacja`:

**Start event** (message: „złożono aplikację") → **user task „Przegląd aplikacji"** (rekruter ocenia CV, ustawia
zmienną `accepted`) → **exclusive gateway „Zaakceptowany?"**:
- gałąź `${accepted == true}` → **user task „Rozmowa"** → **business rule task** (DMN liczy `finalDecision`) →
  **service task „Wyślij ofertę"** → **end event** „Zatrudniony".
- **default flow** (odrzucony) → **service task „Wyślij podziękowanie"** → **end event** „Odrzucony".

Fragment XML BPMN — user task, po nim exclusive gateway z warunkiem:

```xml
<userTask id="przegladAplikacji" name="Przegląd aplikacji"
          flowable:candidateGroups="rekruterzy"/>

<sequenceFlow sourceRef="przegladAplikacji" targetRef="czyZaakceptowany"/>

<exclusiveGateway id="czyZaakceptowany" name="Zaakceptowany?" default="flowOdrzuc"/>

<!-- conditional flow: idziemy tędy tylko gdy zmienna accepted == true -->
<sequenceFlow id="flowAkceptuj" sourceRef="czyZaakceptowany" targetRef="rozmowa">
  <conditionExpression xsi:type="tFormalExpression">${accepted == true}</conditionExpression>
</sequenceFlow>

<!-- default flow: gdy żaden warunek nie zaszedł -->
<sequenceFlow id="flowOdrzuc" sourceRef="czyZaakceptowany" targetRef="wyslijPodziekowanie"/>
```

Silnik parsujący to (**Flowable**, [[wiedza/11-flowable-bpmn/flowable-spring]]) na `czyZaakceptowany` odczyta process
variable `accepted`, zewaluuje `${accepted == true}`; `true` → token do `rozmowa`, `false`/brak → **default** do `wyslijPodziekowanie`.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, czym jest BPMN i czemu jest *jednocześnie* diagramem i wykonywalnym XML-em.
- [ ] Odróżnię **process definition** od **process instance** i wiem, jak się mają (1 : N).
- [ ] Wymienię i rozróżnię events / tasks / gateways / flows oraz pools/lanes.
- [ ] Wyjaśnię koncept **tokenu** i czemu na user task proces potrafi czekać dniami (wait state).
- [ ] Wiem, czym są **process variables** i jak zasilają warunki bramek (`${...}`).
- [ ] Uzasadnię „konfigurowalność bez zmian w kodzie" w kontekście admina tenantu.
- [ ] Napiszę z głowy fragment XML: user task + exclusive gateway z warunkiem i default flow.

### 🔲 Black-box check
- [ ] Co dokładnie robi silnik przy deployu pliku `.bpmn20.xml`, a co przy `startProcessInstance`?
- [ ] Jak token przechodzi przez **parallel gateway** (fork/join) vs **exclusive** (jedna gałąź)?
- [ ] Gdzie i kiedy zapisywany jest stan tokenu i zmiennych? (wait state → baza)
- [ ] Kiedy zadanie **service task** jest asynchroniczne i po co? (transaction boundary)
- [ ] Czym różni się **user task** (wait) od **service task** (przelot)?
- [ ] Co się stanie na XOR, gdy żaden warunek nie jest prawdziwy i nie ma default flow?

### 🎤 Pytania rekrutacyjne
- [ ] „Czym jest BPMN i po co go używać zamiast kodować proces w Javie?"
- [ ] „Różnica: process definition vs process instance?"
- [ ] „Exclusive vs parallel vs inclusive gateway — kiedy który?"
- [ ] „Jak proces może czekać na człowieka przez tydzień bez blokowania wątku?" (token + wait state + baza)
- [ ] „Gdzie umieścisz regułę biznesową 'score ≥ 80'?" (DMN / business rule task, nie w bramce)

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/11-flowable-bpmn.csv`.*
```
Co to BPMN 2.0?;Standard OMG do graficznego modelowania procesów biznesowych, będący jednocześnie wykonywalnym XML-em.
BPMN jest tylko rysunkiem?;Nie — BPMN 2.0 jest executable: ten sam plik .bpmn20.xml jest diagramem i wsadem dla silnika procesowego.
Process definition vs process instance?;Definicja to wersjonowany szablon z pliku BPMN; instancja to konkretne wykonanie (relacja 1:N).
Co reprezentuje token w BPMN?;Znacznik sterowania wędrujący strzałkami przez diagram; aktywny jest element, na którym stoi token.
Kiedy znika instancja procesu?;Gdy wszystkie aktywne tokeny dotrą do end eventów (nie ma już żadnego aktywnego tokenu).
Czym jest wait state?;Trwały punkt zatrzymania (np. user task), gdzie silnik zapisuje stan do bazy i zwalnia wątek.
User task vs service task?;User task czeka na człowieka (wait state); service task wykonuje kod automatycznie i token przelatuje.
Do czego służy business rule task?;Deleguje decyzję do tablicy decyzyjnej DMN (logika reguł poza diagramem BPMN).
Exclusive (XOR) gateway — działanie?;Wybiera dokładnie jedną ścieżkę wg warunków; pierwszy prawdziwy warunek wygrywa.
Parallel (AND) gateway — działanie?;Fork uruchamia wszystkie gałęzie równolegle; join czeka na wszystkie tokeny.
Inclusive (OR) gateway — działanie?;Uruchamia każdą gałąź, której warunek jest prawdziwy (0..n); scala czekając na aktywne.
Event-based gateway — do czego?;Czeka aż nastąpi jedno z konkurujących zdarzeń (np. wiadomość vs timeout).
Conditional flow vs default flow?;Conditional ma conditionExpression i wymaga true; default ("w przeciwnym razie") jest brany, gdy żaden warunek nie zaszedł.
Co to process variables?;Mapa nazwa→wartość przypięta do instancji, przepływająca przez proces i zasilająca warunki bramek (${...}).
Pool vs lane w BPMN?;Pool to uczestnik/organizacja; lane to rola/dział wewnątrz puli — porządkują odpowiedzialność, nie wykonanie.
Typy zdarzeń (events) w BPMN?;message, timer, error, signal (jako start/intermediate/end).
Dlaczego BPMN daje konfigurowalność bez zmian w kodzie?;Zmiana procesu = nowa wersja definicji (diagramu), nie refaktor Javy — admin tenantu dopracowuje przepływ sam.
Co robi silnik na XOR, gdy żaden warunek nie jest true i brak default?;Zgłasza błąd wykonania — token nie ma dokąd pójść.
Który silnik wykonuje BPMN w projekcie?;Flowable (embedded process engine w Spring Boot).
Dobra praktyka: gdzie trzymać logikę reguł?;W DMN (business rule task), a nie w warunkach bramek — bramka mówi którędy, DMN dlaczego.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/11-flowable-bpmn/flowable-spring]] · [[wiedza/11-flowable-bpmn/dmn-reguly]] · [[projekt-rekrutacja/docs/ARCHITEKTURA]]
