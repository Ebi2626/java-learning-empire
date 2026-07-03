---
temat: "CI/CD — Continuous Integration / Delivery / Deployment"
faza: 10
status: nieopanowany
priorytet: 🔴
tags: [java, cloud, cicd, devops]
powiazane: ["[[wiedza/00-fundament/maven]]", "[[wiedza/10-cloud-native/konteneryzacja]]", "[[wiedza/10-cloud-native/kubernetes]]", "[[wiedza/04-narzedzia/testcontainers]]"]
---

# CI/CD — Continuous Integration / Delivery / Deployment

> **TL;DR:** **CI** = częste scalanie do głównej gałęzi z automatycznym **buildem i testami** (szybkie wykrycie
> konfliktów/regresji). **Continuous Delivery** = każda zmiana jest *zdatna do wdrożenia*, ale release odpala człowiek
> (za przyciskiem). **Continuous Deployment** = po zielonym pipeline zmiana **automatycznie** ląduje na produkcji.
> Sam pipeline to **kod** (YAML w repo), buduje artefakt **raz** i wdraża go **wiele razy** (*build once, deploy many*).

## 1. Co — definicje i pojęcia

Trzy różne rzeczy pod jednym skrótem — różnią się **granicą automatyzacji**:

```
        commit / merge do main
              │
   ┌──────────▼───────────┐
   │  Continuous           │  build + testy przy KAŻDYM merge.
   │  Integration (CI)     │  Cel: nie łamać main, szybko wykryć regresję/konflikt.
   └──────────┬───────────┘
              │  artefakt "release-ready"
   ┌──────────▼───────────┐
   │  Continuous           │  zmiana ZAWSZE zdatna do wdrożenia,
   │  Delivery             │  ale deploy na prod = decyzja człowieka (manual approval / przycisk).
   └──────────┬───────────┘
              │  brak bramki manualnej
   ┌──────────▼───────────┐
   │  Continuous           │  zielony pipeline ⇒ AUTOMATYCZNY deploy na produkcję.
   │  Deployment           │  Wymaga dojrzałych testów, monitoringu i rollbacku.
   └──────────────────────┘
```

- **Continuous Integration** — programiści **często** (kilka razy dziennie) scalają pracę do wspólnej gałęzi
  (`main`/`trunk`), a serwer CI po każdym pushu robi **build + testy**. Dzięki temu konflikty i regresje wychodzą
  po minutach, a nie po tygodniach *merge hell*. Sercem CI jest **zielony main** — złamany build blokuje zespół.
- **Continuous Delivery** — rozszerzenie CI: każdy artefakt, który przeszedł pipeline, jest **gotowy do wdrożenia**
  na produkcję. Wdrożenie jednak wymaga **ręcznej akceptacji** (bramka: „Deploy to prod" po review/QA/oknie
  serwisowym). Ryzyko biznesowe zostaje po stronie człowieka, ryzyko techniczne — zautomatyzowane.
- **Continuous Deployment** — pełna automatyzacja: po przejściu pipeline zmiana **sama** trafia na produkcję, bez
  klikania. Wymaga wysokiej dojrzałości: solidnych testów, feature flags, monitoringu, szybkiego rollbacku.

**Kluczowe rozróżnienie rekrutacyjne:** *Delivery* = „możemy wdrożyć w każdej chwili" (bramka manualna),
*Deployment* = „wdrażamy automatycznie po każdym zielonym buildzie" (brak bramki). CI to warunek wstępny obu.

## 2. Jak — etapy pipeline i strategie wdrożeń pod spodem

Typowy **pipeline dla Spring Boot 3 / Java 21** (każdy etap to bramka — *fail-fast*, im wcześniej pęknie, tym taniej):

```
checkout → build (Maven) → unit tests → integration tests (Testcontainers)
   → jakość/pokrycie (SonarQube + JaCoCo) → skan zależności (OWASP/Snyk)
   → build obrazu (Docker) → push do registry → migracje DB (Flyway) → deploy (k8s)
```

1. **Checkout** — pobranie kodu z Git (`actions/checkout`), ustalenie wersji (tag/commit SHA).
2. **Build** — kompilacja i pakowanie JAR-a przez **Maven** ([[wiedza/00-fundament/maven]]): `mvn -B package`.
   Tu włącza się **cache** zależności (`~/.m2`), żeby nie ściągać świata przy każdym runie.
3. **Testy jednostkowe** — szybkie, bez I/O (`mvn test`). Pierwsza szybka bramka regresji.
4. **Testy integracyjne** — z realnymi zależnościami przez **Testcontainers**
   ([[wiedza/04-narzedzia/testcontainers]]): prawdziwy Postgres/Kafka/Redis w kontenerze na czas testu.
   Wymaga dostępnego Docker daemona na runnerze.
5. **Analiza jakości / pokrycia** — **JaCoCo** liczy code coverage, **SonarQube** robi static analysis
   (bugs, code smells, security hotspots) i pilnuje *quality gate* (np. pokrycie nowego kodu ≥ 80%).
6. **Skan bezpieczeństwa zależności** — **OWASP Dependency-Check** / **Snyk** szukają znanych podatności (CVE)
   w bibliotekach ([[wiedza/08-bezpieczenstwo/owasp]]). *Software Composition Analysis (SCA)*.
7. **Build obrazu kontenera** — z JAR-a powstaje obraz ([[wiedza/10-cloud-native/konteneryzacja]]):
   `Dockerfile`, Jib albo `spring-boot:build-image` (Buildpacks). Tag = wersja/commit SHA.
8. **Push do rejestru** — obraz ląduje w **container registry** (GHCR, ECR, Docker Hub, Harbor).
   Ten sam, niezmienny (*immutable*) obraz pojedzie na każde środowisko — zasada **build once, deploy many**.
9. **Migracje DB** — **Flyway** ([[wiedza/06-persystencja/flyway]]) wykonuje wersjonowane skrypty schematu
   przed/podczas startu aplikacji. Migracje muszą być **wstecznie kompatybilne** (patrz strategie niżej).
10. **Deploy** — aplikacja rusza na **Kubernetes** ([[wiedza/10-cloud-native/kubernetes]]) — `kubectl apply`,
    Helm albo GitOps (niżej).

### Artefakty i rejestry
- **Maven repository** (Nexus/Artifactory/GitHub Packages) — przechowuje JAR-y/biblioteki (wersjonowane, `SNAPSHOT`
  vs release). Cache warstwy `~/.m2` w CI przyspiesza build.
- **Container registry** — przechowuje obrazy kontenerów. Obraz identyfikujemy tagiem (najlepiej **immutable**, np.
  SHA lub semver), nie ruchomym `latest`, żeby deploy był powtarzalny i audytowalny.

### Narzędzia CI/CD (świadomość)
- **GitHub Actions** — pipeline jako **workflow YAML** w `.github/workflows/`. Hierarchia:
  **workflow** → **jobs** (równoległe lub zależne) → **steps** (kolejne komendy/akcje). Job biegnie na **runnerze**
  (VM/kontener, hosted lub self-hosted). `actions/setup-java` + `actions/cache` (lub `cache: maven`) cache'ują `~/.m2`.
- **GitLab CI** — `.gitlab-ci.yml`, koncepcja *stages/jobs*, wbudowany registry i runners.
- **Jenkins** — weteran, `Jenkinsfile` (*Declarative/Scripted Pipeline*), ogromny ekosystem pluginów; self-hosted,
  wymaga utrzymania. Model *master + agents*.

### Strategie wdrożeń (deployment strategies) — jak ograniczać ryzyko releasu

| Strategia | Jak działa | Zysk | Koszt / ryzyko |
|---|---|---|---|
| **Rolling update** | pody wymieniane partiami: nowe wstają, stare gasną stopniowo | brak downtime, natywne w k8s | przez chwilę **dwie wersje naraz** (kompatybilność API/DB!) |
| **Blue-green** | dwa pełne środowiska: „blue" (stare) i „green" (nowe); przełączasz cały ruch naraz | natychmiastowy rollback (przełącz z powrotem), brak mieszania wersji | **2× zasoby**, migracje DB muszą pasować do obu |
| **Canary** | nowa wersja dostaje najpierw np. 5% ruchu; obserwujesz metryki; zwiększasz stopniowo | wczesne wykrycie problemu na małej próbce | potrzeba dobrego monitoringu i routingu (service mesh/ingress) |

Wspólny mianownik: nigdy nie wymieniamy 100% od razu bez siatki bezpieczeństwa. Rolling to domyślny mechanizm k8s;
blue-green daje najczystszy rollback; canary — najostrożniejszy, ale najbardziej wymagający operacyjnie.

### GitOps (świadomość)
Pożądany stan klastra jest **deklaratywnie** opisany w Git (manifesty k8s/Helm). Kontroler w klastrze — **ArgoCD**
lub **Flux** — **synchronizuje** rzeczywisty stan z tym, co w repo (*pull-based*). Zalety: Git = jedno źródło prawdy
i audytu, rollback = `git revert`, brak `kubectl` z laptopa. Pipeline nie „pcha" na klaster — commituje pożądany stan.

### Zarządzanie sekretami w pipeline
Sekrety (hasła, tokeny, klucze) **nigdy w repo** (nawet w prywatnym — historia Git jest wieczna). Zamiast tego:
**GitHub Secrets** / **HashiCorp Vault** / *cloud secret manager*, wstrzykiwane jako zmienne środowiskowe w runtime
pipeline. W k8s: `Secret` (najlepiej szyfrowany, np. Sealed Secrets / External Secrets). Uwaga na **wyciek do logów**
(patrz pułapki).

## 3. Dlaczego / kiedy — praktyki i pułapki

**Dobre praktyki:**
- **Szybki feedback + fail-fast** — najtańsze/najszybsze testy pierwsze; pipeline pęka jak najwcześniej.
- **Pipeline jako kod** — definicja w repo, wersjonowana, review'owana; koniec „klikania w UI Jenkinsa".
- **Powtarzalność (reproducibility)** — ten sam commit → ten sam wynik; przypięte wersje, cache, brak stanu z runnera.
- **Build once, deploy many** — jeden immutable artefakt/obraz przez dev → staging → prod; nie przebudowujemy per env.
- **Trunk-based development** — krótkożyjące gałęzie, częste merge do `main`, feature flags zamiast długich branchy.
- **Wersjonowanie semantyczne (SemVer)** — `MAJOR.MINOR.PATCH`; sygnalizuje zakres zmiany (breaking vs feature vs fix).

**Pułapki:**
- **Flaky tests** — testy losowo raz zielone, raz czerwone (zależność od czasu, kolejności, sieci). Podkopują
  zaufanie („zrób retry"), maskują realne regresje. Karać, izolować, naprawiać — nie ignorować.
- **Wolny pipeline** — 40-minutowy build zabija częstość merge'y. Cache, równoległość, dzielenie testów, szybkie
  bramki na początku.
- **Brak rollbacku** — deploy bez planu wycofania to hazard; miej wersjonowane obrazy i szybką ścieżkę powrotu
  (blue-green/GitOps `revert`).
- **Sekrety w logach** — `echo`/debug wypisujące env, albo error zawierający token. Maskowanie (GitHub maskuje
  wartości Secrets), zakaz logowania wrażliwych pól, skan repo (git-secrets/gitleaks).

**Kiedy Deployment, a kiedy tylko Delivery:** pełne Continuous Deployment ma sens przy dojrzałych testach, dobrym
monitoringu i tolerancji biznesu na częste zmiany. Systemy regulowane / krytyczne często zostają przy **Delivery**
z bramką manualną (zgodność, okna serwisowe, audyt).

## Przykład w praktyce

Serwis Spring Boot 3 (Java 21) na k8s. Workflow GitHub Actions: build → test → obraz → push. Cache Maven skraca
build z 6 do ~2 min; obraz taguje się SHA commita; deploy dokłada osobny job z GitOps (ArgoCD wykrywa nowy tag).

```yaml
name: ci-cd

on:
  push:
    branches: [ main ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write            # push do GHCR
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven            # cache ~/.m2 zależności

      - name: Build + testy (unit + integration/Testcontainers)
        run: mvn -B verify        # verify = package + testy integracyjne + JaCoCo

      - name: Skan zależności (OWASP)
        run: mvn -B org.owasp:dependency-check-maven:check

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}   # sekret, NIE w repo

      - name: Build + push obrazu (Buildpacks, tag = SHA)
        run: |
          mvn -B spring-boot:build-image \
            -Dspring-boot.build-image.imageName=ghcr.io/${{ github.repository }}:${{ github.sha }}
          docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
```

Deploy dopina się osobno: albo job `kubectl set image` (rolling update), albo — GitOps — commit nowego tagu do repo
manifestów, który ArgoCD sam zsynchronizuje z klastrem.

---

## ✅ Kryteria opanowania
- [ ] Rozróżnię CI / Continuous Delivery / Continuous Deployment (gdzie leży bramka manualna).
- [ ] Wymienię etapy pipeline dla Spring Boot i wiem, co robi każdy.
- [ ] Wyjaśnię rolling / blue-green / canary i ich kompromisy.
- [ ] Rozumiem „build once, deploy many" i po co immutable obrazy.
- [ ] Wiem, jak (nie) zarządzać sekretami w pipeline.
- [ ] Znam GitOps i różnicę push vs pull deploy.

### 🔲 Black-box check
- [ ] Czym różni się Continuous Delivery od Continuous Deployment? (bramka manualna vs jej brak)
- [ ] Co dokładnie cache'uje `cache: maven` i po co? (`~/.m2`, przyspieszenie buildu)
- [ ] Dlaczego rolling update wymaga kompatybilności między wersjami? (dwie wersje działają naraz)
- [ ] Jak GitOps synchronizuje klaster i czemu to pull-based? (ArgoCD/Flux ciągną stan z Git)
- [ ] Gdzie w pipeline wykrywasz podatne zależności? (OWASP Dependency-Check / Snyk — SCA)

### 🎤 Pytania rekrutacyjne
- [ ] „Czym różni się CI od CD?" (i które CD?)
- [ ] „Opisz pipeline od commita do produkcji dla Spring Boot."
- [ ] „Blue-green vs canary — kiedy co?"
- [ ] „Jak trzymasz sekrety w CI, żeby nie wyciekły?"
- [ ] „Co to flaky test i czemu jest groźny?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/10-cloud-native.csv`.*
```
Co to Continuous Integration?;Częste scalanie do wspólnej gałęzi z automatycznym buildem i testami — szybkie wykrycie konfliktów i regresji.
Continuous Delivery vs Continuous Deployment?;Delivery: każda zmiana zdatna do wdrożenia, ale deploy odpala człowiek (bramka). Deployment: deploy automatyczny po zielonym pipeline.
Warunek wstępny Continuous Delivery/Deployment?;Działające CI (automatyczny build + testy przy każdym merge).
Zasada "build once, deploy many"?;Buduj artefakt/obraz raz, ten sam immutable obraz wdrażaj na dev/staging/prod — powtarzalność i audytowalność.
Typowe etapy pipeline Spring Boot?;checkout → build (Maven) → unit → integration (Testcontainers) → jakość (SonarQube/JaCoCo) → skan CVE (OWASP/Snyk) → obraz → push → migracje (Flyway) → deploy (k8s).
Co robi JaCoCo, a co SonarQube?;JaCoCo mierzy code coverage; SonarQube robi static analysis (bugs, smells, security) i pilnuje quality gate.
Co wykrywa OWASP Dependency-Check / Snyk?;Znane podatności (CVE) w zależnościach — Software Composition Analysis (SCA).
Hierarchia GitHub Actions?;workflow → jobs (równoległe/zależne) → steps; job biegnie na runnerze (hosted/self-hosted).
Po co cache ~/.m2 w CI?;Żeby nie pobierać zależności Maven przy każdym runie — przyspiesza build.
Rolling update — jak i jakie ryzyko?;Pody wymieniane partiami, bez downtime; ryzyko: przez chwilę działają dwie wersje naraz (kompatybilność API/DB).
Blue-green deployment?;Dwa pełne środowiska (stare/nowe), ruch przełączany naraz; natychmiastowy rollback, ale 2× zasoby.
Canary deployment?;Nowa wersja dostaje najpierw mały % ruchu; przy dobrych metrykach zwiększasz — wczesne wykrycie problemu.
Co to GitOps?;Pożądany stan klastra deklaratywnie w Git; ArgoCD/Flux (pull) synchronizują klaster z repo; rollback = git revert.
Gdzie trzymać sekrety w pipeline?;Nigdy w repo — GitHub Secrets / Vault / secret manager, wstrzykiwane jako env w runtime.
Co to flaky test?;Test raz zielony, raz czerwony bez zmiany kodu (czas/kolejność/sieć); podkopuje zaufanie i maskuje regresje.
Co znaczy fail-fast w pipeline?;Najszybsze/najtańsze bramki pierwsze; pipeline pęka jak najwcześniej, żeby oszczędzić czas i zasoby.
Trunk-based development?;Krótkożyjące gałęzie, częsty merge do main, feature flags zamiast długich branchy.
SemVer — co oznaczają segmenty?;MAJOR.MINOR.PATCH: MAJOR = breaking change, MINOR = nowa funkcja wstecznie kompatybilna, PATCH = poprawka.
Czemu tagować obraz SHA/semver, a nie latest?;Immutable, powtarzalny i audytowalny deploy; latest jest ruchomy i psuje reproducibility.
Czym są jobs vs steps w CI?;Jobs to jednostki wykonywane (równolegle/zależnie) na runnerach; steps to kolejne komendy/akcje w obrębie joba.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/00-fundament/maven]] · [[wiedza/04-narzedzia/testcontainers]] · [[wiedza/08-bezpieczenstwo/owasp]] · [[wiedza/06-persystencja/flyway]] · [[wiedza/10-cloud-native/konteneryzacja]] · [[wiedza/10-cloud-native/kubernetes]]
