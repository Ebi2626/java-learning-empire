---
temat: "Synchronizacja i java.util.concurrent"
faza: 3
status: nieopanowany
priorytet: 🟡
tags: [java, wspolbieznosc]
powiazane: ["[[wiedza/02-jvm/java-memory-model]]", "[[wiedza/01-jezyk/kolekcje]]", "[[wiedza/03-wspolbieznosc/podstawy]]"]
---

# Synchronizacja i java.util.concurrent

> **TL;DR:** `synchronized` daje wzajemne wykluczanie (mutual exclusion) na **intrinsic lock/monitorze** obiektu i ustanawia relację *happens-before*; `java.util.concurrent` dokłada elastyczniejsze narzędzia — jawne **Lock** (tryLock, fairness, ReadWrite/StampedLock), **atomics** na sprzętowym **CAS** (lock-free), kolekcje współbieżne (**ConcurrentHashMap** — blokada/CAS na kubełku, odczyty bez blokad) oraz synchronizatory (**CountDownLatch/CyclicBarrier/Semaphore/Phaser**). Zasada: **im mniej blokad i im mniejsze ich ziarno, tym lepiej** — a najlepsza blokada to jej brak (atomics, immutability).

## 1. Co — definicje i API

Synchronizacja rozwiązuje dwa problemy naraz: **atomowość** (żeby złożona operacja nie została przerwana w połowie) i **widoczność** (żeby zmiana jednego wątku była widoczna dla innych — patrz [[wiedza/02-jvm/java-memory-model]]).

### `synchronized`
Każdy obiekt w Javie ma **intrinsic lock** (zwany też **monitorem**). `synchronized` pozyskuje ten monitor przy wejściu i zwalnia przy wyjściu (także przez wyjątek).

- **Monitor jest reentrant (wznawialny):** ten sam wątek może wejść w kolejny `synchronized` na tym samym obiekcie bez zakleszczenia siebie (licznik wejść rośnie/maleje). Bez tego rekurencyjne metody synchronizowane blokowałyby się.
- **Na jakim obiekcie?**
  - `synchronized` metoda **instancyjna** → blokada na `this`.
  - `synchronized` metoda **statyczna** → blokada na obiekcie `Klasa.class` (inny monitor niż `this`!).
  - `synchronized (obiekt) { ... }` → blokada na wskazanym obiekcie.
- **Metoda vs blok — ziarno blokady (lock granularity):** metoda synchronizuje całe ciało; blok pozwala zawęzić sekcję krytyczną do minimum. **Nigdy nie blokuj na publicznym/współdzielonym obiekcie** (np. `this`, `String`, cache'owany `Integer`) — obcy kod może pozyskać ten sam monitor i wywołać deadlock. Dobra praktyka: **prywatny, `final` dedykowany lock**.

```java
private final Object lock = new Object();   // dedykowany, prywatny monitor
private int stan;

void update() {
    // praca bez blokady...
    synchronized (lock) {                    // wąska sekcja krytyczna
        stan++;
    }
}
```

### `wait / notify / notifyAll`
Mechanizm oczekiwania na warunek. **Wywoływane MUSZĄ być pod monitorem** tego samego obiektu (inaczej `IllegalMonitorStateException`).

- `wait()` — **zwalnia monitor** i usypia wątek, aż ktoś zrobi `notify`/`notifyAll` (lub przyjdzie interrupt/timeout).
- `notify()` budzi **jeden** losowy czekający wątek; `notifyAll()` budzi **wszystkie**.
- **Guarded block — zawsze w pętli `while`, nigdy `if`:**

```java
synchronized (lock) {
    while (!warunekSpełniony()) {   // pętla, nie if!
        lock.wait();
    }
    // tu warunek na pewno spełniony
}
```

Dlaczego `while`: (1) **spurious wakeups** — JVM/OS może obudzić wątek bez `notify`; (2) między obudzeniem a odzyskaniem monitora inny wątek mógł zmienić warunek. Po obudzeniu trzeba **ponownie sprawdzić** warunek. `notifyAll()` jest **bezpieczniejszy** niż `notify()`, bo `notify()` może obudzić wątek czekający na *inny* warunek (a właściwy pozostanie uśpiony na zawsze — „missed signal / lost wakeup").

### Pakiet `java.util.concurrent.locks`
Jawne blokady dają to, czego nie ma `synchronized`:

- **`ReentrantLock`** — jak monitor, ale z dodatkami:
  - `tryLock()` / `tryLock(timeout, unit)` — pozyskaj **lub się poddaj** (kluczowe przeciw deadlockom).
  - **fairness** (`new ReentrantLock(true)`) — kolejność FIFO zamiast domyślnej „barging" (szybszej, ale głodzącej).
  - `lockInterruptibly()` — oczekiwanie na blokadę można przerwać `interrupt`.
  - **Zawsze `unlock()` w `finally`** (`synchronized` zwalnia sam; jawny lock — nie).

```java
private final ReentrantLock lock = new ReentrantLock();

void bezpiecznie() {
    lock.lock();
    try {
        // sekcja krytyczna
    } finally {
        lock.unlock();          // ZAWSZE w finally
    }
}
```

- **`ReadWriteLock` / `ReentrantReadWriteLock`** — dwie blokady: **read** (współdzielona, wielu czytelników naraz) i **write** (wyłączna, jeden pisarz, blokuje wszystkich). Zysk przy przewadze odczytów nad zapisami.
- **`StampedLock`** (Java 8+) — nie-reentrant, ale ma **optimistic read**: `tryOptimisticRead()` zwraca „stamp" bez blokowania; po odczycie sprawdzamy `validate(stamp)` — jeśli w międzyczasie nikt nie pisał, odczyt był darmowy (zero synchronizacji). Najszybszy przy dominacji odczytów.

```java
long stamp = sl.tryOptimisticRead();
double x = this.x, y = this.y;           // czytamy optymistycznie
if (!sl.validate(stamp)) {                // ktoś pisał? -> pesymistyczny fallback
    stamp = sl.readLock();
    try { x = this.x; y = this.y; }
    finally { sl.unlockRead(stamp); }
}
```

- **`Condition`** — odpowiednik `wait/notify` dla jawnych Lock. Z jednego Lock można utworzyć **wiele Condition** (`lock.newCondition()`) — np. osobny `notFull` i `notEmpty` dla kolejki. To rozwiązuje problem „missed signal": budzisz precyzyjnie tylko czekających na dany warunek (`signalAll`), zamiast wszystkich. Metody: `await()` / `signal()` / `signalAll()` (też pod pozyskanym lockiem).

### Klasy `atomic` (`java.util.concurrent.atomic`)
`AtomicInteger`, `AtomicLong`, `AtomicReference`, `AtomicBoolean` i tablicowe. Dają **atomowe operacje bez blokady** (lock-free) przez sprzętowy **CAS (compare-and-swap)**: `incrementAndGet()`, `compareAndSet(exp, new)`, `getAndUpdate(fn)`, `accumulateAndGet(...)`.

- **`LongAdder` / `LongAccumulator`** — przy **dużej rywalizacji** (contention) na jednym liczniku CAS ciągle zawodzi i się powtarza. `LongAdder` rozprasza zapisy na wiele wewnętrznych komórek (`Cell[]`) i sumuje dopiero w `sum()` — dużo szybszy dla liczników pod obciążeniem (kosztem droższego odczytu i braku pojedynczej atomowej wartości).

### Kolekcje współbieżne
- **`ConcurrentHashMap`** — wysoce skalowalna mapa (patrz sekcja 2). **Nie dopuszcza `null`** (ani klucza, ani wartości) — bo `get`==null byłoby dwuznaczne („brak" vs „null"). Ma **atomowe** `compute`, `computeIfAbsent`, `merge`, `putIfAbsent`.
- **`CopyOnWriteArrayList`** (i `...Set`) — przy każdej modyfikacji kopiuje całą tablicę. Idealna, gdy **dużo odczytów, mało zapisów** (np. lista listenerów). Iteracja jest bez blokad i bez `ConcurrentModificationException` (widzi snapshot).
- **`BlockingQueue`** — kolejka blokująca do wzorca **producent-konsument**: `put` blokuje gdy pełna, `take` blokuje gdy pusta.
  - `ArrayBlockingQueue` — ograniczona (bounded), pojedynczy lock.
  - `LinkedBlockingQueue` — opcjonalnie bounded, oddzielne locki head/tail (lepszy throughput).
- **`ConcurrentLinkedQueue`** — nieograniczona, **lock-free** (Michael-Scott na CAS), nieblokująca — gdy nie potrzeba blokowania.

### Synchronizatory
- **`CountDownLatch`** — **jednorazowa** bramka. `countDown()` zmniejsza licznik, `await()` czeka aż dojdzie do 0. Nie da się zresetować.
- **`CyclicBarrier`** — **wielokrotny** punkt spotkania: N wątków czeka na `await()`, aż zbierze się komplet, po czym rusza dalej (opcjonalna akcja bariery). Reużywalna między „rundami".
- **`Semaphore`** — zarządza N **pozwoleń** (permits): `acquire()` / `release()`. Ogranicza liczbę wątków w sekcji (np. pula połączeń). `Semaphore(1)` ≈ mutex (ale niereentrant i „own"-agnostyczny).
- **`Phaser`** — elastyczny hybryda Latch+Barrier z dynamiczną liczbą uczestników (`register`/`arriveAndAwaitAdvance`) i fazami.
- **`Exchanger`** — punkt wymiany danych między **dwoma** wątkami (każdy oddaje obiekt i dostaje ten od partnera).

## 2. Jak — monitory, CAS, ConcurrentHashMap pod spodem

### Monitor / intrinsic lock
Nagłówek obiektu ma **mark word**. HotSpot optymalizuje blokady wielostopniowo:
1. **Biased locking** (wycofywane w nowszych JDK) — gdy zawsze ten sam wątek.
2. **Lightweight lock** — bez rywalizacji: CAS wskaźnika w mark word (tanie, w user-space).
3. **Heavyweight lock** — przy rywalizacji: „inflacja" do prawdziwego monitora OS (`ObjectMonitor`), wątki idą do kolejki, następuje **park/unpark** przez OS (drogie context switche). Stąd `synchronized` jest tanie bez kontencji, a drogie pod dużą rywalizacją.

`synchronized` daje też **memory barriers**: wejście = *acquire*, wyjście = *release* — to ustanawia *happens-before* (zapisy przed `unlock` widoczne po `lock` — patrz [[wiedza/02-jvm/java-memory-model]]).

### CAS (compare-and-swap) — jak działa lock-free
CAS to atomowa instrukcja CPU (`cmpxchg` na x86): „ustaw pole na `new` **tylko jeśli** aktualnie ma wartość `expected`; zwróć czy się udało". Atomics działają w pętli:

```
do {
    old = get();
    next = old + 1;
} while (!compareAndSet(old, next));   // powtórz jeśli ktoś nas ubiegł
```

Brak blokady = brak usypiania wątków; wątek przegrywający po prostu **próbuje jeszcze raz** (spin). Zaleta: brak deadlocków, brak context-switchy. Wada: pod **dużą rywalizacją** wiele wątków kręci się w pętli (marnują CPU) — wtedy `LongAdder` bije `AtomicLong`.

- **Problem ABA:** wątek widzi `A`, ktoś zmienia `A→B→A`; CAS się powiedzie, choć „między" coś się działo. Rozwiązanie: **`AtomicStampedReference`** (para wartość+stempel/wersja) lub `AtomicMarkableReference`.

### ConcurrentHashMap pod spodem (Java 8+)
- Wewnątrz to **tablica kubełków (bins)**, każdy kubełek to lista lub (przy kolizjach ≥8 i dużej tablicy) **drzewo czerwono-czarne** — jak w `HashMap`.
- **Zapis:** jeśli kubełek jest pusty → wstawienie przez **CAS** (bez blokady). Jeśli kubełek ma już elementy → **`synchronized` na pierwszym węźle tego kubełka** (blokada tylko lokalna, per-bin). Różne kubełki modyfikowane są **równolegle**.
- **Odczyt:** **bez blokad** — pola węzłów są `volatile`, więc `get` widzi spójny stan bez synchronizacji (dlatego świetnie skaluje się przy odczytach).
- **Resize:** wspierany współbieżnie — wątki „pomagają" migrować kubełki (`transfer`), a licznik rozmiaru jest rozproszony jak w `LongAdder` (`CounterCell[]`).
- To dlatego CHM skaluje się liniowo z liczbą rdzeni tam, gdzie `Collections.synchronizedMap` (jeden globalny lock) staje się wąskim gardłem. (Stary CHM z Javy 7 używał segmentów — teraz ziarno to pojedynczy kubełek.)

### False sharing (krótko)
Dwie niezależne zmienne trafiające do **tej samej linii cache (cache line, ~64B)** powodują, że zapis do jednej **unieważnia** linię drugiej w innym rdzeniu — spadek wydajności mimo braku logicznej współpracy. `LongAdder` unika tego paddingiem (`@Contended`). Świadomość problemu wystarczy na start.

## 3. Dlaczego / kiedy — dobór narzędzia, pułapki

**Drabinka doboru (od najlżejszego):**
1. **Nic** — dane immutable / thread-confined (najszybsze i najbezpieczniejsze).
2. **`volatile`** — pojedyncza flaga/publikacja referencji, gdy nie ma złożonej operacji read-modify-write.
3. **Atomic** — licznik / lock-free struktura (CAS). `LongAdder` gdy licznik pod dużym obciążeniem.
4. **Kolekcja współbieżna** — mapa/lista/kolejka współdzielona → CHM/COWList/BlockingQueue zamiast ręcznej synchronizacji.
5. **`synchronized`** — proste sekcje krytyczne, gdy nie potrzeba tryLock/timeout/fairness.
6. **`ReentrantLock` / RW / StampedLock** — gdy potrzeba: timeout, przerywalność, fairness, wielu czytelników, wiele warunków (Condition).

**Kiedy co z locków:**
- Więcej odczytów niż zapisów → `ReadWriteLock`, a przy dominacji odczytów → `StampedLock` (optimistic).
- Musisz się poddać zamiast czekać w nieskończoność → `tryLock(timeout)`.

**Unikanie deadlocku:**
- **Stała, globalna kolejność pozyskiwania blokad** — jeśli każdy wątek bierze locki A, potem B (nigdy B→A), cykl niemożliwy.
- **`tryLock` z timeoutem** — jeśli nie zdobędziesz drugiej blokady, zwolnij pierwszą i spróbuj później (unikasz „hold and wait").
- Trzymaj locki **jak najkrócej**, nie wołaj obcego kodu pod lockiem (ryzyko re-entrant deadlock / inwersji kolejności).

**Pułapki:**
- `wait/notify` z `if` zamiast `while` (spurious wakeups + lost signal). Preferuj wyższe abstrakcje (BlockingQueue, Condition, Latch).
- Blokowanie na `this` / publicznym obiekcie / `String` / cache'owanym `Integer`.
- Zapomniany `unlock()` poza `finally`.
- Blokada statyczna vs instancyjna to **różne monitory** (`Klasa.class` ≠ `this`) — łatwo o iluzję synchronizacji.
- `ConcurrentHashMap`: `get`+`put` to **dwie** operacje (race). Użyj **atomowego** `compute`/`merge`/`putIfAbsent`.
- `size()` na kolekcjach współbieżnych bywa przybliżony/kosztowny.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

**Poprawny licznik: `AtomicInteger` vs `synchronized`.**

```java
// Wersja 1: synchronized — poprawna, blokująca
class LicznikSync {
    private int n = 0;
    synchronized void inc() { n++; }        // n++ to read-modify-write => musi być atomowe
    synchronized int get() { return n; }
}

// Wersja 2: AtomicInteger — poprawna, lock-free (CAS), zwykle szybsza pod kontencją
class LicznikAtomic {
    private final AtomicInteger n = new AtomicInteger();
    void inc() { n.incrementAndGet(); }
    int get() { return n.get(); }
}
// Uwaga: `int n; n++;` bez żadnej synchronizacji to KLASYCZNY BŁĄD — gubi inkrementy (lost update).
```

**Producent-konsument z `BlockingQueue`** (kolejka zdejmuje z głowy problem wait/notify):

```java
BlockingQueue<Zadanie> kolejka = new ArrayBlockingQueue<>(1000);

// Producent
Runnable producent = () -> {
    try {
        while (true) kolejka.put(nastepneZadanie());   // blokuje gdy pełna
    } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
};

// Konsument
Runnable konsument = () -> {
    try {
        while (true) {
            Zadanie z = kolejka.take();                 // blokuje gdy pusta
            przetworz(z);
        }
    } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
};
```

Realnie: pule wątków (`ExecutorService`) mają BlockingQueue w środku; `Semaphore` ogranicza równoległe wywołania do zewnętrznego API (rate limiting); `CountDownLatch` czeka aż wystartują wszystkie serwisy przy bootstrapie; `CyclicBarrier` synchronizuje rundy w symulacji; `ConcurrentHashMap` jako współbieżny cache; `AtomicLong`/`LongAdder` jako metryki/liczniki żądań.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, na jakim obiekcie blokuje `synchronized` (metoda instancyjna vs statyczna vs blok) i czemu dedykowany prywatny lock jest bezpieczniejszy.
- [ ] Wiem, czemu `wait` musi być w pętli `while` (spurious wakeups) i czemu `notifyAll` bezpieczniejszy niż `notify`.
- [ ] Rozumiem CAS i lock-free; potrafię napisać atomowy licznik i wskazać problem ABA oraz rolę `LongAdder`.
- [ ] Wyjaśnię, dlaczego `ConcurrentHashMap` skaluje się (CAS/blokada per-kubełek, odczyty bez blokad, brak null).
- [ ] Dobiorę narzędzie: atomic vs synchronized vs ReentrantLock vs RW/StampedLock vs kolekcja współbieżna.
- [ ] Umiem uniknąć deadlocku (kolejność blokad, tryLock z timeoutem).
- [ ] Rozróżniam synchronizatory: CountDownLatch (jednorazowy) vs CyclicBarrier (wielokrotny) vs Semaphore vs Phaser vs Exchanger.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co to intrinsic lock/monitor i dlaczego jest reentrant?
- [ ] Jak działa CAS na poziomie CPU i czemu daje lock-free?
- [ ] Co dzieje się w `ConcurrentHashMap` przy `put` do pustego vs zajętego kubełka?
- [ ] Czemu `synchronized` bywa tanie, a bywa drogie (lightweight vs heavyweight lock)?
- [ ] Jak `synchronized`/Lock ustanawiają happens-before?
- [ ] Czym różni się StampedLock optimistic read od read locka w ReadWriteLock?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „`synchronized` vs `ReentrantLock` — kiedy który?"
- [ ] „Dlaczego `wait` w pętli, a nie w `if`?"
- [ ] „Jak działa `ConcurrentHashMap` i czemu jest szybsza od `synchronizedMap`?"
- [ ] „Co to CAS i problem ABA?"
- [ ] „`AtomicLong` vs `LongAdder` — kiedy który?"
- [ ] „Jak unikasz deadlocku?"
- [ ] „CountDownLatch vs CyclicBarrier vs Semaphore?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/03-wspolbieznosc.csv`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Na jakim obiekcie blokuje synchronized metoda instancyjna?;Na this (intrinsic lock/monitor tego obiektu).
Na jakim obiekcie blokuje synchronized metoda statyczna?;Na obiekcie Klasa.class (inny monitor niż this).
Co znaczy, że intrinsic lock jest reentrant?;Ten sam wątek może wejść ponownie w synchronized na tym samym obiekcie bez zakleszczenia siebie (licznik wejść).
Czemu blokować na prywatnym final locku, a nie na this?;Obcy kod mógłby pozyskać ten sam publiczny monitor i spowodować deadlock; dedykowany lock jest niedostępny z zewnątrz.
Czym różni się ziarno blokady: metoda vs blok synchronized?;Metoda blokuje całe ciało; blok pozwala zawęzić sekcję krytyczną do minimum (mniejsze ziarno, większa równoległość).
Czemu wait() musi być w pętli while, a nie if?;Przez spurious wakeups oraz zmianę warunku między obudzeniem a odzyskaniem monitora — warunek trzeba sprawdzić ponownie.
Czemu notifyAll jest bezpieczniejszy niż notify?;notify może obudzić wątek czekający na inny warunek (lost/missed signal); notifyAll budzi wszystkich i właściwy sprawdzi warunek.
Gdzie muszą być wołane wait/notify/notifyAll?;Pod pozyskanym monitorem tego samego obiektu, inaczej IllegalMonitorStateException.
Co robi wait() z monitorem?;Zwalnia monitor i usypia wątek do notify/interrupt/timeout, po obudzeniu ponownie pozyskuje monitor.
Czym ReentrantLock góruje nad synchronized?;tryLock (z timeoutem), fairness, lockInterruptibly, wiele Condition — kosztem ręcznego unlock w finally.
Gdzie zwalniać jawny Lock?;Zawsze w bloku finally (lock.unlock()), bo w przeciwieństwie do synchronized nie zwalnia się sam.
Kiedy ReadWriteLock?;Gdy dużo odczytów, mało zapisów — wielu czytelników naraz (read shared), jeden pisarz wyłącznie (write).
Co daje StampedLock optimistic read?;tryOptimisticRead + validate(stamp): odczyt bez blokady, jeśli nikt nie pisał — najszybszy przy dominacji odczytów.
Co to CAS (compare-and-swap)?;Atomowa instrukcja CPU: ustaw pole na new tylko jeśli ma wartość expected; podstawa lock-free (atomics w pętli spin).
Na czym polega problem ABA?;Wartość zmienia się A->B->A, więc CAS się powiedzie mimo zmian; rozwiązanie: AtomicStampedReference (wartość+wersja).
Kiedy LongAdder zamiast AtomicLong?;Przy dużej rywalizacji na liczniku — rozprasza zapisy na wiele komórek (Cell[]) i sumuje w sum(); szybszy zapis, droższy odczyt.
Jak ConcurrentHashMap obsługuje zapis?;Pusty kubełek: wstawienie przez CAS; zajęty: synchronized na pierwszym węźle kubełka — różne kubełki równolegle.
Czemu odczyty w ConcurrentHashMap są bez blokad?;Pola węzłów są volatile, więc get widzi spójny stan bez synchronizacji — stąd skalowalność.
Czemu ConcurrentHashMap nie dopuszcza null?;get()==null byłoby dwuznaczne: brak klucza czy wartość null — więc null jest zakazany.
Kiedy CopyOnWriteArrayList?;Dużo odczytów, mało zapisów (np. listenery); zapis kopiuje całą tablicę, iteracja bez blokad na snapshocie.
CountDownLatch vs CyclicBarrier?;Latch jednorazowy (await czeka aż licznik zejdzie do 0); Barrier wielokrotny punkt spotkania N wątków, reużywalny.
Do czego Semaphore?;Zarządza N pozwoleniami (acquire/release) — ogranicza liczbę wątków w sekcji (np. rate limit / pula).
Jak unikać deadlocku?;Stała globalna kolejność pozyskiwania blokad oraz tryLock z timeoutem (zwolnij i ponów zamiast hold-and-wait).
Co to false sharing?;Niezależne zmienne w tej samej linii cache (~64B) wzajemnie unieważniają ją między rdzeniami; padding (@Contended) temu zapobiega.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/02-jvm/java-memory-model]] · [[wiedza/01-jezyk/kolekcje]] · [[wiedza/03-wspolbieznosc/podstawy]] · [[wiedza/03-wspolbieznosc/executors]]
