---
temat: "Spring Boot Actuator"
faza: 5
status: nieopanowany
priorytet: 🔴
tags: [java, spring, obserwowalnosc]
powiazane: ["[[wiedza/10-cloud-native/kubernetes]]", "[[wiedza/10-cloud-native/obserwowalnosc]]", "[[wiedza/05-spring/autokonfiguracja]]"]
---

# Spring Boot Actuator

> **TL;DR:** **Actuator** dodaje gotowe **production-ready endpoints** (`/health`, `/metrics`, `/loggers`, `/prometheus`…)
> do monitoringu i zarządzania aplikacją w runtime. Pod spodem metryki zbiera **Micrometer** (fasada jak SLF4J dla logów)
> i eksportuje je do Prometheus/innych. Kluczowe dla k8s: **liveness** (czy restartować pod) i **readiness** (czy kierować ruch) probes.

## 1. Co — definicja i API

**Spring Boot Actuator** to moduł, który wystawia **operacyjny interfejs** do działającej aplikacji: stan zdrowia,
metryki, konfigurację, poziomy logów, listę beanów itd. To fundament **obserwowalności** (observability) i integracji
z platformami orkiestracji (Kubernetes) oraz systemami monitoringu (Prometheus, Grafana).

Dodanie sprowadza się do jednej zależności:
```groovy
implementation 'org.springframework.boot:spring-boot-starter-actuator'
```

Po dodaniu Actuator udostępnia **endpointy** pod prefiksem `/actuator` (web) — ale **domyślnie eksponuje tylko `/health`**.
Resztę trzeba świadomie włączyć (patrz sekcja 3 — bezpieczeństwo).

Kluczowe endpointy:

| Endpoint | Do czego |
|---|---|
| `/health` | Stan zdrowia (UP/DOWN) — agregat health indicators; **probes k8s** |
| `/info` | Metadane aplikacji (wersja, build, git — wkład przez `InfoContributor`) |
| `/metrics` | Lista metryk Micrometera; `/metrics/{name}` — konkretna metryka |
| `/prometheus` | Metryki w formacie scrape'owanym przez Prometheus |
| `/env` | `Environment` — properties, profile (wrażliwe! sekrety) |
| `/beans` | Wszystkie beany w `ApplicationContext` |
| `/mappings` | Mapowania `@RequestMapping` (jakie ścieżki → handler) |
| `/loggers` | Poziomy logów + **zmiana poziomu w runtime** (POST!) |
| `/threaddump` | Zrzut wątków (diagnoza deadlock/blokad) |
| `/heapdump` | Zrzut sterty (`.hprof` do analizy OOM) |
| `/conditions` | Raport auto-configuration: co i dlaczego się (nie) załączyło |
| `/httpexchanges` | Ostatnie żądania/odpowiedzi HTTP (wymaga beana repozytorium) |
| `/scheduledtasks` | Zadania `@Scheduled` i ich harmonogram |

Endpointy dzielą się na **web** (przez HTTP) i **JMX**. Włączamy je (enable) i eksponujemy (expose) niezależnie.

## 2. Jak — endpointy, health, Micrometer pod spodem

### Health — indicators, grupy, probes

`/health` to **agregat** wielu `HealthIndicator`-ów. Spring Boot auto-konfiguruje wiele z nich, gdy wykryje bibliotekę:
- `DataSourceHealthIndicator` (`db`) — wykonuje testowe zapytanie do bazy,
- `DiskSpaceHealthIndicator` (`diskSpace`) — sprawdza wolne miejsce,
- `PingHealthIndicator` (`ping`) — zawsze UP, banalny liveness,
- Redis, RabbitMQ, Kafka, Mongo… każdy dostawca ma swój indicator.

Ogólny status = **najgorszy** ze składowych (DOWN „wygrywa"). Szczegóły widać po `management.endpoint.health.show-details=always`.

**Grupy zdrowia (health groups)** pozwalają zdefiniować podzbiór indicatorów pod osobnym endpointem — dokładnie to
napędza **probes** dla [[wiedza/10-cloud-native/kubernetes]]:

- **Liveness probe** (`/actuator/health/liveness`) — *czy proces żyje / czy trzeba go zrestartować?* Powinien padać
  tylko przy stanie **nieodwracalnym** (np. zakleszczony wątek, uszkodzony stan wewnętrzny). Padnięcie ⇒ k8s **restartuje pod**.
- **Readiness probe** (`/actuator/health/readiness`) — *czy aplikacja jest gotowa przyjmować ruch?* Padnięcie ⇒ k8s
  **usuwa pod z Service/load balancera** (przestaje kierować ruch), ale **nie restartuje**. Wraca do ruchu, gdy znów READY.

**Kluczowa różnica:** liveness = „restartuj mnie", readiness = „nie dawaj mi teraz ruchu". Pomylenie ich jest klasycznym błędem
— np. wpięcie zależności do bazy w liveness powoduje **kaskadowe restarty** całej floty, gdy baza chwilowo znika (a wystarczyłby
readiness — pody przeczekałyby awarię). Probes włącza się flagą:
```properties
management.endpoint.health.probes.enabled=true
```
Spring Boot 3 na k8s auto-wykrywa środowisko i steruje readiness przez `ApplicationAvailability` (stany
`LivenessState`, `ReadinessState`) — podczas graceful shutdown przełącza się na `REFUSING_TRAFFIC`.

### Micrometer — fasada metryk

Metryki Actuatora napędza **Micrometer**: to **fasada / SLF4J dla metryk**. Piszesz kod raz przeciw
API Micrometera (`MeterRegistry`), a **backend** (Prometheus, Graphite, Datadog, CloudWatch…) podłączasz zależnością.
Dodanie `micrometer-registry-prometheus` sprawia, że pojawia się endpoint `/actuator/prometheus`.

Rodzaje mierników (`Meter`):
- **Counter** — monotonicznie rosnący licznik (liczba requestów, błędów). Tylko w górę.
- **Gauge** — chwilowa wartość, która rośnie i maleje (rozmiar kolejki, użycie pamięci, liczba aktywnych sesji).
- **Timer** — mierzy **czas trwania** i **liczbę** zdarzeń naraz (latencja endpointów); daje count, sum, max, percentyle.
- **DistributionSummary** — rozkład **wartości** nie-czasowych (np. rozmiar payloadu w bajtach).

**Tagi (dimensions)** to pary klucz-wartość doklejane do metryki (`endpoint=/orders`, `status=200`, `region=eu`).
Pozwalają filtrować/agregować w Prometheus/Grafana. **Uwaga na kardynalność** — tag o wysokiej liczbie wartości
(np. `userId`, `requestId`) eksploduje liczbę serii czasowych i zabija Prometheusa.

Własne metryki wstrzykujesz przez `MeterRegistry`. Micrometer zbiera też metryki automatyczne: JVM (heap, GC, wątki),
`http.server.requests` (Timer per endpoint), pula połączeń HikariCP, Tomcat itp.

## 3. Dlaczego / kiedy — probes k8s, bezpieczeństwo, pułapki

### Ekspozycja endpointów
Domyślnie po HTTP wystawiony jest **tylko `/health`**. Resztę włączasz świadomie:
```properties
management.endpoints.web.exposure.include=health,info,prometheus,metrics,loggers
# albo wszystko (ostrożnie!):
# management.endpoints.web.exposure.include=*
management.endpoints.web.exposure.exclude=env,beans
```
**Nie eksponuj wszystkiego bez zastanowienia.** `/env`, `/heapdump`, `/threaddump`, `/beans`, `/configprops` ujawniają
**wrażliwe dane** (sekrety w properties, pełny stan aplikacji, zawartość pamięci). To wektor ataku i wyciek danych.

### Bezpieczeństwo Actuatora
- **Osobny port zarządzania:** `management.server.port=9090` przenosi endpointy na inny port, którego **nie wystawiasz
  publicznie** (tylko sieć wewnętrzna / sidecar). Skutecznie oddziela ruch operacyjny od użytkowego.
- **Spring Security:** zabezpiecz `/actuator/**` uwierzytelnieniem/autoryzacją (rola `ACTUATOR_ADMIN`). Health/probes zwykle
  zostawia się publiczne (k8s musi je odpytać bez tokenu), resztę chronisz.
- **Sanityzacja:** wartości `/env` i `/configprops` są domyślnie maskowane (`show-values=NEVER`) — nie polegaj tylko na tym.

### Pułapki
- **`/health` z detalami bez zabezpieczenia** — `show-details=always` publicznie ujawnia strukturę infrastruktury (adresy baz).
- **Zależności zewnętrzne w liveness** — kaskadowe restarty (opisane wyżej).
- **Wysoka kardynalność tagów** — degradacja monitoringu.
- **`/heapdump` publicznie** — ktoś może pobrać zrzut pamięci z sekretami/danymi.
- **Koszt metryk** — Timer z percentylami (`histogram`) generuje wiele serii; włączaj świadomie per metryka.

**Kiedy używać:** praktycznie każda usługa produkcyjna na k8s/chmurze. **Kiedy uważać:** ekspozycja publiczna —
zawsze przez osobny port lub Security. Zobacz szerszy kontekst: [[wiedza/10-cloud-native/obserwowalnosc]].

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Serwis zamówień na Kubernetes: probes napędzają health-checki, a metryki lecą do Prometheusa. Poniżej ekspozycja +
własny `HealthIndicator` (sprawdza zewnętrzne API płatności) + własna metryka `Counter` zliczająca złożone zamówienia.

`application.properties`:
```properties
management.endpoints.web.exposure.include=health,info,prometheus,loggers
management.endpoint.health.show-details=when-authorized
management.endpoint.health.probes.enabled=true
management.server.port=9090
management.info.git.mode=full
```

Własny health indicator:
```java
@Component
class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final PaymentClient client;

    PaymentGatewayHealthIndicator(PaymentClient client) {
        this.client = client;
    }

    @Override
    public Health health() {
        try {
            var latency = client.ping();               // np. HEAD /status
            return Health.up()
                    .withDetail("latencyMs", latency)
                    .build();
        } catch (Exception e) {
            return Health.down(e)                       // ⇒ /health = DOWN
                    .withDetail("gateway", "unreachable")
                    .build();
        }
    }
}
```

Własna metryka (Counter) + wkład do `/info`:
```java
@Service
class OrderService {

    private final Counter placedOrders;

    OrderService(MeterRegistry registry) {
        this.placedOrders = Counter.builder("orders.placed")
                .description("Liczba złożonych zamówień")
                .tag("channel", "web")                  // tag/dimension
                .register(registry);
    }

    void placeOrder(Order order) {
        // ... logika domenowa ...
        placedOrders.increment();                       // widoczne w /prometheus
    }
}

@Component
class BuildInfoContributor implements InfoContributor {
    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("region", "eu-central-1");   // pojawia się w /info
    }
}
```

Runtime debugging bez redeploya — podniesienie logów pakietu na `DEBUG` przez `/loggers`:
```bash
curl -X POST localhost:9090/actuator/loggers/com.acme.orders \
     -H 'Content-Type: application/json' \
     -d '{"configuredLevel":"DEBUG"}'
```

Manifest k8s wpina probes:
```yaml
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 9090 }
readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: 9090 }
```

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, czym jest Actuator i po co (production-ready endpoints, obserwowalność).
- [ ] Znam kluczowe endpointy i wiem, które są wrażliwe.
- [ ] Rozumiem różnicę liveness vs readiness i skutki ich pomylenia na k8s.
- [ ] Wiem, że Micrometer to fasada metryk (jak SLF4J) i znam 4 typy mierników.
- [ ] Napiszę własny `HealthIndicator` i własną metrykę `Counter` z głowy.
- [ ] Wiem, jak bezpiecznie eksponować endpointy (osobny port, Security).

### 🔲 Black-box check
- [ ] Jak `/health` agreguje wynik z wielu `HealthIndicator`? (najgorszy status wygrywa)
- [ ] Co technicznie robi readiness probe, gdy padnie, a co liveness? (usunięcie z LB vs restart)
- [ ] Jak endpoint `/prometheus` powstaje i skąd bierze dane? (Micrometer registry + zależność)
- [ ] Czym różni się `include` od `enable` dla endpointów?
- [ ] Dlaczego wysoka kardynalność tagów jest groźna?

### 🎤 Pytania rekrutacyjne
- [ ] „Liveness vs readiness — czym się różnią i po co k8s dwie sondy?"
- [ ] „Jak zmienić poziom logów bez restartu aplikacji?" (`/loggers`, POST)
- [ ] „Co to Micrometer i jak ma się do Prometheusa?"
- [ ] „Jak zabezpieczyć endpointy Actuatora na produkcji?"
- [ ] „Kiedy użyjesz Counter, a kiedy Gauge?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/05-spring.csv`. Format: `Pytanie;Odpowiedź`.*

```
Co dodaje Spring Boot Actuator?;Production-ready endpointy do monitoringu i zarządzania aplikacją w runtime (health, metrics, loggers...).
Jaka zależność włącza Actuator?;spring-boot-starter-actuator.
Który endpoint jest domyślnie eksponowany po HTTP?;Tylko /health — resztę trzeba świadomie włączyć przez exposure.include.
Jak wyeksponować wybrane endpointy?;management.endpoints.web.exposure.include=health,info,prometheus (lub * dla wszystkich).
Co robi /loggers przez POST?;Zmienia poziom logowania danego pakietu w runtime, bez restartu aplikacji.
Do czego /threaddump i /heapdump?;Diagnostyka: zrzut wątków (deadlocki) i zrzut sterty (.hprof, analiza OOM).
Do czego /conditions?;Raport auto-configuration — co i dlaczego się (nie) załączyło.
Jak /health liczy status ogólny?;Agreguje HealthIndicatory; wynik = najgorszy status (DOWN wygrywa nad UP).
Przykłady wbudowanych health indicators?;db (DataSource), diskSpace, ping oraz per dostawca (Redis, Kafka, Mongo...).
Liveness probe — co oznacza padnięcie?;Proces w nieodwracalnym stanie ⇒ Kubernetes restartuje pod.
Readiness probe — co oznacza padnięcie?;Aplikacja niegotowa ⇒ k8s usuwa pod z load balancera (nie kieruje ruchu), ale nie restartuje.
Klasyczny błąd z probes?;Zależność do bazy w liveness ⇒ kaskadowe restarty floty przy chwilowej awarii bazy (powinien być readiness).
Czym jest Micrometer?;Fasadą/abstrakcją metryk (jak SLF4J dla logów); kod raz, backend (Prometheus, Datadog...) przez zależność.
Cztery typy mierników Micrometera?;Counter (rosnący), Gauge (chwilowa wartość +/-), Timer (czas+liczba), DistributionSummary (rozkład wartości).
Counter vs Gauge?;Counter rośnie monotonicznie (liczba zdarzeń); Gauge to chwilowa wartość, która rośnie i maleje.
Co robią tagi (dimensions) metryk?;Doklejają wymiary (endpoint, status) do filtrowania/agregacji; uwaga na wysoką kardynalność.
Dlaczego wysoka kardynalność tagów jest groźna?;Każda kombinacja wartości to nowa seria czasowa — eksplozja serii zabija Prometheusa.
Jak dodać własny health check?;Zaimplementować HealthIndicator (@Component) i zwrócić Health.up()/down() z detalami.
Jak stworzyć własną metrykę?;Wstrzyknąć MeterRegistry i zbudować np. Counter.builder("nazwa").register(registry).
Jak bezpiecznie wystawić Actuator na produkcji?;Osobny port (management.server.port), zabezpieczenie Spring Security, nie eksponować /env, /heapdump publicznie.
Jak dorzucić dane do /info?;Zaimplementować InfoContributor i dodać detale przez Info.Builder.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/05-spring/autokonfiguracja]] · [[wiedza/10-cloud-native/kubernetes]] · [[wiedza/10-cloud-native/obserwowalnosc]]
