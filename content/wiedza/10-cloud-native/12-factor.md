---
temat: "The Twelve-Factor App"
faza: 10
status: nieopanowany
priorytet: 🟡
tags: [java, cloud, cloud-native]
powiazane: ["[[wiedza/10-cloud-native/kubernetes]]", "[[wiedza/10-cloud-native/cicd]]", "[[wiedza/05-spring/konfiguracja-profile]]", "[[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]]"]
---

# The Twelve-Factor App

> **TL;DR:** **12-factor** to metodologia (autorzy Heroku) budowy aplikacji **SaaS / cloud-native**: 12 zasad,
> które czynią appkę **przenośną**, **skalowalną poziomo** i gotową na **ciągłe wdrażanie**. Sedno: aplikacja
> **bezstanowa**, konfiguracja w **środowisku**, logi na **stdout**, sama wystawia **port**. To fundament pod
> kontenery i Kubernetes — Spring Boot 3 domyślnie prowadzi Cię ku 12-factor.

## 1. Co — definicja i po co

**The Twelve-Factor App** (Adam Wiggins, Heroku, 2011) to zestaw 12 reguł opisujących, jak pisać usługi, które:
- **działają jako usługa** (long-running, wiele identycznych wdrożeń),
- da się **przenieść** między środowiskami/dostawcami bez zmian kodu,
- **skalują się poziomo** (dokładanie replik, nie „większy serwer"),
- nadają się do **continuous deployment** (dev/prod parity, automatyzacja).

Nie jest to specyfikacja ani framework — to **zbiór ograniczeń projektowych**. Aplikacja spełniająca 12 czynników
jest naturalnie „**container-friendly**": łatwo ją zapakować w obraz OCI i uruchomić na
[[wiedza/10-cloud-native/kubernetes|Kubernetes]] jako `Deployment` z wieloma replikami. Spring Boot 3 (embedded server,
externalized config, actuator, structured logging) daje większość tego „z pudełka".

```
Klasyczna appka „na serwerze"          12-factor / cloud-native
───────────────────────────────        ───────────────────────────────
stan w sesji HTTP w pamięci        →    stan w bazie/Redis, proces bezstanowy
config w application.properties     →    config w zmiennych środowiskowych
logi do plików + logrotate          →    logi na stdout, agregator zbiera
WAR wrzucany do zewn. Tomcata       →    JAR z embedded Tomcatem, sam słucha portu
pionowe skalowanie (RAM/CPU)        →    poziome (N identycznych replik + LB)
```

## 2. Jak — 12 czynników w praktyce Spring Boot

**I. Codebase** — jedna baza kodu w VCS (Git), **wiele wdrożeń** (dev/staging/prod) z tej samej bazy.
Jeden `pom.xml` → jeden artefakt; różnica między środowiskami to **config**, nie osobny branch/kod. Współdzielony
kod → biblioteka (dependency), nie kopia repo.

**II. Dependencies** — zależności **jawne i izolowane**, nigdy „system-wide". W Javie robi to
[[wiedza/00-fundament/maven|Maven]]: `pom.xml` deklaruje wszystko, a `spring-boot-maven-plugin` buduje **fat JAR**
z zaszytymi bibliotekami. Żadnych ukrytych JAR-ów w `$CLASSPATH` maszyny. Wersje przypięte (Spring Boot BOM).

**III. Config** — konfiguracja **w środowisku**, nie w kodzie. Wszystko, co różni się między wdrożeniami
(URL bazy, hasła, klucze) → zmienne środowiskowe. Spring Boot mapuje `SPRING_DATASOURCE_URL` na
`spring.datasource.url` (relaxed binding). Profile (`@Profile`, `application-prod.yml`) porządkują, ale
**sekrety nie do repo** → env / Vault / K8s Secrets. Szczegóły: [[wiedza/05-spring/konfiguracja-profile]].

**IV. Backing services** — baza, kolejka, cache, SMTP traktuj jako **podłączalne zasoby** identyfikowane przez
**URL/credentials** w configu. Wymiana lokalnego Postgresa na RDS = zmiana `SPRING_DATASOURCE_URL`, zero zmian
w kodzie. `DataSource`, `RabbitTemplate`, `RedisConnectionFactory` konfigurowane wyłącznie z env.

**V. Build, release, run** — **trzy rozdzielone etapy**: *build* (kod → artefakt: `mvn package` → JAR → obraz),
*release* (artefakt + config = niezmienny, wersjonowany release), *run* (uruchomienie release'u). Release jest
**immutable** — nie edytujesz kodu na produkcji; nowa zmiana = nowy build+release. To realizuje pipeline
[[wiedza/10-cloud-native/cicd|CI/CD]] (rollback = uruchom poprzedni release).

**VI. Processes** — aplikacja to **jeden lub więcej BEZSTANOWYCH procesów**. Stan trwały **w bazie/cache**
([[wiedza/10-cloud-native/kubernetes|zewnętrzny]] backing service), **NIE w pamięci JVM ani w sesji HTTP**.
To **klucz do skalowania i load-balancingu**: dowolna replika obsłuży dowolne żądanie, bo nie ma „sticky"
stanu. Dlatego uwierzytelnianie przez [[wiedza/08-bezpieczenstwo/jwt-oauth-oidc|JWT]] (token niesie stan,
weryfikowalny lokalnie) zamiast `HttpSession` w pamięci. Cache lokalny = optymalizacja, nie źródło prawdy.

**VII. Port binding** — aplikacja **sama eksportuje usługę przez port**, jest samowystarczalna. Spring Boot ma
**embedded Tomcat/Netty** — `java -jar app.jar` startuje serwer na `server.port`. Żadnego zewnętrznego kontenera
serwletów (nie deployujesz WAR do Tomcata) — to Twój proces *jest* serwerem HTTP.

**VIII. Concurrency** — skaluj przez **procesy** (model procesów), poziomo: więcej **replik**, nie większa maszyna.
Różne typy pracy → różne typy procesów (web vs worker/consumer kolejki). W Kubernetes:
`kubectl scale --replicas=N` lub HPA (Horizontal Pod Autoscaler) — [[wiedza/10-cloud-native/kubernetes]]. Możliwe
tylko dzięki bezstanowości (czynnik VI).

**IX. Disposability** — procesy **jednorazowe**: **szybki start** i **graceful shutdown**, odporne na nagłe ubicie
(SIGKILL). Spring Boot: `server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase` — kończy
in-flight requesty po SIGTERM. Praca robocza **idempotentna / z ACK po zakończeniu**, by ubita replika nie
zgubiła zadania. Szybki start ważny dla autoscalingu i częstych deployów.

**X. Dev/prod parity** — **minimalizuj różnice** dev↔prod: te same backing services, te same wersje.
Nie „H2 lokalnie, Postgres na prod" — **[[wiedza/04-narzedzia/testcontainers|Testcontainers]]** uruchamia
prawdziwego Postgresa/Kafkę w kontenerze podczas testów, więc dev i CI grają na tym samym silniku co prod.
Krótki dystans czasowy (deployuj często) i osobowy (autorzy wdrażają).

**XI. Logs** — logi to **strumień zdarzeń** pisany na **stdout/stderr**; aplikacja **nie zarządza plikami** ani
rotacją. Środowisko (Docker/K8s/agregator: Loki, ELK) przechwytuje strumień i archiwizuje. Spring Boot domyślnie
loguje na konsolę; na prod włącz **JSON** (structured logging w Boot 3.4+ lub `logstash-logback-encoder`) dla
maszynowego parsowania. Szczegóły: [[wiedza/04-narzedzia/logowanie]].

**XII. Admin processes** — zadania jednorazowe/administracyjne (migracje DB, skrypty naprawcze, dane inicjalne)
uruchamiaj jako **procesy jednorazowe** w **tym samym release/środowisku** co appka. Migracje: Flyway/Liquibase
przy starcie lub jako osobny `Job` w K8s. Nie „ręczny SQL na produkcji" — to samo repo, ten sam config.

**Beyond 12-factor** (Kevin Hoffman) — nowoczesne rozszerzenia, warto mieć świadomość:
- **API first** — kontrakt (OpenAPI) przed implementacją;
- **Telemetry / Observability** — metryki, logi, **distributed tracing** (Spring Boot Actuator + Micrometer +
  OpenTelemetry) jako pierwszorzędny wymiar, nie dodatek;
- **Authentication & Authorization** — tożsamość i uprawnienia od początku (OAuth2/OIDC, mTLS), nie „na później".

## 3. Dlaczego / kiedy — bezstanowość, skalowanie, pułapki

- **Bezstanowość to fundament** — czynnik VI umożliwia VIII (concurrency) i IX (disposability). Jeśli replika
  trzyma stan w pamięci/sesji, load-balancer musi mieć **sticky sessions**, a autoscaling/rollout gubi dane
  użytkowników. Stąd JWT zamiast serwerowej sesji, Redis na współdzielony cache/rate-limit.
- **Dlaczego to fundament pod kontenery/K8s** — kontener jest efemeryczny i wielokrotny; K8s może ubić i przenieść
  Pod w każdej chwili, wstrzykuje config przez env/Secrets, zbiera logi ze stdout, skaluje repliki. Appka
  **nie-12-factor** (pliki lokalne, stan w pamięci, config w JAR-ze) w tym modelu się „wykrwawia".
- **Pułapki:**
  - **Stan w `HttpSession`** — psuje skalowanie; najczęstszy grzech migracji legacy.
  - **Config w `application.properties` z sekretami w repo** — wyciek + brak przenośności. Env/Vault.
  - **Logowanie do pliku** w kontenerze — ginie przy restarcie, brak agregacji.
  - **Brak graceful shutdown** — zrywane requesty przy rolling update.
  - **Zapis do lokalnego dysku kontenera** — znika przy restarcie; użyj object storage/DB.
  - **Dev na H2, prod na Postgres** — „u mnie działa"; łap różnice Testcontainerami.
- **Kiedy NIE dogmatycznie** — 12-factor celuje w **stateless SaaS**. Stateful systemy (bazy, brokery, gry
  z sesją real-time) łamią część reguł świadomie — dlatego istnieją `StatefulSet`, PersistentVolume. Znaj zasady,
  by wiedzieć, kiedy je łamiesz.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)
Mikroserwis REST na Spring Boot 3 / Java 21, wdrażany na Kubernetes: `mvn package` buduje fat JAR (II, V),
Jib/buildpacks pakuje w obraz (V), pipeline CI/CD publikuje niezmienny release (V). Pod czyta `SPRING_DATASOURCE_URL`
i sekrety z K8s Secret (III), łączy się z zarządzanym Postgresem po URL (IV), słucha na `8080` embedded Tomcatem (VII).
Autentykacja przez JWT — proces bezstanowy (VI), więc HPA skaluje do N replik za load-balancerem (VIII). SIGTERM →
graceful shutdown dokańcza requesty (IX). Logi w JSON na stdout zbiera Loki (XI), Actuator + Micrometer + OTel dają
observability (beyond 12-factor). Migracje Flyway przy starcie (XII). Testy z Testcontainers = dev/prod parity (X).

---

## ✅ Kryteria opanowania
- [ ] Wymienię wszystkie 12 czynników i jednym zdaniem powiem, co znaczą w Spring Boot.
- [ ] Wyjaśnię, dlaczego **bezstanowość** (VI) jest kluczem do skalowania i disposability.
- [ ] Uzasadnię JWT zamiast `HttpSession` w kontekście load-balancingu.
- [ ] Wiem, czemu config idzie do **środowiska**, a logi na **stdout**.
- [ ] Rozumiem, dlaczego 12-factor jest fundamentem pod kontenery/Kubernetes.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Czym różnią się etapy **build / release / run** i czemu release jest immutable?
- [ ] Co konkretnie robi embedded Tomcat wobec czynnika **Port binding**?
- [ ] Jak K8s realizuje czynniki III, VIII, IX, XI (Secrets, HPA, SIGTERM, stdout)?
- [ ] Czemu „dev na H2, prod na Postgres" łamie **dev/prod parity** i jak ratują Testcontainers?
- [ ] Co to „beyond 12-factor" i które rozszerzenia są najważniejsze?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Co to 12-factor app i po co powstało?"
- [ ] „Dlaczego aplikacja cloud-native musi być bezstanowa?"
- [ ] „Gdzie trzymasz konfigurację i sekrety, a gdzie NIE?"
- [ ] „Jak Spring Boot wspiera 12-factor out of the box?"
- [ ] „Czemu logujesz na stdout zamiast do pliku?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Czym jest The Twelve-Factor App?;Metodologia (Heroku) budowy aplikacji SaaS/cloud-native — 12 zasad dla przenośności, skalowalności i continuous deployment.
Czynnik I (Codebase)?;Jedna baza kodu w VCS, wiele wdrożeń z tej samej bazy; różni je config, nie kod.
Czynnik II (Dependencies)?;Zależności jawne i izolowane — w Javie Maven/pom.xml + fat JAR ze spring-boot-maven-plugin, żadnych system-wide.
Czynnik III (Config)?;Konfiguracja w środowisku (zmienne env), nie w kodzie; sekrety poza repo (env/Vault/K8s Secrets).
Czynnik IV (Backing services)?;Baza/kolejka/cache jako podłączalne zasoby przez URL w configu — wymiana bez zmiany kodu.
Czynnik V (Build/release/run)?;Trzy rozdzielone etapy; release = artefakt + config, immutable i wersjonowany; realizuje CI/CD.
Czynnik VI (Processes)?;Procesy BEZSTANOWE — stan w bazie/cache, nie w pamięci/sesji; podstawa skalowania i load-balancingu.
Czynnik VII (Port binding)?;Aplikacja sama wystawia usługę przez port — embedded Tomcat w Spring Boot, bez zewn. serwera.
Czynnik VIII (Concurrency)?;Skalowanie poziome przez procesy/repliki (nie większa maszyna); w K8s HPA / kubectl scale.
Czynnik IX (Disposability)?;Szybki start i graceful shutdown, odporność na nagłe ubicie; w Boot server.shutdown=graceful.
Czynnik X (Dev/prod parity)?;Minimalne różnice dev↔prod — te same backing services; pomaga Testcontainers.
Czynnik XI (Logs)?;Logi jako strumień zdarzeń na stdout/stderr; aplikacja nie zarządza plikami — agregator przechwytuje.
Czynnik XII (Admin processes)?;Zadania jednorazowe (migracje, skrypty) jako procesy w tym samym release/środowisku; np. Flyway.
Dlaczego JWT zamiast HttpSession w cloud-native?;Bo proces ma być bezstanowy — token niesie stan weryfikowalny lokalnie, więc każda replika obsłuży żądanie bez sticky sessions.
Dlaczego 12-factor to fundament pod Kubernetes?;Kontener jest efemeryczny i wielokrotny — K8s wstrzykuje config przez env/Secrets, zbiera logi ze stdout, ubija/skaluje repliki; wymaga bezstanowości.
Co to "beyond 12-factor"?;Rozszerzenia (Kevin Hoffman): m.in. API first, telemetry/observability, authentication/authorization jako pierwszorzędne wymiary.
Dlaczego release ma być immutable?;By nie edytować kodu na produkcji — każda zmiana to nowy build+release, co daje powtarzalność i łatwy rollback.
Jak Spring Boot wspiera port binding?;Ma wbudowany serwer (Tomcat/Netty) — java -jar startuje HTTP na server.port, bez zewnętrznego kontenera serwletów.
Co grozi za trzymanie stanu w HttpSession w pamięci?;Wymusza sticky sessions na load-balancerze, a autoscaling/rolling update gubi dane użytkowników.
Gdzie w 12-factor trafiają sekrety?;Do środowiska (zmienne env / Vault / K8s Secrets), NIGDY do repozytorium/JAR-a.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/10-cloud-native/kubernetes]] · [[wiedza/10-cloud-native/cicd]] · [[wiedza/00-fundament/maven]] · [[wiedza/05-spring/konfiguracja-profile]] · [[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]] · [[wiedza/04-narzedzia/testcontainers]] · [[wiedza/04-narzedzia/logowanie]]
