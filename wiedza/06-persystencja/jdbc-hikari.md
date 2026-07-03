---
temat: "JDBC i connection pooling (HikariCP)"
faza: 6
status: nieopanowany
priorytet: 🔴
tags: [java, jdbc, bazy]
powiazane: ["[[wiedza/06-persystencja/sql-podstawy]]", "[[wiedza/06-persystencja/transakcje]]", "[[wiedza/08-bezpieczenstwo/owasp]]"]
---

# JDBC i connection pooling (HikariCP)

> **TL;DR:** **JDBC** to standardowe API Javy do rozmowy z relacyjną bazą (interfejsy w `java.sql`, konkretne
> **sterowniki** per baza). Zawsze używaj **`PreparedStatement`** (parametryzacja → brak **SQL injection**),
> pilnuj **transakcji** (`autocommit` domyślnie `true` — pułapka!) i **zamykaj `Connection`** (wycieki wieszają aplikację).
> Nawiązanie połączenia jest **drogie** (TCP + handshake + auth = dziesiątki ms), więc pod spodem trzyma się je w **puli**.
> **HikariCP** (domyślny w Spring Boot 3, najszybszy) to reużywa. Kluczowe: **`maximumPoolSize`** — i wbrew intuicji
> **mała pula (~10) zwykle wystarcza**; „więcej połączeń" to najczęściej gorzej, nie lepiej.

## 1. Co — definicja i API

**JDBC (Java Database Connectivity)** to zestaw interfejsów w pakiecie `java.sql` (moduł `java.sql`). Sama Java
definiuje kontrakt; producent bazy dostarcza **sterownik (driver)** implementujący te interfejsy dla swojego protokołu
sieciowego. Dla PostgreSQL: `org.postgresql:postgresql` (klasa `org.postgresql.Driver`). Dzięki temu **kod aplikacji
jest niezależny od bazy** — piszesz do interfejsów, wymieniasz tylko sterownik i URL.

Kluczowe typy:

- **`DriverManager`** — stary sposób pozyskania połączenia: `DriverManager.getConnection(url, user, pass)`.
  Tworzy **świeże** połączenie za każdym razem — bez puli, bez zarządzania. **Nie używaj w aplikacjach.**
- **`DataSource`** (`javax.sql.DataSource`) — **preferowana** fabryka połączeń. Za tym interfejsem kryje się
  najczęściej **pula** (HikariCP). `dataSource.getConnection()` zwraca połączenie **z puli** (patrz niżej).
  W Spring Boot masz gotowy bean `DataSource`.
- **`Connection`** — pojedyncza sesja z bazą. Steruje transakcją (`commit`, `rollback`, `setAutoCommit`), tworzy
  statementy. **Zasób do zamknięcia** (`AutoCloseable`).
- **`Statement`** — surowe SQL bez parametrów. **Podatny na SQL injection** przy sklejaniu stringów — unikaj.
- **`PreparedStatement`** — SQL z placeholderami `?`, parametry ustawiane osobno (`setString`, `setLong`…).
  **Domyślny wybór zawsze.**
- **`CallableStatement`** — do wywołań procedur składowanych (`{call proc(?, ?)}`).
- **`ResultSet`** — kursor po wynikach zapytania; iterujesz `while (rs.next())`, czytasz `rs.getString("col")`.

Minimalny odczyt (try-with-resources zamyka wszystko automatycznie):

```java
String sql = "SELECT id, email FROM users WHERE status = ?";
try (Connection con = dataSource.getConnection();
     PreparedStatement ps = con.prepareStatement(sql)) {
    ps.setString(1, "ACTIVE");
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) {
            System.out.println(rs.getLong("id") + " " + rs.getString("email"));
        }
    }
} // con.close() = ZWROT do puli, nie fizyczne zamknięcie połączenia
```

## 2. Jak — `PreparedStatement`, pula pod spodem

### `PreparedStatement` — dlaczego ZAWSZE

Trzy powody, w kolejności ważności:

1. **Ochrona przed SQL injection (najważniejsze).** Parametry `?` **nie są sklejane** ze stringiem SQL —
   sterownik/baza traktują je jako **dane**, nigdy jako fragment składni. Nie da się „wyjść" z parametru i wstrzyknąć SQL.

   ```java
   // ❌ PODATNE na SQL injection — konkatenacja
   String email = request.getParameter("email");           // np. ' OR '1'='1' --
   String bad = "SELECT * FROM users WHERE email = '" + email + "'";
   stmt.executeQuery(bad);   // atakujący zmienia SENS zapytania → wyciek całej tabeli

   // ✅ BEZPIECZNE — parametryzacja
   PreparedStatement ps = con.prepareStatement(
       "SELECT * FROM users WHERE email = ?");
   ps.setString(1, email);   // wartość jest DANĄ, nie kodem — cokolwiek by nie zawierała
   ps.executeQuery();
   ```

   To fundamentalna warstwa obrony z [[wiedza/08-bezpieczenstwo/owasp|OWASP Top 10 — Injection]]. Uwaga: parametryzować
   można **wartości**, nie identyfikatory (nazwy tabel/kolumn) — te trzeba walidować allow-listą.

2. **Prekompilacja / cache planu.** Baza parsuje i planuje zapytanie **raz** dla danego szablonu, a potem
   wykonuje z różnymi parametrami (PostgreSQL: prepared statements po stronie serwera po kilku wykonaniach).
   Mniej parsowania = niższy narzut CPU bazy.

3. **Poprawność typów** — `setTimestamp`, `setBigDecimal` itd. bez ręcznego formatowania i lokalizacji.

### Transakcje na poziomie JDBC (pułapka `autocommit`)

Domyślnie `Connection` ma **`autoCommit = true`** — **każdy** statement jest natychmiast commitowany osobno.
To pułapka: dwie operacje, które muszą być atomowe (np. zdjęcie z konta A, dopisanie na konto B), przy autocommit
mogą się rozjechać, jeśli druga padnie.

```java
con.setAutoCommit(false);          // wyłącz — startuje transakcja logiczna
try {
    ps1.executeUpdate();           // debet
    ps2.executeUpdate();           // kredyt
    con.commit();                  // atomowo utrwal oba
} catch (SQLException e) {
    con.rollback();                // cofnij oba
    throw e;
} finally {
    con.setAutoCommit(true);       // przywróć przed zwrotem do puli
}
```

W Springu tego **nie robisz ręcznie** — robi to `@Transactional` (patrz [[wiedza/06-persystencja/transakcje]]),
który pod spodem woła dokładnie `setAutoCommit(false)` / `commit` / `rollback` na połączeniu z DataSource.

### Batch — wydajność przy wielu zapisach

Zamiast N round-tripów do bazy, grupujesz operacje i wysyłasz razem:

```java
con.setAutoCommit(false);
try (PreparedStatement ps = con.prepareStatement("INSERT INTO log(msg) VALUES (?)")) {
    for (String m : messages) {
        ps.setString(1, m);
        ps.addBatch();                 // dokładaj do paczki
    }
    ps.executeBatch();                 // jeden przelot zamiast N
    con.commit();
}
```

Dla PostgreSQL warto do URL dodać `reWriteBatchedInserts=true`, by sterownik sklejał inserty. Batch redukuje głównie
**koszt latencji sieciowej** (round-trips), który przy pojedynczych zapisach dominuje.

### Dlaczego POOL — pula pod spodem

**Otwarcie połączenia jest drogie:** handshake TCP, negocjacja TLS, uwierzytelnienie, ustanowienie sesji po stronie
bazy (w PostgreSQL każde połączenie to **osobny proces** backendu). Realny koszt to **dziesiątki milisekund** i pamięć
po stronie bazy. Robienie tego per żądanie HTTP zabiłoby throughput i zasypało bazę procesami.

**Connection pool** utrzymuje pulę **już otwartych** połączeń. `getConnection()` **wypożycza** wolne połączenie
(mikrosekundy), a `close()` **zwraca je do puli** zamiast fizycznie zamykać. To dlatego „zamknięcie" jest tanie i
konieczne — inaczej połączenie zostaje wypożyczone na zawsze (wyciek).

**HikariCP** jest **domyślną pulą w Spring Boot** i najszybszą na rynku, bo:
- minimalny narzut na wypożyczenie/zwrot (zoptymalizowane struktury, `ConcurrentBag`),
- **brak zbędnych warstw** i abstrakcji nad JDBC (cienki proxy na `Connection`/`Statement`),
- inteligentne zarządzanie cyklem życia połączeń (walidacja, retirement).

## 3. Dlaczego / kiedy — sizing, wycieki, pułapki

### Kluczowe ustawienia HikariCP

| Ustawienie | Znaczenie | Uwaga |
|---|---|---|
| **`maximumPoolSize`** | max liczba połączeń w puli | **najważniejsze**; domyślnie 10 |
| `minimumIdle` | ile trzymać bezczynnych „na gotowo" | domyślnie = `maximumPoolSize` (stała pula — zwykle OK) |
| `connectionTimeout` | ile czekać na wolne połączenie, zanim rzuci wyjątek | domyślnie 30 s |
| `idleTimeout` | po jakim czasie zwolnić nadmiarowe idle połączenie | działa tylko gdy `minimumIdle < max` |
| **`maxLifetime`** | max żywotność połączenia zanim zostanie odświeżone | **krótszy niż timeout bazy/proxy!** (domyślnie 30 min) |
| `leakDetectionThreshold` | log ostrzeżenia, gdy połączenie wypożyczone dłużej niż X | **włącz w dev** (np. 20 s) do polowania na wycieki |

**`maxLifetime` krótszy niż limity infrastruktury** — jeśli PostgreSQL, PgBouncer, firewall czy load balancer zrywa
bezczynne połączenia po np. 10 min, a pula myśli, że są żywe, dostaniesz losowe „connection reset". Ustaw `maxLifetime`
z zapasem **poniżej** najniższego takiego limitu.

### Sizing puli — mit „więcej = lepiej"

Intuicja podpowiada dużą pulę „żeby starczyło". **To błąd.** Zbyt wiele połączeń **szkodzi bazie**: więcej procesów/
kontekstów, walka o CPU, dyski, locki → **spadek** przepustowości i wzrost latencji. Baza z ograniczoną liczbą rdzeni
i dysków obsłuży efektywnie tylko tyle równoległych zapytań, ile ma zasobów; reszta i tak czeka w kolejce — lepiej,
by czekała **w aplikacji** (tania kolejka) niż w bazie.

Punkt wyjścia (formuła z HikariCP / PostgreSQL wiki):

```
connections ≈ (rdzenie_bazy × 2) + efektywna_liczba_wrzecion_dysku
```

W praktyce **pula ~10 połączeń wystarcza** dla ogromnej większości serwisów, nawet obsługujących tysiące żądań/s
(bo zapytania trwają milisekundy — jedno połączenie „przerabia" wiele żądań). Skaluj dopiero, gdy metryki pokazują
realne czekanie na pulę. Pamiętaj o **sumie po wszystkich instancjach**: 20 podów × pula 10 = 200 połączeń do bazy —
łatwo przekroczyć `max_connections` PostgreSQL. Wtedy wchodzi zewnętrzny pooler (PgBouncer).

### Wycieki połączeń (connection leak)

Najczęstsza produkcyjna katastrofa: kod pobiera `Connection` i **nie zamyka** (brak try-with-resources, wyjątek przed
`close`, trzymanie połączenia w polu). Skutek: połączenie **nie wraca** do puli. Po kilku takich pula się **wyczerpuje**,
kolejne `getConnection()` blokuje się do `connectionTimeout`, a potem sypie wyjątkami — **cała aplikacja wisi**, choć
baza jest zdrowa. Obrona: **zawsze try-with-resources**, `leakDetectionThreshold` w dev, monitoring metryk puli
(`hikaricp_connections_active` / `pending` w Micrometer/Actuator).

### Jak JPA/Hibernate leży NA JDBC

JPA/Hibernate to **nadbudowa nad JDBC**, nie zamiennik. `EntityManager` **pobiera `Connection` z tego samego
`DataSource`** (puli HikariCP), tłumaczy JPQL/HQL na SQL, wykonuje przez `PreparedStatement`, mapuje `ResultSet` na
encje. Cały opisany wyżej świat — pula, transakcje, wycieki — obowiązuje **tak samo** pod Hibernate. Dlatego problemy
z pulą dotykają aplikacji JPA identycznie, a `spring.jpa.show-sql` / logi Hibernate pokazują ostatecznie zwykłe SQL-e
lecące przez JDBC. Relacja z transakcjami: [[wiedza/06-persystencja/transakcje]] (Spring `@Transactional` steruje
`autocommit`/`commit`/`rollback` na połączeniu z puli).

## Przykład w praktyce

Serwis Spring Boot 3 (Java 21) + PostgreSQL. Konfiguracja HikariCP w `application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://db:5432/app?reWriteBatchedInserts=true
    username: app
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 10          # start; skaluj wg metryk, nie „na oko"
      minimum-idle: 10               # = max → stała pula
      connection-timeout: 30000      # 30 s czekania na połączenie
      idle-timeout: 600000           # 10 min
      max-lifetime: 1740000          # 29 min — PONIŻEJ limitu bazy/proxy
      leak-detection-threshold: 20000  # ostrzeż o wypożyczeniu > 20 s
```

Repozytorium (JPA) i ręczne JDBC leżą na **tej samej** puli; metryki `hikaricp_connections_*` przez Actuator +
Micrometer trafiają do Prometheusa i alarmują, gdy `active` sięga `max` (objaw wycieku lub za małej puli).

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, czym jest JDBC i po co sterownik per baza.
- [ ] Uzasadnię, dlaczego `PreparedStatement` zawsze — i pokażę różnicę z konkatenacją (SQL injection).
- [ ] Wiem, że `autocommit` jest domyślnie `true` i jak ręcznie prowadzić transakcję JDBC.
- [ ] Wytłumaczę, dlaczego istnieje pula połączeń (koszt otwarcia) i czemu HikariCP jest szybki.
- [ ] Znam `maximumPoolSize`, `maxLifetime`, `leakDetectionThreshold` i regułę sizingu (mała pula wystarcza).
- [ ] Rozpoznam i wyjaśnię wyciek połączeń oraz jak wiesza aplikację.

### 🔲 Black-box check
- [ ] Co dokładnie zwraca `dataSource.getConnection()` i co robi `close()` na połączeniu z puli?
- [ ] Dlaczego parametr `?` w `PreparedStatement` nie da się wstrzyknąć? (dane vs składnia)
- [ ] Co się dzieje przy `autocommit=true`, gdy druga z dwóch operacji rzuci wyjątek?
- [ ] Czemu `maxLifetime` musi być krótszy niż timeout bazy/proxy?
- [ ] Co robi `EntityManager` na poziomie JDBC? (skąd bierze Connection)
- [ ] Dlaczego duża pula może obniżyć przepustowość bazy?

### 🎤 Pytania rekrutacyjne
- [ ] „Jak chronisz się przed SQL injection?" (PreparedStatement/parametryzacja, nie sklejanie)
- [ ] „Po co pula połączeń? Czemu nie otwierać połączenia per request?"
- [ ] „Jaki `maximumPoolSize` byś ustawił i dlaczego akurat tyle?"
- [ ] „Aplikacja wisi, baza jest zdrowa — co podejrzewasz?" (wyciek połączeń, wyczerpana pula)
- [ ] „Gdzie w tym wszystkim jest Hibernate?" (nadbudowa nad JDBC, ta sama pula)

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/06-persystencja.csv`. Format: `Pytanie;Odpowiedź`.*

```
Czym jest JDBC?;Standardowe API Javy (java.sql) do relacyjnych baz; konkretny sterownik implementuje je per baza.
DriverManager vs DataSource?;DriverManager tworzy świeże połączenie za każdym razem; DataSource (preferowany) to fabryka, zwykle z pulą pod spodem.
Dlaczego zawsze PreparedStatement?;Chroni przed SQL injection (parametry to dane, nie składnia), umożliwia cache planu i dba o typy.
Czemu konkatenacja SQL jest niebezpieczna?;Dane użytkownika stają się częścią składni zapytania — atakujący zmienia jego sens (SQL injection).
Czy w PreparedStatement można parametryzować nazwę tabeli?;Nie — tylko wartości; identyfikatory trzeba walidować allow-listą.
Jaka jest domyślna wartość autoCommit w JDBC?;true — każdy statement commitowany osobno (pułapka dla operacji atomowych).
Jak prowadzić transakcję ręcznie w JDBC?;setAutoCommit(false), potem commit() na sukces lub rollback() na wyjątek.
Do czego służy batch (addBatch/executeBatch)?;Grupuje wiele operacji w jeden przelot do bazy — redukuje koszt round-tripów sieciowych.
Dlaczego otwarcie połączenia jest drogie?;TCP + TLS handshake + uwierzytelnienie + sesja w bazie = dziesiątki ms (w PostgreSQL osobny proces backendu).
Co robi connection pool?;Trzyma otwarte połączenia i wypożycza je (mikrosekundy) zamiast otwierać nowe per żądanie.
Co znaczy close() na połączeniu z puli?;Zwraca połączenie do puli do reużycia, nie zamyka go fizycznie.
Czemu HikariCP jest najszybszy?;Mały narzut, brak zbędnych warstw nad JDBC, zoptymalizowane wypożyczanie (ConcurrentBag).
Najważniejsze ustawienie puli?;maximumPoolSize — górny limit połączeń.
Dlaczego maxLifetime musi być krótszy niż timeout bazy/proxy?;By pula odświeżyła połączenie zanim baza/proxy je zerwie — inaczej losowe connection reset.
Mit "większa pula = lepiej"?;Fałsz — za dużo połączeń przeciąża bazę (CPU/dyski/locki) i obniża throughput; mała pula ~10 zwykle wystarcza.
Formuła startowa na rozmiar puli?;~ (rdzenie_bazy × 2) + liczba wrzecion dysków; skalować dopiero wg metryk.
Czym jest wyciek połączenia i jego skutek?;Niezamknięty Connection nie wraca do puli; po wyczerpaniu puli getConnection() blokuje i aplikacja wisi.
Jak wykrywać wycieki w HikariCP?;leakDetectionThreshold (log przy zbyt długim wypożyczeniu) + metryki hikaricp_connections_active/pending.
Jak Hibernate/JPA korzysta z JDBC?;EntityManager pobiera Connection z tego samego DataSource (puli) i wykonuje SQL przez PreparedStatement.
Który pooler jest domyślny w Spring Boot?;HikariCP.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/06-persystencja/sql-podstawy]] · [[wiedza/06-persystencja/transakcje]] · [[wiedza/08-bezpieczenstwo/owasp]]
