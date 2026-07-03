---
temat: "Executor Framework i pule wątków"
faza: 3
status: nieopanowany
priorytet: 🟡
tags: [java, wspolbieznosc]
powiazane: ["[[wiedza/03-wspolbieznosc/completablefuture]]", "[[wiedza/03-wspolbieznosc/virtual-threads]]", "[[wiedza/02-jvm/model-pamieci]]"]
---

# Executor Framework i pule wątków

> **TL;DR:** **Executor Framework** oddziela *co* uruchomić (`Runnable`/`Callable`) od *jak* to wykonać (pula wątków).
> Zamiast `new Thread()` na żądanie używasz **`ThreadPoolExecutor`**, który **reużywa** wątki, **ogranicza** ich liczbę
> i zarządza cyklem życia. Klucz to algorytm przyjmowania zadania (**core → queue → max → reject**) oraz świadomy dobór
> `corePoolSize` / `maximumPoolSize` / `workQueue`. Fabryki `Executors` są wygodne, ale mają **pułapki** (nieograniczona kolejka → OOM,
> nieograniczona liczba wątków → wyczerpanie). W Javie 21 dla zadań IO-bound rozważ zamiast pulowania **virtual threads**.

## 1. Co — definicja i API

**Problem, który to rozwiązuje.** Tworzenie `new Thread(r).start()` na każde żądanie jest złe z trzech powodów:
- **Brak reużycia** — start i teardown wątku OS jest kosztowny (alokacja stosu ~1 MB, syscalle, praca schedulera). Przy dużym ruchu narzut startu wątku bywa większy niż samo zadanie.
- **Nieograniczona liczba wątków** — pod obciążeniem powstaje ich tysiące. Każdy to pamięć (stosy) + rywalizacja o CPU (**context switching** rośnie kwadratowo). Efekt: `OutOfMemoryError: unable to create new native thread` albo *thrashing*.
- **Brak zarządzania cyklem życia** — nie masz wspólnego punktu do zamknięcia, kolejkowania, obsługi przeciążenia, nazewnictwa wątków ani metryk.

**Rozwiązanie: pula wątków** — stały (lub ograniczony) zbiór wątków-robotników, które w pętli pobierają zadania z kolejki. Wątki żyją długo i są reużywane.

**Hierarchia interfejsów:**
```
Executor                       execute(Runnable)  — najprostszy: "wykonaj to"
   └── ExecutorService         submit()→Future, invokeAll, shutdown, awaitTermination — cykl życia + wynik
          └── ScheduledExecutorService   schedule(), scheduleAtFixedRate(), scheduleWithFixedDelay() — opóźnienia/cykliczność
```

```java
Executor exec = Executors.newFixedThreadPool(4);
exec.execute(() -> System.out.println("fire and forget"));

ExecutorService es = Executors.newFixedThreadPool(4);
Future<Integer> f = es.submit(() -> 40 + 2);   // Callable → Future
Integer wynik = f.get();                        // blokuje aż do wyniku
es.shutdown();
```

- **`Executor`** — 1 metoda `execute(Runnable)`. Sama abstrakcja "oddziel zgłoszenie od wykonania".
- **`ExecutorService`** — dodaje `submit()` (zwraca `Future`), `invokeAll`/`invokeAny`, oraz zarządzanie: `shutdown`, `shutdownNow`, `awaitTermination`, `isShutdown`, `isTerminated`. Implementuje `AutoCloseable` (Java 19+ → można w try-with-resources; `close()` = `shutdown()` + blokujące `awaitTermination`).
- **`ScheduledExecutorService`** — zadania z opóźnieniem i cykliczne.

## 2. Jak — algorytm ThreadPoolExecutor pod spodem

Prawie wszystkie fabryki `Executors` zwracają `ThreadPoolExecutor` (TPE). Jego pełny konstruktor:

```java
new ThreadPoolExecutor(
    int corePoolSize,                    // rdzeń — wątki trzymane "na stałe"
    int maximumPoolSize,                 // twardy limit liczby wątków
    long keepAliveTime, TimeUnit unit,   // jak długo bezczynny wątek POWYŻEJ core żyje
    BlockingQueue<Runnable> workQueue,   // kolejka zadań oczekujących
    ThreadFactory threadFactory,         // jak tworzyć wątki (nazwa, daemon, priorytet, UncaughtExceptionHandler)
    RejectedExecutionHandler handler);   // co zrobić, gdy nie da się przyjąć zadania
```

### Parametry i jak WSPÓŁGRAJĄ
- **`corePoolSize`** — liczba wątków utrzymywanych nawet gdy są bezczynne (chyba że `allowCoreThreadTimeOut(true)`). Pula startuje **pusta** i tworzy wątki *leniwie* w miarę napływu zadań (albo `prestartAllCoreThreads()`).
- **`maximumPoolSize`** — maksimum wątków. Pole między core a max to wątki "awaryjne", tworzone tylko gdy **kolejka jest pełna**.
- **`keepAliveTime`** — czas, po którym bezczynny wątek *powyżej* core jest zwijany, żeby pula skurczyła się z powrotem do core po ustaniu piku.
- **`workQueue`** — bufor zadań między `core` a `max`. Jej *rodzaj* radykalnie zmienia zachowanie puli (patrz niżej).
- **`threadFactory`** — kontrola nad tworzonymi wątkami: **nadaj sensowne nazwy** (bezcenne w thread dumpach), ustaw `daemon`, `UncaughtExceptionHandler`. Domyślna fabryka daje bezużyteczne `pool-1-thread-3`.
- **`RejectedExecutionHandler`** — polityka odrzucenia, gdy pula jest nasycona (max wątków + pełna kolejka) albo po `shutdown`.

### DOKŁADNY algorytm przyjmowania zadania (kluczowy black-box)
Gdy wołasz `execute(task)`, TPE decyduje w tej kolejności:
```
1. Jeśli liczba działających wątków < corePoolSize:
        → utwórz NOWY wątek (core) i uruchom w nim to zadanie.        [preferuj nowy wątek]
2. W przeciwnym razie spróbuj WŁOŻYĆ zadanie do workQueue:
        → jeśli offer() się powiódł → zostanie odebrane przez wolny wątek.
3. Jeśli kolejka PEŁNA (offer() zwrócił false):
        → jeśli liczba wątków < maximumPoolSize → utwórz NOWY wątek (poza-core) na to zadanie.
        → w przeciwnym razie → ODRZUĆ przez RejectedExecutionHandler.
```
**Konsekwencja nieintuicyjna:** wątki *ponad* `core` powstają dopiero **po zapełnieniu kolejki**, nie gdy core są zajęte.
Dlatego **nieograniczona kolejka (`LinkedBlockingQueue` bez limitu) sprawia, że `maximumPoolSize` jest martwe** — krok 3 nigdy nie zajdzie, pula nigdy nie urośnie ponad core.

### Rodzaje kolejek (`workQueue`)
- **`SynchronousQueue`** — pojemność 0, "przekazanie z rąk do rąk". `offer()` udaje się tylko, gdy jest wątek gotowy odebrać. Efekt: krok 2 prawie zawsze pada → pula szybko rośnie do `max`. Używane w `newCachedThreadPool`.
- **`LinkedBlockingQueue` (nieograniczona)** — domyślnie `Integer.MAX_VALUE`. Krok 2 zawsze się udaje → pula nie rośnie ponad core, a zadania piętrzą się w pamięci → **ryzyko `OutOfMemoryError`**. Używane w `newFixedThreadPool`/`newSingleThreadExecutor`.
- **`ArrayBlockingQueue` (ograniczona)** — bufor o stałej pojemności. Jedyny wariant, w którym cały algorytm (core → queue → max → reject) działa jak w podręczniku. **Zalecany do własnych pul** — daje *backpressure*.

### RejectedExecutionHandler — polityki
- **`AbortPolicy`** (domyślna) — rzuca `RejectedExecutionException`. Głośna porażka; woła o obsługę.
- **`CallerRunsPolicy`** — zadanie wykonuje **wątek zgłaszający** (np. thread requestu). To naturalny **backpressure**: producent zwalnia, bo sam się zajął pracą. Świetne przy przeciążeniu.
- **`DiscardPolicy`** — po cichu wyrzuca nowe zadanie. Niebezpieczne (gubi dane bez śladu).
- **`DiscardOldestPolicy`** — usuwa najstarsze zadanie z kolejki i ponawia próbę dodania nowego.

## 3. Dlaczego / kiedy — dobór puli i pułapki fabryk

### Fabryki Executors i ich PUŁAPKI
| Fabryka | Konfiguracja | Pułapka |
|---|---|---|
| `newFixedThreadPool(n)` | core = max = n, `LinkedBlockingQueue` nieograniczona | Kolejka rośnie bez limitu → **OOM** przy zalewie zadań; brak backpressure. |
| `newCachedThreadPool()` | core = 0, max = `Integer.MAX_VALUE`, `SynchronousQueue`, keepAlive 60 s | Pula rośnie **bez granic** → tysiące wątków → **wyczerpanie** pamięci/CPU. |
| `newSingleThreadExecutor()` | 1 wątek, `LinkedBlockingQueue` nieograniczona | Serializacja zadań (przydatne), ale ta sama nieograniczona kolejka → OOM. |
| `newScheduledThreadPool(n)` | pula + `DelayedWorkQueue` | Wyjątek w `scheduleAtFixedRate` **po cichu zabija** dalsze wykonania danego zadania — łap wyjątki w środku. |

**Rada Josha Blocha / "Effective Java" i Brian Goetza:** w produkcji **konfiguruj własny `ThreadPoolExecutor`** zamiast fabryk — ograniczona kolejka + jawna polityka odrzuceń + nazwana `ThreadFactory`.

### execute vs submit
- **`execute(Runnable)`** — nic nie zwraca; wyjątek z zadania trafia do `UncaughtExceptionHandler` wątku.
- **`submit(Callable/Runnable)`** — zwraca **`Future`**; wyjątek jest **przechwytywany** i schowany w `Future` (wychodzi dopiero z `get()` jako `ExecutionException`). **Pułapka:** `submit(runnable)` bez sprawdzenia `Future` **połyka wyjątki** po cichu.

### Future.get
- **Blokujące** — czeka na wynik. Wariant `get(timeout, unit)` rzuca `TimeoutException`.
- Wyjątki: `ExecutionException` (opakowuje wyjątek zadania — prawdziwa przyczyna w `getCause()`), `InterruptedException`, `CancellationException`.

### Poprawne zamykanie
```java
es.shutdown();                                   // przestań przyjmować NOWE; dokończ trwające i z kolejki
if (!es.awaitTermination(30, TimeUnit.SECONDS))  // czekaj z limitem
    es.shutdownNow();                            // przerwij trwające (interrupt) + zwróć nieuruchomione zadania
```
- **`shutdown()`** — łagodne: kończy to, co przyjęte; odrzuca nowe.
- **`shutdownNow()`** — agresywne: `interrupt()` na wątkach robotników, opróżnia kolejkę i zwraca listę nieuruchomionych zadań. Działa tylko, jeśli zadania **reagują na interrupt**.
- **`awaitTermination`** — blokuje do faktycznego wygaszenia (albo timeout). Sam `shutdown()` nie czeka.

### ForkJoinPool i common pool
- **`ForkJoinPool`** — pula do dziel-i-rządź z **work-stealing**: każdy wątek ma własną deque; bezczynny "kradnie" zadania z końca deque zajętego. Minimalizuje rywalizację i wyrównuje obciążenie.
- **Common pool** (`ForkJoinPool.commonPool()`, domyślnie ~`rdzenie − 1` wątków) jest **współdzielony** przez `parallelStream()` i domyślnie przez `CompletableFuture` (metody bez podanego executora). **Uwaga:** operacja **blokująca** (IO, `sleep`, blokujący `get`) na common pool **zagładza** wszystkie inne parallel streamy/CompletableFuture w JVM. Do zadań blokujących zawsze podawaj **własny executor**.

### SIZING puli — ile wątków?
- **CPU-bound** (obliczenia, minimum IO): `N ≈ liczba_rdzeni + 1`. Więcej wątków niż rdzeni tylko dokłada context switchów.
- **IO-bound** (bazy, sieć, dysk): **znacznie więcej** niż rdzeni — wątki i tak głównie czekają.
- **Wzór (Brian Goetz, "Java Concurrency in Practice", pochodna prawa Little'a):**
```
N_threads = N_cpu * U_cpu * (1 + W/C)
   N_cpu — liczba rdzeni (Runtime.getRuntime().availableProcessors())
   U_cpu — docelowe wykorzystanie CPU (0..1)
   W/C   — stosunek czasu oczekiwania (Wait) do czasu obliczeń (Compute)
```
Przy czystym CPU (W/C = 0, U = 1) wychodzi ~`N_cpu`. Przy IO gdzie czekamy 9× dłużej niż liczymy (W/C = 9) — ~10× więcej wątków.

### Wpływ virtual threads (Java 21)
**Virtual threads** ([[wiedza/03-wspolbieznosc/virtual-threads]]) wywracają model: są tak tanie, że dla zadań **IO-bound stosuje się je zamiast pulowania** — `Executors.newVirtualThreadPerTaskExecutor()` tworzy **nowy wątek na zadanie**, bez kolejki i limitu. Reguła: **virtual threads się NIE pooluje** (pool istnieje po to, by reużywać drogie zasoby; virtual thread nie jest drogi). Klasyczne pule platform threads zostają sensowne dla zadań **CPU-bound** (tam limit = rdzenie ma sens) i jako **limiter zasobów** (np. semafor na liczbę równoległych połączeń).

## Przykład w praktyce

Własny `ThreadPoolExecutor` zamiast fabryki — dla puli obsługującej wywołania zewnętrznego API (IO-bound), z backpressure i sensownym nazewnictwem:

```java
ThreadFactory tf = Thread.ofPlatform()
        .name("api-worker-", 0)                 // api-worker-0, api-worker-1, ...
        .daemon(true)
        .factory();

ExecutorService pool = new ThreadPoolExecutor(
        10,                                      // corePoolSize
        50,                                      // maximumPoolSize
        60L, TimeUnit.SECONDS,                   // keepAlive dla wątków poza core
        new ArrayBlockingQueue<>(200),           // OGRANICZONA kolejka → backpressure, brak OOM
        tf,
        new ThreadPoolExecutor.CallerRunsPolicy()); // przeciążenie spowalnia producenta

try {
    Future<String> f = pool.submit(() -> callExternalApi());
    String body = f.get(5, TimeUnit.SECONDS);    // timeout, nie wieczne blokowanie
} catch (ExecutionException e) {
    log.error("zadanie rzuciło", e.getCause());  // prawdziwa przyczyna w getCause()
} finally {
    pool.shutdown();
    if (!pool.awaitTermination(30, TimeUnit.SECONDS)) pool.shutdownNow();
}
```
Spotkasz to w warstwie integracji (klienci HTTP, migracje wsadowe), w kontenerach ustawiając rozmiar puli względem `availableProcessors()` (uwaga na cgroup limity — [[wiedza/02-jvm/model-pamieci]]).

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię trzy powody, dla których `new Thread()` na żądanie jest złe (reużycie, limit, cykl życia).
- [ ] Odtworzę z głowy algorytm przyjmowania zadania: core → queue → max → reject.
- [ ] Wiem, czemu nieograniczona kolejka czyni `maximumPoolSize` martwym.
- [ ] Znam pułapki `newFixedThreadPool` (OOM) i `newCachedThreadPool` (wyczerpanie wątków).
- [ ] Skonfiguruję własny `ThreadPoolExecutor` z ograniczoną kolejką i polityką odrzuceń.
- [ ] Dobiorę rozmiar puli dla zadania CPU-bound i IO-bound (wzór Goetza).
- [ ] Poprawnie zamknę pulę (`shutdown` + `awaitTermination` + `shutdownNow`).
- [ ] Wiem, czemu nie blokować na common pool i czemu nie pulować virtual threads.

### 🔲 Black-box check
- [ ] Co dokładnie robi `execute(task)` krok po kroku wewnątrz TPE?
- [ ] Kiedy powstaje wątek "poza core" — i dlaczego dopiero wtedy?
- [ ] Czym różni się `SynchronousQueue` od `LinkedBlockingQueue` w kontekście wzrostu puli?
- [ ] Gdzie ląduje wyjątek z zadania w `execute` vs `submit`?
- [ ] Co robi `shutdownNow`, jeśli zadanie ignoruje interrupt?
- [ ] Dlaczego blokujący `get()` w `CompletableFuture` bez własnego executora jest groźny?
- [ ] Jak `work-stealing` w `ForkJoinPool` wyrównuje obciążenie?

### 🎤 Pytania rekrutacyjne
- [ ] „Czemu nie tworzyć wątku na każde żądanie?"
- [ ] „Wyjaśnij parametry `ThreadPoolExecutor` i jak ze sobą współgrają."
- [ ] „Dlaczego `newFixedThreadPool` może wywołać OOM?"
- [ ] „`execute` vs `submit` — różnica i obsługa wyjątków?"
- [ ] „Jak dobrać liczbę wątków w puli?"
- [ ] „Co to common pool i czemu nie wolno na nim blokować?"
- [ ] „Jak virtual threads zmieniają podejście do pulowania?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/03-wspolbieznosc.csv`.*

```
Dlaczego nie tworzyć new Thread() na każde żądanie?;Brak reużycia (kosztowny start/teardown), nieograniczona liczba wątków (OOM, context switching), brak wspólnego zarządzania cyklem życia.
Hierarchia interfejsów Executor Framework?;Executor (execute) → ExecutorService (submit/Future, shutdown) → ScheduledExecutorService (schedule, cyklicznie).
Czym różni się execute od submit?;execute(Runnable) nic nie zwraca (wyjątek → UncaughtExceptionHandler); submit zwraca Future i przechwytuje wyjątek (wychodzi z get() jako ExecutionException).
Parametry ThreadPoolExecutor?;corePoolSize, maximumPoolSize, keepAliveTime, workQueue, threadFactory, RejectedExecutionHandler.
Algorytm przyjmowania zadania w TPE?;Jeśli <core → nowy wątek; inaczej do kolejki; jeśli kolejka pełna i <max → nowy wątek; inaczej odrzuć przez handler.
Kiedy TPE tworzy wątek ponad corePoolSize?;Dopiero gdy workQueue jest PEŁNA (a liczba wątków < maximumPoolSize) — nie gdy core są zajęte.
Czemu nieograniczona kolejka unieważnia maximumPoolSize?;offer() do kolejki zawsze się udaje, więc nigdy nie dochodzi do kroku "kolejka pełna → nowy wątek" — pula nie rośnie ponad core.
Zachowanie SynchronousQueue jako workQueue?;Pojemność 0, przekazanie z rąk do rąk; offer udaje się tylko gdy wątek czeka na odbiór → pula szybko rośnie do max (newCachedThreadPool).
Pułapka newFixedThreadPool?;Używa nieograniczonej LinkedBlockingQueue → zadania piętrzą się w pamięci → ryzyko OutOfMemoryError.
Pułapka newCachedThreadPool?;maximumPoolSize = Integer.MAX_VALUE + SynchronousQueue → nieograniczona liczba wątków → wyczerpanie pamięci/CPU.
Polityki RejectedExecutionHandler?;AbortPolicy (rzuca RejectedExecutionException), CallerRunsPolicy (wykonuje wątek zgłaszający — backpressure), DiscardPolicy (cicho gubi), DiscardOldestPolicy (usuwa najstarsze i ponawia).
Po co CallerRunsPolicy?;Backpressure — zadanie wykonuje wątek producenta, więc producent zwalnia zamiast przeciążać pulę.
Rola ThreadFactory?;Kontrola tworzonych wątków: nazwa (thread dumpy), daemon, priorytet, UncaughtExceptionHandler.
shutdown vs shutdownNow?;shutdown — kończy przyjęte, odrzuca nowe; shutdownNow — interrupt na wątkach, opróżnia kolejkę i zwraca nieuruchomione zadania.
Do czego awaitTermination?;Blokuje (z timeoutem) aż pula faktycznie się wygasi; sam shutdown() nie czeka na zakończenie.
Jakie wyjątki rzuca Future.get?;ExecutionException (opakowuje wyjątek zadania, przyczyna w getCause()), InterruptedException, CancellationException; get(timeout) też TimeoutException.
Co to work-stealing w ForkJoinPool?;Każdy wątek ma własną deque zadań; bezczynny wątek "kradnie" zadania z końca deque zajętego — wyrównuje obciążenie, minimalizuje rywalizację.
Czemu nie blokować na ForkJoinPool.commonPool()?;Common pool jest współdzielony przez parallel streams i domyślnie CompletableFuture; blokada zagładza wszystkie inne zadania w JVM — podaj własny executor.
Wzór Goetza na rozmiar puli?;N_threads = N_cpu * U_cpu * (1 + W/C), gdzie W/C to stosunek czasu oczekiwania do obliczeń.
Ile wątków dla CPU-bound vs IO-bound?;CPU-bound ≈ liczba rdzeni + 1; IO-bound znacznie więcej (wątki głównie czekają).
Czy pulować virtual threads (Java 21)?;Nie — są tak tanie, że tworzy się wątek na zadanie (newVirtualThreadPerTaskExecutor); pule zostają dla CPU-bound i jako limiter zasobów.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/03-wspolbieznosc/completablefuture]] · [[wiedza/03-wspolbieznosc/virtual-threads]] · [[wiedza/02-jvm/model-pamieci]]
