---
temat: "Maven zaawansowany"
faza: 4
status: nieopanowany
priorytet: 🔴
tags: [java, narzedzia, maven]
powiazane: ["[[wiedza/00-fundament/maven]]", "[[wiedza/00-fundament/jdk-jre-jvm]]", "[[wiedza/04-narzedzia/git]]", "[[wiedza/10-cloud-native/konteneryzacja]]"]
---

# Maven zaawansowany

> **TL;DR:** Maven to nie tylko `mvn clean install`. Realna praca to: **reactor** wielomodułowy (parent POM + agregacja),
> **mediacja** konfliktów zależności (*nearest-wins*, `dependency:tree`), **BOM** i `dependencyManagement`, **profile** i
> `settings.xml` (mirrors, credentials), oraz **surefire vs failsafe** (unit vs integration). Pluginy podpinają swoje
> *goals* do **faz** cyklu życia — to serce Mavena.

Ta notatka **uzupełnia** podstawy z [[wiedza/00-fundament/maven]] (fazy, koordynaty GAV, cykl `clean`/`default`/`site`) —
tutaj głębia potrzebna do realnej pracy zespołowej i CI/CD.

## 1. Co — plugin-goal, fazy i default bindings

Maven sam z siebie **nic nie robi** — wszystko wykonują **pluginy** przez swoje **goals** (`plugin:goal`, np. `compiler:compile`).
Faza cyklu życia (`compile`, `test`, `package`…) to tylko *nazwa*, z którą **powiązane** są konkretne goals.

- **default bindings per packaging** — samo `<packaging>jar</packaging>` domyślnie podpina:
  `resources:resources` → `compiler:compile` → `resources:testResources` → `compiler:testCompile` →
  `surefire:test` → `jar:jar` → `install:install` → `deploy:deploy`. Dla `war` w `package` siedzi `war:war`,
  dla `pom` prawie nic (agregator). To dlatego samo `mvn package` „wie", co zrobić.
- **plugin executions** — własne podpięcie goala do fazy w `<build><plugins>`:
  ```xml
  <execution>
    <id>integration-tests</id>
    <phase>integration-test</phase>   <!-- do której fazy -->
    <goals><goal>integration-test</goal></goals>
  </execution>
  ```
- **`<pluginManagement>`** — jak `dependencyManagement`, ale dla pluginów: ustala **wersje i konfigurację**
  centralnie (w parent POM), a moduły potomne tylko *deklarują użycie* pluginu bez wersji.

Diagram powiązań (packaging `jar`):
```
faza:        compile        test          package     install     deploy
              │              │              │            │           │
goal:   compiler:compile  surefire:test  jar:jar   install:install deploy:deploy
```

## 2. Jak — reactor, mediacja, cykl pod spodem

### Multi-module reactor
Gdy `<packaging>pom</packaging>` ma `<modules>`, Maven uruchamia **reactor**: buduje wszystkie moduły w **jednym** przebiegu.

- **Agregacja vs dziedziczenie — to dwie różne rzeczy** (często mylone):
  - **Agregacja** (`<modules>` w rodzicu) — „zbuduj te moduły razem". Steruje *co* i *w jakiej kolejności*.
  - **Dziedziczenie** (`<parent>` w dziecku) — „odziedzicz `dependencyManagement`, properties, wersje pluginów".
  - Rodzic-agregator i parent-dziedziczenia **nie muszą być tym samym POM-em** (choć zwykle są).
- **Kolejność budowania** — Maven sam liczy **topological sort** grafu zależności między modułami
  (nie kolejność w `<modules>`!). Jeśli `service` zależy od `common`, `common` zbuduje się pierwszy.
- **`mvn -pl :service -am`** — zbuduj tylko moduł `service` **oraz** jego zależności (`-am` = *also make*).
  `-amd` (*also make dependents*) — moduł i wszystko, co od niego zależy. `-pl` przyjmuje `groupId:artifactId`,
  ścieżkę katalogu lub `!moduł` (wyklucz). W CI: `-pl` + `-am` daje szybsze buildy tylko zmienionych fragmentów.
- **`-rf :moduł`** (*resume from*) — wznów przerwany reactor od danego modułu.

### Mediacja konfliktów zależności
Zależności są **przechodnie** (transitive) — graf, nie lista. Gdy dwie ścieżki dają **różne wersje** tej samej biblioteki,
Maven **musi wybrać jedną** (na classpath jest jedna wersja artefaktu).

- **Reguła: *nearest-wins*** (najbliższa w drzewie, licząc głębokość od korzenia POM-a) — **NIE „najwyższa wersja"**.
  Przy remisie głębokości wygrywa **pierwsza zadeklarowana** (kolejność w POM). To częste źródło subtelnych bugów:
  transitywnie „przyciągasz" starszą wersję, bo jest bliżej.
- **`mvn dependency:tree`** — pokazuje pełne drzewo i (z `-Dverbose`) które wersje zostały *omitted for conflict*.
  Podstawowe narzędzie diagnostyki. `dependency:analyze` — wykrywa użyte-niezadeklarowane i zadeklarowane-nieużywane.
- **`<exclusions>`** — wytnij konkretną zależność przechodnią (np. wykluczyć `commons-logging`, `logback` gdy chcesz `log4j2`).
- **`<optional>true</optional>`** — zależność **nie propaguje się** przechodnio do konsumentów mojego artefaktu
  (używam jej, ale nie zmuszam użytkowników mojej biblioteki do jej wciągania).
- **scope `import`** — działa **tylko** w `<dependencyManagement>` z `<type>pom</type>`: „wciągnij cudzy
  `dependencyManagement`" — mechanizm **BOM**.

### dependencyManagement i BOM
- **`<dependencyManagement>`** — *ustala wersje* (i scope/exclusions) zależności, **bez** dodawania ich na classpath.
  Dziecko deklaruje `<dependency>` **bez `<version>`** — wersja spływa z zarządzania. Jedno miejsce prawdy o wersjach.
- **BOM (Bill of Materials)** — osobny POM (`<packaging>pom</packaging>`) zawierający **wyłącznie** `<dependencyManagement>`
  z komplementem spójnych wersji. Importujesz go przez `scope import`:
  ```xml
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>3.3.2</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
  ```
  `spring-boot-dependencies` zarządza setkami wersji (Jackson, Tomcat, Hibernate…) — dzięki temu deklarujesz `spring-boot-starter-web`
  **bez wersji** i masz gwarancję zgodności. (Wariant: dziedziczenie po `spring-boot-starter-parent`, które robi to samo + konfiguruje pluginy.)

### Profile, properties, filtrowanie
- **Profile** — warunkowe fragmenty POM (dodatkowe zależności, pluginy, properties). **Aktywacja**:
  `-Pprod` (jawnie), przez `<property>` (np. `env=ci`), `<jdk>[21,)</jdk>`, `<os>`, plik istnieje/nie, lub `<activeByDefault>true</activeByDefault>`.
  **Pułapka:** `activeByDefault` **przestaje działać**, gdy jakikolwiek inny profil jest aktywny jawnie — nie polegaj na nim krytycznie.
- **Properties** — `<properties>` (np. `<java.version>21</java.version>`), używane jako `${java.version}`.
- **Filtrowanie zasobów** — `<resources><filtering>true</filtering>` podmienia `${...}` w plikach (np. `application.properties`)
  wartościami properties w czasie budowy. Uwaga na kolizję z placeholderami Springa `${...}` → Spring Boot zmienia delimiter na `@...@`.

### settings.xml
Konfiguracja **użytkownika/środowiska**, nie projektu (nie commituje się!). Lokalizacja: **`~/.m2/settings.xml`** (user)
i `${MAVEN_HOME}/conf/settings.xml` (global); user nadpisuje global. Zawiera:
- **`<mirrors>`** — przekierowanie repozytoriów (np. `<mirrorOf>*</mirrorOf>` → firmowy Nexus/Artifactory jako proxy Central).
- **`<servers>`** — **credentials** (`<id>` musi pasować do `<id>` repozytorium/mirrora), hasła można szyfrować (`mvn --encrypt-password`).
- **`<proxies>`** — proxy HTTP korporacyjne.
- **`<profiles>`/`<activeProfiles>`** — profile globalne (np. ustawienia repozytoriów).

### effective-pom
`mvn help:effective-pom` — pokazuje **wynikowy** POM po scaleniu: dziedziczenie z parentów, `super-pom` Mavena
(domyślne repozytoria/pluginy), aktywne profile, interpolacja properties. Nieoceniony przy debugowaniu „skąd ta wersja/plugin".
`mvn help:effective-settings` — analogicznie dla `settings.xml`.

### surefire vs failsafe — kluczowe rozróżnienie
| | **surefire** | **failsafe** |
|---|---|---|
| Cel | testy **jednostkowe** | testy **integracyjne** |
| Faza | `test` | `integration-test` (uruchom) + `verify` (sprawdź wynik) |
| Nazewnictwo | `*Test`, `Test*` | `*IT`, `IT*`, `*ITCase` |
| Przy porażce | build pada **od razu** | pada w `verify` → daje szansę na **cleanup** (np. `post-integration-test`: zgasić kontener/DB) |

Failsafe rozdziela *uruchomienie* od *weryfikacji*, żeby testy startujące ciężkie zasoby (Testcontainers, embedded serwer)
zawsze mogły posprzątać. Dlatego integracyjne odpala się przez **`mvn verify`**, a nie `mvn test`.

## 3. Dlaczego / kiedy — praktyki, pułapki, ważne pluginy

### Ważne pluginy
- **maven-compiler-plugin** — używaj **`<release>21</release>`** (nie `source`+`target`): gwarantuje, że nie użyjesz API nowszego niż target.
- **maven-jar-plugin** — pakuje `.jar`, ustawia `Main-Class` w manifeście.
- **maven-shade-plugin vs spring-boot-maven-plugin** — obie robią „fat/uber JAR":
  - **shade** — *wtapia* klasy zależności do jednego JAR-a (może *relocate* pakiety, by uniknąć konfliktów). Ogólnego przeznaczenia.
  - **spring-boot-maven-plugin** — *nested JARs* (JAR w JAR-ze, własny loader `PropertiesLauncher`), **nie** rozpakowuje zależności; `repackage` + `build-image` (buildpacks). Standard dla Spring Boota.
- **jacoco-maven-plugin** — pokrycie testami (`prepare-agent` przed testami, `report`, `check` z progami).
- **maven-enforcer-plugin** — **wymusza reguły** buildu: min. wersja Mavena/JDK, brak duplikatów zależności (`dependencyConvergence`),
  zakazane zależności (`bannedDependencies`). Wywala build zanim problem trafi na prod.
- **versions-maven-plugin** — `versions:display-dependency-updates`, `versions:set` (podbij wersję we wszystkich modułach reactora).
- **maven-dependency-plugin** — `tree`, `analyze`, `copy-dependencies`, `go-offline`.

### Maven Wrapper (mvnw)
`mvnw`/`mvnw.cmd` + `.mvn/wrapper/` — skrypt, który **pobiera i uruchamia dokładnie wskazaną wersję Mavena** (bez globalnej instalacji).
Po co: **reprodukowalność** — wszyscy w zespole i CI budują tą samą wersją Mavena. Generujesz: `mvn wrapper:wrapper -Dmaven=3.9.8`.
Commitujesz do repo — nowy dev robi `./mvnw verify` bez instalowania Mavena.

### SNAPSHOT vs release, deploy
- **`-SNAPSHOT`** (`1.2.0-SNAPSHOT`) — wersja rozwojowa, **mutowalna**: Maven może codziennie pobierać nowszy build ze zdalnego repo
  (osobne `snapshotRepository`). Nie do produkcji.
- **release** (`1.2.0`) — **immutable**: raz opublikowana wersja nie może się zmienić (repozytoria to egzekwują).
- **`mvn deploy`** — publikuje artefakt do **zdalnego** repo (Nexus/Artifactory) wg `<distributionManagement>` i `<servers>` w settings.
  (`install` = lokalny `~/.m2`; `deploy` = zdalny.)

### Reproducible builds (zajawka)
Domyślnie JAR-y zawierają **timestampy** i kolejność plików zależną od systemu → dwa buildy tego samego commita dają **różne** artefakty (różny hash).
Maven wspiera **reproducible builds**: ustaw `<project.build.outputTimestamp>` (stała data, np. z commita) — plugin `jar`/`assembly`
normalizuje timestampy i porządek. Efekt: **bit-identyczny** artefakt → weryfikowalność łańcucha dostaw (supply chain).

### Pułapki
- Mylenie *nearest-wins* z „najwyższa wersja wygrywa" → zły debug konfliktów.
- Poleganie na `activeByDefault` przy jednoczesnym `-P`.
- Integracyjne testy jako `*Test` (odpala je surefire w `test` zamiast failsafe w `verify`).
- Wersje pluginów niezpinane w `pluginManagement` → build niereprodukowalny między maszynami.
- Credentials w `pom.xml` zamiast w `settings.xml`.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Firmowy monorepo Spring Boot: **parent POM** importuje `spring-boot-dependencies` (BOM), pina wersje pluginów w `pluginManagement`,
enforcer wymusza JDK 21. Moduły `common`, `service-api`, `service-app` w reactorze. CI robi `./mvnw -pl :service-app -am verify`:
surefire odpala unit testy, failsafe (`*IT`) odpala testy z Testcontainers w `integration-test`, jacoco liczy pokrycie w `verify`.
`settings.xml` na runnerze CI kieruje mirror na Nexus i podaje credentials do `deploy`.

### Przykład: `<build><plugins>` + profil
```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <configuration><release>21</release></configuration>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-failsafe-plugin</artifactId>
      <executions>
        <execution>
          <goals>
            <goal>integration-test</goal>
            <goal>verify</goal>          <!-- weryfikacja wyniku w fazie verify -->
          </goals>
        </execution>
      </executions>
    </plugin>
    <plugin>
      <groupId>org.jacoco</groupId>
      <artifactId>jacoco-maven-plugin</artifactId>
      <executions>
        <execution><id>prep</id><goals><goal>prepare-agent</goal></goals></execution>
        <execution><id>rep</id><phase>verify</phase><goals><goal>report</goal></goals></execution>
      </executions>
    </plugin>
  </plugins>
</build>

<profiles>
  <profile>
    <id>ci</id>
    <activation>
      <property><name>env</name><value>ci</value></property>   <!-- mvn ... -Denv=ci -->
    </activation>
    <properties>
      <maven.test.failure.ignore>false</maven.test.failure.ignore>
    </properties>
  </profile>
</profiles>
```

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, jak `plugin:goal` wiąże się z fazami i co daje packaging (default bindings).
- [ ] Odróżnię **agregację** od **dziedziczenia** w multi-module i wiem, jak reactor liczy kolejność.
- [ ] Wyjaśnię **nearest-wins** i przeprowadzę diagnozę konfliktu przez `dependency:tree`.
- [ ] Zbuduję BOM / użyję `scope import` i `dependencyManagement`.
- [ ] Rozróżnię **surefire vs failsafe** i wiem, czemu integracyjne idą przez `verify`.
- [ ] Skonfiguruję profil z aktywacją i sekcję `<build><plugins>` z głowy.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co dokładnie robi `mvn package` dla packaging `jar` (które goals, w jakich fazach)?
- [ ] Jak Maven wybiera wersję przy konflikcie? (nearest-wins, remis → pierwsza zadeklarowana)
- [ ] Czym różni się `<dependencyManagement>` od `<dependencies>`?
- [ ] Co robi `scope import` i czym jest BOM?
- [ ] Gdzie leży `settings.xml`, co w nim jest i czemu nie w POM?
- [ ] `-pl -am` — co dokładnie zbuduje?
- [ ] Czemu integracyjne testy padają w `verify`, a nie w `test`?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Jak działa mediacja zależności w Mavenie?" (nearest-wins, nie najwyższa)
- [ ] „surefire vs failsafe — różnica i po co?"
- [ ] „Jak zarządzasz wersjami w wielu modułach?" (parent POM + dependencyManagement/BOM)
- [ ] „Po co Maven Wrapper?"
- [ ] „SNAPSHOT vs release — różnica?"
- [ ] „shade vs spring-boot-maven-plugin?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/04-narzedzia.csv`. Format: `Pytanie;Odpowiedź`.*

```
Co wykonuje pracę w Mavenie — faza czy plugin?;Plugin przez swoje goal (plugin:goal); faza to tylko nazwa, do której goals są powiązane.
Reguła wyboru wersji przy konflikcie zależności?;Nearest-wins — najbliższa w drzewie od korzenia POM (NIE najwyższa wersja); remis głębokości → pierwsza zadeklarowana.
Czym różni się agregacja od dziedziczenia w multi-module?;Agregacja (<modules>) mówi CO zbudować i w jakiej kolejności; dziedziczenie (<parent>) przekazuje dependencyManagement, properties, wersje pluginów.
Jak Maven ustala kolejność budowy modułów reactora?;Sortowaniem topologicznym grafu zależności między modułami, nie kolejnością w <modules>.
Co robi mvn -pl :moduł -am?;Buduje wskazany moduł oraz jego zależności (-am = also make).
Do czego służy scope import?;Tylko w <dependencyManagement> z <type>pom</type> — importuje cudzy dependencyManagement (mechanizm BOM).
Co to BOM?;Bill of Materials — POM zawierający tylko <dependencyManagement> ze spójnymi wersjami, importowany przez scope import (np. spring-boot-dependencies).
Czym różni się <dependencyManagement> od <dependencies>?;dependencyManagement ustala wersje/scope BEZ dodawania na classpath; dependencies faktycznie dodaje zależność.
Co robi <optional>true</optional>?;Zależność jest używana lokalnie, ale NIE propaguje się przechodnio do konsumentów artefaktu.
surefire vs failsafe?;surefire = unit testy (*Test) w fazie test; failsafe = integracyjne (*IT) w integration-test (uruchom) i verify (sprawdź) — daje szansę na cleanup.
Którą komendą uruchamiasz testy integracyjne (failsafe)?;mvn verify (nie mvn test) — bo weryfikacja wyniku jest w fazie verify.
Gdzie leży settings.xml i co zawiera?;~/.m2/settings.xml (user) i conf/ (global); mirrors, servers (credentials), proxies, profiles — dane środowiska, nie projektu.
Do czego mvn help:effective-pom?;Pokazuje wynikowy POM po scaleniu dziedziczenia, super-pom, aktywnych profili i interpolacji properties.
Po co Maven Wrapper (mvnw)?;Pobiera i uruchamia dokładnie ustaloną wersję Mavena bez globalnej instalacji — reprodukowalność w zespole i CI.
SNAPSHOT vs release?;SNAPSHOT to mutowalna wersja rozwojowa (repo może pobrać nowszy build); release jest immutable — raz opublikowana się nie zmienia.
Różnica mvn install vs deploy?;install kopiuje artefakt do lokalnego ~/.m2; deploy publikuje do zdalnego repo (Nexus/Artifactory) wg distributionManagement.
shade vs spring-boot-maven-plugin?;shade wtapia klasy zależności do jednego JAR (może relocate); spring-boot używa nested JARs z własnym loaderem i nie rozpakowuje zależności.
Do czego maven-enforcer-plugin?;Wymusza reguły buildu (min. JDK/Maven, dependencyConvergence, bannedDependencies) — wywala build wcześnie.
Dlaczego używać <release>21</release> zamiast source+target?;Gwarantuje, że nie użyjesz API nowszego niż target — pilnuje zgodności z wersją bytecode.
Pułapka activeByDefault w profilach?;Przestaje działać, gdy jakikolwiek inny profil jest aktywowany jawnie (np. przez -P).
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/00-fundament/maven]] · [[wiedza/00-fundament/jdk-jre-jvm]] · [[wiedza/04-narzedzia/git]] · [[wiedza/10-cloud-native/konteneryzacja]]
