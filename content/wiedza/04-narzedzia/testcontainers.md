---
temat: "Testcontainers"
faza: 4
status: nieopanowany
priorytet: 🔴
tags: [java, narzedzia, testy]
powiazane: ["[[wiedza/06-persystencja/jpa-hibernate]]", "[[wiedza/10-cloud-native/konteneryzacja]]", "[[wiedza/04-narzedzia/junit-mockito]]"]
---

# Testcontainers

> **TL;DR:** **Testcontainers** startuje **efemeryczne kontenery Docker/OCI** (PostgreSQL, Kafka, Redis…)
> na czas testu i sprząta je po nim, dzięki czemu testy integracyjne działają na **realnej** zależności
> zamiast na mocku/H2. W Spring Boot 3.1+ wpina się to jednym `@ServiceConnection`, a start (kosztowny)
> współdzielisz **static `@Container`** / **singleton container pattern**.

## 1. Co — definicja i API

### PROBLEM, który rozwiązuje
Testy integracyjne warstwy persystencji tradycyjnie robiono na **bazie in-memory (H2, HSQLDB)** albo na mockach.
To **kłamie**:
- H2 to **inny silnik SQL** niż PostgreSQL — inne typy (`jsonb`, `array`, `uuid`, `citext`), inne funkcje,
  inna składnia (`ILIKE`, `RETURNING`, window functions), inna semantyka izolacji transakcji, inne sekwencje.
- Zapytanie działające na H2 potrafi wywalić się na produkcyjnym Postgresie i odwrotnie — a błąd wychodzi
  dopiero na proddzie. Tryby zgodności H2 (`MODE=PostgreSQL`) łatają tylko ułamek różnic.
- **Mock repozytorium** testuje twój kod, ale nie testuje **mapowania ORM, migracji Flyway/Liquibase,
  dialektu Hibernate ani realnych zapytań** — czyli tego, co najczęściej się psuje.

**Zasada:** *test na tej samej bazie, co produkcja.* Jeśli prod to PostgreSQL 16 — testuj na PostgreSQL 16.

### Czym jest Testcontainers
Biblioteka Javy (są też porty na inne języki), która **programowo z poziomu testu**:
1. uruchamia kontener z obrazu OCI (np. `postgres:16-alpine`),
2. czeka aż usługa w środku będzie **gotowa** (wait strategy),
3. udostępnia **dynamiczny, zmapowany port** hosta,
4. **niszczy kontener** po zakończeniu testów (przez sidecar **Ryuk**).

Wymaga działającego **runtime OCI** (Docker, Podman, Colima, Rancher Desktop) na maszynie/CI.

Minimalny przykład (JUnit 5, moduł dedykowany):
```java
@Testcontainers
class RepositoryIT {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("app")
            .withUsername("test")
            .withPassword("test");

    @Test
    void containerRuns() {
        assertTrue(postgres.isRunning());
        String jdbc = postgres.getJdbcUrl();   // jdbc:postgresql://localhost:<random-port>/app
    }
}
```

### GenericContainer vs moduły dedykowane
- **Moduły dedykowane** — gotowe klasy z wiedzą o danej usłudze: `PostgreSQLContainer`, `KafkaContainer`,
  `MongoDBContainer`, `MySQLContainer`, `GenericContainer`. Dają metody typu `getJdbcUrl()`,
  `getBootstrapServers()`, sensowną domyślną wait strategy i porty. **Zawsze preferuj moduł, jeśli istnieje.**
- **`GenericContainer`** — dla dowolnego obrazu bez modułu (własny serwis, Nginx, WireMock w kontenerze):
  sam podajesz porty, env, wait strategy.

```java
GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
    .withExposedPorts(6379)
    .waitingFor(Wait.forListeningPort());
```

### Kluczowe API konfiguracji
- **`withExposedPorts(int...)`** — porty *wewnątrz* kontenera do udostępnienia na hoście.
- **`getMappedPort(int)`** — zwraca **losowy port hosta** przypisany do portu kontenera. Nigdy nie zakładaj
  stałego portu — kontenery startują na losowych portach, by nie kolidować (kilka testów równolegle).
- **`withEnv(k, v)`** — zmienne środowiskowe (np. `POSTGRES_PASSWORD`, konfiguracja obrazu).
- **`getHost()`** — host runtime (zwykle `localhost`, ale nie zawsze — używaj tej metody, nie hardcode).
- **Wait strategy** — o tym niżej; bez niej test może dołączyć do jeszcze niegotowej usługi.
- **Logi** — `container.getLogs()` lub `.withLogConsumer(new Slf4jLogConsumer(log))` do debugowania startu.

## 2. Jak — cykl życia, integracja Spring pod spodem

### Cykl życia kontenera
Testcontainers wpina się w cykl życia JUnit 5 przez rozszerzenie aktywowane adnotacją `@Testcontainers`:
1. **start** — pobranie obrazu (jeśli brak lokalnie), `docker create` + `docker start`,
2. **wait** — blokowanie aż wait strategy zaświeci na zielono,
3. **testy** — używają zmapowanego portu,
4. **stop** — `docker stop` + `docker rm`.

### `@Testcontainers`, `@Container` — statyczny vs instancyjny (WYDAJNOŚĆ)
- **`@Testcontainers`** na klasie — rejestruje `TestcontainersExtension`, który zarządza polami `@Container`.
- **`@Container` na polu `static`** — kontener startuje **raz przed wszystkimi testami klasy** i gaśnie
  po ostatnim. **Współdzielony** = jeden drogi start na całą klasę.
- **`@Container` na polu instancyjnym (niestatycznym)** — kontener startuje **przed każdą metodą testową**
  i gaśnie po niej. Pełna izolacja, ale **N testów = N startów** kontenera.

> Start kontenera to sekundy (obraz, init bazy, migracje). **Domyślnie używaj `static`.** Izolację danych
> między testami zapewniaj transakcyjnym rollbackiem, czyszczeniem tabel lub `@Sql`, a nie restartem kontenera.

| Wariant | Kiedy start | Izolacja | Koszt |
|---|---|---|---|
| `static @Container` | raz na klasę | współdzielony stan | 🟢 niski |
| instancyjny `@Container` | przed każdym testem | pełna | 🔴 wysoki |

### Wait strategy — dlaczego to krytyczne
Kontener „uruchomiony" (proces PID 1 żyje) ≠ „usługa gotowa" (baza przyjmuje połączenia). Bez czekania
dostaniesz flaky testy (connection refused). Strategie:
- `Wait.forListeningPort()` — port nasłuchuje (proste, ale usługa może jeszcze się inicjalizować),
- `Wait.forLogMessage("...ready to accept connections.*", 1)` — regex w logach (tego używa `PostgreSQLContainer`),
- `Wait.forHttp("/health").forStatusCode(200)` — health-check HTTP,
- `.waitingFor(strategy).withStartupTimeout(Duration.ofSeconds(60))`.

Moduły dedykowane mają **sensowne domyślne** wait strategy — dlatego działają „z pudełka".

### Integracja ze Spring Boot — 3 sposoby (od starego do nowego)

**A. `@DynamicPropertySource` (działa od Spring Boot 2.2, uniwersalne).**
Port kontenera jest losowy i znany dopiero po starcie — nie wpiszesz go statycznie do `application.yml`.
Ta metoda **wstrzykuje właściwości do `Environment` przed utworzeniem kontekstu**:
```java
@SpringBootTest
@Testcontainers
class JpaRepositoryIT {

    @Container
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);       // suppliery, nie wartości!
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
    }
}
```
Metoda musi być `static`, a wartości podaje się jako **suppliery** (`pg::getJdbcUrl`) — ewaluowane
leniwie, po starcie kontenera.

**B. `@ServiceConnection` (Spring Boot 3.1+, zalecane).**
Boot **sam rozpoznaje typ kontenera** (Postgres → datasource, Kafka → bootstrap servers, Redis → connection)
i konfiguruje odpowiednie properties. Zero boilerplate `@DynamicPropertySource`:
```java
@SpringBootTest
@Testcontainers
class JpaRepositoryIT {

    @Container
    @ServiceConnection                                   // Boot łączy kontekst z tym kontenerem
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");
}
```
Pod spodem działa mechanizm `ConnectionDetails` — Boot ma `ConnectionDetailsFactory` dla znanych kontenerów,
która z uruchomionego kontenera wyciąga URL/host/port/credentiale i wystawia jako bean, nadpisując
autokonfigurację datasource.

**C. `@Import(TestcontainersConfiguration.class)` + `@TestConfiguration`.**
Kontenery definiuje się jako **beany** w osobnej klasie konfiguracyjnej — reużywalne między testami
i spinane z podglądem lokalnym (`SpringApplication.from(...).with(...)` w `src/test`, tryb „dev-time”):
```java
@TestConfiguration(proxyBeanMethods = false)
class TestcontainersConfiguration {
    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgres() {
        return new PostgreSQLContainer<>("postgres:16-alpine");
    }
}
// w teście: @SpringBootTest @Import(TestcontainersConfiguration.class)
```

### Ryuk — sprzątanie
Testcontainers startuje sidecar **Ryuk** (`testcontainers/ryuk`) — mały kontener-strażnik, który
**usuwa kontenery/sieci/wolumeny testu**, nawet gdy JVM padnie (kill -9, timeout CI). To gwarantuje,
że nie zostają „sieroty" zjadające zasoby. Można go wyłączyć (`TESTCONTAINERS_RYUK_DISABLED=true`),
ale wtedy sam odpowiadasz za sprzątanie.

### Reuse i sieci
- **Reuse** (`.withReuse(true)` + `testcontainers.reuse.enable=true` w `~/.testcontainers.properties`)
  — kontener **nie jest zatrzymywany** po testach i **kolejny run go współdzieli**. Ryuk go wtedy nie ubija.
  Świetne do **lokalnej pętli dev** (szybsze powtórne uruchomienia); **nie włączaj tego na CI**
  (ma być czysto i deterministycznie).
- **`Network`** — gdy kilka kontenerów musi się widzieć nawzajem (np. app-kontener → Kafka → Zookeeper),
  wrzucasz je do wspólnej `Network.newNetwork()` i adresujesz po **network alias**, nie po `localhost`.

## 3. Dlaczego / kiedy — wydajność, Podman, pułapki

### Wydajność — start jest drogi
Największy koszt to **start kontenera** (pull obrazu raz, potem init bazy + migracje za każdym startem).
Optymalizacje:
- **Współdziel** — `static @Container` zamiast instancyjnego.
- **Singleton container pattern** — jeden kontener na **cały suite testów** (wiele klas), startowany ręcznie:
  ```java
  abstract class AbstractIT {
      static final PostgreSQLContainer<?> PG =
          new PostgreSQLContainer<>("postgres:16-alpine");
      static { PG.start(); }   // start raz; brak @Container = brak stop → gaśnie z JVM
  }
  ```
  Klasy testowe dziedziczą po `AbstractIT` — jeden Postgres na wszystkie. Reużywalne z `@ServiceConnection`
  przez wspólną klasę konfiguracyjną. Dbaj tylko o **czyszczenie stanu** między testami.
- **Reuse** lokalnie, **lekkie obrazy** (`-alpine`), **spójny kontekst Springa** (nie mnóż `@SpringBootTest`
  o różnych konfiguracjach — każda unikalna konfiguracja = nowy kontekst).

### PODMAN — kompatybilność (praktyka)
Testcontainers gada z runtime przez **socket Docker API**. Podman go emuluje, ale trzeba go wskazać:
- **Uruchom socket Podmana** i wskaż go zmienną:
  ```bash
  systemctl --user enable --now podman.socket
  export DOCKER_HOST="unix:///run/user/$(id -u)/podman/podman.sock"
  ```
  (odpowiednik `podman info` → `Docker-compatible API`). Nowsze Testcontainers autodetekcją znajdują
  socket Podmana, ale jawne `DOCKER_HOST` jest najpewniejsze.
- **Ryuk z Podman rootless** bywa problematyczny (uprawnienia do zarządzania kontenerami/socketem) —
  jeśli Ryuk się wywala, ustaw **`TESTCONTAINERS_RYUK_DISABLED=true`**. Kosztem jest brak
  automatycznego sprzątania, więc dopilnuj `stop()` / `podman ps` po sesji.
- Alternatywnie `export TESTCONTAINERS_RYUK_PRIVILEGED=true` (Ryuk wymaga trybu privileged) —
  albo skonfiguruj socket tak, by miał uprawnienia. Na Podman rootless najczęściej wystarcza kombinacja
  `DOCKER_HOST` + (w razie problemów) `TESTCONTAINERS_RYUK_DISABLED`.

### Pułapki
- **Hardcode portu/hosta** — zawsze `getMappedPort()` / `getHost()`, nigdy `5432`/`localhost` na sztywno.
- **Brak wait strategy przy `GenericContainer`** — flaky „connection refused".
- **Instancyjny `@Container` przez pomyłkę** — dramatyczne spowolnienie suite'u.
- **`reuse` na CI** — pozostawia stan między buildami, testy przestają być deterministyczne.
- **Brak Dockera/Podmana na CI** — Testcontainers wymaga runtime OCI (potrzebny „docker-in-docker"
  albo host socket na agencie).
- **Wartości zamiast supplierów** w `@DynamicPropertySource` — NPE, bo kontener jeszcze nie wystartował.
- **Wolne obrazy/pull na CI** — cache’uj warstwy obrazów w pipeline.

### Kiedy NIE
- Czysty **test jednostkowy** logiki domenowej (bez I/O) — nie potrzebujesz kontenera, mockuj granice.
- Środowisko **bez runtime OCI** i bez możliwości go dodać.

## Przykład w praktyce
Test repozytorium JPA na realnym PostgreSQL, Spring Boot 3, Java 21:
```java
@DataJpaTest                                            // tnie kontekst do warstwy JPA
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)  // NIE podmieniaj na H2!
@Testcontainers
class UserRepositoryIT {

    @Container
    @ServiceConnection                                  // Boot 3.1+: auto-config datasource
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired UserRepository repo;

    @Test
    void savesAndQueriesByEmailIgnoreCase() {
        repo.save(new User("Edwin", "Edwin@Example.com"));

        // działa na realnym Postgresie: ILIKE, indeksy, dialekt — czego H2 nie odda
        var found = repo.findByEmailIgnoreCase("edwin@example.com");

        assertThat(found).isPresent();
    }
}
```
Kluczowe: `@AutoConfigureTestDatabase(replace = NONE)` — bez tego `@DataJpaTest` **podmieniłby**
datasource na H2 i cały sens by przepadł. `static` + `@ServiceConnection` = jeden kontener na klasę,
zero boilerplate.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, czemu H2/mock nie zastąpi realnego Postgresa w testach persystencji.
- [ ] Rozumiem cykl życia kontenera i różnicę `static` vs instancyjny `@Container` (wydajność).
- [ ] Znam 3 sposoby integracji ze Spring Boot i wiem, który jest najnowszy/zalecany.
- [ ] Wiem, po co jest Ryuk, reuse i Network.
- [ ] Skonfiguruję Testcontainers pod Podman (DOCKER_HOST, RYUK_DISABLED).
- [ ] Napiszę test repozytorium JPA na `PostgreSQLContainer` z głowy.

### 🔲 Black-box check
- [ ] Co dokładnie robi `@Testcontainers` i jak wpina się w JUnit 5?
- [ ] Skąd Spring wie, jaki URL/port ma kontener przy `@ServiceConnection`? (`ConnectionDetailsFactory`)
- [ ] Dlaczego `@DynamicPropertySource` używa supplierów, a nie wartości?
- [ ] Czym różni się „kontener wystartował" od „usługa gotowa"? (wait strategy)
- [ ] Co robi Ryuk i co się stanie, gdy JVM padnie w środku testu?
- [ ] Dlaczego port jest losowy i jak go poznać? (`getMappedPort`)

### 🎤 Pytania rekrutacyjne
- [ ] „Czemu nie testować na H2 zamiast Testcontainers?”
- [ ] „`@Container` static vs instancyjny — kiedy który i jaki jest koszt?”
- [ ] „Jak wstrzyknąć losowy port kontenera do Springa?” (`@DynamicPropertySource` / `@ServiceConnection`)
- [ ] „Jak przyspieszyć wolny suite integracyjny?” (współdzielenie, singleton pattern, reuse lokalnie)
- [ ] „Uruchamiasz na Podman zamiast Dockera — co konfigurujesz?” (DOCKER_HOST, Ryuk)

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/04-narzedzia.csv`.*

```
Co robi Testcontainers?;Startuje efemeryczne kontenery Docker/OCI (baza, kolejka…) na czas testu i sprząta po nim, dając testy na realnej zależności zamiast mocka/H2.
Dlaczego H2 nie zastępuje PostgreSQL w testach?;Inny silnik SQL — inne typy (jsonb, uuid), funkcje, składnia i semantyka; zapytanie działające na H2 może paść na prodowym Postgresie.
Do czego służy @Testcontainers?;Aktywuje rozszerzenie JUnit 5 zarządzające cyklem życia pól @Container (start/stop).
Różnica: @Container static vs instancyjny?;Static = kontener startuje raz na klasę (współdzielony, szybki); instancyjny = przed każdym testem (izolowany, N startów, wolny).
Co domyślnie preferować: static czy instancyjny @Container?;Static — start jest drogi; izolację danych rób rollbackiem/czyszczeniem, nie restartem kontenera.
GenericContainer vs moduł dedykowany?;Moduł (PostgreSQLContainer, KafkaContainer…) zna usługę i ma getJdbcUrl/wait strategy; GenericContainer to dowolny obraz z ręczną konfiguracją.
Co robi getMappedPort()?;Zwraca losowy port hosta przypisany do portu kontenera — nigdy nie zakładaj stałego portu.
Po co wait strategy?;Kontener uruchomiony ≠ usługa gotowa; bez czekania na gotowość testy są flaky (connection refused).
Przykłady wait strategy w Testcontainers?;Wait.forListeningPort(), Wait.forLogMessage(regex), Wait.forHttp(path).forStatusCode(200).
Do czego @DynamicPropertySource?;Wstrzykuje właściwości (URL, port, credentiale) do Environment przed startem kontekstu; wartości jako suppliery, metoda static.
Co daje @ServiceConnection (Spring Boot 3.1+)?;Boot sam rozpoznaje typ kontenera i konfiguruje datasource/kafkę/redis — zastępuje ręczny @DynamicPropertySource.
Jak Spring wie o URL kontenera przy @ServiceConnection?;Przez ConnectionDetailsFactory, która z uruchomionego kontenera wystawia bean ConnectionDetails nadpisujący autokonfigurację.
Co to Ryuk w Testcontainers?;Sidecar-strażnik usuwający kontenery/sieci/wolumeny testu, nawet gdy JVM padnie — gwarancja sprzątania.
Co robi reuse (testcontainers.reuse.enable)?;Kontener nie jest zatrzymywany po testach i kolejny run go współdzieli — szybsza pętla lokalna; NIE na CI (niedeterministyczne).
Singleton container pattern?;Jeden kontener na cały suite (start w static/abstract bazie, bez @Container/stop) — minimalizuje kosztowne starty.
Testcontainers z Podman — co ustawić?;DOCKER_HOST na socket Podmana (podman.socket); przy Podman rootless czasem TESTCONTAINERS_RYUK_DISABLED=true lub RYUK_PRIVILEGED=true.
Po co @AutoConfigureTestDatabase(replace=NONE) z @DataJpaTest?;Bez tego Boot podmienia datasource na H2 — psuje sens testu na realnym Postgresie z Testcontainers.
Kiedy NIE używać Testcontainers?;Przy czystych testach jednostkowych logiki bez I/O oraz gdy brak runtime OCI (Docker/Podman) na maszynie/CI.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/06-persystencja/jpa-hibernate]] · [[wiedza/10-cloud-native/konteneryzacja]] · [[wiedza/04-narzedzia/junit-mockito]]
