---
temat: "Wątki wirtualne (Virtual Threads, Project Loom)"
faza: 3
status: nieopanowany
priorytet: 🟢
tags: [java, wspolbieznosc, loom]
powiazane: ["[[wiedza/03-wspolbieznosc/executors]]", "[[wiedza/03-wspolbieznosc/completablefuture]]", "[[wiedza/03-wspolbieznosc/watki-podstawy]]"]
---

# Wątki wirtualne (Virtual Threads, Project Loom)

> **TL;DR:** **Virtual threads** (Java 21, **JEP 444**, stabilne) to lekkie wątki harmonogramowane przez **JVM** na małej puli
> **carrier threads** (wątków platformy). Przy **blokującym IO** wątek wirtualny jest **odmontowywany** (unmount) od nośnika,
> który obsługuje inne — dzięki temu prosty, **blokujący** kod obsługuje **miliony** równoczesnych zadań IO bez „callback hell".
> Nie pomagają na **CPU-bound**, nie należy ich pulować, a `synchronized` i ramki natywne **przypinają** (pin) nośnik.

## 1. Co — definicja i API

**Platform thread** (klasyczny `Thread`) to cienki wrapper na wątek **systemu operacyjnego**: własny stos (~1 MB), harmonogramowany
przez OS, praktyczny limit **tysięcy** na maszynę. Model **thread-per-request** + blokujące IO nie skaluje się — brakuje wątków,
zanim wysyci się CPU/sieć.

**Virtual thread** to `Thread` (ta sama klasa, `Thread.isVirtual() == true`), ale **niezwiązany na stałe z wątkiem OS**.
Jest harmonogramowany przez **JVM** na puli **carrier threads** (nośników; domyślnie `ForkJoinPool`, liczba ≈ liczbie rdzeni).
Stos żyje na **heapie** (jako obiekt `Continuation`), jest **tani** i rośnie leniwie → można mieć **miliony** naraz.

```java
// 1) pojedynczy wątek
Thread t = Thread.ofVirtual().name("worker").start(() -> doWork());
Thread.startVirtualThread(() -> doWork());          // skrót

// 2) najczęściej: executor, jeden virtual thread NA ZADANIE
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10_000; i++) {
        exec.submit(() -> callRemoteService());     // blokujące IO — OK!
    }
} // close() czeka na wszystkie zadania (AutoCloseable)
```

Kluczowa zmiana filozofii: **wracasz do prostego, blokującego stylu** (`response = client.send(req)`), a skalowalność zapewnia
runtime, nie Ty. To rozwiązuje bolączki reaktywności: **„colored functions"** (async „zaraża" sygnatury), **callback hell**,
gubione stacktrace i trudny debug. Wątek wirtualny ma **normalny, czytelny stacktrace** i działa z debuggerem.

## 2. Jak — mounting / continuations / pinning pod spodem

**Mounting / unmounting.** Uruchomienie kodu wątku wirtualnego = **zamontowanie** (mount) go na wolnym carrier thread. Gdy kod
trafia na **blokującą operację świadomą Loom** (np. `socket.read`, `Thread.sleep`, `lock`, `BlockingQueue.take`):

1. JVM **odmontowuje** (unmount) wątek wirtualny od nośnika — jego stos zostaje zapisany na heapie.
2. Nośnik jest **wolny** i natychmiast montuje inny wątek wirtualny (obsługuje kolejne zadanie).
3. Po gotowości IO (dane w socketcie, zwolniony lock) scheduler **wznawia** wątek wirtualny na **dowolnym** wolnym nośniku
   (przywraca zapisany stos) i wykonanie płynie dalej — dla programisty jak zwykły powrót z blokującego wywołania.

**Continuations.** Pod spodem stoi mechanizm **delimited continuations** (`jdk.internal.vm.Continuation`): zdolność do
**zamrożenia** (yield) i **wznowienia** (resume) wykonania wraz z jego stosem. Virtual thread = *continuation + scheduler*.
`unmount` to `yield` (kopiuje aktywne ramki stosu na heap), `mount` to `resume` (kopiuje z powrotem). To dlatego stos jest
tani i „elastyczny" — nie rezerwujesz z góry 1 MB jak przy stosie OS.

**Loom-aware JDK.** Żeby to zadziałało, **przerobiono biblioteki `java.*`**: `java.net` (sockety), `java.nio`, `HttpClient`,
`Thread.sleep`, `java.util.concurrent` (locki, kolejki) **ustępują** (unmount) zamiast blokować wątek OS. Blokujące IO
używa pod spodem nieblokującego mechanizmu (poller/`epoll`), a JVM dokleja iluzję blokowania. **Twój kod tego nie widzi.**

**Pinning (przypięcie) — najważniejsza pułapka.** W pewnych sytuacjach wątek wirtualny **nie może się odmontować** i wtedy
**blokuje nośnik** (carrier), tracąc korzyść z Loom:

- **blok `synchronized`** — wejście w monitor **przypina** wątek wirtualny do nośnika; jeśli w środku zablokujesz IO,
  nośnik stoi (w Java 21). ➜ zamiast tego używaj **`ReentrantLock`** (jest Loom-aware — pozwala na unmount).
- **wywołanie metody natywnej / `native` frame / foreign function (JNI, FFM)** — również przypina.

> **Stan w Java 21:** pinning przez `synchronized` jest realny; obejście = `ReentrantLock` + diagnostyka
> `-Djdk.tracePinnedThreads=full`. **Uwaga:** w **nowszych wydaniach (JDK 24, JEP 491)** pinning na `synchronized` został
> w dużej mierze **zaadresowany** — monitory nie przypinają. Ale na certyfikacji/rozmowie o **Java 21** odpowiadaj: `synchronized` przypina.

Jeden nośnik zablokowany to strata; gdy WSZYSTKIE nośniki przypięte i czekają, aplikacja może stanąć. Domyślnie pula nośników =
liczba rdzeni, więc kilka przypięć potrafi zdławić throughput.

## 3. Dlaczego / kiedy — IO vs CPU, nie pulować, pułapki

**Kiedy POMAGAJĄ (IO-bound, wysoka współbieżność):**
- dużo zadań, które **głównie czekają** na IO (baza, REST, kolejka) — klasyczny serwer thread-per-request;
- chcesz **prostoty blokującego kodu** przy skali reaktywnej — bez przepisywania na `Flux`/`Mono`.

**Kiedy NIE pomagają (CPU-bound):**
- zadania **liczące** (kompresja, kryptografia, przetwarzanie obrazów) — nie ma czego odmontowywać, i tak jesteś ograniczony
  liczbą **rdzeni**. Milion wątków wirtualnych liczących nie policzy szybciej niż `Runtime.availableProcessors()` rdzeni.
  Tu właściwe są pule platform o rozmiarze ≈ liczby rdzeni (`ForkJoinPool`/`fixed pool`).

**Zasady użycia:**
- **NIE pooluj wątków wirtualnych.** Są **tanie** (nie ma czego amortyzować). Antywzorzec: `newFixedThreadPool` wątków wirtualnych
  — to sztuczny limit współbieżności. Reguła: **jeden virtual thread na jedno zadanie**, i niech skończy życie z zadaniem.
  Jeśli chcesz ograniczyć współbieżność (np. do bazy), użyj `Semaphore`, nie puli.
- **`ThreadLocal`** działa, ale przy **milionach** wątków każdy własny egzemplarz to koszt pamięci; rozważ **`ScopedValue`**
  (preview w 21, JEP 446) — niemutowalny, dziedziczony strukturalnie, tańszy.
- **Nie cache'uj wątku ani nie zakładaj tożsamości nośnika** (może się zmienić między unmount/mount).

**Uzupełnienia (preview w Java 21):**
- **Structured concurrency** — `StructuredTaskScope` (JEP 453, **preview**): traktuje grupę pod-zadań jako jednostkę
  (wspólny czas życia, propagacja błędu/anulowania, brak wycieku wątków). Naturalnie łączy się z virtual threads.
- **`ScopedValue`** (JEP 446, **preview**): następca `ThreadLocal` dla ery Loom.

**Wpływ na frameworki.** Spring Boot 3.2+ / Tomcat / Jetty potrafią obsługiwać żądania na wątkach wirtualnych
(`spring.threads.virtual.enabled=true`). Efekt: **wyższy throughput** przy blokujących kontrolerach i JDBC — **bez** przepisywania
aplikacji na reaktywność. To dla wielu zespołów główny powód, dla którego reaktywność przestaje być konieczna.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

**10 000 równoległych zadań IO** — każde „śpi" 1 s (imitacja wolnego wywołania sieciowego):

```java
Runnable io = () -> {
    try { Thread.sleep(Duration.ofSeconds(1)); }   // blokujące IO
    catch (InterruptedException e) { Thread.currentThread().interrupt(); }
};

// A) VIRTUAL THREADS — ~1 s całości, 10 000 wątków bez problemu
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i -> exec.submit(io));
} // ~1 s: wszystkie odmontowane podczas sleep, garstka nośników wystarcza

// B) PULA PLATFORM (np. 200 wątków) — ~50 s (10_000 / 200 zadań × 1 s)
try (var exec = Executors.newFixedThreadPool(200)) {
    IntStream.range(0, 10_000).forEach(i -> exec.submit(io));
} // wątki OS blokują się na sleep; przepustowość ogranicza rozmiar puli
// Próba newFixedThreadPool(10_000) platform → ~10 GB stosów, ryzyko OOM.
```

Realny odpowiednik: API-gateway agregujący kilka usług per żądanie (fan-out), warstwa integracji z wieloma zewnętrznymi REST-ami,
albo migawka „legacy" blokującego serwisu na wyższy throughput przez samo włączenie flagi w Springu.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię różnicę **platform thread** vs **virtual thread** (kto harmonogramuje, gdzie stos, ile ich naraz).
- [ ] Opiszę **mount/unmount** i rolę **carrier threads** przy blokującym IO.
- [ ] Wiem, że pod spodem to **continuations** (yield/resume stosu na heap).
- [ ] Rozumiem **pinning** (`synchronized`, native) i obejście (`ReentrantLock`); znam stan w Java 21 vs nowsze JDK.
- [ ] Wiem, kiedy **NIE** pomagają (CPU-bound) i czemu **nie pulować**.
- [ ] Odróżniam co **stabilne** (JEP 444) od **preview** (`StructuredTaskScope`, `ScopedValue`) w Java 21.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co dokładnie robi JVM, gdy virtual thread wywoła `socket.read()`? (unmount → nośnik wolny → resume po IO)
- [ ] Dlaczego stos wątku wirtualnego jest tani, a platformowego drogi? (heap `Continuation` vs stały stos OS ~1 MB)
- [ ] Czemu `synchronized` szkodzi, a `ReentrantLock` nie? (pinning nośnika vs Loom-aware lock)
- [ ] Dlaczego pula 10 000 wątków wirtualnych bije pulę 200 platform na zadaniach IO, ale nie na CPU?
- [ ] Czemu nie należy pulować wątków wirtualnych i jak zamiast tego ograniczyć współbieżność? (`Semaphore`)

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Czym są virtual threads i jaki problem rozwiązują?" (thread-per-request + blokujące IO bez limitu wątków OS)
- [ ] „Virtual threads vs programowanie reaktywne — kiedy co?" (prostota blokująca vs backpressure/streaming)
- [ ] „Co to pinning i jak go uniknąć?"
- [ ] „Czy virtual threads przyspieszą zadanie CPU-bound?" (nie — ogranicza liczba rdzeni)
- [ ] „Dlaczego `newVirtualThreadPerTaskExecutor`, a nie pula o stałym rozmiarze?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/03-wspolbieznosc.csv`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Co to virtual thread (Java 21)?;Lekki wątek harmonogramowany przez JVM na puli carrier threads; stos na heapie, mogą być miliony naraz.
Który JEP i wersja ustabilizowały virtual threads?;JEP 444, Java 21 (stabilne, nie preview).
Kto harmonogramuje virtual thread, a kto platform thread?;Virtual — JVM (na carrier threads), platform — system operacyjny.
Co to carrier thread?;Wątek platformy (OS) z puli, na którym JVM montuje wątki wirtualne; domyślnie ForkJoinPool ≈ liczba rdzeni.
Ile pamięci zajmuje stos platform thread?;Ok. 1 MB (rezerwowany), stąd praktyczny limit tysięcy wątków OS.
Co to mounting i unmounting?;Mount = uruchomienie virtual threada na carrierze; unmount = odmontowanie przy blokującym IO, by nośnik obsłużył inne.
Co dzieje się przy blokującym IO w virtual threadzie?;Wątek się odmontowuje (stos na heap), nośnik obsługuje inne zadania, a po gotowości IO wątek jest wznawiany.
Jaki mechanizm realizuje virtual threads pod spodem?;Delimited continuations (yield/resume) — zamrożenie i wznowienie stosu; virtual thread = continuation + scheduler.
Dlaczego blokujące IO nie blokuje nośnika w Loom?;Biblioteki java.* (sockety, sleep, locki) są Loom-aware — ustępują (unmount) zamiast blokować wątek OS.
Co to pinning?;Sytuacja, gdy virtual thread nie może się odmontować i blokuje carrier thread — traci korzyść z Loom.
Co przypina virtual thread w Java 21?;Blok synchronized oraz ramki natywne (JNI/FFM).
Czym zastąpić synchronized, by uniknąć pinningu?;ReentrantLock (jest Loom-aware, pozwala na unmount).
Jak zdiagnozować pinning?;Flagą -Djdk.tracePinnedThreads=full.
Kiedy virtual threads NIE pomagają?;Przy zadaniach CPU-bound — nie ma czego odmontować, i tak ogranicza liczba rdzeni.
Czy należy pulować virtual thready?;Nie — są tanie, jeden wątek na zadanie; do ograniczenia współbieżności użyj Semaphore.
Jak stworzyć virtual thread?;Thread.startVirtualThread(r), Thread.ofVirtual().start(r) lub Executors.newVirtualThreadPerTaskExecutor().
Co z ThreadLocal przy milionach virtual threads?;Działa, ale kosztuje pamięć; rozważ ScopedValue (preview w 21, JEP 446).
Co to StructuredTaskScope i jaki ma status w Java 21?;Structured concurrency (JEP 453) — grupa pod-zadań jako jednostka; w Java 21 preview.
Jak virtual threads zmieniają Springa?;Włączenie (spring.threads.virtual.enabled) daje wyższy throughput bez przepisywania na reaktywność.
Virtual threads vs reaktywność — główna przewaga?;Prosty, blokujący, debugowalny kod (czytelny stacktrace) zamiast callback hell i colored functions.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/03-wspolbieznosc/executors]] · [[wiedza/03-wspolbieznosc/completablefuture]] · [[wiedza/03-wspolbieznosc/watki-podstawy]] · [[wiedza/02-jvm/model-pamieci]]
