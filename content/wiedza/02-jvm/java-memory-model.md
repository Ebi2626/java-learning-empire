---
temat: "Java Memory Model (JMM)"
faza: 2
status: nieopanowany
priorytet: 🟡
tags: [java, jvm, wspolbieznosc]
powiazane: ["[[wiedza/03-wspolbieznosc/synchronizacja]]", "[[wiedza/03-wspolbieznosc/podstawy]]", "[[wiedza/02-jvm/model-pamieci]]"]
---

# Java Memory Model (JMM)

> **TL;DR:** **JMM** (JSR-133, od Javy 5, aktualne w Javie 21) to *kontrakt* mówiący, **kiedy** zapis w jednym wątku jest gwarantowanie **widoczny** dla odczytu w innym i **jaka kolejność** operacji jest zagwarantowana. Bez niego **cache per-rdzeń**, **rejestry** i **reordering** (kompilator + CPU) sprawiają, że współbieżny kod jest nieprzewidywalny. Sercem modelu jest relacja **happens-before**: jeśli `A` HB `B`, to efekty `A` są widoczne dla `B`. `volatile`, `synchronized` i `final` to narzędzia, które tę relację ustanawiają.

## 1. Co — definicja i API

**Problem, który JMM rozwiązuje.** Naiwnie zakładasz, że jest jedna „pamięć", a wątki po kolei odczytują i zapisują zmienne. Sprzęt i kompilator łamią tę intuicję na trzy sposoby:

- **Visibility (widoczność):** każdy rdzeń CPU ma własny **cache** (L1/L2) i rejestry. Wątek A zapisuje `x = 1` — wartość może zostać w cache/rejestrze rdzenia A i **nigdy** nie dotrzeć do rdzenia, na którym działa wątek B. B widzi starą wartość, potencjalnie w nieskończoność.
- **Reordering (zmiana kolejności):** dla wydajności **kompilator** (i JIT — [[wiedza/02-jvm/jit]]) oraz **CPU** (out-of-order execution, store buffers) mogą wykonać instrukcje w innej kolejności niż w kodzie źródłowym. W obrębie jednego wątku efekt jest identyczny (*as-if-serial*), ale **inny wątek** może zaobserwować zapisy w przemieszanej kolejności.
- **Atomicity (atomowość):** operacja złożona jak `i++` to w rzeczywistości read-modify-write (odczyt, inkrementacja, zapis) — trzy kroki, które inny wątek może „poprzeplatać".

Bez zdefiniowanego modelu każda z tych rzeczy jest zależna od platformy → kod działa na Twoim laptopie, a psuje się pod obciążeniem na innym CPU. **JMM to minimalny zbiór gwarancji przenośnych między architekturami** (x86, ARM…), których kompilator i JVM muszą dotrzymać.

**API / narzędzia, którymi ustanawiasz gwarancje:**

| Narzędzie | Widoczność | Zakaz reorderingu | Atomowość | Wzajemne wykluczanie |
|---|:---:|:---:|:---:|:---:|
| `volatile` | ✅ | ✅ (wokół dostępu) | ❌ (nie dla `i++`) | ❌ |
| `synchronized` | ✅ | ✅ | ✅ (całej sekcji) | ✅ |
| `final` | ✅ (bezpieczna publikacja) | ✅ (freeze) | — | — |
| `java.util.concurrent.atomic.*` | ✅ | ✅ | ✅ (pojedyncze op.) | ❌ |

Minimalny przykład — flaga stopu:
```java
class Worker {
    private volatile boolean running = true;  // volatile jest KLUCZOWE
    void loop() { while (running) { /* praca */ } }
    void stop() { running = false; }          // zmiana widoczna dla loop()
}
```

## 2. Jak — happens-before i bariery pod spodem

**Relacja happens-before (HB)** to jądro JMM. Jeśli akcja `A` *happens-before* `B`, to: (1) wszystko, co `A` zapisało do pamięci, jest **widoczne** dla `B`, oraz (2) `A` nie może zostać przestawione za `B`. Uwaga: HB to relacja o **widoczności i kolejności**, a *nie* o czasie zegarowym.

Reguły ustanawiające HB (JSR-133):

1. **Program order** — w obrębie **jednego wątku** każda akcja HB następną w kolejności programu. (To daje intuicję sekwencyjności *wewnątrz* wątku.)
2. **Monitor lock** — **zwolnienie** monitora (`unlock` / wyjście z `synchronized`) HB każde **kolejne nabycie** *tego samego* monitora.
3. **Volatile** — **zapis** pola `volatile` HB każdy **kolejny odczyt** *tego samego* pola.
4. **Thread start** — wywołanie `Thread.start()` HB każda akcja w uruchomionym wątku (nowy wątek widzi wszystko, co ustawiono przed startem).
5. **Thread join** — każda akcja w wątku HB powrót z `Thread.join()` na tym wątku (widzisz wyniki po dołączeniu).
6. **Final fields** — poprawna inicjalizacja pola `final` w konstruktorze HB (przez *freeze*) jego odczyt przez inny wątek, o ile referencja nie „wyciekła" z konstruktora.

HB jest **przechodnia**: jeśli `A` HB `B` i `B` HB `C`, to `A` HB `C`. To pozwala łączyć reguły — np. zapis zwykłego pola *przed* zapisem `volatile` staje się widoczny dla wątku, który *odczyta* to `volatile` i wykona odczyt zwykłego pola.

**Semantyka poszczególnych narzędzi:**

- **`volatile`** — daje **widoczność** (odczyt zawsze widzi ostatni zapis) i **zakaz reorderingu** wokół dostępu. **Nie daje atomowości operacji złożonych**: `volatile int i; i++;` wciąż ma race, bo to read-modify-write. Do liczników użyj `AtomicInteger` lub `synchronized`.
- **`synchronized`** — łączy **wzajemne wykluczanie** (tylko jeden wątek w bloku na danym monitorze) z **happens-before przez monitor** (reguła 2). Sekcja krytyczna jest atomowa *względem innych wątków synchronizujących się na tym samym monitorze*.
- **Pola `final`** — mechanizm **final field freeze**: JMM wstawia barierę na końcu konstruktora, tak że wątek, który zobaczy w pełni skonstruowany obiekt, widzi też **kompletne** wartości pól `final` (i obiektów przez nie osiągalnych) — bez dodatkowej synchronizacji. To fundament **bezpiecznej publikacji** obiektów niemutowalnych (np. `String`). Warunek: `this` nie może wyciec z konstruktora.

**Bariery pamięci sprzętu (koncepcyjnie).** Powyższe gwarancje JVM realizuje, wstawiając w kod maszynowy **bariery pamięci** (memory fences), które ograniczają reordering na poziomie CPU:

- **LoadLoad** — odczyty przed barierą kończą się przed odczytami po niej.
- **StoreStore** — zapisy przed barierą są widoczne przed zapisami po niej.
- **LoadStore** / **StoreLoad** — analogicznie mieszane (StoreLoad jest najdroższa, np. `mfence` na x86).

W przybliżeniu: **odczyt `volatile`** działa jak bariera **LoadLoad + LoadStore** (acquire), a **zapis `volatile`** jak **StoreStore + StoreLoad** (release). `synchronized` stawia bariery przy `lock`/`unlock`. Na **x86** (silny model TSO) większość barier jest tania lub darmowa, ale StoreLoad kosztuje; na **ARM/POWER** (słaby model) bariery są realnie wstawiane i kosztują więcej — dlatego bugi z brakiem `volatile` częściej „wychodzą" na ARM.

## 3. Dlaczego / kiedy — pułapki (data race, publikacja)

**Data race — definicja formalna.** Race występuje, gdy: (1) **dwa dostępy** do tej samej zmiennej, (2) **co najmniej jeden to zapis**, (3) **nie są uporządkowane relacją happens-before**. Program **bez** race'ów (poprawnie zsynchronizowany) zachowuje się **sekwencyjnie spójnie** — możesz rozumować jak o przeplocie. Program **z** race'em jest w praktyce **undefined behavior**: JMM nie gwarantuje żadnej sensownej wartości, kompilator ma prawo do agresywnych optymalizacji (np. wyhoistować odczyt poza pętlę → pętla nieskończona), a wynik zależy od CPU/JIT/obciążenia.

**Atomowość `long`/`double` i word tearing.** JMM pozwala, by zapis/odczyt 64-bitowych `long`/`double` był **nieatomowy** — na 32-bitowych platformach może być rozbity na dwa 32-bitowe zapisy. Inny wątek może wtedy zobaczyć „pół starej, pół nowej" wartości (**word tearing**). Deklaracja `volatile long`/`volatile double` **wymusza atomowość**. (Na 64-bitowych JVM w praktyce jest atomowo, ale spec tego nie gwarantuje bez `volatile` — nie polegaj na tym.)

**Safe publication — wzorce.** „Publikacja" = udostępnienie referencji do obiektu innemu wątkowi. Aby publikacja była **bezpieczna** (drugi wątek zobaczył w pełni zbudowany obiekt), użyj jednego z:

- inicjalizacja przez **statyczny inicjalizator** (gwarancja przez class-init lock),
- zapis referencji do pola **`volatile`** (lub przez `AtomicReference`),
- zapis referencji **w bloku `synchronized`** (odczyt też pod tym monitorem),
- pola **`final`** obiektu niemutowalnego (freeze),
- umieszczenie w **kolekcji współbieżnej** (`ConcurrentHashMap`, `BlockingQueue`… — ich metody ustanawiają HB między `put` a `get`/`take`).

**Kiedy co:**
- czytasz/piszesz **jedną flagę lub referencję**, bez logiki „przeczytaj-zmodyfikuj-zapisz" → `volatile`;
- **licznik / złożona operacja** na jednej zmiennej → `AtomicInteger`/`AtomicLong` (CAS, lock-free);
- **inwariant obejmujący wiele pól** naraz → `synchronized` lub `Lock` ([[wiedza/03-wspolbieznosc/synchronizacja]]);
- **obiekt niemutowalny** dzielony między wątki → pola `final`, nie potrzebujesz dalszej synchronizacji.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Klasyczny bug: wątek roboczy nie zatrzymuje się na komendę stop, bo flaga nie jest `volatile`. JIT ma prawo odczytać `running` **raz** przed pętlą i trzymać w rejestrze → pętla nigdy nie widzi zmiany.

```java
// ŹLE — data race, pętla może nigdy się nie zakończyć
class BadWorker implements Runnable {
    private boolean running = true;          // brak volatile!
    public void run() { while (running) doWork(); }
    public void stop() { running = false; }  // zmiana może nie dotrzeć do run()
}

// DOBRZE — volatile ustanawia happens-before (zapis HB odczyt)
class GoodWorker implements Runnable {
    private volatile boolean running = true; // volatile
    public void run() { while (running) doWork(); }
    public void stop() { running = false; }  // widoczne natychmiast
}
```

Spotkasz to np. w pulach wątków, konsumentach kolejek, watcherach plików, serwerach — wszędzie, gdzie jeden wątek sygnalizuje zamknięcie drugiemu. Alternatywa dla flagi: `Thread.interrupt()` + sprawdzanie `isInterrupted()` (współpracuje z blokującym I/O). W bibliotekach spotkasz też **double-checked locking**, który jest poprawny **tylko** gdy pole jest `volatile` — inaczej reordering pozwala opublikować częściowo skonstruowany obiekt.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię trzy problemy, które JMM rozwiązuje: visibility, reordering, atomicity.
- [ ] Wymienię reguły happens-before i pokażę, jak `volatile`/`synchronized`/`final` je ustanawiają.
- [ ] Podam definicję data race (dwa dostępy, min. jeden zapis, brak HB) i czemu to UB.
- [ ] Wiem, czemu `volatile i++` nie jest atomowe i czym to zastąpić.
- [ ] Wyjaśnię safe publication i wymienię jej wzorce.
- [ ] Rozumiem final field freeze i word tearing dla `long`/`double`.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Czemu bez `volatile` pętla `while(flag)` może się nie zakończyć? (rejestr/hoisting/cache)
- [ ] Co dokładnie gwarantuje odczyt `volatile`, a czego NIE gwarantuje?
- [ ] Jak `synchronized` daje jednocześnie mutex i widoczność? (monitor → HB)
- [ ] Jak działa final field freeze i kiedy się psuje? (wyciek `this` z konstruktora)
- [ ] Jak bariery LoadLoad/StoreStore/StoreLoad realizują semantykę `volatile`?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Czym jest happens-before i po co?"
- [ ] „`volatile` vs `synchronized` — różnice i kiedy co?"
- [ ] „Czy `volatile` wystarczy do licznika? Dlaczego nie?"
- [ ] „Co to data race i czemu program z race'em jest nieprzewidywalny?"
- [ ] „Jak bezpiecznie opublikować obiekt między wątkami?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Czym jest Java Memory Model?;Kontrakt (JSR-133) mówiący, kiedy zapis w jednym wątku jest widoczny dla innego i jaka kolejność operacji jest gwarantowana.
Trzy problemy, które rozwiązuje JMM?;Visibility (widoczność między cache/rejestrami rdzeni), reordering (kompilator+CPU zmieniają kolejność), atomicity.
Co to happens-before?;Relacja: jeśli A HB B, to efekty zapisów A są widoczne dla B i A nie może być przestawione za B.
Czy happens-before dotyczy czasu zegarowego?;Nie — dotyczy widoczności i kolejności pamięciowej, nie realnego czasu.
Reguła HB dla volatile?;Zapis pola volatile HB każdy kolejny odczyt tego samego pola.
Reguła HB dla monitora?;Zwolnienie monitora HB każde kolejne nabycie tego samego monitora.
Reguła HB dla Thread.start?;Wywołanie start() HB wszystkie akcje w uruchomionym wątku.
Reguła HB dla Thread.join?;Wszystkie akcje w wątku HB powrót z join() na tym wątku.
Czy happens-before jest przechodnia?;Tak — jeśli A HB B i B HB C, to A HB C.
Co gwarantuje volatile?;Widoczność (odczyt widzi ostatni zapis) i zakaz reorderingu wokół dostępu; NIE daje atomowości operacji złożonych.
Czemu volatile i++ jest błędne?;i++ to read-modify-write (3 kroki); volatile nie robi go atomowym — potrzebny AtomicInteger lub synchronized.
Co daje synchronized poza wzajemnym wykluczaniem?;Happens-before przez monitor: unlock HB kolejny lock na tym samym monitorze — czyli widoczność.
Co to final field freeze?;Bariera na końcu konstruktora: wątek widzący w pełni zbudowany obiekt widzi kompletne pola final (o ile this nie wyciekło).
Definicja data race?;Dwa dostępy do tej samej zmiennej, min. jeden zapis, bez uporządkowania relacją happens-before.
Czemu data race to problem?;Program z race'em jest w praktyce UB — brak gwarancji wartości, kompilator może agresywnie optymalizować, wynik zależy od CPU/JIT.
Co to word tearing?;Nieatomowy zapis long/double (na 32-bit rozbity na dwa) — inny wątek widzi pół starej, pół nowej wartości; volatile wymusza atomowość.
Wzorce safe publication?;Zapis referencji do volatile, do pola final, w bloku synchronized, statyczny inicjalizator lub kolekcja współbieżna.
Jak volatile mapuje się na bariery sprzętu?;Odczyt ~ LoadLoad+LoadStore (acquire), zapis ~ StoreStore+StoreLoad (release).
Czemu while(flag) bez volatile może być nieskończone?;JIT może odczytać flag raz i trzymać w rejestrze/cache — zmiana z innego wątku nigdy nie jest widoczna.
volatile vs synchronized — kluczowa różnica?;volatile: tylko widoczność+kolejność pojedynczego pola, bez mutexu; synchronized: mutex + widoczność całej sekcji + atomowość względem monitora.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/03-wspolbieznosc/podstawy]] · [[wiedza/03-wspolbieznosc/synchronizacja]] · [[wiedza/02-jvm/model-pamieci]] · [[wiedza/02-jvm/jit]]
