---
temat: "Konfiguracja zewnętrzna (externalized config) i profile w Spring Boot"
faza: 5
status: nieopanowany
priorytet: 🔴
tags: [java, spring, konfiguracja]
powiazane: ["[[wiedza/10-cloud-native/12-factor]]", "[[wiedza/08-bezpieczenstwo/owasp]]", "[[wiedza/05-spring/ioc-di]]"]
---

# Konfiguracja zewnętrzna (externalized config) i profile w Spring Boot

> **TL;DR:** **Jeden zbudowany artefakt (JAR/obraz) uruchamiasz w każdym środowisku bez rebuildu** — różnice
> (URL bazy, hasła, timeouty) wstrzykujesz z zewnątrz. Spring Boot łączy **wiele źródeł** wg ustalonego
> **priorytetu** (CLI > env vars > `application-{profile}.yml` > `application.yml` > defaulty w kodzie).
> **Profile** (`spring.profiles.active=prod`) przełączają zestawy właściwości i beanów. Sekrety **nigdy w repo**
> — env / Vault / k8s Secrets. To realizacja zasady **config** z [[wiedza/10-cloud-native/12-factor|12-factor]].

## 1. Co — externalized config i po co

**Externalized configuration** = konfiguracja żyje **poza** kodem (i poza artefaktem), więc ten sam bytecode
zachowuje się różnie zależnie od środowiska. To bezpośrednio zasada **III (Config)** metodyki
[[wiedza/10-cloud-native/12-factor|12-factor app]]: *„store config in the environment"*, **strict separation of
config from code**. Konsekwencja: **build once, deploy anywhere** — obraz z CI leci przez `dev → staging → prod`
bez rekompilacji; zmienia się tylko wstrzykiwana konfiguracja.

Konfiguracja to wszystko, co różni się **między deploymentami**: connection string do bazy, adresy usług,
credentiale, feature flagi, poziomy logowania, rozmiary pul wątków. Anty-wzorzec: `if (env == "prod")` w kodzie
albo osobny build per środowisko (dwa artefakty = dwa różne, nieprzetestowane byty).

W Spring Boot punktem wejścia jest abstrakcja **`Environment`** — trzyma uporządkowaną listę
**`PropertySource`** (każde źródło to nazwana mapa klucz→wartość). Odczyt: `env.getProperty("my.app.url")`
przechodzi po źródłach **w kolejności priorytetu** i zwraca pierwsze trafienie.

Minimalny przykład — property w `application.yml` i wstrzyknięcie:
```yaml
# src/main/resources/application.yml
my:
  app:
    url: http://localhost:8080
    timeout: 5s
```
```java
@Value("${my.app.url}") String url;   // podstawienie w czasie tworzenia beana
```

## 2. Jak — hierarchia źródeł i binding pod spodem

### Priorytet źródeł (od NAJWYŻSZEGO)
Gdy ten sam klucz jest w wielu miejscach, **wygrywa źródło wyżej** (nadpisuje niższe). Praktyczna hierarchia
Spring Boot 3 (uproszczona, ale w kolejności, którą warto znać na rekrutacji):

```
1. Argumenty CLI                     --my.app.url=... (args do main)
2. Zmienne środowiskowe (env vars)   MY_APP_URL=...  (relaxed binding!)
3. application-{profile}.properties/yml  (profil aktywny nadpisuje bazowe)
4. application.properties/yml            (bazowe, spakowane w JAR lub obok)
5. @PropertySource na @Configuration
6. Defaulty w kodzie                 @Value("${x:DEFAULT}"), SpringApplication.setDefaultProperties
```
> **Reguła kciuka:** *im „bliżej" runtime i im bardziej zewnętrzne źródło, tym wyższy priorytet.* Dlatego
> zmienną środowiskową / argument CLI **zawsze** możesz nadpisać wartość spakowaną w JAR — bez rebuildu.
> (Pełna lista Spring ma ~17 poziomów: m.in. `SPRING_APPLICATION_JSON`, JNDI, `RandomValuePropertySource`,
> config z `spring.config.import`, test properties `@TestPropertySource` — ale ta piątka to rdzeń.)

**Ważne o profilach:** pliki `application.yml` i `application-{profile}.yml` **nie zastępują się** —
są **łączone (merge)**, a wartości z profilu nadpisują bazowe. Czyli w bazowym trzymasz defaulty, w
`application-prod.yml` tylko **różnice**.

### `application.properties` vs `application.yml`
Ten sam model danych, dwie składnie. **YAML** — hierarchiczny, czytelny dla zagnieżdżeń i list, jeden plik może
zawierać wiele dokumentów per profil (`---` + `spring.config.activate.on-profile`). **`.properties`** — płaski
`klucz=wartość`, brak zagnieżdżeń i typów. Jeśli **oba** są na classpath, `.properties` ma pierwszeństwo nad
`.yml`. Uwaga na YAML: `no`, `off`, `yes` bywają parsowane jako boolean; typuj jawnie (`"no"` w cudzysłowie).

### `@Value` vs `@ConfigurationProperties`
```java
// @Value — pojedyncze wartości, SpEL, brak type-safety na poziomie grupy
@Value("${my.app.timeout:5s}")   // ':' = default gdy brak klucza
private Duration timeout;
```
- **`@Value`** — dobre do 1-2 luźnych wartości. Wady: string-owe klucze rozsiane po kodzie, brak grupowania,
  **brak wsparcia relaxed binding dla nazw**, trudna walidacja, literówka = błąd dopiero w runtime.

```java
// @ConfigurationProperties — type-safe binding CAŁEGO zestawu
@ConfigurationProperties(prefix = "my.app")
@Validated
public record AppProps(
        @NotBlank String url,
        @DefaultValue("5s") Duration timeout,
        List<String> allowedOrigins) {}
```
- **`@ConfigurationProperties`** — **preferowany dla zestawów właściwości**:
  - **type-safe binding** — Spring mapuje właściwości na pola typu POJO/`record` (konwersje: `Duration`,
    `DataSize`, enum, `List`, `Map`, zagnieżdżone obiekty).
  - **relaxed binding** — jedna właściwość, wiele zapisów: `my.app.allowedOrigins`, `my.app.allowed-origins`
    (kebab), `MY_APP_ALLOWED_ORIGINS` (env), `MY_APP_ALLOWEDORIGINS` — wszystkie trafią w to samo pole.
    **Kanoniczny zapis w plikach: kebab-case** (`allowed-origins`).
  - **walidacja** — `@Validated` + adnotacje Bean Validation (`@NotBlank`, `@Min`) → **fail-fast** przy starcie,
    nie w środku żądania.
  - **grupowanie + IDE support** — jeden obiekt zamiast rozsypanych `@Value`; z zależnością
    `spring-boot-configuration-processor` IDE podpowiada klucze i pokazuje docsy w `application.yml`.

### Rejestracja `@ConfigurationProperties`
Sam `@ConfigurationProperties` na klasie **nie tworzy beana** — trzeba go zarejestrować:
- **`@EnableConfigurationProperties(AppProps.class)`** — jawnie wskazujesz klasy (na `@Configuration`).
- **`@ConfigurationPropertiesScan`** — skanuje pakiety i rejestruje wszystkie `@ConfigurationProperties`
  (analogia do component scan). Wtedy klasa **nie** potrzebuje `@Component`.
- Wariant „bean bezpośrednio": `@Component @ConfigurationProperties(...)` (mutowalny POJO z setterami).
  Dla `record` (immutable, constructor binding) użyj `@EnableConfigurationProperties`/`@ConfigurationPropertiesScan`.

### Relaxed binding i zmienne środowiskowe
Systemy operacyjne pozwalają tylko na `[A-Z0-9_]` w nazwach env. Reguła: `my.app.url` (property) ↔
**`MY_APP_URL`** (env): kropki i myślniki → `_`, wszystko UPPERCASE. `MY_APP_URL → my.app.url`. Dzięki temu
w k8s/Dockerze podajesz env-y, a Spring i tak trafia w pole POJO.

## 3. Dlaczego / kiedy — środowiska, sekrety, profile, pułapki

### Profile — przełączanie zestawów
```java
@Bean
@Profile("dev")           // bean tylko gdy profil 'dev' aktywny
DataSource h2() { ... }

@Bean
@Profile("prod")
DataSource postgres() { ... }
```
- **`@Profile`** na beanie/`@Configuration` → warunkowa rejestracja.
- Aktywacja: `spring.profiles.active=prod` (property), `--spring.profiles.active=prod` (CLI),
  `SPRING_PROFILES_ACTIVE=prod` (env — najczęściej w k8s), lub w `application.yml`.
- **Pliki per profil:** `application-dev.yml`, `application-prod.yml` — ładowane, gdy dany profil aktywny.
- **Profile groups** — jedna nazwa włącza kilka profili:
  ```yaml
  spring:
    profiles:
      group:
        prod: [prod, db-postgres, metrics]   # active=prod → włącza całą trójkę
  ```
- **`@Profile("!prod")`** = wszystko poza prod; można łączyć wyrażeniami (`dev & !ci`).
- Anty-wzorzec: profil `default` z „prawie produkcyjnymi" ustawieniami → łatwo o wyciek. Trzymaj bazowe defaulty
  bezpieczne/lokalne, a prod jawnie włączaj profilem.

### Sekrety — NIGDY w repozytorium
Hasła, klucze API, tokeny **nie mogą** trafić do `application.yml` w gitcie (to podatność z
[[wiedza/08-bezpieczenstwo/owasp|OWASP]] — *Security Misconfiguration* / *Cryptographic Failures*; leak w historii
git jest praktycznie nieusuwalny). Zamiast tego:
- **zmienne środowiskowe** wstrzykiwane przez orkiestrator,
- **HashiCorp Vault** (dynamiczne sekrety, rotacja; `spring-cloud-vault`),
- **Kubernetes Secrets** montowane jako **env** lub **plik** (i szyfrowane at-rest / przez KMS),
- lokalnie: plik poza repo (`.gitignore`) lub `spring.config.import=optional:file:./secrets.properties`.

W repo trzymasz **tylko** nie-sekretne defaulty i **placeholdery** (`spring.datasource.password=${DB_PASSWORD}`).

### Config w kontenerze / Kubernetes
- **ConfigMap** → wstrzykiwany jako **env vars** (relaxed binding robi resztę) albo **zamontowany plik**
  (`application.yml` w `/config`, który Spring Boot ładuje automatycznie z `./config/`).
- **Secret** → jak wyżej, ale dla wrażliwych danych.
- Wzorzec: obraz jest **immutable**, ConfigMap/Secret to warstwa konfiguracji dopinana przy starcie poda.

### Spring Cloud Config (świadomość)
Przy wielu mikroserwisach zamiast N razy zarządzać plikami — **scentralizowana konfiguracja**:
**Spring Cloud Config Server** serwuje properties (backend: git/Vault), klienci pobierają je przy starcie
(`spring.config.import=configserver:...`). Zalety: jedno źródło prawdy, audyt (git), odświeżanie przez
`@RefreshScope` + `/actuator/refresh` lub Spring Cloud Bus. Alternatywy: Consul, k8s ConfigMap + reloader.

### Importowanie configu i `@PropertySource`
- **`@PropertySource("classpath:custom.properties")`** — na `@Configuration`, dokłada źródło (uwaga: domyślnie
  **nie** czyta YAML, tylko `.properties`; niski priorytet — pkt 5 hierarchii).
- **`spring.config.import`** (Boot 2.4+, nowoczesne podejście) — deklaratywne dołączanie configu **z `application.yml`**,
  z obsługą `optional:` i wielu backendów:
  ```yaml
  spring.config.import: "optional:file:./local.yml,configserver:,vault:"
  ```

### Częste pułapki
- `@Value` z literówką w kluczu → brak beana/`null` w runtime, nie przy starcie.
- Zapomniany `@EnableConfigurationProperties`/`@ConfigurationPropertiesScan` → NPE, bo brak beana propsów.
- YAML: złe wcięcia lub `off`/`no` jako boolean.
- Nadpisywanie env-em przy złym mapowaniu nazwy (kebab vs env) — pamiętaj `MY_APP_X`.
- Sekret w `application.yml` „tylko na chwilę" → i tak zostaje w historii gita.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)
Serwis płatności: `record @ConfigurationProperties` + profile `dev`/`prod` + nadpisanie env-em na produkcji.

```java
// PaymentProperties.java — type-safe, immutable, walidowany zestaw właściwości
@ConfigurationProperties(prefix = "payment")
@Validated
public record PaymentProperties(
        @NotBlank String apiUrl,
        @NotBlank String apiKey,
        @DefaultValue("3s") Duration timeout,
        @Min(1) int maxRetries) {}

@SpringBootApplication
@ConfigurationPropertiesScan          // rejestruje PaymentProperties jako bean
public class App { public static void main(String[] a){ SpringApplication.run(App.class,a);} }

@Service
class PaymentService {
    private final PaymentProperties props;
    PaymentService(PaymentProperties props) { this.props = props; }   // wstrzyknięty type-safe config
}
```
```yaml
# application.yml — bazowe, bezpieczne defaulty (BEZ sekretu)
payment:
  api-url: http://localhost:9000/mock   # relaxed binding: api-url ↔ apiUrl
  api-key: ${PAYMENT_API_KEY:dummy-key} # placeholder + default dla lokalu
  timeout: 3s
  max-retries: 3
---
# application-prod.yml — tylko RÓŻNICE dla prod
payment:
  api-url: https://api.payments.io/v2
  max-retries: 5
```
```bash
# PROD: jeden i ten sam JAR/obraz; profil + sekret z env (relaxed binding: PAYMENT_API_KEY → payment.api-key)
SPRING_PROFILES_ACTIVE=prod \
PAYMENT_API_KEY=$(vault kv get -field=key secret/payment) \
java -jar app.jar
# albo doraźnie argumentem CLI (najwyższy priorytet):
java -jar app.jar --spring.profiles.active=prod --payment.timeout=8s
```
Efekt: `api-url` z profilu `prod`, `api-key` z Vault/env (nie z repo), `timeout` nadpisany argumentem CLI —
wszystko na **jednym artefakcie**, zgodnie z 12-factor.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, czym jest externalized config i jak realizuje zasadę Config z 12-factor (build once, deploy anywhere).
- [ ] Podam hierarchię źródeł Spring Boot i wskażę, co kogo nadpisuje (CLI > env > profil > bazowe > kod).
- [ ] Uzasadnię, kiedy `@ConfigurationProperties` bije `@Value` (type-safe, relaxed binding, walidacja, grupowanie).
- [ ] Poprawnie zarejestruję propsy (`@ConfigurationPropertiesScan` vs `@EnableConfigurationProperties`).
- [ ] Skonfiguruję profile + profile groups i aktywuję je env-em/CLI.
- [ ] Wskażę, gdzie trzymać sekrety i dlaczego nie w repo.

### 🔲 Black-box check
- [ ] Co robi `Environment`/`PropertySource` i jak przebiega `getProperty`? (kolejność źródeł)
- [ ] Jak działa relaxed binding: `MY_APP_URL` → jaka property? A `my.app.allowed-origins` z jakich zapisów bindnie?
- [ ] Czy `application.yml` i `application-prod.yml` się zastępują czy łączą? (merge, profil nadpisuje)
- [ ] Dlaczego `record` z `@ConfigurationProperties` potrzebuje `@ConfigurationPropertiesScan`, a nie `@Component`?
- [ ] Jak jeden artefakt dostaje inny config na dev i prod bez rebuildu?

### 🎤 Pytania rekrutacyjne
- [ ] „`@Value` czy `@ConfigurationProperties` — kiedy co i dlaczego?"
- [ ] „Jaka jest kolejność (priorytet) źródeł konfiguracji w Spring Boot?"
- [ ] „Jak przekazujesz różny config i sekrety na różne środowiska w k8s?"
- [ ] „Co to profile groups i jak aktywujesz profil na produkcji?"
- [ ] „Jak w praktyce realizujesz zasadę Config z 12-factor?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/05-spring.csv`.*
```
Co to externalized configuration?;Config trzymany poza kodem/artefaktem — ten sam build działa różnie w różnych środowiskach (12-factor: Config).
Którą zasadę 12-factor realizuje externalized config?;III. Config — store config in the environment, strict separation of config from code.
Priorytet źródeł konfiguracji Spring Boot (od najwyższego)?;Argumenty CLI > zmienne środowiskowe > application-{profile}.yml > application.yml > defaulty w kodzie.
Czy application.yml i application-prod.yml się zastępują?;Nie — są łączone (merge); wartości z aktywnego profilu nadpisują bazowe.
application.properties vs application.yml — który wygrywa gdy oba?;.properties ma pierwszeństwo nad .yml; YAML lepszy dla zagnieżdżeń/list i wielu dokumentów per profil.
Kiedy @ConfigurationProperties zamiast @Value?;Dla zestawu właściwości: type-safe binding, relaxed binding, walidacja @Validated, grupowanie, IDE support.
Co to relaxed binding?;Jedna property, wiele zapisów: myKebab / my-kebab / MY_KEBAB — wszystkie mapują na to samo pole.
Kanoniczny zapis property w pliku YAML?;kebab-case, np. allowed-origins.
Jak zmienna środowiskowa MY_APP_URL mapuje się na property?;Na my.app.url — _ → kropka, UPPERCASE → lowercase (relaxed binding).
Jak walidować @ConfigurationProperties?;@Validated na klasie + adnotacje Bean Validation (@NotBlank, @Min) → fail-fast przy starcie.
Czym różni się @ConfigurationPropertiesScan od @EnableConfigurationProperties?;Scan skanuje pakiety i rejestruje wszystkie propsy; Enable rejestruje jawnie wskazane klasy.
Czy sama @ConfigurationProperties tworzy bean?;Nie — trzeba @EnableConfigurationProperties, @ConfigurationPropertiesScan albo @Component na klasie.
Do czego służy @Profile?;Warunkowa rejestracja beana/konfiguracji zależnie od aktywnego profilu (np. @Profile("prod")).
Jak aktywować profil?;spring.profiles.active (property), --spring.profiles.active (CLI) lub SPRING_PROFILES_ACTIVE (env).
Co to profile group (spring.profiles.group)?;Jedna nazwa profilu włącza kilka innych naraz, np. prod → [prod, db-postgres, metrics].
Gdzie trzymać sekrety (nie w repo)?;Zmienne środowiskowe, HashiCorp Vault, Kubernetes Secrets — nigdy w application.yml w gitcie.
Jak ConfigMap trafia do aplikacji Spring Boot?;Jako env vars (relaxed binding) albo zamontowany plik application.yml w ./config/.
Do czego Spring Cloud Config Server?;Scentralizowana konfiguracja dla wielu serwisów (backend git/Vault), pobierana przez spring.config.import=configserver:.
spring.config.import vs @PropertySource?;spring.config.import — deklaratywnie w application.yml, obsługuje optional:/wiele backendów/YAML; @PropertySource — na @Configuration, domyślnie tylko .properties, niski priorytet.
Jak nadpisać property spakowaną w JAR bez rebuildu?;Podać ją jako argument CLI lub zmienną środowiskową (wyższy priorytet niż application.yml).
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/10-cloud-native/12-factor]] · [[wiedza/08-bezpieczenstwo/owasp]] · [[wiedza/05-spring/ioc-di]] · [[wiedza/10-cloud-native/konteneryzacja]]
