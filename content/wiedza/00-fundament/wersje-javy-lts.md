---
temat: "Wersje Javy, cykl wydawniczy i LTS"
faza: 0
status: nieopanowany
priorytet: 🔴
tags: [java, fundament, lts, jdk]
powiazane: ["[[wiedza/00-fundament/jdk-jre-jvm]]", "[[wiedza/00-fundament/sdkman]]", "[[wiedza/01-jezyk/record-sealed-enum]]", "[[wiedza/01-jezyk/virtual-threads]]"]
---

# Wersje Javy, cykl wydawniczy i LTS

> **TL;DR:** Od Javy 9 nowy **feature release** wychodzi **co 6 miesięcy** (np. 24, 25), ale wsparcie produkcyjne na lata mają tylko wydania **LTS** (**8, 11, 17, 21**, następne **25** we wrześniu 2025) — i to na nich siedzą firmy. Do nowego projektu w 2026 celuj w **Java 21 LTS** (virtual threads + nowoczesna składnia).

## 1. Co — release cadence vs LTS

Do Javy 8 wydania były nieregularne (lata przerwy, „big bang"). Od **Javy 9 (2017)** obowiązuje **time-based release cadence**: co **6 miesięcy** wychodzi pełnoprawny **feature release** (marzec i wrzesień), z numerem rosnącym o 1. Data jest sztywna — **funkcja gotowa na czas wjeżdża, niegotowa czeka pół roku** (model „train", nie „feature freeze in indefinitely").

- **Feature release** (np. 22, 23, 24) — wsparcie tylko do następnego wydania (~6 mies.). Świetne do nauki/eksperymentów, **ryzykowne na produkcji** (musisz aktualizować co pół roku).
- **LTS (Long-Term Support)** — wybrane wydania z **wieloletnim wsparciem** (patche bezpieczeństwa, backporty). Kadencja LTS: początkowo co 3 lata (**11 → 17 → 21**, wcześniej **8**), od Javy 21 ustalono **co 2 lata**, więc kolejny LTS to **25 (wrzesień 2025)**, potem 29 itd.

```
8        11           17           21      25      (LTS — produkcja, wsparcie na lata)
●━━━━━━━━●━━━━━━━━━━━━●━━━━━━━━━━━━●━━━━━━━●━━━━━ ...
2014    2018         2021         2023    2025
   9 10  12 ... 16   18 19 20    22 23 24 ...     (feature — co 6 mies., wsparcie ~6 mies.)
```

**LTS to pojęcie wsparcia/dystrybucji, nie składni języka** — sama specyfikacja Javy nie zna „LTS". To dostawcy (Oracle, Adoptium, Red Hat…) zobowiązują się łatać dane wydanie dłużej. Dlatego „Java 21 LTS" znaczy: *wydanie 21, dla którego dostaniesz aktualizacje 21.0.x przez kilka lat*.

## 2. Jak — co realnie zmieniło się 8 → 11 → 17 → 21

Każdy skok między LTS-ami to suma feature releów pomiędzy nimi. Najważniejsze:

**Java 8 (2014) — rewolucja funkcyjna.** Punkt odniesienia całej „nowoczesnej" Javy:
- **Lambdy** (`x -> x * 2`) i **funkcyjne interfejsy** (`Function`, `Predicate`, `Supplier`).
- **Stream API** (`list.stream().filter(...).map(...).collect(...)`) — deklaratywne przetwarzanie kolekcji.
- `Optional`, `default` metody w interfejsach, nowe **Date/Time API** (`java.time`).

**Java 9–11 (przejście 8 → 11) — modularyzacja i porządki:**
- **JPMS** (Java Platform Module System, „Project Jigsaw", J9) — `module-info.java`, silna enkapsulacja (`exports`, `requires`). Rozbił monolityczny `rt.jar` na **moduły** (`java.base`…). W praktyce większość appek go **nie używa** wprost, ale rozbił JDK i umożliwił `jlink`.
- **`var`** (J10) — **local variable type inference**: `var list = new ArrayList<String>();` (typ wnioskowany, *nie* dynamiczny — statyczne typowanie pozostaje).
- **`HttpClient`** (J11) — nowoczesny klient HTTP/2 zamiast `HttpURLConnection`. Usunięto przestarzałe moduły (CORBA, Java EE).
- **Uruchamianie pliku źródłowego** bez kompilacji: `java Hello.java` (J11).

**Java 12–17 (przejście 11 → 17) — nowoczesna składnia (data-oriented):**
- **Records** (`record Point(int x, int y) {}`, J16) — niemutowalne **data carriers**; kompilator generuje konstruktor, `equals`/`hashCode`/`toString`, akcesory. Zob. [[wiedza/01-jezyk/record-sealed-enum]].
- **Sealed classes/interfaces** (`sealed ... permits ...`, J17) — kontrola, kto może dziedziczyć (zamknięta hierarchia → bezpieczne `switch`).
- **Pattern matching for `instanceof`** (J16): `if (o instanceof String s) {...}` — bez ręcznego castu.
- **Text blocks** (`"""..."""`, J15) — wieloliniowe stringi (JSON/SQL/HTML bez sklejania).
- **Switch expressions** (`var r = switch(x) { case A -> 1; ... };`, J14) — zwracają wartość, strzałkowa składnia.
- **Helpful NullPointerExceptions** (J14) — komunikat mówi, *która* zmienna była `null`.
- Nowe GC: **ZGC** i **Shenandoah** (low-latency).

**Java 18–21 (przejście 17 → 21) — współbieżność i dokończenie pattern matchingu:**
- **Virtual Threads** (J21, „Project Loom") — lekkie wątki zarządzane przez JVM, tysiące/miliony na jednej platformie. Rozwiązują problem „thread-per-request" bez async/reaktywności. Zob. [[wiedza/01-jezyk/virtual-threads]].
- **Pattern matching for `switch`** (J21) — `switch (shape) { case Circle c -> ...; case Square s -> ...; }`, w komplecie z **record patterns** (`case Point(int x, int y) ->`) i sealed (kompletność sprawdzana statycznie).
- **Sequenced Collections** (J21) — `getFirst()`/`getLast()`/`reversed()` na uporządkowanych kolekcjach.
- **Structured Concurrency** i **Scoped Values** — jako **preview** (patrz niżej).

### preview / incubator features
Część nowości wchodzi „na próbę", by zebrać feedback przed ustabilizowaniem API:
- **Preview features** — gotowe funkcje *języka/API*, ale API może się jeszcze zmienić. Wymagają **dwóch flag**: kompilacja `javac --release 21 --enable-preview ...` i uruchomienie `java --enable-preview ...`. Bez tego — błąd kompilacji. **Nie używaj na produkcji** (API może zniknąć/zmienić się w następnym wydaniu).
- **Incubator modules** — całe nowe *moduły/API* (np. dawne Vector API, Foreign Function & Memory) w pakiecie `jdk.incubator.*`, wymagają `--add-modules`.
- Cykl bywa: *incubator → preview → preview (2. runda) → standard*. Np. Virtual Threads były preview w 19/20, **finalne w 21**.

## 3. Dlaczego / kiedy — kompromisy: dystrybucje i licencje

„JDK 21" to **specyfikacja + referencyjny kod OpenJDK**; pod tą samą wersją istnieje wiele **buildów** różnych dostawców. Różnią się głównie **licencją, supportem i dodatkami**, a nie składnią języka.

| Dystrybucja | Kto | Licencja | Uwagi |
|---|---|---|---|
| **OpenJDK (oracle.com/openjdk)** | społeczność/Oracle | **GPLv2 + Classpath Exception** | Referencyjne buildy, tylko bieżąca wersja |
| **Oracle JDK** | Oracle | **NFTC** (od 17) / wcześniej komercyjna | Patrz niżej — pułapka licencyjna |
| **Eclipse Temurin (Adoptium)** | Eclipse Foundation | GPLv2+CE | **Domyślny wybór** — neutralny, darmowy, LTS-y długo wspierane |
| **Red Hat build of OpenJDK** | Red Hat | GPLv2+CE | Wsparcie w ekosystemie RHEL/OpenShift |
| **Amazon Corretto** | Amazon | GPLv2+CE | Darmowy LTS, dobry w AWS |
| **Azul Zulu / Platform Core** | Azul | GPLv2+CE | Darmowy + płatny support; Zing (C4 GC) komercyjny |

**Kwestia licencyjna (kluczowa pułapka):**
- **OpenJDK buildy** (Temurin, Corretto, Red Hat, Azul Zulu) → **GPLv2 + Classpath Exception**: darmowe do dowolnego użytku, w tym produkcyjnego, bez „zarażania" GPL Twojego kodu (to robi Classpath Exception).
- **Oracle JDK** → od Javy 17 na licencji **NFTC** (Oracle No-Fee Terms and Conditions): darmowe do produkcji, **ale tylko dla danej wersji**; po zakończeniu darmowego okresu dla wydania dalsze patche wymagają **płatnej subskrypcji**. Wcześniej (Java 11–16) Oracle JDK na produkcji **wymagał płatnej licencji** — stąd masowa migracja firm na OpenJDK buildy.
- **Reguła praktyczna:** jeśli nie kupujesz świadomie supportu Oracle, **bierz Temurin/Corretto** i nie ryzykujesz auditem licencyjnym.

**Czemu firmy zostają na LTS:**
- **Stabilność i support** — patche bezpieczeństwa na lata bez przepisywania pod nową składnię co 6 mies.
- **Zgodność ekosystemu** — frameworki (Spring, Hibernate), narzędzia i biblioteki certyfikują się i testują głównie na LTS-ach.
- **Koszt migracji** — każdy major skok niesie ryzyko (usunięte API, zmiany w reflection/modułach, zachowanie GC). Robienie tego co pół roku jest nieopłacalne dla dużych systemów.

**Jak wybrać wersję do nowego projektu (2026):**
- **Domyślnie: Java 21 LTS.** Virtual threads + records + pattern matching + sealed = nowoczesny, ergonomiczny stack, a wsparcie i ekosystem są dojrzałe.
- **25 LTS** rozważ, gdy biblioteki/frameworki już ją w pełni wspierają i potrzebujesz jej nowości — ale na świeżym LTS bywa „za wcześnie" względem ekosystemu.
- **Nie zaczynaj na 8/11** nowego projektu — brak nowoczesnej składni i wygasające wsparcie.

**Migracja między wersjami — pułapki:**
- **Usunięte/zmienione API** — np. wycięte Java EE/CORBA (8→11), zmiany w `sun.misc.Unsafe`, deprecjacje.
- **Silna enkapsulacja JPMS** — od 16/17 odbicie (reflection) do wnętrz JDK jest domyślnie blokowane; stary kod może wymagać `--add-opens`/`--add-exports`. To częsty „illegal reflective access".
- **Zmiana domyślnego GC, parsowania dat, kodowania** (UTF-8 jako domyślne od 18) — subtelne różnice w runtime.
- **Narzędzia migracji:** `jdeps` (analiza zależności i użycia internal API), `jdeprscan` (skan przestarzałych API). Migruj **skokami przez LTS** (8→11→17→21), nie po jednym wydaniu.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)
Startujesz nowy serwis Spring Boot. W `pom.xml`/`build.gradle` ustawiasz `<java.version>21</java.version>`, a lokalnie przez [[wiedza/00-fundament/sdkman|SDKMAN]] instalujesz **Temurin 21** (`sdk install java 21.0.x-tem`). W `Dockerfile` bazujesz na `eclipse-temurin:21-jre`. Dzięki 21 włączasz **virtual threads** w Tomcacie (`spring.threads.virtual.enabled=true`) bez przepisywania na WebFlux. Gdy ktoś zaproponuje Oracle JDK „bo z oracle.com" — przypominasz o pułapce NFTC i zostajesz przy OpenJDK buildzie.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię różnicę między feature release (co 6 mies.) a LTS i podam listę LTS-ów (8/11/17/21/25).
- [ ] Wiem, czemu firmy zostają na LTS i czemu to nie jest decyzja o składni języka.
- [ ] Wymienię flagowe cechy każdego skoku 8→11→17→21 z głowy.
- [ ] Rozumiem `--enable-preview` i różnicę preview vs incubator.
- [ ] Znam różnicę licencyjną Oracle JDK (NFTC) vs OpenJDK buildy (GPL+CE) i potrafię wybrać dystrybucję.
- [ ] Uzasadnię wybór Java 21 LTS do nowego projektu i znam pułapki migracji.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Dlaczego „LTS" nie istnieje w specyfikacji języka, tylko u dostawców? (support, nie składnia)
- [ ] Co konkretnie znaczy GPLv2 **+ Classpath Exception** i czemu chroni Twój kod przed „zarażeniem" GPL?
- [ ] Co naprawdę robi `--enable-preview` na etapie kompilacji **i** uruchomienia (dwie flagi)?
- [ ] Czemu po skoku na 17 pojawia się „illegal reflective access" i co z tym robi `--add-opens`?
- [ ] Czym różni się `var` (J10) od dynamicznego typowania? (type inference, nadal statyczne)

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Czym różni się feature release od LTS i które wersje są LTS?"
- [ ] „Co przyniosła Java 8 / 11 / 17 / 21?" (lambdy/streamy · moduły+var+HttpClient · records/sealed/text blocks · virtual threads)
- [ ] „Którą wersję wybrałbyś do nowego projektu i czemu?"
- [ ] „Oracle JDK czy OpenJDK — czym się różnią licencyjnie?"
- [ ] „Co to feature preview i jak je włączyć?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/00-fundament.csv`.*
```
Co ile wychodzi feature release Javy od wersji 9?;Co 6 miesięcy (marzec i wrzesień), data sztywna — gotowe funkcje wjeżdżają, reszta czeka.
Które wersje Javy są LTS?;8, 11, 17, 21 — następna 25 (wrzesień 2025).
Jaka jest kadencja LTS od Javy 21?;Co 2 lata (wcześniej co 3 lata: 11→17→21), więc kolejne to 25, 29...
Czemu firmy zostają na LTS?;Wieloletnie patche bezpieczeństwa, dojrzały ekosystem/frameworki i niski koszt — bez przepisywania co 6 mies.
Flagowe cechy Javy 8?;Lambdy, Stream API, funkcyjne interfejsy, Optional, java.time, default methods.
Co przyniosło przejście 8→11?;JPMS (moduły), var (J10), HttpClient HTTP/2, uruchamianie pliku .java; usunięto Java EE/CORBA.
Flagowe cechy przejścia 11→17?;Records, sealed classes, pattern matching for instanceof, text blocks, switch expressions, helpful NPE.
Flagowe cechy Javy 21?;Virtual threads (Loom), pattern matching for switch + record patterns, sequenced collections.
Jak włączyć preview feature?;Dwie flagi: javac --release N --enable-preview oraz java --enable-preview.
Czym różni się preview od incubator?;Preview = gotowa funkcja języka/API (możliwa zmiana API); incubator = nowy moduł jdk.incubator.* (wymaga --add-modules).
Na jakiej licencji są OpenJDK buildy (Temurin, Corretto)?;GPLv2 + Classpath Exception — darmowe na produkcji, bez "zarażania" Twojego kodu GPL.
Pułapka licencyjna Oracle JDK?;Od 17 licencja NFTC (darmowa per wersja, potem płatny support); 11-16 Oracle JDK na produkcji wymagał płatnej licencji.
Którą wersję wybrać do nowego projektu w 2026?;Java 21 LTS (virtual threads + nowoczesna składnia, dojrzały ekosystem), dystrybucja Temurin/Corretto.
Czym jest var w Javie?;Local variable type inference (J10) — kompilator wnioskuje typ; typowanie nadal statyczne, nie dynamiczne.
Czym różni się bytecode od LTS?;LTS to model wsparcia dostawcy dla danego wydania, nie cecha języka/bytecode.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/00-fundament/jdk-jre-jvm]] · [[wiedza/00-fundament/sdkman]] · [[wiedza/01-jezyk/record-sealed-enum]] · [[wiedza/01-jezyk/virtual-threads]]
