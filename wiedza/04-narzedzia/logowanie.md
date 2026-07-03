---
temat: "Logowanie w Javie (SLF4J + Logback/Log4j2)"
faza: 4
status: nieopanowany
priorytet: 🔴
tags: [java, narzedzia, obserwowalnosc]
powiazane: ["[[wiedza/10-cloud-native/obserwowalnosc]]", "[[wiedza/04-narzedzia/spring-boot-config]]", "[[wiedza/00-fundament/jdk-jre-jvm]]"]
---

# Logowanie w Javie (SLF4J + Logback/Log4j2)

> **TL;DR:** Loguj przez **SLF4J** (fasadę, jedno API), a implementację (**Logback** — domyślny w Spring Boot, albo **Log4j2** — szybki, async) podepnij na classpath jako *binding*. Używaj **logowania parametrycznego** `log.info("user {} zrobił {}", id, akcja)` (leniwe formatowanie), poziomów **TRACE→ERROR** z sensem, **MDC** na correlation/trace id per żądanie (i **czyść** go w thread poolu), a w chmurze — **logów strukturalnych JSON** korelowanych z tracingiem. Nigdy nie loguj sekretów/PII i nie stosuj **log-and-throw**.

## 1. Co — definicja i API

Logowanie to **strukturalny, sterowalny zapis zdarzeń** aplikacji (co się stało, kiedy, w jakim kontekście) do skonfigurowanych ujść (konsola, plik, agregator).

**Dlaczego nie `System.out.println`?**
- brak **poziomów** — nie wyłączysz szumu na produkcji bez rekompilacji;
- brak **kontekstu** (timestamp, wątek, logger, correlation id, poziom) — dokładasz go ręcznie;
- brak **appenderów/rotacji** — `stdout` puchnie, nie wyślesz do pliku/ELK selektywnie;
- **synchroniczny** zapis na `System.out` (globalny lock `PrintStream`) — dławi throughput;
- nieparsowalny, niekonfigurowalny per pakiet, nie da się routować.

**Minimalny przykład (SLF4J API):**
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public void place(Long userId, String action) {
        log.info("user {} zrobił {}", userId, action);   // parametrycznie, nie konkatenacja
        log.debug("szczegóły zamówienia: {}", order);      // DEBUG — tylko diagnostyka
    }
}
```

- `Logger`/`LoggerFactory` pochodzą z **`org.slf4j`** — to **API/fasada**, nie implementacja.
- Nazwa loggera zwykle = FQCN klasy (`getClass()`), bo po niej konfiguruje się **poziomy per pakiet**.
- Konwencja: `private static final Logger log` (jeden na klasę, wątkowo-bezpieczny).

## 2. Jak — fasada, bindingi, MDC pod spodem

### SLF4J jako fasada vs implementacje
**SLF4J** (*Simple Logging Facade for Java*) to tylko **interfejsy** (`Logger`, `Marker`, `MDC`). Sam nic nie zapisuje — deleguje do implementacji wybranej **w czasie ładowania klas** na podstawie classpath.

```
   Twój kod  ──►  SLF4J API (fasada)  ──►  binding  ──►  implementacja
   log.info()     org.slf4j.Logger        slf4j-*        Logback / Log4j2 / JUL
```

- **Logback** — natywnie implementuje SLF4J (ten sam autor, Ceki Gülcü). **Domyślny w Spring Boot** (`spring-boot-starter-logging`). Solidny, dobry XML config, MDC, rolling.
- **Log4j2** — osobna implementacja; podpinana bridgem `log4j-slf4j2-impl`. **Wydajny** dzięki **async loggerom na LMAX Disruptor** (lock-free ring buffer), garbage-free mode. Wybierasz go przy dużym wolumenie logów. (Nie mylić z martwym Log4j 1.x; podatność **Log4Shell/CVE-2021-44228** dotyczyła Log4j2 < 2.17 — JNDI lookup w wiadomościach.)

**Po co w ogóle fasada?**
1. **Wymienialność implementacji** bez zmiany kodu — biblioteka loguje przez SLF4J, aplikacja decyduje, co jest pod spodem.
2. **Jedno API** dla całego drzewa zależności — inaczej każda lib logowałaby po swojemu.
3. **Logowanie parametryczne + MDC** w kontrakcie API, niezależnie od backendu.

### Bindingi i problem wielu implementacji na classpath
Do działania potrzebujesz **dokładnie jednego bindingu**:

| Sytuacja | Artefakt |
|---|---|
| SLF4J → Logback | `logback-classic` (dostarcza binding) |
| SLF4J → Log4j2 | `log4j-slf4j2-impl` |
| SLF4J → brak/no-op | `slf4j-nop` (cisza) |

**Problem: wiele bindingów na classpath** → SLF4J 1.x wypisywał `Class path contains multiple SLF4J bindings` i wybierał losowy; SLF4J 2.x używa `ServiceLoader` i też ostrzega o wielu providerach. Skutek: podwójne logi albo brak logów. **Diagnoza:** `mvn dependency:tree`, wyklucz nadmiarowe bindingi (w Spring Boot `spring-boot-starter-log4j2` wymaga wykluczenia `spring-boot-starter-logging`).

**Bridge/adaptery** — przekierowują *inne* API logowania do SLF4J, żeby wszystko schodziło się w jednej implementacji:
- **`jul-to-slf4j`** — przechwytuje `java.util.logging` (JUL) i kieruje do SLF4J (wymaga `SLF4JBridgeHandler.install()`; ma narzut, bo JUL i tak tworzy `LogRecord`).
- **`log4j-over-slf4j`** — udaje API Log4j 1.x, ale zapisuje przez SLF4J (dla starych libów).
- **`jcl-over-slf4j`** — to samo dla Jakarta Commons Logging.

> **Pułapka: bridge kontra binding.** Nie wolno mieć jednocześnie `log4j-over-slf4j` (bridge Log4j→SLF4J) i `slf4j-log4j12`/`log4j-slf4j-impl` (binding SLF4J→Log4j) — powstaje **pętla** (`StackOverflowError`). Zasada: dla każdego frameworka albo *bridge do SLF4J*, albo *binding z SLF4J*, nigdy oba.

### Poziomy — TRACE / DEBUG / INFO / WARN / ERROR
Uporządkowane rosnąco wg wagi; ustawiony poziom X przepuszcza X i wyższe.

| Poziom | Kiedy | Prod domyślnie |
|---|---|---|
| **TRACE** | najdrobniejsze kroki (wejście/wyjście metody, wartości w pętli) | wyłączony |
| **DEBUG** | diagnostyka dla dewelopera (payloady, gałęzie decyzyjne) | wyłączony |
| **INFO** | istotne zdarzenia biznesowe (start aplikacji, „zamówienie X złożone") | włączony |
| **WARN** | coś podejrzanego, ale obsłużone (retry, deprecacja, fallback) | włączony |
| **ERROR** | operacja nieudana, wymaga uwagi (wyjątek zrywający request) | włączony |

Reguły: WARN ≠ „coś krzyknęło" tylko „przyjrzyj się"; ERROR to realny problem (alerty!). Nie loguj przewidywalnej walidacji użytkownika jako ERROR.

### Logowanie parametryczne — dlaczego lepsze niż konkatenacja
```java
log.debug("user " + id + " ma " + count + " zamówień");   // ŹLE
log.debug("user {} ma {} zamówień", id, count);           // DOBRZE
```
- **Konkatenacja `+`** buduje `String` (i wywołuje `toString()`) **zawsze**, *zanim* logger sprawdzi poziom → koszt nawet gdy `DEBUG` wyłączony.
- **Parametrycznie** SLF4J robi **leniwe formatowanie**: sprawdza poziom, i dopiero jeśli włączony, składa wiadomość przez `MessageFormatter` (podstawia `{}`). Gdy poziom wyłączony — **zero kosztu** formatowania i alokacji.
- Bonus: dedykowany argument `Throwable` na końcu drukuje stack trace: `log.error("nie udało się {}", op, ex);` (ostatni arg to wyjątek, nie `{}`).

**Guard `isDebugEnabled` dla DROGICH argumentów:** parametryzacja nie ratuje przed kosztem *wyliczenia* argumentu (serializacja, zapytanie).
```java
if (log.isDebugEnabled()) {
    log.debug("stan: {}", expensiveDump());   // expensiveDump() woła się tylko gdy DEBUG on
}
// alternatywa Java 8+: Supplier przez fluent API SLF4J 2.x
log.atDebug().addArgument(() -> expensiveDump()).log("stan: {}");
```

### MDC (Mapped Diagnostic Context) — kontekst per wątek
**MDC** to `ThreadLocal<Map<String,String>>` — wkładasz do niego klucze (np. `traceId`, `userId`), a **pattern** logu automatycznie je dokleja do każdej linii z tego wątku. Idealne do **correlation/trace id per żądanie**.

```java
import org.slf4j.MDC;

MDC.put("traceId", traceId);
try {
    log.info("obsługuję żądanie");   // pattern %X{traceId} dołoży id
} finally {
    MDC.clear();                     // KRYTYCZNE — patrz niżej
}
```

> **Pułapka wątków i thread poolów:** MDC siedzi w `ThreadLocal`. W puli wątki są **reużywane**, więc jeśli nie wyczyścisz (`MDC.clear()` / `MDC.remove(key)`), **wartość wycieknie do następnego żądania** obsłużonego przez ten sam wątek → obcy `traceId` w logach. Zawsze czyść w `finally`. Przy przekazywaniu pracy do innego wątku (`@Async`, `CompletableFuture`, executor) MDC **się nie propaguje** automatycznie — trzeba go przenieść (`MDC.getCopyOfContextMap()` + wrapper `Runnable`, albo `TaskDecorator` w Spring, albo biblioteka jak `mdc-context` / instrumentacja OTel).

### Logowanie strukturalne / JSON
Zamiast płaskiego tekstu — **JSON z polami** (`{"level":"INFO","traceId":"...","msg":"...","userId":42}`).
- **Po co w cloud:** agregatory (**ELK/Elasticsearch**, **Grafana Loki**, Splunk) **parsują pola**, indeksują i pozwalają filtrować/agregować (`userId:42 AND level:ERROR`) zamiast regexować tekst. MDC staje się osobnymi polami.
- Realizacja: `logstash-logback-encoder` (`LogstashEncoder`) albo `log4j2` `JsonTemplateLayout`.
- Łączy się z tracingiem — `traceId`/`spanId` w polach spinają logi z spanami. Zob. [[wiedza/10-cloud-native/obserwowalnosc]].

### Konfiguracja (Logback + Spring Boot)
Spring Boot czyta **`logback-spring.xml`** (nie `logback.xml`, bo `-spring` daje dostęp do `<springProfile>` i `<springProperty>`). Elementy:
- **Appender** — dokąd (Console, File, **RollingFile** — rotacja po czasie/rozmiarze), **AsyncAppender** — bufor + osobny wątek.
- **Pattern/encoder** — format linii (`%d %-5level [%thread] %logger{36} %X{traceId} - %msg%n`) lub `JsonEncoder`.
- **Poziomy per pakiet** — `<logger name="org.hibernate.SQL" level="DEBUG"/>`; root ustawia domyślny.
- **Profile** — inny appender/poziom per środowisko przez `<springProfile name="prod">`.

```xml
<configuration>
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} %-5level [%thread] %logger{36} traceId=%X{traceId} - %msg%n</pattern>
    </encoder>
  </appender>

  <springProfile name="prod">
    <appender name="JSON" class="ch.qos.logback.core.rolling.RollingFileAppender">
      <file>app.log</file>
      <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>app-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
        <maxFileSize>100MB</maxFileSize>
        <maxHistory>30</maxHistory>
      </rollingPolicy>
      <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>
    <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
      <appender-ref ref="JSON"/>
      <queueSize>8192</queueSize>
      <discardingThreshold>0</discardingThreshold>   <!-- 0 = nie gub nawet INFO -->
    </appender>
    <root level="INFO"><appender-ref ref="ASYNC"/></root>
  </springProfile>

  <springProfile name="dev">
    <root level="DEBUG"><appender-ref ref="CONSOLE"/></root>
  </springProfile>
</configuration>
```
(W `application.yml` można też sterować poziomami: `logging.level.org.springframework.web=DEBUG`.)

### Wydajność
- **AsyncAppender / async logger (Log4j2 Disruptor)** — I/O zapisu dzieje się w osobnym wątku, wątek aplikacji tylko wrzuca do kolejki → mniejsze opóźnienie na ścieżce żądania. Uwaga: przy pełnej kolejce domyślnie **gubi** logi poniżej WARN (`discardingThreshold`) — ustaw świadomie.
- **Koszt stack trace** — `new Throwable().getStackTrace()` / logowanie wyjątku jest **drogie** (chodzenie po stosie). Nie loguj wyjątków w gorącej pętli; unikaj rzucania wyjątków dla przepływu sterowania.
- **Koszt argumentów** — patrz guard `isXEnabled`.

## 3. Dlaczego / kiedy — praktyki, bezpieczeństwo, wydajność

**Dobre praktyki i pułapki:**
- **Nie loguj sekretów/PII/haseł** — tokeny, hasła, PESEL, numery kart, klucze API. To wyciek danych i naruszenie RODO/GDPR. Maskuj (`****1234`) albo w ogóle pomijaj. Uważaj na `toString()` całych obiektów (DTO z hasłem).
- **`log-and-throw` = antywzorzec:** `catch (E e){ log.error(...); throw e; }` na każdym poziomie → ten sam błąd zalogowany 5 razy (szum, false-positive w alertach). **Zasada:** albo obsłuż (i zaloguj), albo rzuć wyżej — **nie oba**. Loguj raz, na granicy (np. `@ControllerAdvice`).
- **Logowanie w pętli** — jedna linia na iterację przy milionie elementów zadławi I/O i zaśmieci. Agreguj (loguj podsumowanie/co N-ty/sample).
- **Właściwy poziom** — nie INFO dla debug-spamu, nie ERROR dla przewidywalnej walidacji.
- **Nie łap i nie loguj bez kontekstu** — `catch(Exception e){ log.error(e.getMessage()); }` gubi stack trace (przekaż `e` jako ostatni arg, nie `e.getMessage()`), a `catch(...){}` (połknięcie) jest jeszcze gorsze. Dodaj kontekst biznesowy (id, operacja).
- **Log injection** — nie wstawiaj surowego inputu użytkownika bez sanityzacji (znaki nowej linii mogą sfałszować wpisy). Strukturalny JSON to ogranicza.

**Korelacja logów z tracingiem (OpenTelemetry):**
W systemie rozproszonym pojedyncze żądanie przechodzi przez wiele serwisów. **OpenTelemetry** nadaje `traceId`/`spanId` i propaguje je nagłówkami (W3C `traceparent`). Instrumentacja (np. **Micrometer Tracing** w Spring Boot 3, agent OTel) **automatycznie wkłada `traceId`/`spanId` do MDC**, więc pattern/JSON logu je zawiera. Dzięki temu w Grafanie/Kibanie klikasz ze spana wprost w logi tego żądania (i odwrotnie). To spina trzy filary observability — logi, metryki, trace. Zob. [[wiedza/10-cloud-native/obserwowalnosc]].

**Kiedy Logback, kiedy Log4j2:** domyślnie **Logback** (Spring Boot, mniej zależności, wystarcza). **Log4j2** przy bardzo wysokim wolumenie logów, gdzie liczą się async logger na Disruptorze i garbage-free mode.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Serwis REST na Spring Boot 3 / Java 21. Wpinasz **filtr**, który dla każdego żądania ustawia correlation id w MDC, a globalny handler loguje błędy **raz**:

```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String cid = Optional.ofNullable(req.getHeader("X-Correlation-Id"))
                             .orElse(UUID.randomUUID().toString());
        MDC.put("traceId", cid);            // dostępne w każdym logu tego wątku
        res.setHeader("X-Correlation-Id", cid);
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.clear();                    // sprzątanie — wątek wraca do puli Tomcata
        }
    }
}

@RestControllerAdvice
class GlobalErrorHandler {
    private static final Logger log = LoggerFactory.getLogger(GlobalErrorHandler.class);
    @ExceptionHandler(Exception.class)
    ResponseEntity<?> handle(Exception e) {
        log.error("nieobsłużony błąd requestu", e);   // raz, z pełnym stack trace + traceId z MDC
        return ResponseEntity.status(500).body(Map.of("error", "internal"));
    }
}
```
```java
// Poprawne vs złe logowanie
log.info("user {} złożył zamówienie {}", userId, orderId);       // OK: parametrycznie, INFO
log.error("płatność {} nieudana", paymentId, ex);                // OK: kontekst + wyjątek

log.info("user " + user);                                        // ŹLE: konkatenacja + PII w toString()
log.error(e.getMessage());                                       // ŹLE: brak stack trace i kontekstu
catch (Exception e) { log.error("błąd", e); throw e; }           // ŹLE: log-and-throw
log.info("hasło={}", password);                                  // ŹLE: sekret w logu
```

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, czemu `System.out.println` to nie logowanie.
- [ ] Rozróżniam fasadę SLF4J od implementacji i wiem, po co warstwa fasady.
- [ ] Rozumiem bindingi i bridge (jul-to-slf4j, log4j-over-slf4j) oraz problem wielu implementacji/pętli.
- [ ] Dobiorę poziom (TRACE→ERROR) do sytuacji.
- [ ] Umiem uzasadnić logowanie parametryczne i guard `isDebugEnabled`.
- [ ] Skonfiguruję MDC w filtrze i wiem, czemu trzeba go czyścić w thread poolu.
- [ ] Wiem, po co JSON/strukturalne logi w chmurze i jak spina się z tracingiem.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Jak SLF4J wybiera implementację w runtime (binding / ServiceLoader)?
- [ ] Co dokładnie oszczędza logowanie parametryczne przy wyłączonym poziomie?
- [ ] Na czym technicznie stoi MDC i skąd bierze się wyciek w puli wątków?
- [ ] Czemu `log4j-over-slf4j` + binding SLF4J→Log4j daje `StackOverflowError`?
- [ ] Jak `traceId` z OpenTelemetry trafia do linii logu?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Po co SLF4J skoro jest Logback?"
- [ ] „Czemu `log.info("x=" + x)` jest gorsze niż `log.info("x={}", x)`?"
- [ ] „Jak zrealizujesz correlation id per żądanie?" (MDC + filtr + czyszczenie)
- [ ] „Co to log-and-throw i czemu to antywzorzec?"
- [ ] „Logback vs Log4j2 — kiedy co?"
- [ ] „Jak korelujesz logi z tracingiem w systemie rozproszonym?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/04-narzedzia.csv`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Czym jest SLF4J?;Fasadą logowania (Simple Logging Facade for Java) — jedno API/interfejsy, bez zapisu; deleguje do implementacji wybranej po classpath.
Po co warstwa fasady logowania?;Wymienialność implementacji bez zmiany kodu, jedno API dla całego drzewa zależności, parametryzacja i MDC w kontrakcie.
Jaka implementacja logowania jest domyślna w Spring Boot?;Logback (starter spring-boot-starter-logging).
Czym wyróżnia się Log4j2?;Wydajnością — async loggery na LMAX Disruptor (lock-free) i garbage-free mode; dobry przy dużym wolumenie logów.
Co to binding SLF4J?;Artefakt łączący SLF4J z konkretną implementacją (np. logback-classic, log4j-slf4j2-impl); potrzebny dokładnie jeden.
Do czego służy jul-to-slf4j / log4j-over-slf4j?;To bridge — przekierowują logi z java.util.logging / Log4j 1.x do SLF4J, by wszystko schodziło się w jednej implementacji.
Czemu nie wolno mieć bridge i bindingu tego samego frameworka naraz?;Powstaje pętla przekierowań (np. log4j-over-slf4j + binding SLF4J→Log4j) → StackOverflowError.
Wymień poziomy logowania od najniższego.;TRACE, DEBUG, INFO, WARN, ERROR (ustawiony poziom przepuszcza siebie i wyższe).
Kiedy WARN, a kiedy ERROR?;WARN = podejrzane, ale obsłużone (retry, fallback); ERROR = operacja nieudana wymagająca uwagi/alertu.
Czemu logowanie parametryczne bije konkatenację?;Leniwe formatowanie — SLF4J najpierw sprawdza poziom i dopiero potem składa wiadomość; przy wyłączonym poziomie zero kosztu i alokacji.
Kiedy stosować guard isDebugEnabled?;Gdy sam argument jest drogi do wyliczenia (serializacja, zapytanie) — parametryzacja nie chroni przed kosztem policzenia argumentu.
Czym jest MDC?;Mapped Diagnostic Context — ThreadLocal Map String→String; wkładasz np. traceId/userId, a pattern logu dokleja je do linii z tego wątku.
Główna pułapka MDC w thread poolu?;Wątki są reużywane — bez MDC.clear()/remove wartość wycieka do następnego żądania na tym samym wątku; czyść w finally.
Po co logi strukturalne (JSON) w chmurze?;Agregatory (ELK/Loki) parsują i indeksują pola, umożliwiając filtrowanie/agregację zamiast regexowania tekstu; MDC staje się polami.
Który plik konfiguracyjny Logback w Spring Boot i czemu?;logback-spring.xml — daje springProfile i springProperty (profile/właściwości Springa); logback.xml ich nie ma.
Co robi AsyncAppender i jego ryzyko?;Zapis I/O w osobnym wątku (wątek aplikacji tylko kolejkuje); ryzyko: przy pełnej kolejce gubi logi poniżej progu (discardingThreshold).
Czemu log-and-throw to antywzorzec?;Ten sam błąd logowany wielokrotnie na każdym poziomie → szum i fałszywe alerty; loguj raz na granicy albo obsłuż, nie oba.
Jak traceId z OpenTelemetry trafia do logów?;Instrumentacja (Micrometer Tracing / agent OTel) wkłada traceId/spanId do MDC, więc pattern lub encoder JSON automatycznie je dołącza.
Czemu System.out.println to nie logowanie?;Brak poziomów, kontekstu, appenderów/rotacji i routingu; synchroniczny zapis pod globalnym lockiem dławi throughput.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/10-cloud-native/obserwowalnosc]] · [[wiedza/04-narzedzia/spring-boot-config]] · [[wiedza/00-fundament/jdk-jre-jvm]]
