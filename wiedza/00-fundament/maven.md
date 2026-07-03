---
temat: "Maven (build tool)"
faza: 0
status: nieopanowany
priorytet: 🔴
tags: [java, fundament, build, maven]
powiazane: ["[[wiedza/00-fundament/jdk-jre-jvm]]", "[[wiedza/04-build/maven-gleboko]]", "[[wiedza/04-build/gradle]]"]
---

# Maven (build tool)

> **TL;DR:** **Maven** to deklaratywne narzędzie do budowania i zarządzania zależnościami w Javie oparte na zasadzie
> **convention over configuration** — opisujesz projekt w `pom.xml` (kim jest przez **GAV** + czego potrzebuje),
> a Maven sam wie *jak* go zbudować przez ustalony **build lifecycle**. Zależności ściąga **tranzytywnie** z repozytoriów
> (local `~/.m2` → central), a konflikty wersji rozwiązuje regułą **nearest-wins**.

## 1. Co — definicja i API

Maven (Apache Maven 3.9+) robi trzy rzeczy: **buduje** projekt (kompilacja → testy → artefakt), **zarządza zależnościami**
(ściąga JAR-y i ich zależności) oraz **standaryzuje** strukturę projektu. Filozofia: **convention over configuration** —
jeśli trzymasz się konwencji (układ katalogów, nazwy faz), `pom.xml` jest mały, bo Maven zna domyślne zachowanie.
Dwa projekty trzymające się konwencji budują się tym samym `mvn package`, niezależnie od tego, kto je pisał.

**Standardowy układ katalogów** (to *jest* konwencja — łamiesz ją tylko świadomie):
```
projekt/
├── pom.xml                  # opis projektu (Project Object Model)
├── src/
│   ├── main/java/           # kod produkcyjny → trafia do artefaktu
│   ├── main/resources/      # pliki niebędące kodem (application.yml, ...)
│   ├── test/java/           # kod testów → NIE trafia do artefaktu
│   └── test/resources/
└── target/                  # WSZYSTKO generowane: .class, JAR, raporty (czyszczone przez `mvn clean`)
```

**Minimalny `pom.xml`** (Java 21, JAR):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- GAV: tożsamość TEGO artefaktu -->
    <groupId>com.example</groupId>        <!-- "namespace", zwykle odwrócona domena -->
    <artifactId>moja-aplikacja</artifactId> <!-- nazwa artefaktu -->
    <version>1.0.0-SNAPSHOT</version>      <!-- SNAPSHOT = wersja w rozwoju, nadpisywalna -->
    <packaging>jar</packaging>             <!-- jar (domyślne) | war | pom (agregat) -->

    <properties>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.10.2</version>
            <scope>test</scope>            <!-- tylko do kompilacji/uruchomienia testów -->
        </dependency>
    </dependencies>
</project>
```

**GAV** (`groupId:artifactId:version`) to **współrzędne** jednoznacznie identyfikujące każdy artefakt w ekosystemie —
tak ty opisujesz swój projekt i tak wskazujesz cudze zależności. `packaging` mówi, co produkujesz: `jar`, `war`
(aplikacja webowa), albo `pom` (projekt-agregat bez własnego kodu — patrz multi-module).

**Podstawowe komendy:**
```bash
mvn validate     # sprawdź, czy pom jest poprawny i kompletny
mvn compile      # skompiluj src/main/java → target/classes
mvn test         # uruchom testy (najpierw kompiluje)
mvn package      # zbuduj JAR/WAR w target/
mvn verify       # uruchom testy integracyjne / sprawdzenia jakości
mvn install      # zainstaluj artefakt do local repo (~/.m2) — dostępny dla innych Twoich projektów
mvn clean        # usuń katalog target/
mvn clean package   # łączenie: najpierw clean, potem package (świeży build)
```

## 2. Jak — co dzieje się pod spodem

### Build lifecycle, fazy i goale

Maven ma trzy wbudowane **lifecycle**: `clean`, `default` (build) i `site`. Najważniejszy jest **default**,
złożony z uporządkowanych **faz** (phases):
```
validate → compile → test → package → verify → install → deploy
```
**Kluczowa mechanika: faza uruchamia WSZYSTKIE fazy przed nią.** `mvn package` wykona po kolei
`validate → compile → test → package`. Dlatego nie ma sensu pisać `mvn compile package` — `package` i tak zawiera
compile. Stąd też testy odpalają się przy każdym `package`/`install` (chyba że je pominiesz `-DskipTests`).

**Faza vs goal** — to nie to samo:
- **Faza** (phase) — etap *cyklu* (np. `compile`). Sama nic nie robi; jest "wieszakiem".
- **Goal** (goal) — konkretne zadanie *pluginu* (np. `compiler:compile`, `surefire:test`). To goale wykonują pracę.
- **Bindowanie**: pluginy **podpinają swoje goale do faz**. Domyślnie (dla `packaging=jar`) `compiler:compile`
  jest podpięty do fazy `compile`, `surefire:test` do `test`, `jar:jar` do `package`. Gdy uruchamiasz fazę,
  Maven odpala wszystkie goale podpięte do niej i do faz wcześniejszych.

Goal można też uruchomić bezpośrednio, poza cyklem: `mvn dependency:tree`, `mvn compiler:compile`.
Składnia `plugin:goal`.

### Pluginy

Maven sam w sobie prawie nic nie robi — **cała praca to pluginy**. Rdzeń to silnik, który podpina goale pluginów do faz.
Najważniejsze:
- **maven-compiler-plugin** — kompiluje `.java` → `.class` (czyta `maven.compiler.release`).
- **maven-surefire-plugin** — uruchamia testy *jednostkowe* w fazie `test`.
- **maven-failsafe-plugin** — testy *integracyjne* w fazie `verify` (oddzielone, by IT nie psuły szybkiego `test`).
- **maven-shade-plugin** / spring-boot-maven-plugin — budują **fat/uber JAR** (wszystkie zależności w jednym
  uruchamialnym JAR-ze, z poprawnym `Main-Class`). Bez tego JAR zawiera tylko Twój kod, bez zależności.

### Zależności, scope i tranzytywność

Deklarujesz zależność przez jej **GAV**; Maven ściąga ten JAR **i jego zależności** (**tranzytywnie**) — nie musisz
ręcznie wymieniać całego drzewa. Obejrzysz je przez `mvn dependency:tree`.

**Scope** określa, *kiedy* zależność jest na classpath:
| scope | compile main | compile/run test | w artefakcie | typowy przykład |
|---|---|---|---|---|
| `compile` (domyślny) | ✅ | ✅ | ✅ | biblioteka używana w kodzie produkcyjnym |
| `provided` | ✅ | ✅ | ❌ | API serwletów / Lombok — dostarcza je runtime/kontener |
| `runtime` | ❌ | ✅ | ✅ | sterownik JDBC — potrzebny tylko przy działaniu, nie do kompilacji |
| `test` | ❌ | ✅ | ❌ | JUnit, Mockito |

**Rozwiązywanie konfliktów wersji — nearest-wins.** Gdy dwie różne zależności ciągną tranzytywnie *różne wersje*
tej samej biblioteki, Maven wybiera tę **najbliższą w drzewie zależności** (najmniejsza głębokość od Twojego projektu),
a nie najwyższą wersję. Przy remisie głębokości wygrywa pierwsza zadeklarowana. To częste źródło błędów typu
`NoSuchMethodError` w runtime — diagnozujesz `mvn dependency:tree`, a naprawiasz przez jawne zadeklarowanie
pożądanej wersji u siebie (staje się "najbliższa") lub przez `dependencyManagement`.

### Repozytoria

Zależności żyją w **repozytoriach** (przeszukiwane w tej kolejności):
1. **local** (`~/.m2/repository`) — cache na Twojej maszynie. Pierwsze miejsce, gdzie Maven szuka; `mvn install`
   wkłada tu Twoje artefakty.
2. **central** (Maven Central) — domyślne publiczne repo, ściąga się stamtąd, czego nie ma lokalnie.
3. **remote** — firmowe (Nexus/Artifactory) lub inne, dodawane w `pom.xml`/`settings.xml`; często jako proxy/cache
   między Tobą a central.

### Multi-module i BOM (skrót — szczegóły w fazie 04)

- **Multi-module**: projekt nadrzędny z `packaging=pom` agreguje moduły (`<modules>`), które dziedziczą po nim
  konfigurację. Pozwala budować zestaw powiązanych artefaktów jedną komendą.
- **BOM / `dependencyManagement`**: sekcja, która *centralizuje wersje* zależności bez ich aktywowania —
  deklarujesz GAV w `<dependencies>` **bez `<version>`**, a wersję bierze z `dependencyManagement`. **BOM**
  (Bill of Materials, importowany przez `<scope>import</scope>`) to gotowy zestaw spójnych wersji (np. Spring Boot BOM)
  — gwarantuje, że biblioteki "grają ze sobą". Detale: [[wiedza/04-build/maven-gleboko]].

## 3. Dlaczego / kiedy — kompromisy i wąskie gardła

- **Plus**: jednolitość. Każdy projekt Maven budujesz tak samo, struktura jest przewidywalna, CI/IDE wiedzą, co robić.
  Deklaratywność = mniej "magii w skryptach".
- **Minus / pułapki**:
  - **Wielogadatliwość XML** i sztywność — niestandardowa logika buildu jest bolesna (trzeba pisać/konfigurować pluginy).
  - **Konflikty wersji tranzytywnych** — nearest-wins bywa nieintuicyjny; "działa u mnie" rozbija się o inne drzewo
    zależności gdzie indziej. Stąd waga BOM-ów i `dependency:tree`.
  - **SNAPSHOT-y** — wersja `-SNAPSHOT` jest nadpisywalna i może się zmienić między buildami → niedeterministyczny
    build, jeśli zależysz od cudzych SNAPSHOT-ów.
- **Maven vs Gradle (alternatywa)**: **Gradle** to drugie dominujące narzędzie buildu w JVM. Używa skryptów
  (Groovy/Kotlin DSL) zamiast deklaratywnego XML — jest **bardziej elastyczny** (łatwo customowa logika), ma
  **incremental build + build cache + daemon**, więc bywa szybszy na dużych/wielomodułowych projektach. Cena:
  imperatywność = więcej swobody, ale i więcej miejsc na bałagan; krzywa nauki jest stroma. **Wybór:** Maven, gdy
  cenisz konwencję, prostotę i przewidywalność (typowe backendy, Spring); Gradle, gdy potrzebujesz nietrywialnej
  logiki buildu, wydajności na monorepo albo robisz Androida (tam Gradle jest standardem). Szczegóły:
  [[wiedza/04-build/gradle]].

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)
Spring Boot API: `pom.xml` importuje **Spring Boot BOM** (`dependencyManagement`), więc dodajesz `spring-boot-starter-web`
**bez wersji** — wersje są spójne. `spring-boot-maven-plugin` jest podpięty do fazy `package` i robi **executable fat JAR**.
W CI lecisz `mvn clean verify` (clean → kompilacja → testy `surefire` → testy integracyjne `failsafe`). Gdy w produkcji
wyskakuje `NoSuchMethodError`, robisz `mvn dependency:tree` i widzisz, że dwie biblioteki ciągnęły różne wersje
np. Jacksona — naprawiasz przez przypięcie wersji w `dependencyManagement`.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię "convention over configuration" i pokażę standardowy układ katalogów.
- [ ] Napiszę z głowy minimalny poprawny `pom.xml` z GAV i jedną zależnością `test`.
- [ ] Wytłumaczę różnicę faza vs goal i dlaczego faza uruchamia wcześniejsze fazy.
- [ ] Wymienię scope'y i wiem, który trafia do artefaktu, a który nie.
- [ ] Wyjaśnię tranzytywność i regułę **nearest-wins** przy konflikcie wersji.
- [ ] Wiem, gdzie Maven szuka zależności (local → central → remote).

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co dokładnie robi `mvn package` krok po kroku? (które fazy i goale, w jakiej kolejności)
- [ ] Czym jest goal i jak pluginy podpinają goale do faz? (`compiler:compile` → faza `compile`)
- [ ] Co właściwie robi Maven, gdy zależność ma zależności? (drzewo, tranzytywność, `dependency:tree`)
- [ ] Jak rozstrzygany jest konflikt dwóch wersji tej samej biblioteki? (nearest-wins, nie highest)
- [ ] Czym różni się `mvn install` od `mvn package`? (instalacja do `~/.m2` vs tylko `target/`)
- [ ] Po co `maven-shade`/boot plugin — czym zwykły JAR różni się od fat JAR?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Czym jest Maven i co to GAV?"
- [ ] „Wymień fazy lifecycle i powiedz, co odpala `mvn install`."
- [ ] „Różnica między scope `provided` a `compile`?"
- [ ] „Masz konflikt wersji zależności — jak go zdiagnozujesz i naprawisz?"
- [ ] „Maven czy Gradle i dlaczego?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/00-fundament.csv`.*
```
Co oznacza zasada "convention over configuration" w Maven?;Trzymanie się ustalonych konwencji (układ katalogów, nazwy faz) sprawia, że pom.xml jest mały, bo Maven zna domyślne zachowanie.
Co to GAV?;groupId:artifactId:version — współrzędne jednoznacznie identyfikujące artefakt w ekosystemie Maven.
Gdzie trafia kod produkcyjny, a gdzie testowy?;Produkcyjny w src/main/java, testowy w src/test/java; wszystko generowane (.class, JAR) ląduje w target/.
Wymień fazy default lifecycle.;validate → compile → test → package → verify → install → deploy.
Co robi uruchomienie fazy w Maven?;Wykonuje tę fazę ORAZ wszystkie fazy ją poprzedzające (np. package odpala validate, compile, test).
Czym różni się faza od goala?;Faza to etap cyklu (wieszak, sama nic nie robi); goal to konkretne zadanie pluginu (plugin:goal) podpięte do fazy — to goale wykonują pracę.
Co robi maven-surefire-plugin?;Uruchamia testy jednostkowe w fazie test.
Czym jest fat/uber JAR i czym go zbudować?;JAR ze wszystkimi zależnościami i Main-Class, uruchamialny samodzielnie; budują go maven-shade-plugin lub spring-boot-maven-plugin.
Co oznacza scope provided?;Zależność dostępna przy kompilacji i testach, ale NIE pakowana do artefaktu (dostarcza ją runtime/kontener, np. API serwletów, Lombok).
Który scope nie trafia do artefaktu, a jest tylko do testów?;test (np. JUnit, Mockito).
Co to zależności tranzytywne?;Zależności Twoich zależności, które Maven ściąga automatycznie — obejrzysz je przez mvn dependency:tree.
Jak Maven rozwiązuje konflikt dwóch wersji tej samej biblioteki?;Regułą nearest-wins — wybiera wersję najbliższą w drzewie zależności (najmniejsza głębokość), nie najwyższą.
W jakiej kolejności Maven szuka zależności?;Local repo (~/.m2) → Maven Central → repozytoria remote (np. Nexus/Artifactory).
Czym różni się mvn install od mvn package?;package buduje artefakt w target/; install dodatkowo instaluje go do local repo (~/.m2), by inne Twoje projekty mogły go używać.
Do czego służy dependencyManagement / BOM?;Centralizuje wersje zależności (deklarujesz GAV bez version); BOM to importowany zestaw spójnych wersji gwarantujący kompatybilność.
Kiedy Gradle zamiast Maven?;Gdy potrzebujesz elastycznej logiki buildu, wydajności (incremental build, cache, daemon) na dużym monorepo lub robisz Androida.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/00-fundament/jdk-jre-jvm]] · [[wiedza/04-build/maven-gleboko]] · [[wiedza/04-build/gradle]]
