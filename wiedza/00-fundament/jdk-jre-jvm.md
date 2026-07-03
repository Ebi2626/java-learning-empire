---
temat: "JDK vs JRE vs JVM"
faza: 0
status: w-trakcie
priorytet: 🔴
tags: [java, fundament, jvm]
powiazane: ["[[wiedza/00-fundament/wersje-javy-lts]]", "[[wiedza/02-jvm/jit]]", "[[wiedza/02-jvm/classloading]]"]
---

# JDK vs JRE vs JVM

> **TL;DR:** **JVM** uruchamia bytecode (`.class`), **JRE** = JVM + biblioteki standardowe (środowisko *uruchomieniowe*),
> **JDK** = JRE + narzędzia deweloperskie (kompilator `javac`, `jar`, `jlink`…). Piszesz w JDK, działasz na JVM.

## 1. Co — definicje i API

```
┌──────────────────────── JDK (Java Development Kit) ───────────────────────┐
│  Narzędzia deweloperskie: javac, jar, javadoc, jlink, jdeps, jshell, jfr  │
│  ┌──────────────────── JRE (Java Runtime Environment) ─────────────────┐  │
│  │  Biblioteki standardowe (java.base, java.sql, ... = moduły JDK)     │  │
│  │  ┌──────────────── JVM (Java Virtual Machine) ─────────────────┐    │  │
│  │  │  Classloader · Interpreter · JIT (C1/C2) · GC · Runtime data │    │  │
│  │  └─────────────────────────────────────────────────────────────┘    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
```

- **JVM** — *specyfikacja* (JVM Spec) i jej *implementacja* (HotSpot to najpopularniejsza; są też GraalVM, OpenJ9).
  Wykonuje **bytecode**, niezależny od OS/CPU. To dlatego „write once, run anywhere".
- **JRE** — to, co potrzebne tylko do *uruchamiania*: JVM + biblioteki klas (`String`, `List`, `HttpClient`…).
  Od Javy 11 Oracle **nie wydaje już osobnego JRE** — dystrybuujesz JDK albo własny runtime przez `jlink`.
- **JDK** — pełen pakiet developera. To instalujesz przez [[wiedza/00-fundament/sdkman|SDKMAN]].

Minimalny cykl:
```bash
javac Hello.java     # JDK: źródło .java  → bytecode Hello.class
java Hello           # JRE/JVM: ładuje Hello.class i wykonuje
```

## 2. Jak — co dzieje się pod spodem

Droga od `.java` do wykonania:

1. **Kompilacja (javac, w JDK):** `Hello.java` → `Hello.class`. To **bytecode** — instrukcje dla maszyny
   *stosowej* (np. `iload`, `invokevirtual`, `getstatic`), nie dla fizycznego CPU. Zobacz: `javap -c Hello`.
2. **Uruchomienie (`java`):** startuje proces JVM, który:
   - **Class loading** ([[wiedza/02-jvm/classloading]]): classloadery (bootstrap → platform → application)
     ładują `.class` z classpath/module path, **weryfikują** bytecode (bezpieczeństwo!), linkują, inicjalizują.
   - **Interpreter:** na starcie JVM **interpretuje** bytecode instrukcja po instrukcji (wolno, ale od razu).
   - **JIT** ([[wiedza/02-jvm/jit]]): „gorące" metody (często wywoływane) są kompilowane do **kodu maszynowego**
     (C1 szybko/lekko, potem C2 mocno optymalizująco). Stąd **warm-up** — Java jest wolniejsza w pierwszych ms,
     potem dorównuje C++. To klucz do uczciwego benchmarkowania (rozgrzej JVM albo użyj JMH).
   - **Runtime data areas:** heap (obiekty), stosy wątków, metaspace (metadane klas) — [[wiedza/02-jvm/model-pamieci]].
   - **GC** ([[wiedza/02-jvm/garbage-collection]]): w tle zwalnia nieosiągalne obiekty.

**Bytecode jest przenośny, kod maszynowy nie.** Ten sam `Hello.class` działa na Linux/ARM i Windows/x86 —
różne JVM tłumaczą go na różny kod maszynowy. Przenośność jest na poziomie bytecode, nie źródła.

## 3. Dlaczego / kiedy — kompromisy i wąskie gardła

- **Warm-up / czas startu** — bolączka przy serverless/CLI/funkcjach (krótko żyją, nie zdążą się rozgrzać).
  Rozwiązania: **GraalVM native image** (AOT — kompilacja z wyprzedzeniem, start w ms, mały RAM, ale traci
  dynamikę i część reflection), CDS/AppCDS (współdzielenie metadanych klas), Project Leyden (w toku).
- **Zużycie pamięci** — JVM ma narzut (metaspace, JIT, GC). W kontenerach **ustaw `-Xmx` / `MaxRAMPercentage`**,
  bo inaczej JVM może źle odczytać limit cgroup (nowsze JDK są „container-aware", ale weryfikuj — [[wiedza/10-cloud-native/konteneryzacja]]).
- **„Write once, run anywhere"** w praktyce = „test everywhere" — różnice JVM/wersji/GC bywają istotne.
- **Kiedy native vs JVM:** długo żyjący serwis (Spring Boot na k8s) → klasyczny JVM (JIT wygrywa na throughput);
  krótkie/serverless/CLI → rozważ native image.

## Przykład w praktyce
Budujesz Spring Boot API. W obrazie kontenera dajesz **JRE-podobny runtime** (mały, np. z `jlink`/distroless),
ale CI buduje JARa **JDK-iem**. Diagnozując OOM na produkcji używasz narzędzi z **JDK** (`jmap`, `jfr`, `jstack`) —
dlatego w obrazach debugowych bywa pełen JDK, a w produkcyjnych odchudzony runtime.

---

## ✅ Kryteria opanowania
- [x] Wyjaśnię różnicę JDK/JRE/JVM jednym zdaniem i diagramem.
- [ ] Wiem, czemu bytecode jest przenośny, a kod maszynowy nie.
- [ ] Rozumiem warm-up i jego skutek dla benchmarków.
- [ ] Potrafię uzasadnić wybór native image vs JVM.
- [ ] Wiem, czemu w kontenerze trzeba uważać na pamięć JVM.

### 🔲 Black-box check
- [ ] Co dokładnie robi `javac`, a co `java`? (kompilacja vs ładowanie+wykonanie)
- [ ] Czym różni się interpretacja od JIT i kiedy JVM przełącza się na JIT?
- [ ] Co to bytecode i czym jest „maszyna stosowa"? (pokaż `javap -c`)
- [ ] Dlaczego od Javy 11 nie ma osobnego JRE i co to zmienia w deploymencie?
- [ ] Czym jest HotSpot, a czym GraalVM/OpenJ9?

### 🎤 Pytania rekrutacyjne
- [ ] „Czym różni się JDK od JRE i JVM?"
- [ ] „Dlaczego Java jest przenośna?" (bytecode + JVM, nie magia)
- [ ] „Czemu pierwsze żądania do serwisu są wolniejsze?" (warm-up/JIT/lazy class loading)
- [ ] „Native image vs JVM — kiedy co?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/00-fundament.csv`.*
```
Co wykonuje JVM?;Bytecode z plików .class (nie kod źródłowy, nie kod maszynowy bezpośrednio).
JDK = ?;JRE + narzędzia deweloperskie (javac, jar, jlink, jdeps, jshell...).
JRE = ?;JVM + biblioteki standardowe (środowisko uruchomieniowe).
Dlaczego Java jest przenośna?;Kompiluje się do bytecode niezależnego od OS/CPU; konkretna JVM tłumaczy go na kod maszynowy.
Co robi JIT?;Kompiluje "gorące" metody z bytecode do natywnego kodu maszynowego w trakcie działania (C1→C2).
Czym jest warm-up JVM?;Początkowy okres interpretacji zanim JIT zoptymalizuje gorący kod — stąd wolniejsze pierwsze wywołania.
Od której wersji Oracle nie wydaje osobnego JRE?;Od Javy 11 — dystrybuuje się JDK lub własny runtime przez jlink.
HotSpot to?;Najpopularniejsza implementacja JVM (od Oracle/OpenJDK).
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/00-fundament/wersje-javy-lts]] · [[wiedza/02-jvm/jit]] · [[wiedza/10-cloud-native/konteneryzacja]]
