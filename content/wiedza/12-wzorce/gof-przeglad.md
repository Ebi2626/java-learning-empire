---
temat: "Wzorce projektowe GoF — przegląd"
faza: 12
status: nieopanowany
priorytet: 🟡
tags: [java, wzorce, gof]
powiazane: ["[[wiedza/12-wzorce/antywzorce]]", "[[wiedza/05-spring/proxy-aop]]", "[[wiedza/09-architektura/warstwy-hexagonal]]", "[[wiedza/01-jezyk/pattern-matching]]"]
---

# Wzorce projektowe GoF — przegląd

> **TL;DR:** Wzorce projektowe (**design patterns**) to **sprawdzone, nazwane rozwiązania powtarzalnych
> problemów projektowych** oraz **wspólny język** zespołu ("użyjmy Strategy"). GoF dzieli je na trzy kategorie:
> **kreacyjne** (jak tworzyć obiekty), **strukturalne** (jak je składać), **behawioralne** (jak współpracują).
> To **narzędzie, nie cel** — nie stosuj na siłę ([[wiedza/12-wzorce/antywzorce]]), a nowoczesna Java
> (lambdy, rekordy, sealed) zmienia realizację wielu z nich.

## 1. Co — definicja i kategorie

**"Gang of Four"** = czterej autorzy (Gamma, Helm, Johnson, Vlissides) książki *Design Patterns* (1994),
która skatalogowała **23 wzorce**. Wzorzec to nie gotowa biblioteka ani kod do skopiowania — to **opis
problemu i sprawdzonego schematu rozwiązania**, który adaptujesz do swojego kontekstu. Dwie wartości:

1. **Rozwiązanie** — ktoś już przemyślał ten problem; unikasz wynajdywania koła i typowych pułapek.
2. **Wspólny słownik** — "to jest fasada nad tym podsystemem" niesie więcej informacji niż akapit opisu.
   Nazwa wzorca to skrót myślowy w code review i na tablicy.

Trzy kategorie GoF:

```
┌─────────────── KREACYJNE (tworzenie obiektów) ────────────────┐
│  Singleton · Factory Method · Abstract Factory · Builder ·    │
│  Prototype                                                    │
├─────────────── STRUKTURALNE (składanie obiektów) ─────────────┤
│  Adapter · Decorator · Facade · Proxy · Composite ·           │
│  Bridge · Flyweight                                           │
├─────────────── BEHAWIORALNE (współpraca / algorytmy) ─────────┤
│  Strategy · Observer · Template Method · Iterator · Command · │
│  State · Chain of Responsibility · Visitor · Mediator ·       │
│  Memento · Interpreter                                        │
└───────────────────────────────────────────────────────────────┘
```

**Kluczowa obserwacja:** większość wzorców już *jest* w JDK i Springu. Nie uczysz się ich od zera —
uczysz się je **rozpoznawać** i **nazywać**.

## 2. Jak — kategorie i przykłady z JDK/Springa

### KREACYJNE

- **Singleton** — problem: potrzebujesz **dokładnie jednej** instancji i globalnego dostępu.
  Klasyczna realizacja (prywatny konstruktor + statyczne pole) ma **pułapki**: łamie testowalność
  (globalny stan), bywa niepoprawny wielowątkowo (double-checked locking wymaga `volatile`),
  utrudnia mockowanie. Najbezpieczniejsza wersja to **enum singleton** (JVM gwarantuje jednokrotną
  inicjalizację i odporność na serializację/refleksję).
  **Uwaga terminologiczna:** *singleton bean* w Springie to **inne pojęcie** — to **scope** kontenera
  (jedna instancja **na kontekst**, nie globalnie na JVM), zarządzana przez DI, w pełni testowalna
  i mockowalna. Spring daje korzyści singletona bez wad wzorca GoF.
- **Factory Method / Simple Factory** — problem: nie chcesz w kodzie klienta wołać `new Konkretna()`,
  bo wybór typu ma być scentralizowany/podmienny. *Simple Factory* to statyczna metoda zwracająca produkt
  (nie jest formalnym wzorcem GoF, ale bardzo częsta). *Factory Method* deleguje wybór do podklasy.
  JDK: `List.of(...)`, `Integer.valueOf(int)`, `Optional.of(...)`, `Calendar.getInstance()`,
  `Executors.newFixedThreadPool(...)`.
- **Abstract Factory** — problem: tworzysz **rodziny** powiązanych obiektów bez wiązania z konkretnymi
  klasami. JDK: `DocumentBuilderFactory` / `TransformerFactory` (rodziny parserów XML),
  `javax.xml.parsers.*`.
- **Builder** — problem: klasa z **wieloma polami** (część opcjonalnych) → teleskopowe konstruktory
  są nieczytelne. Builder daje **czytelne, płynne (fluent) tworzenie krok po kroku** i pozwala na
  obiekty niemutowalne. JDK: `StringBuilder`, `Stream.Builder`, `HttpRequest.newBuilder()`,
  `Locale.Builder`. Lombok automatyzuje to adnotacją `@Builder`. (Przykład kodu niżej.)
- **Prototype** — problem: tworzenie obiektu od zera jest kosztowne → **klonujesz** istniejący.
  JDK: `Cloneable`/`Object.clone()` (dziś raczej unikany — płytka kopia, brak konstruktora;
  preferuj **copy constructor** lub `record` z metodą kopiującą). Spring: bean scope `prototype`
  (nowa instancja przy każdym `getBean`).

### STRUKTURALNE

- **Adapter** — problem: masz obiekt o interfejsie A, a kod oczekuje interfejsu B → **adapter tłumaczy**
  jeden na drugi. JDK: `Arrays.asList()` (tablica → `List`), `InputStreamReader` (bajty → znaki).
  Architektura **hexagonal** ([[wiedza/09-architektura/warstwy-hexagonal]]): adaptery to warstwa
  łącząca porty domeny z konkretną technologią (REST/JPA/Kafka).
- **Decorator** — problem: chcesz **dynamicznie dodać zachowanie** obiektowi bez dziedziczenia i bez
  puchnięcia hierarchii klas. Wzorzec **opakowuje** obiekt tym samym interfejsem. Podręcznikowy przykład:
  `java.io` — `new BufferedReader(new FileReader(...))`; `BufferedReader` opakowuje dowolny `Reader`
  i dokłada buforowanie. Analogicznie `BufferedInputStream`, `GZIPInputStream`. (Przykład kodu niżej.)
- **Facade** — problem: podsystem jest złożony → dajesz **uproszczony interfejs** ukrywający szczegóły.
  W Springu warstwa `@Service` często pełni rolę fasady: jedna metoda `złóżZamówienie()` koordynuje
  repozytoria, walidację, płatności i zdarzenia, chowając to przed kontrolerem.
- **Proxy** — problem: potrzebujesz **zastępnika** kontrolującego dostęp do obiektu (lazy loading,
  cache, bezpieczeństwo, transakcje). **Spring AOP** ([[wiedza/05-spring/proxy-aop]]) opakowuje beany
  w proxy, by dołożyć `@Transactional`, `@Cacheable`, `@Async` bez zmiany kodu — stąd np. samowywołanie
  metody wewnątrz beana **omija** proxy. Pod spodem: **JDK dynamic proxy** (dla interfejsów,
  `java.lang.reflect.Proxy`) lub CGLIB (dla klas). Hibernate używa proxy do lazy loading encji.
- **Composite** — problem: traktuj **pojedynczy element i grupę jednolicie** (drzewo). JDK: Swing
  `Component`/`Container`; struktury DOM.
- **Bridge** — problem: oddziel **abstrakcję** od **implementacji**, by zmieniać je niezależnie
  (zamiast eksplozji podklas). JDK: JDBC — `DriverManager`/`Connection` (abstrakcja) vs sterowniki
  konkretnych baz (implementacja).
- **Flyweight** — problem: wiele identycznych, niemutowalnych obiektów zjada pamięć → **współdziel**
  instancje. JDK: **Integer cache** — `Integer.valueOf(int)` cache'uje `-128..127`, więc
  `Integer.valueOf(100) == Integer.valueOf(100)` jest `true`, a `200 == 200` (przez `valueOf`) już nie
  ([[wiedza/01-jezyk/typy-i-pamiec]]). Podobnie `String` intern pool.

### BEHAWIORALNE

- **Strategy** — problem: masz **wymienne algorytmy** i chcesz je podmieniać w runtime. W nowoczesnej
  Javie strategia to zwykle **lambda / interfejs funkcyjny**, nie osobna klasa. Kanoniczny przykład:
  `Comparator` przekazany do `list.sort(...)` — każda lambda to inna strategia sortowania.
  (Przykład kodu niżej.)
- **Observer** — problem: obiekty mają **reagować na zdarzenia** innego obiektu bez ścisłego wiązania.
  JDK: listenery (Swing `ActionListener`), `PropertyChangeListener`, `Flow` (reactive streams).
  Spring: **Application Events** (`ApplicationEventPublisher` + `@EventListener`) — luźne wiązanie
  komponentów w aplikacji.
- **Template Method** — problem: szkielet algorytmu jest stały, ale niektóre kroki mają być
  nadpisywalne → klasa bazowa definiuje **metodę szkieletową** wołającą **kroki/haki** (hook methods)
  z podklas. JDK: `AbstractList`/`AbstractMap` (implementujesz kilka metod, resztę dostajesz gotową);
  Spring: `JdbcTemplate`, `RestTemplate` (szkielet + Twój callback).
- **Iterator** — problem: przechodź po kolekcji **bez ujawniania jej struktury**. JDK: `Iterator`/
  `Iterable` (pętla for-each to cukier na `Iterator`), `ListIterator`.
- **Command** — problem: **opakuj żądanie w obiekt** (kolejkowanie, undo, log). JDK: `Runnable`/
  `Callable` przekazywane do `ExecutorService`; `Runnable` = command.
- **State** — problem: obiekt zmienia zachowanie zależnie od **stanu wewnętrznego** → wydziel każdy
  stan do osobnej klasy zamiast wielkiego `switch`. Często modelowane przez `enum` ze zdefiniowanym
  zachowaniem lub sealed hierarchię.
- **Chain of Responsibility** — problem: żądanie ma przejść przez **łańcuch handlerów**, z których
  każdy albo obsługuje, albo przekazuje dalej. JDK/Jakarta: **filtry Servlet** (`FilterChain`).
  Spring Security to łańcuch filtrów bezpieczeństwa ([[wiedza/08-bezpieczenstwo/spring-security]]) —
  każdy filtr (uwierzytelnianie, autoryzacja, CSRF) obsługuje swój aspekt i woła `chain.doFilter(...)`.
- **Visitor** — problem: dodawaj **nowe operacje** na hierarchii obiektów bez modyfikowania ich klas
  (double dispatch). Klasycznie żmudny. W nowoczesnej Javie **pattern matching** na `sealed` typach
  ([[wiedza/01-jezyk/pattern-matching]]) w dużej mierze go **zastępuje** — `switch` z wzorcami po
  rekordach daje ten sam efekt (nowa operacja = nowy `switch`) znacznie zwięźlej.

## 3. Dlaczego / kiedy — nie na siłę, nowoczesna Java

- **Wzorzec to narzędzie, nie cel.** Nadużywanie ("wszystko przez fabrykę i strategię") tworzy
  **antywzorce** ([[wiedza/12-wzorce/antywzorce]]): przeinżynierowanie (over-engineering),
  niepotrzebne warstwy abstrakcji, kod trudniejszy niż problem, który rozwiązuje. Wzorzec wprowadzasz,
  gdy **czujesz ból** (zmienność, duplikacja, sztywność), nie prewencyjnie.
- **Nowoczesna Java zmienia realizację:**
  - **Lambdy / interfejsy funkcyjne** → Strategy, Command, Template Method (callback), Observer bywają
    dziś jedną linijką, bez osobnych klas.
  - **Rekordy + copy constructor** → często zastępują Builder i Prototype dla prostych danych.
  - **Sealed + pattern matching** → zastępują Visitor i wiele hierarchii z `instanceof`.
  - **Enum** → najlepszy Singleton; naturalny nośnik State.
  - **DI kontenera (Spring)** → przejmuje rolę wielu fabryk i singletonów (i robi to lepiej —
    testowalnie).
- **Rozpoznawanie > implementowanie.** W praktyce rzadko piszesz "goły" wzorzec — częściej **korzystasz**
  z niego przez JDK/Spring i musisz to **nazwać** (rozmowa rekrutacyjna, code review, dyskusja o designie).

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Budujesz endpoint zamówień. `OrderService` (`@Service`) to **Facade** nad repozytoriami i płatnościami.
Wewnątrz Spring owija go w **Proxy** dla `@Transactional`. Cenę liczysz **Strategy** (lambda per typ
klienta). Odpowiedź budujesz **Builderem**. Wysyłasz `OrderPlacedEvent` — **Observer** przez Spring Events.
Request przechodzi wcześniej przez **Chain of Responsibility** filtrów Spring Security. Pięć wzorców
w jednym flow, których nie napisałeś ręcznie.

```java
// BUILDER — czytelne tworzenie niemutowalnego obiektu z wieloma polami
HttpRequest req = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/orders"))
        .header("Content-Type", "application/json")
        .timeout(Duration.ofSeconds(5))
        .POST(HttpRequest.BodyPublishers.ofString(body))
        .build();

// STRATEGY przez lambdę — Comparator to wymienny algorytm sortowania
orders.sort(Comparator.comparing(Order::createdAt).reversed());
orders.sort((a, b) -> Double.compare(a.total(), b.total())); // inna strategia

// DECORATOR — BufferedReader opakowuje Reader, dokładając buforowanie
try (BufferedReader in = new BufferedReader(new FileReader("orders.csv"))) {
    String line = in.readLine(); // zachowanie dodane dynamicznie przez opakowanie
}
```

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, czym jest wzorzec (rozwiązanie + wspólny język) i wymienię 3 kategorie GoF.
- [ ] Wskażę wzorzec za `BufferedReader`, `HttpRequest.newBuilder()`, `Comparator`, `@Transactional`.
- [ ] Odróżnię *singleton bean* Springa od wzorca Singleton GoF.
- [ ] Wiem, że lambdy/rekordy/sealed zmieniają realizację Strategy/Builder/Visitor.
- [ ] Rozumiem, że wzorzec to narzędzie — i kiedy jego nadużycie staje się antywzorcem.

### 🔲 Black-box check
- [ ] Jak działa Decorator w `java.io` i czemu można je zagnieżdżać dowolnie?
- [ ] Dlaczego samowywołanie metody `@Transactional` wewnątrz beana nie działa? (proxy)
- [ ] Czemu `Integer.valueOf(127) == Integer.valueOf(127)`, a dla 200 nie? (Flyweight / cache)
- [ ] Jak pattern matching na sealed zastępuje Visitor?
- [ ] Czym Simple Factory różni się od Factory Method GoF?

### 🎤 Pytania rekrutacyjne
- [ ] "Podaj przykłady wzorców GoF w samym JDK." (Builder, Decorator, Iterator, Factory, Flyweight...)
- [ ] "Czy singleton bean Springa to wzorzec Singleton?" (nie — to scope kontenera)
- [ ] "Jak Spring realizuje `@Transactional`?" (Proxy + AOP)
- [ ] "Kiedy NIE stosować wzorca?" (na siłę / prewencyjnie → over-engineering)
- [ ] "Jak nowoczesna Java zmienia klasyczne wzorce?" (lambdy, rekordy, sealed)

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/12-wzorce.csv`. Format: `Pytanie;Odpowiedź`.*

```
Czym jest wzorzec projektowy?;Sprawdzone, nazwane rozwiązanie powtarzalnego problemu projektowego + wspólny język zespołu (nie gotowy kod).
Trzy kategorie wzorców GoF?;Kreacyjne (tworzenie), strukturalne (składanie), behawioralne (współpraca/algorytmy).
Co oznacza "GoF"?;Gang of Four — czterej autorzy książki Design Patterns (1994) z katalogiem 23 wzorców.
Najbezpieczniejszy Singleton w Javie?;Enum singleton — JVM gwarantuje jednokrotną inicjalizację i odporność na serializację/refleksję.
Czym różni się singleton bean Springa od wzorca Singleton?;Bean singleton to scope kontenera (jedna instancja na kontekst, testowalna przez DI), nie globalny statyczny singleton GoF.
Przykład Factory w JDK?;Integer.valueOf(int), List.of(...), Executors.newFixedThreadPool(...), Calendar.getInstance().
Przykład Abstract Factory w JDK?;DocumentBuilderFactory / TransformerFactory (rodziny parserów XML).
Problem rozwiązywany przez Builder?;Klasa z wieloma (opcjonalnymi) polami — zastępuje teleskopowe konstruktory czytelnym, fluent tworzeniem.
Przykłady Buildera w JDK?;StringBuilder, Stream.Builder, HttpRequest.newBuilder(), Locale.Builder; w Lomboku @Builder.
Co to Prototype i jak dziś w Javie?;Tworzenie obiektu przez klonowanie istniejącego; dziś preferuj copy constructor / record zamiast Cloneable.clone().
Co robi Adapter?;Tłumaczy interfejs jednego obiektu na interfejs oczekiwany przez klienta (np. Arrays.asList, InputStreamReader).
Podręcznikowy przykład Decoratora w JDK?;BufferedReader opakowujący Reader — dynamicznie dodaje buforowanie bez dziedziczenia.
Rola @Service jako wzorca?;Facade — uproszczony interfejs koordynujący złożony podsystem (repozytoria, płatności, zdarzenia).
Jak Spring realizuje @Transactional/@Cacheable?;Przez Proxy (JDK dynamic proxy dla interfejsów lub CGLIB dla klas) w Spring AOP.
Czemu samowywołanie metody @Transactional nie działa?;Wywołanie wewnętrzne omija proxy — adnotacja działa tylko na wywołaniach przez proxy.
Co ilustruje Integer cache?;Flyweight — Integer.valueOf cache'uje -128..127, więc te instancje są współdzielone (==).
Strategy w nowoczesnej Javie?;Wymienny algorytm jako lambda/interfejs funkcyjny; kanoniczny przykład to Comparator.
Observer w Springu?;Application Events — ApplicationEventPublisher + @EventListener dla luźnego wiązania komponentów.
Co to Template Method i przykład?;Szkielet algorytmu z nadpisywalnymi krokami/hakami; JDK AbstractList, Spring JdbcTemplate.
Który wzorzec kryje for-each po kolekcji?;Iterator — Iterable/Iterator; for-each to cukier składniowy na Iterator.
Gdzie w Javie widać Chain of Responsibility?;Filtry Servlet (FilterChain) i łańcuch filtrów Spring Security.
Czym pattern matching zastępuje Visitor?;switch z wzorcami po sealed/record — nowa operacja = nowy switch, bez double dispatch.
Runnable/Callable jako wzorzec?;Command — żądanie opakowane w obiekt, kolejkowane w ExecutorService.
Główna zasada stosowania wzorców?;Narzędzie, nie cel — stosuj gdy czujesz ból (zmienność/duplikacja), nie prewencyjnie (inaczej over-engineering).
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/12-wzorce/antywzorce]] · [[wiedza/05-spring/proxy-aop]] · [[wiedza/09-architektura/warstwy-hexagonal]] · [[wiedza/01-jezyk/pattern-matching]] · [[wiedza/08-bezpieczenstwo/spring-security]] · [[wiedza/01-jezyk/typy-i-pamiec]]
