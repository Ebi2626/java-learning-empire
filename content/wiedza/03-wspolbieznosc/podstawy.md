---
temat: "Podstawy współbieżności"
faza: 3
status: nieopanowany
priorytet: 🟡
tags: [java, wspolbieznosc]
powiazane: ["[[wiedza/03-wspolbieznosc/synchronizacja]]", "[[wiedza/03-wspolbieznosc/executors]]", "[[wiedza/03-wspolbieznosc/virtual-threads]]", "[[wiedza/02-jvm/java-memory-model]]"]
---

# Podstawy współbieżności

> **TL;DR:** **Wątki** żyją w jednym procesie i **współdzielą heap** (stąd race condition), a JVM zleca ich
> harmonogramowanie **OS-owi** (preemptive, brak gwarancji kolejności). Zadania definiujesz przez `Runnable`/`Callable`,
> uruchamiasz na `Thread`; **interrupt jest kooperacyjny** (ustawia flagę), a klasyczne bolączki to
> **race condition, deadlock, livelock, starvation**. Wątek platformowy to koszt (~1 MB stosu + context switch).

## 1. Co — proces vs wątek, po co współbieżność

**Proces** to izolowana instancja programu z własną przestrzenią adresową (osobny heap, deskryptory, uprawnienia).
**Wątek (thread)** to lekka jednostka wykonania *wewnątrz* procesu. Kluczowa różnica:

- **Procesy są izolowane** — komunikacja tylko przez IPC (pipe, socket, shared memory jawnie mapowany). Awaria jednego nie kładzie drugiego.
- **Wątki współdzielą pamięć** — ten sam **heap** i te same obiekty. Każdy wątek ma tylko **własny stos** (zmienne lokalne, ramki wywołań) i program counter. To współdzielenie daje wydajną komunikację, ale jest **źródłem race conditions** — patrz [[wiedza/02-jvm/java-memory-model]].

```
Proces JVM
├── Heap (WSPÓŁDZIELONY: obiekty, static, tablice)   ← tu powstają wyścigi
├── Metaspace (metadane klas)
└── Wątki:
    ├── Thread-1: [własny stos] [PC] [thread-locals]
    ├── Thread-2: [własny stos] [PC] [thread-locals]
    └── ...
```

**Po co współbieżność:**
- **Przepustowość (throughput)** — obsłuż wiele żądań naraz zamiast po kolei (serwer HTTP).
- **Responsywność** — nie blokuj UI/głównego wątku podczas długiej operacji I/O.
- **Wykorzystanie wielu rdzeni** — pojedynczy wątek liczy na jednym core; żeby użyć N rdzeni, potrzeba równoległości.

**Współbieżność ≠ równoległość (concurrency vs parallelism):**
- **Concurrency** = *struktura* — wiele zadań „w toku" w tym samym okresie, mogą przeplatać się na jednym rdzeniu (interleaving). Dotyczy *radzenia sobie* z wieloma rzeczami naraz.
- **Parallelism** = *wykonanie* — zadania fizycznie liczone jednocześnie na wielu rdzeniach. Dotyczy *robienia* wielu rzeczy naraz.
- Program może być współbieżny bez równoległości (jeden rdzeń, przeplatanie) i odwrotnie rzadko ma sens. Współbieżność ma sens także dla I/O-bound (czekanie), nie tylko CPU-bound.

## 2. Jak — cykl życia, scheduling, przerwania

### Tworzenie zadań: Thread vs Runnable vs Callable

- **`Runnable`** — `void run()`, nie zwraca wyniku i nie może rzucać checked exception. Preferowany sposób opisania zadania (rozdziela *co* robić od *jak* uruchomić).
- **`Callable<V>`** — `V call() throws Exception`, **zwraca wynik i może rzucić wyjątek**. Używany z `ExecutorService.submit()` → daje `Future<V>`. Zobacz [[wiedza/03-wspolbieznosc/executors]].
- **`Thread`** — sama *jednostka wykonania*. Dziedziczenie po `Thread` jest antywzorcem (sztywno wiąże zadanie z mechanizmem); przekazuj `Runnable` do konstruktora.

```java
Runnable r = () -> System.out.println("hej z " + Thread.currentThread().getName());
Thread t = new Thread(r);
t.start();   // NIE t.run() — run() wykona się w bieżącym wątku, bez nowego wątku!
```

> `start()` tworzy nowy wątek OS i wywołuje w nim `run()`. `run()` bezpośrednio to zwykłe wywołanie metody — synchronicznie, bez współbieżności.

### Cykl życia wątku (`Thread.State`)

```
         start()                       scheduler
  NEW ───────────► RUNNABLE ◄───────────────────────┐
                    │  │  │                          │
       lock zajęty  │  │  └─ wait()/join()/park() ──► WAITING ──── notify()/unpark()
                    │  └──── sleep(t)/wait(t)/join(t) ► TIMED_WAITING ── timeout
      synchronized ─┘                                 │
                    ▼                                  │
                 BLOCKED ──── zdobył monitor ──────────┘
                    │
           run() koniec / wyjątek
                    ▼
               TERMINATED
```

- **NEW** — utworzony (`new Thread(...)`), jeszcze nie `start()`.
- **RUNNABLE** — gotowy do działania lub aktualnie działa (JVM nie rozróżnia „running"; to widok logiczny — o rdzeń decyduje OS). Uwaga: wątek blokujący się na I/O sieciowym w JVM zwykle nadal jest RUNNABLE (JVM nie widzi bloku na poziomie OS).
- **BLOCKED** — czeka na wejście do bloku `synchronized` (monitor zajęty przez inny wątek).
- **WAITING** — czeka bezterminowo: `Object.wait()`, `Thread.join()`, `LockSupport.park()`. Wybudza go zdarzenie (`notify`/`unpark`).
- **TIMED_WAITING** — jak WAITING, ale z limitem czasu: `sleep(ms)`, `wait(ms)`, `join(ms)`.
- **TERMINATED** — `run()` zakończył się (normalnie lub wyjątkiem). Wątku nie da się „zrestartować".

### Harmonogramowanie (scheduling)

Wątki platformowe JVM to cienka nakładka na wątki OS. Harmonogram jest **preemptive** i **zarządzany przez OS**:
scheduler może **wywłaszczyć** wątek w dowolnym momencie i przydzielić rdzeń innemu.

- **Brak gwarancji kolejności ani sprawiedliwości** — nie zakładaj, w jakiej kolejności wykonają się wątki ani że każdy dostanie czas. `Thread.setPriority()` to tylko *podpowiedź*, którą OS może zignorować.
- Poprawność programu **nie może zależeć** od czasu ani szybkości wątków — musi wynikać z synchronizacji ([[wiedza/03-wspolbieznosc/synchronizacja]]).

### Wątki daemon vs user

- **User thread** — trzyma JVM przy życiu. `main` jest user thread.
- **Daemon thread** — „tło" (np. GC, timery). **JVM kończy działanie, gdy zostają tylko wątki daemon** — są wtedy ubijane bez `finally`/cleanup, więc nie licz na dokończenie ich pracy.
- Ustawiasz `t.setDaemon(true)` **przed** `start()`.

### Model przerwań (interruption) — kooperacyjny

Java **nie ma bezpiecznego „force kill" wątku** (`stop()` jest deprecated — zostawia niespójny stan). Zamiast tego jest
**kooperacyjny interrupt**:

- `t.interrupt()` **tylko ustawia flagę** (interrupt status). Wątek sam musi ją sprawdzać i zareagować.
- Metody blokujące (`sleep`, `wait`, `join`, blokady w `java.util.concurrent`) reagują, rzucając **`InterruptedException`** i **czyszcząc flagę**.
- Kod czysto obliczeniowy sprawdza flagę sam: `Thread.currentThread().isInterrupted()`.

**Jak poprawnie obsłużyć** (dwie poprawne opcje):
```java
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // 1) przywróć flagę i zakończ/propaguj
    return;
}
```
- **Nie połykaj** wyjątku (`catch (InterruptedException e) { }`) — gubisz sygnał anulowania.
- Skoro `InterruptedException` czyści flagę, albo **przywróć ją** (`interrupt()`), albo **propaguj wyjątek** wyżej. Nigdy jedno i drugie zignorowane.

### sleep / join / yield

- **`Thread.sleep(ms)`** — usypia bieżący wątek na czas, **nie zwalnia** trzymanych locków. Stan TIMED_WAITING.
- **`t.join()`** — bieżący wątek czeka, aż `t` się zakończy (WAITING/TIMED_WAITING). Do „poczekaj na wynik".
- **`Thread.yield()`** — tylko *sugestia* dla schedulera „mogę oddać rdzeń". Bez gwarancji, rzadko potrzebne.

## 3. Dlaczego / kiedy — problemy współbieżności i thread-safety

### Problemy

- **Race condition** — poprawność zależy od *względnego czasu/kolejności* wątków; niektóre przeploty dają zły wynik (błąd logiki, np. check-then-act, read-modify-write).
- **Data race** — dwa wątki dostępują do tej samej zmiennej, co najmniej jeden pisze, **bez synchronizacji/happens-before**. To węższe, formalne pojęcie z modelu pamięci → [[wiedza/02-jvm/java-memory-model]]. Data race może dać nie tylko zły wynik, ale i widoczność „nieaktualnych" wartości oraz reorderowanie.
- **Deadlock** — wątki wzajemnie blokują się w cyklu i żaden nie postępuje. Powstaje, gdy zachodzą **wszystkie 4 warunki Coffmana**:
  1. **Mutual exclusion** — zasób trzymany na wyłączność.
  2. **Hold and wait** — trzymam jeden zasób i czekam na kolejny.
  3. **No preemption** — zasobu nie da się odebrać siłą.
  4. **Circular wait** — istnieje cykl oczekiwań (A czeka na B, B na A).
  Złam którykolwiek → nie ma deadlocku. Najpraktyczniej łamie się **circular wait** przez globalną **kolejność zajmowania locków**.
- **Livelock** — wątki *nie są zablokowane*, ciągle coś robią, ale w kółko reagują na siebie i nie postępują (jak dwie osoby ustępujące sobie w korytarzu).
- **Starvation** — wątek nigdy nie dostaje potrzebnego zasobu/czasu CPU (np. przez niesprawiedliwy lock albo ciągły napływ wątków o wyższym priorytecie).

### Thread-safety i jej poziomy

**Thread-safe** = klasa/kod zachowuje się poprawnie przy dostępie z wielu wątków bez dodatkowej synchronizacji ze strony klienta. Praktyczne poziomy (klasyfikacja m.in. z JCiP):

- **Immutable** — obiekt niezmienny po utworzeniu (np. `String`, `record` z niemutowalnymi polami). Zawsze thread-safe bez żadnej synchronizacji.
- **Thread-safe (bezwarunkowo)** — poprawny bez zewnętrznej synchronizacji (`ConcurrentHashMap`, `AtomicInteger`).
- **Conditionally thread-safe (warunkowo bezpieczny)** — pojedyncze operacje bezpieczne, ale sekwencje wymagają synchronizacji przez klienta (np. iteracja po `Collections.synchronizedList` wymaga blokady na kolekcji).
- **Not thread-safe (niebezpieczny)** — wymaga pełnej synchronizacji zewnętrznej (`ArrayList`, `HashMap`, `SimpleDateFormat`).

### Koszt wątków platformowych

- **Stos** — każdy platform thread rezerwuje stos rzędu **~1 MB** (`-Xss`). 10 000 wątków ≈ ~10 GB samych stosów → OOM.
- **Context switch** — przełączenie wątku przez OS kosztuje (zapis/odczyt rejestrów, unieważnienie cache, TLB). Za dużo wątków = więcej czasu na przełączanie niż na pracę.
- **Limit liczby wątków** — ograniczony przez pamięć i limity OS. Model „thread-per-request" na blokującym I/O nie skaluje się do dziesiątek tysięcy połączeń.
- To motywacja dla **[[wiedza/03-wspolbieznosc/virtual-threads]]** (Java 21, JEP 444): tanie wątki wirtualne (stos na heapie, montowane na garści carrier threads) — thread-per-request bez kosztu platform threads.

## Przykład w praktyce

**Race condition — nieatomowy licznik** (klasyka: read-modify-write nie jest niepodzielny):
```java
class Counter {
    private int count = 0;
    void increment() { count++; }   // = tymczas = count; tymczas+1; count = tymczas  (3 kroki!)
    int get() { return count; }
}
// 2 wątki po 1_000_000 increment() → wynik ZWYKLE < 2_000_000: zgubione aktualizacje.
// Fix: AtomicInteger / synchronized / lock → [[wiedza/03-wspolbieznosc/synchronizacja]]
```

**Deadlock — dwa locki w odwrotnej kolejności:**
```java
Object A = new Object(), B = new Object();

// Wątek 1
synchronized (A) {
    synchronized (B) { /* ... */ }   // trzyma A, chce B
}

// Wątek 2
synchronized (B) {
    synchronized (A) { /* ... */ }   // trzyma B, chce A  → circular wait → DEADLOCK
}
// Fix: zawsze zajmuj A przed B (globalna kolejność locków → łamie warunek 4 Coffmana).
```

Realnie: warstwa serwisowa, gdzie dwie metody przelewają środki między kontami i lockują konta w kolejności ich wystąpienia w argumentach — `transfer(a,b)` i `transfer(b,a)` deadlockują. Fix: lockuj konta w kolejności ich `id`.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię różnicę proces vs wątek i co dokładnie wątki współdzielą (heap), a co mają własne (stos).
- [ ] Odróżnię concurrency od parallelism jednym zdaniem.
- [ ] Naszkicuję cykl życia wątku i wskażę, co powoduje każde przejście.
- [ ] Wiem, czym różni się `Runnable` od `Callable` i dlaczego `start()` ≠ `run()`.
- [ ] Poprawnie obsłużę `InterruptedException` (nie połykać, przywrócić flagę).
- [ ] Wymienię 4 warunki Coffmana i sposób złamania deadlocku.
- [ ] Podam poziomy thread-safety i koszt platform threads.

### 🔲 Black-box check
- [ ] Dlaczego `count++` z wielu wątków gubi aktualizacje? (read-modify-write nie jest atomowy)
- [ ] Czym różni się race condition od data race?
- [ ] Co realnie robi `interrupt()` i czemu jest „kooperacyjny"?
- [ ] Czemu wątek na blokującym I/O bywa w stanie RUNNABLE, a nie BLOCKED?
- [ ] Kiedy JVM się kończy w kontekście wątków daemon vs user?
- [ ] Czy `sleep()` zwalnia locki? (nie — w przeciwieństwie do `wait()`)

### 🎤 Pytania rekrutacyjne
- [ ] „Współbieżność a równoległość — różnica?"
- [ ] „Czym jest deadlock i jak go zapobiegać?" (4 warunki, kolejność locków)
- [ ] „Jak poprawnie anulować/przerwać wątek?" (interrupt, nie stop())
- [ ] „Co się stanie, jak wywołasz `run()` zamiast `start()`?"
- [ ] „Co znaczy, że klasa jest thread-safe? Podaj poziomy."
- [ ] „Dlaczego platform threads są drogie i co je zastępuje w Java 21?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/03-wspolbieznosc.csv`.*
```
Proces vs wątek — pamięć?;Procesy izolowane (osobny heap, IPC); wątki współdzielą heap tego samego procesu, mają tylko własny stos i PC.
Współbieżność vs równoległość?;Concurrency = struktura/przeplatanie wielu zadań w tym samym okresie; parallelism = fizyczne wykonanie naraz na wielu rdzeniach.
Po co współbieżność?;Przepustowość (throughput), responsywność (nie blokować UI/I/O), wykorzystanie wielu rdzeni.
start() vs run()?;start() tworzy nowy wątek i wywołuje w nim run(); run() bezpośrednio to zwykłe synchroniczne wywołanie w bieżącym wątku.
Runnable vs Callable?;Runnable: void run(), bez wyniku i bez checked exception; Callable<V>: V call() throws Exception — zwraca wynik i może rzucić wyjątek.
Stany wątku w Javie?;NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED.
Kiedy wątek jest BLOCKED?;Gdy czeka na wejście do bloku synchronized (monitor zajęty przez inny wątek).
WAITING vs TIMED_WAITING?;WAITING — czekanie bezterminowe (wait/join/park); TIMED_WAITING — z limitem czasu (sleep(t)/wait(t)/join(t)).
Jaki jest scheduler wątków w JVM?;Preemptive, zarządzany przez OS — brak gwarancji kolejności i sprawiedliwości; priority to tylko podpowiedź.
Wątek daemon vs user?;JVM kończy się, gdy zostają tylko wątki daemon (ubijane bez cleanup); user thread trzyma JVM przy życiu.
Co robi interrupt()?;Ustawia flagę interrupt status (kooperacyjnie); metody blokujące rzucają InterruptedException i czyszczą flagę.
Jak poprawnie obsłużyć InterruptedException?;Nie połykać: albo przywrócić flagę (Thread.currentThread().interrupt()), albo propagować wyjątek wyżej.
Czy sleep() zwalnia locki?;Nie — sleep() usypia wątek, ale trzyma monitory; w przeciwieństwie do wait(), które zwalnia monitor.
Co robi Thread.join()?;Bieżący wątek czeka, aż docelowy wątek się zakończy.
Race condition vs data race?;Race condition — poprawność zależy od kolejności wątków; data race — nieusynchronizowany dostęp do zmiennej z zapisem (formalnie, z modelu pamięci).
Dlaczego count++ nie jest atomowy?;To read-modify-write (odczyt, +1, zapis) — trzy kroki, które inny wątek może przeplecieć → zgubione aktualizacje.
4 warunki Coffmana deadlocku?;Mutual exclusion, hold and wait, no preemption, circular wait — złamanie któregokolwiek eliminuje deadlock.
Jak najłatwiej zapobiec deadlockowi?;Złamać circular wait — narzucić globalną kolejność zajmowania locków.
Livelock vs starvation?;Livelock — wątki działają, ale w kółko reagują na siebie bez postępu; starvation — wątek nigdy nie dostaje zasobu/CPU.
Poziomy thread-safety?;Immutable, thread-safe (bezwarunkowo), conditionally thread-safe (wymaga synchronizacji klienta), not thread-safe.
Koszt platform thread?;Stos ~1 MB, koszt context switchu, ograniczony limit liczby wątków — stąd virtual threads w Java 21.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/03-wspolbieznosc/synchronizacja]] · [[wiedza/03-wspolbieznosc/executors]] · [[wiedza/03-wspolbieznosc/virtual-threads]] · [[wiedza/02-jvm/java-memory-model]]
