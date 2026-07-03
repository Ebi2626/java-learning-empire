---
temat: "Model pamięci JVM (runtime data areas)"
faza: 2
status: nieopanowany
priorytet: 🟡
tags: [java, jvm]
powiazane: ["[[wiedza/02-jvm/garbage-collection]]", "[[wiedza/02-jvm/java-memory-model]]", "[[wiedza/01-jezyk/typy-i-pamiec]]"]
---

# Model pamięci JVM (runtime data areas)

> **TL;DR:** JVM dzieli pamięć na **runtime data areas** — **heap** (współdzielony, obiekty, GC-owany, dzielony na young/old) i **metaspace** (metadane klas, w pamięci natywnej — zastąpił PermGen w Java 8) są wspólne dla całego procesu; **stos wątku**, **PC register** i **native method stack** są per-wątek. Alokacja jest tania dzięki **TLAB** (bump-the-pointer), a niektóre obiekty przez **escape analysis** w ogóle nie trafiają na heap. To *nie* jest [[wiedza/02-jvm/java-memory-model|Java Memory Model]] (semantyka współbieżności) — to fizyczny układ pamięci runtime.

## 1. Co — definicja i API

**Runtime data areas** (JVM Spec, rozdz. 2.5) to obszary pamięci, które JVM tworzy przy starcie procesu i wątków. Dwa są **współdzielone** przez wszystkie wątki, reszta jest **per-thread**.

```
┌───────────────────────── PROCES JVM (HotSpot, Java 21) ─────────────────────────┐
│                                                                                  │
│  WSPÓŁDZIELONE (per-process)                                                      │
│  ┌────────────────────────────── HEAP (-Xms / -Xmx) ─────────────────────────┐   │
│  │  Young Generation                          Old Generation (Tenured)       │   │
│  │  ┌────────┬──────────┬──────────┐          ┌──────────────────────────┐   │   │
│  │  │  Eden  │ Survivor │ Survivor │          │  długo żyjące obiekty     │   │   │
│  │  │        │   S0     │   S1     │  ──MinorGC──►                        │   │   │
│  │  └────────┴──────────┴──────────┘          └──────────────────────────┘   │   │
│  │  ← alokacja obiektów (TLAB), sprząta GC →                                  │   │
│  └───────────────────────────────────────────────────────────────────────────┘   │
│  ┌───────────── METASPACE (pamięć natywna, NIE heap; -XX:MaxMetaspaceSize) ───┐   │
│  │  metadane klas (Klass), pola, metody, constant pool, JIT stubs…            │   │
│  └───────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  PER-THREAD (jeden komplet na wątek)                                              │
│  ┌── Wątek 1 ──────────────┐  ┌── Wątek 2 ──────────────┐                         │
│  │ JVM Stack (-Xss)        │  │ JVM Stack (-Xss)        │   ← ramki (frames)      │
│  │  ├ frame ┐              │  │  ├ frame ┐              │                         │
│  │  ├ frame ┤ LVA+operand  │  │  ├ frame ┤              │                         │
│  │  └ frame ┘              │  │  └ frame ┘              │                         │
│  │ PC register             │  │ PC register             │                         │
│  │ Native Method Stack     │  │ Native Method Stack     │                         │
│  └─────────────────────────┘  └─────────────────────────┘                         │
└──────────────────────────────────────────────────────────────────────────────────┘
```

- **Heap** — współdzielony, tu żyją **wszystkie obiekty i tablice**. Zarządzany przez [[wiedza/02-jvm/garbage-collection|GC]]. Flagi: `-Xms` (rozmiar startowy), `-Xmx` (max).
- **JVM Stack (stos wątku)** — jeden na wątek, złożony z **ramek (frames)**; ramka na każde wywołanie metody. Flaga `-Xss` (np. `-Xss1m`). Przechowuje zmienne lokalne, operandy, zwroty. Zbyt głęboka rekurencja → **`StackOverflowError`**.
- **PC register (Program Counter)** — jeden na wątek; trzyma adres aktualnie wykonywanej instrukcji bytecode (dla metod native jest *undefined*).
- **Native Method Stack** — stos dla metod natywnych (JNI, C/C++). W HotSpot zwykle to stos C danego wątku.
- **Metaspace** — metadane klas: struktura `Klass`, tablice metod/pól, runtime constant pool. Od **Java 8 zastąpił PermGen** i leży w **pamięci natywnej** (nie na heap), więc rośnie dynamicznie do limitu OS lub `-XX:MaxMetaspaceSize`.

Typowy kod, gdzie leży co:
```java
void m() {
    int x = 42;                 // local variable array (na stosie, w ramce m)
    Point p = new Point(1, 2);  // obiekt Point → heap; referencja p → stos
    int[] a = new int[1000];    // tablica → heap; referencja a → stos
}
```

## 2. Jak — co dzieje się pod spodem

### Układ obiektu na heap (object layout)
Każdy obiekt to nie tylko jego pola. HotSpot dokłada **nagłówek (object header)**:

```
[ mark word (8B) ][ klass pointer (4B compressed / 8B) ][ pola ][ padding do 8B ]
```
- **mark word** — identity hashcode, wiek GC (age dla promocji), bity blokad (biased/thin/heavyweight lock), flagi GC.
- **klass pointer** — wskaźnik do metadanych klasy w **metaspace** (kim jest ten obiekt).
- **pola** — wartości pól instancji; **padding** dopełnia do wielokrotności **8 bajtów** (8-byte alignment), bo alokator adresuje co 8B.

**Compressed oops** (`-XX:+UseCompressedOops`, domyślnie ON): referencje 32-bitowe zamiast 64-bitowych. Dzięki wyrównaniu 8B JVM koduje adres jako `oop << 3`, więc 32-bitową liczbą zaadresuje **do ~32 GB** heapu. Powyżej tego progu JVM przełącza się na 64-bitowe wskaźniki (2× większe referencje, gorsza gęstość cache) — dlatego heap 30 GB potrafi mieścić *więcej* obiektów niż 34 GB.

Pusty `Object` to **16 B** (12 B header + 4 B padding). To liczy się przy milionach małych obiektów.

### Ramka stosu (stack frame)
Nowa ramka powstaje przy każdym wywołaniu metody, znika przy powrocie. Zawiera:
- **Local Variable Array (LVA)** — indeksowana tablica: parametry + zmienne lokalne. Slot 0 to `this` (metody instancyjne). `long`/`double` zajmują 2 sloty.
- **Operand Stack** — roboczy stos maszyny stosowej JVM; instrukcje bytecode (`iload`, `iadd`, `invokevirtual`) pushują/popują tu operandy. Jego max głębokość jest znana w compile-time (atrybut `max_stack`).
- **Frame Data** — runtime constant pool reference, adres powrotu, wsparcie dla wyjątków.

Zobacz `javap -c -v` — pokaże `max_stack`, `max_locals` dla każdej metody.

### Generacje na heap
Oparte na **hipotezie generacyjnej**: większość obiektów umiera młodo.
- **Young Generation** = **Eden** + 2 × **Survivor** (S0, S1). Nowe obiekty → Eden. **Minor GC** kopiuje żywych z Eden+aktywnego survivora do drugiego survivora, inkrementując **age**. Po przekroczeniu `-XX:MaxTenuringThreshold` (domyślnie do 15) obiekt jest **promowany** do Old.
- **Old Generation (Tenured)** — długo żyjące obiekty. Sprzątane przez **Major/Full GC** (rzadziej, ale drożej).

Uwaga: **G1 GC** (domyślny w Java 21) fizycznie dzieli heap na **regiony**, a nie na jeden ciągły young/old — logika generacyjna jest zachowana, ale rozłożona po regionach. ZGC/Shenandoah (region-based, mostly concurrent) też inaczej.

### TLAB — czemu alokacja jest szybka
**Thread-Local Allocation Buffer**: każdy wątek dostaje własny kawałek Edenu. Alokacja to zwykle **bump-the-pointer** — wystarczy przesunąć wskaźnik o rozmiar obiektu i porównać z końcem TLAB. **Bez synchronizacji** (bufor prywatny wątku), zwykle kilka instrukcji CPU. Dopiero gdy TLAB się skończy, wątek pobiera nowy z Edenu (to wymaga synchronizacji, ale zdarza się rzadko). Duże obiekty idą prosto do heapu poza TLAB. `-XX:+UseTLAB` (domyślnie ON).

### Typy referencji (java.lang.ref)
Decydują, kiedy GC może zebrać obiekt.

| Typ | Zbierany gdy… | Zastosowanie |
|-----|---------------|--------------|
| **Strong** | nigdy dopóki osiągalny | zwykłe `Object o = ...` |
| **SoftReference** | brak innych referencji **i** brakuje pamięci | cache wrażliwy na pamięć |
| **WeakReference** | brak innych **strong** referencji (najbliższy GC) | `WeakHashMap`, kanonikalizacja, listenery |
| **PhantomReference** | po finalizacji, przed zwolnieniem | **`Cleaner`** — deterministyczne sprzątanie zasobów natywnych |

`WeakHashMap` trzyma klucze przez WeakReference — wpis znika, gdy nikt inny nie trzyma klucza. `Cleaner` (Java 9+) zastąpił `finalize()` (deprecated/usuwany) — używa phantom refs do zwalniania natywnych uchwytów.

### Escape analysis i scalar replacement — obiekt może NIE trafić na heap
JIT (C2) analizuje, czy referencja do obiektu **„ucieka"** poza metodę (do innego wątku, do pola, przez return). Jeśli nie ucieka:
- **Scalar replacement** — obiekt „rozbijany" na osobne zmienne skalarne, które JIT trzyma w rejestrach/na stosie. Alokacja na heap **znika**, GC nie ma czego sprzątać.
- **Lock elision** — synchronizacja na obiekcie nieuciekającym jest usuwana.

```java
double dist(int x, int y) {
    Point p = new Point(x, y);   // nie ucieka → C2 może NIE alokować na heap
    return Math.sqrt(p.x*p.x + p.y*p.y);
}
```
Dlatego mikrobenchmark „new w pętli" bywa darmowy — ale tylko po rozgrzaniu JIT i tylko gdy obiekt naprawdę nie ucieka (`-XX:+DoEscapeAnalysis`, domyślnie ON).

### Jak flagi mapują się na obszary
- `-Xms` / `-Xmx` → **heap** (start / max).
- `-Xss` → rozmiar **stosu** na wątek.
- `-XX:MaxMetaspaceSize` → limit **metaspace** (domyślnie nieograniczony do pamięci OS).
- `-XX:+UseCompressedOops`, `-XX:MaxTenuringThreshold`, `-XX:+UseTLAB` → jak wyżej.

## 3. Dlaczego / kiedy — kompromisy i wąskie gardła

- **Warianty `OutOfMemoryError` — komunikat wskazuje obszar:**
  - `OutOfMemoryError: Java heap space` — zabrakło **heapu**; za mały `-Xmx` albo **leak** (obiekty pozostają osiągalne). Diagnoza: heap dump (`-XX:+HeapDumpOnOutOfMemoryError`), MAT.
  - `OutOfMemoryError: Metaspace` — za dużo klas (dynamiczne generowanie proxy, częsty redeploy, classloader leak). Podnieś `-XX:MaxMetaspaceSize` lub napraw wyciek classloaderów.
  - `OutOfMemoryError: unable to create native thread` — wyczerpano **pamięć natywną / limit OS** na stosy wątków (za dużo wątków × `-Xss`, `ulimit -u`, `/proc/sys/kernel/threads-max`). To *nie* problem heapu — pomaga *zmniejszenie* `-Xmx`/`-Xss` albo pule/virtual threads.
- **`StackOverflowError` vs `OutOfMemoryError` — kluczowa różnica:** SOE to wyczerpanie **jednego stosu wątku** (za głęboka rekurencja) — dotyka jeden wątek, nie proces. OOM to wyczerpanie **współdzielonego** obszaru (heap/metaspace) lub pamięci natywnej — zwykle kładzie cały proces. SOE = „za głęboko", OOM = „za dużo".
- **Compressed oops ~32 GB** — nie skacz z heapu 30 GB na 40 GB „na zapas": tracisz compressed oops i możesz zmieścić *mniej* danych. Trzymaj się <~32 GB albo skacz od razu mocno wyżej.
- **Metaspace bez limitu** — domyślnie może zjeść całą pamięć hosta/kontenera przy wycieku classloaderów; w kontenerze warto ustawić `MaxMetaspaceSize`.
- **Za duży `-Xss` × wiele wątków** — każdy wątek rezerwuje `-Xss`; 10 000 wątków × 1 MB = 10 GB pamięci natywnej. Stąd sens **virtual threads** (Loom) i pul.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)
Spring Boot na k8s dostaje `OutOfMemoryError: Java heap space` pod obciążeniem. Włączasz `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps`, analizujesz w **Eclipse MAT** — dominator tree pokazuje `HashMap` cache rosnący bez ograniczeń (brak eviction). Zamieniasz go na cache z `SoftReference`/limitem (Caffeine). Osobno w logach po każdym hot-redeployu widać `OutOfMemoryError: Metaspace` — to **classloader leak** (stary `WebappClassLoader` trzymany przez wątek/thread-local), więc metaspace nie zwalnia klas mimo undeploy. Podczas profilowania JFR widzisz, że alokacje w gorącej ścieżce znikają po rozgrzaniu — to **escape analysis + scalar replacement** eliminuje krótkożyjące obiekty DTO.

---

## ✅ Kryteria opanowania
- [ ] Wymienię wszystkie runtime data areas i powiem, które są współdzielone, a które per-thread.
- [ ] Wyjaśnię układ obiektu (header: mark word + klass pointer, pola, padding 8B) i próg ~32 GB compressed oops.
- [ ] Opiszę zawartość ramki stosu (LVA, operand stack, frame data).
- [ ] Wytłumaczę TLAB i bump-the-pointer jako powód szybkiej alokacji.
- [ ] Podam 4 typy referencji i po jednym zastosowaniu każdej.
- [ ] Wyjaśnię, jak escape analysis + scalar replacement mogą uniknąć heapu.
- [ ] Zmapuję `-Xmx`/`-Xms`/`-Xss`/`MaxMetaspaceSize` na obszary.
- [ ] Odróżnię warianty OOM od StackOverflowError.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Czemu metaspace jest w pamięci natywnej, a nie na heap, i co to zmieniło względem PermGen?
- [ ] Co dokładnie siedzi w mark word i po co (hashcode, age, blokady)?
- [ ] Dlaczego alokacja przez TLAB nie wymaga synchronizacji?
- [ ] Jak działa promocja Eden → Survivor → Old i czym jest tenuring threshold?
- [ ] Skąd bierze się dokładnie próg ~32 GB dla compressed oops (8B alignment → `oop<<3`)?
- [ ] Kiedy `WeakReference` vs `SoftReference` vs `PhantomReference`?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Gdzie leżą obiekty, a gdzie zmienne lokalne i referencje?"
- [ ] „Czym różni się StackOverflowError od OutOfMemoryError?"
- [ ] „Co zastąpiło PermGen i dlaczego?"
- [ ] „Jak Java może nie zaalokować obiektu na heap?" (escape analysis / scalar replacement)
- [ ] „Czemu heap >32 GB może mieścić mniej danych niż 31 GB?"
- [ ] „Do czego WeakHashMap i Cleaner?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Które runtime data areas są współdzielone przez wszystkie wątki?;Heap oraz metaspace (metadane klas). Stos, PC register i native method stack są per-thread.
Gdzie żyją obiekty i tablice w JVM?;Zawsze na heap; na stosie leżą tylko zmienne lokalne prymitywów i referencje do obiektów.
Co zastąpiło PermGen w Java 8?;Metaspace — metadane klas przeniesione z heapu do pamięci natywnej, rośnie dynamicznie do limitu OS/MaxMetaspaceSize.
Z czego składa się nagłówek obiektu w HotSpot?;Mark word (hashcode, age GC, bity blokad) + klass pointer (wskaźnik do metadanych klasy w metaspace).
Po co padding w object layout?;Dopełnienie rozmiaru obiektu do wielokrotności 8 bajtów (8-byte alignment wymagany przez alokator).
Ile bajtów zajmuje pusty Object w HotSpot (compressed oops)?;16 bajtów (12 B nagłówka + 4 B paddingu).
Dlaczego compressed oops działają tylko do ~32 GB?;32-bitowa referencja + 8-byte alignment koduje adres jako oop<<3, co adresuje do 4G*8B ≈ 32 GB.
Z czego składa się ramka stosu (stack frame)?;Local variable array (parametry+lokalne), operand stack (maszyna stosowa) i frame data (constant pool ref, adres powrotu, wyjątki).
Co siedzi w slocie 0 local variable array metody instancyjnej?;Referencja this.
Czym jest TLAB i czemu daje szybką alokację?;Thread-Local Allocation Buffer — prywatny kawałek Edenu; alokacja to bump-the-pointer bez synchronizacji, kilka instrukcji CPU.
Jakie są 4 typy referencji i po jednym zastosowaniu?;Strong (zwykłe), Soft (cache wrażliwy na pamięć), Weak (WeakHashMap), Phantom (Cleaner, sprzątanie zasobów natywnych).
Kiedy GC zbiera SoftReference vs WeakReference?;Weak — przy najbliższym GC gdy brak strong refs; Soft — dopiero gdy brakuje pamięci.
Co robi escape analysis + scalar replacement?;Jeśli obiekt nie ucieka z metody, C2 rozbija go na skalary w rejestrach/na stosie — obiekt nie trafia na heap, GC nie sprząta.
Które flagi mapują się na które obszary?;-Xms/-Xmx → heap; -Xss → stos wątku; -XX:MaxMetaspaceSize → metaspace.
Czym różni się StackOverflowError od OutOfMemoryError?;SOE = wyczerpanie jednego stosu wątku (za głęboka rekurencja), dotyka wątek; OOM = wyczerpanie współdzielonego heapu/metaspace lub pamięci natywnej, kładzie proces.
Co oznacza OutOfMemoryError: unable to create native thread?;Wyczerpano pamięć natywną/limit OS na stosy wątków (za dużo wątków × -Xss) — pomaga zmniejszenie -Xss/-Xmx lub virtual threads, nie większy heap.
Jak wygląda przepływ obiektu przez generacje young?;Alokacja w Eden → Minor GC kopiuje żywych do Survivora (age++) → po MaxTenuringThreshold promocja do Old.
Który GC jest domyślny w Java 21 i jak dzieli heap?;G1 GC — dzieli heap na regiony (nie ciągły young/old), zachowując logikę generacyjną rozłożoną po regionach.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/02-jvm/garbage-collection]] · [[wiedza/02-jvm/java-memory-model]] · [[wiedza/01-jezyk/typy-i-pamiec]] · [[wiedza/00-fundament/jdk-jre-jvm]] · [[wiedza/02-jvm/jit]]
