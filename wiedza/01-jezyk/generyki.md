---
temat: "Generyki (Generics) w Javie"
faza: 1
status: nieopanowany
priorytet: 🔴
tags: [java, jezyk, generics, type-system]
powiazane: ["[[wiedza/00-fundament/jdk-jre-jvm]]", "[[wiedza/01-jezyk/kolekcje]]", "[[wiedza/02-jvm/bytecode]]", "[[wiedza/01-jezyk/varargs]]"]
---

# Generyki (Generics) w Javie

> **TL;DR:** Generyki to **parametryzacja typem** dająca **type safety w czasie kompilacji** (koniec z ręcznym
> rzutowaniem i `ClassCastException` w runtime). Kluczowy black-box: **type erasure** — generyki istnieją tylko
> dla kompilatora, w runtime są **wymazane** do typów surowych/granic. Stąd `List<String>` i `List<Integer>` to
> w bytecode ta sama klasa, nie zrobisz `new T[]` ani `instanceof List<String>`. **PECS** (Producer Extends,
> Consumer Super) rządzi wildcardami, a `List<String>` **nie jest** podtypem `List<Object>` (inwariancja).

## 1. Co — definicja i API

**Generyki** pozwalają parametryzować klasy, interfejsy i metody **typem** podawanym dopiero w miejscu użycia.
Zamiast `Object` + rzutowanie, kompilator zna konkretny typ i sam go pilnuje.

**Po co — dwie korzyści:**
```java
// PRZED generykami (raw type) — kompiluje się, ale wybucha w runtime:
List list = new ArrayList();
list.add("tekst");
Integer x = (Integer) list.get(0);   // ClassCastException dopiero przy uruchomieniu!

// Z generykami — błąd wychwycony przez KOMPILATOR, brak rzutowania:
List<String> list = new ArrayList<>();
list.add("tekst");
String s = list.get(0);              // bez (String) — kompilator wie, że to String
// list.add(42);                     // BŁĄD KOMPILACJI — nigdy nie dotrze do runtime
```
Dwa zyski: **type safety w czasie kompilacji** (błąd typu = błąd kompilacji, nie wyjątek na produkcji)
oraz **koniec z rzutowaniem** (czytelniejszy, bezpieczniejszy kod).

**Klasy/interfejsy generyczne** — parametr typu w nawiasach `<>`:
```java
public class Box<T> {                 // T = parametr typu
    private T value;
    public void set(T value) { this.value = value; }
    public T get() { return value; }
}
public interface Repository<T, ID> {  // wiele parametrów
    T findById(ID id);
}
Box<String> b = new Box<>();          // diamond operator <> — inferencja od Javy 7
```

**Konwencja nazw parametrów typu** (czysto umowna, kompilator nie wie o znaczeniu):
- `T` — Type, `E` — Element (kolekcje), `K` — Key, `V` — Value (mapy), `N` — Number, `R` — Result, `S, U`...

**Metody generyczne** — własny parametr typu przed typem zwracanym:
```java
public static <T> T firstOrNull(List<T> list) {
    return list.isEmpty() ? null : list.get(0);
}
String s = firstOrNull(List.of("a", "b"));   // T zinferowane jako String
```

**Bounded type parameters** (ograniczenia górne) — `extends` zawęża dopuszczalne typy i **odblokowuje API granicy**:
```java
// T musi implementować Comparable<T> — inaczej nie wywołasz compareTo:
public static <T extends Comparable<T>> T max(List<T> list) {
    T best = list.get(0);
    for (T x : list)
        if (x.compareTo(best) > 0) best = x;  // dostępne, bo T extends Comparable<T>
    return best;
}
// Wiele granic przez & (klasa, potem interfejsy): <T extends Number & Comparable<T>>
```
`extends` w bound dotyczy **i klas, i interfejsów** (zawsze `extends`, nigdy `implements`).

## 2. Jak — type erasure pod spodem

To jest najważniejszy black-box generyków. **Generyki to mechanizm CZASU KOMPILACJI.** Kompilator
używa ich do sprawdzenia typów, po czym **wymazuje** (type erasure) i wstawia rzutowania. W bytecode
generyków **prawie nie ma** — zostają jako metadane (sygnatury) dla kompilatora i refleksji, ale nie wpływają
na semantykę wykonania.

**Co robi erasure (algorytm):**
- parametr **nieograniczony** `T` → zamieniany na `Object`,
- parametr **ograniczony** `<T extends Number>` → zamieniany na **granicę** (`Number`),
- usuwa argumenty typu z typów sparametryzowanych (`List<String>` → `List`),
- wstawia **rzutowania** (casts) tam, gdzie kod czyta wartości generyczne,
- generuje **bridge methods** dla zachowania polimorfizmu.

```java
// PISZESZ:
class Box<T extends Number> {
    T value;
    T get() { return value; }
}
Box<Integer> b = new Box<>();
Integer i = b.get();

// KOMPILATOR WIDZI PO ERASURE (mniej więcej):
class Box {
    Number value;                    // T → granica Number
    Number get() { return value; }
}
Box b = new Box();
Integer i = (Integer) b.get();       // cast wstawiony PRZEZ kompilator, niewidoczny w źródle
```

**Dowód, że to jedna klasa w runtime:**
```java
List<String> a = new ArrayList<>();
List<Integer> b = new ArrayList<>();
System.out.println(a.getClass() == b.getClass());  // true! — obie to ArrayList.class
```

### Reifiable vs non-reifiable types
Typ jest **reifiable**, jeśli jego pełna informacja istnieje w **runtime** (przetrwał erasure):
`String`, `Integer`, `List` (raw), `List<?>` (unbounded), tablice typów reifiable, typy prymitywne.
Typ jest **non-reifiable**, jeśli erasure usuwa jego argumenty typu: `List<String>`, `Map<K,V>`, `T`.
Większość ograniczeń generyków wynika wprost z tego, że typy parametryzowane są **non-reifiable**.

### Konsekwencje type erasure (komplet)
- **Nie zrobisz `new T[]`** ani `new ArrayList<T>[]` — JVM w runtime nie zna `T`, a tablice muszą znać
  swój typ komponentu (są reifiable). Obejście: `(T[]) new Object[n]` + `@SuppressWarnings("unchecked")`,
  albo `Array.newInstance(clazz, n)` z przekazanym `Class<T>` (token typu).
- **Nie zrobisz `obj instanceof List<String>`** — w runtime istnieje tylko `List`, nie da się sprawdzić
  argumentu typu. Dozwolone jest tylko `obj instanceof List<?>` lub `instanceof List` (raw).
- **Brak generycznych wyjątków** — nie można `class MyEx<T> extends Exception`, bo `catch` jest mechanizmem
  runtime, a JVM nie potrafi rozróżnić `catch (MyEx<A>)` od `catch (MyEx<B>)` (oba wymazane do `MyEx`).
  Nie wolno też używać parametru typu w `catch`.
- **Brak statycznych pól typu `T`** — `T` należy do instancji, nie do klasy; pole statyczne jest wspólne
  dla wszystkich parametryzacji, więc `T` nie miałoby sensu.
- **Bridge methods (metody mostkowe)** — generowane automatycznie, by erasure nie zepsuło **nadpisywania**:
  ```java
  class Node<T> { T value; void set(T v) { this.value = v; } }
  class StringNode extends Node<String> {
      @Override void set(String v) { ... }       // sygnatura set(String)
  }
  // Po erasure Node.set ma sygnaturę set(Object). Bez mostka StringNode.set(String)
  // NIE nadpisywałby set(Object) → polimorfizm by nie działał. Kompilator dodaje w StringNode:
  //   void set(Object v) { set((String) v); }   // <-- BRIDGE METHOD (syntetyczna)
  ```
  Mostki są oznaczone flagami `synthetic` + `bridge` w bytecode; widać je np. przez `javap -p`.
- **Heap pollution & `@SafeVarargs`** — *heap pollution* to sytuacja, gdy zmienna typu parametryzowanego
  wskazuje obiekt o niezgodnym typie (możliwe, bo runtime nie pilnuje argumentów typu). Klasyczne źródło:
  **generyczne varargs**, bo `T...` to ukryta tablica `T[]` (non-reifiable):
  ```java
  @SafeVarargs                                   // obiecuję, że nie psuję heapa
  static <T> List<T> listOf(T... items) {        // bez adnotacji: unchecked/varargs warning
      return new ArrayList<>(Arrays.asList(items));
  }
  ```
  `@SafeVarargs` (na metodzie `static`/`final`/`private`/konstruktorze) wycisza ostrzeżenie i jest
  **kontraktem** programisty, że metoda nie zapisuje do tablicy varargs ani nie ujawnia jej na zewnątrz.

### Raw types (i czemu ich unikać)
**Raw type** to użycie klasy generycznej **bez** argumentów typu (`List` zamiast `List<String>`) —
istnieje wyłącznie dla **kompatybilności wstecznej** z kodem sprzed Javy 5.
```java
List raw = new ArrayList<String>();
raw.add(42);                 // kompiluje się (tylko warning) — wyłącza kontrolę typów!
List<String> s = raw;        // unchecked warning
String x = s.get(0);         // ClassCastException w runtime — bomba z opóźnionym zapłonem
```
Raw type **wyłącza całą kontrolę generyków** dla danej zmiennej (nie tylko brak argumentu — degraduje też
metody do erasure). Unikaj. Gdy typ jest nieistotny, użyj **`List<?>`** (bezpieczne) zamiast `List` (raw).

## 3. Dlaczego / kiedy — pułapki: wildcards, PECS i inwariancja

### Inwariancja: `List<String>` NIE jest podtypem `List<Object>`
Mimo że `String` jest podtypem `Object`, **typy generyczne są inwariantne**:
```java
List<String> ls = new ArrayList<>();
List<Object> lo = ls;        // BŁĄD KOMPILACJI — i słusznie!
```
**Dlaczego to musi być zakazane:** gdyby było dozwolone, mielibyśmy dziurę w typach:
```java
List<Object> lo = ls;        // gdyby przeszło...
lo.add(42);                  // dodajemy Integer do listy widzianej jako List<Object>
String s = ls.get(0);        // ...a ls to wciąż List<String> → ClassCastException!
```
Kompilator blokuje to już na przypisaniu, żeby `ls` zawsze trzymało wyłącznie `String`. (Uwaga kontrast:
**tablice są kowariantne** — `Object[] arr = new String[1]` przejdzie i da `ArrayStoreException` w runtime;
generyki wybierają bezpieczeństwo w czasie kompilacji.)

### Wildcards — jak obejść inwariancję bezpiecznie
- **Unbounded `<?>`** — „lista czegoś, nie wiem czego". Można czytać jako `Object`, **nie można dodawać**
  (poza `null`), bo nie wiadomo, jaki typ jest akceptowany.
  ```java
  void printAll(List<?> list) { for (Object o : list) System.out.println(o); }  // bierze List<cokolwiek>
  ```
- **Upper bounded `<? extends T>`** — „T lub jego podtyp". **PRODUCER**: można **czytać** jako `T`,
  **nie można dodawać** (oprócz `null`) — nie wiesz, który konkretny podtyp tam siedzi.
  ```java
  List<? extends Number> nums = new ArrayList<Integer>();  // OK, obchodzi inwariancję
  Number n = nums.get(0);     // czytanie OK (każdy element jest Number)
  // nums.add(1);             // BŁĄD — może to List<Double>, nie wolno wepchnąć Integera
  ```
- **Lower bounded `<? super T>`** — „T lub jego nadtyp". **CONSUMER**: można **dodawać** `T` i jego podtypy,
  ale czytać tylko jako `Object`.
  ```java
  List<? super Integer> sink = new ArrayList<Number>();    // OK
  sink.add(42);               // dodawanie OK (Integer pasuje do każdego nadtypu)
  Object o = sink.get(0);     // czytać można tylko jako Object
  ```

### PECS — Producer Extends, Consumer Super (konkretnie)
Reguła wyboru wildcardu: jeśli struktura **produkuje** (dajesz z niej dane) → `extends`; jeśli **konsumuje**
(wkładasz do niej dane) → `super`. Kanoniczny przykład `copy`:
```java
public static <T> void copy(List<? super T> dst, List<? extends T> src) {
    for (T item : src)        // src PRODUKUJE T → ? extends T (czytamy)
        dst.add(item);        // dst KONSUMUJE T → ? super T  (zapisujemy)
}
List<Number> dst = new ArrayList<>();
List<Integer> src = List.of(1, 2, 3);
copy(dst, src);               // działa: kopiujemy Integer-y do listy Number — niemożliwe bez wildcardów
```
Bez PECS musiałbyś mieć `List<T>` po obu stronach i `copy` nie przyjąłby `List<Integer>` → `List<Number>`.
Mnemonik z `Collections`: `Collections.copy(dest, src)`, `addAll(c, ? extends E)` itd.

**Praktyczna zasada doboru:** parametr tylko-do-odczytu → `? extends`; tylko-do-zapisu → `? super`;
i czytasz, i piszesz → konkretny typ bez wildcardu.

### Inferencja typów (type inference)
Kompilator wnioskuje argumenty typu z kontekstu (argumenty wywołania, typ docelowy przypisania):
```java
var list = List.<String>of("a");   // jawny argument typu metody (rzadko potrzebny)
Map<String, List<Integer>> m = new HashMap<>();  // diamond <> — inferencja z lewej strony
List<String> e = Collections.emptyList();        // target typing — T zinferowane jako String
```
Od Javy 10 `var` współpracuje z generykami, ale **nie zastępuje** ich — `var x = new ArrayList<>()` da
`ArrayList<Object>` (uważaj na utratę typu).

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)
**Warstwa repozytorium / serwisu.** Generyczne `Repository<T, ID>` w Spring Data (`JpaRepository<User, Long>`)
to klasa generyczna z bounded parametrami. Pisząc własny generyczny serwis trafisz na PECS:
```java
public <T> void publishAll(EventBus bus, Collection<? extends T> events) { ... }  // events produkują
```
Type erasure ugryzie Cię w **deserializacji** (Jackson) — `objectMapper.readValue(json, List.class)` zgubi
typ elementu, więc potrzebujesz `TypeReference<List<User>>` (reifikacja typu poprzez anonimową podklasę,
która zachowuje sygnaturę generyczną w metadanych — obejście erasure). Podobnie token `Class<T>` przekazujesz,
gdy musisz w runtime tworzyć instancje/tablice typu generycznego.

---

## ✅ Kryteria opanowania
*Temat DOMKNIĘTY, gdy odpowiesz na wszystko BEZ zaglądania.*

- [ ] Wyjaśnię, że generyki dają type safety w czasie kompilacji i eliminują rzutowanie.
- [ ] Wytłumaczę type erasure: co dokładnie się wymazuje (nieograniczone → Object, ograniczone → granica) i czemu.
- [ ] Wymienię konsekwencje erasure: brak `new T[]`, brak `instanceof List<String>`, brak generycznych wyjątków, bridge methods.
- [ ] Zastosuję PECS i uzasadnię wybór `extends`/`super` na przykładzie `copy`.
- [ ] Wyjaśnię inwariancję (`List<String>` ≠ podtyp `List<Object>`) kontrprzykładem z `ClassCastException`.
- [ ] Odróżnię reifiable od non-reifiable i powiem, co z tego wynika.
- [ ] Napiszę metodę generyczną z bounded parameter z głowy.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co fizycznie zostaje z `List<String>` w bytecode? (raw `List` + casty + sygnatura w metadanych)
- [ ] Czemu `new ArrayList<String>().getClass() == new ArrayList<Integer>().getClass()` daje `true`?
- [ ] Czym jest bridge method i jaki problem rozwiązuje? (nadpisywanie po erasure)
- [ ] Czym jest heap pollution i kiedy `@SafeVarargs` jest uprawnione?
- [ ] Czemu nie ma generycznych wyjątków ani `new T[]`?
- [ ] Reifiable vs non-reifiable — które typy są które?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Co to type erasure i jakie ma konsekwencje?"
- [ ] „Wyjaśnij PECS." (Producer Extends, Consumer Super — z przykładem `copy`)
- [ ] „Czemu `List<String>` nie jest `List<Object>`, skoro `String` jest `Object`?"
- [ ] „Różnica `<?>` vs `<? extends T>` vs `<? super T>` — co można czytać/zapisywać?"
- [ ] „Czemu nie da się `new T[]`?" / „Czemu raw types są złe?"
- [ ] „Tablice są kowariantne, generyki inwariantne — czemu i jaki to ma skutek?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/01-jezyk.csv`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Dwie główne korzyści generyków?;Type safety w czasie kompilacji (błąd typu = błąd kompilacji) oraz eliminacja ręcznego rzutowania.
Co oznaczają T, E, K, V?;Type, Element, Key, Value — umowne nazwy parametrów typu (kompilator nie zna ich znaczenia).
Co to bounded type parameter?;Ograniczenie parametru, np. <T extends Comparable<T>> — zawęża typy i odblokowuje API granicy; zawsze extends (też dla interfejsów).
Co to type erasure?;Wymazanie informacji o generykach po kompilacji: nieograniczone T→Object, ograniczone→granica, usunięcie argumentów typu, wstawienie castów i bridge methods.
Czy List<String> i List<Integer> to ta sama klasa w runtime?;Tak — getClass() zwraca to samo (ArrayList.class), bo argumenty typu są wymazane przez erasure.
Czemu nie da się new T[]?;Bo erasure usuwa T w runtime, a tablice muszą znać swój typ komponentu (są reifiable); obejście: (T[]) new Object[n] lub Array.newInstance.
Czemu nie ma instanceof List<String>?;W runtime istnieje tylko raw List (erasure) — argumentu typu nie da się sprawdzić; dozwolone tylko instanceof List<?>.
Czym jest bridge method?;Syntetyczna metoda generowana przez kompilator, by nadpisywanie działało po erasure (np. set(Object) delegujący do set(String)).
Co to heap pollution i po co @SafeVarargs?;Heap pollution: zmienna typu parametryzowanego wskazuje obiekt niezgodnego typu; @SafeVarargs to kontrakt, że generyczne varargs nie psują heapa, wyciszający warning.
Czemu unikać raw types?;Wyłączają całą kontrolę generyków (degradacja do erasure) — przesuwają błędy typów z kompilacji do runtime (ClassCastException); istnieją tylko dla kompatybilności wstecznej.
Reifiable vs non-reifiable type?;Reifiable: pełna info istnieje w runtime (String, List, List<?>, tablice prymitywów). Non-reifiable: erasure usuwa argumenty typu (List<String>, T, Map<K,V>).
PECS — co to?;Producer Extends, Consumer Super: gdy struktura produkuje dane → <? extends T>; gdy konsumuje → <? super T>.
Co można robić z List<? extends T>?;Czytać elementy jako T (producer); NIE wolno dodawać (poza null) — nie wiadomo, który podtyp.
Co można robić z List<? super T>?;Dodawać T i jego podtypy (consumer); czytać tylko jako Object.
Czemu List<String> nie jest podtypem List<Object>?;Generyki są inwariantne — gdyby były podtypem, można by przez List<Object> wstawić nie-String i dostać ClassCastException przy odczycie.
Czemu nie ma generycznych wyjątków?;catch działa w runtime, a po erasure JVM nie odróżni MyEx<A> od MyEx<B> — nie można złapać typu parametryzowanego.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/00-fundament/jdk-jre-jvm]] · [[wiedza/01-jezyk/kolekcje]] · [[wiedza/02-jvm/bytecode]] · [[wiedza/01-jezyk/varargs]]
