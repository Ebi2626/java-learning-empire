---
temat: "Programowanie obiektowe w Javie (OOP)"
faza: 1
status: nieopanowany
priorytet: 🔴
tags: [java, jezyk, oop, polimorfizm, dziedziczenie]
powiazane: ["[[wiedza/00-fundament/jdk-jre-jvm]]", "[[wiedza/01-jezyk/generics]]", "[[wiedza/01-jezyk/rekordy-sealed]]", "[[wiedza/02-jvm/model-pamieci]]"]
---

# Programowanie obiektowe w Javie (OOP)

> **TL;DR:** OOP w Javie opiera się na czterech filarach — **encapsulation**, **inheritance**,
> **polymorphism**, **abstraction**. Stan ukrywasz za interfejsem (program *to an interface*),
> zachowanie wybierasz w runtime przez **dynamic dispatch** (`invokevirtual` + vtable),
> a strukturę budujesz raczej z **kompozycji** niż z głębokiego dziedziczenia. Klasa = *co to jest*
> (stan + implementacja), interfejs = *co to potrafi* (kontrakt).

## 1. Co — cztery filary, klasy vs interfejsy, dostęp

### Cztery filary — konkretnie

**Encapsulation** — łączysz stan i operacje na nim w jednej jednostce i **ukrywasz reprezentację**
za publicznym API. Pole `private`, dostęp przez metody → możesz zmienić wewnętrzną reprezentację bez
psucia klientów. Encapsulation to nie „gettery na wszystko", tylko utrzymanie **inwariantów**.

```java
final class Konto {
    private long groszy;                       // reprezentacja ukryta
    void wplac(long g) {
        if (g <= 0) throw new IllegalArgumentException();   // inwariant: dodatnie kwoty
        groszy += g;
    }
    long saldoGroszy() { return groszy; }       // brak settera → klasa pilnuje stanu
}
```

**Abstraction** — wystawiasz *istotne* operacje, chowasz szczegóły *jak*. `List<T>` mówi „dodaj/pobierz po
indeksie"; czy pod spodem jest tablica (`ArrayList`) czy lista wiązana (`LinkedList`) — nieistotne dla klienta.
Abstrakcję realizujesz przez **interface** i **abstract class**.

**Inheritance** — klasa pochodna przejmuje stan i zachowanie bazowej (`class B extends A`), modeluje relację
**is-a** i pozwala reużyć/rozszerzyć kod. W Javie jedna klasa bazowa (single inheritance of state),
wiele interfejsów.

**Polymorphism** — jedna referencja typu bazowego, wiele konkretnych zachowań w runtime. `Figura f = new Kolo();
f.pole();` woła `Kolo.pole()`. To **subtype polymorphism** (override). Osobno: **ad-hoc polymorphism** =
overloading (rozstrzygany w kompilacji) i **parametric polymorphism** = generics ([[wiedza/01-jezyk/generics]]).

### Klasy vs interfejsy

| | **class** | **interface** |
|---|---|---|
| Stan (pola instancji) | tak | nie (tylko `public static final` stałe) |
| Implementacja metod | tak | `default` / `static` / `private` (od 8/9) |
| Liczba „rodziców" | 1 (`extends`) | wiele (`implements`) |
| Konstruktor | tak | nie |
| Modeluje | *co to jest* (is-a + stan) | *co to potrafi* (kontrakt, can-do) |

### Interfejsy — default / static / private (Java 8/9) i po co

```java
interface Powitanie {
    String imie();
    default String hello() { return "Cześć, " + sformatuj(imie()); }  // 8: domyślna impl
    private String sformatuj(String s) { return s.trim(); }            // 9: współdzielona logika, ukryta
    static Powitanie anonimowy() { return () -> "?"; }                 // 8: metoda fabryczna na interfejsie
}
```

- **`default`** (Java 8) — pozwala **dodać metodę do interfejsu bez psucia istniejących implementacji**
  (interface evolution). Dzięki temu do `Collection` dorzucono `stream()`, `forEach()` itd. bez łamania
  milionów klas. To jest właściwy powód istnienia default methods.
- **`static`** (Java 8) — metody pomocnicze/fabryczne trzymane „przy interfejsie" zamiast w osobnej klasie
  `XxxUtils` (np. `List.of(...)`, `Comparator.comparing(...)`).
- **`private`** (Java 9) — wydzielasz wspólny kod z kilku metod `default`/`static`, nie wystawiając go w API.

### Klasy abstrakcyjne vs interfejsy — kiedy które

- **abstract class** — gdy masz **wspólny stan** (pola), **częściową implementację**, chcesz wymusić
  konstruktor i relację **is-a** w jednej linii dziedziczenia. Może mieć `protected` pola, metody finalne.
- **interface** — gdy definiujesz **kontrakt/zdolność** (capability), który mogą spełnić **niespokrewnione**
  klasy, i chcesz **wielodziedziczenia typu**. Brak stanu instancji.
- Reguła praktyczna: **interface domyślnie**, abstract class dopiero gdy musisz współdzielić stan/konstrukcję.
  Częsty wzorzec: interfejs + opcjonalna `Abstract...` baza ze szkieletem (template method), np.
  `Collection` + `AbstractCollection`.

### Modyfikatory dostępu — dokładnie co widać skąd

| modyfikator | ta klasa | ten **pakiet** | podklasa (inny pakiet) | reszta świata |
|---|:---:|:---:|:---:|:---:|
| `private` | ✅ | ❌ | ❌ | ❌ |
| *brak* (**package-private**) | ✅ | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ✅ (przez dziedziczenie) | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

Subtelność `protected`: w innym pakiecie podklasa widzi `protected` składnik **tylko na obiektach swojego
typu** (lub podtypu), nie na dowolnej referencji typu bazowego. Package-private (brak słowa) to silne,
często niedoceniane narzędzie hermetyzacji wewnątrz modułu. W Javie 9+ dochodzi jeszcze granica **modułu**
(`module-info.java` decyduje, które `public` API jest faktycznie wyeksportowane na zewnątrz).

## 2. Jak — pod spodem (dynamic dispatch, vtable)

### Override vs overload — runtime vs compile time

- **Overriding (przesłanianie)** — podklasa daje nową implementację metody o tej samej sygnaturze.
  Wybór implementacji następuje **w runtime** na podstawie **rzeczywistego typu obiektu** (dynamic dispatch).
  Wymaga zgodnej sygnatury i kowariantnego typu zwracanego; stąd adnotacja `@Override` (kompilator
  pilnuje, że faktycznie coś przesłaniasz).
- **Overloading (przeciążanie)** — kilka metod o **tej samej nazwie, różnych parametrach**. Którą wołasz
  rozstrzyga **kompilator** po **statycznym typie** argumentów (static binding). Brak dynamiki.

```java
class A { Object f() { return "A"; } }
class B extends A { @Override String f() { return "B"; } }  // override (kowariantny zwrot)

void g(Object o) { /* ... */ }
void g(String s) { /* ... */ }   // overload — wybór po typie statycznym
A a = new B();
a.f();              // RUNTIME: "B" (dynamic dispatch — rzeczywisty typ to B)
g((Object) "x");    // COMPILE: woła g(Object), mimo że obiekt to String
```

### Jak JVM wybiera metodę — vtable i invokevirtual

Wywołanie metody instancyjnej kompiluje się do bytecode `invokevirtual` (dla metod interfejsu —
`invokeinterface`). To **nie** wkompilowuje adresu konkretnej metody — JVM rozwiązuje cel **w czasie wykonania**:

1. Każda załadowana klasa ma w metaspace **vtable** (virtual method table) — tablicę wskaźników do
   faktycznych implementacji metod wirtualnych. Metoda przesłonięta w podklasie ma w jej vtable **ten sam
   indeks** co w bazie, ale wskazuje na nadpisaną implementację.
2. Każdy obiekt na stercie ma w nagłówku wskaźnik do swojej klasy (a stąd do vtable).
3. `invokevirtual` bierze referencję, idzie do nagłówka → klasy → vtable, wybiera wpis pod znanym (stałym)
   indeksem i skacze tam. Stąd nazwa **dynamic / virtual dispatch** — adres zależy od **rzeczywistego**
   typu obiektu, nie od typu referencji.
4. `invokeinterface` jest podobne, ale bez gwarancji stałego indeksu (klasa implementuje wiele interfejsów),
   więc bywa o włos droższe — JVM cache'uje ostatnie trafienie.

JIT to często **devirtualizuje**: jeśli w danym miejscu obserwuje tylko jeden typ (monomorphic call site),
wkleja metodę inline i pomija vtable. Metody `private`, `static`, `final` i konstruktory **nie są
wirtualne** (`invokespecial`/`invokestatic`) — cel znany w kompilacji, brak narzutu dispatch.
Dlatego `final` na metodzie bywa też mikrooptymalizacją (choć dziś JIT i tak to wyłapuje).

### Pola i metody statyczne NIE są polimorficzne

Pola są wiązane **statycznie** (po typie referencji), a metody `static` są *hidden*, nie *overridden*.
`A a = new B(); a.pole` i wywołania statyczne idą po typie deklarowanym — to częsta pułapka.

## 3. Dlaczego / kiedy — kompozycja vs dziedziczenie

### „Program to an interface, not an implementation"

Zależ od **typu abstrakcyjnego**, nie konkretnego: `List<X> l = new ArrayList<>();`, parametr `Collection`
a nie `ArrayList`. Wtedy podmieniasz implementację (inny algorytm, mock w teście, dekorator) bez dotykania
klientów. To podstawa luźnego sprzężenia i [[wiedza/03-frameworki/dependency-injection|DI]].

### „Favor composition over inheritance"

- **Fragile base class problem (krucha klasa bazowa)** — podklasa zależy od **wewnętrznych** szczegółów
  bazy (które metody woła która). Zmiana w bazie — nawet zgodna z jej kontraktem — cicho psuje podklasę.
  Klasyczny przykład: `HashSet` z nadpisanym `add`/`addAll`, gdzie `addAll` wewnętrznie woła `add` →
  podwójne liczenie. Dziedziczenie łamie hermetyzację: podklasa widzi i zależy od *jak*, nie tylko *co*.
- **Inheritance is white-box, composition is black-box.** Przy kompozycji trzymasz innego obiekta w polu
  i **delegujesz** — zależysz tylko od jego publicznego kontraktu, nie od bebechów.
- Kompozycja daje elastyczność w runtime (możesz podmienić komponent), unika sztywnej hierarchii i pułapek
  z `protected`. Dziedziczenia używaj, gdy **naprawdę** zachodzi is-a i baza jest *projektowana* pod
  rozszerzanie (udokumentowana, albo zamknięta przez `final`/`sealed`).

```java
// Zamiast: class Stos extends ArrayList<T>  (dziedziczy 30 metod, które łamią inwariant stosu)
final class Stos<T> {
    private final List<T> dane = new ArrayList<>();   // kompozycja + delegacja
    void push(T t) { dane.add(t); }
    T pop() { return dane.remove(dane.size() - 1); }
}
```

### Diamond problem i jak Java go rozwiązuje

Wielodziedziczenie **stanu** jest zakazane (klasa ma jednego `extends`), więc klasycznego rombu pól nie ma.
Ale od metod `default` w interfejsach romb **zachowania** jest możliwy: dwa interfejsy z `default` metodą
o tej samej sygnaturze. Java wymusza wtedy **jawne rozstrzygnięcie** — kod się nie skompiluje, dopóki nie
przesłonisz metody i nie wskażesz źródła przez `Interfejs.super.metoda()`:

```java
interface A { default String hi() { return "A"; } }
interface B { default String hi() { return "B"; } }
class C implements A, B {
    @Override public String hi() { return A.super.hi(); }   // jawny wybór — bez tego błąd kompilacji
}
```

Reguły rozstrzygania: konkretna metoda klasy > metoda nadtypu bardziej szczegółowego > konflikt → musisz
nadpisać ręcznie. Brak „domyślnego zwycięzcy" — to świadoma decyzja, by uniknąć cichych niejednoznaczności C++.

### Klasy zagnieżdżone

| typ | trzyma ref. do enclosing? | dostęp | kiedy |
|---|:---:|---|---|
| **static nested** | nie | jak każda klasa, ale „przyklejona" nazwą | logiczne grupowanie, np. `Map.Entry` |
| **inner** (niestatyczna) | **tak** | widzi pola enclosing instance | gdy obiekt logicznie należy do instancji zewnętrznej (np. iterator) |
| **local** | tak (w metodzie instancyjnej) | widzi `final`/effectively final zmienne lokalne | jednorazowa klasa w jednej metodzie |
| **anonymous** | tak | jw., bez nazwy, w miejscu użycia | jednorazowa implementacja interfejsu/klasy (przed lambdami) |

```java
class Zewn {
    private int x = 1;
    static class Stat { /* NIE widzi x */ }
    class Inner { int wez() { return x; } }     // widzi x; trzyma ukrytą ref Zewn.this
    void m() {
        int local = 5;                          // effectively final
        Runnable r = new Runnable() {           // anonymous
            public void run() { System.out.println(x + local); }
        };
    }
}
```

**Pułapka pamięciowa:** `inner`, `local` i `anonymous` trzymają **ukrytą referencję do obiektu zewnętrznego**
(`Zewn.this`). Jeśli taka instancja przeżyje (np. listener, wątek), trzyma przy życiu cały enclosing object →
**memory leak**. Gdy nie potrzebujesz stanu zewnętrznego — rób `static nested` (albo lambdę). Anonymous classes
w roli prostych funkcji wyparte przez lambdy, ale lambda **nie** ma własnego `this` (to enclosing `this`),
a anonymous class — ma.

### `final`, `this`, `super`

- **`final` pole** — przypisywane raz (w deklaracji/konstruktorze), klucz do **immutability** i
  bezpieczeństwa wątkowego (gwarancje publikacji w JMM — [[wiedza/02-jvm/model-pamieci]]).
- **`final` metoda** — nie do przesłonięcia (zamyka punkt rozszerzenia; `invokevirtual` może być
  zdevirtualizowane).
- **`final` klasa** — nie do dziedziczenia (np. `String`); chroni inwarianty i bezpieczeństwo.
- **`this`** — referencja do bieżącej instancji; `this(...)` woła inny konstruktor tej samej klasy.
- **`super`** — dostęp do składowych nadklasy; `super(...)` woła konstruktor bazy (musi być **pierwszą**
  instrukcją konstruktora); `super.metoda()` woła wersję z nadklasy mimo override.

## Przykład w praktyce
Warstwa repozytorium w Spring Boot: kontroler zależy od **interfejsu** `UserRepository`, nie od
`JpaUserRepository` (program to an interface) — w teście wstrzykujesz mock/in-memory, na produkcji JPA.
Serwis składa się z wstrzykniętych zależności (**kompozycja**, nie dziedziczenie po „bazowym serwisie").
Gdy wołasz `repo.save(u)`, JVM przez `invokeinterface` + dynamic dispatch trafia w konkretną implementację —
to ten sam mechanizm polimorfizmu, który napędza całe DI i AOP-owe proxy.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię każdy z 4 filarów na konkretnym przykładzie kodu (nie ogólnikowo).
- [ ] Wiem dokładnie, co widać przy `private`/package-private/`protected`/`public`.
- [ ] Rozróżniam override (runtime, dynamic dispatch) i overload (compile time) i umiem to obronić.
- [ ] Wyjaśnię vtable i działanie `invokevirtual` od podstaw.
- [ ] Uzasadnię „favor composition over inheritance" przez fragile base class problem.
- [ ] Wiem, jak Java rozwiązuje diamond problem dla interfejsów (`X.super.m()`).
- [ ] Znam 4 rodzaje klas zagnieżdżonych i wyciek pamięci z inner/anonymous.

### 🔲 Black-box check
- [ ] Co robi bytecode `invokevirtual` w runtime krok po kroku (header → klasa → vtable → skok)?
- [ ] Czemu `static`/`private`/`final` metody nie są wirtualne i co to zmienia (`invokespecial`)?
- [ ] Czemu pola i metody statyczne NIE są polimorficzne (binding po typie referencji)?
- [ ] Po co naprawdę powstały `default` methods (interface evolution: `stream()` w `Collection`)?
- [ ] Jak inner class fizycznie trzyma `Outer.this` i jak to powoduje memory leak?
- [ ] Czym różni się `this` w lambdzie vs w anonymous class?

### 🎤 Pytania rekrutacyjne
- [ ] „Czym różni się przesłanianie od przeciążania i kiedy następuje wiązanie?"
- [ ] „Klasa abstrakcyjna vs interfejs — kiedy które?"
- [ ] „Dlaczego favor composition over inheritance?" (fragile base class)
- [ ] „Czy Java ma wielodziedziczenie i jak rozwiązuje diamond problem?"
- [ ] „Co dokładnie widzi `protected` z innego pakietu?"
- [ ] „Po co `default` methods skoro mamy klasy abstrakcyjne?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/01-jezyk.csv`.*
```
4 filary OOP?;Encapsulation, inheritance, polymorphism, abstraction.
Encapsulation to?;Ukrycie reprezentacji za publicznym API i utrzymanie inwariantów — nie "gettery na wszystko".
Klasa vs interfejs — co modeluje?;Klasa = co to JEST (stan + impl); interfejs = co to POTRAFI (kontrakt, can-do).
Po co default methods (Java 8)?;Dodawanie metod do interfejsu bez psucia istniejących implementacji (interface evolution, np. stream() w Collection).
Po co private methods w interfejsie (Java 9)?;Wydzielenie wspólnej logiki z metod default/static bez wystawiania jej w API.
Kiedy abstract class zamiast interfejsu?;Gdy potrzebujesz wspólnego STANU (pól), częściowej implementacji lub konstruktora; inaczej domyślnie interfejs.
Co widzi protected?;Ta klasa, cały pakiet oraz podklasy (w innym pakiecie tylko na obiektach swojego typu).
Co widzi package-private (brak modyfikatora)?;Tylko klasy z tego samego pakietu.
Override vs overload — kiedy wiązanie?;Override = runtime (dynamic dispatch po typie obiektu); overload = compile time (po typie statycznym argumentów).
Jak invokevirtual wybiera metodę?;Z referencji → nagłówek obiektu → klasa → vtable → wpis pod stałym indeksem → skok do implementacji rzeczywistego typu.
Które metody NIE są wirtualne?;static, private, final i konstruktory (invokestatic/invokespecial) — cel znany w kompilacji.
Czy pola są polimorficzne?;Nie — pola i metody static wiążą się statycznie po typie referencji, nie obiektu.
Fragile base class problem?;Podklasa zależy od wewnętrznych szczegółów bazy; zgodna zmiana bazy cicho psuje podklasę → stąd favor composition over inheritance.
Jak Java rozwiązuje diamond problem?;Brak wielodziedziczenia stanu; przy konflikcie default methods wymusza ręczny override z wyborem przez Interfejs.super.metoda().
Inner class a memory leak?;Trzyma ukrytą referencję do Outer.this; jeśli przeżyje (listener/wątek), blokuje GC obiektu zewnętrznego — używaj static nested gdy stan niepotrzebny.
this w lambdzie vs anonymous class?;W lambdzie this = enclosing instance; anonymous class ma własny this.
final na metodzie/klasie?;final metoda = brak override; final klasa = brak dziedziczenia (np. String) — chroni inwarianty, umożliwia devirtualizację.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/00-fundament/jdk-jre-jvm]] · [[wiedza/01-jezyk/generics]] · [[wiedza/01-jezyk/rekordy-sealed]] · [[wiedza/02-jvm/model-pamieci]] · [[wiedza/03-frameworki/dependency-injection]]
