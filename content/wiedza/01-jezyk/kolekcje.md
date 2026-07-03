---
temat: "Java Collections Framework"
faza: 1
status: w-trakcie
priorytet: 🔴
tags: [java, jezyk, kolekcje, collections, hashmap]
powiazane: ["[[wiedza/01-jezyk/typy-i-pamiec]]", "[[wiedza/01-jezyk/typy-i-pamiec]]", "[[wiedza/01-jezyk/generyki]]", "[[wiedza/03-wspolbieznosc/synchronizacja]]"]
---

# Java Collections Framework

> **TL;DR:** **JCF** to spójna hierarchia interfejsów (`Iterable→Collection→List/Set/Queue/Deque`, osobno `Map`)
> i ich implementacji o znanych charakterystykach **Big-O**. Dobór = znajomość kosztów operacji: `ArrayList` (tablica,
> O(1) `get`), `HashMap`/`HashSet` (O(1) na poprawnym `equals`/`hashCode`), `TreeMap`/`TreeSet` (posortowane, O(log n)),
> `ArrayDeque` (stos/kolejka). Sercem jest **HashMap** — buckety + hash + `equals` przy kolizji; zły `hashCode` cicho psuje wyszukiwanie.

## 1. Co — hierarchia i API

Dwa rozłączne korzenie. `Map` **nie** jest `Collection` (przechowuje pary, nie pojedyncze elementy).

```
Iterable
  └── Collection
        ├── List   ── ArrayList, LinkedList, (Vector/Stack — legacy)
        ├── Set    ── HashSet, LinkedHashSet, TreeSet
        └── Queue
              └── Deque ── ArrayDeque, LinkedList; PriorityQueue (Queue, nie Deque)

Map  (osobno!)  ── HashMap, LinkedHashMap, TreeMap
        └── SortedMap └── NavigableMap (TreeMap)
```

- **`List`** — uporządkowana sekwencja z dostępem po indeksie, dopuszcza duplikaty.
- **`Set`** — zbiór bez duplikatów (kontrolowane przez `equals`/`hashCode` lub `compareTo`).
- **`Queue`/`Deque`** — kolejka FIFO / kolejka dwustronna (też jako stos LIFO).
- **`Map`** — odwzorowanie klucz→wartość, unikalne klucze. `SortedMap`/`NavigableMap` dokładają porządek i nawigację (`floorKey`, `ceilingKey`, `headMap`…).

```java
List<String> l   = new ArrayList<>();
Set<Integer> s   = new HashSet<>();
Map<String,Integer> m = new HashMap<>();
Deque<Integer> stack = new ArrayDeque<>();   // push/pop/peek — preferowany stos
```

### Implementacje — kiedy które

**List**
- **`ArrayList`** — tablica dynamiczna. `get`/`set` po indeksie **O(1)**, `add` na końcu zamortyzowane O(1) (czasem realloc + kopia), `add`/`remove` w środku **O(n)** (przesuwanie elementów). **Domyślny wybór** dla list.
- **`LinkedList`** — lista podwójnie wiązana. `add`/`remove` na końcach **O(1)**, ale `get(i)` **O(n)** (wędrówka po węzłach) i każdy węzeł to osobny obiekt (gorszy cache locality, większy narzut pamięci). **Rzadko optymalna** — nawet kolejka/dek lepiej wychodzi na `ArrayDeque`. Sensowna głównie przy częstym `add/remove` przez `ListIterator` w środku.

**Set**
- **`HashSet`** — oparty na `HashMap`. `add`/`contains`/`remove` **O(1)** średnio, **bez gwarancji kolejności**.
- **`LinkedHashSet`** — `HashSet` + lista wiązana przeplatająca elementy: zachowuje **kolejność wstawiania**, narzut na wskaźniki, dalej O(1).
- **`TreeSet`** — **drzewo czerwono-czarne** (samobalansujące BST). Elementy **posortowane**, operacje **O(log n)**, wymaga `Comparable` albo `Comparator`. Daje API nawigacyjne (`first`, `last`, `headSet`, `ceiling`).

**Map**
- **`HashMap`** — patrz sekcja 2. O(1) średnio, bez kolejności.
- **`LinkedHashMap`** — `HashMap` z listą wiązaną; tryb access-order pozwala zbudować **cache LRU** (nadpisz `removeEldestEntry`).
- **`TreeMap`** — `NavigableMap` na drzewie czerwono-czarnym, klucze posortowane, O(log n).

**Queue/Deque**
- **`ArrayDeque`** — kolejka kołowa na tablicy. **Preferowany stos i kolejka** (szybszy niż `Stack`/`LinkedList`, brak synchronizacji). Nie dopuszcza `null`.
- **`PriorityQueue`** — **kopiec binarny** (min-heap wg `Comparator`/`Comparable`). `offer`/`poll` O(log n), `peek` O(1). Nie jest posortowana całościowo — gwarantuje tylko, że na szczycie jest minimum.

## 2. Jak — wnętrze HashMap / HashSet

`HashMap` to tablica **bucketów** (`Node<K,V>[] table`), domyślnie 16, **zawsze potęga dwójki**.

**Ścieżka `put(key, value)`:**
1. **`hashCode()`** klucza → `int`. JVM dodatkowo **miesza bity** (`h ^ (h >>> 16)`), by górne bity wpłynęły na indeks — to ratuje słabe `hashCode` przed kolizjami.
2. **Indeks bucketa:** `index = hash & (n - 1)` (bo `n` = potęga 2, to tańszy odpowiednik `hash % n`).
3. Bucket pusty → wstaw `Node`. Bucket zajęty → **kolizja**: idziemy po łańcuchu i porównujemy `equals()`. Jeśli `equals` zwróci `true` — to ten sam klucz, **nadpisujemy wartość**; jeśli nie — dopinamy nowy węzeł.

**Ścieżka `get(key)`** jest analogiczna: ten sam hash → ten sam bucket → `equals` przeszukuje łańcuch. Stąd:

> **Bez poprawnego `hashCode` klucz może trafić do innego bucketa niż przy `put` — i `get` zwróci `null` mimo że obiekt "tam jest".** To nie wyjątek, to ciche zgubienie danych.

**Kontrakt `equals`/`hashCode` (krytyczny):**
- Jeśli `a.equals(b)` ⇒ `a.hashCode() == b.hashCode()` (obowiązkowo). Złamanie ⇒ duplikaty w `HashSet`, gubienie kluczy w `HashMap`.
- Odwrotnie nie musi zachodzić (różne obiekty mogą mieć ten sam hash — to dozwolona kolizja).
- **Klucze powinny być niemutowalne** w polach liczących się do hasha. Mutacja klucza po wstawieniu zmienia jego bucket → obiekt staje się „nieosiągalny" przez `get`.

**Treeify (Java 8+):** gdy łańcuch w jednym buckecie przekroczy **8 węzłów** (`TREEIFY_THRESHOLD`) **i** tablica ma ≥ 64 buckety, łańcuch zamienia się w **drzewo czerwono-czarne** — degraduje najgorszy przypadek kolizji z O(n) do **O(log n)** (obrona przed atakiem hash-flooding). Poniżej `UNTREEIFY_THRESHOLD` (6) wraca do listy.

**Resize i load factor:** `load factor = 0.75` (kompromis pamięć ↔ liczba kolizji). Gdy `size > capacity * 0.75`, tablica **podwaja rozmiar** i wszystkie wpisy są **rehashowane** do nowych bucketów (kosztowne O(n)). Stąd: znając docelowy rozmiar, podaj **initial capacity** (`new HashMap<>(expected/0.75 + 1)`), by uniknąć wielokrotnych resize.

**`HashSet`** = `HashMap`, gdzie elementy to klucze, a wartością jest stały `PRESENT`. Cała mechanika (buckety, treeify, kontrakt `equals`/`hashCode`) jest identyczna.

**Fail-fast iteratory i `ConcurrentModificationException`:** kolekcja trzyma licznik **`modCount`** (zmieniany przy każdej strukturalnej modyfikacji). Iterator zapamiętuje go przy starcie; jeśli wykryje rozjazd przy `next()`, rzuca **`ConcurrentModificationException`** — natychmiast, zamiast działać na niespójnych danych. Dotyczy to też modyfikacji w **jednym** wątku (klasyczna pułapka: `for` po liście z `list.remove(x)` w środku). Bezpieczne usuwanie: `Iterator.remove()` albo `Collection.removeIf(...)`. Fail-fast to mechanizm **best-effort do wykrywania błędów**, nie gwarancja współbieżności.

## 3. Dlaczego / kiedy — dobór i pułapki

**Dobór wg Big-O** — to nie akademicka ozdoba, to decyzja architektoniczna:
- Częste `contains`/lookup po kluczu → `HashMap`/`HashSet` (O(1)), nie `List` (O(n) `contains`).
- Potrzebny porządek posortowany / zakresy → `TreeMap`/`TreeSet` (O(log n) ale uporządkowane).
- Potrzebna stabilna kolejność wstawiania → `LinkedHashMap`/`LinkedHashSet`.
- Indeksowany losowy dostęp → `ArrayList`. Stos/kolejka → `ArrayDeque`.

**Comparable vs Comparator** (dla `TreeSet`/`TreeMap`/`sort`):
- **`Comparable<T>`** — *naturalny* porządek typu, metoda `compareTo` w samej klasie (jeden porządek).
- **`Comparator<T>`** — porządek *zewnętrzny*, przekazywany osobno; dowolnie wiele (`Comparator.comparing(...).thenComparing(...)`, `.reversed()`).
- **Pułapka spójności z `equals`:** w `TreeSet`/`TreeMap` o unikalności decyduje `compareTo`/`compare` **== 0**, a **nie** `equals`. Jeśli komparator uzna dwa „różne" obiekty za równe (zwróci 0), drzewo potraktuje je jako duplikat. Komparator powinien być *consistent with equals*.

**Niemutowalne kolekcje i ich pułapki:**
- **`List.of(...)`, `Set.of(...)`, `Map.of(...)`** (Java 9+) — prawdziwie niemodyfikowalne (każda próba mutacji → `UnsupportedOperationException`). **Nie dopuszczają `null`** (rzucają `NPE`), a `Set.of`/`Map.of` rzucają przy **duplikatach** kluczy/elementów.
- **`Collections.unmodifiableList(x)`** — to tylko **widok read-only** na oryginał: jeśli ktoś trzyma referencję do `x` i go zmodyfikuje, zmiana **przebije się** przez widok. To „opakowanie", nie kopia.
- Niemutowalność jest **płytka** — kolekcja jest „zamrożona", ale jej elementy (jeśli mutowalne) nadal można zmieniać.

**Współbieżność (krótko):** klasyczne kolekcje **nie są thread-safe**. Pod presją wątków używa się `ConcurrentHashMap` (segmentowane/CAS-owe, brak globalnego locka), `CopyOnWriteArrayList`, kolejek z `java.util.concurrent` — a `Collections.synchronizedMap` daje tylko gruboziarnisty lock. Szczegóły: [[wiedza/03-wspolbieznosc/synchronizacja]].

### Tabela złożoności (Big-O, średni przypadek)

| Operacja / struktura | `get`/lookup | `add`/`put` | `remove` | `contains` | Kolejność |
|---|---|---|---|---|---|
| `ArrayList`       | O(1) (indeks) | O(1)* koniec / O(n) środek | O(n) | O(n) | wstawiania |
| `LinkedList`      | O(n) | O(1) końce | O(1) przy iteratorze / O(n) szukanie | O(n) | wstawiania |
| `HashSet`/`HashMap` | O(1) | O(1) | O(1) | O(1) | brak |
| `LinkedHashSet`/`LinkedHashMap` | O(1) | O(1) | O(1) | O(1) | wstawiania / access |
| `TreeSet`/`TreeMap` | O(log n) | O(log n) | O(log n) | O(log n) | posortowana |
| `ArrayDeque`      | — | O(1) końce | O(1) końce | O(n) | FIFO/LIFO |
| `PriorityQueue`   | O(1) peek | O(log n) | O(log n) | O(n) | wg priorytetu (szczyt) |

\* zamortyzowane; pojedynczy `add` może wywołać realloc + kopię O(n). Worst-case `HashMap` to O(log n) (po treeify), O(n) bez niego.

## Przykład w praktyce
W warstwie serwisowej zliczasz wystąpienia: `Map<String,Integer> counts = new HashMap<>(); counts.merge(word, 1, Integer::sum);`
— O(1) na słowo. Gdy kluczem jest **własna klasa** (np. `record OrderKey(...)`), `record` daje poprawne `equals`/`hashCode`
za darmo; ze zwykłą klasą **musisz** je wygenerować, inaczej deduplikacja w `Set`/grupowanie w `Map` cicho zawiedzie.
Raport posortowany po kluczu → przepisz wynik do `TreeMap`. Cache ostatnio użytych → `LinkedHashMap` w trybie access-order.

---

## ✅ Kryteria opanowania
- [ ] Narysuję hierarchię JCF i wskażę, czemu `Map` jest osobno.
- [ ] Opiszę krok po kroku `put`/`get` w `HashMap` (hash → bucket → `equals`).
- [ ] Wyjaśnię, czemu zły `hashCode` psuje wyszukiwanie, a nie rzuca wyjątku.
- [ ] Wiem, co to treeify, load factor 0.75 i resize.
- [ ] Wytłumaczę `ConcurrentModificationException` przez `modCount` i jak go uniknąć.
- [ ] Dobiorę kolekcję na podstawie tabeli Big-O.
- [ ] Rozróżnię `List.of` vs `Collections.unmodifiable*` i ich pułapki.

### 🔲 Black-box check
- [ ] Jak `HashMap` liczy indeks bucketa i czemu pojemność to potęga 2?
- [ ] Co się dzieje przy kolizji i po przekroczeniu 8 węzłów w buckecie?
- [ ] Dlaczego mutowalny klucz „znika" z `HashMap`?
- [ ] Czemu `TreeSet` o unikalności decyduje przez `compareTo`, nie `equals`?
- [ ] Czemu `LinkedList` rzadko bije `ArrayList`/`ArrayDeque`?
- [ ] Jak działa fail-fast i czym różni się od thread-safety?

### 🎤 Pytania rekrutacyjne
- [ ] „Jak działa `HashMap` w środku?" (buckety, hash, `equals`, treeify, resize)
- [ ] „Co się stanie, jeśli źle nadpiszę `hashCode`?"
- [ ] „`ArrayList` vs `LinkedList` — kiedy co?" (prawie zawsze `ArrayList`)
- [ ] „`HashMap` vs `TreeMap` vs `LinkedHashMap`?"
- [ ] „Kiedy `ConcurrentModificationException` i jak go uniknąć?"
- [ ] „`Comparable` vs `Comparator`?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/01-jezyk.csv`.*
```
Czy Map jest częścią Collection?;Nie — Map ma osobną hierarchię (przechowuje pary klucz-wartość, nie pojedyncze elementy).
Jak HashMap liczy indeks bucketa?;index = hash & (n-1), gdzie hash to wymieszany hashCode, a n to pojemność (potęga 2).
Dlaczego pojemność HashMap to potęga 2?;Bo hash & (n-1) jest tańszym odpowiednikiem modulo i równomiernie rozkłada bity.
Co dzieje się przy kolizji w HashMap?;Łańcuch w buckecie przeszukiwany jest przez equals — równy klucz nadpisuje wartość, nierówny dopina nowy węzeł.
Dlaczego kontrakt equals/hashCode jest krytyczny?;Zły hashCode kieruje klucz do innego bucketa niż przy put — get cicho zwraca null mimo że obiekt istnieje.
Co to treeify w HashMap (Java 8+)?;Gdy łańcuch w buckecie ma >8 węzłów (i tablica ≥64), zamienia się w drzewo czerwono-czarne — worst-case O(n)→O(log n).
Ile wynosi load factor HashMap i co robi?;0.75; gdy size > capacity*0.75 tablica podwaja rozmiar i rehashuje wpisy (resize, O(n)).
Czym jest HashSet w środku?;HashMap, gdzie elementy to klucze, a wartość to stały obiekt PRESENT.
ArrayList vs LinkedList get(i)?;ArrayList O(1) (tablica), LinkedList O(n) (wędrówka po węzłach) — ArrayList prawie zawsze lepszy.
Kiedy TreeSet/TreeMap zamiast HashSet/HashMap?;Gdy potrzebny posortowany porządek lub API nawigacyjne (floor/ceiling/headMap); kosztem O(log n) zamiast O(1).
Jaką strukturą jest TreeMap/TreeSet?;Drzewo czerwono-czarne (samobalansujące BST), operacje O(log n).
Jaka kolekcja na stos/kolejkę?;ArrayDeque — szybszy niż Stack i LinkedList, nie dopuszcza null.
Co zwraca PriorityQueue na szczycie?;Element o najmniejszym priorytcie wg Comparatora/Comparable (min-heap); nie jest globalnie posortowana.
Skąd ConcurrentModificationException?;Iterator wykrywa zmianę modCount kolekcji podczas iteracji (fail-fast); unikaj przez Iterator.remove lub removeIf.
Comparable vs Comparator?;Comparable = naturalny porządek w klasie (compareTo, jeden); Comparator = porządek zewnętrzny, wiele, składalny.
List.of() vs Collections.unmodifiableList()?;List.of to prawdziwie niemodyfikowalna kopia (bez null); unmodifiableList to read-only widok — zmiany oryginału się przebijają.
Pułapka mutowalnego klucza w HashMap?;Zmiana pola wpływającego na hashCode po wstawieniu zmienia bucket — klucz staje się nieosiągalny przez get.
Jak uniknąć wielu resize w HashMap?;Podać initial capacity dopasowaną do oczekiwanego rozmiaru (expected/0.75 + 1).
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/01-jezyk/typy-i-pamiec]] · [[wiedza/01-jezyk/typy-i-pamiec]] · [[wiedza/01-jezyk/generyki]] · [[wiedza/03-wspolbieznosc/synchronizacja]]
