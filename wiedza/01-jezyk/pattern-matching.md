---
temat: "Pattern matching i nowoczesny switch"
faza: 1
status: nieopanowany
priorytet: 🟡
tags: [java, jezyk, pattern-matching, switch, java21]
powiazane: ["[[wiedza/01-jezyk/record-sealed-enum]]", "[[wiedza/01-jezyk/sealed-classes]]", "[[wiedza/00-fundament/wersje-javy-lts]]"]
---

# Pattern matching i nowoczesny switch

> **TL;DR:** Od Javy 16–21 `instanceof` i `switch` zyskały **pattern matching** — testujesz typ i od razu dostajesz
> *binding variable* (`if (o instanceof String s)`), a `switch` stał się **wyrażeniem** ze strzałkami (`->`, bez
> fall-through, `yield`), które dopasowuje **type patterns** (`case Shape s`), **guarded patterns** (`case String s when ...`),
> `case null` i **dekonstruuje rekordy** (`case Point(int x, int y)`). W parze z `sealed` daje **exhaustive switch bez
> `default`** — kompilator wymusza pokrycie wszystkich permitów (algebraiczne typy danych, ADT). To zastępuje wzorzec
> **Visitor** i drabiny `if/else if (instanceof)`. **Wszystkie te cechy są stabilne (final) w Java 21 LTS.**

## 1. Co — definicja i API

### Pattern matching for `instanceof` (Java 16, JEP 394 — stable)
Łączy test typu z rzutowaniem i deklaracją zmiennej (*binding variable*):
```java
// PRZED:
if (o instanceof String) {
    String s = (String) o;      // ręczne, powtórzone rzutowanie
    return s.length();
}
// PO (Java 16+):
if (o instanceof String s) {    // s to binding variable
    return s.length();          // s jest już typu String, bez castu
}
```

### Switch expressions (Java 14, JEP 361 — stable)
`switch` może **zwracać wartość** (jest wyrażeniem), używa strzałek `->` (brak *fall-through*, brak `break`),
a w bloku zwraca przez `yield`:
```java
int dni = switch (miesiac) {
    case JAN, MAR, MAY, JUL, AUG, OCT, DEC -> 31;
    case APR, JUN, SEP, NOV               -> 30;
    case FEB                              -> { yield rok % 4 == 0 ? 29 : 28; } // blok → yield
};                                                                            // ; bo to wyrażenie
```
Switch *expression* musi być **wyczerpujący** (*exhaustiveness*) — dla `enum` wymaga wszystkich stałych
albo `default`; w przeciwnym razie błąd kompilacji.

### Pattern matching for `switch` (Java 21, JEP 441 — **final**)
Sfinalizowane w 21 (preview w 17–20). W `case` można dać **type pattern**, **guarded pattern** (`when`),
oraz `case null`:
```java
String opis = switch (obj) {
    case null            -> "brak";                       // jawna obsługa null
    case Integer i       -> "int = " + i;                 // type pattern
    case String s when s.isBlank() -> "pusty napis";      // guarded pattern (when)
    case String s        -> "napis dł. " + s.length();
    default              -> "coś innego";
};
```
Kolejność `case` ma znaczenie — dopasowanie jest **od góry**; `case String s` przed
`case String s when ...` to błąd (*pattern dominance* — wcześniejszy wzorzec „przesłania" późniejszy).

### Record patterns / dekonstrukcja (Java 21, JEP 440 — **final**)
Dopasowanie do rekordu wyciąga jego komponenty (*deconstruction*), także **zagnieżdżone**:
```java
record Point(int x, int y) {}
record Line(Point from, Point to) {}

String s = switch (obj) {
    case Point(int x, int y)              -> "punkt " + x + "," + y;     // dekonstrukcja
    case Line(Point(var x1, var y1),
              Point(var x2, var y2))      -> "linia " + x1 + " -> " + x2; // zagnieżdżona
    default                              -> "?";
};
```

## 2. Jak — flow scoping, exhaustiveness, jak kompilator to sprawdza

### Flow scoping (zakres binding variable)
Binding variable z `instanceof` obowiązuje **dokładnie tam, gdzie kompilator dowiódł, że test się powiódł** —
to **flow scoping** (zasięg sterowany przepływem), a nie sztywno „blok po `if`". Dlatego działa też w warunkach
złożonych i po `return`/`throw`:
```java
if (o instanceof String s && s.length() > 3) { ... } // s widoczne po && (lewa strona już prawdziwa)
if (!(o instanceof String s)) return;                // early return
// tutaj s JEST widoczne — bo gdyby o nie było String, już byśmy wyszli
System.out.println(s.length());
```
Kompilator analizuje, w których ścieżkach binding jest „definitely matched". Po `||` w pozytywnej gałęzi
binding nie jest widoczny, bo nie ma dowodu dopasowania.

### Exhaustiveness — jak kompilator to sprawdza
Switch *expression* (i switch *statement* z patternami) musi pokryć **wszystkie możliwe wartości**, inaczej
*„the switch ... does not cover all possible input values"*. Kompilator dowodzi pokrycia przez:
- typ selektora (np. `enum` → znane stałe),
- przy `sealed` — zamknięty zbiór `permits` (zna **wszystkie** podtypy w czasie kompilacji),
- gdy nie umie dowieść — wymaga `default` (lub `case null, default`).

**Kluczowa synergia z `sealed`** (zob. [[wiedza/01-jezyk/record-sealed-enum]]): nad `sealed` hierarchią switch jest
exhaustive **bez `default`** — kompilator wie, że istnieją tylko wyliczone permity. To realizacja **algebraicznych
typów danych** (sum types). Gdy dodasz nowy permit, switch **przestanie się kompilować** dopóki go nie obsłużysz —
bezpieczeństwo w czasie kompilacji zamiast cichego błędu w runtime:
```java
sealed interface Shape permits Circle, Square {}
record Circle(double r) implements Shape {}
record Square(double a) implements Shape {}

double pole = switch (shape) {                  // BEZ default!
    case Circle(double r) -> Math.PI * r * r;
    case Square(double a) -> a * a;
};                                              // dodanie Triangle → błąd kompilacji tutaj
```

### Obsługa `null` i kolejność
- Klasyczny `switch` rzuca `NullPointerException` na `null`. Switch z patternami **nadal rzuca NPE**, chyba że
  jest jawny `case null` (albo `case null, default`).
- Dopasowanie idzie **z góry na dół**; pierwszy pasujący `case` wygrywa. Kompilator odrzuca *zdominowane* wzorce
  (np. nieosiągalny `case String s` po wcześniejszym `case CharSequence cs`).
- `case String s when cond` — najpierw musi zgodzić się typ, potem strażnik (`when`); jeśli `when` da `false`,
  szukamy dalej. Jeśli żaden guarded pattern nie złapie wartości danego typu, a brak fallbacka → musi być `default`,
  by zachować exhaustiveness.

## 3. Dlaczego / kiedy — zastępuje Visitor / `instanceof`

- **Zastępuje drabiny `instanceof`** — zamiast `if (x instanceof A a) {...} else if (x instanceof B b) {...}`
  jeden czytelny, exhaustive `switch`. Mniej boilerplate'u, mniej miejsca na pominięty przypadek.
- **Zastępuje wzorzec Visitor** — Visitor istniał, bo Java nie miała dopasowania po typie; trzeba było pisać
  `accept(visitor)` + interfejs z metodą na każdy podtyp. `sealed` + record patterns dają to samo (rozszerzanie
  „po operacjach") **bez ceremoniału** double-dispatch. Dodanie operacji = nowy `switch`, a nie zmiana całej hierarchii.
- **Kiedy NIE / pułapki:** dla zachowania *należącego do typu* nadal lepszy bywa **polimorfizm** (metoda w interfejsie),
  zwłaszcza gdy operacja jest „częścią" obiektu. Pattern matching wybieraj, gdy operacji jest wiele i są zewnętrzne
  wobec danych (np. serializacja, obliczenia, render) — wtedy ADT + switch wygrywa.
- **Duży krok ku ekspresyjności:** Java zbliża się do języków funkcyjnych (Scala, Kotlin, Rust `match`/`enum`).
  Kombinacja `records` + `sealed` + pattern matching = **wbudowane ADT** z kontrolą wyczerpywalności przez kompilator —
  modelujesz dane deklaratywnie, a kompilator pilnuje kompletności.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Modelowanie wyniku/zdarzeń jako sealed hierarchii i liczenie pól figur z guarded patterns:
```java
sealed interface Shape permits Circle, Square, Rectangle {}
record Circle(double r)            implements Shape {}
record Square(double a)            implements Shape {}
record Rectangle(double w, double h) implements Shape {}

static double area(Shape s) {
    return switch (s) {
        case null                          -> 0.0;                  // obsługa null
        case Circle(double r) when r <= 0  -> 0.0;                  // guarded: niepoprawny promień
        case Circle(double r)              -> Math.PI * r * r;
        case Square(double a)              -> a * a;
        case Rectangle(double w, double h) -> w * h;
    };  // BEZ default — sealed gwarantuje exhaustiveness; nowy permit → błąd kompilacji
}
```
Typowe miejsca: warstwa domeny (np. `sealed interface PaymentResult permits Success, Declined, Pending`),
parsowanie AST/komend, obsługa typów zdarzeń w event-driven, mapowanie DTO. Dawniej tu był Visitor lub `instanceof`.

---

## ✅ Kryteria opanowania
- [ ] Napiszę z głowy `instanceof` z binding variable i wyjaśnię **flow scoping**.
- [ ] Wyjaśnię różnicę między switch *statement* a *expression* (wartość, `->`, `yield`, brak fall-through).
- [ ] Zbuduję exhaustive `switch` nad `sealed` **bez `default`** i uzasadnię, czemu kompilator to akceptuje.
- [ ] Użyję guarded pattern (`when`) i poprawnie ułożę kolejność `case` (dominance).
- [ ] Zdekonstruuję rekord, także zagnieżdżony (record pattern).
- [ ] Wyjaśnię, jak to zastępuje Visitor / drabinę `instanceof` i kiedy mimo to wybrać polimorfizm.
- [ ] Wiem, które cechy są **final w Java 21** (wszystkie z tej notatki).

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co to *flow scoping* i czemu binding działa po `!(o instanceof T t) return;`?
- [ ] Jak kompilator dowodzi *exhaustiveness* dla `sealed`? Co się stanie po dodaniu permitu?
- [ ] Jak `switch` z patternami traktuje `null` bez `case null`? (NPE)
- [ ] Czym jest *pattern dominance* i kiedy `case` jest nieosiągalny?
- [ ] Dlaczego switch *expression* nie ma fall-through, a `yield` jest potrzebny w bloku?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Jak pattern matching zastępuje wzorzec Visitor?"
- [ ] „Czym jest exhaustive switch nad sealed interface i co daje brak `default`?"
- [ ] „Różnica między switch statement a switch expression?"
- [ ] „Co to guarded pattern i jak działa kolejność `case`?"
- [ ] „Które z tych cech są stabilne, a które były preview — od której wersji?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Co daje pattern matching for instanceof?;Test typu + automatyczne rzutowanie do binding variable, np. if (o instanceof String s) — s jest już typu String bez castu.
Od której wersji instanceof pattern matching jest stable?;Java 16 (JEP 394).
Co to flow scoping?;Zasięg binding variable sterowany przepływem — zmienna widoczna tam, gdzie kompilator dowiódł, że test się powiódł (np. po early return przy !(o instanceof T t)).
Czym różni się switch expression od statement?;Expression zwraca wartość, używa -> (bez fall-through), w bloku yield, musi być wyczerpujący i kończy się średnikiem.
Po co yield w switch?;Zwraca wartość z bloku { } w switch expression (gdy nie używamy jednolinijkowego ->).
Od której wersji switch expressions są stable?;Java 14 (JEP 361).
Co to exhaustiveness w switch?;Wymóg pokrycia wszystkich możliwych wartości selektora; inaczej błąd kompilacji.
Kiedy switch nad sealed nie potrzebuje default?;Gdy obsłuży wszystkie permity — kompilator zna zamknięty zbiór podtypów (algebraiczne typy danych).
Co się stanie po dodaniu nowego permitu do sealed?;Exhaustive switch przestanie się kompilować, dopóki nowy podtyp nie zostanie obsłużony (bezpieczeństwo w czasie kompilacji).
Co to guarded pattern?;case z warunkiem when, np. case String s when s.length() > 5 — dopasowanie typu plus dodatkowy predykat.
Jak switch z patternami traktuje null?;Nadal rzuca NPE, chyba że jest jawny case null (lub case null, default).
Co to record pattern / dekonstrukcja?;case Point(int x, int y) — wyciąga komponenty rekordu; można zagnieżdżać, np. Line(Point(var x1,var y1), Point(...)).
Od której wersji record patterns i pattern matching for switch są final?;Java 21 (JEP 440 record patterns, JEP 441 pattern matching for switch).
Co to pattern dominance?;Sytuacja, gdy wcześniejszy case przesłania późniejszy (czyni go nieosiągalnym) — błąd kompilacji.
Jak pattern matching zastępuje Visitor?;sealed + record patterns + switch dają dopasowanie po typie i exhaustiveness bez double-dispatch i metod accept().
Dlaczego switch expression nie ma fall-through?;Strzałki -> wykonują tylko jedną gałąź; nie ma przeskoku do kolejnych case, więc break jest zbędny.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/01-jezyk/record-sealed-enum]] · [[wiedza/01-jezyk/sealed-classes]] · [[wiedza/00-fundament/wersje-javy-lts]]
