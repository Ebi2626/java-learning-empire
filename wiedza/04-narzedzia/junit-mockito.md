---
temat: "Testy jednostkowe — JUnit 5 + Mockito"
faza: 4
status: nieopanowany
priorytet: 🔴
tags: [java, narzedzia, testy]
powiazane: ["[[wiedza/04-narzedzia/assertj]]", "[[wiedza/04-narzedzia/testcontainers]]", "[[wiedza/03-spring/spring-boot-test]]"]
---

# Testy jednostkowe — JUnit 5 (Jupiter) + Mockito

> **TL;DR:** **JUnit 5** to framework do pisania i uruchamiania testów (Platform = silnik, Jupiter = API,
> Vintage = zgodność z JUnit 4), a **Mockito** dostarcza **test doubles** (mock/stub/spy) do izolowania
> testowanej jednostki od jej zależności. Test jednostkowy = **jedna klasa** + zamockowani współpracownicy,
> szybki i deterministyczny (**FIRST**), pisany w schemacie **AAA** (Arrange-Act-Assert).

## 1. Co — definicja i API

### Architektura JUnit 5 (po co rozdział na 3 moduły)

JUnit 5 to nie jeden artefakt, tylko **trzy podprojekty** — rozdzielone celowo, żeby oddzielić *silnik
uruchamiający* od *API do pisania testów*:

```
┌──────────────────────── JUnit Platform ─────────────────────────┐
│  Fundament: uruchamia testy, TestEngine SPI, Launcher API        │
│  (to na nim stoją IDE, Maven Surefire, Gradle)                   │
│  ┌──────────────── Jupiter ────────────┐ ┌──── Vintage ───────┐  │
│  │  Nowe API (@Test, @Nested, asercje) │ │  TestEngine dla    │  │
│  │  + jupiter-engine (TestEngine)      │ │  starych testów    │  │
│  │                                     │ │  JUnit 3/4         │  │
│  └─────────────────────────────────────┘ └────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

- **JUnit Platform** — definiuje **`TestEngine` SPI** i **Launcher**. Dzięki temu na tym samym silniku
  potrafią działać *różne* frameworki testowe (Jupiter, Vintage, ale też Spock, Cucumber, ArchUnit).
  IDE i narzędzia budujące (Surefire/Gradle) integrują się **raz** z Platformą, nie z każdym frameworkiem.
- **Jupiter** = *programming model* (adnotacje, asercje) **+** `jupiter-engine` (implementacja `TestEngine`).
- **Vintage** = `TestEngine`, który uruchamia stare testy **JUnit 4/3** na nowej Platformie — migracja
  stopniowa, bez przepisywania wszystkiego naraz. Gdy nie masz legacy testów, Vintage nie jest potrzebny.

> **Sens rozdziału:** *rozszerzalność* (można dopiąć własny engine) i *stabilne API dla narzędzi*
> (Launcher). To odpowiedź na sztywność JUnit 4, gdzie logika odkrywania i uruchamiania testów była zlana.

### Cykl życia i adnotacje (Jupiter)

```java
class OrderServiceTest {
    @BeforeAll  static void initDb() {}      // raz, przed wszystkimi testami (static, bo instancja per-metoda)
    @BeforeEach void setUp() {}              // przed KAŻDYM @Test — świeży stan
    @Test       void placesOrder() {}
    @AfterEach  void cleanUp() {}            // po każdym teście
    @AfterAll   static void closeDb() {}     // raz, na końcu
}
```

- **Domyślnie JUnit tworzy NOWĄ instancję klasy testowej dla każdej metody** `@Test`
  (`Lifecycle.PER_METHOD`) — dlatego pola między testami się nie „przeciekają", a `@BeforeAll`/`@AfterAll`
  muszą być `static` (nie ma jednej wspólnej instancji). Można to zmienić na `@TestInstance(PER_CLASS)`.
- **`@DisplayName("Rzuca wyjątek, gdy magazyn pusty")`** — czytelna nazwa w raporcie (może zawierać spacje, emoji).
- **`@Disabled("flaky, JIRA-123")`** — wyłącza test (lepsze niż `@Ignore` z JUnit 4, wymaga powodu jako dokumentacji).
- **`@Nested`** — klasa wewnętrzna (niestatyczna!) grupująca testy jednego kontekstu; buduje strukturę
  „given some state → …" i pozwala mieć osobne `@BeforeEach` per kontekst.
- **`@Tag("slow")` / `@Tag("integration")`** — etykiety do filtrowania (uruchom tylko `unit`, pomiń `slow` na CI).

### Asercje, assumptions, timeout

```java
assertEquals(expected, actual);                 // uwaga na kolejność: (oczekiwane, faktyczne)
assertTrue(list.isEmpty());
assertThrows(IllegalArgumentException.class,     // sprawdza TYP wyjątku + zwraca go do dalszej inspekcji
             () -> service.process(null));
assertAll("user",                                // grupuje — RAPORTUJE WSZYSTKIE porażki naraz (nie zatrzymuje się na 1.)
    () -> assertEquals("Ala", u.name()),
    () -> assertEquals(30,    u.age()));
assertTimeout(Duration.ofMillis(100), () -> fastOp());   // fail jeśli trwa dłużej

assumeTrue("CI".equals(System.getenv("ENV")));   // assumption: jeśli fałsz → test ABORTED (nie failed) — pomijany warunkowo
```

- **Asercja vs assumption:** niespełniona **asercja** = test **failed** (błąd w kodzie); niespełniona
  **assumption** = test **aborted/skipped** (nie dotyczy tego środowiska — nie jest to porażka).

### Testy parametryzowane

```java
@ParameterizedTest
@ValueSource(ints = {1, 2, 3})            // proste literały (jeden argument)
void isPositive(int n) { assertTrue(n > 0); }

@ParameterizedTest
@CsvSource({"2,3,5", "0,0,0", "-1,1,0"})  // wiele argumentów w wierszu CSV
void adds(int a, int b, int expected) { assertEquals(expected, a + b); }

@ParameterizedTest
@EnumSource(Status.class)                 // po wszystkich wartościach enuma
void everyStatusHandled(Status s) { assertNotNull(handler.handle(s)); }

@ParameterizedTest
@MethodSource("blankStrings")             // najpotężniejsze: fabryka Stream<Arguments> (obiekty, nie tylko literały)
void rejectsBlank(String s) { assertThrows(...); }
static Stream<String> blankStrings() { return Stream.of("", " ", "\t"); }
```

## 2. Jak — Extension API, mechanika mocków pod spodem

### Model rozszerzeń (Extension API) — zastępuje Runnery i Rule z JUnit 4

W **JUnit 4** były dwa niekompatybilne mechanizmy rozszerzeń: **`@RunWith(Runner)`** (tylko **jeden** na klasę!)
oraz **`@Rule`/`@ClassRule`**. Nie dało się łączyć np. Springa i Mockito przez `@RunWith`.

**JUnit 5** ma **jeden, spójny `Extension` API**. Rozszerzenie implementuje **callbacki** (interfejsy typu
`BeforeEachCallback`, `AfterEachCallback`, `ParameterResolver`, `TestExecutionExceptionHandler`), a Jupiter
wywołuje je w odpowiednich punktach cyklu życia. Rozszerzeń można dopiąć **wiele**:

```java
@ExtendWith(MockitoExtension.class)      // <-- to zastępuje @RunWith(MockitoJUnitRunner.class) z JUnit 4
class OrderServiceTest { ... }
```

- **`MockitoExtension`** inicjalizuje pola `@Mock`/`@InjectMocks` przed każdym testem (przez `ParameterResolver`
  i callbacki) oraz — w domyślnym trybie `STRICT_STUBS` — weryfikuje po teście, czy nie ma **nieużytych stubów**
  (wykrywa martwe `when(...)`, sygnał złego testu).
- **Wstrzykiwanie parametrów** — `ParameterResolver` pozwala JUnit **podać argumenty do metod** testowych/lifecycle.
  Stąd możesz pisać `@Test void t(TestInfo info)`, `@Test void t(@Mock Repo repo)` — dostawca (Jupiter core,
  Mockito) rozstrzyga typ i wstrzykuje instancję. Ten sam mechanizm zasila testy parametryzowane.

### Mockito — mechanika pod spodem

Mockito **generuje w locie podklasę/proxy** mockowanego typu (przez ByteBuddy — subclassing; od Mockito 5
domyślnie `mockmaker-inline` używa **instrumentacji**, co pozwala mockować też `final`). Każde wywołanie
metody na mocku jest **przechwytywane**, zapisywane (do `verify`) i porównywane z zarejestrowanymi
**stubami** (`when(...)`). To dlatego mockowanie *finalnych/statycznych* metod historycznie było trudne —
klasyczny subclassing nie mógł ich nadpisać.

```java
@Mock        OrderRepository repo;        // Mockito tworzy mock (proxy)
@InjectMocks OrderService    service;     // wstrzykuje powyższe mocki (przez konstruktor/setter/pole)

when(repo.findById(1L)).thenReturn(Optional.of(order));   // stub: zwróć wartość
when(repo.save(any())).thenThrow(new DbException());       // stub: rzuć wyjątek

// doReturn().when() — potrzebne dla SPY albo metod void/rzucających:
doReturn(order).when(spyRepo).findById(1L);                // NIE wywołuje realnej metody (when(spy.x()) BY wywołało!)
```

- **`when(x).thenReturn(y)`** wywołuje `x` naprawdę, żeby złapać wywołanie — na **spy** to **odpala realny kod**
  (efekty uboczne!). Dlatego na spy używa się **`doReturn(y).when(spy).x()`**, które nie wykonuje realnej metody.
  Tak samo `doThrow`, `doNothing` (dla `void`), `doAnswer`.
- **Argument matchers** (`any()`, `eq()`, `anyLong()`, `argThat(...)`) — **REGUŁA: jeśli używasz choć jednego
  matchera w wywołaniu, WSZYSTKIE argumenty muszą być matcherami.** Literał mieszany z matcherem → `InvalidUseOfMatchersException`.
  Stąd: `verify(repo).save(eq(1L), any())` — `1L` opakowane w `eq`, nie gołe.

### verify, ArgumentCaptor, BDDMockito

```java
verify(repo).save(order);                 // dokładnie raz (times(1))
verify(repo, times(2)).save(any());
verify(repo, never()).delete(any());
verify(repo, atLeast(1)).findById(anyLong());
verifyNoInteractions(emailSender);        // mock w ogóle nie dotknięty

ArgumentCaptor<Order> captor = ArgumentCaptor.forClass(Order.class);
verify(repo).save(captor.capture());      // przechwytuje FAKTYCZNY argument
assertEquals(APPROVED, captor.getValue().status());   // asercja na tym, co przekazano

// BDDMockito — czytelniejsza, spójna z Given-When-Then:
given(repo.findById(1L)).willReturn(Optional.of(order));   // = when().thenReturn()
then(repo).should().save(order);                            // = verify()
```

## 3. Dlaczego / kiedy — dobór, antywzorce testowe

### Taksonomia test doubles (Meszaros) — kluczowe różnice

| Typ | Rola |
|---|---|
| **Dummy** | Wypełniacz — przekazywany, nigdy nieużywany (np. `null` do konstruktora). |
| **Stub** | Zwraca **zaprogramowane** odpowiedzi na wywołania (steruje *wejściem* testu). |
| **Spy** | **Prawdziwy** obiekt z częściowym nadpisaniem + **nagrywa** wywołania (można zweryfikować). |
| **Mock** | Obiekt z **oczekiwaniami** co do wywołań — weryfikuje *zachowanie* (interakcje). |
| **Fake** | Działająca, lecz uproszczona implementacja (np. `HashMap`-owe „repo", in-memory H2). |

- **Stub vs Mock:** stub → asercja na **stanie** (co zwrócił kod); mock → asercja na **interakcji**
  (czy zawołano `save`). Mockito łączy oba (obiekt Mockito bywa i stubem, i mockiem).
- **Spy** = częściowy mock realnego obiektu — używaj **oszczędnie**; często sygnał, że klasa robi za dużo.

### Piramida testów, TDD, FIRST

```
        /\        E2E / UI      — mało, wolne, kruche (cały system, np. Selenium)
       /  \       Integration   — średnio (Spring context, DB, [[wiedza/04-narzedzia/testcontainers]])
      /____\      Unit          — DUŻO, szybkie, izolowane (JUnit + Mockito)
```

- **Test jednostkowy** izoluje **jedną klasę**; współpracownicy = mocki. **Integracyjny** sprawdza
  współdziałanie modułów (realna DB, `@SpringBootTest`). **E2E** — cały przepływ z perspektywy użytkownika.
- **TDD** (Red-Green-Refactor): najpierw failujący test, potem minimalny kod, potem refaktor — test steruje projektem.
- **AAA (Arrange-Act-Assert):** przygotuj dane/mocki → wykonaj jedną akcję → sprawdź wynik. Jedna sekcja Act.
- **FIRST:** **F**ast, **I**solated (niezależne od siebie i kolejności), **R**epeatable (deterministyczne,
  bez zależności od czasu/sieci), **S**elf-validating (pass/fail, nie „patrz w log"), **T**imely (pisane blisko kodu).

### Pułapki i kiedy NIE mockować

- **Nadmiar mocków** — jeśli test to głównie `when/verify`, testujesz **implementację, nie zachowanie**.
  Refaktor produkcyjny (bez zmiany zachowania) rozwala takie testy → są **kruche i bezwartościowe**.
- **Mockowanie typów, których nie posiadasz** (biblioteki, `HttpClient`, JDBC) — mock utrwala Twoje
  *założenia* o cudzym API, które mogą być błędne. Zamiast tego owiń bibliotekę własnym adapterem i mockuj **jego**.
- **Nie mockuj value objects / prostych typów** (`String`, `List`, DTO, encje) — używaj realnych instancji.
- **Mockowanie static/final** (`mockStatic`, mockito-inline) — możliwe, ale to **code smell**: statyczne
  zależności utrudniają testowanie. Wolisz wstrzyknąć zależność niż mockować `LocalDate.now()`.
- **Nie mockuj testowanej klasy** ani czystych funkcji bez zależności — testuj je bezpośrednio.
- **Nie mockuj tego, co i tak zweryfikuje test integracyjny** (np. mapowanie ORM) — użyj [[wiedza/04-narzedzia/testcontainers]].

## Przykład w praktyce — test serwisu z zamockowanym repozytorium

Typowy `@Service` w Spring Boot: `OrderService` zależy od `OrderRepository` i `EmailSender`. W teście
**jednostkowym** obie zależności są mockowane — sprawdzamy logikę serwisu, nie DB ani SMTP.

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock        OrderRepository repo;
    @Mock        EmailSender     email;
    @InjectMocks OrderService    service;   // dostaje repo + email przez konstruktor

    @Test
    @DisplayName("Zatwierdza zamówienie i wysyła potwierdzenie, gdy jest zapas")
    void approvesAndNotifiesWhenInStock() {
        // Arrange
        var order = new Order(1L, "SKU-1", 2, Status.NEW);
        given(repo.findById(1L)).willReturn(Optional.of(order));
        given(repo.save(any(Order.class))).willAnswer(inv -> inv.getArgument(0));

        // Act
        Order result = service.approve(1L);

        // Assert — stan
        assertEquals(Status.APPROVED, result.status());

        // Assert — interakcja + zawartość argumentu
        var captor = ArgumentCaptor.forClass(Order.class);
        then(repo).should().save(captor.capture());
        assertEquals(Status.APPROVED, captor.getValue().status());
        then(email).should().send(eq("SKU-1"), any());     // wszystkie argumenty matcherami
    }

    @Test
    void throwsWhenOrderMissing() {
        given(repo.findById(99L)).willReturn(Optional.empty());
        assertThrows(OrderNotFoundException.class, () -> service.approve(99L));
        then(email).shouldHaveNoInteractions();            // mail się NIE wysłał
    }
}
```

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię rozdział JUnit 5 na Platform / Jupiter / Vintage i po co on jest.
- [ ] Rozumiem cykl życia testu i czemu `@BeforeAll` jest `static` (instancja per-metoda).
- [ ] Napiszę test parametryzowany z `@MethodSource` i wiem, czym różni się od `@CsvSource`.
- [ ] Wyjaśnię Extension API jako następcę Runnerów i Rule; wiem, co robi `MockitoExtension`.
- [ ] Rozróżnię dummy/stub/spy/mock/fake i podam, kiedy który.
- [ ] Wiem, kiedy potrzebne jest `doReturn().when()` zamiast `when().thenReturn()`.
- [ ] Znam regułę matcherów (wszystkie albo żaden) i użyję `ArgumentCaptor`.
- [ ] Wymienię antywzorce testowe i przypadki, gdy NIE mockować.

### 🔲 Black-box check
- [ ] Co dokładnie robi `TestEngine` SPI i czemu daje rozszerzalność?
- [ ] Jak Mockito technicznie tworzy mock (proxy/subclassing/instrumentacja) i czemu `final` był problemem?
- [ ] Dlaczego `when(spy.method())` jest niebezpieczne, a `doReturn().when(spy)` nie?
- [ ] Czym różni się niespełniona asercja od niespełnionej assumption (failed vs aborted)?
- [ ] Co robi `STRICT_STUBS` i czemu wykrywanie nieużytych stubów jest przydatne?

### 🎤 Pytania rekrutacyjne
- [ ] „Czym różni się stub od mocka?" (stan vs interakcja)
- [ ] „Kiedy NIE należy mockować?" (typy, których nie posiadasz; value objects; nadmiar mocków)
- [ ] „Co zastąpiło `@RunWith` i `@Rule` w JUnit 5?" (Extension API, `@ExtendWith`)
- [ ] „Jak zweryfikować argument przekazany do zależności?" (`ArgumentCaptor`)
- [ ] „Test jednostkowy vs integracyjny — gdzie granica?" (izolacja jednej klasy vs realne zależności)
- [ ] „Co oznacza FIRST / AAA?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/04-narzedzia.csv`.*
```
Z jakich 3 modułów składa się JUnit 5?;Platform (silnik + TestEngine SPI + Launcher), Jupiter (nowe API + engine), Vintage (engine dla JUnit 3/4).
Po co JUnit Platform / TestEngine SPI?;Rozszerzalność — różne frameworki (Jupiter, Spock, Cucumber) działają na jednym silniku, a narzędzia integrują się raz.
Do czego służy moduł Vintage?;Uruchamia stare testy JUnit 3/4 na Platformie JUnit 5 — stopniowa migracja bez przepisywania.
Czemu @BeforeAll musi być static?;JUnit domyślnie tworzy nową instancję klasy testowej per metoda (PER_METHOD), więc nie ma wspólnej instancji dla metody hook.
Różnica @BeforeEach vs @BeforeAll?;@BeforeEach wykonuje się przed KAŻDYM testem; @BeforeAll raz, przed wszystkimi (static).
Do czego @Nested?;Niestatyczna klasa wewnętrzna grupująca testy jednego kontekstu, z własnym @BeforeEach.
Asercja vs assumption?;Niespełniona asercja = test FAILED (błąd kodu); niespełniona assumption = test ABORTED/skipped (nie dotyczy środowiska).
Co robi assertAll?;Wykonuje wszystkie asercje w grupie i raportuje WSZYSTKIE porażki naraz, nie zatrzymując się na pierwszej.
Co robi assertThrows?;Sprawdza, że blok rzuca wyjątek danego typu, i zwraca ten wyjątek do dalszej inspekcji.
Różnica @CsvSource vs @MethodSource?;@CsvSource — literały w wierszach CSV; @MethodSource — fabryka Stream<Arguments>, pozwala przekazać dowolne obiekty.
Co zastąpiło @RunWith i @Rule z JUnit 4?;Jednolity Extension API (@ExtendWith) z callbackami — można dopiąć wiele rozszerzeń naraz.
Co robi MockitoExtension?;Inicjalizuje @Mock/@InjectMocks przed testem i w trybie STRICT_STUBS wykrywa nieużyte stuby.
Wymień 5 test doubles.;Dummy (wypełniacz), Stub (zwraca zaprogramowane wartości), Spy (realny obiekt + nagrywa), Mock (weryfikuje interakcje), Fake (uproszczona działająca implementacja).
Stub vs Mock — asercja na czym?;Stub — asercja na stanie/wyniku; Mock — asercja na interakcji (czy metodę wywołano).
Kiedy używać doReturn().when() zamiast when().thenReturn()?;Dla spy i metod void/rzucających — when(spy.x()) wywołałoby realną metodę; doReturn().when() nie wykonuje realnego kodu.
Reguła argument matchers w Mockito?;Jeśli używasz choć jednego matchera, WSZYSTKIE argumenty muszą być matcherami (literał opakuj w eq()).
Do czego ArgumentCaptor?;Przechwytuje faktyczny argument przekazany do mocka, by zrobić na nim asercję.
BDDMockito — odpowiedniki?;given().willReturn() = when().thenReturn(); then().should() = verify().
Czemu mockowanie final/static było trudne?;Klasyczny mock to podklasa/proxy przez subclassing — nie mógł nadpisać final; potrzebna instrumentacja (mockito-inline, domyślna w Mockito 5).
Co oznacza FIRST?;Fast, Isolated, Repeatable, Self-validating, Timely — cechy dobrego testu jednostkowego.
Co oznacza AAA?;Arrange (przygotuj dane/mocki), Act (jedna akcja), Assert (sprawdź wynik).
Kiedy NIE mockować?;Value objects/prostych typów, typów których nie posiadasz (owiń adapterem), oraz gdy test zamienia się w testowanie implementacji.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/04-narzedzia/assertj]] · [[wiedza/04-narzedzia/testcontainers]] · [[wiedza/03-spring/spring-boot-test]]
