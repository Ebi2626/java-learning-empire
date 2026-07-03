---
temat: "Obserwowalność (Observability)"
faza: 10
status: nieopanowany
priorytet: 🔴
tags: [java, cloud, obserwowalnosc]
powiazane: ["[[wiedza/05-spring/actuator]]", "[[wiedza/04-narzedzia/logowanie]]", "[[wiedza/10-cloud-native/kubernetes]]", "[[wiedza/09-architektura/monolit-vs-mikroserwisy]]"]
---

# Obserwowalność (Observability)

> **TL;DR:** **Monitoring** odpowiada na *znane* pytania (predefiniowane dashboardy, alerty). **Observability**
> to zdolność zadawania *nowych* pytań o stan systemu na podstawie danych telemetrycznych. Stoi na **trzech
> filarach**: **metrics** (liczby w czasie), **logs** (zdarzenia), **traces** (ścieżka żądania przez usługi).
> W ekosystemie Java: **Micrometer** → **Prometheus** → **Grafana**, standard **OpenTelemetry**, a spoiwem jest
> **trace-id** wstrzykiwany do logów (MDC).

## 1. Co — definicja i API

**Monitoring** = mierzysz to, o czym już wiesz, że może się zepsuć. Masz dashboard „CPU / RAM / liczba 500-tek"
i alert „gdy latency > 500 ms". Świetne, dopóki awaria pasuje do jednego z Twoich przygotowanych pytań.

**Observability** = system emituje na tyle bogatą telemetrię, że możesz zadać pytanie, którego **nie przewidziałeś**
w momencie projektowania: „dlaczego akurat żądania od klienta X, w regionie EU, dla endpointu `/checkout`, po
deployu wersji 4.2, mają p99 3 s?". Nie musisz dokładać nowego kodu instrumentacji — odpowiedź jest już w danych.
Formalnie: system jest *observable*, jeśli jego stan wewnętrzny da się wywnioskować z jego wyjść (telemetrii).

Różnica jest praktyczna: monitoring to *known-unknowns*, observability to *unknown-unknowns*. W monolicie często
wystarczy monitoring. W systemach **rozproszonych / mikroserwisach** ([[wiedza/09-architektura/monolit-vs-mikroserwisy]])
jedno żądanie użytkownika przechodzi przez 10 usług — bez observability nie znajdziesz, *która* i *dlaczego* zawiodła.

**Trzy filary** (three pillars):

| Filar | Co to | Pytanie, na które odpowiada |
|-------|-------|------------------------------|
| **Metrics** | zagregowane liczby w czasie (time series) | *Ile? Jak szybko? Jak bardzo obciążone?* |
| **Logs** | dyskretne zdarzenia z kontekstem | *Co dokładnie się stało w tym momencie?* |
| **Traces** | ścieżka jednego żądania przez usługi | *Gdzie w łańcuchu poszedł czas / błąd?* |

Filary się uzupełniają, nie zastępują: metryka mówi *że* jest źle, trace *gdzie*, log *dlaczego*.

## 2. Jak — trzy filary, a pod spodem Micrometer / OTel / Prometheus

### Filar 1: METRICS

Metryki to **agregaty** — tanie w przechowywaniu, bo nie trzymasz każdego zdarzenia, tylko liczby próbkowane w czasie.
Typy instrumentów:

- **Counter** — monotonicznie rosnący licznik (liczba żądań, błędów). Nigdy nie maleje; liczysz *rate* (pochodną).
- **Gauge** — wartość, która rośnie i maleje (rozmiar kolejki, liczba aktywnych połączeń, zajętość heapu).
- **Histogram / Summary** — rozkład wartości (latency) w bucketach; pozwala liczyć percentyle (p50/p95/p99).
  Uwaga: percentyle liczone lokalnie (`Summary`) **nie sumują się** między instancjami — dlatego Prometheus
  preferuje **histogram** (buckety agreguje się poprawnie serwerowo, `histogram_quantile`).

Dwie kanoniczne „metodologie" doboru metryk:

- **RED** (dla usług obsługujących żądania) — **R**ate (żądań/s), **E**rrors (błędnych/s), **D**uration (latency).
- **USE** (dla zasobów: CPU, dysk, pula) — **U**tilization (% zajętości), **S**aturation (kolejka/oczekiwanie), **E**rrors.

**Micrometer** ([[wiedza/05-spring/actuator]]) to **fasada metryk** (vendor-neutral, „SLF4J dla metryk"). Piszesz
kod przeciw API Micrometera, a *registry* eksportuje do konkretnego backendu (Prometheus, Datadog, CloudWatch…).
W Spring Boot 3 Micrometer jest wbudowany; Actuator wystawia endpoint **`/actuator/prometheus`** w formacie tekstowym.

**Prometheus** to time-series database działająca w modelu **pull** — sam odpytuje (*scrape*) endpointy `/metrics`
co N sekund. Zapytania piszesz w **PromQL** (`rate(http_server_requests_seconds_count[5m])`). **Grafana** czyta
z Prometheusa i rysuje dashboardy. Model pull jest wygodny w k8s (service discovery znajduje pody automatycznie).

### Filar 2: LOGS

Logi to zdarzenia. Kluczowa praktyka: **structured logging** — zamiast wolnego tekstu emituj **JSON** z polami
(`level`, `service`, `traceId`, `userId`, `latencyMs`). Wtedy log jest przeszukiwalny i agregowalny w Loki/ELK,
a nie tylko `grep`-owalny. Szczegóły, appendery i **MDC** — patrz [[wiedza/04-narzedzia/logowanie]].

### Filar 3: TRACING (distributed tracing)

Śledzi **jedno żądanie** wędrujące przez wiele usług. Pojęcia:

- **Trace** — cały przebieg żądania end-to-end, identyfikowany przez **trace-id**.
- **Span** — pojedyncza operacja w tym przebiegu (wywołanie usługi, zapytanie DB), z własnym **span-id**,
  czasem trwania i relacją *parent → child*. Trace to drzewo spanów.
- **Context propagation** — trace-id i span-id są przekazywane między usługami, zwykle w nagłówkach HTTP
  (standard **W3C Trace Context**: nagłówek `traceparent`). Dzięki temu usługa B wie, że jej span należy do
  tego samego trace'u, co żądanie z usługi A. To spina rozproszone wywołania w jedno drzewo.

### Pod spodem: OpenTelemetry (OTel)

**OpenTelemetry** to *cross-language standard* obserwowalności (CNCF): jednolite **API** (do instrumentacji w kodzie),
**SDK** (implementacja: sampling, batching, eksport) oraz **Collector** (osobny proces: odbiera, przetwarza,
routuje telemetrię do backendów — Prometheus, Tempo, Jaeger). OTel obejmuje **wszystkie trzy filary** (metrics,
logs, traces) w jednym modelu. Instrumentacja bywa **automatyczna** (Java agent `-javaagent:opentelemetry-javaagent.jar`
instrumentuje popularne biblioteki bez zmian w kodzie) lub **manualna** (ręcznie tworzysz spany/metryki).

W świecie Spring: **Micrometer Tracing** działa jako **most (bridge)** — dostarcza spójne API tracingu w aplikacji,
a pod spodem deleguje do OTel (lub Brave). Spring Boot 3 przeszedł na model Micrometer Observation, gdzie jedna
„obserwacja" generuje jednocześnie metrykę i span.

### Korelacja: spoiwo trzech filarów

Magia dzieje się, gdy **trace-id trafia do logów** przez **MDC** (Mapped Diagnostic Context —
[[wiedza/04-narzedzia/logowanie]]). Micrometer/OTel wkłada `traceId` i `spanId` do MDC, a pattern loggera je
drukuje. Wtedy: widzisz błąd w tracie → kopiujesz trace-id → filtrujesz logi po tym trace-id → masz *dokładnie* te
linie logów, które dotyczą tego jednego żądania. Trzy filary łączą się w jedną narrację.

## 3. Dlaczego / kiedy — korelacja, alerting, pułapki

**Health / readiness.** Osobno od telemetrii diagnostycznej istnieją *probes* stanu. Actuator wystawia
`/actuator/health` z grupami **liveness** (czy proces żyje — restart, gdy nie) i **readiness** (czy gotowy
przyjmować ruch — usuń z load balancera, gdy nie). Kubernetes odpytuje je jako `livenessProbe` / `readinessProbe`
([[wiedza/10-cloud-native/kubernetes]]).

**Alerting i SLO.** Alerty w Prometheusie wysyła **Alertmanager** (deduplikacja, grupowanie, routing do Slack/PagerDuty).
Nowoczesne podejście: nie alertuj na surowe metryki, tylko na **SLO** (Service Level Objective, np. „99.9% żądań
< 300 ms w 30 dni"). **SLI** to zmierzony wskaźnik, **SLO** to cel, a **error budget** = dopuszczalna liczba
naruszeń (100% − SLO). Gdy budżet się wyczerpuje, wstrzymujesz nowe wdrożenia na rzecz stabilności — to zamienia
niezawodność w mierzalny, negocjowalny zasób.

**Pułapki:**

- **Kardynalność metryk (metric cardinality)** — najgroźniejsza. Każda unikalna kombinacja **tagów/labeli** to
  osobna time series. Tag o wysokiej kardynalności (`userId`, `requestId`, pełny URL z ID) tworzy miliony serii
  i **zabija Prometheus** (eksplozja pamięci, OOM). Reguła: tagi to *ograniczone, znane* zbiory (endpoint, status,
  region) — **nigdy** wartości nieograniczone. Wysokokardynalne dane należą do logów/traców, nie metryk.
- **Koszt i sampling tracingu** — pełne tracowanie 100% ruchu generuje ogromny wolumen. Stosuje się **sampling**
  (np. 1–10% traców, lub *tail-based sampling* — zachowaj trace, jeśli miał błąd/był wolny). Kompromis:
  za mały sampling → gubisz rzadkie problemy; za duży → koszt storage.
- **Logowanie danych wrażliwych** — PII, hasła, tokeny, numery kart w logach/tagach = incydent bezpieczeństwa
  i naruszenie RODO. Maskuj / nie loguj payloadów.
- **Overhead instrumentacji** — auto-instrumentacja agentem i eksport dokładają narzut CPU/latency; mierz go.

**Kiedy to szczególnie ważne:** im bardziej **rozproszony** system, tym większy zysk. W monolicie stack trace
często wystarczy. W mikroserwisach ([[wiedza/09-architektura/monolit-vs-mikroserwisy]]) bez distributed tracingu
debugowanie żądania przechodzącego przez 8 usług jest praktycznie niewykonalne.

## Przykład w praktyce

Spring Boot 3, Java 21. Wystawiamy metryki i propagujemy trace-id.

`application.yml` — ekspozycja endpointu Prometheusa i trace-id w logach:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, prometheus   # /actuator/prometheus scrapowany przez Prometheus
  tracing:
    sampling:
      probability: 0.1                 # sampluj 10% traców (koszt!)

logging:
  pattern:
    # trace-id / span-id z MDC wstrzyknięte przez Micrometer Tracing → korelacja logów z tracem
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

Zależności (Gradle): `micrometer-registry-prometheus` (eksport metryk) + `micrometer-tracing-bridge-otel`
+ `opentelemetry-exporter-otlp` (traces do OTel Collectora / Tempo).

Własna metryka **Timer** (mierzy rate + duration, czyli RED „za darmo"):
```java
@Service
class CheckoutService {
    private final Timer checkoutTimer;

    CheckoutService(MeterRegistry registry) {
        this.checkoutTimer = Timer.builder("checkout.duration")
            .tag("channel", "web")          // NISKA kardynalność — OK
            // .tag("userId", id)           // NIGDY: wysoka kardynalność zabije Prometheus
            .publishPercentileHistogram()   // histogram → poprawne p99 po agregacji
            .register(registry);
    }

    Order checkout(Cart cart) {
        return checkoutTimer.record(() -> process(cart));  // rejestruje czas i licznik
    }
}
```

Efekt: `GET /actuator/prometheus` zwraca m.in. `checkout_duration_seconds_bucket{channel="web",le="0.1"}`.
Prometheus scrapuje, Grafana rysuje p99, a gdy `process()` woła inną usługę HTTP, `traceparent` niesie trace-id —
w logach obu usług widnieje ten sam `%X{traceId}`, więc awarię odtwarzasz end-to-end.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię różnicę monitoring vs observability (known vs unknown questions).
- [ ] Wymienię trzy filary i powiem, na jakie pytanie każdy odpowiada.
- [ ] Rozróżnię Counter / Gauge / Histogram i wyjaśnię RED oraz USE.
- [ ] Wytłumaczę trace / span / trace-id / context propagation.
- [ ] Opiszę stack Micrometer → Prometheus → Grafana i rolę OpenTelemetry.
- [ ] Wyjaśnię, jak trace-id w MDC łączy logi z tracem.
- [ ] Wiem, czemu wysoka kardynalność tagów zabija Prometheus.

### 🔲 Black-box check
- [ ] Dlaczego Prometheus jest pull, a nie push, i co to daje w k8s?
- [ ] Czemu percentyle typu `Summary` nie sumują się między instancjami, a histogram tak?
- [ ] Jak konkretnie trace-id wędruje między usługami? (nagłówek `traceparent`, W3C Trace Context)
- [ ] Czym różni się liveness od readiness probe i po co ten podział?
- [ ] Co robi Micrometer Tracing jako „most" do OTel?

### 🎤 Pytania rekrutacyjne
- [ ] „Czym observability różni się od monitoringu?"
- [ ] „Jak zdiagnozowałbyś wolne żądanie przechodzące przez 5 mikroserwisów?"
- [ ] „Co to kardynalność metryki i czemu jest niebezpieczna?"
- [ ] „Jak połączysz log z konkretnym śladem (trace)?"
- [ ] „Co to SLO, SLI i error budget?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/10-cloud-native.csv`.*

```
Monitoring vs observability?;Monitoring = znane pytania/dashboardy (known-unknowns); observability = zadawanie NOWYCH pytań o stan z telemetrii (unknown-unknowns).
Trzy filary observability?;Metrics (liczby w czasie), Logs (zdarzenia), Traces (ścieżka żądania przez usługi).
Counter vs Gauge?;Counter rośnie monotonicznie (liczysz rate); Gauge rośnie i maleje (rozmiar kolejki, zajętość heapu).
Do czego histogram w metrykach?;Do rozkładu wartości (latency) w bucketach i liczenia percentyli p95/p99, poprawnie agregowalnych serwerowo.
Co to metodologia RED?;Rate (żądań/s), Errors (błędów/s), Duration (latency) — dla usług obsługujących żądania.
Co to metodologia USE?;Utilization, Saturation, Errors — dla zasobów (CPU, dysk, pula połączeń).
Co to Micrometer?;Vendor-neutralna fasada metryk (SLF4J dla metryk); eksportuje do backendów jak Prometheus, Datadog.
Model działania Prometheusa?;Pull — sam scrapuje endpointy /metrics co N sekund; time-series DB, zapytania w PromQL.
Rola Grafany?;Wizualizacja — czyta z Prometheusa i rysuje dashboardy.
Trace vs span?;Trace = całe żądanie end-to-end (trace-id); span = pojedyncza operacja w nim (span-id), z relacją parent→child.
Jak trace-id wędruje między usługami?;Context propagation przez nagłówki HTTP — standard W3C Trace Context, nagłówek traceparent.
Co to OpenTelemetry (OTel)?;Cross-language standard observability (API/SDK/Collector) obejmujący metrics, logs i traces.
Rola Micrometer Tracing?;Most (bridge) — spójne API tracingu w aplikacji delegujące pod spodem do OTel lub Brave.
Jak połączyć logi z tracem?;Wstrzyknąć trace-id do MDC i drukować w logu; filtrujesz logi po trace-id z tracingu.
Co robi Alertmanager?;Odbiera alerty z Prometheusa, deduplikuje, grupuje i routuje do Slack/PagerDuty.
SLI vs SLO vs error budget?;SLI = zmierzony wskaźnik; SLO = cel (np. 99.9%); error budget = dopuszczalny margines naruszeń (100% − SLO).
Dlaczego wysoka kardynalność tagów jest groźna?;Każda kombinacja labeli to osobna time series; wysokokardynalne tagi (userId, requestId) tworzą miliony serii i zabijają Prometheus.
Po co sampling w tracingu?;100% traców = ogromny wolumen/koszt; sampluje się część (np. 10% lub tail-based dla błędnych/wolnych).
Liveness vs readiness probe?;Liveness = czy proces żyje (restart, gdy nie); readiness = czy gotowy na ruch (usuń z LB, gdy nie).
Czym jest structured logging?;Logowanie zdarzeń jako JSON z polami (traceId, level, latency) — przeszukiwalne i agregowalne, nie tylko grep.
Endpoint metryk Prometheusa w Spring Boot?;/actuator/prometheus, wystawiony przez Actuator w formacie tekstowym Prometheusa.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/05-spring/actuator]] · [[wiedza/04-narzedzia/logowanie]] · [[wiedza/10-cloud-native/kubernetes]] · [[wiedza/09-architektura/monolit-vs-mikroserwisy]]
