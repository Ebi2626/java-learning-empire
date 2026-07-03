# ROADMAP — Java dla fullstacka (Angular + Java/Spring)

Praktyczna, fazowana mapa drogowa. Każda faza ma: **cel**, **tematy**, **narzędzia/frameworki**,
**wzorce**, **kamień milowy** (co umieć/zbudować) i **link do notatek** w `wiedza/`.

Kolejność jest celowa: każda faza buduje na poprzedniej. Fazy 0–5 to fundament zatrudnialności;
6–10 to to, czego realnie używa się w pracy fullstacka; 11–12 dopinają projekt docelowy.

> **Legenda priorytetu:** 🔴 krytyczne na rozmowie/w pracy · 🟡 ważne · 🟢 wyróżnik/zaawansowane

---

## Faza 0 — Fundament i środowisko 🔴
**Cel:** sprawne środowisko + poprawny model mentalny „co uruchamia mój kod".

- **Tematy:** JDK vs JRE vs JVM; wersje Javy i LTS (8/11/**17**/**21**); cykl wydawniczy (6 mies.);
  czym jest bytecode; `javac` → `.class` → JVM; classpath vs module path; kompilacja vs interpretacja vs JIT.
- **Narzędzia:** **SDKMAN!** (zarządzanie wersjami JDK), **Maven** (główny build tool w pracy), Gradle (świadomość),
  IntelliJ IDEA, `jshell`, `java`/`javac`/`jar`/`jlink`/`jdeps`.
- **Decyzja zawodowa:** masz JDK 25, ale **ucz się i celuj w 21 (LTS)**; znaj różnice 8→17→21.
- **Kamień milowy:** zainstaluj przez SDKMAN JDK 21, zbuduj „hello world" Mavenem, uruchom `jshell`.
- **Notatki:** [[wiedza/00-fundament/jdk-jre-jvm]] · [[wiedza/00-fundament/wersje-javy-lts]] · [[wiedza/00-fundament/maven]] · [[wiedza/00-fundament/sdkman]]

## Faza 1 — Język Java 🔴
**Cel:** płynność w nowoczesnej Javie (nie Javie z 2010).

- **Podstawy:** typy prymitywne vs referencyjne, autoboxing, `String` (immutability, pool, `StringBuilder`),
  `equals`/`hashCode`/`toString`, kontrakt equals-hashCode, `Comparable`/`Comparator`.
- **OOP:** klasy, interfejsy, `default`/`static`/`private` metody w interfejsach, dziedziczenie vs kompozycja,
  modyfikatory dostępu, klasy zagnieżdżone/wewnętrzne/anonimowe.
- **Nowoczesne konstrukcje:** `record`, `sealed` klasy/interfejsy, `enum` (z metodami), `var` (inferencja),
  **pattern matching** (`instanceof`, `switch`), text blocks, `Optional`.
- **Generyki:** typy, ograniczenia (`extends`/`super`), wildcards, type erasure (i jego konsekwencje).
- **Kolekcje:** `List`/`Set`/`Map`/`Queue`, implementacje (ArrayList vs LinkedList, HashMap vs TreeMap),
  złożoność operacji, kiedy co.
- **Funkcyjnie:** interfejsy funkcyjne, lambdy, referencje do metod, **Stream API** (map/filter/reduce/collect,
  leniwość, `Collectors`), `Optional` jako monada.
- **Wyjątki:** checked vs unchecked, `try-with-resources`, własne wyjątki, antywzorce.
- **Wzorce:** Builder, Factory, Strategy (przez lambdy), Iterator, Template Method.
- **Kamień milowy:** napisz mały silnik reguł/parser używając recordów, sealed types i Streamów.
- **Notatki:** katalog [[wiedza/01-jezyk]] (osobna notatka na każdy podtemat).

## Faza 2 — JVM od środka 🟡 (tu giną black-boxy)
**Cel:** rozumieć, co naprawdę robi maszyna wirtualna — pamięć, GC, JIT, classloading.

- **Pamięć:** stack vs heap, metaspace, model obiektu, referencje (strong/soft/weak/phantom).
- **Garbage Collection:** generacje, G1 (domyślny), ZGC/Shenandoah (low-latency), kiedy „stop-the-world",
  jak czytać logi GC, dostrajanie (`-Xmx`, `-Xms`).
- **JIT:** interpreter → C1 → C2, inlining, deoptymizacja, warm-up (kluczowe dla benchmarków!).
- **Classloading:** hierarchia classloaderów, delegacja, `ClassNotFoundException` vs `NoClassDefFoundError`.
- **Java Memory Model:** happens-before, `volatile`, widoczność — fundament współbieżności.
- **Narzędzia:** **JFR** (Java Flight Recorder), **JMC**, `jstack`, `jmap`, `jstat`, async-profiler, VisualVM.
- **Wzorce/wąskie gardła:** wycieki pamięci, memory pressure, false sharing.
- **Kamień milowy:** wywołaj i zdiagnozuj OutOfMemoryError oraz przeanalizuj heap dump.
- **Notatki:** [[wiedza/02-jvm/model-pamieci]] · [[wiedza/02-jvm/garbage-collection]] · [[wiedza/02-jvm/jit]] · [[wiedza/02-jvm/classloading]]

## Faza 3 — Współbieżność 🟡
**Cel:** pisać poprawny, bezpieczny współbieżnie kod (kluczowe w backendach).

- **Podstawy:** wątki, `Runnable`/`Callable`, race conditions, deadlock/livelock/starvation.
- **Synchronizacja:** `synchronized`, `volatile`, `Lock`/`ReentrantLock`, `Atomic*`, `java.util.concurrent`.
- **Pule i zadania:** `ExecutorService`, `ThreadPoolExecutor` (tuning!), `CompletableFuture` (async pipelines).
- **Kolekcje współbieżne:** `ConcurrentHashMap`, `CopyOnWriteArrayList`, `BlockingQueue`.
- **Loom 🟢:** **wątki wirtualne** (Java 21+) — game changer dla I/O-bound backendów; structured concurrency.
- **Wzorce:** Producer-Consumer, Thread Pool, Future/Promise, Immutability jako strategia.
- **Kamień milowy:** zbuduj pipeline z `CompletableFuture` i porównaj pule wątków platform vs virtual.
- **Notatki:** [[wiedza/03-wspolbieznosc/podstawy]] · [[wiedza/03-wspolbieznosc/executors]] · [[wiedza/03-wspolbieznosc/virtual-threads]]

## Faza 4 — Narzędzia, testy, jakość 🔴
**Cel:** profesjonalny warsztat — bez tego nie wejdziesz do zespołu.

- **Build:** **Maven** głęboko (lifecycle, fazy, pluginy, dependency scope, BOM, multi-module), repozytoria, `settings.xml`.
- **Testy:** **JUnit 5** (Jupiter), **Mockito**, **AssertJ**, **Testcontainers** (testy z realną bazą/Kafką w kontenerze),
  parametryzacja, test pyramid, TDD świadomie.
- **Jakość:** SLF4J + Logback (logowanie), Checkstyle/Spotless/PMD, JaCoCo (pokrycie), SonarQube.
- **Wzorce testowe:** AAA (Arrange-Act-Assert), Test Data Builder, fakes vs mocks vs stubs.
- **Kamień milowy:** projekt multi-module Maven z testami jednostkowymi i integracyjnymi (Testcontainers).
- **Notatki:** [[wiedza/04-narzedzia/maven-gleboko]] · [[wiedza/04-narzedzia/junit-mockito]] · [[wiedza/04-narzedzia/testcontainers]] · [[wiedza/04-narzedzia/logowanie]]

## Faza 5 — Spring i Spring Boot 🔴 (serce pracy)
**Cel:** budować realne REST API — to ~80% codziennej pracy backendowej fullstacka.

- **Rdzeń:** IoC/DI, kontener, beany, scope'y, `@Configuration`/`@Bean`, autowiring, cykl życia beana,
  **jak działa proxy (CGLIB/JDK)** — częsty black-box.
- **Boot:** auto-konfiguracja (jak naprawdę działa!), starters, `application.yml`, profile, `@ConditionalOn*`.
- **Web:** Spring MVC, `@RestController`, mapowanie żądań, walidacja (`jakarta.validation`),
  globalna obsługa błędów (`@ControllerAdvice`), content negotiation.
- **Konfiguracja:** `@ConfigurationProperties`, externalized config, secrets.
- **Obserwowalność:** Spring Boot Actuator, health checks, metryki.
- **Świadomość:** Spring WebFlux (reaktywne) — kiedy ma sens vs MVC.
- **Wzorce:** DI, Repository, Service Layer, DTO, Facade, dependency inversion.
- **Kamień milowy:** REST API CRUD z walidacją, profilami, obsługą błędów i Actuatorem.
- **Notatki:** [[wiedza/05-spring/ioc-di]] · [[wiedza/05-spring/proxy-aop]] · [[wiedza/05-spring/autokonfiguracja]] · [[wiedza/05-spring/mvc-rest]] · [[wiedza/05-spring/actuator]]

## Faza 6 — Persystencja i bazy danych 🔴
**Cel:** świadomie pracować z bazą — najczęstsze źródło bugów i problemów wydajności.

- **SQL:** solidny SQL (JOIN-y, indeksy, plany zapytań, transakcje, poziomy izolacji).
- **JDBC:** podstawa pod wszystkim; connection pooling (**HikariCP**).
- **JPA/Hibernate:** encje, relacje, **lazy vs eager**, **problem N+1** (klasyczny black-box!),
  cykl życia encji, persistence context, `@Transactional` (jak działa, propagacja, pułapki),
  cache pierwszego/drugiego poziomu, `fetch join`, projekcje, DTO.
- **Spring Data JPA:** repozytoria, query methods, `@Query`, paginacja, specyfikacje.
- **Migracje:** **Flyway** (lub Liquibase) — wersjonowanie schematu.
- **Bazy:** PostgreSQL (domyślny wybór), świadomość NoSQL (Redis cache, Mongo).
- **Wzorce:** Repository, Unit of Work, DAO, Optimistic/Pessimistic Locking.
- **Kamień milowy:** model z relacjami, migracje Flyway, świadome wyeliminowanie N+1, testy z Testcontainers + Postgres.
- **Notatki:** [[wiedza/06-persystencja/jpa-hibernate]] · [[wiedza/06-persystencja/transakcje]] · [[wiedza/06-persystencja/n-plus-1]] · [[wiedza/06-persystencja/flyway]]

## Faza 7 — Projektowanie API 🔴
**Cel:** API, które dobrze konsumuje się z Angulara i jest utrzymywalne.

- **REST:** dojrzałość (Richardson), zasoby, czasowniki, kody statusu, idempotentność, HATEOAS (świadomość).
- **Kontrakt:** **OpenAPI/Swagger** (springdoc), generowanie klienta TS dla Angulara, wersjonowanie API.
- **Mapowanie:** DTO vs encja (zawsze rozdzielaj!), **MapStruct**, walidacja wejścia.
- **Przekrojowe:** paginacja, filtrowanie, sortowanie, obsługa błędów (Problem Details RFC 7807), CORS (ważne dla Angulara!).
- **Świadomość:** gRPC, GraphQL — kiedy zamiast REST.
- **Wzorce:** DTO, Adapter, API Gateway, BFF (Backend-for-Frontend).
- **Kamień milowy:** API z OpenAPI, wygenerowany klient TS, poprawny CORS + Problem Details.
- **Notatki:** [[wiedza/07-api-design/rest]] · [[wiedza/07-api-design/openapi]] · [[wiedza/07-api-design/dto-mapstruct]] · [[wiedza/07-api-design/cors-bledy]]

## Faza 8 — Bezpieczeństwo 🔴
**Cel:** poprawne uwierzytelnianie/autoryzacja — krytyczne i często źle robione.

- **Podstawy:** authn vs authz, sesje vs tokeny, hashowanie haseł (BCrypt), HTTPS/TLS.
- **Spring Security:** łańcuch filtrów (jak działa!), `SecurityFilterChain`, authentication providers,
  method security (`@PreAuthorize`).
- **Tokeny:** **JWT** (budowa, walidacja, pułapki), **OAuth2 / OIDC**, Keycloak (popularny IdP).
- **OWASP:** Top 10, SQL injection, XSS, CSRF (i czemu przy JWT+SPA inaczej), secrets management.
- **Wzorce:** Token-based auth, RBAC/ABAC, Gateway-level security.
- **Kamień milowy:** zabezpiecz API JWT/OIDC (Keycloak), role + method security, przejdź checklistę OWASP.
- **Notatki:** [[wiedza/08-bezpieczenstwo/spring-security]] · [[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]] · [[wiedza/08-bezpieczenstwo/owasp]]

## Faza 9 — Architektura i wzorce aplikacyjne 🟡
**Cel:** strukturyzować większe systemy; świadomy dobór podejścia.

- **Warstwy:** warstwowa, **hexagonal/ports&adapters**, clean architecture, DDD-lite (encje, agregaty, value objects).
- **Style:** monolit (modularny!) vs mikroserwisy — **kiedy co** (większość projektów = monolit, i to OK).
- **Multitenancy 🟢:** strategie (osobna baza / osobny schemat / dyskryminator), izolacja danych — **kluczowe dla projektu**.
- **Komunikacja:** synchroniczna (REST) vs asynchroniczna (Kafka/RabbitMQ), event-driven, outbox pattern.
- **Wzorce:** GoF (przegląd), Repository, CQRS (świadomość), Saga, Circuit Breaker (Resilience4j).
- **Kamień milowy:** zaprojektuj architekturę projektu rekrutacyjnego (ADR-y) z wybraną strategią multitenancy.
- **Notatki:** [[wiedza/09-architektura/warstwy-hexagonal]] · [[wiedza/09-architektura/multitenancy]] · [[wiedza/09-architektura/monolit-vs-mikroserwisy]]

## Faza 10 — Cloud-native, konteneryzacja, DevOps 🔴 (wyróżnik fullstacka)
**Cel:** zapakować, wdrożyć, obserwować — pełen cykl od kodu do produkcji.

- **Konteneryzacja:** **Docker/Podman**, Dockerfile (multi-stage), **Jib**/buildpacks, obrazy warstwowe Spring Boot,
  GraalVM native image (świadomość — szybki start, mały ślad).
- **Orkiestracja:** **Kubernetes** (pody, deployments, services, ingress, configmaps/secrets, HPA), Helm (świadomość).
- **12-factor app**, externalized config, graceful shutdown, health/readiness probes.
- **Load balancing:** L4 vs L7, ingress, sesje bezstanowe (dlatego JWT), skalowanie poziome.
- **Obserwowalność:** **Micrometer** → Prometheus → Grafana; logi strukturalne; **OpenTelemetry** (tracing rozproszony).
- **CI/CD:** GitHub Actions / GitLab CI, build → test → obraz → deploy; GitOps (ArgoCD — świadomość).
- **Wzorce:** Sidecar, Service Discovery, Config Server, Circuit Breaker, Health Check.
- **Kamień milowy:** skonteneryzowany Spring Boot, deployment na lokalny k8s (kind/minikube), metryki w Grafanie.
- **Notatki:** [[wiedza/10-cloud-native/konteneryzacja]] · [[wiedza/10-cloud-native/kubernetes]] · [[wiedza/10-cloud-native/obserwowalnosc]] · [[wiedza/10-cloud-native/cicd]]

## Faza 11 — BPMN i Flowable 🟢 (rdzeń projektu docelowego)
**Cel:** silnik procesów biznesowych konfigurowalny przez adminów tenantów.

- **BPMN 2.0:** elementy (zadania, bramki, zdarzenia), modelowanie procesu rekrutacji.
- **Flowable:** silnik procesów/zadań, osadzenie w Spring Boot, deployment definicji procesów,
  user tasks, service tasks, listenery, zmienne procesowe, formularze, DMN (reguły decyzyjne).
- **Konfigurowalność:** wersjonowanie procesów per tenant, edycja przez admina (Flowable Modeler / własne UI).
- **Wzorce:** Process Engine, State Machine, Human Task, Workflow.
- **Kamień milowy:** proces rekrutacji w Flowable osadzony w Springu, wersjonowany per tenant.
- **Notatki:** [[wiedza/11-flowable-bpmn/bpmn-podstawy]] · [[wiedza/11-flowable-bpmn/flowable-spring]] · [[wiedza/11-flowable-bpmn/multitenant-procesy]]

## Faza 12 — Wzorce projektowe i refaktoryzacja 🟡 (przekrojowo)
**Cel:** wspólny język wzorców; rozpoznawać i stosować świadomie (nie na siłę).

- **GoF:** kreacyjne (Factory, Builder, Singleton), strukturalne (Adapter, Decorator, Facade, Proxy),
  behawioralne (Strategy, Observer, Template Method, State).
- **Enterprise:** Repository, Service Layer, DTO, Unit of Work, Dependency Injection, Specification.
- **Antywzorce:** God Object, anemic domain model (i kontrowersja), premature optimization, magic strings.
- **Refaktoryzacja:** code smells, bezpieczne kroki, oparcie o testy.
- **Notatki:** [[wiedza/12-wzorce/gof-przeglad]] · [[wiedza/12-wzorce/enterprise]] · [[wiedza/12-wzorce/antywzorce]]

---

## Jak korzystać z roadmapy

- **Nie rób jej liniowo do końca.** Po fazach 0–1 i podstawach 4–5 możesz zacząć budować, dociągając
  resztę „just-in-time" pod potrzeby projektu rekrutacyjnego.
- Po każdym temacie: wypełnij **Kryteria opanowania** w notatce i wygeneruj fiszki Anki.
- Co fazę: mały **kamień milowy** w `cwiczenia/` (kod > teoria).
- Śledzenie postępu: [[MOC]] zawiera tabelę statusu opanowania.

## Sugerowana ścieżka czasowa (orientacyjnie)

| Blok | Fazy | Efekt |
|---|---|---|
| **1. Język i warsztat** | 0–1, 4 | Piszesz idiomatyczną Javę + testy + Maven |
| **2. Backend webowy** | 5, 6, 7 | Budujesz realne REST API z bazą |
| **3. Produkcyjność** | 8, 10 | Bezpieczne, skonteneryzowane, obserwowalne API |
| **4. Głębia** | 2, 3, 9 | Brak black-boxów: JVM, współbieżność, architektura |
| **5. Projekt docelowy** | 11 + integracja całości | System rekrutacyjny (portfolio) |
