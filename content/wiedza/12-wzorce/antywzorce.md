---
temat: "Antywzorce i code smells (czego unikać)"
faza: 12
status: nieopanowany
priorytet: 🟡
tags: [java, wzorce, antywzorce, jakosc]
powiazane: ["[[wiedza/12-wzorce/refaktoryzacja]]", "[[wiedza/05-spring/ioc-di]]", "[[wiedza/09-architektura/ddd-lite]]", "[[wiedza/07-api-design/dto-mapstruct]]"]
---

# Antywzorce i code smells (czego unikać)

> **TL;DR:** **Antywzorzec** to rozwiązanie, które *wygląda* rozsądnie, a systematycznie **szkodzi** (God Object, Anemic Domain Model, Service Locator, field injection). **Code smell** to nie błąd, lecz **sygnał ostrzegawczy** w kodzie (Long Method, Magic Numbers, Feature Envy) sugerujący ukryty problem. Lekarstwem są **SOLID / DRY / KISS / YAGNI / prawo Demeter** oraz systematyczna [[wiedza/12-wzorce/refaktoryzacja|refaktoryzacja]].

## 1. Co — definicja i API

- **Antywzorzec (anti-pattern)** — powtarzalny, *pozornie dobry* sposób rozwiązania problemu, który w praktyce
  generuje więcej kosztów niż korzyści (utrudnia zmiany, testy, zrozumienie). Kluczowe: **wygląda jak wzorzec**,
  ktoś świadomie go stosuje, a mimo to szkodzi. Przykład: `Singleton` użyty jako globalny stan.
- **Code smell (zapach kodu)** — **objaw**, nie choroba. To fragment kodu, który *nie jest błędem* (kompiluje się,
  testy przechodzą), ale **sygnalizuje** głębszy problem projektowy. Termin z *Refactoring* (Fowler/Beck).
  Przykład: metoda na 300 linii = smell „Long Method"; sam w sobie nie łamie działania, ale utrudnia zmiany.

Różnica: smell to **czerwona lampka do zbadania**; antywzorzec to **konkretna, nazwana pułapka projektowa**.
Wiele antywzorców objawia się przez smells. Reakcją na oba jest refaktoryzacja pod osłoną testów.

```java
// Smell: Magic Number + Long Parameter List
if (order.getStatus() == 3) { applyDiscount(o, 0.15, true, false, 5, "PLN"); }
// vs czytelnie: stała nazwana + obiekt parametrów
if (order.getStatus() == OrderStatus.SHIPPED) { pricing.apply(new DiscountRequest(...)); }
```

## 2. Jak — konkretne antywzorce i naprawy

### Antywzorce projektowe (OO / architektura)

- **God Object / God Class** — jedna klasa „wie i robi wszystko" (setki linii, wiele odpowiedzialności).
  *Naprawa:* wydziel odpowiedzialności do osobnych klas (Single Responsibility), *Extract Class*, kompozycja.
- **Anemic Domain Model** — encje to worki getterów/setterów **bez logiki**, cała reguła biznesowa siedzi
  w serwisach. To **debata**: w podejściu proceduralnym/CRUD często akceptowalne i praktyczne; w DDD to antywzorzec,
  bo model traci enkapsulację i staje się „transaction script". *Naprawa:* przenieś inwarianty do encji/agregatu,
  metody typu `order.confirm()` zamiast `orderService.confirm(order)`. Zob. [[wiedza/09-architektura/ddd-lite]].
- **Spaghetti Code** — brak struktury, sterowanie skacze wszędzie, silne sprzężenie, brak warstw.
  *Naprawa:* wprowadź warstwy/moduły, wyodrębnij metody, testy charakteryzujące przed zmianą.
- **Service Locator** — komponent sam „wyciąga" zależności z globalnego rejestru (`locator.get(Foo.class)`).
  Zależności są **ukryte** (nie widać ich w konstruktorze), trudniej testować (trzeba stubować locator).
  *Naprawa:* **Dependency Injection** — kontener wstrzykuje jawne zależności przez konstruktor
  ([[wiedza/05-spring/ioc-di]]). DI robi zależności widocznymi i testowalnymi bez globalnego stanu.
- **Singleton jako ukryty globalny stan** — statyczny `getInstance()` niosący mutowalny stan = globalna zmienna
  w przebraniu: utrudnia testy (stan przecieka między testami), ukrywa zależności, psuje wielowątkowość.
  *Naprawa:* zwykły bean o zasięgu singleton **zarządzany przez kontener DI** (jeden obiekt, ale wstrzykiwany,
  nie globalny), stan trzymaj jawnie.

### Code smells (poziom metody/klasy)

- **Magic Numbers / Strings** — literały `86400`, `"ADMIN"` rozsiane po kodzie. *Naprawa:* nazwane stałe /
  `enum` / konfiguracja (`Duration.ofDays(1)`, `Role.ADMIN`).
- **Primitive Obsession** — reprezentowanie pojęć domenowych gołymi `String`/`int` (np. `String email`,
  `long amountInCents`). Brak walidacji i typowania, łatwo pomylić argumenty. *Naprawa:* **value objects** /
  `record` (`record Email(String value){...}`, `record Money(long cents, Currency ccy){}`), które enkapsulują
  walidację i niosą znaczenie. Zob. [[wiedza/01-jezyk/record-sealed-enum]].
- **Long Method** — metoda robi za dużo. *Naprawa:* *Extract Method*, jedna metoda = jeden poziom abstrakcji.
- **Long Parameter List** — 5+ parametrów, łatwo pomylić kolejność. *Naprawa:* **obiekt parametrów** (Parameter
  Object) lub **Builder** dla opcjonalnych pól.
- **Feature Envy** — metoda bardziej interesuje się danymi *innej* klasy niż własnej (ciągłe `other.getX().getY()`).
  *Naprawa:* przenieś metodę tam, gdzie są dane (*Move Method*).
- **Shotgun Surgery** — jedna zmiana biznesowa wymaga drobnych edycji w kilkunastu miejscach. *Naprawa:* skonsoliduj
  rozproszoną odpowiedzialność w jednym module/klasie (przeciwieństwo Divergent Change).
- **Duplicated Code** — ten sam fragment skopiowany. Łamie **DRY**; poprawka w jednym miejscu omija resztę.
  *Naprawa:* wyodrębnij wspólną metodę/klasę/strategię.
- **Premature Optimization** — komplikowanie kodu „na wszelki wypadek" bez pomiaru („root of all evil", Knuth).
  *Naprawa:* najpierw czytelność i poprawność, optymalizuj **po** profilowaniu realnego wąskiego gardła (YAGNI).

### Antywzorce spring-specyficzne (Spring Boot 3, Java 21)

- **Field injection** (`@Autowired` na polu) — ukrywa zależności, uniemożliwia `final`, utrudnia testy
  jednostkowe bez kontenera, pozwala tworzyć obiekt w stanie niepełnym. *Naprawa:* **constructor injection**
  (patrz przykład niżej). Zob. [[wiedza/05-spring/ioc-di]].
- **Fat Controller** — logika biznesowa/transakcje/mapowanie w `@RestController`. *Naprawa:* kontroler tylko
  przyjmuje żądanie, waliduje, deleguje do serwisu i zwraca odpowiedź (cienka warstwa web).
- **Encje w warstwie web** — zwracanie/przyjmowanie encji JPA przez API. Wycieka model persystencji, ryzyko
  lazy-loading poza sesją, over-posting, sprzężenie kontraktu z bazą. *Naprawa:* **DTO** + mapper
  ([[wiedza/07-api-design/dto-mapstruct]]).
- **Catch-and-swallow** — `catch (Exception e) {}` (pusty blok albo tylko `e.printStackTrace()`). Gubi błędy,
  maskuje awarie. *Naprawa:* obsłuż sensownie lub przepuść dalej (rethrow), zaloguj z kontekstem, nie połykaj.
  Zob. [[wiedza/01-jezyk/wyjatki]].
- **N+1 select** — 1 zapytanie po listę + N zapytań po relacje encji. *Naprawa:* `JOIN FETCH` /
  `@EntityGraph` / batch fetch. Zob. [[wiedza/06-persystencja/n-plus-1]].
- **@Transactional na złej warstwie / self-invocation** — adnotacja na kontrolerze/repo, albo wywołanie
  metody `@Transactional` z **tej samej klasy** (proxy się nie aktywuje → transakcja nie startuje). *Naprawa:*
  transakcje na warstwie serwisu; unikaj self-invocation (wydziel metodę do innego beana).
  Zob. [[wiedza/06-persystencja/transakcje]].
- **Distributed Monolith** — mikroserwisy współdzielące bazę / synchronicznie łańcuchowo od siebie zależne:
  koszt rozproszenia bez korzyści (deploy razem, awaria kaskadowa). *Naprawa:* granice wg domeny, autonomia
  danych, komunikacja asynchroniczna. Zob. [[wiedza/09-architektura/monolit-vs-mikroserwisy]].
- **Logowanie sekretów** — hasła, tokeny, PII w logach. *Naprawa:* maskowanie, nie loguj wrażliwych pól,
  osobne poziomy. Zob. [[wiedza/04-narzedzia/logowanie]].

## 3. Dlaczego / kiedy — SOLID / DRY / KISS / YAGNI / Demeter

Zasady, które systematycznie zapobiegają powyższym pułapkom:

- **SOLID:**
  - **S — Single Responsibility:** klasa ma jeden powód do zmiany (lek na God Object, Divergent Change).
  - **O — Open/Closed:** otwarte na rozszerzenie, zamknięte na modyfikację (dodawaj przez polimorfizm/strategię,
    nie przez rozrastające się `if/switch`).
  - **L — Liskov Substitution:** podtyp da się wstawić za typ bazowy bez łamania kontraktu.
  - **I — Interface Segregation:** wiele wąskich interfejsów zamiast jednego „grubego" (klient nie zależy od
    metod, których nie używa).
  - **D — Dependency Inversion:** zależ od abstrakcji, nie od konkretów — fundament DI (lek na Service Locator).
- **DRY** (Don't Repeat Yourself) — jedna reprezentacja wiedzy; lek na Duplicated Code i Shotgun Surgery.
  Uwaga: nie „DRY na siłę" — przypadkowa zbieżność kodu to nie duplikacja wiedzy.
- **KISS** (Keep It Simple) — najprostsze działające rozwiązanie; lek na Spaghetti i przeinżynierowanie.
- **YAGNI** (You Aren't Gonna Need It) — nie buduj pod hipotetyczną przyszłość; lek na Premature Optimization
  i spekulatywną generyczność.
- **Prawo Demeter (Law of Demeter, „mów do przyjaciół, nie do obcych")** — obiekt woła tylko: własne metody,
  swoje pola, argumenty i tworzone lokalnie obiekty. Łańcuchy `a.getB().getC().getD()` (train wreck) łamią je
  i zwiększają sprzężenie; lek na Feature Envy i kruche zależności.

**Kiedy NIE walczyć na siłę:** Anemic Model bywa OK w prostym CRUD; „duplikacja" w testach bywa czytelniejsza
niż nadmierna abstrakcja; nie każdy `if` to smell. Zasady to heurystyki, nie dogmaty — celem jest **koszt zmiany**,
nie estetyczny puryzm.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

W typowym Spring Boot API code review wychwytuje: encję JPA zwracaną z kontrolera (→ DTO), `@Autowired` na polu
(→ konstruktor), pusty `catch`, `@Transactional` na kontrolerze i pętlę generującą N+1. To codzienne, „zwykłe"
decyzje, które kumulują dług techniczny.

**God class → rozbicie:**
```java
// PRZED: God Object — wszystko w jednym
class OrderService {
    void place(Order o){ validate(o); calcTax(o); charge(o); sendEmail(o); writeAudit(o); }
    // + walidacja, podatki, płatność, mail, audyt... 800 linii
}
// PO: SRP — każda odpowiedzialność osobno, komponowane przez DI
class OrderService {
    private final OrderValidator validator;
    private final TaxCalculator tax;
    private final PaymentGateway payment;
    private final NotificationService notifier;
    OrderService(OrderValidator v, TaxCalculator t, PaymentGateway p, NotificationService n){
        this.validator=v; this.tax=t; this.payment=p; this.notifier=n;
    }
    void place(Order o){ validator.validate(o); tax.apply(o); payment.charge(o); notifier.confirm(o); }
}
```

**Field injection → constructor injection:**
```java
// ANTYWZORZEC: field injection — ukryta zależność, brak final, trudny test
@Service
class ReportService {
    @Autowired private ReportRepository repo;   // nie da się final, potrzeba kontenera by wstrzyknąć
}
// NAPRAWA: constructor injection — jawne, final, testowalne bez Springa
@Service
class ReportService {
    private final ReportRepository repo;
    ReportService(ReportRepository repo){ this.repo = repo; } // jeden konstruktor → @Autowired zbędne
}
```

---

## ✅ Kryteria opanowania
- [ ] Rozróżnię antywzorzec od code smell (pułapka projektowa vs sygnał ostrzegawczy).
- [ ] Wskażę God Object w kodzie i rozbiję go wg SRP.
- [ ] Wyjaśnię, czemu constructor injection > field injection i przepiszę przykład.
- [ ] Wymienię 3 antywzorce spring-specyficzne i ich naprawę.
- [ ] Podam literę SOLID adresującą dany smell.
- [ ] Wiem, kiedy Anemic Model / „naruszenie DRY" jest akceptowalne.

### 🔲 Black-box check
- [ ] Czym różni się smell od antywzorca i od zwykłego buga?
- [ ] Dlaczego Service Locator jest gorszy od DI mimo że „też wstrzykuje"?
- [ ] Czemu `@Transactional` nie działa przy self-invocation? (proxy)
- [ ] Co to Primitive Obsession i jak `record` value object ją leczy?
- [ ] Na czym polega prawo Demeter i czym jest „train wreck"?
- [ ] Czemu zwracanie encji JPA z kontrolera to zły pomysł?

### 🎤 Pytania rekrutacyjne
- [ ] „Wymień kilka code smells i jak je usuwasz."
- [ ] „Field injection czy constructor injection — dlaczego?"
- [ ] „Co to SOLID? Podaj literę na przykładzie."
- [ ] „Anemic Domain Model — antywzorzec czy nie?"
- [ ] „Jak wykryć i naprawić N+1?"
- [ ] „Czym jest distributed monolith?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/12-wzorce.csv`.*
```
Czym jest antywzorzec?;Powtarzalnym, pozornie dobrym rozwiązaniem, które systematycznie szkodzi (utrudnia zmiany/testy).
Czym jest code smell?;Sygnałem ostrzegawczym w kodzie — nie błędem, lecz objawem głębszego problemu projektowego.
Smell vs antywzorzec?;Smell to czerwona lampka do zbadania; antywzorzec to konkretna, nazwana pułapka projektowa.
Co to God Object i jak naprawić?;Klasa robiąca wszystko; rozbić wg Single Responsibility (Extract Class, kompozycja).
Co to Anemic Domain Model?;Encje bez logiki, reguły w serwisach; w DDD antywzorzec, w prostym CRUD często akceptowalny.
Co to Primitive Obsession i lek?;Reprezentowanie pojęć domenowych gołymi String/int; lek: value objects / record.
Co to Long Parameter List i lek?;Zbyt wiele parametrów; lek: obiekt parametrów (Parameter Object) lub Builder.
Co to Feature Envy?;Metoda bardziej interesuje się danymi innej klasy niż własnej; lek: Move Method.
Co to Shotgun Surgery?;Jedna zmiana biznesowa wymaga edycji w wielu miejscach; lek: skonsolidować odpowiedzialność.
Co to Magic Number/String i lek?;Literał bez nazwy w kodzie; lek: nazwana stała / enum / konfiguracja.
Dlaczego Singleton bywa antywzorcem?;Bo jako statyczny mutowalny stan = globalna zmienna: psuje testy i ukrywa zależności.
Service Locator vs DI?;Locator ukrywa zależności i wymaga globalnego rejestru; DI wstrzykuje jawne zależności — testowalne.
Field injection vs constructor injection?;Konstruktor: jawne zależności, pola final, test bez kontenera; pole: ukryte, brak final.
Czemu nie zwracać encji JPA z kontrolera?;Wyciek modelu persystencji, lazy poza sesją, over-posting; użyj DTO + mapper.
Co to catch-and-swallow?;Złapanie wyjątku i zignorowanie (pusty catch); maskuje błędy — obsłuż lub przepuść dalej.
Czemu @Transactional nie działa przy self-invocation?;Bo wywołanie z tej samej klasy omija proxy Springa — transakcja nie startuje.
Co to distributed monolith?;Mikroserwisy sprzężone (wspólna baza / synchroniczne łańcuchy) — koszt rozproszenia bez korzyści.
Rozwiń SOLID.;Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion.
Co znaczy DRY, KISS, YAGNI?;Nie powtarzaj wiedzy; trzymaj prosto; nie buduj pod hipotetyczną przyszłość.
Co mówi prawo Demeter?;Wołaj tylko własne metody/pola, argumenty i lokalnie tworzone obiekty — unikaj a.getB().getC().
Co to Premature Optimization?;Komplikowanie kodu bez pomiaru; optymalizuj po profilowaniu realnego wąskiego gardła (YAGNI).
Co to N+1 i lek?;1 zapytanie po listę + N po relacje; lek: JOIN FETCH / @EntityGraph / batch fetch.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/12-wzorce/refaktoryzacja]] · [[wiedza/05-spring/ioc-di]] · [[wiedza/09-architektura/ddd-lite]] · [[wiedza/01-jezyk/record-sealed-enum]] · [[wiedza/07-api-design/dto-mapstruct]] · [[wiedza/01-jezyk/wyjatki]] · [[wiedza/06-persystencja/n-plus-1]] · [[wiedza/06-persystencja/transakcje]] · [[wiedza/09-architektura/monolit-vs-mikroserwisy]] · [[wiedza/04-narzedzia/logowanie]]
