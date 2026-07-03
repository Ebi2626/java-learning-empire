---
temat: "SQL i relacyjne bazy danych — podstawy"
faza: 6
status: nieopanowany
priorytet: 🔴
tags: [java, sql, bazy]
powiazane: ["[[wiedza/06-persystencja/jpa-hibernate]]", "[[wiedza/06-persystencja/transakcje]]", "[[wiedza/06-persystencja/indeksy]]"]
---

# SQL i relacyjne bazy danych — podstawy

> **TL;DR:** Baza **relacyjna** = dane w **tabelach** (relacje) z **kluczami** i **constraints**, na których pracuje **SQL**.
> Zapytanie `SELECT` piszesz w jednej kolejności, ale silnik wykonuje je w **innej logicznej kolejności**
> (`FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT`) — stąd np. aliasu z `SELECT` nie użyjesz w `WHERE`.
> **INDEKSY** przyspieszają odczyt, ale spowalniają zapis; **transakcje** dają **ACID**, a **poziomy izolacji** to świadomy
> kompromis między poprawnością a współbieżnością. **NULL** to nie „zero" — to „nieznane", z logiką trójwartościową.

## 1. Co — model relacyjny, DDL/DML/DQL, anatomia SELECT

### Model relacyjny
- **Tabela** (relation) — zbiór **wierszy** (rows / tuples) o tym samym zestawie **kolumn** (columns / attributes).
- **Klucz główny** (`PRIMARY KEY`) — kolumna(-y) jednoznacznie identyfikujące wiersz; `NOT NULL` + `UNIQUE`.
  W PostgreSQL zwykle `id BIGINT GENERATED ALWAYS AS IDENTITY` (nowocześniej niż `SERIAL`).
- **Klucz obcy** (`FOREIGN KEY`) — kolumna wskazująca `PRIMARY KEY` innej tabeli; wymusza **integralność referencyjną**
  (nie wstawisz zamówienia dla nieistniejącego klienta). Opcje `ON DELETE CASCADE / SET NULL / RESTRICT`.
- **Ograniczenia** (constraints) — reguły egzekwowane przez bazę, nie aplikację:
  `NOT NULL`, `UNIQUE`, `CHECK (cena > 0)`, `PRIMARY KEY`, `FOREIGN KEY`, `DEFAULT`.
  Zasada: **constraint w bazie to ostatnia linia obrony** — aplikacja może mieć bug, baza pilnuje zawsze.

```sql
CREATE TABLE klient (
    id       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email    TEXT    NOT NULL UNIQUE,
    kraj     TEXT    NOT NULL DEFAULT 'PL'
);
CREATE TABLE zamowienie (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    klient_id  BIGINT NOT NULL REFERENCES klient(id) ON DELETE CASCADE,
    kwota      NUMERIC(10,2) NOT NULL CHECK (kwota >= 0),
    utworzono  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### DDL vs DML vs DQL
| Grupa | Znaczenie | Polecenia | Uwaga |
|---|---|---|---|
| **DDL** | Data **Definition** | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` | zmienia **schemat**; w PostgreSQL DDL jest **transakcyjne** (można w `BEGIN...ROLLBACK`) |
| **DML** | Data **Manipulation** | `INSERT`, `UPDATE`, `DELETE`, `MERGE` | zmienia **dane** |
| **DQL** | Data **Query** | `SELECT` | tylko czyta (część zalicza do DML) |
| **TCL/DCL** | transakcje / uprawnienia | `COMMIT`, `ROLLBACK`, `GRANT`, `REVOKE` | sterowanie |

### Anatomia SELECT i LOGICZNA kolejność wykonania
Piszesz `SELECT ... FROM ... WHERE ...`, ale silnik **rozumuje** w tej kolejności:

```
1. FROM / JOIN   → zbuduj zbiór wierszy (iloczyn tabel + warunki złączenia)
2. WHERE         → odfiltruj wiersze (przed grupowaniem, bez agregatów)
3. GROUP BY      → pogrupuj w koszyki
4. HAVING        → odfiltruj GRUPY (tu wolno agregaty)
5. SELECT        → policz wyrażenia i aliasy, wybierz kolumny
6. DISTINCT      → usuń duplikaty
7. ORDER BY      → posortuj (tu alias z SELECT JUŻ widoczny)
8. LIMIT/OFFSET  → obetnij
```

**Konsekwencja, którą MUSISZ rozumieć:** alias zdefiniowany w `SELECT` **nie istnieje jeszcze** w `WHERE`
(WHERE wykonuje się przed SELECT), ale **jest widoczny** w `ORDER BY` (sortowanie jest po SELECT).

```sql
-- BŁĄD: alias 'netto' nie istnieje w WHERE
SELECT kwota * 0.81 AS netto FROM zamowienie WHERE netto > 100;   -- ERROR: column "netto" does not exist
-- OK: powtórz wyrażenie w WHERE, alias działa dopiero w ORDER BY
SELECT kwota * 0.81 AS netto FROM zamowienie
WHERE kwota * 0.81 > 100
ORDER BY netto DESC;
```

## 2. Jak — JOINy, agregacja, CTE, indeksy, plan, izolacja pod spodem

### JOINy
Złączenie łączy wiersze z dwóch tabel po warunku (zwykle `FK = PK`).

```
klient          zamowienie
A               A, A, B (B nie ma zamówień → widoczny tylko w LEFT JOIN)
B
```

| JOIN | Zwraca |
|---|---|
| `INNER JOIN` | tylko pary spełniające warunek (przecięcie) |
| `LEFT [OUTER] JOIN` | wszystkie z lewej + dopasowane z prawej; brak → `NULL` po prawej |
| `RIGHT [OUTER] JOIN` | odwrotność LEFT (rzadko — zwykle zamień kolejność tabel) |
| `FULL [OUTER] JOIN` | wszystkie z obu stron; niedopasowane → `NULL` po drugiej stronie |
| `CROSS JOIN` | iloczyn kartezjański (każdy z każdym), bez warunku |

```sql
-- Klienci i liczba ich zamówień; klienci bez zamówień też (LEFT JOIN + COALESCE)
SELECT k.email, COUNT(z.id) AS liczba_zamowien
FROM klient k
LEFT JOIN zamowienie z ON z.klient_id = k.id
GROUP BY k.email
ORDER BY liczba_zamowien DESC;
```

**Pułapka:** warunek na tabelę „prawą" w `LEFT JOIN` daj w `ON`, nie w `WHERE` — `WHERE z.kwota > 100`
zamieni LEFT w INNER (odfiltruje wiersze z `NULL`).

### Agregacja: GROUP BY, HAVING vs WHERE
- Funkcje agregujące: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`. `COUNT(*)` liczy wiersze, `COUNT(kol)` pomija `NULL`.
- **`WHERE` filtruje wiersze PRZED grupowaniem, `HAVING` filtruje GRUPY PO agregacji.**
  W `WHERE` nie użyjesz `SUM(...)`; w `HAVING` — owszem.

```sql
SELECT klient_id, SUM(kwota) AS suma
FROM zamowienie
WHERE utworzono >= date '2026-01-01'   -- najpierw odfiltruj wiersze (tania praca)
GROUP BY klient_id
HAVING SUM(kwota) > 1000               -- potem odfiltruj grupy
ORDER BY suma DESC;
```

### Podzapytania i CTE (WITH)
- **Podzapytanie** — `SELECT` wewnątrz innego (`WHERE ... IN (SELECT ...)`, scalar subquery, `EXISTS`).
- **CTE** (`WITH`) — nazwane podzapytanie, poprawia czytelność, można łańcuchować i rekurencyjnie (`WITH RECURSIVE`).
  W PostgreSQL od wersji 12 CTE domyślnie **nie jest** barierą optymalizacji (bywają inline'owane).

```sql
WITH duzi_klienci AS (
    SELECT klient_id, SUM(kwota) AS suma
    FROM zamowienie
    GROUP BY klient_id
    HAVING SUM(kwota) > 10000
)
SELECT k.email, d.suma
FROM duzi_klienci d
JOIN klient k ON k.id = d.klient_id;
```
`EXISTS` bywa szybszy niż `IN` przy dużych podzbiorach i lepiej radzi sobie z `NULL`.

### INDEKSY
Indeks to dodatkowa struktura (domyślnie **B-tree**) przyspieszająca wyszukiwanie kosztem miejsca i zapisów.

- **B-tree** — zbalansowane drzewo posortowanych kluczy; wspiera `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, prefiks `LIKE 'abc%'`.
- **Kiedy pomaga:** kolumny w `WHERE`, warunkach `JOIN` (klucze obce!), `ORDER BY` (unika sortowania).
- **Indeks złożony (composite)** — kolejność kolumn ma znaczenie: `INDEX(a, b)` obsłuży `WHERE a = ?`
  i `WHERE a = ? AND b = ?`, ale **nie** samo `WHERE b = ?` (reguła **leftmost prefix**).
- **Covering index** (`INCLUDE`) — indeks zawiera wszystkie potrzebne kolumny → **index-only scan**, baza nie sięga do tabeli.
- **Selektywność** — indeks opłaca się gdy warunek zawęża do małego % wierszy. Kolumna typu `plec` (2 wartości)
  = niska selektywność → planner i tak zrobi seq scan.
- **Kiedy SZKODZĄ:** każdy `INSERT/UPDATE/DELETE` musi zaktualizować wszystkie indeksy tabeli → narzut zapisu i miejsca.
  Nadmiar indeksów = wolne zapisy. Indeks na kolumnie o niskiej selektywności = bezużyteczny balast.

```sql
CREATE INDEX idx_zam_klient ON zamowienie (klient_id);               -- pod JOIN/FK
CREATE INDEX idx_zam_data   ON zamowienie (utworzono DESC);          -- pod ORDER BY
CREATE INDEX idx_zam_kl_dat ON zamowienie (klient_id, utworzono);    -- composite (leftmost)
CREATE UNIQUE INDEX idx_klient_email ON klient (email);              -- egzekwuje UNIQUE
```

### PLAN ZAPYTANIA — EXPLAIN / EXPLAIN ANALYZE
- `EXPLAIN` — pokazuje **plan** (szacunki), bez uruchamiania.
- `EXPLAIN ANALYZE` — **uruchamia** i pokazuje rzeczywiste czasy i liczby wierszy (uwaga: wykonuje też `UPDATE/DELETE`!).
- Czytasz plan **od najbardziej wciętego węzła (od środka na zewnątrz)**.

Kluczowe węzły:
- **Seq Scan** — czyta całą tabelę. OK dla małych tabel lub gdy wybieramy większość wierszy; zły dla dużych + wąski filtr.
- **Index Scan** — po indeksie do wierszy tabeli. Dobre przy selektywnym `WHERE`.
- **Index Only Scan** — dane wprost z indeksu (covering).
- **Bitmap Heap Scan** — kompromis przy średniej selektywności.

```
EXPLAIN ANALYZE SELECT * FROM zamowienie WHERE klient_id = 42;
--> Index Scan using idx_zam_klient  (cost=0.29..8.31 rows=3 width=..) (actual time=0.02..0.03 rows=3 ...)
```
Sygnały ostrzegawcze: `Seq Scan` na dużej tabeli z filtrem; ogromna rozbieżność `rows` szacowane vs `actual`
(nieaktualne statystyki → `ANALYZE`); kosztowny `Sort` (rozważ indeks pod `ORDER BY`).

### TRANSAKCJE i ACID (co pod spodem)
Transakcja = grupa operacji „wszystko albo nic" między `BEGIN` a `COMMIT`/`ROLLBACK`.

- **A**tomicity — całość się wykona albo cofnie (`ROLLBACK`). Realizowane przez **WAL** (write-ahead log).
- **C**onsistency — po commit baza spełnia wszystkie constraints (przechodzi z jednego poprawnego stanu w drugi).
- **I**solation — równoległe transakcje nie „widzą" swoich niedokończonych zmian (patrz poziomy niżej).
  PostgreSQL używa **MVCC** (Multi-Version Concurrency Control) — czytelnicy nie blokują pisarzy i odwrotnie.
- **D**urability — po `COMMIT` dane przetrwają awarię (zapis WAL na dysk, `fsync`).

### POZIOMY IZOLACJI i anomalie
Anomalie współbieżności:
- **Dirty read** — czytasz dane innej **niezacommitowanej** transakcji.
- **Non-repeatable read** — ten sam wiersz odczytany dwa razy daje różne wartości (ktoś go zmienił i zacommitował między odczytami).
- **Phantom read** — ten sam warunek `WHERE` zwraca inny **zbiór** wierszy (ktoś dodał/usunął pasujący wiersz).

| Poziom izolacji | Dirty read | Non-repeatable | Phantom |
|---|---|---|---|
| READ UNCOMMITTED | możliwy* | możliwy | możliwy |
| READ COMMITTED | **nie** | możliwy | możliwy |
| REPEATABLE READ | **nie** | **nie** | możliwy** |
| SERIALIZABLE | **nie** | **nie** | **nie** |

\* W PostgreSQL `READ UNCOMMITTED` zachowuje się jak `READ COMMITTED` — brudnych odczytów nie ma nigdy.
\*\* Wg standardu SQL. W PostgreSQL `REPEATABLE READ` (snapshot isolation) eliminuje też phantomy; `SERIALIZABLE`
dodaje wykrywanie konfliktów serializacji (może zwrócić błąd `could not serialize` → retry w aplikacji).
Domyślny poziom w PostgreSQL to **READ COMMITTED**.

### Blokady (zajawka)
- **Row-level lock** — `SELECT ... FOR UPDATE` blokuje wybrane wiersze do końca transakcji, by inny nie zmodyfikował.
  Wzorzec przy „przeczytaj-zmodyfikuj-zapisz" bez utraty aktualizacji.
- `FOR UPDATE SKIP LOCKED` / `NOWAIT` — przydatne przy kolejkach zadań.
- Uwaga na **deadlocki** (dwie transakcje czekają na siebie krzyżowo) — trzymaj stałą kolejność blokowania.

```sql
BEGIN;
SELECT saldo FROM konto WHERE id = 1 FOR UPDATE;   -- blokada wiersza
UPDATE konto SET saldo = saldo - 100 WHERE id = 1;
COMMIT;
```

## 3. Dlaczego / kiedy — normalizacja, NULL, dobór, pułapki

### NORMALIZACJA (1NF/2NF/3NF)
Cel: usunąć **redundancję** i **anomalie** modyfikacji (jedną prawdę trzymaj w jednym miejscu).
- **1NF** — wartości **atomowe** (żadnych list w komórce), brak powtarzających się grup kolumn.
- **2NF** — 1NF + każdy atrybut **niekluczowy** zależy od **całego** klucza (dotyczy kluczy złożonych — brak zależności częściowej).
- **3NF** — 2NF + brak **zależności przechodnich** (atrybut niekluczowy nie zależy od innego niekluczowego).
  Reguła-mnemonik: *„każdy atrybut zależy od klucza, całego klucza i tylko od klucza"*.

### DENORMALIZACJA — świadomy kompromis
Czasem **celowo** duplikujesz dane (np. przechowujesz `suma_zamowienia` zamiast liczyć `JOIN`+`SUM` co odczyt),
by uniknąć kosztownych złączeń przy odczycie. Płacisz: redundancja + ryzyko niespójności (musisz utrzymać kopię w zgodzie).
Zasada: **normalizuj domyślnie, denormalizuj świadomie** pod zmierzony problem wydajności (read-heavy, raporty, cache).

### NULL i logika trójwartościowa
`NULL` = **„wartość nieznana"**, nie zero i nie pusty string. Wynik porównania z `NULL` to **UNKNOWN** (nie TRUE/FALSE).
- `WHERE x = NULL` → **nigdy** nie zwróci wiersza (nawet gdy `x IS NULL`). Używaj `IS NULL` / `IS NOT NULL`.
- `NULL = NULL` → `UNKNOWN`, nie TRUE.
- `x <> 5` **nie** zwróci wierszy gdzie `x IS NULL` (bo UNKNOWN ≠ TRUE) — częsta pułapka przy „wszystko oprócz 5".
- Agregaty (`SUM`, `AVG`) **pomijają** `NULL`; `COUNT(kol)` też. `COUNT(*)` liczy wszystko.
- `NULL` w `IN (...)` i `NOT IN (...)` bywa zdradliwe — `NOT IN` z `NULL` w liście da pusty wynik.

```sql
SELECT * FROM zamowienie WHERE kwota IS NULL;         -- poprawnie
SELECT * FROM zamowienie WHERE kwota <> 100;          -- POMIJA wiersze z kwota IS NULL!
SELECT * FROM zamowienie WHERE kwota <> 100 OR kwota IS NULL;   -- pełny „wszystko oprócz 100"
SELECT COALESCE(kwota, 0) FROM zamowienie;            -- zamień NULL na wartość domyślną
```

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)
W backendzie Spring Boot (Java 21) piszesz repozytorium, które zwraca „top klientów miesiąca". Naiwne zapytanie robi
`Seq Scan` po milionie zamówień — dodajesz composite index `(klient_id, utworzono)`, sprawdzasz `EXPLAIN ANALYZE`,
że pojawił się `Index Scan`. Serwis `@Transactional` (`READ COMMITTED`) w scenariuszu „doładuj saldo" cierpi na lost update —
przełączasz krytyczny fragment na `SELECT ... FOR UPDATE`. Model masz w 3NF, ale raport dzienny denormalizujesz do tabeli
podsumowań. Bug „brakuje klientów w raporcie" okazuje się `WHERE status <> 'active'` gubiącym wiersze z `status IS NULL`.
Całość spina JPA/Hibernate ([[wiedza/06-persystencja/jpa-hibernate]]), ale rozumienie SQL pod spodem jest niezbędne (N+1, plany).

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię model relacyjny: tabele, klucze PK/FK, constraints i po co integralność referencyjna.
- [ ] Podam LOGICZNĄ kolejność wykonania SELECT i wyjaśnię, czemu aliasu z SELECT nie użyję w WHERE.
- [ ] Narysuję różnicę INNER/LEFT/FULL JOIN i wiem, czemu warunek prawej tabeli daję w ON, nie WHERE.
- [ ] Rozróżnię WHERE vs HAVING i wiem, kiedy CTE zamiast podzapytania.
- [ ] Wyjaśnię B-tree, composite index (leftmost prefix), covering index, selektywność i kiedy indeks SZKODZI.
- [ ] Odczytam plan z EXPLAIN ANALYZE (seq scan vs index scan) i wskażę sygnały ostrzegawcze.
- [ ] Wymienię ACID, 3 anomalie i wypełnię tabelę „poziom → co eliminuje".
- [ ] Opiszę 1NF/2NF/3NF i uzasadnię świadomą denormalizację.
- [ ] Wyjaśnię logikę trójwartościową i pułapkę `= NULL`.

### 🔲 Black-box check
- [ ] Czemu `WHERE alias > 100` (alias z SELECT) rzuca błąd, a `ORDER BY alias` nie?
- [ ] Co dokładnie robi MVCC w PostgreSQL i czemu czytelnicy nie blokują pisarzy?
- [ ] Jak `EXPLAIN ANALYZE` różni się od `EXPLAIN` i czemu może zmienić dane?
- [ ] Czemu indeks na kolumnie o niskiej selektywności bywa ignorowany przez planner?
- [ ] Czemu `SELECT ... FOR UPDATE` chroni przed lost update?
- [ ] Co zwraca `NULL = NULL` i czemu to nie TRUE?

### 🎤 Pytania rekrutacyjne
- [ ] „Różnica WHERE vs HAVING?"
- [ ] „Kiedy indeks pomaga, a kiedy szkodzi?"
- [ ] „Wyjaśnij poziomy izolacji i anomalie — co eliminuje REPEATABLE READ?"
- [ ] „Co to ACID? Jak baza gwarantuje atomicity i durability?"
- [ ] „Masz LEFT JOIN i gubisz wiersze — co się stało?"
- [ ] „Czym jest NULL i jaka jest pułapka porównań?"
- [ ] „1NF/2NF/3NF — i kiedy świadomie denormalizujesz?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/06-persystencja.csv`.*
```
Do czego służą DDL, DML, DQL?;DDL definiuje schemat (CREATE/ALTER/DROP), DML zmienia dane (INSERT/UPDATE/DELETE), DQL czyta (SELECT).
Co wymusza FOREIGN KEY?;Integralność referencyjną — wartość musi istnieć jako PRIMARY KEY w tabeli docelowej.
Jaka jest LOGICZNA kolejność wykonania SELECT?;FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT.
Czemu aliasu z SELECT nie użyjesz w WHERE?;WHERE wykonuje się PRZED SELECT, więc alias jeszcze nie istnieje; w ORDER BY już działa.
Różnica WHERE vs HAVING?;WHERE filtruje wiersze przed grupowaniem (bez agregatów), HAVING filtruje grupy po agregacji.
INNER JOIN vs LEFT JOIN?;INNER zwraca tylko pasujące pary; LEFT zwraca wszystkie z lewej + NULL tam, gdzie brak dopasowania.
Czemu warunek prawej tabeli w LEFT JOIN daj w ON, nie WHERE?;WHERE na kolumnie prawej tabeli odrzuca wiersze z NULL i zamienia LEFT w INNER.
Co robi CROSS JOIN?;Iloczyn kartezjański — każdy wiersz z każdym, bez warunku złączenia.
Kiedy indeks B-tree pomaga?;Przy WHERE/JOIN/ORDER BY na selektywnych kolumnach oraz prefiksowym LIKE 'abc%'.
Co to leftmost prefix w indeksie złożonym?;INDEX(a,b) obsłuży WHERE a lub WHERE a AND b, ale nie samo WHERE b.
Co to covering index?;Indeks zawierający wszystkie potrzebne kolumny (INCLUDE) → index-only scan bez sięgania do tabeli.
Kiedy indeks SZKODZI?;Spowalnia INSERT/UPDATE/DELETE (aktualizacja indeksów), zajmuje miejsce; bezużyteczny przy niskiej selektywności.
Seq Scan vs Index Scan?;Seq Scan czyta całą tabelę (OK dla małych/szerokich zapytań), Index Scan idzie po indeksie (dobre dla selektywnego WHERE).
EXPLAIN vs EXPLAIN ANALYZE?;EXPLAIN pokazuje szacowany plan bez uruchomienia; EXPLAIN ANALYZE uruchamia zapytanie i podaje realne czasy/wiersze.
Co oznacza ACID?;Atomicity, Consistency, Isolation, Durability — gwarancje niezawodności transakcji.
Co to dirty read?;Odczyt danych innej, jeszcze niezacommitowanej transakcji.
Co to non-repeatable read a phantom read?;Non-repeatable: ten sam wiersz zmienia wartość między odczytami; phantom: ten sam WHERE zwraca inny ZBIÓR wierszy.
Co eliminuje SERIALIZABLE?;Wszystkie anomalie — dirty, non-repeatable i phantom read.
Domyślny poziom izolacji w PostgreSQL?;READ COMMITTED.
Co robi SELECT ... FOR UPDATE?;Blokuje wybrane wiersze do końca transakcji, chroniąc przed lost update.
Na czym polega 3NF?;Brak zależności przechodnich — atrybut niekluczowy zależy tylko od klucza, nie od innego niekluczowego.
Kiedy stosuje się denormalizację?;Świadomie, pod zmierzony problem wydajności odczytu — kosztem redundancji i ryzyka niespójności.
Czemu WHERE x = NULL nie działa?;Porównanie z NULL daje UNKNOWN (nie TRUE); trzeba użyć IS NULL / IS NOT NULL.
Czemu WHERE x <> 5 gubi wiersze z NULL?;NULL <> 5 to UNKNOWN, nie TRUE; dopisz OR x IS NULL, by je objąć.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/06-persystencja/jpa-hibernate]] · [[wiedza/06-persystencja/transakcje]] · [[wiedza/06-persystencja/indeksy]]
