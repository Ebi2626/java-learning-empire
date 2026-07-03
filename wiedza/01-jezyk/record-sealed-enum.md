---
temat: "record, sealed, enum — nowoczesne typy danych"
faza: 1
status: nieopanowany
priorytet: 🔴
tags: [java, jezyk, record, sealed, enum, adt]
powiazane: ["[[wiedza/01-jezyk/pattern-matching]]", "[[wiedza/01-jezyk/immutability]]", "[[wiedza/00-fundament/jdk-jre-jvm]]"]
---

# record, sealed, enum — nowoczesne typy danych

> **TL;DR:** **`record`** (Java 16) to niemutowalny *nominal tuple* — kompilator generuje pola `final`, kanoniczny konstruktor, akcesory, `equals`/`hashCode`/`toString`. **`sealed`** (Java 17) zamyka hierarchię przez `permits` — kompilator zna *wszystkie* podtypy, co daje **exhaustive `switch`** bez `default`. Razem: `sealed interface` + `record`-warianty + `switch` z pattern matching = **algebraic data types (ADT)** w Javie.

## 1. Co — definicja i API

### record — data carrier
```java
record Point(int x, int y) {}          // jedna linia = pełnoprawna klasa
var p = new Point(3, 4);
p.x();                                   // 3  (akcesor, NIE getX())
p.equals(new Point(3, 4));               // true (value equality)
p.toString();                            // "Point[x=3, y=4]"
```
Deklarujesz **components** w nagłówku; reszta jest generowana. `record` jest niejawnie `final`, dziedziczy po `java.lang.Record`, nie może `extends` (ale może `implements` interfejsy).

### sealed — zamknięta hierarchia
```java
sealed interface Shape permits Circle, Rectangle {}
record Circle(double r) implements Shape {}
record Rectangle(double w, double h) implements Shape {}
```
`permits` wylicza **dokładny** zbiór dozwolonych podtypów. Każdy podtyp MUSI być `final`, `sealed` albo `non-sealed`.

### enum — to pełnoprawna klasa
```java
enum Planet {
    EARTH(5.976e24, 6.37e6),
    MARS(6.421e23, 3.39e6);                     // stałe = instancje (singletony)

    private final double mass, radius;
    Planet(double mass, double radius) {        // konstruktor: zawsze private
        this.mass = mass; this.radius = radius;
    }
    double gravity() { return 6.67e-11 * mass / (radius * radius); }
}
Planet.EARTH.gravity();
Planet.values();                                // Planet[] (kopia tablicy)
Planet.valueOf("MARS");                         // MARS (IllegalArgumentException jak brak)
Planet.EARTH.ordinal();                         // 0
```

## 2. Jak — co generuje kompilator / pod spodem

### record — desugaring
Dla `record Point(int x, int y) {}` `javac` produkuje (sprawdź `javap -p Point`):

```java
final class Point extends java.lang.Record {
    private final int x;                         // komponenty → pola FINAL
    private final int y;

    Point(int x, int y) { this.x = x; this.y = y; }   // KANONICZNY konstruktor

    public int x() { return x; }                 // akcesory = nazwa komponentu()
    public int y() { return y; }

    public boolean equals(Object o) { ... }       // value-based: porównuje WSZYSTKIE komponenty
    public int hashCode() { ... }                 // spójne z equals
    public String toString() { ... }              // "Point[x=.., y=..]"
}
```
- `equals`/`hashCode`/`toString` używają **invokedynamic** (`ObjectMethods.bootstrap`) — kompilator nie wkleja kodu wprost, generuje wywołanie bootstrap, które buduje implementację z `MethodHandle` po komponentach. Stąd mały bytecode i spójność.
- **Compact constructor** — walidacja/normalizacja bez listy parametrów; przypisania pól dorabia kompilator na końcu:
```java
record Range(int lo, int hi) {
    Range {                                       // brak (int lo, int hi) i brak this.lo=...
        if (lo > hi) throw new IllegalArgumentException("lo>hi");
        lo = Math.min(lo, hi);                    // można normalizować parametry przed przypisaniem
    }
}
```
- Możesz **nadpisać** dowolny akcesor lub dać jawny kanoniczny konstruktor, ale zwykle nie warto.
- **Niemutowalność jest płytka (shallow):** pole referencyjne `List<X>` wskazuje wciąż na mutowalną listę — w compact constructor rób `List.copyOf(...)` żeby uzyskać prawdziwą immutability.

### sealed — kontrakt sprawdzany przy kompilacji
- `permits` zapisywane jest w atrybucie klasy **`PermittedSubclasses`** (ClassFile). JVM **weryfikuje** przy ładowaniu, że klasa próbująca implementować `sealed` jest na liście — to gwarancja *runtime*, nie tylko składniowa.
- Podtypy muszą być w **tym samym module** (lub pakiecie, jeśli unnamed module).
- Modyfikatory podtypu zamykają vs otwierają dalsze rozszerzanie:
  - `final` — koniec hierarchii (typowo dla `record`).
  - `sealed` — podtyp sam wylicza swoje `permits` (hierarchia zagnieżdżona).
  - `non-sealed` — celowo **otwiera** gałąź na dowolne dziedziczenie.
- Efekt kluczowy: kompilator zna **pełen** zbiór wariantów → potrafi sprawdzić **exhaustiveness** `switch`. Brak pokrytego wariantu = błąd kompilacji, nie `default`.

### enum — pod spodem
- `enum E {...}` to cukier na `final class E extends java.lang.Enum<E>`; każda stała to `public static final` instancja, inicjalizowana w bloku `static`. Konstruktor jest zawsze prywatny — nie da się stworzyć nowych instancji → naturalne singletony.
- **Constant-specific body** (per-constant method): gdy stała ma `{ ... }`, kompilator tworzy **anonimową podklasę** enuma:
```java
enum Op {
    PLUS  { public int apply(int a, int b) { return a + b; } },
    TIMES { public int apply(int a, int b) { return a * b; } };
    public abstract int apply(int a, int b);      // każda stała MUSI dostarczyć ciało
}
Op.PLUS.apply(2, 3);   // 5
```
- **`EnumMap` / `EnumSet`** są szybkie, bo wykorzystują `ordinal()`:
  - `EnumMap` to wewnętrznie **tablica** indeksowana `ordinal()` (O(1), brak hashowania, brak kolizji, zachowuje kolejność deklaracji).
  - `EnumSet` to **bitset** (jeden `long` dla ≤64 stałych — `RegularEnumSet`; `JumboEnumSet` dla większych). Operacje zbiorowe = operacje bitowe.
- `values()` zwraca **świeżą kopię** tablicy przy każdym wywołaniu (obronnie — tablice są mutowalne) → nie wołaj w pętli krytycznej.
- `switch` na enumie kompiluje się do skoku po `ordinal()` (tablica skoków), więc jest tani.

## 3. Dlaczego / kiedy — kompromisy i wąskie gardła

| Konstrukt | Używaj gdy | NIE używaj gdy |
|---|---|---|
| `record` | DTO, value object, klucz mapy, wynik metody, krotka, zdarzenie, węzeł AST | obiekt mutowalny, encja JPA z `@Id`/proxy, potrzebujesz dziedziczenia pól |
| `sealed` | znasz pełen zbiór wariantów (stany, komendy, wyniki, drzewa) | hierarchia ma być otwarta na rozszerzenia przez plugin/innych |
| `enum` | skończony, znany zbiór stałych; strategia per-stała; flagi (`EnumSet`) | zbiór zmienia się dynamicznie/runtime; potrzeba wielu instancji tej samej „wartości” |

- **record + JPA:** Hibernate wymaga konstruktora bezargumentowego i mutowalnych pól — `record` jako encja `@Entity` nie działa. Świetnie za to jako **projekcja DTO** (`SELECT new ...` / interface-based ma swoje miejsce).
- **record nie zastępuje klasy z zachowaniem** — to *data*, nie *behavior*. Logika bogata → zwykła klasa.
- **sealed kontra otwartość:** zamykasz API. Dodanie wariantu psuje binarną kompatybilność, ale to *zaleta* — kompilator wskaże wszystkie `switch` do uzupełnienia (fail-fast). Wzorzec Visitor staje się zbędny.
- **enum a serializacja/persistencja:** zapisuj po `name()`, nie po `ordinal()` — `ordinal` zmienia się przy przestawieniu stałych i cicho psuje dane.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Modelowanie pól figur jako **ADT** — `sealed interface` + `record`-warianty + **exhaustive switch** z pattern matching (Java 21):

```java
sealed interface Shape permits Circle, Rectangle, Triangle {}

record Circle(double r)               implements Shape {}
record Rectangle(double w, double h)  implements Shape {}
record Triangle(double base, double h) implements Shape {}

static double area(Shape s) {
    return switch (s) {                                  // brak default!
        case Circle c              -> Math.PI * c.r() * c.r();
        case Rectangle(double w, double h) -> w * h;     // record deconstruction
        case Triangle t            -> 0.5 * t.base() * t.h();
    };                                                    // dodaj 4. wariant → switch się NIE skompiluje
}
```
Dodanie `record Square(...) implements Shape` zmusza kompilator do błędu na każdym `switch` nad `Shape` bez `default` — to właśnie *kontrola wariantów*. W praktyce: modelowanie stanów (`sealed interface PaymentState`), komend/eventów (CQRS/event sourcing), wyników (`Result = Success | Failure`), węzłów parsera/AST.

---

## ✅ Kryteria opanowania
- [ ] Wyliczę z głowy, co kompilator generuje dla `record` i napiszę compact constructor z walidacją.
- [ ] Wyjaśnię, czemu niemutowalność recordu jest *płytka* i jak ją pogłębić.
- [ ] Powiem, czym jest `permits` i dlaczego daje exhaustive switch (atrybut `PermittedSubclasses`).
- [ ] Rozróżnię `final` / `sealed` / `non-sealed` na podtypie.
- [ ] Wyjaśnię, czemu `EnumMap`/`EnumSet` są wydajne (tablica po `ordinal`, bitset).
- [ ] Złożę `sealed interface` + `record` + exhaustive `switch` jako ADT.

### 🔲 Black-box check (wyjaśnij od podstaw)
- [ ] Czemu `equals`/`hashCode` recordu działają przez `invokedynamic`, a nie zwykły kod?
- [ ] Co realnie robi compact constructor — gdzie kompilator dokleja przypisania pól?
- [ ] Gdzie zapisany jest `permits` i kto go egzekwuje (kompilator vs JVM przy load-time)?
- [ ] Co generuje się dla enuma z ciałem per-stałą (constant-specific body)?
- [ ] Czemu `values()` zwraca kopię i jaki to ma koszt w pętli?
- [ ] Dlaczego exhaustive switch nad sealed nie wymaga `default`?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Czym record różni się od klasy z Lombok `@Value`?” (język vs procesor adnotacji, `Record` base, ADT-friendly)
- [ ] „Po co `sealed`, skoro mam `final` i `abstract`?” (zamknięty *zbiór* podtypów + exhaustiveness)
- [ ] „Czemu enum to dobry singleton?” (prywatny konstruktor, gwarancja JVM, serializacja safe)
- [ ] „Jak zamodelujesz `Result<Success|Failure>` w Javie?” (sealed + records + switch)
- [ ] „Record jako encja JPA — czemu nie?” (no-arg ctor, mutowalność, proxy)

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/01-jezyk.csv`. Format: `Pytanie;Odpowiedź`.*
```
Od której Javy record jest finalny (stabilny)?;Java 16 (preview 14/15, finalne w 16).
Co kompilator generuje dla recordu?;Pola final, kanoniczny konstruktor, akcesory nazwa(), equals/hashCode/toString.
Jak nazywa się akcesor komponentu x w recordzie?;x() — nie getX(); nazwa równa nazwie komponentu.
Po co compact constructor?;Walidacja i normalizacja argumentów bez listy parametrów; przypisania pól dorabia kompilator.
Czy record może dziedziczyć po klasie?;Nie (jest final i extends java.lang.Record); może tylko implements interfejsów.
Czemu niemutowalność recordu jest płytka?;Pola są final, ale referencja może wskazywać na mutowalny obiekt — kopiuj (List.copyOf) w compact ctor.
Co robi słowo kluczowe permits w sealed?;Wylicza dokładny, zamknięty zbiór dozwolonych podtypów hierarchii.
Jakie modyfikatory musi mieć podtyp sealed?;final, sealed albo non-sealed.
Co daje non-sealed?;Celowo otwiera gałąź sealed-hierarchii na dowolne dziedziczenie.
Gdzie zapisany jest zbiór permits i kto go egzekwuje?;W atrybucie ClassFile PermittedSubclasses; weryfikuje go JVM przy ładowaniu klasy.
Czemu sealed daje exhaustive switch bez default?;Kompilator zna pełen zbiór wariantów, więc sam sprawdza pokrycie wszystkich przypadków.
Czym pod spodem jest enum?;Final class extends java.lang.Enum; każda stała to public static final instancja, konstruktor prywatny.
Co to constant-specific body w enumie?;Ciało {...} przy stałej → kompilator tworzy anonimową podklasę enuma z nadpisaną metodą.
Czemu EnumMap jest szybki?;To tablica indeksowana ordinal() — O(1), bez hashowania i kolizji, zachowuje kolejność deklaracji.
Czym jest EnumSet pod spodem?;Bitset (long dla <=64 stałych, RegularEnumSet; JumboEnumSet wyżej) — operacje zbiorowe to operacje bitowe.
Czemu values() warto cache'ować?;Zwraca świeżą kopię tablicy przy każdym wywołaniu (obronnie) — koszt w gorącej pętli.
Po name() czy ordinal() persystować enum?;Po name() — ordinal zmienia się przy przestawieniu stałych i cicho psuje dane.
Jak zamodelować ADT w Javie?;sealed interface + record-warianty + exhaustive switch z pattern matching.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/01-jezyk/pattern-matching]] · [[wiedza/01-jezyk/immutability]] · [[wiedza/00-fundament/jdk-jre-jvm]]
