---
temat: "Konteneryzacja aplikacji Java (Docker / Podman)"
faza: 10
status: nieopanowany
priorytet: 🔴
tags: [java, cloud, docker, konteneryzacja]
powiazane: ["[[wiedza/00-fundament/jdk-jre-jvm]]", "[[wiedza/10-cloud-native/kubernetes]]"]
---

# Konteneryzacja aplikacji Java (Docker / Podman)

> **TL;DR:** **Kontener** to izolowany *proces* Linuxa (namespaces + cgroups), a **NIE** maszyna wirtualna —
> współdzieli jądro hosta, więc jest lekki i startuje w ms. Aplikację pakujesz w **obraz OCI** (warstwy,
> read-only + cienka warstwa zapisu) opisany **Dockerfile**. Dla Javy klucz to **multi-stage build**
> (JDK/Maven → sam JRE), **layered jars** Spring Boota (lepszy cache) i **JVM świadoma cgroup**
> (`-XX:MaxRAMPercentage`, nie sztywne `-Xmx`). **Podman** = daemonless + rootless zamiennik `docker`.

## 1. Co — definicja i API

**Kontener** to nie „mała maszyna wirtualna". To **zwykły proces** na jądrze hosta, któremu odebrano widok
na resztę systemu za pomocą dwóch mechanizmów jądra Linuxa:

- **namespaces** — *izolacja widoczności*: własny PID (proces widzi siebie jako PID 1), własny mount (własny
  filesystem), network (własny stack sieciowy/interfejsy), UTS (hostname), IPC, user (mapowanie UID). Proces
  „nie wie", że obok są inne kontenery.
- **cgroups** (control groups) — *limity zasobów*: ile pamięci, ile CPU, ile I/O dany proces może zużyć.
  To cgroup zabija proces (**OOMKilled**), gdy przekroczy limit pamięci.

```
┌─────────────────── VM (hypervisor) ───────────────────┐   ┌──── Kontenery ────┐
│ App │ App                                             │   │ App │ App │ App   │
│ Bins/Libs │ Bins/Libs                                 │   │ Bins/Libs (obraz) │
│ Guest OS  │ Guest OS   ← PEŁNE jądro gościa dla każdej │   ├───────────────────┤
├───────────┴───────────┤                               │   │ Container runtime │
│      Hypervisor        │  ← ciężki, boot w sekundach   │   ├───────────────────┤
├────────────────────────┤                              │   │ WSPÓLNE jądro hosta│ ← lekkie, start w ms
│      Host OS / HW       │                              │   │   Host OS / HW     │
└────────────────────────┘                              │   └───────────────────┘
```

**Kontener vs VM:** VM wirtualizuje *sprzęt* i ma **własne pełne jądro** gościa (izolacja mocniejsza, ale
narzut GB RAM i boot w sekundach). Kontener wirtualizuje *system operacyjny* i **współdzieli jądro** hosta
(narzut MB, start w ms, ale słabsza granica bezpieczeństwa — exploit jądra = ucieczka).

**Obraz OCI** (Open Container Initiative — standard formatu) to niemutowalny szablon zbudowany z **warstw**:
- każda instrukcja Dockerfile (`RUN`, `COPY`, `ADD`) tworzy nową **warstwę read-only** (diff filesystemu),
- warstwy są **współdzielone i cache'owane** — jeśli warstwa się nie zmieniła, build ją pomija, a rejestr
  nie wysyła jej ponownie,
- w czasie uruchomienia na wierzch dokładana jest cienka **warstwa zapisu** (copy-on-write); po zatrzymaniu
  kontenera jej zawartość ginie (dlatego stan trzymasz w volume/bazie, nie w kontenerze).

**Dockerfile** — deklaratywny przepis na obraz. Najważniejsze instrukcje:

| Instrukcja | Znaczenie |
|---|---|
| `FROM` | obraz bazowy (start warstw), np. `eclipse-temurin:21-jre` |
| `WORKDIR` | katalog roboczy dla kolejnych komend |
| `COPY` / `ADD` | kopiuje pliki z kontekstu build do obrazu (`ADD` umie też URL/tar — używaj `COPY`) |
| `RUN` | wykonuje komendę **w czasie build** (tworzy warstwę), np. `mvn package` |
| `ENV` | zmienna środowiskowa (build i runtime) |
| `EXPOSE` | *dokumentuje* port (nie publikuje go — to robi `-p` przy `run`) |
| `ENTRYPOINT` | stały „exe" kontenera, np. `["java","-jar","app.jar"]` |
| `CMD` | domyślne *argumenty* (nadpisywalne z CLI); z `ENTRYPOINT` łączą się w komendę |

**Docker vs Podman.** CLI jest kompatybilne (`podman` ≈ `docker`; często alias `docker=podman`), ale:
- **daemonless** — Docker ma jeden uprzywilejowany demon (`dockerd`) jako root, single point of failure;
  Podman uruchamia kontenery jako zwykłe **procesy potomne** (fork/exec), bez demona.
- **rootless** — Podman domyślnie działa bez roota (user namespaces), więc kompromitacja kontenera nie daje
  roota na hoście. Domyślne, bezpieczniejsze na Fedorze. W tej notatce komendy piszę jako `podman`.

## 2. Jak — warstwy, multi-stage, JVM w cgroup pod spodem

### Multi-stage build — mały i bezpieczny obraz
Naiwny obraz z JDK + Maven + cache `.m2` waży ~700 MB i zawiera kompilator + narzędzia (zbędne i groźne
w runtime). **Multi-stage build** dzieli build na etapy; do finalnego obrazu **kopiujesz tylko artefakt**:

```dockerfile
# --- etap 1: BUILD (ciężki, z JDK + Maven) ---
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn -q dependency:go-offline          # warstwa cache zależności (nie zmienia się przy edycji kodu)
COPY src ./src
RUN mvn -q clean package -DskipTests

# --- etap 2: RUNTIME (lekki, sam JRE) ---
FROM eclipse-temurin:21-jre AS runtime
WORKDIR /app
COPY --from=build /app/target/app.jar app.jar   # tylko artefakt, zero narzędzi build
ENTRYPOINT ["java","-jar","app.jar"]
```

Finalny obraz nie ma Mavena ani `javac` → mniejszy (mniejszy attack surface, szybszy pull).

### Layered jars — cache dopasowany do Javy
Zwykły „fat jar" Spring Boota to **jedna warstwa** — zmiana jednej linii kodu unieważnia cache całego JARa
(dziesiątki MB zależności). Spring Boot buduje **layered jar** rozbity na warstwy wg zmienności:
`dependencies` (rzadko), `spring-boot-loader`, `snapshot-dependencies`, `application` (twój kod, często).
Kopiując je osobno (`COPY --from=build /app/dependencies/ ./`, potem `application/`) sprawiasz, że rebuild po
zmianie kodu wysyła tylko warstwę `application` — reszta z cache. Rozpakowanie: `java -Djarmode=layertools -jar app.jar extract`.

### Budowanie obrazu BEZ Dockerfile
- **`spring-boot:build-image`** (Cloud Native Buildpacks / Paketo) — `mvn spring-boot:build-image`; buildpack
  wykrywa Javę, dobiera JRE, ustawia sensowne flagi pamięci, tworzy warstwowany obraz. Bez Dockerfile.
- **Jib** (Google) — plugin Mavena/Gradle budujący **zoptymalizowany, warstwowany obraz bez demona i bez
  Dockerfile**; rozdziela zależności/zasoby/klasy na warstwy, jest reproducible. `mvn com.google.cloud.tools:jib-maven-plugin:build`.

### Obrazy bazowe
- **`eclipse-temurin`** — standardowy, dobrze utrzymany OpenJDK (`-jdk`, `-jre`); rozsądny default.
- **distroless** (`gcr.io/distroless/java21`) — bez shella, bez menedżera pakietów, prawie same biblioteki
  runtime → najmniejszy attack surface. Debugowanie trudniejsze (brak `sh`).
- **alpine** — bardzo mały, ale oparty na **musl libc** zamiast glibc → **pułapka**: HotSpot bywał tu mniej
  przetestowany, zdarzają się subtelne różnice (DNS, locale, wydajność). Preferuj distroless lub Temurin nad alpine dla Javy.

### JVM w cgroup — najważniejszy kawałek (patrz [[wiedza/00-fundament/jdk-jre-jvm]])
Kluczowe pytanie: **czy JVM widzi limity kontenera, czy limity całego hosta?**

- **JVM jest container-aware** od **JDK 10+** (backport do **8u191**): czyta limity **cgroup** (pamięć i CPU),
  a nie parametry fizycznej maszyny. Starsze JVM widziały RAM/CPU *hosta* → ustawiały ogromny heap → **OOMKilled**.
- **Pamięć:** zamiast sztywnego `-Xmx512m` używaj **`-XX:MaxRAMPercentage=75.0`** — heap skaluje się do
  limitu cgroup, więc ten sam obraz działa przy różnych limitach. Pamiętaj o **narzucie poza heap**
  (metaspace, stosy wątków, kod JIT, bufory) — limit kontenera musi być > heap, inaczej cgroup zabija proces.
- **CPU:** liczba dostępnych CPU (z cgroup) steruje **domyślną liczbą wątków GC** i rozmiarem
  **`ForkJoinPool.commonPool`** (`Runtime.availableProcessors()`). Za mały limit CPU → mało wątków GC/równoległości;
  ułamkowe limity (np. 0.5 CPU) bywają zaokrąglane — warto weryfikować `activeProcessorCount`.

## 3. Dlaczego / kiedy — pamięć, CPU, dobre praktyki

**Kiedy konteneryzować:** praktycznie zawsze przy deploymencie na chmurę/[[wiedza/10-cloud-native/kubernetes]] —
powtarzalne środowisko, izolacja, skalowanie. Kompromis: warstwa abstrakcji, świadomość limitów, obserwowalność.

**Dobre praktyki:**
- **Nie root w kontenerze** — utwórz usera (`USER 1000`); przy rootless Podman i tak bezpieczniej, ale
  zdefiniuj to jawnie (obowiązuje też na k8s z `runAsNonRoot`).
- **`.dockerignore`** — nie pchaj `target/`, `.git`, sekretów do kontekstu build (mniejszy kontekst, brak wycieków).
- **Tagowanie** — nie polegaj na `latest` w produkcji; taguj wersją/commit SHA (powtarzalność, rollback).
- **Mały obraz** — multi-stage + JRE/distroless: szybszy pull, mniejszy attack surface, niższy koszt.
- **Healthcheck** — endpoint (`/actuator/health`) dla liveness/readiness (k8s podejmuje decyzje o restarcie).
- **Graceful shutdown** — reaguj na **SIGTERM**: dokończ żądania, zamknij pule/połączenia. Spring Boot 3 ma
  `server.shutdown=graceful`. Uwaga: proces musi być **PID 1** obsługującym sygnały (`exec` form ENTRYPOINT!).

**GraalVM native image w kontenerze** (patrz [[wiedza/00-fundament/jdk-jre-jvm]]): AOT-kompilacja aplikacji do
natywnego pliku wykonywalnego → **start w ms i mały RAM** (brak warm-up JIT, brak metaspace). Idealne do
serverless/scale-to-zero i drobnych obrazów (można wręcz `scratch`/distroless-static). Koszt: dłuższy build,
utrata dynamiki (reflection/proxy wymagają metadanych; wspiera to Spring Boot 3 / `native-image`), niższy
peak throughput niż rozgrzany JIT dla długo żyjących serwisów.

## Przykład w praktyce

Multi-stage Dockerfile z **layered jar** Spring Boota 3 (Java 21), non-root, container-aware JVM:

```dockerfile
# ---------- BUILD ----------
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn -q -B dependency:go-offline
COPY src ./src
RUN mvn -q -B clean package -DskipTests
RUN java -Djarmode=layertools -jar target/*.jar extract --destination extracted

# ---------- RUNTIME ----------
FROM eclipse-temurin:21-jre AS runtime
WORKDIR /app
# osobne warstwy = lepszy cache (deps rzadko, application często)
COPY --from=build /app/extracted/dependencies/ ./
COPY --from=build /app/extracted/spring-boot-loader/ ./
COPY --from=build /app/extracted/snapshot-dependencies/ ./
COPY --from=build /app/extracted/application/ ./
# nie root
RUN useradd -r -u 1001 appuser
USER appuser
EXPOSE 8080
# heap skaluje się do limitu cgroup, nie do RAM hosta
ENTRYPOINT ["java","-XX:MaxRAMPercentage=75.0","org.springframework.boot.loader.launch.JarLauncher"]
```

Komendy (Podman na Fedorze — `docker` działa tak samo):

```bash
podman build -t moja-apka:1.0 .

# limit pamięci i CPU → JVM to CZYTA z cgroup
podman run --rm -p 8080:8080 \
  --memory=512m --cpus=2 \
  moja-apka:1.0

# weryfikacja, co JVM widzi
podman run --rm --memory=512m --cpus=2 moja-apka:1.0 \
  java -XX:+PrintFlagsFinal -version | grep -Ei "MaxHeapSize|ActiveProcessorCount"
```

Alternatywa bez Dockerfile: `mvn spring-boot:build-image` (Paketo buildpack) albo Jib.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, że kontener to proces z namespaces + cgroups, a NIE maszyna wirtualna — i czym różni się od VM.
- [ ] Rozumiem warstwy obrazu OCI (read-only + warstwa zapisu) i jak działa cache warstw.
- [ ] Napiszę z głowy multi-stage Dockerfile dla Spring Boota (build z JDK → runtime z JRE).
- [ ] Wiem, po co layered jars i czym różni się `spring-boot:build-image` od Jiba.
- [ ] Rozumiem, że JVM jest container-aware i czemu używamy `MaxRAMPercentage`, nie sztywnego `-Xmx`.
- [ ] Wiem, dlaczego dochodzi do OOMKilled i jak CPU limit wpływa na wątki GC/ForkJoinPool.

### 🔲 Black-box check
- [ ] Które mechanizmy jądra dają izolację, a które limity? (namespaces vs cgroups)
- [ ] Czym RÓŻNI się kontener od VM na poziomie jądra? (współdzielone jądro vs własne jądro gościa)
- [ ] Co dokładnie tworzy nową warstwę obrazu i dlaczego kolejność `COPY`/`RUN` wpływa na cache?
- [ ] Czym różni się `ENTRYPOINT` od `CMD` i co robi forma exec vs shell przy sygnałach?
- [ ] Od której wersji JVM czyta limity cgroup i co robiła wcześniej?
- [ ] Czemu limit kontenera musi być większy niż heap? (metaspace, stosy, JIT, bufery poza heap)
- [ ] Podman vs Docker — co znaczy daemonless i rootless?

### 🎤 Pytania rekrutacyjne
- [ ] „Kontener to maszyna wirtualna?" (nie — proces z namespaces + cgroups, wspólne jądro)
- [ ] „Jak zbudować mały, bezpieczny obraz dla aplikacji Java?" (multi-stage, JRE/distroless, non-root)
- [ ] „Aplikacja jest OOMKilled mimo `-Xmx` — dlaczego?" (narzut poza heap > limit cgroup / stary JVM)
- [ ] „Po co layered jars i czym jest Jib?"
- [ ] „Czemu ustawiamy `MaxRAMPercentage` zamiast `-Xmx` w kontenerze?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/10-cloud-native.csv`.*

```
Czym jest kontener (technicznie)?;Izolowanym procesem Linuxa — izolacja przez namespaces, limity zasobów przez cgroups; NIE maszyną wirtualną.
Kontener vs VM — kluczowa różnica?;VM ma własne pełne jądro gościa (ciężki, boot w sekundach); kontener współdzieli jądro hosta (lekki, start w ms).
Co dają namespaces?;Izolację widoczności: własny PID, mount, network, UTS, IPC, user — proces nie widzi reszty systemu.
Co dają cgroups?;Limity zasobów (pamięć, CPU, I/O); to cgroup zabija proces jako OOMKilled po przekroczeniu pamięci.
Co to obraz OCI?;Niemutowalny szablon zbudowany z warstw read-only wg standardu Open Container Initiative.
Jak zbudowane są warstwy obrazu?;Każda instrukcja RUN/COPY/ADD tworzy warstwę read-only; na wierzchu cienka warstwa zapisu (copy-on-write), ginąca po stopie.
Po co cache warstw?;Niezmienione warstwy są pomijane w build i nie wysyłane do rejestru — szybszy build i pull.
Różnica ENTRYPOINT vs CMD?;ENTRYPOINT to stały "exe" kontenera; CMD to domyślne, nadpisywalne argumenty; łączą się w pełną komendę.
Co robi EXPOSE?;Tylko dokumentuje port; faktyczne opublikowanie robi -p przy podman/docker run.
Docker vs Podman?;Podman jest daemonless (bez uprzywilejowanego demona) i rootless (user namespaces) przy kompatybilnym CLI.
Co to multi-stage build?;Podział Dockerfile na etapy: ciężki build z JDK/Maven, do finalnego obrazu kopiowany tylko artefakt na lekki JRE.
Po co layered jars w Spring Boot?;Rozbijają JAR na warstwy wg zmienności (dependencies vs application) → zmiana kodu unieważnia tylko warstwę application, reszta z cache.
Czym jest Jib?;Plugin Google (Maven/Gradle) budujący zoptymalizowany, warstwowany obraz BEZ Dockerfile i bez demona.
Co robi spring-boot:build-image?;Buduje obraz przez Cloud Native Buildpacks (Paketo) bez Dockerfile, z sensownymi flagami JVM.
Distroless — po co?;Obraz bez shella i menedżera pakietów → minimalny attack surface; trudniejsze debugowanie.
Pułapka obrazu alpine dla Javy?;Alpine używa musl libc zamiast glibc — możliwe subtelne różnice/mniejsze przetestowanie HotSpota; preferuj Temurin/distroless.
Od kiedy JVM jest container-aware?;Od JDK 10+ (backport 8u191) — czyta limity cgroup (pamięć i CPU) zamiast parametrów hosta.
Czemu MaxRAMPercentage zamiast -Xmx w kontenerze?;Heap skaluje się do limitu cgroup, więc ten sam obraz działa przy różnych limitach pamięci.
Czemu app jest OOMKilled mimo ustawionego heapu?;Poza heap jest narzut (metaspace, stosy wątków, kod JIT, bufory); jeśli limit cgroup < heap + narzut, cgroup zabija proces.
Jak CPU limit wpływa na JVM?;Steruje availableProcessors → domyślną liczbą wątków GC i rozmiarem ForkJoinPool.commonPool.
Dlaczego nie uruchamiać kontenera jako root?;Kompromitacja kontenera nie eskaluje do roota na hoście; użyj USER i runAsNonRoot na k8s.
Po co exec form ENTRYPOINT i graceful shutdown?;By proces był PID 1 odbierającym SIGTERM i mógł dokończyć żądania oraz zamknąć zasoby przed zabiciem.
GraalVM native image w kontenerze — zysk i koszt?;Zysk: start w ms i mały RAM (brak warm-up JIT); koszt: dłuższy build, utrata dynamiki, niższy peak throughput dla długo żyjących serwisów.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/00-fundament/jdk-jre-jvm]] · [[wiedza/10-cloud-native/kubernetes]]
