---
temat: "Refaktoryzacja"
faza: 12
status: nieopanowany
priorytet: 🟡
tags: [java, wzorce, refaktoryzacja, jakosc]
powiazane: ["[[wiedza/04-narzedzia/junit-mockito]]", "[[wiedza/12-wzorce/antywzorce]]", "[[wiedza/01-jezyk/pattern-matching]]"]
---

# Refaktoryzacja

> **TL;DR:** **Refactoring** (Fowler) to zmiana **wewnętrznej** struktury kodu **bez zmiany zewnętrznego zachowania**, wykonywana w **małych, bezpiecznych krokach** pod ochroną **testów**. Bez testów to nie refaktoryzacja — to przepisywanie. Cel: czytelność, utrzymywalność, spłata **technical debt** i przygotowanie gruntu pod nową funkcję („make the change easy, then make the easy change").

## 1. Co — definicja i API

**Refaktoryzacja** = dyscyplinowana technika restrukturyzacji istniejącego kodu, która zmienia jego **strukturę wewnętrzną** (nazwy, podział na metody/klasy, przepływ), **nie zmieniając obserwowalnego zachowania** (kontrakt, wynik, efekty uboczne pozostają te same). Kluczowe słowa Fowlera: **małe kroki** (*small steps*) i **zachowanie zachowania** (*behaviour-preserving transformations*).

Dwie rzeczy, których refaktoryzacja **NIE** jest:
- **Nie** dodaje funkcjonalności ani nie naprawia bugów — te robisz **osobno** (nie mieszaj commitów „refactor" z „feature"/„fix"). To dwa różne kapelusze, które zakładasz naprzemiennie.
- **Nie** jest przepisaniem od zera (*rewrite*). Rewrite wyrzuca kod i buduje na nowo; refaktoryzacja przekształca istniejący, krok po kroku, cały czas mając działający system.

Rozróżnienie „refactoring" (małe m) vs „Refactoring" (nazwana transformacja z katalogu): pierwsze to czynność, drugie to konkretny, nazwany przepis, np. **Extract Method**, **Rename**.

## 2. Jak — katalog refaktoryzacji, rola testów

### Rola testów — siatka bezpieczeństwa (*safety net*)

To fundament. Skąd wiesz, że zachowanie się **nie zmieniło**? Z **testów**. Zielony pakiet testów przed refaktoryzacją i zielony po każdym małym kroku daje pewność, że transformacja była *behaviour-preserving*. **Bez testów manipulujesz kodem „na ślepo" — to nie refaktoryzacja, tylko ryzykowne przepisywanie.**

- Testy **jednostkowe** (szybkie, [[wiedza/04-narzedzia/junit-mockito|JUnit + Mockito]]) są idealne — dają natychmiastowy feedback po każdym kroku.
- Gdy kodu **nie da się** przetestować (splątane zależności), najpierw wprowadzasz **characterization tests** (testy charakteryzujące — utrwalają *obecne* zachowanie, nawet jeśli jest błędne), potem refaktoryzujesz. To technika z „Working Effectively with Legacy Code" (Feathers): legacy = kod bez testów.
- Automatyczne refaktoryzacje IDE są bezpieczniejsze, ale testy dają pewność, że nic się nie rozjechało (np. przy refleksji, serializacji, DI po nazwach).

### Wyzwalacze: code smells

Nie refaktoryzujesz „bo można", tylko gdy widzisz **code smell** — objaw, że coś jest nie tak w strukturze (szerzej: [[wiedza/12-wzorce/antywzorce]]). Przykłady i ich typowe lekarstwa:

| Smell | Typowa refaktoryzacja |
|---|---|
| Long Method | **Extract Method** |
| Duplicated Code | **Extract Method** / **Pull Up Method** |
| Large Class | **Extract Class** |
| Long Parameter List | **Introduce Parameter Object** |
| Feature Envy (metoda używa głównie cudzych danych) | **Move Method** |
| Magic Number | **Replace Magic Number with Constant** |
| Switch/if-else po typie | **Replace Conditional with Polymorphism** |
| Zagnieżdżone `if`-y | **Replace Nested Conditional with Guard Clauses** |

### Katalog kluczowych refaktoryzacji (Fowler)

**Codzienne, najczęstsze (chleb powszedni):**
- **Extract Method / Extract Function** — fragment metody → osobna, dobrze nazwana metoda. Zamiast komentarza „// oblicz rabat" masz `calculateDiscount(...)`. **Najważniejsza** refaktoryzacja tnia struktury.
- **Inline Method** — odwrotność: gdy metoda jest trywialna i nazwa nie mówi więcej niż ciało, wstaw ciało w miejsce wywołania. Bywa krokiem pośrednim przed inną refaktoryzacją.
- **Extract Variable** (dawniej *Introduce Explaining Variable*) — złożone wyrażenie → nazwana zmienna, np. `boolean isEligible = age >= 18 && hasConsent;`.
- **Rename (Method/Variable/Class)** — **najczęstsza i najbardziej niedoceniana**. Dobra nazwa to najtańsza dokumentacja. IDE robi to bezpiecznie w całym projekcie.

**Organizacja kodu między metodami i klasami:**
- **Extract Class** — jedna klasa robi za dużo (SRP) → wydziel spójną odpowiedzialność do nowej klasy.
- **Move Method / Move Field** — przenieś element tam, gdzie „grawituje" (gdzie są dane, których używa) — leczy *Feature Envy*.
- **Replace Temp with Query** — zmienna tymczasowa → metoda-zapytanie; ułatwia potem Extract Method (nie ma lokalnego stanu do przekazania).
- **Introduce Parameter Object** — kilka parametrów zawsze podróżujących razem → jeden obiekt (świetny kandydat na `record` w Javie 21).
- **Encapsulate Field** — publiczne pole → prywatne + akcesory (kontrola, walidacja, możliwość zmiany reprezentacji).

**Upraszczanie warunków (conditional logic):**
- **Decompose Conditional** — rozłóż skomplikowany `if/else`: warunek → metoda (`isSummer(date)`), gałęzie → metody.
- **Replace Nested Conditional with Guard Clauses** — zamiast piramidy zagnieżdżeń, wczesne `return`/`throw` dla przypadków brzegowych; „szczęśliwa ścieżka" zostaje płaska.
- **Replace Magic Number with Constant** — `0.05` → `private static final double VAT_RATE = 0.05;`.
- **Replace Conditional with Polymorphism** — rozgałęzienie po *typie* (`switch(shape.type)`) → polimorfizm/method dispatch. W Javie 21: albo hierarchia klas z nadpisaniem metody, albo **`sealed` interface + pattern matching w `switch`** ([[wiedza/01-jezyk/pattern-matching]]). `sealed` daje kompilatorowi **exhaustiveness checking** — po dodaniu nowego wariantu kompilator wskaże każdy `switch`, który trzeba uzupełnić (bezpieczniej niż „gubiący się" `default`).

### Wsparcie IDE

**IntelliJ IDEA** (`Ctrl+Alt+Shift+T`) wykonuje wiele z tych refaktoryzacji **automatycznie i bezpiecznie**: Rename z aktualizacją wszystkich referencji (także w komentarzach/stringach opcjonalnie), Extract Method z wykryciem parametrów i wartości zwracanej, Extract Variable/Constant/Field, Inline, Change Signature, Move, Introduce Parameter Object. To zdejmuje z człowieka mechaniczną, błędogenną robotę — ale **testy i tak potrzebne** (IDE nie widzi refleksji, konfiguracji po nazwach, kontraktów runtime).

## 3. Dlaczego / kiedy — dług techniczny, kiedy NIE

### Po co refaktoryzować

- **Czytelność** — kod czyta się wielokrotnie częściej niż pisze; nazwy i mały rozmiar metod = szybsze zrozumienie.
- **Utrzymywalność** — łatwiej zmieniać, mniejsza szansa na regresję.
- **Redukcja technical debt** — patrz niżej.
- **Przygotowanie pod nową funkcję** — słynne Kent Beckowe: *„make the change easy (warning: this may be hard), then make the easy change."* Najpierw refaktoryzuj tak, by planowana zmiana stała się prosta, potem dopiero ją wprowadź.

### Dług techniczny (*technical debt*)

Metafora Warda Cunninghama: pójście na skróty w kodzie jest jak **zaciągnięcie kredytu** — przyspiesza dostawę teraz, ale płacisz **odsetki** w postaci wolniejszego przyszłego rozwoju. Dopóki „długu" nie **spłacisz** (refaktoryzacją), każda kolejna zmiana kosztuje więcej.

- **Świadomy** dług (*deliberate*): „wiemy, że to skrót, wypuszczamy MVP, wrócimy" — może być **rozsądny**, jeśli spłacisz.
- **Nieświadomy** dług (*inadvertent*): „nie wiedzieliśmy, że tak się nie robi" — najgorszy, bo rośnie niezauważony.
- **Spłata**: ciągła, małymi porcjami (Boy Scout Rule), nie „wielki refaktoryzacyjny sprint kiedyś".

### Boy Scout Rule

*„Zostaw kod czystszym, niż go zastałeś."* (Robert C. Martin, za regułą skautów). Dotykasz pliku przy okazji zadania? Popraw drobiazg: zła nazwa, magiczna liczba, zbędny komentarz. Suma tysięcy drobnych ulepszeń > jeden heroiczny refaktoring, który nigdy nie nadejdzie.

### Refaktoryzacja a code review i TDD

- **Code review**: małe, skupione commity refaktoryzacyjne **osobno** od zmian funkcjonalnych — recenzent od razu widzi „tu tylko przesunięto/przemianowano, zachowanie bez zmian" i ufa (zwłaszcza jeśli to automatyczne refaktoryzacje IDE + zielone testy). Wymieszany diff („refactor + feature") jest trudny do recenzji i ukrywa błędy.
- **TDD** — cykl **red → green → REFACTOR**: (1) napisz padający test (*red*), (2) najprostszy kod, żeby przeszedł (*green*), (3) **refaktoryzuj** przy zielonych testach — usuń duplikację, popraw nazwy. Trzeci krok to nie opcja, to część rytmu; testy z kroku 1–2 są siatką bezpieczeństwa dla kroku 3.

### Kiedy NIE refaktoryzować

- **Gdy nie masz testów** (i nie zamierzasz najpierw dopisać characterization tests) — to nie będzie refaktoryzacja, tylko hazard.
- **Tuż przed deadlinem, bez potrzeby** — refaktoryzacja „przy okazji" pod presją zwiększa ryzyko regresji. Wyjątek: refaktoryzacja, która realnie **ułatwia** pilną zmianę („make the change easy first").
- **Gdy kusi Cię rewrite** — refaktoryzacja **≠** przepisanie od zera. Wielki rewrite („Big Rewrite") to klasyczna pułapka: długo brak działającego systemu, ryzyko powtórzenia tych samych błędów.
- **Kod, którego i tak zaraz usuwasz/nie ruszasz** — nie polerujesz martwego lub stabilnego, nietykanego kodu bez powodu (YAGNI dla refaktoryzacji).

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

**A) Extract Method + Guard Clauses na długiej metodzie** (typowo w warstwie serwisowej):

```java
// PRZED: Long Method, magiczne liczby, zagnieżdżenia
double finalPrice(Order o) {
    double result;
    if (o != null) {
        if (!o.items().isEmpty()) {
            double sum = 0;
            for (var i : o.items()) sum += i.price() * i.qty();
            if (sum > 500) result = sum * 0.9;   // rabat 10%
            else result = sum;
        } else result = 0;
    } else throw new IllegalArgumentException("null order");
    return result;
}

// PO: guard clauses + Extract Method + stała zamiast magic number
private static final double VOLUME_DISCOUNT_THRESHOLD = 500;
private static final double VOLUME_DISCOUNT_RATE = 0.9;

double finalPrice(Order o) {
    if (o == null) throw new IllegalArgumentException("null order");
    if (o.items().isEmpty()) return 0;
    double subtotal = subtotal(o);
    return subtotal > VOLUME_DISCOUNT_THRESHOLD
            ? subtotal * VOLUME_DISCOUNT_RATE
            : subtotal;
}

private double subtotal(Order o) {
    return o.items().stream()
            .mapToDouble(i -> i.price() * i.qty())
            .sum();
}
```
Zewnętrzne zachowanie **identyczne** — te same testy jednostkowe świecą się na zielono przed i po.

**B) Replace Conditional with Polymorphism → sealed + pattern matching** (Java 21):

```java
// PRZED: switch po "typie", łatwo zapomnieć o nowym kształcie
double area(Shape s) {
    switch (s.kind()) {
        case CIRCLE:    return Math.PI * s.r() * s.r();
        case RECTANGLE: return s.w() * s.h();
        default: throw new IllegalStateException();
    }
}

// PO: sealed + record + pattern matching switch (exhaustive)
sealed interface Shape permits Circle, Rectangle {}
record Circle(double r) implements Shape {}
record Rectangle(double w, double h) implements Shape {}

double area(Shape s) {
    return switch (s) {                       // brak default -> kompilator wymusza pełność
        case Circle c    -> Math.PI * c.r() * c.r();
        case Rectangle r -> r.w() * r.h();
    };
}
```
Po dodaniu `Triangle` kompilator **nie skompiluje** `switch`, dopóki nie obsłużysz nowego wariantu — regresja wyłapana na etapie kompilacji, nie na produkcji.

---

## ✅ Kryteria opanowania
- [ ] Zdefiniuję refaktoryzację jednym zdaniem: zmiana struktury **bez** zmiany zachowania, w małych krokach.
- [ ] Wyjaśnię, dlaczego bez testów to nie refaktoryzacja.
- [ ] Wymienię z głowy 6+ nazwanych refaktoryzacji Fowlera i pasujące do nich smells.
- [ ] Pokażę Replace Conditional with Polymorphism na `sealed` + pattern matching i wyjaśnię exhaustiveness.
- [ ] Wyjaśnię technical debt (świadomy/nieświadomy, odsetki, spłata) i Boy Scout Rule.
- [ ] Umiem powiedzieć, kiedy **NIE** refaktoryzować.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Czym różni się refaktoryzacja od przepisania (rewrite) i od naprawy buga?
- [ ] Jak refaktoryzować kod bez testów? (characterization tests, legacy code)
- [ ] Dlaczego commity „refactor" trzyma się osobno od „feature"?
- [ ] Jak `sealed` daje exhaustiveness w `switch` i czemu to bezpieczniejsze niż `default`?
- [ ] Co konkretnie robi IntelliJ przy Rename/Extract Method i czemu i tak potrzebne są testy?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Co to refaktoryzacja i jak zapewniasz, że nie psujesz zachowania?"
- [ ] „Jakie code smells znasz i jak je usuwasz?"
- [ ] „Jak podchodzisz do refaktoryzacji legacy code bez testów?"
- [ ] „Czym jest dług techniczny i jak go zarządzasz?"
- [ ] „Wyjaśnij cykl red-green-refactor."

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Czym jest refaktoryzacja (Fowler)?;Zmiana wewnętrznej struktury kodu bez zmiany zewnętrznego zachowania, w małych bezpiecznych krokach.
Dlaczego bez testów to nie refaktoryzacja?;Bez zielonych testów nie masz dowodu, że zachowanie się nie zmieniło — to ryzykowne przepisywanie na ślepo.
Rola testów w refaktoryzacji?;Siatka bezpieczeństwa (safety net) — zielone przed i po każdym kroku potwierdza behaviour-preserving.
Co to code smell?;Powierzchowny objaw problemu w strukturze kodu, będący wyzwalaczem refaktoryzacji (np. Long Method, Duplicated Code).
Najczęstsza i najważniejsza refaktoryzacja?;Rename oraz Extract Method — dobre nazwy i mały rozmiar metod to najtańsza czytelność.
Extract Method?;Wydzielenie fragmentu kodu do osobnej, dobrze nazwanej metody.
Inline Method?;Odwrotność Extract — wstawienie ciała trywialnej metody w miejsce wywołania.
Introduce Parameter Object?;Grupę parametrów podróżujących razem zamieniasz w jeden obiekt (w Javie 21 często record).
Replace Conditional with Polymorphism?;Rozgałęzienie po typie zastępujesz polimorfizmem/method dispatch (lub sealed + pattern matching).
Jak sealed pomaga przy Replace Conditional?;Daje exhaustiveness checking — kompilator wymusza obsłużenie każdego wariantu w switch, bez ryzykownego default.
Replace Nested Conditional with Guard Clauses?;Zagnieżdżone if-y zastępujesz wczesnymi return/throw dla przypadków brzegowych; happy path zostaje płaski.
Decompose Conditional?;Rozbicie złożonego if/else: warunek i gałęzie wydzielone do nazwanych metod.
Replace Magic Number with Constant?;Nienazwaną literałową liczbę zastępujesz nazwaną stałą (static final).
Co to technical debt (Cunningham)?;Metafora: skrót w kodzie to kredyt — przyspiesza teraz, ale płacisz odsetki (wolniejszy rozwój) do czasu spłaty refaktoryzacją.
Dług świadomy vs nieświadomy?;Świadomy (deliberate) — celowy skrót, rozsądny jeśli spłacony; nieświadomy (inadvertent) — z niewiedzy, rośnie niezauważony.
Boy Scout Rule?;Zostaw kod czystszym, niż go zastałeś — spłacaj dług małymi porcjami przy okazji.
Cykl TDD?;Red (padający test) → Green (najprostszy kod, by przeszedł) → Refactor (posprzątaj przy zielonych testach).
Cytat Kenta Becka o zmianie?;Make the change easy (this may be hard), then make the easy change — najpierw refaktoruj, potem dodaj funkcję.
Kiedy NIE refaktoryzować?;Bez testów, pod presją deadline'u bez potrzeby, oraz gdy kusi rewrite (refaktoryzacja to nie przepisanie od zera).
Czemu commity refactor i feature trzymać osobno?;Ułatwia code review — recenzent widzi, że zachowanie bez zmian, i mixed diff nie ukrywa błędów.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/04-narzedzia/junit-mockito]] · [[wiedza/12-wzorce/antywzorce]] · [[wiedza/01-jezyk/pattern-matching]] · [[wiedza/01-jezyk/record-sealed-enum]]
</content>
</invoke>
