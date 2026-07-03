---
temat: "CompletableFuture — asynchroniczne pipeline'y"
faza: 3
status: nieopanowany
priorytet: 🟡
tags: [java, wspolbieznosc]
powiazane: ["[[wiedza/03-wspolbieznosc/executors]]", "[[wiedza/03-wspolbieznosc/virtual-threads]]", "[[wiedza/03-wspolbieznosc/futures]]"]
---

# CompletableFuture — asynchroniczne pipeline'y

> **TL;DR:** `CompletableFuture<T>` to **komponowalny** future/promise: zamiast blokującego `future.get()`
> budujesz **pipeline** transformacji (`thenApply`, `thenCompose`, `thenCombine`) z callbackami i deklaratywną
> obsługą błędów (`exceptionally`, `handle`). Domyślnie działa na `ForkJoinPool.commonPool()` —
> dla zadań **IO zawsze podawaj własny `Executor`**. W erze **virtual threads** (Java 21) blokujący kod jest tani,
> więc CF przydaje się głównie tam, gdzie realnie potrzebujesz *kompozycji* niezależnych wyników.

## 1. Co — definicja i API

Zwykły `Future<T>` (z `ExecutorService.submit`) jest **anemiczny**:

- **`get()` blokuje** wątek do końca zadania — nie ma innego sposobu odebrania wyniku;
- **brak kompozycji** — nie da się powiedzieć „gdy skończysz, zrób X", nie łącząc ręcznie wątków;
- **brak callbacków** — nie zarejestrujesz reakcji na ukończenie;
- **brak obsługi wyjątków** poza tym, że `get()` rzuca `ExecutionException` (owija oryginalny wyjątek);
- `isDone()` / polling to jedyna nieblokująca opcja — brzydka.

`CompletableFuture<T>` (Java 8, `java.util.concurrent`) implementuje `Future<T>` **oraz** `CompletionStage<T>`.
`CompletionStage` to interfejs kompozycji — właśnie stąd biorą się metody `then*`. „Completable" = możesz go
**dopełnić z zewnątrz** (`complete(v)`, `completeExceptionally(e)`), co czyni z niego **promise** (piszący koniec)
i jednocześnie **future** (czytający koniec).

**Tworzenie:**

```java
// zwraca wynik (Supplier) — najczęstsze
CompletableFuture<String> a = CompletableFuture.supplyAsync(() -> pobierzUsera());

// bez wyniku (Runnable) → CompletableFuture<Void>
CompletableFuture<Void> b = CompletableFuture.runAsync(() -> log("start"));

// już gotowy, natychmiast ukończony (np. w testach / short-circuit)
CompletableFuture<String> c = CompletableFuture.completedFuture("cache-hit");

// pusty, do ręcznego dopełnienia (promise)
CompletableFuture<String> p = new CompletableFuture<>();
p.complete("z callbacku innej biblioteki");
```

## 2. Jak — executor, kompozycja, wyjątki pod spodem

### Który executor wykonuje zadanie

Warianty **bez** jawnego `Executor` (`supplyAsync(sup)`, `thenApplyAsync(fn)`) używają
**`ForkJoinPool.commonPool()`** — współdzielonej puli JVM o rozmiarze ~`liczba_rdzeni - 1`.

> ⚠️ **Pułapka #1 (starvation):** `commonPool` jest zaprojektowana pod **krótkie, CPU-bound** zadania.
> Jeśli wrzucisz tam zadania **blokujące IO** (HTTP, JDBC), kilka takich zadań **zapcha całą pulę** i zagłodzi
> resztę aplikacji (w tym parallel streams, które też jej używają). **Dla IO zawsze podawaj własny `Executor`:**

```java
Executor io = Executors.newFixedThreadPool(32);          // pula pod IO
CompletableFuture.supplyAsync(() -> restClient.get(url), io);
```

Jeśli `commonPool` ma <2 wątki (maszyna 1-rdzeniowa), JVM i tak spada do wątku per zadanie — nie polegaj na tym.

### Kompozycja — map vs flatMap vs zip

| Metoda | Analogia | Kiedy |
|---|---|---|
| `thenApply(fn)` | `map` | funkcja zwraca **zwykłą wartość** `U` |
| `thenCompose(fn)` | `flatMap` | funkcja zwraca **`CompletableFuture<U>`** — spłaszcza, unika `CF<CF<U>>` |
| `thenCombine(other, bi)` | `zip` | łączy **dwa niezależne** future gdy oba gotowe |
| `thenAccept(cons)` | konsument | wynik jest, nie zwracasz nowego (`CF<Void>`) |
| `thenRun(runnable)` | efekt | nie zależy od wyniku, tylko „gdy skończone" (`CF<Void>`) |

```java
// thenApply — transformacja synchroniczna wyniku
CompletableFuture<Integer> len = supplyAsync(() -> pobierzTekst()).thenApply(String::length);

// thenCompose — kolejny krok jest ASYNCHRONICZNY (zwraca CF)
CompletableFuture<Order> ord = supplyAsync(() -> pobierzUserId())
        .thenCompose(id -> pobierzZamowienieAsync(id));   // bez tego byłoby CF<CF<Order>>

// thenCombine — dwa niezależne wywołania, złożenie wyników
CompletableFuture<Profil> profil = userCF.thenCombine(kontoCF, (u, k) -> new Profil(u, k));
```

### Warianty `*Async` — na jakiej puli wykona się kolejny krok

To najczęściej mylona rzecz:

- `thenApply(fn)` — **bez `Async`** — krok wykona się na **tym wątku, który dopełnił** poprzedni etap
  (albo, jeśli już był gotowy, na wątku wołającym). Tanie, ale krok „przykleja się" do cudzego wątku.
- `thenApplyAsync(fn)` — zleci krok **na nowo** do `commonPool()`.
- `thenApplyAsync(fn, myExecutor)` — zleci krok na **wskazaną pulę**.

Reguła: chcesz **oddać** wykonanie z krytycznego wątku (np. nie liczyć na wątku Netty/event-loopu) → użyj
`*Async` z własnym executorem. Krótka, czysto CPU transformacja bez blokowania → wariant bez `Async` jest OK.

> ⚠️ **Pułapka #2 (zapomniany suffix `Async`):** ciężka operacja w `thenApply` może wykonać się na wątku,
> którego nie kontrolujesz (np. wątku HTTP klienta) i go zablokować.

### Obsługa wyjątków pod spodem

Wyjątek z dowolnego etapu **propaguje się w dół pipeline'u** owinięty w **`CompletionException`**
(analogicznie jak `ExecutionException` przy `get()`). Kolejne `thenApply/thenCompose` są wtedy **pomijane**,
aż do handlera:

- **`exceptionally(fn)`** — łapie **tylko błąd**, mapuje `Throwable → wartość zastępcza` (recovery).
- **`handle(bi)`** — dostaje **`(wynik, wyjątek)`** — jeden jest `null`. Działa **zawsze**, może zmienić wynik.
- **`whenComplete(bi)`** — dostaje `(wynik, wyjątek)` jak `handle`, ale jest to **efekt uboczny**:
  **nie zmienia** wyniku/wyjątku (świetne do logowania/cleanup, propaguje oryginał dalej).

```java
supplyAsync(() -> ryzykowne(), io)
    .thenApply(x -> x * 2)
    .exceptionally(ex -> -1)                       // fallback tylko na błąd
    .whenComplete((v, ex) -> log("done: " + v));   // log, nie zmienia wyniku
```

> ⚠️ **Pułapka #3 (połknięty wyjątek):** jeśli **nigdzie** nie ma `handle/exceptionally/whenComplete`
> i **nie wołasz `join()/get()`**, wyjątek **znika po cichu** — future kończy się „wyjątkowo", ale nikt tego nie widzi.

### Łączenie wielu future

- **`allOf(cf...)`** → `CompletableFuture<Void>` — dopełnia się, gdy **wszystkie** gotowe.
  ⚠️ Zwraca **`Void`** — wyniki musisz **pozbierać sam** po `join()`:

```java
CompletableFuture<A> a = supplyAsync(this::fa, io);
CompletableFuture<B> b = supplyAsync(this::fb, io);
CompletableFuture.allOf(a, b).join();          // czeka na oba
Result r = new Result(a.join(), b.join());     // join() już nie blokuje — są gotowe
```

- **`anyOf(cf...)`** → `CompletableFuture<Object>` — dopełnia się, gdy **którykolwiek** pierwszy (np. „najszybsze źródło wygrywa").

### Timeouty (Java 9+)

- **`orTimeout(t, unit)`** — po czasie dopełnia **wyjątkowo** `TimeoutException`.
- **`completeOnTimeout(default, t, unit)`** — po czasie dopełnia **wartością domyślną** (graceful).

> Uwaga: timeout **nie przerywa** zadania w tle — ono wciąż zajmuje wątek. To „poddanie się", nie anulowanie.

### `join()` vs `get()`

- `get()` — deklaruje **checked** `InterruptedException`/`ExecutionException` (mniej wygodne w lambdach/streamach).
- `join()` — rzuca **unchecked** `CompletionException` — dlatego pasuje do `.map(CompletableFuture::join)` w streamach.

Obie **blokują** — dlatego woła się je **na końcu**, nie w środku pipeline'u.

## 3. Dlaczego / kiedy — pułapki, reaktywność vs virtual threads

**Kiedy CF świeci:** masz **kilka niezależnych operacji** (najczęściej IO), chcesz je puścić **równolegle**
i **złożyć** wyniki — bez ręcznego żonglowania wątkami i latchami.

**Zbiorcze pułapki:**

1. **Blokowanie w `commonPool` → starvation** — najgroźniejsza. Zawsze własny `Executor` dla IO.
2. **Połknięte wyjątki** — brak handlera + brak `join()` = cisza. Zawsze domykaj pipeline `handle`/`whenComplete`.
3. **Zapomniany `Async`** — kod przykleja się do cudzego wątku (event-loop, wątek klienta HTTP).
4. **`join()` w środku pipeline'u** — niweczy asynchroniczność (blokujesz).
5. **`allOf` zwraca `Void`** — trzeba osobno `join()` każdy future po wynik.
6. **Timeout nie anuluje** zadania w tle — wątek dalej pracuje.

### Kontrast z reaktywnością (Mono/Flux)

CF jest **eager** (gorący): `supplyAsync` **startuje od razu**. Reactor (`Mono`/`Flux`, Project Reactor)
to **leniwe strumienie** — nic się nie dzieje bez `subscribe()`, za to dają **backpressure**, bogate operatory,
`0..N` elementów i retry/scheduling. CF to raczej **jednorazowy `0..1`** wynik. Do prostego „równolegle 2-3 wywołania
i złóż" — CF wystarczy; do strumieni zdarzeń/backpressure → reaktywność.

### Jak virtual threads zmieniają rachunek

W Javie 21 [[wiedza/03-wspolbieznosc/virtual-threads|virtual threads]] sprawiają, że **blokujący kod jest tani** —
`thread.join()` na milionach lekkich wątków nie kosztuje wątków OS. Znika główny powód sięgania po CF, jakim była
**ucieczka od blokowania**. Zamiast pipeline'u callbacków możesz pisać **prosty, sekwencyjny, blokujący kod**
i zrównoleglać go np. przez `StructuredTaskScope`:

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var u = scope.fork(() -> pobierzUsera());       // blokujące, ale na virtual thread
    var k = scope.fork(() -> pobierzKonto());
    scope.join().throwIfFailed();
    return new Profil(u.get(), k.get());            // czytelniej niż thenCombine
}
```

Wniosek: CF **nie znika**, ale w kodzie IO-bound na Javie 21 coraz częściej wybierzesz virtual threads +
structured concurrency (czytelność, propagacja błędów, `try-with-resources`). CF zostaje tam, gdzie masz
gotowe API zwracające `CompletionStage` albo realnie budujesz asynchroniczny graf transformacji.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Endpoint agregujący: równolegle woła **3 usługi** (user, konto, rekomendacje), składa odpowiedź,
z **timeoutem** i **fallbackiem** na błąd — klasyczny „scatter-gather" w warstwie serwisowej mikroserwisu.

```java
private static final Executor IO = Executors.newFixedThreadPool(64);   // WŁASNA pula pod IO

public Profil zbudujProfil(long userId) {
    CompletableFuture<User> user = CompletableFuture
            .supplyAsync(() -> userClient.get(userId), IO)
            .orTimeout(500, TimeUnit.MILLISECONDS);

    CompletableFuture<Konto> konto = CompletableFuture
            .supplyAsync(() -> kontoClient.get(userId), IO)
            .orTimeout(500, TimeUnit.MILLISECONDS);

    // usługa niekrytyczna — degradujemy do pustej listy zamiast wywalać cały profil
    CompletableFuture<List<Reko>> reko = CompletableFuture
            .supplyAsync(() -> rekoClient.get(userId), IO)
            .completeOnTimeout(List.of(), 300, TimeUnit.MILLISECONDS)
            .exceptionally(ex -> List.of());

    return CompletableFuture.allOf(user, konto, reko)          // czekaj na wszystkie
            .thenApply(v -> new Profil(user.join(), konto.join(), reko.join()))
            .handle((profil, ex) -> {                          // domknięcie: nic nie połkniemy
                if (ex != null) {
                    log.error("Profil failed", ex);            // ex to CompletionException(...)
                    throw new ProfilUnavailableException(ex);  // kontrolowana degradacja
                }
                return profil;
            })
            .join();                                           // blokujemy DOPIERO na końcu
}
```

Zwróć uwagę: własna pula `IO`, każdy future z timeoutem, usługa niekrytyczna z `completeOnTimeout` + `exceptionally`,
`allOf` + `join()` po wyniki, i `handle` domykające pipeline, żeby żaden wyjątek nie zniknął.

---

## ✅ Kryteria opanowania
- [ ] Wymienię 4 ograniczenia zwykłego `Future` i pokażę, jak CF je usuwa.
- [ ] Wiem, że domyślnie działa na `commonPool()` i **dlaczego** IO wymaga własnego `Executor`.
- [ ] Odróżnię `thenApply` / `thenCompose` / `thenCombine` (map/flatMap/zip) i wiem, kiedy który.
- [ ] Wyjaśnię różnicę `thenApply` vs `thenApplyAsync` (na jakim wątku wykona się krok).
- [ ] Odróżnię `exceptionally` / `handle` / `whenComplete` i wiem, co robi `CompletionException`.
- [ ] Napiszę z głowy scatter-gather na 3 usługi z timeoutem i fallbackiem.
- [ ] Uzasadnię, kiedy zamiast CF wybrać virtual threads + `StructuredTaskScope`.

### 🔲 Black-box check
- [ ] Co dokładnie robi `commonPool()` i czemu blokowanie w niej powoduje starvation?
- [ ] Dlaczego `thenCompose` „spłaszcza", a `thenApply` daje `CF<CF<T>>`?
- [ ] Na jakim wątku wykona się `thenApply` bez `Async`? A z `Async`? A z `Async(exec)`?
- [ ] Czym różni się `handle` od `whenComplete` co do **zmiany** wyniku?
- [ ] Czemu `allOf` zwraca `Void` i jak wtedy odebrać wyniki?
- [ ] Czy `orTimeout` anuluje zadanie w tle? (nie)
- [ ] Kiedy wyjątek zostanie „połknięty"?

### 🎤 Pytania rekrutacyjne
- [ ] „Jakie problemy `Future` rozwiązuje `CompletableFuture`?"
- [ ] „Domyślnie na jakiej puli działa CF i jaki to niesie problem przy IO?"
- [ ] „`thenApply` vs `thenCompose` vs `thenCombine` — różnice?"
- [ ] „`exceptionally` vs `handle` vs `whenComplete`?"
- [ ] „Jak zrównoleglić 3 wywołania i złożyć wynik z timeoutem?"
- [ ] „Czy virtual threads w Javie 21 wypierają CompletableFuture?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/03-wspolbieznosc.csv`. Format: `Pytanie;Odpowiedź`.*

```
Jakie 4 ograniczenia ma zwykły Future?;Blokujący get, brak kompozycji, brak callbacków, brak deklaratywnej obsługi wyjątków.
Co implementuje CompletableFuture poza Future?;CompletionStage — interfejs kompozycji (metody then*).
Dlaczego CF to "promise" i "future"?;Można go dopełnić z zewnątrz (complete/completeExceptionally = promise) i odczytać wynik (future).
supplyAsync vs runAsync?;supplyAsync bierze Supplier i zwraca wynik (CF<T>); runAsync bierze Runnable bez wyniku (CF<Void>).
Do czego completedFuture?;Zwraca już ukończony CF z gotową wartością (short-circuit, cache-hit, testy).
Na jakiej puli działa CF domyślnie?;ForkJoinPool.commonPool() (~rdzenie-1), pod krótkie CPU-bound zadania.
Czemu IO w commonPool jest złe?;Blokujące zadania zapychają wspólną pulę i głodzą resztę aplikacji (starvation) — podawaj własny Executor.
thenApply vs thenCompose?;thenApply=map (fn zwraca wartość); thenCompose=flatMap (fn zwraca CF, spłaszcza, unika CF<CF>).
Do czego thenCombine?;Łączy dwa NIEZALEŻNE future gdy oba gotowe (zip), np. (u,k)->Profil.
thenAccept vs thenRun?;thenAccept konsumuje wynik (CF<Void>); thenRun uruchamia Runnable niezależny od wyniku.
thenApply vs thenApplyAsync?;Bez Async krok biegnie na wątku dopełniającym poprzedni etap; z Async trafia do commonPool (lub podanego executora).
exceptionally vs handle?;exceptionally łapie tylko błąd i mapuje na wartość; handle dostaje (wynik, wyjątek) — działa zawsze i może zmienić wynik.
Co robi whenComplete?;Efekt uboczny na (wynik, wyjątek) — NIE zmienia wyniku ani wyjątku (logowanie/cleanup), propaguje oryginał.
Jak propaguje się wyjątek w pipeline CF?;Owinięty w CompletionException, pomija kolejne etapy aż do handlera.
Kiedy wyjątek zostanie połknięty?;Gdy brak handle/exceptionally/whenComplete i nie wołasz join()/get().
Co zwraca allOf i jak odebrać wyniki?;CF<Void> po ukończeniu wszystkich; wyniki pozbierać osobno przez join() na każdym future.
Co robi anyOf?;Dopełnia się gdy KTÓRYKOLWIEK pierwszy się skończy (CF<Object>) — np. najszybsze źródło.
orTimeout vs completeOnTimeout?;orTimeout dopełnia wyjątkowo TimeoutException; completeOnTimeout dopełnia wartością domyślną (Java 9+).
Czy orTimeout anuluje zadanie w tle?;Nie — zadanie dalej zajmuje wątek; timeout to poddanie się, nie anulowanie.
join vs get?;get rzuca checked (Interrupted/Execution)Exception; join rzuca unchecked CompletionException — wygodne w streamach.
Jak virtual threads zmieniają rachunek CF?;Blokujący kod staje się tani, więc znika główny powód CF; częściej sekwencyjny kod + StructuredTaskScope.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/03-wspolbieznosc/executors]] · [[wiedza/03-wspolbieznosc/futures]] · [[wiedza/03-wspolbieznosc/virtual-threads]] · [[wiedza/03-wspolbieznosc/reactor-mono-flux]]
