---
temat: "Wyjątki (exceptions) w Javie"
faza: 1
status: nieopanowany
priorytet: 🔴
tags: [java, jezyk, wyjatki, error-handling]
powiazane: ["[[wiedza/02-jvm/model-pamieci]]", "[[wiedza/02-jvm/classloading]]", "[[wiedza/05-spring/transakcje]]", "[[wiedza/12-wzorce/refaktoryzacja]]"]
---

# Wyjątki (exceptions) w Javie

> **TL;DR:** Wyjątek to **obiekt** (`Throwable`) reprezentujący anomalię, który *odwija* (unwind) stos wątku aż do pasującego `catch`.
> **Checked** (`Exception`, nie-`RuntimeException`) — kompilator wymusza obsługę; **unchecked** (`RuntimeException`, `Error`) — nie.
> Używaj **try-with-resources** do zasobów, **multi-catch** do skrótu, **łańcucha przyczyn** by nie tracić kontekstu.
> Nigdy nie *połykaj* wyjątków i nie używaj ich do sterowania przepływem.

## 1. Co — hierarchia i API

Cała hierarchia ma jeden korzeń: **`Throwable`**. Tylko obiekt `Throwable` (lub podklasy) można `throw` i złapać w `catch`.

```
                       Throwable  ──── (implements Serializable)
                      /         \
                 Error           Exception ◄── "checked" domyślnie
              (unchecked)        /        \
         OutOfMemoryError       │          RuntimeException ◄── "unchecked"
         StackOverflowError     │          /      |        \
         NoClassDefFoundError   │   NullPointer  Illegal   ClassCast
                                │   Exception    Argument  IndexOutOfBounds
                        IOException, SQLException,
                        InterruptedException ... (checked)
```

- **`Error`** — błędy **nieodwracalne**, sygnalizowane przez JVM/runtime; aplikacja zwykle nie powinna ich łapać.
  - `OutOfMemoryError` — wyczerpana sterta/metaspace (powiązane: [[wiedza/02-jvm/model-pamieci]]).
  - `StackOverflowError` — przepełnienie stosu wątku (np. nieskończona rekurencja).
  - `NoClassDefFoundError` — klasa była przy kompilacji, brak jej w runtime ([[wiedza/02-jvm/classloading]]).
- **`Exception` (checked)** — warunki *spodziewane*, na które kod **musi** zareagować: `IOException`, `SQLException`,
  `InterruptedException`. Kompilator wymusza `try/catch` albo `throws` w sygnaturze.
- **`RuntimeException` (unchecked)** — **błędy programisty** / naruszenia kontraktu, których nie da się sensownie
  obsłużyć lokalnie: `NullPointerException` (NPE), `IllegalArgumentException`, `IllegalStateException`,
  `ClassCastException`, `IndexOutOfBoundsException`.

**Reguła kompilatora (Catch-or-Specify):** dla **checked** musisz albo złapać, albo zadeklarować `throws`.
Dla **unchecked** kompilator milczy.

```java
// checked: kompilator NIE pozwoli zignorować
void read(Path p) throws IOException {        // 'throws' = "przekazuję obsługę wyżej"
    Files.readString(p);
}

// unchecked: zero ceremonii, ale i zero gwarancji obsługi
int parse(String s) {
    return Integer.parseInt(s);               // może rzucić NumberFormatException (RuntimeException)
}
```

## 2. Jak — try-with-resources, suppressed, stack trace pod spodem

### try / catch / finally — kolejność wykonania
1. Wykonuje się `try`.
2. Jeśli wyjątek — JVM szuka **pierwszego** pasującego `catch` (od góry; podtyp musi być **przed** nadtypem,
   inaczej błąd kompilacji *"exception has already been caught"*).
3. `finally` wykonuje się **prawie zawsze** — po normalnym wyjściu *i* po wyjątku, *nawet po `return` w `try`*.

```java
try {
    return 1;                 // wartość obliczona, ale...
} finally {
    System.out.println("zawsze"); // ...to wykona się PRZED faktycznym powrotem
}
```

**Pułapki `finally`:**
- **`return` w `finally` nadpisuje** wszystko — także wyjątek lecący z `try`/`catch` (zostaje *połknięty*!):
  ```java
  try { throw new RuntimeException("X"); }
  finally { return 42; }     // ANTYWZORZEC: wyjątek znika, metoda zwraca 42
  ```
- **`finally` NIE wykona się** tylko gdy: `System.exit()` (kończy JVM), zabicie procesu, albo `Error` typu
  awaria JVM / nieskończona pętla w `try`. `System.exit()` w `try` → `finally` pominięte.

### try-with-resources (TWR) — Java 7+
Każdy zasób implementujący **`AutoCloseable`** (lub `Closeable`) jest **automatycznie** zamykany — kompilator
generuje `finally` z wywołaniem `close()`. Eliminuje klasyczny błąd „zapomniałem zamknąć w finally".

```java
try (var in = Files.newInputStream(src);
     var out = Files.newOutputStream(dst)) {   // dwa zasoby
    in.transferTo(out);
}   // close() wywołane automatycznie
```

- **Kolejność zamykania jest ODWROTNA** do otwierania (LIFO): najpierw `out.close()`, potem `in.close()`.
- **Suppressed exceptions:** jeśli `try` rzuci wyjątek **i** `close()` też rzuci, to wyjątek z `try` jest
  *główny*, a wyjątek z `close()` zostaje **dołączony** przez `Throwable.addSuppressed(...)` (zamiast go
  *maskować*, jak robił ręczny `finally`). Odczyt: `getSuppressed()`. W stack trace widać go jako `Suppressed:`.
  ```java
  try (AutoCloseable r = () -> { throw new IllegalStateException("close fail"); }) {
      throw new RuntimeException("body fail");
  } catch (Exception e) {
      e.getMessage();                 // "body fail"  (główny)
      e.getSuppressed()[0];           // IllegalStateException "close fail"
  }
  ```
- Java 9+: w TWR można użyć **effectively final** zmiennej zadeklarowanej wcześniej: `try (resource) { ... }`.

### multi-catch — Java 7+
Jeden `catch` dla kilku niepowiązanych typów (`|`), gdy obsługa jest identyczna:
```java
try { ... }
catch (IOException | SQLException e) {   // typy NIE mogą być w relacji dziedziczenia
    log.error("I/O lub DB", e);
}
```
Zmienna `e` jest tu **niejawnie `final`**; jej typ statyczny to najbliższy wspólny nadtyp.

### rethrow i precise rethrow — Java 7+
*Precise rethrow:* kompilator analizuje, jakie **konkretnie** typy mogą wpaść do `catch (Exception e)`,
więc możesz przerzucić wąsko, mimo łapania szeroko:
```java
void m() throws IOException, SQLException {     // dokładne typy, nie 'throws Exception'
    try { riskyIO(); riskyDb(); }
    catch (Exception e) {                       // łapię szeroko...
        log.warn("retry");
        throw e;                                // ...ale kompilator wie: tu lecą tylko IOException/SQLException
    }
}
```

### Łańcuch przyczyn (chaining) — „Caused by"
Owijając niskopoziomowy wyjątek w domenowy, **zachowaj oryginał** jako *cause*:
```java
try { jdbc.query(...); }
catch (SQLException e) {
    throw new RepositoryException("Nie udało się pobrać usera", e);  // konstruktor (msg, cause)
}
```
- `getCause()` zwraca przyczynę; w stack trace pojawia się sekcja **`Caused by:`**.
- `initCause(cause)` — gdy klasa wyjątku nie ma konstruktora z `cause` (można wywołać **raz**).
- **Gubienie przyczyny** (`throw new X(e.getMessage())` zamiast `new X(msg, e)`) to częsty antywzorzec — tracisz
  oryginalny stack trace i kontekst.

### Stack trace pod spodem — koszt wydajności
- Konstruktor `Throwable` woła **`fillInStackTrace()`**, które *przechodzi cały stos wątku* i zapisuje ramki.
  To **najdroższa** część tworzenia wyjątku — często droższa niż samo `throw`/`catch`.
- Dlatego wyjątki do **sterowania przepływem** są kosztowne (patrz antywzorce).
- Optymalizacja dla wyjątków-sygnałów (nie diagnostycznych): nadpisz `fillInStackTrace()` by zwracał `this`,
  albo użyj konstruktora `Throwable(msg, cause, enableSuppression, writableStackTrace=false)` — pomija zbieranie ramek.
  ```java
  // wyjątek-sygnał bez kosztownego stack trace
  class FlowSignal extends RuntimeException {
      FlowSignal() { super(null, null, false, false); }
  }
  ```
- HotSpot potrafi też po wielokrotnym rzuceniu *tego samego* wyjątku z gorącej ścieżki **wyzerować** stack trace
  (opcja `-XX:-OmitStackTraceInFastThrow` przywraca pełny) — stąd zaskakujące „puste" NPE w logach produkcyjnych.

## 3. Dlaczego / kiedy — antywzorce i debata checked vs unchecked

### Własne wyjątki — checked czy unchecked?
- **Unchecked (extends `RuntimeException`)** — gdy wywołujący **i tak nie może sensownie zareagować** (błąd
  konfiguracji, naruszenie niezmiennika, błąd domenowy propagowany do globalnego handlera). **Współczesny trend.**
- **Checked (extends `Exception`)** — gdy istnieje **realna, lokalna** akcja naprawcza, którą chcesz *wymusić*
  na wywołującym (np. retry, fallback). W praktyce stosowane coraz rzadziej.

### Antywzorce (czerwone flagi na review)
1. **Połykanie (swallowing)** — pusty `catch` lub `catch { /* ignore */ }`. Błąd znika bez śladu.
   ```java
   try { ... } catch (Exception e) { }   // NIGDY — przynajmniej zaloguj z 'e'
   ```
2. **Za szeroki `catch (Exception | Throwable)`** — łapie też `RuntimeException`/`Error`, których nie chciałeś
   (np. `OutOfMemoryError`, `InterruptedException`). Łap **najwęższy** sensowny typ.
3. **Wyjątki do sterowania przepływem** — np. pętla łapiąca `NoSuchElementException` zamiast `hasNext()`.
   Kosztowne (`fillInStackTrace`) i nieczytelne. Wyjątki są od *wyjątkowych* sytuacji.
4. **Log-and-throw** — `log.error(e); throw e;`. Ten sam błąd loguje się wielokrotnie po drodze w górę stosu.
   Zasada: **albo loguj, albo rzucaj** — loguj **raz**, na granicy (boundary), gdzie *naprawdę* obsługujesz.
5. **Tracenie oryginalnej przyczyny** — `throw new MyEx(e.getMessage())` zamiast `new MyEx(msg, e)`.
6. **`catch` + brak rethrow + brak obsługi** — udajesz, że obsłużyłeś, a tylko zamiotłeś pod dywan.
7. **`return`/`break`/`continue` w `finally`** — niejawnie połyka wyjątki.
8. **Łapanie `InterruptedException` bez przywrócenia flagi** — zawsze `Thread.currentThread().interrupt();`
   albo propaguj (powiązane: [[wiedza/03-wspolbieznosc/podstawy]]).

### Debata checked vs unchecked — czemu frameworki (Spring) wybrały unchecked
Checked exceptions miały wymusić solidną obsługę błędów, ale w praktyce:
- **Łamią enkapsulację** — `throws SQLException` przecieka detal warstwy danych do warstw wyżej.
- **Niestabilne sygnatury** — dodanie checked do metody zmienia API wszystkich wywołujących.
- **Zachęcają do antywzorców** — zmęczeni `try/catch` programiści połykają je pustym `catch`.
- **Źle współgrają z lambdami / Stream API** — `Function`/`Supplier` nie deklarują `throws`, więc checked w
  lambdzie wymaga brzydkiego owijania.

Dlatego **Spring** owija np. `SQLException` w hierarchię **`DataAccessException` (unchecked)**, a większość
nowoczesnych bibliotek (Hibernate, Spring, Jackson) używa wyłącznie unchecked. Centralna obsługa idzie wtedy do
jednego miejsca (np. `@ControllerAdvice` / `@ExceptionHandler`). To kontekst dla [[wiedza/05-spring/transakcje]]:
domyślny rollback w Springu następuje **tylko dla `RuntimeException`/`Error`**, a dla checked trzeba `rollbackFor`.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)
W warstwie repozytorium łapiesz niskopoziomowy, *checked* wyjątek I/O i przerzucasz domenowy *unchecked*
z zachowaniem przyczyny; zasoby zamykasz przez try-with-resources, a globalny handler loguje **raz** na granicy HTTP.

```java
public List<User> loadUsers(Path csv) {
    try (var lines = Files.lines(csv)) {            // AutoCloseable Stream — TWR zamknie plik
        return lines.skip(1).map(this::parseLine).toList();
    } catch (IOException e) {
        // owijamy checked → unchecked, ZACHOWUJĄC cause; NIE logujemy tutaj (zrobi to handler wyżej)
        throw new UserImportException("Nie udało się wczytać " + csv, e);
    }
}

// granica aplikacji (Spring MVC) — JEDNO miejsce na logowanie
@ExceptionHandler(UserImportException.class)
ResponseEntity<ApiError> handle(UserImportException ex) {
    log.error("Import users failed", ex);           // log RAZ, z pełnym 'Caused by'
    return ResponseEntity.status(500).body(new ApiError(ex.getMessage()));
}
```

---

## ✅ Kryteria opanowania
- [ ] Narysuję hierarchię `Throwable → Error/Exception → RuntimeException` i przypiszę przykłady.
- [ ] Wyjaśnię różnicę checked vs unchecked i regułę kompilatora (catch-or-specify).
- [ ] Wiem, kiedy `finally` się NIE wykona i czemu `return` w `finally` jest pułapką.
- [ ] Rozumiem kolejność zamykania i suppressed exceptions w try-with-resources.
- [ ] Zawsze zachowuję `cause` i potrafię uzasadnić wybór unchecked dla własnych wyjątków.
- [ ] Rozpoznam i nazwę co najmniej 5 antywzorców obsługi wyjątków.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co dokładnie robi JVM, gdy w `try` poleci wyjątek? (unwinding stosu, szukanie handlera)
- [ ] Co generuje kompilator z `try-with-resources`? (ukryty `finally` + `close()` + `addSuppressed`)
- [ ] Dlaczego tworzenie wyjątku jest drogie? (`fillInStackTrace` przechodzi cały stos)
- [ ] Jak działa precise rethrow — skąd kompilator zna dokładne typy?
- [ ] Czemu w logach pojawiają się NPE bez stack trace? (`OmitStackTraceInFastThrow`)

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Checked vs unchecked — różnica i kiedy którego użyć?"
- [ ] „Czy `finally` zawsze się wykona? Podaj kontrprzykłady."
- [ ] „Co to suppressed exception i kiedy powstaje?"
- [ ] „Czemu Spring używa unchecked exceptions?"
- [ ] „Jak owinąć wyjątek nie tracąc oryginalnej przyczyny?"
- [ ] „Wymień antywzorce obsługi wyjątków."

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Korzeń hierarchii wyjątków w Javie?;Throwable — tylko on (i podklasy) może być throw/catch.
Error vs Exception?;Error = nieodwracalne błędy JVM (OOM, StackOverflow), zwykle nie łapać; Exception = warunki obsługiwalne przez aplikację.
Checked vs unchecked?;Checked (Exception, nie-RuntimeException) — kompilator wymusza catch lub throws; unchecked (RuntimeException, Error) — nie wymusza.
Przykłady RuntimeException (błędy programisty)?;NullPointerException, IllegalArgumentException, IllegalStateException, ClassCastException, IndexOutOfBoundsException.
Kiedy finally się NIE wykona?;Po System.exit(), zabiciu procesu, awarii JVM lub nieskończonej pętli/blokadzie w try.
Co robi return w finally?;Nadpisuje wartość i POŁYKA wyjątek lecący z try/catch — antywzorzec.
Co implementuje zasób użyty w try-with-resources?;AutoCloseable (lub Closeable) — kompilator generuje finally z close().
Kolejność zamykania zasobów w try-with-resources?;Odwrotna do otwierania (LIFO).
Czym jest suppressed exception?;Wyjątek z close(), gdy try już rzucił — dołączany przez addSuppressed(), odczyt getSuppressed(), w trace jako "Suppressed:".
Składnia multi-catch i ograniczenie?;catch (A | B e) — typy nie mogą być w relacji dziedziczenia; e jest niejawnie final.
Co to precise rethrow?;Łapiesz catch (Exception e) i robisz throw e, a kompilator wie, że lecą tylko konkretne typy zadeklarowane w throws.
Jak zachować oryginalną przyczynę wyjątku?;Konstruktor (msg, cause) lub initCause() — w stack trace pojawia się "Caused by:".
Czemu tworzenie wyjątku jest kosztowne?;Konstruktor Throwable woła fillInStackTrace(), które przechodzi cały stos wątku.
Jak stworzyć tani wyjątek-sygnał bez stack trace?;super(msg, cause, false, false) lub override fillInStackTrace() zwracający this.
Czemu Spring/Hibernate używają unchecked?;Checked łamią enkapsulację, destabilizują sygnatury, źle współgrają z lambdami i zachęcają do połykania.
Antywzorzec log-and-throw?;Logowanie i rzucanie tego samego wyjątku → wielokrotne logi; zasada: loguj ALBO rzucaj, loguj raz na granicy.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/02-jvm/model-pamieci]] · [[wiedza/02-jvm/classloading]] · [[wiedza/03-wspolbieznosc/podstawy]] · [[wiedza/05-spring/transakcje]] · [[wiedza/12-wzorce/refaktoryzacja]]
