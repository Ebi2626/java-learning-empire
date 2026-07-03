---
temat: "Garbage Collection w JVM"
faza: 2
status: nieopanowany
priorytet: 🟡
tags: [java, jvm]
powiazane: ["[[wiedza/02-jvm/model-pamieci]]", "[[wiedza/00-fundament/jdk-jre-jvm]]", "[[wiedza/02-jvm/jit]]"]
---

# Garbage Collection w JVM

> **TL;DR:** GC automatycznie zwalnia obiekty **nieosiągalne** z **GC roots** (nie ma ręcznego `free`).
> Opiera się na **weak generational hypothesis** (większość obiektów umiera młodo) → podział na **young/old**.
> W Javie 21 (HotSpot) domyślny jest **G1** (regionowy, cel pauzy `MaxGCPauseMillis`); dla low-latency masz
> **ZGC / Shenandoah** (concurrent, pauzy **sub-ms**), a **Epsilon** to no-op do testów.

## 1. Co — po co GC i osiągalność

**Garbage Collector** to podsystem JVM działający na **heapie** ([[wiedza/02-jvm/model-pamieci]]), który
**automatycznie odzyskuje pamięć** zajętą przez obiekty, których program już nie użyje. W C/C++ robisz to ręcznie
(`malloc`/`free`, `delete`) — stąd całe klasy błędów: *use-after-free*, *double free*, wycieki. W Javie **nie ma
`free`** — o zwolnieniu decyduje GC. Nie znaczy to, że wycieków nie ma (patrz §3), tylko że mają inną naturę.

**Kluczowa idea: osiągalność (reachability), nie liczenie referencji.** Obiekt „żyje" nie dlatego, że ktoś na niego
wskazuje, ale dlatego, że jest **osiągalny przez łańcuch referencji z GC roots**. Dzięki temu HotSpot GC radzi sobie
z **cyklami** (A→B→A, ale oba nieosiągalne z roots → oba do zebrania) — czego naiwny *reference counting* nie potrafi.

**GC roots** to „punkty startowe" grafu osiągalności — referencje spoza heapa lub takie, które muszą przeżyć:
- **zmienne lokalne i parametry na stosach wątków** (frame’y aktywnych metod) oraz rejestry,
- **statyczne pola klas** (`static`) — żyją tak długo, jak załadowana klasa/classloader,
- **referencje z JNI** (native, *global/local JNI references*),
- **aktywne wątki** (`Thread`), monitory (obiekty w `synchronized`), stringi z *string pool*, wewnętrzne struktury JVM.

Wszystko, do czego **nie da się dojść** z żadnego roota → **garbage**, kwalifikuje się do zebrania.

```java
Object a = new Object();  // osiągalny: zmienna lokalna 'a' to root-owy łańcuch
a = null;                 // stary Object nieosiągalny → kandydat do GC (nie natychmiast!)
```
> Uwaga: `a = null` **nie** wywołuje GC ani nie „usuwa" obiektu — jedynie zrywa referencję. Kiedy pamięć wróci,
> decyduje kolektor. `System.gc()` to tylko *sugestia* (można ją zignorować, `-XX:+DisableExplicitGC`).

## 2. Jak — algorytmy i kolektory pod spodem

### 2a. Podstawowe algorytmy
- **Mark-Sweep** — dwie fazy: **mark** (przejdź graf od roots, oznacz żywe) → **sweep** (przejdź heap, zwolnij
  nieoznaczone). Prosty, ale zostawia **fragmentację** (dziury), więc alokacja bywa wolna.
- **Mark-Sweep-Compact** — jak wyżej + **compact**: żywe obiekty upychane obok siebie, wolne miejsce w jednym kawałku
  → szybka alokacja (bump-the-pointer), ale kosztuje przesuwanie obiektów.
- **Copying (scavenge)** — heap dzielony na dwie połowy (*from/to*). Żywe kopiowane do drugiej połowy, stara jest
  czyszczona hurtem. Brak fragmentacji, alokacja błyskawiczna; koszt = ~2× pamięci i praca proporcjonalna do
  **liczby żywych** (nie martwych) — idealne, gdy żywych mało.

### 2b. Hipoteza generacyjna (weak generational hypothesis)
Empiryczna obserwacja: **większość obiektów umiera młodo** (obiekty krótko żyjące dominują), a im obiekt starszy,
tym większa szansa, że przeżyje jeszcze długo. Stąd **generacyjny heap**:

```
                 HEAP (podział generacyjny — model klasyczny)
 ┌───────────────────── Young Generation ─────────────────────┐  ┌── Old Gen (Tenured) ──┐
 │  ┌──────── Eden ────────┐   ┌── Survivor S0 ──┐┌── S1 ──┐    │  │  długo żyjące obiekty  │
 │  │  tu alokują się nowe │   │  przeżyli 1 minor││ (to)  │    │  │  (promocja po N minor  │
 │  │  obiekty (bump ptr)  │   │  (from)          ││       │    │  │   GC / gdy nie mieszczą │
 │  └──────────────────────┘   └─────────────────┘└───────┘    │  │   się w Survivor)      │
 └────────────┬────────────────────────────────────────────────┘  └───────────┬───────────┘
   Minor GC (copying, częsty, szybki, STW krótki)   ── promocja ──▶   Major/Full GC (rzadki, drogi)

   Metaspace (poza heapem): metadane klas — NIE jest zbierany przez GC young/old
```

- **Eden** — tu lądują nowe obiekty. Gdy się zapełni → **minor GC**.
- **Survivor S0/S1** — przeżywców minor GC kopiuje się między survivorami; licznik **age** rośnie z każdym minor GC.
- **Promocja (tenuring)** — obiekt przekroczy próg wieku (`MaxTenuringThreshold`) albo nie mieści się w survivorze
  → awansuje do **Old Gen**.

### 2c. Minor vs Major/Full GC i stop-the-world
- **Minor GC** — sprząta **tylko Young** (algorytm copying). Częsty, ale **krótki** — bo żywych obiektów w Young mało.
- **Major GC** — sprząta **Old**. **Full GC** — całość (Young + Old + często metaspace); najdroższy, unikaj.
- **Stop-the-world (STW)** — GC **zatrzymuje wszystkie wątki aplikacji**, by bezpiecznie przejść po grafie
  (aplikacja nie może zmieniać referencji w trakcie markowania/przesuwania). Wątki stają na **safepoint**.
  STW = **pauza** widoczna jako lag/latency. Cel nowoczesnych kolektorów: robić jak najwięcej **concurrent**
  (równolegle z aplikacją), zostawiając tylko krótkie STW.

### 2d. Kolektory w HotSpot (Java 21)

| Kolektor | Cel | Charakter | Kiedy |
|---|---|---|---|
| **Serial** | prostota | 1 wątek, STW, generacyjny | małe heapy, single-core, kontenery z 1 CPU (`-XX:+UseSerialGC`) |
| **Parallel** | **throughput** | wiele wątków GC, STW, generacyjny | batch/obliczenia, gdzie liczy się przepustowość, pauzy mniej ważne (`-XX:+UseParallelGC`) |
| **G1** | **balans** pauzy/throughput | regionowy, głównie STW ale krótkie i sterowalne | **domyślny od Java 9**, ogólne serwisy, heapy średnie–duże |
| **ZGC** | **low-latency** | concurrent, sub-ms pauzy | duże heapy (do TB), wymóg niskich pauz (`-XX:+UseZGC`) |
| **Shenandoah** | **low-latency** | concurrent, pauzy <1ms | jak ZGC (OpenJDK/Red Hat), (`-XX:+UseShenandoahGC`) |
| **Epsilon** | **no-op** | alokuje, **nie zwalnia** | testy wydajności/alokacji, aż OOM (`-XX:+UseEpsilonGC`) |

**G1 (Garbage-First) — domyślny.** Dzieli heap na **równe regiony** (~1–32 MB), a role (Eden/Survivor/Old) są
przypisywane **regionom dynamicznie** (nie ma stałych ciągłych generacji). Kluczowe cechy:
- **Cel pauzy `-XX:MaxGCPauseMillis`** (domyślnie 200 ms) — G1 wybiera *tyle regionów, ile zdąży posprzątać* w budżecie
  czasu → „garbage first": najpierw regiony z największą ilością śmieci (najlepszy zwrot).
- **Mixed collections** — po fazie concurrent marking G1 w jednym cyklu sprząta Young **oraz** wybrane regiony Old
  (stąd „mixed"), rozkładając koszt czyszczenia Old na wiele krótkich pauz zamiast jednego Full GC.
- **Humongous objects** — obiekt większy niż **½ regionu** trafia w specjalne *humongous regions* (ciągłe).
  Duże tablice/bufory → fragmentacja regionów, gorsza obsługa, częstsze Full GC. To realne wąskie gardło (§3).

**ZGC / Shenandoah — concurrent, sub-ms.** Prawie cała praca (mark, relocation/compact) dzieje się **równolegle
z aplikacją**; STW to tylko krótkie fazy synchronizacyjne → pauzy **<1 ms** niezależnie od rozmiaru heapa.
Mechanika: **colored pointers** (ZGC — metadane GC zakodowane w bitach wskaźnika) i **load/read barriers** —
mały fragment kodu wstrzykiwany przy odczycie referencji, który „naprawia" wskaźnik, gdy obiekt został przeniesiony
w tle. Cena: narzut barier (throughput nieco niższy niż Parallel) i trochę więcej pamięci.

## 3. Dlaczego / kiedy — dobór, tuning i wąskie gardła

### Dobór kolektora
- **Domyślnie zostaw G1** — dobry balans, sam się adaptuje. Nie tuninguj „na zapas".
- **Throughput ponad pauzy** (batch, ETL, benchmark) → **Parallel**.
- **Twarde SLA na latency / p99** (trading, API o niskich ogonach) → **ZGC** lub **Shenandoah**.
- **Malutki kontener / 1 vCPU / krótki proces** → **Serial** (mniejszy narzut niż G1).
- **Test alokacji / mierzenie „ile śmieci produkuję"** → **Epsilon** (proces padnie na OOM, ale bez zakłóceń GC).

### Kluczowe flagi
```
-Xms<rozmiar>              # startowy rozmiar heapa (warto = -Xmx: brak resize, przewidywalność)
-Xmx<rozmiar>              # maksymalny heap (w kontenerze rozważ -XX:MaxRAMPercentage=75)
-XX:+UseG1GC               # G1 (domyślny w Java 21, jawnie dla czytelności)
-XX:+UseZGC                # ZGC (dodaj -XX:+ZGenerational — generacyjny ZGC, od JDK 21)
-XX:+UseParallelGC / UseSerialGC / UseShenandoahGC / UseEpsilonGC
-XX:MaxGCPauseMillis=200   # cel pauzy dla G1 (miękki, nie gwarancja)
-Xlog:gc*                  # logi GC (nowy Unified Logging; zastąpił -XX:+PrintGCDetails)
```

### Jak czytać logi GC (`-Xlog:gc*`)
```
[2.331s][info][gc] GC(12) Pause Young (Normal) (G1 Evacuation Pause) 512M->48M(1024M) 8.442ms
                    │      │                     │                     │    │   │        └ czas pauzy (STW)
                    │      │                     │                     │    │   └ całkowity heap
                    │      │                     │                     │    └ zajęte PO GC
                    │      │                     │                     └ zajęte PRZED GC
                    │      │                     └ przyczyna
                    │      └ typ pauzy (Young / Mixed / Full)
                    └ numer cyklu GC
```
Czego szukać: **częstotliwość** i **czas pauz** (rosnące → problem), **Full GC** (powinno być rzadkie/zero na G1),
oraz czy `heap PO GC` z cyklu na cykl **rośnie** (nie wraca w dół) → sygnał **wycieku**.

### Problemy i wąskie gardła
- **Długie pauzy STW** — zwykle Old/Full GC lub za mały heap. Ratunek: większy `-Xmx`, dostroić `MaxGCPauseMillis`,
  albo przejść na ZGC/Shenandoah. Diagnostyka z logów GC.
- **Wysoki allocation rate** — aplikacja produkuje MB/s śmieci (np. nadmiar krótkotrwałych obiektów, boxing,
  logowanie ze `String` concat) → częste minor GC, presja na CPU. Lek: mniej alokacji, reuse, pooling, unikanie boxingu.
- **Memory leak** — obiekty **retained przez roots**, więc GC ich nie zwolni mimo że logicznie zbędne. Klasyki:
  - **statyczna kolekcja** rosnąca w nieskończoność (`static Map cache` bez eviction),
  - **niezamknięte zasoby** (strumienie, połączenia, wątki) — trzymane przez wewnętrzne referencje,
  - **listenery/callbacki** dopisane i nigdy nie odpięte (`addListener` bez `remove`),
  - `ThreadLocal` w puli wątków bez `remove()`.
  Objaw: heap PO Full GC stale rośnie → w końcu `OutOfMemoryError: Java heap space`.
- **Humongous objects w G1** — duże obiekty (> ½ regionu) fragmentują regiony i wymuszają drogie zbiórki.
  Lek: większy region (`-XX:G1HeapRegionSize`), rozbicie dużych struktur, albo ZGC (nie ma tego problemu).

### finalize() → Cleaner / try-with-resources
`Object.finalize()` jest **deprecated for removal** (od Java 9, oznaczony do usunięcia) — nieprzewidywalny
(nie wiadomo *czy* i *kiedy* się wykona), spowalnia GC, może wskrzeszać obiekty. **Nie używaj.** Zamiast:
- **`try-with-resources`** + `AutoCloseable` — deterministyczne zwolnienie zasobu na końcu bloku (preferowane),
- **`java.lang.ref.Cleaner`** — rejestrujesz akcję sprzątającą wykonywaną, gdy obiekt stanie się *phantom-reachable*
  (siatka bezpieczeństwa dla native/zasobów, gdy ktoś zapomni `close()`).

### Jak diagnozować
- **JFR (JDK Flight Recorder)** — `-XX:StartFlightRecording`, ciągły, tani profiler zdarzeń GC/alokacji;
  analiza w **JMC (JDK Mission Control)**. Najlepsze pierwsze narzędzie do allocation rate i pauz.
- **Heap dump** — `jmap -dump:live,format=b,file=heap.hprof <pid>` lub `-XX:+HeapDumpOnOutOfMemoryError`;
  analiza w **Eclipse MAT** → *dominator tree* i **path to GC roots** wskazuje, co retainuje wyciekające obiekty.
- **`jmap -histo <pid>`** — szybki histogram: które klasy zajmują najwięcej. **`jstat -gc <pid>`** — statystyki GC na żywo.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)
Spring Boot API na k8s zaczyna mieć rosnące p99 i pod pada z `OutOfMemoryError`. Kroki: włączasz `-Xlog:gc*`
i widzisz coraz częstsze **Full GC**, przy czym heap **po GC stale rośnie**. Robisz **heap dump**
(`-XX:+HeapDumpOnOutOfMemoryError`), otwierasz w **MAT** i przez *path to GC roots* okazuje się, że statyczny
`Map<UserId, Session>` (cache bez TTL) retainuje setki tysięcy sesji — klasyczny **memory leak przez static root**.
Fix: eviction/`Caffeine` z limitem. Osobno, żeby ściąć pauzy p99, migrujesz z G1 na **ZGC** (`-XX:+UseZGC`) i pauzy
spadają z ~150 ms do <1 ms.

---

## ✅ Kryteria opanowania
*Temat DOMKNIĘTY, gdy odpowiesz na wszystko BEZ zaglądania.*

- [ ] Wyjaśnię **osiągalność vs reference counting** i czemu GC radzi sobie z cyklami.
- [ ] Wymienię **GC roots** (stosy wątków, static, JNI, aktywne wątki).
- [ ] Odróżnię **mark-sweep / mark-sweep-compact / copying** i wiem, kiedy który wygrywa.
- [ ] Wyjaśnię **weak generational hypothesis** i skąd young/old, minor vs full GC, promocję.
- [ ] Wiem, czym jest **stop-the-world** i safepoint oraz czemu to problem.
- [ ] Dobiorę kolektor (Serial/Parallel/G1/ZGC/Shenandoah/Epsilon) i znam ich flagi.
- [ ] Rozpoznam w logach GC **wyciek** i wymienię jego typowe źródła.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co dokładnie robi faza *mark*, *sweep*, *compact*? Czemu copying jest szybki mimo kopiowania?
- [ ] Dlaczego minor GC jest krótki, a Full GC drogi? (żywych mało w Young)
- [ ] Jak G1 realizuje `MaxGCPauseMillis`? Co to *mixed collection* i *humongous object*?
- [ ] Jak ZGC/Shenandoah osiągają pauzy sub-ms? (concurrent + colored pointers / load barriers)
- [ ] Czemu `finalize()` jest zły i czym go zastąpić? (Cleaner / try-with-resources)
- [ ] Jak w heap dumpie znaleźć przyczynę wycieku? (*path to GC roots*)

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Jak JVM decyduje, że obiekt można zebrać?" (osiągalność z GC roots, nie null/liczenie ref.)
- [ ] „Young vs Old, minor vs major/full GC — po co ten podział?"
- [ ] „Który kolektor jest domyślny w Java 21 i dlaczego?" (G1 — balans)
- [ ] „Masz twarde SLA na latency — jaki GC?" (ZGC/Shenandoah, concurrent, sub-ms)
- [ ] „Aplikacja ma memory leak mimo GC — jak to możliwe i jak zdiagnozować?"
- [ ] „Czemu nie używać `finalize()`?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/02-jvm.csv`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Po co jest Garbage Collector?;Automatycznie zwalnia pamięć obiektów nieosiągalnych z GC roots — nie ma ręcznego free jak w C/C++.
Jak JVM decyduje, że obiekt jest śmieciem?;Gdy jest nieosiągalny przez żaden łańcuch referencji z GC roots (reachability, nie reference counting).
Wymień GC roots.;Zmienne lokalne/parametry na stosach wątków, statyczne pola klas, referencje JNI, aktywne wątki, monitory (synchronized).
Czemu GC z osiągalnością radzi sobie z cyklami?;Bo liczy się dojście z roots, a nie liczba referencji — cykl nieosiągalny z roots jest zbierany, mimo że obiekty wskazują na siebie.
Na czym polega mark-sweep?;Mark: oznacz żywe idąc od roots; Sweep: zwolnij nieoznaczone. Zostawia fragmentację.
Czym różni się mark-sweep-compact od mark-sweep?;Dokłada fazę compact — upycha żywe obiekty obok siebie, eliminuje fragmentację (szybka alokacja bump-the-pointer).
Na czym polega algorytm copying?;Kopiuje żywe obiekty do drugiej połowy heapa, starą czyści hurtem. Praca proporcjonalna do liczby żywych, brak fragmentacji, koszt ~2x pamięci.
Co mówi weak generational hypothesis?;Większość obiektów umiera młodo — stąd podział heapa na young (częsty tani GC) i old (rzadki drogi GC).
Czym różni się minor GC od full GC?;Minor GC sprząta tylko Young (copying, częsty, krótki); Full GC sprząta całość (Young+Old+metaspace, rzadki, drogi).
Co to promocja (tenuring) obiektu?;Awans obiektu z Young do Old, gdy przeżyje próg minor GC (MaxTenuringThreshold) lub nie mieści się w Survivorze.
Co to stop-the-world (STW)?;Pauza, w której GC zatrzymuje wszystkie wątki aplikacji na safepoint, by bezpiecznie przejść po grafie obiektów.
Który GC jest domyślny w Java 21 i czym się cechuje?;G1 — regionowy, balans pauzy/throughput, sterowany celem pauzy MaxGCPauseMillis, robi mixed collections.
Co to humongous object w G1?;Obiekt większy niż połowa regionu — trafia do specjalnych humongous regions, powoduje fragmentację i może wymuszać Full GC.
Jak ZGC i Shenandoah osiągają pauzy sub-ms?;Prawie cała praca (mark, relocation) jest concurrent; używają colored pointers / load(read) barriers, STW to tylko krótkie fazy sync — pauzy <1ms niezależne od rozmiaru heapa.
Do czego służy kolektor Epsilon?;No-op GC: alokuje, ale nie zwalnia pamięci; do testów wydajności/allocation rate — proces padnie na OOM.
Kiedy wybrać Parallel, a kiedy ZGC?;Parallel gdy liczy się throughput (batch), ZGC gdy liczy się niska latency/p99 i duże heapy.
Podaj typowe źródła memory leak w Javie.;Statyczne rosnące kolekcje/cache bez eviction, niezamknięte zasoby, nieodpięte listenery/callbacki, ThreadLocal w puli wątków bez remove().
Czemu finalize() jest zły i czym go zastąpić?;Jest deprecated for removal — nieprzewidywalny, spowalnia GC, może wskrzeszać obiekty. Zamiast: try-with-resources (AutoCloseable) lub Cleaner.
Jak zdiagnozować memory leak?;JFR/JMC (allocation, pauzy), heap dump (jmap / HeapDumpOnOutOfMemoryError) analizowany w MAT — path to GC roots pokazuje, co retainuje obiekty.
Jakie flagi ustawiają rozmiar heapa i kolektor?;-Xms/-Xmx (rozmiar heapa), -XX:+UseG1GC/UseZGC/UseParallelGC (kolektor), -XX:MaxGCPauseMillis (cel pauzy G1).
Jak włączyć logi GC w nowoczesnym JDK?;-Xlog:gc* (Unified Logging) — zastąpiło stare -XX:+PrintGCDetails.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/02-jvm/model-pamieci]] · [[wiedza/00-fundament/jdk-jre-jvm]] · [[wiedza/02-jvm/jit]] · [[wiedza/02-jvm/classloading]]
