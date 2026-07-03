---
temat: "AssertJ — płynne asercje"
faza: 4
status: nieopanowany
priorytet: 🟡
tags: [java, narzedzia, testy]
powiazane: ["[[wiedza/04-narzedzia/junit-mockito]]"]
---

# AssertJ — płynne (fluent) asercje

> **TL;DR:** AssertJ to biblioteka **fluent assertions** — jeden punkt wejścia `assertThat(actual)`
> zwraca *typowany* obiekt asercji, na którym łańcuchujesz metody (`.isEqualTo`, `.contains`…).
> Wygrywa z JUnit i Hamcrest **czytelnością**, **autouzupełnianiem IDE** (metody zależne od typu) i
> **bogatymi komunikatami błędów**. Java 21, AssertJ 3.

## 1. Co — definicja i API

Statyczny punkt startu i jeden import na wszystko:

```java
import static org.assertj.core.api.Assertions.*;   // assertThat, assertThatThrownBy, catchThrowable...

assertThat(actual).isEqualTo(expected);            // wzorzec: assertThat(...).xxx().yyy()
```

`assertThat(x)` jest **przeciążony** — dla `String` zwraca `StringAssert`, dla `List` `ListAssert`,
dla `Integer` `IntegerAssert` itd. Dlatego IDE po kropce podpowiada **tylko metody sensowne dla tego typu**
(np. `.startsWith` dla String, `.containsExactly` dla kolekcji). To fundamentalna przewaga nad JUnit-owym
statycznym `assertEquals(exp, act)`, gdzie kolejność argumentów łatwo pomylić, a autouzupełnianie nie pomaga.

**Wartości / obiekty:**
```java
assertThat(user).isNotNull().isInstanceOf(Admin.class);
assertThat(name).isEqualTo("Ada");
// porównanie POLE-PO-POLU zamiast equals() — ratunek dla DTO/encji bez equals:
assertThat(actualUser).usingRecursiveComparison().isEqualTo(expectedUser);
assertThat(actualUser).usingRecursiveComparison().ignoringFields("id", "createdAt").isEqualTo(expected);
```

**Kolekcje / tablice:**
```java
assertThat(list).hasSize(3)
                .contains("a", "b")                    // zawiera te elementy (w dowolnej ilości/kolejności)
                .containsExactly("a", "b", "c")         // dokładnie te, w tej kolejności
                .containsExactlyInAnyOrder("c", "a", "b"); // dokładnie te, kolejność bez znaczenia
assertThat(users).extracting("name").contains("Ada", "Bob");         // rzutowanie na pole
assertThat(users).extracting(User::getName, User::getAge)            // typowane, wiele pól -> tuple
                 .contains(tuple("Ada", 30));
assertThat(users).filteredOn(u -> u.age() > 18).hasSize(2);
assertThat(scores).allMatch(s -> s >= 0);
```

**Wyjątki:**
```java
assertThatThrownBy(() -> service.findById(99L))
        .isInstanceOf(NotFoundException.class)
        .hasMessage("User 99 not found")
        .hasMessageContaining("not found")
        .hasCauseInstanceOf(SQLException.class);

assertThatExceptionOfType(IllegalArgumentException.class)
        .isThrownBy(() -> validate(null))
        .withMessage("arg required");

// brak wyjątku:
assertThatNoException().isThrownBy(() -> service.ping());
```

**String / liczby / Optional / daty:**
```java
assertThat("hello world").startsWith("hello").containsIgnoringCase("WORLD").hasSize(11);
assertThat(3.14).isCloseTo(3.0, within(0.2));      // tolerancja dla double
assertThat(age).isPositive().isBetween(18, 65);
assertThat(optUser).isPresent().get().extracting(User::getName).isEqualTo("Ada");
assertThat(optEmpty).isEmpty();
assertThat(date).isBefore(LocalDate.now()).isAfter(LocalDate.of(2000, 1, 1));
```

## 2. Jak — co dzieje się pod spodem

- **Mechanika fluent:** każda metoda asercji zwraca `this` (lub nowy, węższy obiekt asercji przy
  `extracting`/`filteredOn`), więc łańcuch to zwykłe wywołania na obiekcie. Wynik `assertThat(x)` to
  konkretny podtyp `AbstractAssert`, wybrany **statycznie w czasie kompilacji** przez przeciążenie —
  stąd typowane podpowiedzi IDE. Gdy asercja jest fałszywa, rzucany jest `AssertionError` z opisem
  **oczekiwane vs otrzymane** (AssertJ formatuje diff, np. dla kolekcji pokazuje brakujące/nadmiarowe).
- **`as()` / opis:** `assertThat(x).as("id użytkownika %s", id).isEqualTo(1L)` dokleja czytelny opis
  do komunikatu błędu — kluczowe w pętlach i przy wielu asercjach. `as()` MUSI stać **przed** właściwą
  asercją (opis jest zapamiętany, użyty przy porażce).
- **`usingRecursiveComparison`** chodzi po grafie obiektu **refleksją**, porównując pola rekurencyjnie
  (a nie przez `equals()`), z kontrolą: `ignoringFields`, `comparingOnlyFields`, `withEqualsForType`,
  `ignoringActualNullFields`. Rozwiązuje problem „encja nie ma sensownego `equals`/`hashCode`".
- **SOFT ASSERTIONS** — domyślnie asercje są **fail-fast**: pierwszy błąd rzuca `AssertionError` i
  reszta się nie wykonuje. Soft assertions **zbierają wszystkie błędy** i zgłaszają je razem na końcu:
  ```java
  SoftAssertions softly = new SoftAssertions();
  softly.assertThat(user.name()).isEqualTo("Ada");
  softly.assertThat(user.age()).isEqualTo(30);
  softly.assertAll();                       // TU raportuje WSZYSTKIE niespełnione naraz

  // wygodniej, z auto-wywołaniem assertAll():
  assertSoftly(s -> {
      s.assertThat(user.name()).isEqualTo("Ada");
      s.assertThat(user.age()).isEqualTo(30);
  });
  ```
  W JUnit 5 istnieje też `@ExtendWith`/`JUnitJupiterSoftAssertions` (wstrzyknięcie i auto-`assertAll`).
- **Asercje niestandardowe:** dziedziczysz po `AbstractAssert<SELF, ACTUAL>`, dodajesz metody
  domenowe (`isActive()`, `hasRole(...)`) i dajesz statyczny entry-point. Można wygenerować je
  generatorem (`assertj-assertions-generator`). Efekt: `assertThat(user).isAdmin().hasEmail(...)`.

## 3. Dlaczego / kiedy — kompromisy i wąskie gardła

- **vs JUnit `assertEquals`:** JUnit wymusza `(expected, actual)` (łatwo odwrócić → mylący komunikat),
  a jego API jest ubogie (`assertTrue(list.contains(x))` gubi kontekst przy porażce). AssertJ czyta się
  jak zdanie i daje precyzyjny diff.
- **vs Hamcrest (`assertThat(x, is(...))`):** Hamcrest matchery są **nietypowane** (`Matcher<T>`),
  autouzupełnianie prawie nie pomaga, a złożone warunki są rozwlekłe. AssertJ = jeden łańcuch, pełne
  IDE-wsparcie, lepsze komunikaty. Migracja zwykle jednokierunkowa: Hamcrest → AssertJ.
- **Pułapki:** `usingRecursiveComparison` po refleksji jest **wolniejszy** i może zaskoczyć przy
  cyklach/lazy-proxy (Hibernacja!) — świadomie ignoruj pola techniczne. `containsExactly` jest wrażliwy
  na kolejność — dla zbiorów/`Map` używaj `...InAnyOrder`. Nie mieszaj `soft.assertThat` z twardym
  `assertThat` w jednym bloku (twarde nadal przerwie test).
- **Kiedy soft:** walidacja **wielu niezależnych pól** jednego wyniku (odpowiedź API, zmapowane DTO) —
  chcesz zobaczyć naraz wszystkie rozjazdy, nie łatać po jednym.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Test warstwy serwisowej Spring Boot: **Mockito** stubuje repozytorium, **AssertJ** weryfikuje wynik.

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock UserRepository repo;
    @InjectMocks UserService service;

    @Test
    void returnsMappedDto() {
        given(repo.findById(1L)).willReturn(Optional.of(new User(1L, "Ada", 30)));

        UserDto dto = service.get(1L);

        assertThat(dto)
            .usingRecursiveComparison()
            .ignoringFields("fetchedAt")
            .isEqualTo(new UserDto(1L, "Ada", 30));
    }

    @Test
    void throwsWhenMissing() {
        given(repo.findById(99L)).willReturn(Optional.empty());
        assertThatThrownBy(() -> service.get(99L))
            .isInstanceOf(NotFoundException.class)
            .hasMessageContaining("99");
    }
}
```

AssertJ współgra z JUnit (rzuca standardowy `AssertionError`, brak własnego runnera) i z Mockito
(uzupełniają się: Mockito ustawia zachowanie zależności, AssertJ sprawdza rezultat).

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, czemu `assertThat(...)` daje lepsze IDE-podpowiedzi niż JUnit/Hamcrest (typowany zwrot z przeciążenia).
- [ ] Napiszę z głowy asercje kolekcji: `contains`, `containsExactly`, `containsExactlyInAnyOrder`, `extracting`.
- [ ] Poprawnie przetestuję wyjątek trzema stylami (`assertThatThrownBy`, `assertThatExceptionOfType...isThrownBy`).
- [ ] Wiem, kiedy i po co użyć `usingRecursiveComparison` oraz jakie ma pułapki.
- [ ] Rozumiem różnicę fail-fast vs soft assertions i użyję `assertSoftly`.

### 🔲 Black-box check
- [ ] Dlaczego łańcuch `.isNotNull().isEqualTo(...)` w ogóle działa? (każda asercja zwraca `this`/węższy assert)
- [ ] Skąd IDE wie, że dla `String` jest `.startsWith`, a dla `List` `.containsExactly`? (przeciążenie `assertThat` → typowany podtyp `AbstractAssert`)
- [ ] Co robi `usingRecursiveComparison` inaczej niż `isEqualTo`? (porównanie pól refleksją vs `equals()`)
- [ ] Jak działa `SoftAssertions.assertAll()` i po co? (zbiera błędy, raportuje razem — nie fail-fast)
- [ ] Do czego służy `as()` i gdzie musi stać w łańcuchu? (opis błędu, przed asercją)

### 🎤 Pytania rekrutacyjne
- [ ] „Czemu AssertJ, a nie asercje JUnit?" (czytelność, IDE, komunikaty, kolejność argumentów)
- [ ] „Jak przetestujesz, że metoda rzuca wyjątek z konkretnym message i cause?"
- [ ] „Masz DTO bez `equals` — jak porównasz je w teście?" (`usingRecursiveComparison`)
- [ ] „Czym są soft assertions i kiedy je stosujesz?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Czym jest AssertJ?;Biblioteką fluent (płynnych) asercji — assertThat(actual) zwraca typowany obiekt asercji do łańcuchowania metod.
Główny statyczny import AssertJ?;org.assertj.core.api.Assertions.* (assertThat, assertThatThrownBy, catchThrowable, tuple...).
Dlaczego AssertJ ma lepsze autouzupełnianie IDE niż JUnit/Hamcrest?;assertThat jest przeciążony i zwraca podtyp AbstractAssert zależny od typu argumentu, więc IDE podpowiada tylko metody sensowne dla tego typu.
Jak działa łańcuch asercji w AssertJ?;Każda metoda asercji zwraca this (lub węższy obiekt asercji), więc kolejne wywołania łańcuchują się na tym samym obiekcie.
Do czego służy usingRecursiveComparison()?;Do porównania obiektów pole-po-polu refleksją zamiast przez equals() — np. dla DTO/encji bez equals.
Jak w usingRecursiveComparison pominąć pole?;.ignoringFields("id","createdAt") (lub comparingOnlyFields, ignoringActualNullFields).
Różnica contains vs containsExactly vs containsExactlyInAnyOrder?;contains = zawiera podane; containsExactly = dokładnie te i w tej kolejności; containsExactlyInAnyOrder = dokładnie te, kolejność bez znaczenia.
Co robi extracting w asercji kolekcji?;Mapuje elementy na wybrane pola/wyrażenia (np. User::getName) i asercję wykonuje na wyniku; wiele pól -> tuple(...).
Co robi filteredOn?;Filtruje elementy kolekcji predykatem/warunkiem przed dalszą asercją (np. .filteredOn(u -> u.age()>18).hasSize(2)).
Jak przetestować wyjątek w AssertJ?;assertThatThrownBy(() -> code).isInstanceOf(X.class).hasMessage("...").hasCauseInstanceOf(Y.class); lub assertThatExceptionOfType(X.class).isThrownBy(() -> code).
Co to soft assertions?;Zbierają wszystkie niespełnione asercje zamiast fail-fast; SoftAssertions.assertAll() (lub assertSoftly(...)) raportuje wszystkie błędy naraz.
Domyślne zachowanie asercji AssertJ?;Fail-fast — pierwsza fałszywa asercja rzuca AssertionError i reszta się nie wykonuje.
Do czego służy as() w AssertJ?;Dodaje czytelny opis do komunikatu błędu (as("id %s", id)); musi stać przed właściwą asercją.
Jak porównać double z tolerancją?;assertThat(x).isCloseTo(expected, within(delta)).
Jak sprawdzić Optional w AssertJ?;assertThat(opt).isPresent().get().extracting(...).isEqualTo(...); lub .isEmpty() / .contains(value).
Jak stworzyć własną asercję AssertJ?;Dziedzicząc po AbstractAssert<SELF, ACTUAL>, dodając metody domenowe i statyczny entry-point (lub generatorem assertj-assertions-generator).
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/04-narzedzia/junit-mockito]]
