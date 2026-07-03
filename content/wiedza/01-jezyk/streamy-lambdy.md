---
temat: "Lambdy i Stream API (programowanie funkcyjne)"
faza: 1
status: nieopanowany
priorytet: 🔴
tags: [java, jezyk, funkcyjne, lambda, stream]
powiazane: ["[[wiedza/01-jezyk/optional]]", "[[wiedza/00-fundament/jdk-jre-jvm]]", "[[wiedza/03-wspolbieznosc/executors]]"]
---

# Lambdy i Stream API (programowanie funkcyjne)

> **TL;DR:** **Lambda** to zwięzła implementacja **interfejsu funkcyjnego** (jedna metoda abstrakcyjna),
> realizowana pod spodem przez **`invokedynamic` + `LambdaMetafactory`** — *nie* przez anonimową klasę.
> **Stream API** to **leniwy** pipeline (źródło → operacje pośrednie → operacja terminalna): nic się nie liczy,
> dopóki nie zawołasz operacji terminalnej, a wtedy elementy płyną **pojedynczo przez cały pipeline** (fuzja), nie warstwa po warstwie.
> Stream jest **jednorazowy**.

## 1. Co — interfejsy funkcyjne, lambdy, referencje do metod, Stream API

### Interfejs funkcyjny

**Interfejs funkcyjny** = interfejs z **dokładnie jedną metodą abstrakcyjną** (SAM — *Single Abstract Method*).
Może mieć dowolnie wiele metod `default`/`static` — one się nie liczą. Adnotacja `@FunctionalInterface` jest
opcjonalna, ale to **kontrakt kompilatora**: gdy dodasz drugą metodę abstrakcyjną, kod się nie skompiluje.

```java
@FunctionalInterface
interface Walidator {
    boolean waliduj(String s);          // jedyna metoda abstrakcyjna
    default Walidator i(Walidator inny) // default — dozwolone
        { return s -> this.waliduj(s) && inny.waliduj(s); }
}
Walidator niepusty = s -> !s.isBlank(); // lambda = implementacja SAM
```

**Wbudowane interfejsy funkcyjne** (`java.util.function`) — uczysz się ich raz, używasz wszędzie:

| Interfejs | Sygnatura SAM | Sens | Przykład |
|---|---|---|---|
| `Function<T,R>` | `R apply(T)` | T → R | `String::length` |
| `BiFunction<T,U,R>` | `R apply(T,U)` | (T,U) → R | `(a,b) -> a+b` |
| `Supplier<T>` | `T get()` | () → T (dostawca) | `() -> new ArrayList<>()` |
| `Consumer<T>` | `void accept(T)` | T → () (efekt uboczny) | `System.out::println` |
| `Predicate<T>` | `boolean test(T)` | T → boolean | `s -> s.isBlank()` |
| `UnaryOperator<T>` | `T apply(T)` | T → T (= `Function<T,T>`) | `String::trim` |
| `BinaryOperator<T>` | `T apply(T,T)` | (T,T) → T (= `BiFunction<T,T,T>`) | `Integer::sum` |

Są też warianty prymitywne (`IntFunction`, `ToIntFunction`, `IntPredicate`, `IntUnaryOperator`…) — patrz sekcja o boxingu.

### Lambdy — składnia i capturing

```java
(int a, int b) -> a + b      // pełna
(a, b) -> a + b              // typy wywnioskowane
x -> x * 2                   // jeden parametr, bez nawiasów
() -> "stała"                // zero parametrów
s -> { log(s); return s.length(); } // blok z return
```

**Capturing — `effectively final`.** Lambda może czytać zmienne lokalne z otaczającego zakresu, ale tylko
**finalne lub *effectively final*** (takie, którym po inicjalizacji nigdy nie przypisano ponownie wartości).

```java
int podatek = 23;                  // effectively final — OK
List.of(100, 200).forEach(c -> System.out.println(c * podatek / 100));
// podatek = 8;  // <-- gdyby to odkomentować, lambda wyżej przestaje się kompilować
```

**Czemu nie można modyfikować zmiennej lokalnej?** Bo zmienne lokalne żyją **na stosie wątku** i giną z metodą,
a lambda może zostać wykonana **później i w innym wątku**. Java rozwiązuje to przez **capture by value**: wartość
jest *kopiowana* do lambdy w chwili utworzenia. Gdyby pozwolono modyfikować zmienną, mielibyśmy dwie rozjechane
kopie (na stosie i w lambdzie) — twórcy języka wybrali zakaz zamiast iluzji współdzielenia. To różni się od pól
obiektu/`static` (żyją na stercie), które lambda widzi przez referencję `this` i **może** modyfikować.
Obejście „mutowalności”: tablica jednoelementowa lub `AtomicInteger` (ale to zwykle zapach — w streamach używaj `reduce`).

### Referencje do metod — 4 rodzaje

Skrót dla lambdy, która tylko woła istniejącą metodę. `::` zamiast `->`.

```java
// 1. STATYCZNA:                 Type::staticMethod
Function<String,Integer> a = Integer::parseInt;     // s -> Integer.parseInt(s)

// 2. INSTANCJI KONKRETNEGO obiektu:   obiekt::method   (bound)
String prefix = "ID-";
Predicate<String> b = prefix::startsWith;           // s -> prefix.startsWith(s)

// 3. INSTANCJI DOWOLNEGO obiektu danego typu:  Type::instanceMethod  (unbound)
Function<String,Integer> c = String::length;        // s -> s.length()
//   pierwszy argument lambdy staje się odbiorcą metody!

// 4. KONSTRUKTOR:               Type::new
Supplier<List<String>> d = ArrayList::new;          // () -> new ArrayList<>()
Function<Integer,int[]> e = int[]::new;             // n -> new int[n]
```

Niuans: rodzaj 2 (bound) wiąże *konkretny* obiekt z domknięcia; rodzaj 3 (unbound) bierze obiekt-odbiorcę z
**pierwszego parametru** funkcji — to najczęstsze źródło konfuzji `obiekt::m` vs `Typ::m`.

### Stream API — anatomia pipeline

```
       ŹRÓDŁO              OPERACJE POŚREDNIE (lazy)            TERMINALNA (uruchamia)
   collection.stream() → .filter(...) .map(...) .sorted() → .collect(...) / .forEach(...)
```

- **Źródła:** `coll.stream()`, `Arrays.stream(arr)`, `Stream.of(...)`, `Stream.iterate/generate` (nieskończone),
  `IntStream.range(0,n)`, `Files.lines(path)`, `pattern.splitAsStream(...)`.
- **Operacje pośrednie (intermediate)** — zwracają nowy `Stream`, są **leniwe**:
  `map`, `filter`, `flatMap`, `sorted`, `distinct`, `limit`, `skip`, `peek`.
- **Operacje terminalne (terminal)** — zwracają nie-stream (wartość/`void`/kolekcję) i **uruchamiają** pipeline:
  `collect`, `forEach`, `reduce`, `count`, `findFirst`, `findAny`, `anyMatch`/`allMatch`/`noneMatch`, `min`/`max`, `toList()`.

## 2. Jak — leniwość i `invokedynamic` pod spodem

### Lambda to NIE anonimowa klasa (odczarowanie `invokedynamic`)

Mit: „lambda to cukier na anonimową klasę”. **Nieprawda.** Anonimowa klasa generuje osobny plik `.class`
(`Outer$1.class`) ładowany przy starcie; tysiąc lambd = tysiąc klas = wolniejszy start i więcej metaspace.

Co robi `javac`:
1. Ciało lambdy trafia do **prywatnej, syntetycznej metody** w klasie, w której lambda powstała
   (np. `lambda$main$0`). To zwykła metoda — bez nowej klasy.
2. W miejscu lambdy `javac` wstawia instrukcję **`invokedynamic`** z *bootstrap methodem* wskazującym na
   `LambdaMetafactory.metafactory(...)`.

Co dzieje się w **runtime** (pierwsze wykonanie tego `invokedynamic`):
3. JVM woła bootstrap → `LambdaMetafactory` **dynamicznie generuje** w pamięci klasę implementującą interfejs
   funkcyjny (np. `Function`), której metoda `apply` deleguje do `lambda$main$0`. Tworzy `CallSite`.
4. `CallSite` jest **zapinany na stałe** (linkowany raz). Kolejne trafienia w to `invokedynamic` to już zwykłe,
   szybkie wywołanie — JIT je inline’uje. Lambda **nie capturująca** żadnej zmiennej jest dodatkowo
   **cache’owana jako singleton** (jeden obiekt na cały program).

Dlaczego tak? **Odroczenie decyzji o kształcie klasy do runtime.** Strategię implementacji lambdy (anonimowa
klasa? metoda + indy? coś przyszłego?) wybiera biblioteka (`LambdaMetafactory`), nie zamrożony bytecode — JDK może
to optymalizować bez rekompilacji Twojego kodu. To jest sedno: `invokedynamic` to „pusty slot”, który mówi JVM
*„dogadaj linkowanie w czasie wykonania”*. Zobacz: `javap -c -p TwojaKlasa` — znajdziesz `invokedynamic #x, 0`
oraz `private static ... lambda$...`, ale **żadnego** `$1` dla lambdy.

### Leniwość i fuzja operacji (odczarowanie „element po elemencie”)

Operacje pośrednie **nic nie wykonują** — budują tylko opis pipeline (łańcuch `Sink`-ów). Praca rusza dopiero w
operacji terminalnej. Co kluczowe: stream **nie materializuje** kolekcji pośrednich. Nie ma „najpierw zmapuj całą
listę, potem przefiltruj całą listę”. Zamiast tego **każdy element jest przepychany przez cały łańcuch**
(`filter`→`map`→…), zanim pobrany zostanie następny — to **fuzja operacji** (operation fusion / loop fusion).

```java
List.of("ala","bok","cep","dom").stream()
    .peek(s -> System.out.println("filter? " + s))
    .filter(s -> s.compareTo("c") >= 0)
    .peek(s -> System.out.println("  map: " + s))
    .map(String::toUpperCase)
    .findFirst();   // terminalna
// Wypisze (przeplot dowodzący fuzji, NIE warstwa po warstwie):
// filter? ala
// filter? bok
// filter? cep
//   map: cep
// (KONIEC — findFirst short-circuit, "dom" nigdy nie odwiedzone)
```

Konsekwencje leniwości:
- **Short-circuiting:** `findFirst`, `anyMatch`, `limit` mogą zatrzymać pipeline, nie odwiedzając reszty (działa
  nawet na **nieskończonych** streamach: `Stream.iterate(1, n->n+1).filter(...).findFirst()`).
- **Brak operacji terminalnej = brak działania.** `list.stream().map(this::sideEffect);` (bez terminalnej) **nic
  nie robi** — częsty bug. `peek` też jest leniwy i bywa pomijany przez optymalizacje (np. gdy `count()` nie
  potrzebuje elementów) — używaj go tylko do debugowania.

### `collect` i `Collectors`

`collect(Collector)` to terminalna „mutowalna redukcja” — akumuluje do kontenera (supplier/accumulator/combiner/finisher).
Najczęstsze fabryki z `Collectors`:

```java
.collect(Collectors.toList());            // (modyfikowalna) — albo .toList() (Java 16+, niemodyfikowalna)
.collect(Collectors.toSet());
.collect(Collectors.toMap(User::id, u -> u)); // UWAGA: duplikat klucza → wyjątek; podaj merge function
.collect(Collectors.groupingBy(User::dept)); // Map<Dept, List<User>>
.collect(Collectors.groupingBy(User::dept, Collectors.counting())); // downstream collector
.collect(Collectors.partitioningBy(User::active)); // Map<Boolean,List<User>> — zawsze 2 klucze
.collect(Collectors.joining(", ", "[", "]"));      // String
```

### `reduce` — identity / accumulator / combiner

```java
int suma = nums.stream().reduce(0, Integer::sum);
//                              ^id  ^accumulator (T,T)->T
// 3-argumentowa forma — gdy typ wyniku ≠ typ elementu, i dla parallel:
int dlugosci = words.stream().reduce(
    0,                         // identity (element neutralny)
    (acc, w) -> acc + w.length(),   // accumulator: (U, T) -> U
    Integer::sum);             // combiner: (U, U) -> U — łączy wyniki cząstkowe wątków
```

- **identity** musi być elementem neutralnym: `combiner(identity, x) == x`. Inaczej parallel da zły wynik.
- **combiner** używany **tylko w parallel** (i musi być zgodny z accumulatorem) — łączy częściowe redukcje.

### Streamy prymitywne i boxing

`Stream<Integer>` pudełkuje (**boxing**) każdy `int` w obiekt `Integer` — narzut pamięci i GC. Dlatego są
wyspecjalizowane **`IntStream`, `LongStream`, `DoubleStream`** operujące na prymitywach:

```java
int suma = users.stream().mapToInt(User::age).sum();      // bez boxingu
IntSummaryStatistics st = IntStream.rangeClosed(1,100).summaryStatistics(); // sum/avg/min/max naraz
double avg = st.getAverage();
// powrót do obiektów: .boxed() → Stream<Integer>; .mapToObj(...) → Stream<T>
```

Dają też `sum()`, `average()`, `range()`/`rangeClosed()`, `summaryStatistics()` — których `Stream<T>` nie ma.

### Stream równoległy (`parallel`) — co pod spodem

`.parallelStream()` / `.parallel()` dzieli źródło **Spliteratorem** na części, przetwarza je w
**`ForkJoinPool.commonPool()`** (domyślnie `liczba_rdzeni - 1` wątków!) i łączy wyniki (`combiner`).
Patrz: [[wiedza/03-wspolbieznosc/executors]].

## 3. Dlaczego / kiedy — parallel, pułapki, jednorazowość

### Kiedy `parallel` pomaga, a kiedy szkodzi

Parallel **nie jest darmowym przyspieszeniem**. Pomaga, gdy:
- **Dużo danych** (rzędu 10k+ elementów) i **kosztowna, niezależna** praca na elemencie (CPU-bound).
- Źródło **dobrze się dzieli** (`ArrayList`, tablica, `IntStream.range` — split O(1)); a NIE `LinkedList`,
  `Stream.iterate`, źródła I/O (dzielą się słabo lub wcale).
- Operacje są **bezstanowe i asocjacyjne** (`reduce`/`collect` z poprawnym combinerem).

Szkodzi / jest groźny, gdy:
- **Mało danych lub tania operacja** — koszt fork/join + scalania > zysk; sekwencyjny szybszy.
- **Współdzielony `commonPool`** — jeden „ciężki” parallel stream **głodzi cały proces** (inne parallel streamy,
  `CompletableFuture` bez własnego pool-a). W aplikacjach serwerowych to realny incydent.
- **Zadania blokujące** (I/O, `sleep`) w `commonPool` — wątków jest tyle co rdzeni, blokada zatka pulę.
- **Side-effecty** w lambdach — `forEach` w parallel nie gwarantuje kolejności, a zapis do współdzielonej,
  niesynchronizowanej kolekcji (`ArrayList`) to **race condition / `ConcurrentModificationException` / utrata danych**.
  Redukuj poprawnie (`collect`/`reduce`), nie mutuj zewnętrznego stanu.
- **`forEach` vs `forEachOrdered`** — pierwszy nie szanuje kolejności napotkania w parallel.

Reguła: domyślnie **sekwencyjnie**; parallel włączaj świadomie, na własnym `ForkJoinPool`, **po zmierzeniu** (JMH),
nie „bo szybciej”.

### Stream jest jednorazowy

Po wywołaniu operacji terminalnej (lub *ponownym* startcie) stream jest „zużyty”:

```java
Stream<String> s = list.stream();
s.forEach(System.out::println);
s.count();   // IllegalStateException: stream has already been operated upon or closed
```

Potrzebujesz wielu przejść → trzymaj **kolekcję** (źródło) albo `Supplier<Stream<T>>` i twórz stream na świeżo.

### `Optional` jako element pipeline

Metody short-circuit (`findFirst`, `findAny`, `min`, `max`, `reduce` jednoargumentowy) zwracają **`Optional<T>`** —
bo wynik może nie istnieć (pusty stream). Łączysz go funkcyjnie tym samym stylem (`map`/`filter`/`orElse`):

```java
String imie = users.stream()
    .filter(User::active)
    .map(User::name)
    .findFirst()           // Optional<String>
    .orElse("brak");
```

Szczegóły: [[wiedza/01-jezyk/optional]].

## Przykład w praktyce (warstwa serwisowa)

```java
// Raport: aktywni użytkownicy pogrupowani po dziale, z sumą wynagrodzeń — w jednym przebiegu.
record User(String name, String dept, boolean active, int salary) {}

Map<String, Integer> placePoDzialach = users.stream()          // źródło
    .filter(User::active)                                      // lazy
    .collect(Collectors.groupingBy(                            // terminalna
        User::dept,
        Collectors.summingInt(User::salary)));                // downstream collector

// CSV bez ręcznej pętli:
String csv = users.stream()
    .map(u -> u.name() + ";" + u.dept())
    .collect(Collectors.joining("\n", "name;dept\n", ""));
```

Gdzie spotkasz: mapowanie encji → DTO w warstwie serwisowej, agregaty raportowe, parsowanie/filtrowanie linii
pliku (`Files.lines`), budowa odpowiedzi API z list. To codzienność każdego Spring Boot serwisu.

---

## ✅ Kryteria opanowania
- [ ] Wymienię 7 wbudowanych interfejsów funkcyjnych i ich sygnatury z głowy.
- [ ] Wyjaśnię `effectively final` i **dlaczego** lambda nie modyfikuje zmiennej lokalnej (stos vs sterta, capture by value).
- [ ] Pokażę 4 rodzaje referencji do metod i różnicę bound vs unbound (`obiekt::m` vs `Typ::m`).
- [ ] Udowodnię, że lambda to `invokedynamic` + `LambdaMetafactory`, a NIE anonimowa klasa (i pokażę to w `javap`).
- [ ] Odróżnię operacje pośrednie od terminalnych i wyjaśnię, że bez terminalnej nic się nie dzieje.
- [ ] Wyjaśnię fuzję operacji (element po elemencie) i short-circuiting na nieskończonym streamie.
- [ ] Uzasadnię, kiedy `parallel` pomaga, a kiedy szkodzi (commonPool, side-effecty, dzielenie źródła).

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co generuje `javac` z lambdy? (syntetyczna metoda `lambda$...` + `invokedynamic`, nie `Outer$1.class`)
- [ ] Co robi `LambdaMetafactory` w runtime i czemu `CallSite` linkuje się tylko raz?
- [ ] Czemu lambda bez capturingu może być singletonem, a capturująca nie?
- [ ] Dlaczego stream przetwarza element po elemencie, a nie warstwa po warstwie? (fuzja, łańcuch Sink)
- [ ] Czym różni się `peek` od `map` i czemu `peek` bywa pomijany?
- [ ] Po co `IntStream` skoro jest `Stream<Integer>`? (boxing, narzut GC, `sum()`/`average()`)
- [ ] Czemu `reduce` ma 3 argumenty i kiedy combiner faktycznie się uruchamia?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Czy lambda to anonimowa klasa?” (nie — `invokedynamic` + `LambdaMetafactory`)
- [ ] „Czemu zmienna w lambdzie musi być effectively final?”
- [ ] „Czym różni się operacja pośrednia od terminalnej? Co uruchamia pipeline?”
- [ ] „Co to leniwość streamów i co daje short-circuiting?”
- [ ] „Kiedy NIE używać `parallelStream()`?” (commonPool, mało danych, side-effecty, źródło słabo dzielone)
- [ ] „Co zwraca `findFirst()` i dlaczego `Optional`?”
- [ ] „`Collectors.toMap` na duplikacie klucza — co się stanie?” (`IllegalStateException` bez merge function)

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/01-jezyk.csv`.*
```
Co to interfejs funkcyjny?;Interfejs z dokładnie jedną metodą abstrakcyjną (SAM); metody default/static się nie liczą. @FunctionalInterface to opcjonalny kontrakt kompilatora.
Function vs BiFunction vs Supplier vs Consumer vs Predicate?;Function T->R, BiFunction (T,U)->R, Supplier ()->T, Consumer T->void, Predicate T->boolean.
UnaryOperator i BinaryOperator to?;Specjalizacje: UnaryOperator<T> = Function<T,T>, BinaryOperator<T> = BiFunction<T,T,T>.
Co znaczy effectively final?;Zmienna lokalna, której po inicjalizacji nigdy nie przypisano ponownie — tylko taką (lub final) może capturować lambda.
Czemu lambda nie może modyfikować zmiennej lokalnej?;Zmienne lokalne żyją na stosie i giną z metodą; lambda przechwytuje je przez kopię (capture by value) i może działać później/w innym wątku — modyfikacja rozjechałaby kopie.
Cztery rodzaje referencji do metod?;Statyczna (Type::m), instancji konkretnego obiektu (obj::m, bound), instancji dowolnego obiektu typu (Type::m, unbound), konstruktor (Type::new).
Różnica obj::m vs Type::m?;obj::m wiąże konkretny obiekt (bound); Type::m bierze odbiorcę z pierwszego argumentu funkcji (unbound).
Czy lambda to anonimowa klasa?;Nie. javac tworzy syntetyczną metodę lambda$... i wstawia invokedynamic; w runtime LambdaMetafactory generuje implementację i linkuje CallSite raz. Brak osobnego pliku $1.class.
Po co invokedynamic dla lambd?;Odracza wybór strategii implementacji do runtime — LambdaMetafactory może ją optymalizować (np. singleton dla niecapturujących) bez rekompilacji kodu.
Operacja pośrednia vs terminalna?;Pośrednia (map/filter/sorted...) zwraca Stream i jest leniwa; terminalna (collect/reduce/forEach/findFirst) zwraca nie-stream i uruchamia pipeline.
Co to fuzja operacji w streamie?;Każdy element przepływa przez cały łańcuch operacji po kolei, zanim weźmiemy następny — nie ma kolekcji pośrednich (loop fusion).
Co robi stream bez operacji terminalnej?;Nic — operacje pośrednie są leniwe; bez terminalnej pipeline się nie wykonuje.
Co daje short-circuiting?;findFirst/anyMatch/limit mogą zatrzymać przetwarzanie bez odwiedzania reszty — działa nawet na nieskończonych streamach.
Czemu IntStream zamiast Stream<Integer>?;Unika boxingu (narzut pamięci/GC) i daje sum()/average()/range()/summaryStatistics().
Trzy argumenty reduce?;identity (element neutralny), accumulator (U,T)->U, combiner (U,U)->U łączący wyniki cząstkowe — combiner używany tylko w parallel.
groupingBy vs partitioningBy?;groupingBy -> Map<K,...> po kluczu z funkcji; partitioningBy -> Map<Boolean,List<T>> zawsze z 2 kluczami wg predykatu.
Kiedy NIE używać parallelStream?;Mało danych/tania operacja, źródło słabo dzielone (LinkedList/iterate), zadania blokujące, side-effecty na współdzielonym stanie — plus rywalizacja o wspólny ForkJoinPool.commonPool.
Dlaczego stream jest jednorazowy?;Po operacji terminalnej jest zużyty; ponowne użycie -> IllegalStateException. Trzymaj kolekcję lub Supplier<Stream>.
Co zwraca findFirst i czemu?;Optional<T> — wynik może nie istnieć (pusty stream); spinasz go dalej przez map/filter/orElse.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/01-jezyk/optional]] · [[wiedza/00-fundament/jdk-jre-jvm]] · [[wiedza/03-wspolbieznosc/executors]]
