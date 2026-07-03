---
temat: "Auto-konfiguracja Spring Boot"
faza: 5
status: nieopanowany
priorytet: 🔴
tags: [java, spring, boot]
powiazane: ["[[wiedza/05-spring/ioc-di]]", "[[wiedza/05-spring/autokonfiguracja]]", "[[wiedza/05-spring/actuator]]"]
---

# Auto-konfiguracja Spring Boot

> **TL;DR (jedno zdanie):** Auto-configuration to zestaw **warunkowych** klas `@AutoConfiguration`, które Boot ładuje z pliku
> `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` w JAR-ach na classpath i aktywuje
> tylko wtedy, gdy spełnione są `@Conditional…` (jest klasa, brak Twojego beana, właściwość włączona) — żadnej magii, sam `if` na classpath.

## 1. Co — definicja i API

**Auto-configuration** = Spring Boot sam tworzy i konfiguruje beany na podstawie tego, **co widzi na classpath** i **co jest w properties**.
Dodałeś `spring-boot-starter-data-jpa` + sterownik + `spring.datasource.url` → dostajesz skonfigurowany `DataSource`, `EntityManagerFactory`,
`JpaTransactionManager` bez pisania ani jednego `@Bean`. Odejmij sterownik z classpath → te beany się nie pojawią. To „konwencja ponad konfiguracją”.

Punkt wejścia to jedna adnotacja na klasie `main`:

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

`@SpringBootApplication` to **meta-adnotacja** = złożenie trzech:

- **`@Configuration`** (a dokładniej `@SpringBootConfiguration`) — ta klasa sama jest źródłem definicji beanów (`@Bean`).
- **`@EnableAutoConfiguration`** — włącza cały mechanizm auto-konfiguracji (patrz sekcja 2). To ono „ściąga” setki gotowych configów.
- **`@ComponentScan`** — skanuje pakiet klasy `main` **i pakiety podrzędne** w poszukiwaniu `@Component`/`@Service`/`@Repository`/`@Controller`.
  Dlatego klasa `main` powinna leżeć w **korzeniu** struktury pakietów — inaczej część Twoich beanów nie zostanie znaleziona.

```
@SpringBootApplication
   ├── @SpringBootConfiguration  (→ @Configuration)   twoje @Bean-y
   ├── @EnableAutoConfiguration  (→ @Import(AutoConfigurationImportSelector))   beany frameworka, warunkowe
   └── @ComponentScan            twoje @Component/@Service/@Repository/@Controller
```

Możesz też użyć tych trzech osobno zamiast `@SpringBootApplication` — efekt identyczny.

## 2. Jak — `AutoConfiguration.imports` i `@Conditional` pod spodem

### Skąd Boot bierze listę kandydatów

`@EnableAutoConfiguration` importuje `AutoConfigurationImportSelector`. Ten na starcie:

1. Skanuje **wszystkie** JAR-y na classpath szukając pliku
   `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
   (Do Spring Boot 2.7 był to `META-INF/spring.factories`, klucz `EnableAutoConfiguration`; od 3.x to zwykły plik tekstowy — jedna FQCN klasy na linię.)
2. Zbiera z nich listę **pełnych nazw klas** oznaczonych `@AutoConfiguration` (np. `DataSourceAutoConfiguration`, `WebMvcAutoConfiguration`, `JacksonAutoConfiguration`).
3. Odfiltrowuje wykluczone (`exclude`) i te, których warunki na pewno nie przejdą.
4. Rejestruje pozostałe jako **kandydatów** — ale każdy z nich jest **warunkowy**.

```
spring-boot-autoconfigure.jar
  └── META-INF/spring/...AutoConfiguration.imports
        org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
        org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
        org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration
        ... (setki wpisów)
```

### Rodzina `@Conditional` — tu jest cały mechanizm

Każda auto-konfiguracja (i jej `@Bean`-y) jest obwieszona warunkami. Bean powstaje **tylko** gdy warunek zwróci `true`:

| Adnotacja | Aktywuje beana, gdy… |
|---|---|
| `@ConditionalOnClass` | dana klasa **jest** na classpath (np. `DataSource.class` → masz JDBC) |
| `@ConditionalOnMissingClass` | danej klasy **nie ma** na classpath |
| `@ConditionalOnMissingBean` | w kontekście **nie zdefiniowano** jeszcze beana tego typu — **klucz do nadpisywania!** |
| `@ConditionalOnBean` | inny bean **już istnieje** (zależność między configami) |
| `@ConditionalOnProperty` | właściwość ma zadaną wartość (np. `spring.h2.console.enabled=true`) |
| `@ConditionalOnWebApplication` / `@ConditionalOnNotWebApplication` | aplikacja jest (nie jest) webowa; można zawęzić do `SERVLET`/`REACTIVE` |
| `@ConditionalOnResource` | plik/zasób istnieje |
| `@ConditionalOnExpression` | wyrażenie SpEL jest prawdziwe |

Typowa auto-konfiguracja wygląda tak:

```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)                       // tylko gdy JDBC na classpath
@EnableConfigurationProperties(DataSourceProperties.class)  // wiązanie properties → obiekt
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean               // ← wycofa się, jeśli Ty zdefiniujesz własny DataSource
    public DataSource dataSource(DataSourceProperties props) {
        return props.initializeDataSourceBuilder().build();
    }
}
```

**`@ConditionalOnMissingBean` to serce „odczarowania”:** auto-konfig zawsze uruchamia się **jako ostatni** i mówi „dam Ci beana,
**chyba że już go masz**”. Dlatego Twoja definicja zawsze wygrywa z domyślną.

### Wiązanie properties (`@ConfigurationProperties`)

Wartości z `application.yml/properties` są mapowane na typowane obiekty przez `@ConfigurationProperties(prefix = "...")`,
włączane najczęściej przez `@EnableConfigurationProperties(XxxProperties.class)`. Relaxed binding pozwala pisać
`server.port`, `SERVER_PORT` (env), `server-port` — wszystko trafi do tego samego pola.

### Kolejność auto-konfiguracji

Configi bywają współzależne (np. transakcje potrzebują `DataSource`). Kolejność sterują:

- `@AutoConfigureBefore(X.class)` / `@AutoConfigureAfter(X.class)` — względnie do innej auto-konfiguracji,
- `@AutoConfigureOrder(n)` — bezwzględny priorytet (mniejsza liczba = wcześniej).

To **nie** jest zwykłe `@Order` — te trzy adnotacje działają wyłącznie w obrębie mechanizmu auto-configu.

## 3. Dlaczego / kiedy — nadpisywanie i diagnostyka

### Startery — czym są, a czym nie

**`spring-boot-starter-*` to kurowany zestaw zależności (BOM-owy „pakiet startowy”), NIE kod.** Starter typu `spring-boot-starter-web`
sam w sobie nie ma logiki — jego `pom` ściąga tranzytywnie: bibliotekę (Tomcat, Jackson, Spring MVC) **oraz** `spring-boot-autoconfigure`
z odpowiednimi klasami `@AutoConfiguration`. Dodanie jednej zależności = biblioteka + jej auto-konfiguracja naraz.
Idiom: „starter przynosi klocki, autoconfigure je składa”.

### Jak podejrzeć, co się faktycznie skonfigurowało

Zamiast zgadywać — **zajrzyj do raportu**:

- `--debug` (albo `debug=true` w properties) → w logu **`ConditionsEvaluationReport`**: sekcje
  *Positive matches* (co się włączyło i dlaczego), *Negative matches* (co odrzucone i **którego warunku zabrakło**), *Exclusions*, *Unconditional*.
- Actuator: endpoint **`/actuator/conditions`** (ten sam raport w JSON) oraz **`/actuator/beans`** (pełna lista beanów w kontekście).

To najważniejsze narzędzie „odczarowania” — na pytanie „czemu nie mam beana X?” odpowiada Negative matches: „bo brakuje klasy Y na classpath / bo property Z=false”.

### Nadpisywanie (override) auto-configu

Chcesz własną konfigurację? **Po prostu zdefiniuj własnego beana tego typu.** Dzięki `@ConditionalOnMissingBean`
auto-konfiguracja **sama się wycofa**:

```java
@Configuration
public class MyDbConfig {
    @Bean
    public DataSource dataSource() {           // Twój bean → OnMissingBean w Boot zwróci false
        return new HikariDataSource(/* własna konfiguracja */);
    }
}
```

Nie musisz nic wyłączać — Boot ustępuje. To celowa asymetria: Twój kod > domyślne konwencje.

### Wykluczanie całych auto-konfiguracji

Gdy chcesz kategorycznie wyłączyć cały config (nie tylko podmienić beana):

```java
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
```

albo w properties (działa też dla klas spoza Twojego kodu, po nazwie):

```properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

### Własna auto-konfiguracja / starter (zajawka)

Piszesz bibliotekę i chcesz, żeby „samo się skonfigurowało” u konsumenta:

1. Napisz klasę `@AutoConfiguration` z `@Bean`-ami owiniętymi w `@ConditionalOnMissingBean`/`@ConditionalOnProperty` (żeby dało się nadpisać i wyłączyć).
2. Zarejestruj ją, dopisując jej FQCN do
   `src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
3. (Konwencja) rozdziel na moduł `xxx-spring-boot-autoconfigure` (kod + wpis) i `xxx-spring-boot-starter` (sam `pom` z zależnościami).

### Pułapki

- **Za dużo na classpath** → coś się aktywuje, czego nie chciałeś (np. embedded DB `H2` w testach). Sprawdź Negative/Positive matches.
- **Klasa `main` za głęboko** → `@ComponentScan` nie widzi części pakietów.
- **Nadpisujesz przez zwykłego beana**, a nie widać efektu → prawdopodobnie dany `@Bean` w auto-configu **nie ma** `@ConditionalOnMissingBean`; wtedy trzeba `exclude`.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Piszesz wewnętrzny moduł `audit-spring-boot-starter`, który ma automatycznie dostarczać `AuditService`, ale zespoły mają móc go podmienić.

```java
@ConfigurationProperties(prefix = "audit")
public record AuditProperties(boolean enabled, String target) {}

@AutoConfiguration
@ConditionalOnClass(AuditService.class)
@ConditionalOnProperty(prefix = "audit", name = "enabled", havingValue = "true", matchIfMissing = true)
@EnableConfigurationProperties(AuditProperties.class)
public class AuditAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean              // konsument może dać własny AuditService
    public AuditService auditService(AuditProperties props) {
        return new DefaultAuditService(props.target());
    }
}
```

Wpis w `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```
com.example.audit.AuditAutoConfiguration
```

**Demonstracja wycofania.** Konsument, który definiuje własny bean:

```java
@Configuration
class Override {
    @Bean
    AuditService auditService() { return new KafkaAuditService(); }  // wygrywa
}
```

Uruchom z `--debug` i zobacz w raporcie:

```
Negative matches:
    AuditAutoConfiguration#auditService:
        Did not match: @ConditionalOnMissingBean found bean 'auditService'
```

To jest cała „magia”: auto-konfig **spytał** „czy ktoś już ma `AuditService`?”, dostał „tak” i **grzecznie się wycofał**.

---

## ✅ Kryteria opanowania
*Temat jest DOMKNIĘTY, gdy odpowiesz na wszystkie BEZ zaglądania.*

- [ ] Rozłożę `@SpringBootApplication` na `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` i powiem, co robi każda.
- [ ] Wskażę plik `AutoConfiguration.imports` (dawniej `spring.factories`) i wyjaśnię jego rolę.
- [ ] Wymienię rodzinę `@Conditional…` i wskażę, które służą do nadpisywania (`@ConditionalOnMissingBean`).
- [ ] Wyjaśnię, czemu własny bean „wygrywa” z auto-configiem (bez żadnego `exclude`).
- [ ] Odróżnię starter (zależności) od `autoconfigure` (kod + warunki).
- [ ] Pokażę, jak podejrzeć decyzje (`--debug`, `/actuator/conditions`, `/actuator/beans`).

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co konkretnie robi `AutoConfigurationImportSelector` na starcie kontekstu?
- [ ] Czym różni się `@ConditionalOnClass` od `@ConditionalOnBean` od `@ConditionalOnProperty`?
- [ ] Dlaczego auto-konfiguracja uruchamia się „na końcu” i co daje `@ConditionalOnMissingBean`?
- [ ] Jak działa kolejność: `@AutoConfigureBefore/After/Order` — i czym różni się od `@Order`?
- [ ] Jak `@ConfigurationProperties` + relaxed binding wiąże `SERVER_PORT` z `server.port`?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Skąd Spring Boot wie, jakie beany utworzyć?” (classpath + properties + `AutoConfiguration.imports` + `@Conditional`)
- [ ] „Jak nadpisać domyślną auto-konfigurację?” (własny bean → `@ConditionalOnMissingBean` się wycofa; albo `exclude`)
- [ ] „Czym jest starter i czy zawiera kod?” (nie — to zestaw zależności ściągający `autoconfigure`)
- [ ] „Jak zdiagnozować, czemu bean się nie pojawił?” (Negative matches z `--debug` / `/actuator/conditions`)
- [ ] „Jak napisać własną auto-konfigurację?” (`@AutoConfiguration` + wpis w `AutoConfiguration.imports`)

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Co to Spring Boot auto-configuration?;Automatyczne tworzenie i konfigurowanie beanów na podstawie tego, co jest na classpath i w properties.
@SpringBootApplication składa się z?;@Configuration (@SpringBootConfiguration) + @EnableAutoConfiguration + @ComponentScan.
Za co odpowiada @ComponentScan w @SpringBootApplication?;Skanuje pakiet klasy main i pakiety podrzędne w poszukiwaniu @Component/@Service/@Repository/@Controller.
Za co odpowiada @EnableAutoConfiguration?;Włącza mechanizm auto-konfiguracji przez import AutoConfigurationImportSelector.
Skąd Boot bierze listę auto-konfiguracji?;Z pliku META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports w JAR-ach na classpath.
Jak nazywał się plik z listą auto-configów przed Boot 2.7?;META-INF/spring.factories (klucz EnableAutoConfiguration).
Co robi @ConditionalOnClass?;Aktywuje beana/config tylko, gdy dana klasa jest obecna na classpath.
Co robi @ConditionalOnMissingBean i czemu jest kluczowe?;Tworzy beana tylko, gdy nie zdefiniowano go wcześniej — dzięki temu Twój bean nadpisuje auto-config.
Co robi @ConditionalOnProperty?;Aktywuje config tylko, gdy właściwość ma zadaną wartość (np. enabled=true).
Co robi @ConditionalOnWebApplication?;Aktywuje config tylko w aplikacji webowej (można zawęzić do SERVLET lub REACTIVE).
Jak nadpisać domyślny bean auto-configu?;Zdefiniować własny bean tego typu — dzięki @ConditionalOnMissingBean auto-config się wycofa.
Jak wyłączyć całą auto-konfigurację?;@SpringBootApplication(exclude=Xxx.class) lub property spring.autoconfigure.exclude=...
Czym jest spring-boot-starter-*?;Kurowany zestaw zależności (nie kod), który ściąga bibliotekę + spring-boot-autoconfigure z warunkowymi configami.
Jak sterować kolejnością auto-konfiguracji?;@AutoConfigureBefore / @AutoConfigureAfter / @AutoConfigureOrder (nie zwykłe @Order).
Jak podejrzeć, co się skonfigurowało?;Uruchom z --debug (ConditionsEvaluationReport) lub użyj /actuator/conditions oraz /actuator/beans.
Co pokazują Negative matches w raporcie warunków?;Odrzucone configi i którego @Conditional zabrakło (np. brak klasy lub property=false).
Jak stworzyć własną auto-konfigurację?;Klasa @AutoConfiguration z warunkowymi @Bean + wpis FQCN w META-INF/spring/...AutoConfiguration.imports.
Jak Boot wiąże properties na obiekty?;@ConfigurationProperties(prefix) + @EnableConfigurationProperties, z relaxed binding (server.port = SERVER_PORT).
Czemu klasa main powinna być w korzeniu pakietów?;Bo @ComponentScan skanuje jej pakiet i podpakiety — inaczej część beanów nie zostanie znaleziona.
Kiedy własny bean NIE nadpisze auto-configu?;Gdy dany @Bean w auto-configu nie ma @ConditionalOnMissingBean — wtedy trzeba użyć exclude.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/05-spring/ioc-di]] · [[wiedza/05-spring/autokonfiguracja]] · [[wiedza/05-spring/actuator]] · [[wiedza/05-spring/konfiguracja-profile]]
