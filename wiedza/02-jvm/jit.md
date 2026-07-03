---
temat: "Kompilacja JIT w JVM (HotSpot)"
faza: 2
status: nieopanowany
priorytet: 🟡
tags: [java, jvm]
powiazane: ["[[wiedza/00-fundament/jdk-jre-jvm]]", "[[wiedza/02-jvm/classloading]]", "[[wiedza/02-jvm/model-pamieci]]"]
---

# Kompilacja JIT w JVM (HotSpot)

> **TL;DR:** JVM startuje **interpretując** bytecode (natychmiast, ale wolno), a w tle **profiluje** i kompiluje „gorący" kod do natywnego kodu maszynowego przez **tiered compilation** (poziom 0 = interpreter → C1 z profilowaniem → C2 mocno optymalizujący). JIT stosuje **spekulatywne** optymalizacje (inlining, escape analysis, devirtualizacja) oparte na profilu; gdy założenie przestaje być prawdziwe, następuje **deoptymalizacja** (uncommon trap) i powrót do interpretera. Cena: **warm-up** — dlatego mikrobenchmarki wymagają JMH.

## 1. Co — definicja i API

**JIT (Just-In-Time compilation)** to kompilacja bytecode do kodu maszynowego **w czasie działania** programu, a nie z wyprzedzeniem. HotSpot nie kompiluje wszystkiego od razu — bazuje na obserwacji, że w typowym programie **niewielki procent kodu** odpowiada za większość czasu wykonania („hot spots", stąd nazwa maszyny wirtualnej). Ten gorący kod warto skompilować agresywnie; resztę taniej zinterpretować.

Nie ma „API" JIT w kodzie aplikacji — sterujesz nim **flagami VM** i obserwujesz **diagnostyką**:

```bash
# co i kiedy się kompiluje (poziomy 1-4, deopty oznaczone 'made not entrant')
java -XX:+PrintCompilation MyApp

# jakie metody zostały (nie)zainlinowane i dlaczego
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining MyApp

# rozmiar i zajętość code cache
java -XX:+PrintCodeCache MyApp
```

W Javie 21 domyślnie działa **tiered compilation** z dwoma kompilatorami: **C1** (client, szybki, lekkie optymalizacje) i **C2** (server, wolny, maksymalnie optymalizujący).

## 2. Jak — tiered compilation i optymalizacje pod spodem

### Interpretacja vs kompilacja — czemu JVM zaczyna od interpretera

Interpreter rusza **od razu** (zero kosztu startowego, żaden kod nie musi być wcześniej skompilowany) i — co kluczowe — **zbiera profil** (które gałęzie się wykonują, jakie typy realnie mają obiekty, jak często wołana jest metoda). Kompilacja jest droga (czas CPU, pamięć). Gdyby JVM kompilowała wszystko od startu, płaciłaby za kod wykonany raz i **nie miałaby danych profilowych** do dobrych spekulacji. Interpreter to więc jednocześnie „szybki start" i „faza zbierania wiedzy".

### Tiered compilation — poziomy i przejścia

Domyślnie (od JDK 8) JVM łączy C1 i C2 w pięć poziomów:

```
Level 0  Interpreter                      — zbiera liczniki i profil
Level 1  C1 bez profilowania              — kod trywialny (getter), pełna optymalizacja C1, brak profilu
Level 2  C1 z licznikami wywołań/back-edge
Level 3  C1 z pełnym profilowaniem        — najczęstsza droga „w górę"
Level 4  C2 — pełne, agresywne optymalizacje
```

Typowa ścieżka gorącej metody: **0 → 3 (C1+profil) → 4 (C2)**. C1 kompiluje szybko i już przyspiesza, a jednocześnie **profiluje**, żeby C2 dostał bogaty profil. Gdy kolejka C2 jest zapchana, metoda może iść **0 → 2 → 3 → 4**. Poziom 1 dostają metody uznane za trywialne (nie ma sensu ich profilować). Poziomy dobiera VM adaptacyjnie na podstawie obciążenia kompilatorów.

### Wykrywanie „gorącego" kodu — liczniki i progi

HotSpot utrzymuje na metodę dwa liczniki:
- **invocation counter** — ile razy metodę wywołano,
- **back-edge counter** — ile razy wykonano skok wstecz (iterację pętli).

Gdy suma przekroczy próg (`-XX:CompileThreshold`, historycznie ~10000 dla C2; w tiered progi są niższe i sterowane osobnymi flagami jak `Tier3InvocationThreshold`), metoda trafia do kolejki kompilacji. Back-edge counter jest osobno ważny, bo pętla może „nagrzać się" w pojedynczym wywołaniu metody.

### OSR (On-Stack Replacement)

Problem: `main()` z długą pętlą jest wywołany **raz**, więc invocation counter nigdy nie urośnie, a pętla może się kręcić minutami. Rozwiązanie: gdy **back-edge counter** przekroczy próg, JVM kompiluje pętlę i podmienia wykonanie **w locie, w środku działającej ramki stosu** — stan (zmienne lokalne, licznik pętli) jest przeniesiony z ramki interpretera do skompilowanego kodu. Metoda „w połowie biegu" nagle zaczyna wykonywać kod natywny. To OSR. W `PrintCompilation` widoczne jako `% ` przy metodzie.

### Profile-Guided Optimization (PGO)

JIT nie zgaduje — **mierzy w runtime**. Przy każdym `if` wie, która gałąź jest brana; przy każdym `invokevirtual`/`invokeinterface` wie, jakie **konkretne typy** faktycznie przewijały się przez call site (tzw. type profile: monomorphic / bimorphic / megamorphic). Ta wiedza napędza optymalizacje niżej. Kluczowa różnica wobec statycznego kompilatora: PGO jest **adaptacyjne** i dopasowane do rzeczywistego ruchu, a nie do przypadków wyobrażonych przez programistę.

### Kluczowe optymalizacje

- **Inlining** — wklejenie ciała wywoływanej metody w miejsce wywołania. To „matka optymalizacji": usuwa koszt wywołania, ale głównie **odsłania kolejne optymalizacje** (EA, constant folding, DCE działają dopiero po zestawieniu kodu razem). Limity: metoda musi być dość mała (`-XX:MaxInlineSize`, ~35 bajtów bytecode; gorące do `FreqInlineSize`, ~325), jest limit głębokości i rozmiaru rozrastającego się callee. Dlatego wielkie metody bywają wolniejsze — nie mieszczą się w budżecie inliningu.
- **Devirtualizacja** — wywołania wirtualne (`invokevirtual`/`invokeinterface`) są domyślnie dynamiczne. Jeśli type profile pokazuje **jeden** typ (monomorphic call site — częste w praktyce, bo interfejs ma jedną implementację), C2 zamienia je na wywołanie bezpośrednie i inlinuje. Wstawia **guard** sprawdzający typ; jeśli guard zawiedzie → deopt.
- **Escape analysis + scalar replacement** — jeśli C2 udowodni, że obiekt **nie ucieka** poza metodę (nie trafia na heap poza jej zasięg), może:
  - **scalar replacement** — nie alokować obiektu wcale, trzymać jego pola w rejestrach (zero pracy dla GC!),
  - **lock elision** — usunąć synchronizację na obiekcie widocznym tylko z jednego wątku,
  - **lock coarsening** — scalić kolejne `synchronized` na tym samym monitorze w jeden większy blok.
- **Loop unrolling** — rozwinięcie iteracji pętli (mniej sprawdzeń warunku i skoków, lepsze wykorzystanie potoku CPU, umożliwia wektoryzację).
- **Branch prediction z profilu** — gałęzie „prawie nigdy nie brane" są przenoszone poza główną ścieżkę (kod ustawiony tak, by CPU dobrze przewidywał).
- **Dead Code Elimination (DCE)** — usunięcie kodu bez efektów obserwowalnych. To zmora benchmarków (patrz niżej).

### Deoptymalizacja (uncommon traps)

Optymalizacje C2 są **spekulatywne** — opierają się na założeniach z profilu („ten call site jest monomorphic", „ta klasa nie ma podklas", „ta gałąź nigdy nie jest brana"). Gdy założenie **przestaje być prawdziwe** (np. classloader doładował nową podklasę, pojawił się nowy typ na call site, weszła „niemożliwa" gałąź), skompilowany kod trafia w **uncommon trap**: wykonanie **wraca do interpretera**, ramka skompilowana jest zrekonstruowana do stanu interpretera, a metoda oznaczana `made not entrant` (nikt nowy w nią nie wejdzie). Później może zostać ponownie skompilowana z uaktualnionym profilem. To właśnie deoptymalizacja daje JIT-owi odwagę do agresywnych spekulacji — jest bezpieczna sieć.

### Code cache

Skompilowany kod natywny leży w **code cache** (osobny obszar pamięci, nie heap; `-XX:ReservedCodeCacheSize`, domyślnie 240 MB przy tiered). Gdy się **zapełni**: JVM przestaje kompilować, w logu pojawia się `CodeCache is full. Compiler has been disabled`, aplikacja **cofa się do interpretera** i drastycznie zwalnia. HotSpot ma mechanizm **sweepera** i (od JDK 9) **segmentowany** code cache (non-nmethods / profiled / non-profiled), by lepiej zarządzać miejscem i eksmitować martwe metody.

## 3. Dlaczego / kiedy — warm-up, benchmarki, wąskie gardła

- **Warm-up** — bezpośrednia konsekwencja modelu JIT. Pierwsze wywołania idą przez interpreter, potem C1, dopiero po tysiącach wywołań C2. Skutki:
  - pierwsze żądania do serwisu są **wolniejsze** (latency spikes na starcie),
  - **serverless / CLI / krótko żyjące** procesy często nie zdążą się rozgrzać → tracą na tym najwięcej.
- **Mikrobenchmarki są trudne** — naiwny „zmierz `System.nanoTime()` wokół pętli" daje bezużyteczne liczby, bo:
  - mierzysz **interpreter/C1**, nie C2 (brak warm-upu),
  - **DCE** wywala kod, którego wynik nigdzie nie trafia („martwy kod eliminowany") — mierzysz pustą pętlę,
  - **constant folding** liczy wynik w czasie kompilacji, jeśli wejście jest stałą,
  - **OSR** kompiluje pętlę benchmarku inaczej niż realny kod byłby kompilowany.
  Dlatego używa się **JMH** (Java Microbenchmark Harness): robi rozgrzewkę (warmup iterations), konsumuje wyniki przez `Blackhole` (blokuje DCE), izoluje state, forkuje JVM. Reguła: **nie ufaj mikrobenchmarkowi bez JMH**.
- **Wąskie gardła i kontrola** — jeśli latencja startu jest krytyczna, można podnieść próg C1/wymusić szybsze tiery, albo zmniejszyć nastawienie na C2. Za dużo kompilacji → presja na code cache i wątki kompilatora.
- **Kontrast z AOT / GraalVM native image** — native image kompiluje **z wyprzedzeniem** (ahead-of-time), przed uruchomieniem:
  - **plusy:** brak warm-upu, start w milisekundach, mniejszy RAM (idealne pod serverless/CLI/skalowanie),
  - **minusy:** brak **adaptacyjnych optymalizacji profilowych** — kompilator nie widzi realnego ruchu, więc nie zrobi devirtualizacji z profilu ani spekulatywnego inliningu na podstawie „co się dzieje teraz". Ograniczenia reflection/dynamiki (closed-world). Na **long-running throughput** klasyczny JIT (C2) zwykle **wygrywa**, bo ma profil i deopty.
  - GraalVM oferuje też **PGO dla native image** (profil zbierany z osobnego biegu instrumentowanego), co częściowo domyka tę lukę — ale nie jest to adaptacja *w locie*.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Wdrażasz Spring Boot API na k8s. Obserwujesz, że **pierwsze 30 sekund po deployu** p99 latency jest kilkukrotnie wyższe — to warm-up JIT + lazy class loading. Reakcje: readiness probe z rozgrzewającym ruchem, ewentualnie **AppCDS** na start klas, a dla funkcji o krótkim życiu rozważasz **native image**. Diagnozując „czemu ta metoda nie przyspiesza", odpalasz `-XX:+PrintCompilation` i widzisz `made not entrant` w pętli — okazuje się, że doładowanie pluginu tworzy **megamorphic** call site i C2 nie może zdewirtualizować, co dobija do deoptów. Zespół pisze mikrobenchmark „nasz mapper vs MapStruct" bez JMH i dostaje absurdalne 0 ns — bo DCE wywaliło wynik; przepisujecie na JMH z `Blackhole`.

---

## ✅ Kryteria opanowania
*Temat jest DOMKNIĘTY, gdy odpowiesz na wszystkie BEZ zaglądania.*

- [ ] Wyjaśnię, czemu JVM zaczyna od interpretera zamiast kompilować od razu.
- [ ] Opiszę poziomy tiered compilation (0–4) i typową ścieżkę 0→3→4.
- [ ] Wiem, jak wykrywany jest gorący kod (invocation + back-edge counter) i czym jest OSR.
- [ ] Wymienię kluczowe optymalizacje C2 (inlining, EA + scalar replacement, devirtualizacja) i rozumiem, czemu inlining jest „matką optymalizacji".
- [ ] Wyjaśnię deoptymalizację (uncommon trap) i po co jest — dlaczego umożliwia agresywne spekulacje.
- [ ] Rozumiem warm-up i czemu mikrobenchmark bez JMH kłamie (DCE, constant folding, brak rozgrzewki).
- [ ] Porównam JIT z AOT/GraalVM native image (warm-up vs adaptacyjne PGO).

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Czym różni się interpretacja od kompilacji JIT i kto zbiera profil?
- [ ] Co robi C1, a co C2, i po co dwa kompilatory zamiast jednego?
- [ ] Jak liczniki (invocation/back-edge) decydują o kompilacji i czym jest próg?
- [ ] Jak działa OSR — czemu jest potrzebny dla długiej pętli w `main`?
- [ ] Co to escape analysis i jak prowadzi do scalar replacement / lock elision / coarsening?
- [ ] Co to uncommon trap i co wywołuje deoptymalizację?
- [ ] Co się dzieje, gdy code cache się zapełni?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Czemu pierwsze żądania są wolniejsze?" (warm-up: interpreter → C1 → C2)
- [ ] „Co to tiered compilation?"
- [ ] „Czemu mikrobenchmarki w Javie są trudne i po co JMH?"
- [ ] „Co to deoptymalizacja i kiedy zachodzi?"
- [ ] „JIT vs AOT/native image — kiedy co wybrać?"
- [ ] „Co to inlining i dlaczego jest tak ważny?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Czemu JVM zaczyna od interpretera, a nie od kompilacji?;Interpreter rusza natychmiast (zero kosztu startu) i zbiera profil runtime; kompilacja jest droga i bez profilu dawałaby słabe spekulacje.
Co to tiered compilation w HotSpot?;Model łączący C1 i C2 w poziomy 0-4: 0=interpreter, 1-3=C1 (z profilowaniem), 4=C2 mocno optymalizujący.
Jaka jest typowa ścieżka gorącej metody przez poziomy?;0 (interpreter) → 3 (C1 z pełnym profilem) → 4 (C2 z agresywnymi optymalizacjami).
Czym różni się C1 od C2?;C1 kompiluje szybko z lekkimi optymalizacjami (i profiluje); C2 kompiluje wolno, ale maksymalnie optymalizuje na podstawie profilu.
Jak HotSpot wykrywa gorący kod?;Dwoma licznikami na metodę: invocation counter (liczba wywołań) i back-edge counter (iteracje pętli); po przekroczeniu progu metoda idzie do kompilacji.
Co to OSR (On-Stack Replacement)?;Kompilacja i podmiana wykonania długiej pętli w locie, w środku działającej ramki stosu, gdy nagrzeje ją back-edge counter (np. pętla w main wywołanym raz).
Co to profile-guided optimization w JIT?;JIT mierzy w runtime, która gałąź if jest brana i jakie typy przewijają się przez call site, i na tej podstawie optymalizuje adaptacyjnie.
Dlaczego inlining nazywa się "matką optymalizacji"?;Sam usuwa koszt wywołania, ale głównie odsłania kolejne optymalizacje (escape analysis, constant folding, DCE) działające na zestawionym kodzie.
Jakie są limity inliningu w HotSpot?;Rozmiar metody (MaxInlineSize ~35 B, gorące FreqInlineSize ~325 B), głębokość i budżet rozrostu callee; duże metody się nie inlinują.
Co daje escape analysis, gdy obiekt nie ucieka z metody?;Scalar replacement (brak alokacji, pola w rejestrach), lock elision (usunięcie zbędnej synchronizacji) i lock coarsening (scalenie sąsiednich synchronized).
Co to devirtualizacja?;Zamiana wywołania wirtualnego na bezpośrednie (i inlining), gdy type profile pokazuje jeden typ (monomorphic); z guardem typu, którego porażka daje deopt.
Co to deoptymalizacja / uncommon trap?;Powrót skompilowanego kodu do interpretera, gdy spekulatywne założenie przestaje być prawdziwe; metoda oznaczana "made not entrant" i może być rekompilowana.
Po co istnieje deoptymalizacja?;Jest siecią bezpieczeństwa, która pozwala C2 robić agresywne, spekulatywne optymalizacje — bo złe założenie da się bezpiecznie cofnąć.
Co się dzieje, gdy zapełni się code cache?;JVM przestaje kompilować (log "CodeCache is full"), aplikacja cofa się do interpretera i drastycznie zwalnia; steruje tym ReservedCodeCacheSize.
Czemu mikrobenchmark bez JMH kłamie?;Brak warm-upu (mierzy interpreter/C1), Dead Code Elimination usuwa nieużyty wynik, constant folding liczy stałe na etapie kompilacji, OSR kompiluje pętlę inaczej.
Jak JMH zapobiega usunięciu wyniku przez DCE?;Konsumuje wyniki przez Blackhole (i robi warmup iterations, forki, izolację state), więc kod nie jest martwy.
Jaka jest główna wada AOT/GraalVM native image wobec JIT?;Brak adaptacyjnych optymalizacji profilowych z runtime (devirtualizacja/spekulacje z realnego ruchu); na long-running throughput C2 zwykle wygrywa.
Jaka jest główna zaleta native image wobec JIT?;Brak warm-upu, start w milisekundach i mniejszy RAM — idealne pod serverless/CLI i krótko żyjące procesy.
Jakimi flagami podejrzeć kompilacje i inlining?;-XX:+PrintCompilation oraz -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/00-fundament/jdk-jre-jvm]] · [[wiedza/02-jvm/classloading]] · [[wiedza/02-jvm/model-pamieci]] · [[wiedza/02-jvm/garbage-collection]] · [[wiedza/10-cloud-native/konteneryzacja]]
