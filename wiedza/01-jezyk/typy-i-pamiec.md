---
temat: "Typy w Javie i ich reprezentacja w pamięci"
faza: 1
status: nieopanowany
priorytet: 🔴
tags: [java, jezyk]
powiazane: ["[[wiedza/02-jvm/model-pamieci]]", "[[wiedza/01-jezyk/kolekcje]]", "[[wiedza/01-jezyk/equals-hashcode]]", "[[wiedza/02-jvm/garbage-collection]]"]
---

# Typy w Javie i ich reprezentacja w pamięci

> **TL;DR:** W Javie są **typy prymitywne** (8 wbudowanych, trzymają *wartość*) i **typy referencyjne**
> (trzymają *referencję* do obiektu na heap). Zmienna lokalna prymitywna i sama referencja żyją na **stacku**,
> obiekt — zawsze na **heap**. Java **zawsze** przekazuje argumenty **przez wartość** (kopiuje wartość prymitywu
> lub kopiuje referencję). Pułapki kluczowe: `==` vs `equals`, cache `Integer` (-128..127), NPE przy unboxingu,
> immutability `String`, kontrakt `equals/hashCode`.

## 1. Co — definicja i API

W Javie istnieją dokładnie **dwie rodziny typów**:

**Typy prymitywne (primitive types)** — 8 wbudowanych, NIE są obiektami, nie mają metod, nie mogą być `null`:

| Typ       | Rozmiar | Zakres / wartości                                  | Domyślna |
|-----------|---------|----------------------------------------------------|----------|
| `byte`    | 8 bit   | −128 .. 127                                        | `0`      |
| `short`   | 16 bit  | −32 768 .. 32 767                                  | `0`      |
| `int`     | 32 bit  | −2³¹ .. 2³¹−1 (≈ ±2,1 mld)                          | `0`      |
| `long`    | 64 bit  | −2⁶³ .. 2⁶³−1                                       | `0L`     |
| `float`   | 32 bit  | IEEE 754 pojedyncza precyzja (~7 cyfr)             | `0.0f`   |
| `double`  | 64 bit  | IEEE 754 podwójna precyzja (~15 cyfr)             | `0.0d`   |
| `char`    | 16 bit  | U+0000 .. U+FFFF (0 .. 65 535, **unsigned**)        | U+0000   |
| `boolean` | JVM-zależny* | `true` / `false`                              | `false`  |

\* JVM Spec nie definiuje rozmiaru `boolean` — w praktyce traktowany jak `int` (4 B) na stosie, a `boolean[]`
często pakowany po 1 bajcie. Nie zakładaj „1 bita".

Wszystkie typy całkowite (poza `char`) są **signed** — Java nie ma typów unsigned (od Javy 8 są tylko
*metody pomocnicze* `Integer.divideUnsigned`, `Long.parseUnsignedLong`…). `char` jest jedynym unsigned typem.

**Typy referencyjne (reference types)** — wszystko inne: klasy (`String`, `Integer`), interfejsy, tablice,
enumy, rekordy. Zmienna trzyma **referencję** (adres/uchwyt) do obiektu na heap, albo `null`.

```java
int x = 42;                 // primitive: x TRZYMA wartość 42
Integer boxed = 42;         // reference: boxed trzyma referencję do obiektu Integer
String s = "hej";           // reference: s trzyma referencję do obiektu String
int[] arr = {1, 2, 3};      // reference: tablica to obiekt na heap
```

Każdy prymityw ma **wrapper** (klasę opakowującą): `int`→`Integer`, `boolean`→`Boolean`, `char`→`Character`,
`long`→`Long` itd. Wrappery są **immutable** i potrzebne wszędzie tam, gdzie wymagane są obiekty
(np. `List<Integer>` — generyki nie przyjmują prymitywów).

## 2. Jak — co dzieje się pod spodem

### Stack vs heap

Trzeba precyzyjnie rozdzielić **trzy rzeczy**:

```
        STACK (per wątek)                         HEAP (współdzielony)
 ┌─────────────────────────────┐          ┌──────────────────────────────┐
 │ int x        = 42           │          │  Integer{ value=42 }         │
 │ Integer b ───┼──────────────┼─────────▶│                              │
 │ String  s ───┼──────────────┼────────┐ │  String{ "hej", hash=... }   │
 │ ... ramki metod (frames)    │        └▶│                              │
 └─────────────────────────────┘          └──────────────────────────────┘
```

- **Zmienna prymitywna lokalna** → wartość leży **bezpośrednio na stacku** (w ramce metody).
- **Referencja** → sama referencja leży na stacku (lub w obiekcie), ale **obiekt, na który wskazuje, jest na heap**.
- **Obiekt** → **zawsze na heap** (zarządzany przez GC — [[wiedza/02-jvm/garbage-collection]]).

Uwaga: **pola prymitywne wewnątrz obiektu** leżą na heap (bo cały obiekt jest na heap), nie na stacku.
„Prymityw na stacku" dotyczy tylko zmiennych *lokalnych*. Szczegóły layoutu — [[wiedza/02-jvm/model-pamieci]].

> Escape analysis (JIT) może zdejmować krótkożyjące obiekty z heap (scalar replacement) — to optymalizacja,
> nie kontrakt języka. Nie polegaj na niej w rozumowaniu o poprawności.

### Autoboxing / unboxing

Kompilator **automatycznie** zamienia prymityw na wrapper (boxing) i odwrotnie (unboxing):

```java
Integer boxed = 5;          // autoboxing  → Integer.valueOf(5)
int prim = boxed;           // unboxing    → boxed.intValue()
List<Integer> list = new ArrayList<>();
list.add(7);                // autoboxing 7 → Integer
int v = list.get(0);        // unboxing
```

**Boxing przechodzi przez `Integer.valueOf(int)`, NIE przez `new Integer(...)`** (konstruktor jest deprecated
od Javy 9, usunięty w nowszych). I tu wchodzi **cache**:

### Integer cache (-128 .. 127)

`Integer.valueOf(int)` zwraca **te same obiekty** dla wartości z zakresu **−128 .. 127** (klasa
`IntegerCache`). Dla wartości spoza zakresu tworzy **nowy** obiekt za każdym razem.

```java
Integer a = 100, b = 100;
System.out.println(a == b);   // true  — ten sam obiekt z cache

Integer c = 200, d = 200;
System.out.println(c == d);   // false — DWA różne obiekty na heap!
System.out.println(c.equals(d)); // true
```

Cache obejmuje też `Long`, `Short`, `Byte`, `Character` (0..127) i `Boolean`. Górną granicę `Integer` można
podnieść flagą `-XX:AutoBoxCacheMax=<n>`. **Nigdy nie buduj logiki na `==` między wrapperami.**

### String: immutability i pool

`String` jest **immutable** — po utworzeniu zawartość się nie zmienia (wewnętrznie `private final byte[] value`;
od Javy 9 *compact strings* — Latin-1 w 1 bajcie albo UTF-16 w 2).

- **String pool / intern:** literały (`"abc"`) trafiają do **string pool** (część heap). Identyczne literały
  to **ten sam obiekt**. `new String("abc")` **zawsze** tworzy nowy obiekt poza poolem; `.intern()` zwraca
  wersję z poolu.

```java
String x = "abc";
String y = "abc";
System.out.println(x == y);            // true  — oba z poolu
String z = new String("abc");
System.out.println(x == z);            // false — z jest poza poolem
System.out.println(x == z.intern());   // true  — intern() zwraca obiekt z poolu
System.out.println(x.equals(z));       // true  — porównujemy zawartość
```

**Dlaczego String jest immutable** (klasyczne pytanie):
1. **Bezpieczeństwo** — Stringi są używane jako nazwy klas, ścieżki, parametry połączeń, klucze; mutowalność
   pozwoliłaby zmienić je po walidacji (TOCTOU).
2. **Cache `hashCode`** — `hashCode()` liczony raz i cache'owany w polu `hash`; bezpieczne tylko gdy treść
   stała. Kluczowe dla wydajności kluczy w `HashMap`.
3. **Współdzielenie / string pool** — wiele referencji może bezpiecznie wskazywać ten sam obiekt.
4. **Thread-safety za darmo** — niemutowalne obiekty są bezpieczne między wątkami bez synchronizacji.

### Konkatenacja vs StringBuilder

Każde `+` na Stringach tworzy **nowy** obiekt (bo immutable). W pętli to kwadratowy koszt:

```java
// ŹLE — O(n²) alokacji w pętli
String s = "";
for (int i = 0; i < n; i++) s += i;

// DOBRZE — O(n)
StringBuilder sb = new StringBuilder();
for (int i = 0; i < n; i++) sb.append(i);
String s = sb.toString();
```

- Pojedyncze `a + b + c` jest OK — kompilator zamienia to na `StringBuilder`/`invokedynamic`
  (`StringConcatFactory` od Javy 9). Problem dotyczy **konkatenacji w pętli**.
- `StringBuilder` — szybki, **nie** thread-safe. `StringBuffer` — synchronizowany, wolniejszy, dziś rzadko
  potrzebny (preferuj `StringBuilder` lub niemutowalne podejścia).

### Kontrakt equals / hashCode

`Object.equals` domyślnie porównuje **referencje** (`==`). Nadpisujemy, gdy chcemy równość *logiczną*.

**5 reguł `equals`** (kontrakt z Javadoc):
1. **Zwrotność (reflexive):** `x.equals(x)` == `true`.
2. **Symetria (symmetric):** `x.equals(y)` ⇔ `y.equals(x)`.
3. **Przechodniość (transitive):** `x.equals(y)` ∧ `y.equals(z)` ⇒ `x.equals(z)`.
4. **Spójność (consistent):** wielokrotne wywołania dają ten sam wynik, jeśli obiekty się nie zmieniły.
5. **`x.equals(null)` == `false`** (nigdy NPE).

**Złota zasada:** *jeśli nadpisujesz `equals`, MUSISZ nadpisać `hashCode`* — bo kontrakt mówi:
**równe obiekty (`equals`) muszą mieć równy `hashCode`**. (Odwrotność nie obowiązuje: różne obiekty mogą mieć
ten sam hash — to kolizja.)

Złamanie tego **psuje `HashMap`/`HashSet`**: obiekt trafia do kubełka (bucket) wg `hashCode`; jeśli dwa „równe"
obiekty mają różne hashe, wylądują w różnych kubełkach i mapa ich „nie znajdzie":

```java
class Point {
    final int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }
    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point p)) return false;   // pattern matching, Java 21
        return x == p.x && y == p.y;
    }
    // BEZ hashCode → HashSet/HashMap zadziałają błędnie!
    @Override public int hashCode() { return Objects.hash(x, y); }
}
```

`record Point(int x, int y) {}` generuje poprawne `equals`/`hashCode`/`toString` **automatycznie** — najprostszy
sposób na ich poprawność. Więcej: [[wiedza/01-jezyk/equals-hashcode]] i [[wiedza/01-jezyk/rekordy]].

### Pass-by-value (rozprawienie się z mitem)

**Java ZAWSZE przekazuje argumenty przez wartość. Zawsze. Bez wyjątków.**

- Dla prymitywu: kopiowana jest **wartość**. Zmiana w metodzie nie wpływa na oryginał.
- Dla obiektu: kopiowana jest **referencja** (wartość referencji = adres). Metoda dostaje *kopię referencji*
  wskazującą na **ten sam** obiekt. Możesz więc **zmutować** obiekt (widoczne na zewnątrz), ale
  **przepięcie referencji** na nowy obiekt jest niewidoczne na zewnątrz.

```java
void mutate(StringBuilder sb) { sb.append("!"); }      // mutacja → widoczna
void reassign(StringBuilder sb) { sb = new StringBuilder("X"); } // przepięcie → NIEwidoczne

var b = new StringBuilder("hej");
mutate(b);    System.out.println(b);   // "hej!"  — ten sam obiekt zmutowany
reassign(b);  System.out.println(b);   // "hej!"  — oryginalna referencja nietknięta
```

Gdyby Java była *pass-by-reference*, `reassign` zmieniłoby `b` na zewnątrz. Nie zmienia — bo metoda dostała
**kopię** referencji. Mit „obiekty przez referencję" bierze się z mylenia *mutacji obiektu* z *przekazaniem
referencji przez wartość*.

## 3. Dlaczego / kiedy — kompromisy i pułapki

- **NPE przy unboxingu** — najczęstsza pułapka. `Integer` może być `null`; unboxing `null` rzuca
  `NullPointerException`:
  ```java
  Integer i = mapa.get("brak");   // null
  int x = i;                      // NPE! (i.intValue() na null)
  Integer sum = a + b;            // jeśli a lub b == null → NPE w arytmetyce
  ```
  Reguła: preferuj **prymitywy**, gdy wartość nie może/nie powinna być `null`. Wrappery tylko gdy potrzebny
  `null` (np. „brak wartości") albo generyki/kolekcje.

- **`==` vs `equals`** — `==` to tożsamość referencji (lub równość wartości dla prymitywów), `equals` to
  równość logiczna. Dla obiektów **prawie zawsze chcesz `equals`**. Cache `Integer` sprawia, że błędne `==`
  „działa" dla małych liczb i **wybucha** dla dużych — najgorszy rodzaj buga (działa w testach, pada na prodzie).

- **Koszt wydajności boxingu** — każdy boxing to potencjalna alokacja na heap + presja na GC + utrata lokalności
  cache (pointer chasing). W gorących pętlach `List<Integer>` jest znacząco wolniejsza niż `int[]`. Rozważ
  prymitywne kolekcje (np. Eclipse Collections, fastutil) lub `IntStream`. Project Valhalla (value types)
  ma to docelowo rozwiązać.

- **`float`/`double` i pieniądze** — IEEE 754 nie reprezentuje dokładnie 0.1; `0.1 + 0.2 != 0.3`. **Nigdy**
  nie używaj `double` do walut — używaj `BigDecimal` (lub liczb całkowitych groszy).

- **`String` jako klucz / mutable jako klucz** — klucze w `HashMap` powinny być **immutable** (lub przynajmniej
  nie zmieniać `hashCode` po wstawieniu). Mutowalny klucz → obiekt „gubi się" w mapie.

- **Memory leak przez string pool / intern** — masowy `intern()` na unikalnych Stringach zapycha pool.
  W praktyce rzadko interujesz ręcznie.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

W warstwie domeny tworzysz `record Money(long grosze, String waluta)` jako klucz w cache/`Map`. Dzięki rekordowi
masz darmowe i poprawne `equals/hashCode`. W DTO z bazy pole `Integer wiek` bywa `null` (kolumna nullable) —
mapując do prymitywu `int` dostaniesz NPE, więc świadomie trzymasz `Integer` i sprawdzasz `null`. W serwisie
budującym duży raport CSV użyjesz `StringBuilder` zamiast `+` w pętli. A na rozmowie ktoś pokaże Ci
`Integer a = 1000; Integer b = 1000; a == b` i spyta, czemu `false` — i wytłumaczysz cache `Integer`.

---

## ✅ Kryteria opanowania
- [ ] Wymienię 8 typów prymitywnych z rozmiarami i zakresami; wiem, że `char` jest unsigned.
- [ ] Wyjaśnię precyzyjnie, co leży na stacku, a co na heap (prymityw vs referencja vs obiekt).
- [ ] Wytłumaczę cache `Integer` (-128..127) i przewidzę wynik `==` dla 100 i 200.
- [ ] Wiem, kiedy unboxing rzuca NPE i jak tego unikać.
- [ ] Podam 5 reguł `equals` i uzasadnię, czemu z `equals` idzie `hashCode`.
- [ ] Udowodnię, że Java jest pass-by-value (kontrprzykład z przepięciem referencji).

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Gdzie *dokładnie* żyje referencja, a gdzie obiekt? A pole prymitywne w obiekcie?
- [ ] Przez którą metodę przechodzi autoboxing i dlaczego to ważne dla cache? (`Integer.valueOf`)
- [ ] Czemu `new String("a") == "a"` to `false`, a `"a" == "a"` to `true`?
- [ ] Dlaczego `String` jest immutable — 4 powody (bezpieczeństwo, hashCode cache, pool, thread-safety)?
- [ ] Co konkretnie psuje się w `HashMap`, gdy nadpiszę `equals` bez `hashCode`?
- [ ] Co robi kompilator z `a + b + c`, a co z `s += i` w pętli?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Czy Java jest pass-by-value czy pass-by-reference?" (zawsze by-value; pokaż przepięcie referencji)
- [ ] „`Integer a = 127; Integer b = 127; a == b`? A dla 128?" (true / false — cache)
- [ ] „Różnica `==` vs `equals` dla obiektów i dla `Integer`?"
- [ ] „Dlaczego `String` jest immutable?"
- [ ] „Kontrakt `equals`/`hashCode` — czemu nadpisując jedno, nadpisujesz drugie?"
- [ ] „Konkatenacja w pętli — co jest nie tak i jak naprawić?" (StringBuilder)
- [ ] „Kiedy unboxing rzuci NPE?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/01-jezyk.csv`.*
```
Ile typów prymitywnych ma Java i czy mają wrappery?;8 (byte, short, int, long, float, double, char, boolean); każdy ma wrapper (Integer, Boolean...).
Zakres int?;−2³¹ .. 2³¹−1, czyli ok. ±2,1 mld (32 bity, signed).
Który typ prymitywny jest unsigned?;Tylko char (16 bitów, 0..65535). Pozostałe całkowite są signed.
Gdzie leży zmienna prymitywna lokalna, referencja i obiekt?;Prymityw lokalny i referencja na stacku (ramka metody); obiekt zawsze na heap.
Przez którą metodę idzie autoboxing inta?;Integer.valueOf(int) — NIE przez new Integer (deprecated/usunięty).
Jaki jest zakres cache Integer?;-128..127 — w tym zakresie valueOf zwraca te same obiekty, więc == daje true.
Integer a=200,b=200; a==b → ?;false — poza cache to dwa różne obiekty; a.equals(b) daje true.
Kiedy unboxing rzuca NPE?;Gdy unboxujemy referencję null, np. int x = (Integer)null lub arytmetyka na null wrapperze.
Różnica == vs equals dla obiektów?;== porównuje referencje (tożsamość), equals — równość logiczną (jeśli nadpisany).
Czemu String jest immutable?;Bezpieczeństwo, cache hashCode, współdzielenie w string pool, thread-safety bez synchronizacji.
new String("abc") == "abc" → ?;false — new tworzy obiekt poza poolem; .intern() zwróciłby obiekt z poolu (==true).
Konkatenacja String w pętli — problem i fix?;Każde + tworzy nowy String (O(n²)); użyj StringBuilder.append → O(n).
StringBuilder vs StringBuffer?;Oba mutowalne; StringBuffer jest synchronizowany (wolniejszy), StringBuilder nie — domyślnie StringBuilder.
5 reguł equals?;Zwrotność, symetria, przechodniość, spójność, x.equals(null)==false.
Czemu nadpisując equals trzeba nadpisać hashCode?;Kontrakt: równe obiekty muszą mieć równy hashCode; inaczej HashMap/HashSet gubią obiekt (zły bucket).
Czy Java jest pass-by-value czy by-reference?;Zawsze by-value; dla obiektów kopiowana jest referencja — mutacja widoczna, ale przepięcie referencji nie.
Czemu double nie nadaje się do pieniędzy?;IEEE 754 nie reprezentuje dokładnie np. 0.1; 0.1+0.2≠0.3 — używaj BigDecimal.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/02-jvm/model-pamieci]] · [[wiedza/02-jvm/garbage-collection]] · [[wiedza/01-jezyk/equals-hashcode]] · [[wiedza/01-jezyk/kolekcje]] · [[wiedza/01-jezyk/rekordy]]
