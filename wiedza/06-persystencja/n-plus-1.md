---
temat: "Problem N+1 zapytań (N+1 select problem)"
faza: 6
status: nieopanowany
priorytet: 🔴
tags: [java, jpa, hibernate, wydajnosc]
powiazane: ["[[wiedza/06-persystencja/jpa-hibernate]]", "[[wiedza/06-persystencja/spring-data-jpa]]", "[[wiedza/06-persystencja/lazy-eager-fetch]]"]
---

# Problem N+1 zapytań (N+1 select problem)

> **TL;DR:** Jedno zapytanie pobiera **N** encji-rodziców, a potem **dla każdego** rodzica leci osobne zapytanie
> po jego relację `lazy` → razem **1 + N** zapytań zamiast 1–2. Działa na małych danych, **eksploduje na produkcji**.
> Leczenie: `JOIN FETCH` / `@EntityGraph` / **batch fetching** (`@BatchSize`) / **projekcje DTO**. `EAGER` to
> **fałszywe rozwiązanie** — robi N+1 zawsze i globalnie.

## 1. Co — definicja i API

**N+1 select problem** to sytuacja, w której odczyt kolekcji encji plus ich relacji generuje:

- **1** zapytanie pobierające listę **N** rodziców (`SELECT * FROM author`),
- **+N** zapytań — po jednym na **każdego** rodzica — pobierających jego relację `lazy`
  (`SELECT * FROM book WHERE author_id = ?` × N).

Razem **1 + N** round-tripów do bazy. Źródłem jest **lazy loading**: relacja (`@OneToMany`, `@ManyToMany`,
a także `@ManyToOne`/`@OneToOne` gdy `lazy`) nie jest ładowana z rodzicem — Hibernate podstawia **proxy**
(kolekcja: `PersistentBag`; strona `*ToOne`: bytecode proxy), a przy pierwszym dotknięciu dopala osobny `SELECT`.

Encje:

```java
@Entity
class Author {
    @Id @GeneratedValue Long id;
    String name;

    // @OneToMany domyślnie LAZY
    @OneToMany(mappedBy = "author")
    List<Book> books = new ArrayList<>();
}

@Entity
class Book {
    @Id @GeneratedValue Long id;
    String title;

    // @ManyToOne domyślnie EAGER (uwaga!) — tu wymuszamy LAZY
    @ManyToOne(fetch = FetchType.LAZY)
    Author author;
}
```

## 2. Jak — skąd bierze się 1+N pod spodem

Klasyczny wyzwalacz: **iteracja po kolekcji `lazy` w pętli** (albo dostęp do `*ToOne` lazy w pętli).

```java
List<Author> authors = em.createQuery("SELECT a FROM Author a", Author.class)
                         .getResultList();          // (1) zapytanie o rodziców

for (Author a : authors) {
    // pierwszy dostęp do books → Hibernate leci OSOBNY SELECT dla TEGO autora
    System.out.println(a.getName() + ": " + a.getBooks().size());  // (+N)
}
```

Log SQL **PRZED** naprawą (dla N = 3 autorów) — `1 + N` = 4 zapytania:

```sql
-- (1) rodzice
select a.id, a.name from author a
-- (+N) po jednym na autora, przy dotknięciu getBooks()
select b.id, b.title, b.author_id from book b where b.author_id = 1
select b.id, b.title, b.author_id from book b where b.author_id = 2
select b.id, b.title, b.author_id from book b where b.author_id = 3
```

**Dlaczego tak działa:** `getBooks()` zwraca leniwy `PersistentBag`. Metoda `size()`/iteracja **inicjalizuje** go —
Hibernate w sesji (kontekst persystencji musi być otwarty, inaczej `LazyInitializationException`) wysyła dedykowany
`SELECT`. Nikt z góry nie wie, że w pętli dotkniesz **wszystkich** kolekcji, więc każda idzie osobno. Analogicznie
`book.getAuthor().getName()` dla listy `Book` z `@ManyToOne(LAZY)` → +N zapytań o autorów.

**Groźny, bo cichy:** na 3 wierszach w teście integracyjnym różnica 4 vs 1 zapytania jest niewidoczna. Na produkcji,
gdzie `N` = 10 000 zamówień, dostajesz **10 001 zapytań** na jeden endpoint: latencja rośnie liniowo z `N`, connection
pool się zapycha, baza dostaje lawinę drobnych zapytań (każde z narzutem round-tripu i parsowania). To najczęstszy
regres wydajnościowy w aplikacjach JPA.

### Jak wykryć

- **Logowanie SQL** — `spring.jpa.show-sql=true` (ubogie) lub logger `org.hibernate.SQL=DEBUG`
  (+ `org.hibernate.orm.jdbc.bind=TRACE` na parametry w Hibernate 6). Widzisz N identycznych `SELECT ... WHERE fk = ?`.
- **Hibernate statistics** — `hibernate.generate_statistics=true`; `Statistics#getPrepareStatementCount()` (liczba
  wykonanych PreparedStatement) — w teście asercja „≤ 2 zapytania" łapie regresję.
- **Narzędzia zliczające/asertujące zapytania:** **datasource-proxy**, **p6spy** (proxy nad `DataSource` logujące
  każde zapytanie), **Hypersistence Utils** (`SQLStatementCountValidator.assertSelectCount(1)` w teście).

## 3. Dlaczego / kiedy — rozwiązania i ich kompromisy

**(1) `JOIN FETCH` w JPQL** — pobiera rodzica **razem z dziećmi jednym zapytaniem** (imperatywnie, per-query).

```java
List<Author> authors = em.createQuery(
    "SELECT DISTINCT a FROM Author a JOIN FETCH a.books", Author.class)
    .getResultList();
```
```sql
-- PO naprawie: 1 zapytanie z JOIN
select a.id, a.name, b.id, b.title, b.author_id
from author a join book b on b.author_id = a.id
```
Minusy: `DISTINCT` żeby zdeduplikować rodziców rozmnożonych joinem (w Hibernate 6 kolekcje i tak są deduplikowane
w pamięci); **nie da się `JOIN FETCH` dwóch kolekcji naraz** → `MultipleBagFetchException` (patrz niżej); **nie łączyć
z paginacją** (patrz pułapka).

**(2) `@EntityGraph`** — to samo **deklaratywnie** na repozytorium, bez pisania JPQL — patrz [[wiedza/06-persystencja/spring-data-jpa]].

```java
@EntityGraph(attributePaths = "books")
List<Author> findAll();          // Spring Data JPA doda fetch join pod spodem
```
Czytelniejsze i wielokrotnego użytku; te same ograniczenia co `JOIN FETCH` (kartezjan + paginacja).

**(3) Batch fetching** — `@BatchSize(size = n)` na kolekcji/encji albo globalnie
`hibernate.default_batch_fetch_size=n`. N+1 zamienia się w **1 + N/rozmiar_partii** dzięki `IN (...)`.

```java
@BatchSize(size = 20)
@OneToMany(mappedBy = "author")
List<Book> books;
```
```sql
-- zamiast N pojedynczych, partie po 20:
select b.* from book b where b.author_id in (?, ?, ?, ... /* do 20 */)
```
**Najlepszy kompromis, gdy potrzebujesz paginacji** — nie robi kartezjana, nie ładuje w pamięci, ładuje leniwie
partiami. Minus: nadal >1 round-trip (ale rzędy wielkości mniej niż N).

**(4) Projekcje DTO** — nie ładuj encji w ogóle; pobierz **tylko potrzebne kolumny** jednym zapytaniem.

```java
record AuthorBookView(String author, String title) {}

List<AuthorBookView> rows = em.createQuery(
    "SELECT new com.app.AuthorBookView(a.name, b.title) FROM Author a JOIN a.books b",
    AuthorBookView.class).getResultList();
```
Najszybsze do odczytów read-only (raporty, listy): brak zarządzania encjami, brak dirty checking, minimalny transfer.
Minus: to nie encje — nie zapiszesz ich z powrotem. (Spring Data: projekcje interfejsowe/klasowe.)

**(5) Drugie zapytanie z `IN`** — najpierw pobierz rodziców, potem **jednym** zapytaniem ich dzieci
`WHERE fk IN (:ids)` i zszyj w pamięci. Ręczny odpowiednik batch fetchingu; przydatne przy paginacji rodziców.

### ⚠️ Pułapka: `JOIN FETCH` + paginacja

Łączenie `JOIN FETCH` **kolekcji** z `setFirstResult/setMaxResults` (lub `Pageable`) → Hibernate loguje:

```
HHH000104: firstResult/maxResults specified with collection fetch; applying in memory
```

Znaczy: join daje **iloczyn kartezjański** (rodzic × dzieci), więc `LIMIT` na poziomie SQL policzyłby złe wiersze —
Hibernate **pobiera CAŁY wynik i pagesuje w pamięci JVM**. Przy dużej tabeli to OOM / potworna latencja.
**Rozwiązanie:** paginuj rodziców bez fetch join, a dzieci dociągnij przez **`@BatchSize`** albo **drugim zapytaniem
z `IN`**. (Fetch join strony `*ToOne` z paginacją jest OK — nie mnoży wierszy.)

### ⚠️ `EAGER` to fałszywe rozwiązanie

Zmiana relacji na `FetchType.EAGER` „naprawia" N+1 w jednym miejscu, ale:
- działa **zawsze i globalnie** — każdy `find`, każde `SELECT a FROM Author a` ciągnie kolekcję, także tam, gdzie
  jej nie potrzebujesz;
- proste `findById` potrafi wygenerować N+1 (Hibernate nie zawsze umie zrobić join dla EAGER kolekcji);
- `@ManyToOne`/`@OneToOne` są **domyślnie EAGER** — to typowe źródło ukrytego N+1.
- **Zasada:** relacje `LAZY` domyślnie, fetch dobierany **per zapytanie** (`JOIN FETCH`/`@EntityGraph`/`@BatchSize`).

### ⚠️ `MultipleBagFetchException`

Próba `JOIN FETCH` **dwóch** kolekcji typu `List` (Hibernate: „bag") naraz:

```
org.hibernate.loader.MultipleBagFetchException: cannot simultaneously fetch multiple bags
```
Bo iloczyn kartezjański dwóch kolekcji (M×K wierszy) jest nie do zdeduplikowania po pozycji. Wyjścia:
zmień `List` na `Set` (staje się „set", nie „bag" — można 2, ale nadal kartezjan), albo — lepiej — **jedna kolekcja
przez `JOIN FETCH`, reszta przez `@BatchSize`**, albo dwa osobne zapytania.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Endpoint `GET /orders` zwraca listę zamówień z pozycjami. Repozytorium robi `findAll()`, a mapper DTO iteruje
`order.getItems()`. W dev (kilka zamówień) działa; po wdrożeniu na tysiącach zamówień p99 skacze z 40 ms do 3 s,
a monitoring bazy pokazuje tysiące identycznych `SELECT ... FROM order_item WHERE order_id = ?`. Fix: `@EntityGraph`
na metodzie repozytorium (gdy bez paginacji) albo **projekcja DTO + `@BatchSize`** (gdy z paginacją), plus test
z `SQLStatementCountValidator.assertSelectCount(2)` blokujący regresję na przyszłość.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, czym jest N+1 i skąd bierze się dokładnie `1 + N` zapytań (lazy proxy + osobny SELECT per rodzic).
- [ ] Napiszę z głowy encje i pętlę generujące N+1 oraz pokażę log SQL przed/po.
- [ ] Wymienię 5 rozwiązań i wybiorę właściwe **z paginacją** (nie `JOIN FETCH` kolekcji!).
- [ ] Wytłumaczę, czemu `EAGER` pogarsza sprawę i czemu `@ManyToOne` jest domyślnie EAGER.
- [ ] Wykryję N+1 przez loggery/statistics/narzędzia i napiszę asercję liczby zapytań.

### 🔲 Black-box check
- [ ] Co dokładnie dzieje się przy pierwszym `getBooks().size()` na leniwej kolekcji? (proxy → SELECT)
- [ ] Czemu `JOIN FETCH` + `Pageable` daje `HHH000104` i pagesuje w pamięci? (kartezjan + zły LIMIT)
- [ ] Jak `@BatchSize` zmienia `1 + N` w `1 + N/rozmiar`? (grupowanie fk przez `IN (...)`)
- [ ] Czemu `EAGER` powoduje N+1 „zawsze i globalnie", a nie tylko w feralnym miejscu?
- [ ] Skąd `MultipleBagFetchException` i jak go obejść?

### 🎤 Pytania rekrutacyjne
- [ ] „Co to problem N+1 i jak go rozpoznasz?"
- [ ] „Masz N+1 przy paginowanej liście — jak naprawisz i czemu nie `JOIN FETCH`?"
- [ ] „Kiedy DTO projekcja, a kiedy fetch join?"
- [ ] „Dlaczego zmiana relacji na EAGER to zły pomysł?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Co to problem N+1 zapytań?;1 zapytanie pobiera N rodziców, a potem osobne zapytanie o relację (lazy) każdego rodzica → 1 + N zapytań zamiast 1-2.
Skąd bierze się +N w N+1?;Z lazy loadingu — dotknięcie relacji (proxy) każdego rodzica w pętli wyzwala osobny SELECT.
Czemu N+1 jest groźny dopiero na produkcji?;Na małych danych różnica niewidoczna; przy dużym N liczba zapytań rośnie liniowo → latencja i obciążenie bazy eksplodują.
Jak wykryć N+1 przez logi?;Logger org.hibernate.SQL=DEBUG (lub spring.jpa.show-sql) pokazuje N identycznych SELECT ... WHERE fk = ?.
Jak wykryć N+1 w teście?;Hibernate statistics (getPrepareStatementCount) albo Hypersistence Utils SQLStatementCountValidator.assertSelectCount(n).
Narzędzia do zliczania zapytań SQL?;datasource-proxy, p6spy, Hypersistence Utils (proxy nad DataSource logujące/asertujące zapytania).
Jak JOIN FETCH naprawia N+1?;Pobiera rodzica razem z dziećmi jednym zapytaniem z joinem zamiast 1 + N osobnych.
Po co DISTINCT przy JOIN FETCH kolekcji?;Join mnoży rodziców przez liczbę dzieci; DISTINCT deduplikuje rodziców (Hibernate 6 dedupikuje kolekcje w pamięci).
Co robi @EntityGraph?;Deklaratywnie (na repozytorium) wymusza fetch relacji jednym zapytaniem — odpowiednik JOIN FETCH bez pisania JPQL.
Jak działa batch fetching (@BatchSize)?;Grupuje ładowanie relacji przez IN (...); N+1 staje się 1 + N/rozmiar_partii.
Ustawienie globalnego batch fetch w Hibernate?;hibernate.default_batch_fetch_size=n.
Co to projekcja DTO i po co?;Zapytanie pobierające tylko potrzebne kolumny do DTO/record zamiast encji — najszybsze do read-only, bez zarządzania encjami.
Czemu nie łączyć JOIN FETCH kolekcji z paginacją?;Iloczyn kartezjański uniemożliwia LIMIT w SQL → Hibernate ostrzega HHH000104 i pagesuje CAŁY wynik w pamięci JVM.
Co znaczy log 'firstResult/maxResults specified with collection fetch; applying in memory'?;Że fetch join kolekcji + paginacja → Hibernate pobrał wszystko i stronicuje w pamięci (ryzyko OOM). Użyj @BatchSize lub dwóch zapytań.
Czemu EAGER to fałszywe rozwiązanie N+1?;Ładuje relację zawsze i globalnie (też gdzie zbędna), sam potrafi generować N+1 — gorsze niż lazy z fetch per-query.
Jaki jest domyślny fetch dla @ManyToOne i @OneToMany?;@ManyToOne/@OneToOne — EAGER (uwaga!); @OneToMany/@ManyToMany — LAZY.
Skąd MultipleBagFetchException?;Z próby JOIN FETCH dwóch kolekcji typu List (bag) naraz — nierozstrzygalny kartezjan; obejście: jedna fetch, reszta @BatchSize.
Które rozwiązanie N+1 wybrać przy paginacji rodziców?;@BatchSize albo drugie zapytanie z IN (NIE JOIN FETCH kolekcji).
Domyślna zasada fetchowania w JPA?;Relacje LAZY, a fetch dobierany per zapytanie (JOIN FETCH / @EntityGraph / @BatchSize).
Dlaczego dostęp do lazy poza sesją rzuca wyjątek?;LazyInitializationException — proxy potrzebuje otwartego kontekstu persystencji, by dopalić SELECT.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/06-persystencja/jpa-hibernate]] · [[wiedza/06-persystencja/spring-data-jpa]] · [[wiedza/06-persystencja/lazy-eager-fetch]]
