---
temat: "Flyway — wersjonowanie schematu bazy"
faza: 6
status: nieopanowany
priorytet: 🔴
tags: [java, bazy, migracje, devops]
powiazane: ["[[wiedza/06-persystencja/jpa-hibernate]]", "[[wiedza/10-cloud-native/cicd]]"]
---

# Flyway — wersjonowanie schematu bazy (database migrations)

> **TL;DR:** **Flyway** to „git dla schematu bazy": trzyma **migracje** jako wersjonowane pliki SQL
> (`V1__init.sql`, `V2__...`), wykonuje je **raz, w kolejności**, przy starcie aplikacji i zapisuje historię
> w tabeli `flyway_schema_history` z **checksumą**. Dzięki temu schemat ewoluuje **spójnie i powtarzalnie**
> u każdego dewelopera i na każdym środowisku. Na produkcji `ddl-auto=validate` — Hibernate **tylko waliduje**,
> a schematem zarządza wyłącznie Flyway.

## 1. Co — problem i definicja

**Problem.** Schemat bazy (`CREATE TABLE`, `ALTER`, indeksy, widoki) nie jest statyczny — ewoluuje z każdym
feature'em. Musi być **identyczny** u dewelopera lokalnie, na CI, na stagingu i na produkcji, i musi dać się
**odtworzyć** dla dowolnej wersji aplikacji. Bez narzędzia kończy się to ręcznym odpalaniem skryptów SQL, dryfem
schematów między środowiskami i pytaniem „czy Ty już puściłeś ten ALTER na stagingu?".

**Czemu `hibernate.ddl-auto=update` jest NIEBEZPIECZNE na produkcji.** Kuszące, bo Hibernate sam dopasuje
schemat do encji. Ale:
- **Nieprzewidywalne** — DDL generuje introspekcja mappingu; różne wersje Hibernate/dialekt mogą wygenerować różny SQL.
- **Brak kontroli** — nie widzisz i nie recenzujesz SQL, który poleci na produkcję; nie ma go w code review.
- **Brak historii** — nie wiesz, co i kiedy zmieniło schemat; nie da się tego cofnąć ani zaudytować.
- **`update` tylko dodaje** — **nie usuwa** kolumn/tabel i **nie zmienia** typów ani constraintów. Usunięcie pola
  z encji zostawia „martwą" kolumnę; zmiana typu bywa ignorowana → dryf między kodem a bazą.
- Na dużej tabeli niekontrolowany `ALTER` może **zablokować** tabelę (długi lock) w środku dnia.

**Flyway = kontrola wersji dla bazy.** Zamiast pozwalać ORM-owi zgadywać, **Ty** piszesz jawne migracje SQL,
wersjonujesz je w repo, a Flyway gwarantuje, że każda zostanie zastosowana **dokładnie raz** i **w kolejności**.

**Rodzaje migracji:**
- **Versioned** (`V1__opis.sql`, `V2__...`) — wykonane **raz**, w rosnącej kolejności wersji. To 95% przypadków:
  `CREATE TABLE`, `ADD COLUMN`, migracje danych. Po zastosowaniu są **niemutowalne**.
- **Repeatable** (`R__opis.sql`, bez numeru wersji) — wykonywane **ponownie za każdym razem, gdy zmieni się ich
  checksum**, i zawsze **po** wszystkich versioned. Idealne do obiektów definiowanych deklaratywnie i idempotentnie:
  **widoki, procedury, funkcje, triggery** (`CREATE OR REPLACE VIEW ...`). Zmieniasz treść → Flyway odtwarza obiekt.

**Naming convention** (kluczowa — literówka = pominięty plik):
```
V     2     __        add_user_email      .sql
prefix wersja separator      opis        rozszerzenie
(V/R)        (podwójne _)  (pojedyncze _ → spacja)
```
`V` = versioned, `R` = repeatable, `U` = undo (płatne). Separator to **`__`** (dwa podkreślenia).

## 2. Jak — historia, checksum, start pod spodem

**Tabela historii `flyway_schema_history`.** Flyway tworzy ją w schemacie bazy i zapisuje w niej każdą
zastosowaną migrację. Kluczowe kolumny:

| kolumna | znaczenie |
|---|---|
| `installed_rank` | kolejność faktycznego zastosowania |
| `version` | numer wersji (`2`) — `NULL` dla repeatable |
| `description` | opis z nazwy pliku |
| `type` | `SQL`, `BASELINE`, ... |
| `checksum` | **CRC-32 treści migracji** — do walidacji |
| `installed_on` / `installed_by` | kiedy i kto zastosował |
| `execution_time` | czas wykonania (ms) |
| `success` | czy migracja się powiodła |

To pełny, audytowalny **log zmian schematu**.

**Walidacja checksum — czemu NIGDY nie edytować zastosowanej migracji.** Przy każdym uruchomieniu Flyway
liczy checksum plików migracji i porównuje z zapisanym w `flyway_schema_history`. Jeśli zmienisz treść już
**zastosowanej** migracji (`V2`), checksum się rozjedzie → **`FlywayValidateException`** przy starcie i aplikacja
**nie wstanie**. To celowe zabezpieczenie: gwarantuje, że baza na produkcji powstała z **dokładnie tego** SQL,
który masz w repo. Poprawka istniejącej zmiany = **nowa migracja „w przód"** (`V3`), a nie edycja `V2`.
(Awaryjnie `flyway repair` przelicza checksumy, ale to obejście, nie rozwiązanie.)

**Jak Flyway startuje ze Spring Bootem (pod spodem):**
1. Dodajesz zależność `org.flywaydb:flyway-core` (dla PostgreSQL 15+ także `flyway-database-postgresql`).
   Spring Boot ma auto-konfigurację — wykrywa `flyway-core` na classpath.
2. Migracje trzymasz domyślnie w `src/main/resources/db/migration`.
3. Przy starcie kontekstu Springa `FlywayAutoConfiguration` uruchamia `flyway.migrate()` **wcześnie** — **zanim**
   zainicjalizuje się `EntityManagerFactory`/JPA. Dzięki temu Hibernate walidujący schemat widzi już **zmigrowaną** bazę.
4. Flyway bierze `DataSource`, blokuje bazę (lock), czyta stan z `flyway_schema_history`, wybiera **pending**
   migracje (wersja wyższa niż najwyższa zastosowana), sortuje po wersji i wykonuje je w **transakcji**
   (na PostgreSQL DDL jest transakcyjny — nieudana migracja się wycofuje).
5. Repeatable z nowym checksumem uruchamiane są na końcu.

**Baseline (dla istniejącej bazy).** Gdy wdrażasz Flyway na **działającej** bazie, która powstała przed nim,
`flyway baseline` oznacza aktualny stan jako wersję bazową (np. `V1`) i tworzy wpis `BASELINE` w historii.
Flyway wtedy **nie próbuje** wykonać migracji o wersji ≤ baseline — traktuje istniejący schemat jako punkt startu.
W Spring Boot: `spring.flyway.baseline-on-migrate=true` (+ `baseline-version`).

## 3. Dlaczego / kiedy — praktyki, zero-downtime, pułapki

**`ddl-auto` w parze z Flyway.** Podział odpowiedzialności: **Flyway = jedyny właściciel schematu**, Hibernate
tylko z niego korzysta. Na produkcji ustaw:
- `spring.jpa.hibernate.ddl-auto=validate` — Hibernate **sprawdza**, czy schemat pasuje do encji i wywala się przy
  starcie, jeśli nie (świetny „safety net" wyłapujący zapomnianą migrację), **albo**
- `none` — Hibernate w ogóle nie dotyka DDL.

Nigdy `update`/`create`/`create-drop` na produkcji.

**Rollback.** **Flyway Community jest forward-only.** Klasy `undo` migrations (`U1__...`) to funkcja **płatna**
(Teams/Enterprise) i w praktyce zawodna (nie każdą zmianę da się bezpiecznie cofnąć — np. `DROP COLUMN` gubi dane).
Praktyka: **nie cofasz — piszesz migrację „w przód"** (`V4`), która naprawia problem. Rollback aplikacji ≠ rollback bazy;
dlatego migracje projektuje się **zgodnie wstecznie**.

**Zero-downtime deploy — wzorzec expand/contract (parallel change).** Podczas deployu rolling stara i nowa wersja
aplikacji działają **jednocześnie** na tej samej bazie, więc schemat musi pasować do obu. Ryzykowną zmianę
(np. rename kolumny) rozbijasz na etapy w kilku releasach:
1. **Expand** — dodaj nowe (nullable) kolumny/tabele; stara i nowa wersja nadal działają. Zmiany **niełamiące**.
2. Kod zaczyna zapisywać do starego i nowego; backfill danych migracją.
3. Kod czyta już tylko z nowego.
4. **Contract** — dopiero po pełnym wdrożeniu usuwasz stare kolumny osobną migracją.

Migracje muszą być **backward-compatible** wobec wciąż działającej poprzedniej wersji aplikacji.

**Dobre praktyki:**
- **Jedna zmiana logiczna na migrację** — łatwiejszy review, mniejszy blast radius.
- **Migracje niemutowalne** — po zmergowaniu/zastosowaniu nigdy nie edytuj (patrz: checksum).
- **Testuj na kopii** produkcji (realny wolumen danych ujawnia długie locki i wolne `ALTER`-y).
- Trzymaj SQL w repo, przechodzi przez **code review** i **CI** ([[wiedza/10-cloud-native/cicd]]).
- Uważaj na **migracje danych** w tej samej transakcji co DDL na dużych tabelach.

**Liquibase jako alternatywa.** Opisuje zmiany w **XML/YAML/JSON/SQL** (changesets), jest **DB-agnostic**
(generuje dialekt pod docelową bazę) i ma **wbudowany rollback** (potrafisz zdefiniować krok odwrotny).
Kompromis: więcej abstrakcji i „magii" — trudniej dokładnie zobaczyć, jaki SQL poleci; Flyway z surowym SQL jest
prostszy i bardziej przewidywalny. Wybór: **wiele różnych baz + potrzeba rollbacku → Liquibase**; **jedna baza
(PostgreSQL), zespół dobrze zna SQL, prostota → Flyway**.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Spring Boot 3 / Java 21 / PostgreSQL. Struktura `src/main/resources/db/migration/`:

`V1__init.sql`
```sql
CREATE TABLE app_user (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username   VARCHAR(50) NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`V2__add_user_email.sql` (dodanie kolumny — niełamiące, nullable = bezpieczne pod rolling deploy)
```sql
ALTER TABLE app_user ADD COLUMN email VARCHAR(255);
CREATE UNIQUE INDEX ux_app_user_email ON app_user (email);
```

`R__active_users_view.sql` (repeatable — odtwarzany, gdy zmieni się treść)
```sql
CREATE OR REPLACE VIEW active_users AS
    SELECT id, username, email FROM app_user WHERE email IS NOT NULL;
```

Konfiguracja `application.yml`:
```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true   # dla istniejącej bazy
  jpa:
    hibernate:
      ddl-auto: validate        # Hibernate TYLKO waliduje; schematem rządzi Flyway
```
`pom.xml`:
```xml
<dependency>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```
Przy starcie aplikacji w logach zobaczysz: `Migrating schema "public" to version "2 - add user email"`,
a `SELECT * FROM flyway_schema_history` pokaże pełną historię z checksumami.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, czemu `ddl-auto=update` jest niebezpieczny na produkcji (nieprzewidywalny, bez historii, nie usuwa/zmienia).
- [ ] Rozróżniam migracje **versioned** (`V`, raz) i **repeatable** (`R`, przy zmianie checksum) i wiem, do czego które.
- [ ] Wiem, co jest w `flyway_schema_history` i jak działa walidacja checksum.
- [ ] Rozumiem, czemu NIGDY nie edytuje się zastosowanej migracji (i co robić zamiast tego).
- [ ] Umiem ustawić Flyway + `ddl-auto=validate` w Spring Boot i wiem, że migracje idą przed JPA.
- [ ] Znam wzorzec **expand/contract** dla zero-downtime deploy.
- [ ] Wiem, że Community jest forward-only i jak wygląda praktyczny „rollback".

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co dokładnie robi Flyway przy starcie aplikacji Spring Boot i w jakiej kolejności względem JPA?
- [ ] Jak Flyway decyduje, które migracje są „pending"?
- [ ] Co się stanie, gdy zmienisz treść zastosowanej migracji `V2` i zrestartujesz aplikację?
- [ ] Kiedy uruchamia się migracja repeatable, a kiedy nie?
- [ ] Po co jest `baseline` i kiedy go użyjesz?
- [ ] Czemu na PostgreSQL nieudana migracja potrafi się sama wycofać?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Czemu nie używać `hibernate.ddl-auto=update` na produkcji?"
- [ ] „Jak Flyway zapewnia, że migracja wykona się dokładnie raz i w kolejności?"
- [ ] „Jak zrobić rename kolumny bez downtime?" (expand/contract)
- [ ] „Jak cofnąć migrację we Flyway Community?" (forward-only → migracja naprawcza w przód)
- [ ] „Flyway vs Liquibase — kiedy co?"
- [ ] „Jak wdrożyć Flyway na istniejącej, działającej bazie?" (baseline)

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Czym jest Flyway?;Narzędziem do wersjonowania schematu bazy (database migrations) — "kontrola wersji dla bazy"; wykonuje migracje SQL raz, w kolejności, i zapisuje historię.
Czemu ddl-auto=update jest niebezpieczny na produkcji?;Jest nieprzewidywalny (DDL z introspekcji), bez kontroli/review, bez historii, tylko dodaje — nie usuwa kolumn/tabel ani nie zmienia typów.
Migracja versioned (V) — kiedy się wykonuje?;Dokładnie raz, w rosnącej kolejności wersji; po zastosowaniu jest niemutowalna.
Migracja repeatable (R) — kiedy się wykonuje?;Za każdym razem, gdy zmieni się jej checksum, zawsze po wszystkich versioned; do widoków, procedur, funkcji (CREATE OR REPLACE).
Naming convention Flyway?;prefix(V/R) + wersja + separator "__" (dwa podkreślenia) + opis + .sql, np. V2__add_user_email.sql.
Co przechowuje tabela flyway_schema_history?;Log zastosowanych migracji: version, description, checksum, installed_on/by, execution_time, success.
Co to checksum w Flyway i po co?;CRC-32 treści migracji porównywane przy starcie z zapisanym w historii — wykrywa zmianę już zastosowanej migracji.
Co się stanie, gdy edytujesz zastosowaną migrację?;Checksum się rozjedzie → FlywayValidateException przy starcie, aplikacja nie wstanie; dlatego migracje są niemutowalne.
Jak "poprawić" błędną, już zastosowaną migrację?;Napisać nową migrację "w przód" (kolejny numer wersji), nie edytować starej.
Czym jest baseline we Flyway?;Oznaczeniem aktualnego stanu istniejącej bazy jako wersji bazowej; Flyway nie wykonuje migracji o wersji <= baseline.
Kiedy Flyway uruchamia migracje w Spring Boot?;Przy starcie kontekstu, wcześnie — PRZED inicjalizacją EntityManagerFactory/JPA, żeby Hibernate widział zmigrowaną bazę.
Jakie ddl-auto ustawić z Flyway na produkcji?;validate (Hibernate tylko sprawdza zgodność schematu z encjami) lub none; nigdy update/create.
Czy Flyway Community wspiera rollback?;Nie — jest forward-only; undo migrations (U) to funkcja płatna. W praktyce piszesz migrację naprawiającą "w przód".
Na czym polega wzorzec expand/contract?;Zmianę schematu rozbijasz na niełamiące etapy (dodaj nowe → migruj dane → przełącz kod → usuń stare), by stara i nowa wersja app działały równocześnie (zero-downtime).
Gdzie domyślnie leżą migracje Flyway w Spring Boot?;W src/main/resources/db/migration (classpath:db/migration).
Flyway vs Liquibase — główna różnica?;Flyway = surowy SQL, prosty, przewidywalny, forward-only; Liquibase = XML/YAML/JSON, DB-agnostic, wbudowany rollback, więcej abstrakcji.
Jakiej zależności wymaga Flyway w Spring Boot?;org.flywaydb:flyway-core (dla PostgreSQL 15+ dodatkowo flyway-database-postgresql); resztę robi auto-konfiguracja.
Dobra praktyka co do rozmiaru migracji?;Jedna zmiana logiczna na migrację — łatwiejszy review i mniejszy blast radius; testuj na kopii produkcji.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/06-persystencja/jpa-hibernate]] · [[wiedza/10-cloud-native/cicd]]
