---
temat: "JPA i Hibernate"
faza: 6
status: nieopanowany
priorytet: 🔴
tags: [java, jpa, hibernate, bazy]
powiazane: ["[[wiedza/06-persystencja/transakcje]]", "[[wiedza/06-persystencja/n-plus-1]]", "[[wiedza/06-persystencja/spring-data-jpa]]"]
---

# JPA i Hibernate

> **TL;DR:** **JPA** (Jakarta Persistence) to *specyfikacja* mapowania obiektowo-relacyjnego (ORM); **Hibernate** to jej
> najpopularniejsza *implementacja*. Sercem jest **persistence context** (`EntityManager`) — cache pierwszego poziomu
> i *identity map*, który przez **dirty checking** zapisuje zmiany managed encji przy `flush` bez jawnego `save()`.
> Największy black-box backendu Java: lazy loading przez **proxy**, `LazyInitializationException`, N+1 i pułapka
> `equals`/`hashCode` na generowanym `id`.

## 1. Co — specyfikacja, implementacja, ORM

**JPA (Jakarta Persistence)** to standard Jakarta EE definiujący *API* (`EntityManager`, `EntityManagerFactory`),
*adnotacje* (`@Entity`, `@Id`, `@ManyToOne`…) i język zapytań **JPQL**. To tylko kontrakt — sam nic nie robi.
Do Jakarta EE 8 nazywał się `javax.persistence`; od Jakarta EE 9+ (Spring Boot 3, Hibernate 6) pakiet to
**`jakarta.persistence`** — to najczęstsza pułapka migracyjna.

**Hibernate** to najpopularniejszy *provider* JPA (alternatywy: EclipseLink, OpenJPA). Poza JPA oferuje własne
rozszerzenia (`@BatchSize`, filtry, `@Formula`, natywne SQL, `StatelessSession`). W Spring Boot 3 domyślnym
providerem jest **Hibernate 6** (Hibernate ORM 6.x, bazujący na Jakarta Persistence 3.1).

**ORM (Object-Relational Mapping)** mapuje obiekty Java na wiersze tabel. Rozwiązuje **object-relational
impedance mismatch** — fundamentalną niezgodność dwóch modeli:

| Świat obiektowy (Java) | Świat relacyjny (SQL) |
|---|---|
| dziedziczenie, polimorfizm | brak (płaskie tabele) |
| referencje/grafy obiektów | klucze obce, JOIN-y |
| tożsamość: `==` (referencja) vs `equals` | tożsamość: klucz główny (PK) |
| kolekcje zagnieżdżone | tabele powiązane, `@OneToMany` |
| brak nawigacji „w tył" bezpłatnie | JOIN działa w obie strony |

ORM daje produktywność (mniej boilerplate SQL, mapowanie grafów), ale kosztem **ukrytej złożoności** — SQL nie
znika, tylko jest generowany „za plecami", co bywa źródłem problemów wydajnościowych (N+1, nadmiarowe kolumny).

```java
@Entity
@Table(name = "authors")
public class Author {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "author_seq")
    @SequenceGenerator(name = "author_seq", sequenceName = "author_seq", allocationSize = 50)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Book> books = new ArrayList<>();
    // ...
}
```

## 2. Jak — persistence context, proxy, dirty checking pod spodem

### EntityManager i Persistence Context

`EntityManager` (odpowiednik Hibernate `Session`) zarządza **persistence context** (PC) — jednostką pracy
w obrębie jednej transakcji. PC jest jednocześnie:

- **Cache pierwszego poziomu (L1)** — obowiązkowy, per-`EntityManager`. Kolejne `find(Author.class, 1L)`
  w tej samej sesji NIE uderzają w bazę — zwracają ten sam obiekt z cache.
- **Identity map** — gwarantuje **jeden obiekt na parę `(typ, id)`** w obrębie sesji. Stąd:

```java
Author a1 = em.find(Author.class, 1L);
Author a2 = em.find(Author.class, 1L);
assert a1 == a2;               // TA SAMA referencja — gwarancja identity map
```

To fundament: dzięki temu Hibernate może śledzić zmiany i unikać duplikatów w grafie. Konsekwencja: PC to
*stateful* obiekt, którego **nie wolno współdzielić między wątkami**; w Spring żyje zwykle w granicach transakcji.

### Stany encji i przejścia

Encja przechodzi przez cztery stany. Znajomość tego diagramu to serce zrozumienia JPA:

```
                  new Author()
                       │
                       ▼
                 ┌───────────┐
                 │ TRANSIENT │  (nowy obiekt, nieznany PC, brak wiersza w DB)
                 └───────────┘
                       │  persist(a)
                       ▼
   find/get ──►  ┌───────────┐  ◄── merge(detached) [zwraca managed KOPIĘ]
   (z DB)        │  MANAGED  │      refresh(a) [nadpisz z DB]
                 │(persistent)│
                 └───────────┘
                   │        │
          detach(a)│        │ remove(a)
          / close()│        │
          / clear()│        ▼
                   │  ┌───────────┐  flush/commit
                   │  │  REMOVED  │──────────────► DELETE, wiersz znika
                   ▼  └───────────┘
             ┌────────────┐
             │  DETACHED  │  (był managed, ale PC zamknięty/wyczyszczony;
             └────────────┘   ma id, ale NIE jest śledzony — dirty checking nie działa)
```

- **TRANSIENT** — świeży obiekt (`new`), PC o nim nie wie, brak wiersza w DB.
- **MANAGED / PERSISTENT** — w PC, śledzony; zmiany pól trafią do DB przy flush (**dirty checking**).
- **DETACHED** — był managed, ale sesja się zamknęła (`close`), został `detach`owany lub zrobiono `clear`.
  Ma `id`, lecz zmiany już NIE są śledzone. Dostęp do lazy-pól → `LazyInitializationException`.
- **REMOVED** — zaplanowany do usunięcia; `DELETE` poleci przy flush.

Metody przejść: `persist` (transient → managed), `merge` (detached → managed, **zwraca nową managed instancję**,
oryginał zostaje detached!), `remove` (managed → removed), `detach`, `find` (ładuje jako managed),
`refresh` (nadpisuje stan obiektu z DB).

### Dirty checking — „magiczny" UPDATE bez save()

Przy przejściu w stan managed Hibernate robi **snapshot** (kopię wartości pól). Przy `flush` porównuje aktualny
stan ze snapshotem; jeśli coś się zmieniło — generuje `UPDATE`. Dlatego to działa **bez wywołania `save()`**:

```java
@Transactional
public void renameAuthor(Long id, String newName) {
    Author a = em.find(Author.class, id);  // MANAGED + snapshot
    a.setName(newName);                     // tylko mutacja w pamięci
    // brak save()! Przy commit → dirty check → UPDATE authors SET name=? WHERE id=?
}
```

To zaskakuje juniorów („czemu zmiana się zapisała, przecież nic nie zapisywałem?"). Odwrotnie — zmiana na
**detached** encji NIE zapisze się bez `merge`.

### Flush — kiedy SQL faktycznie leci do bazy

`flush` synchronizuje PC z bazą (wykonuje nagromadzone INSERT/UPDATE/DELETE), ale **nie commituje** transakcji.
Zdarza się:

1. **Automatycznie przed zapytaniem JPQL/HQL** (żeby zapytanie widziało niezapisane zmiany) — tryb `AUTO`.
2. **Przy commit** transakcji.
3. **Ręcznie** przez `em.flush()`.

`FlushModeType.AUTO` (domyślny) vs `COMMIT` (flush tylko przy commit — ryzykowne, zapytania mogą nie widzieć
zmian). Kluczowe: SQL jest **odroczony i batchowany** — Hibernate kolejkuje operacje, co pozwala na batch insert
(o ile strategia id na to pozwala — patrz niżej).

### Lazy loading, proxy i bytecode enhancement

Dla `LAZY` relacji Hibernate NIE ładuje danych od razu. Zamiast tego wstawia:

- **Proxy** — dla `@ManyToOne`/`@OneToOne` tworzy dynamiczny podtyp encji (CGLIB/ByteBuddy). Pola są puste do
  pierwszego dostępu; wtedy proxy „doładowuje" dane SELECT-em (o ile sesja jest otwarta).
- **PersistentCollection** — dla `@OneToMany`/`@ManyToMany` opakowuje kolekcję (`PersistentBag`, `PersistentSet`);
  ładuje elementy przy pierwszym dostępie (`size()`, iteracja).
- **Bytecode enhancement** (opcjonalne, build-time) — pozwala na lazy ładowanie pojedynczych *pól* (np. lazy
  `@Basic` czy `@ManyToOne` bez proxy) i `@LazyGroup`.

## 3. Dlaczego / kiedy — pułapki (LazyInit, equals, fetch)

### LazyInitializationException — klasyczny błąd

Dostęp do lazy relacji **poza sesją/transakcją** (PC zamknięty) rzuca `LazyInitializationException` — proxy nie
ma jak wykonać SELECT.

```java
public Author getAuthor(Long id) {
    return repo.findById(id).orElseThrow();   // transakcja kończy się TU
}
// w kontrolerze/serializacji JSON:
author.getBooks().size();   // 💥 LazyInitializationException — sesja już zamknięta
```

**Naprawa (dobre → złe):**
1. **Pobierz dane w transakcji** — `JOIN FETCH` w zapytaniu (`SELECT a FROM Author a JOIN FETCH a.books`)
   lub **EntityGraph** (`@EntityGraph`). To najczystsze rozwiązanie.
2. **DTO projection** — mapuj na DTO w obrębie transakcji, nie wypuszczaj encji na warstwę web.
3. ❌ `spring.jpa.open-in-view=true` (OSIV) — trzyma sesję do końca żądania. Domyślnie włączony w Spring Boot,
   ale to **anty-wzorzec**: maskuje N+1, trzyma połączenie DB przez cały render. Zaleca się `open-in-view=false`.
4. ❌ Zmiana na `EAGER` — leczy objaw, tworzy N+1 i ładuje za dużo.

### Fetch: LAZY vs EAGER

Domyślne fetch typy (WARTO ZNAĆ NA PAMIĘĆ):

| Relacja | Domyślny fetch |
|---|---|
| `@ManyToOne` | **EAGER** |
| `@OneToOne` | **EAGER** |
| `@OneToMany` | **LAZY** |
| `@ManyToMany` | **LAZY** |

**Zaleca się `LAZY` wszędzie** (`@ManyToOne(fetch = FetchType.LAZY)`), a potrzebne dane dociągać jawnie przez
`JOIN FETCH`/EntityGraph. `EAGER` jest globalny i niekontrolowalny — nawet gdy dana relacja nie jest potrzebna,
ładuje się zawsze, mnożąc JOIN-y i N+1.

### N+1 — najczęstszy problem wydajnościowy

Iterując po `authors` i sięgając do `author.getBooks()` dla każdego, dostajesz 1 zapytanie o autorów + N zapytań
o książki. Szczegóły i rozwiązania: [[wiedza/06-persystencja/n-plus-1]].

### Strategie generowania id (`@GeneratedValue`)

| Strategia | Jak działa | Uwagi (PostgreSQL) |
|---|---|---|
| `IDENTITY` | kolumna auto-increment (`SERIAL`/`BIGSERIAL`) | **wyłącza batch insert** — Hibernate musi znać id od razu po INSERT, więc nie może kolejkować wielu wstawień |
| `SEQUENCE` | dedykowana sekwencja DB | **preferowana** z PostgreSQL; z `allocationSize` pobiera id w **partiach** (pooled optimizer) → mniej round-tripów, umożliwia batch insert |
| `AUTO` | provider wybiera | w Hibernate 6 na PostgreSQL zwykle → `SEQUENCE` (globalna `hibernate_sequence` znikła w H6, każdy encja ma swoją) |
| `TABLE` | osobna tabela jako licznik | przenośne, ale wolne (blokady) — unikać |

Klucz: **`SEQUENCE` z `allocationSize` = 50** pozwala Hibernate zarezerwować pulę id bez ciągłych zapytań i
grupować INSERT-y w batch (`hibernate.jdbc.batch_size`). `IDENTITY` to zabija.

### @Enumerated — pułapka STRING vs ORDINAL

```java
@Enumerated(EnumType.ORDINAL)   // ❌ zapisuje 0,1,2 wg KOLEJNOŚCI w enum
@Enumerated(EnumType.STRING)    // ✅ zapisuje nazwę: "ACTIVE"
private Status status;
```

**`ORDINAL` (domyślny, gdy brak adnotacji!) jest niebezpieczny** — dodanie/przestawienie wartości enuma przesuwa
liczby i cicho psuje istniejące dane. **Zawsze używaj `STRING`** (lub Hibernate 6 `@JdbcTypeCode` dla natywnego enum PG).

### equals / hashCode dla encji — subtelna pułapka

Naiwne `equals`/`hashCode` na `id` łamią się, bo `id` jest **null dla transient** i zmienia się po `persist`:

```java
Set<Author> set = new HashSet<>();
Author a = new Author();      // id == null
set.add(a);                   // hash liczony z null
em.persist(a);                // id staje się np. 42 → hash się ZMIENIA
set.contains(a);              // ❌ false! obiekt „zgubiony" w koszyku
```

**Zasady (Vlad Mihalcea):**
- Jeśli istnieje **klucz biznesowy/naturalny** (np. `isbn`, `email`, `uuid`) — użyj go w `equals`/`hashCode`.
- Jeśli nie — użyj `id` w `equals`, ale **stały `hashCode`** (np. `return getClass().hashCode();` lub stała).
  Wtedy hash nie zmienia się przy `persist`, a `equals` porównuje `id` (z ochroną na null).
- Uwzględnij proxy: porównuj przez `Hibernate.getClass(this)`, nie `getClass()`.

### Cache drugiego poziomu (L2)

Opcjonalny, współdzielony **między sesjami** (nie per-EntityManager jak L1). Wymaga providera (Ehcache,
Infinispan, Caffeine) i `@Cacheable` na encji + `hibernate.cache.use_second_level_cache=true`. Świadomość: pomaga
przy encjach read-mostly, ale grozi **nieświeżymi danymi** i komplikuje spójność w klastrze. Domyślnie wyłączony —
włączaj świadomie.

### cascade i orphanRemoval

- `cascade` — propaguje operacje z rodzica na dzieci: `PERSIST`, `MERGE`, `REMOVE`, `REFRESH`, `DETACH`, `ALL`.
  `CascadeType.ALL` na `@OneToMany` → `persist(author)` zapisze też jego `books`.
- `orphanRemoval = true` — usuwa dziecko z DB, gdy zostanie **usunięte z kolekcji rodzica**
  (`author.getBooks().remove(book)` → `DELETE`). Różni się od `CascadeType.REMOVE` (to kaskada usunięcia rodzica).

## Przykład w praktyce

Encja z relacją dwustronną — kluczowa jest **strona właścicielska (owning)** vs **odwrotna (inverse)**:

```java
@Entity
@Table(name = "books")
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "book_seq")
    @SequenceGenerator(name = "book_seq", sequenceName = "book_seq", allocationSize = 50)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Enumerated(EnumType.STRING)          // ✅ nie ORDINAL
    private BookStatus status;

    @Embedded
    private Money price;                   // @Embeddable value object

    // STRONA WŁAŚCICIELSKA: tu jest klucz obcy (kolumna author_id w tabeli books)
    @ManyToOne(fetch = FetchType.LAZY)    // ✅ LAZY (domyślnie byłby EAGER)
    @JoinColumn(name = "author_id")
    private Author author;
}

@Entity
public class Author {
    // ...
    // STRONA ODWROTNA: mappedBy wskazuje pole "author" w Book;
    // Hibernate NIE zarządza tą kolekcją jako źródłem FK — czyta klucz z Book.author
    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Book> books = new ArrayList<>();

    // ZAWSZE synchronizuj OBIE strony przez helper — inaczej pamięć rozjedzie się z DB:
    public void addBook(Book book) {
        books.add(book);
        book.setAuthor(this);   // ustaw stronę właścicielską — inaczej FK będzie null!
    }
    public void removeBook(Book book) {
        books.remove(book);
        book.setAuthor(null);
    }
}
```

Kluczowa zasada relacji dwukierunkowych:
- **Strona właścicielska** = ta z `@JoinColumn` (bez `mappedBy`), trzyma **klucz obcy**. Tylko jej zmiany są
  zapisywane do FK.
- **Strona odwrotna** = ta z `mappedBy`. Ignorowana przy zapisie FK — służy tylko do nawigacji.
- Musisz **synchronizować obie strony w pamięci** (helper `addBook`), bo Hibernate nie robi tego za Ciebie;
  inaczej graf w pamięci kłamie względem tego, co poleci do bazy.

Demonstracja pełnego cyklu życia + dirty checking + LazyInit:

```java
@Transactional
public void demo(EntityManager em) {
    Author a = new Author();           // TRANSIENT
    a.setName("Bloch");
    em.persist(a);                     // → MANAGED, INSERT odroczony do flush

    Book b = new Book();
    b.setTitle("Effective Java");
    a.addBook(b);                      // synchronizacja obu stron + cascade PERSIST

    a.setName("Joshua Bloch");         // dirty checking → UPDATE przy commit (brak save()!)

    em.flush();                        // INSERT-y i UPDATE lecą teraz do DB
    em.clear();                        // wszystkie encje → DETACHED

    Author d = em.find(Author.class, a.getId());  // znów MANAGED (nowy SELECT)
}   // commit → flush + COMMIT

// Poza transakcją:
Author detached = service.findAuthor(1L);
detached.getBooks().size();            // 💥 LazyInitializationException
```

---

## ✅ Kryteria opanowania
- [ ] Odróżnię JPA (spec) od Hibernate (implementacja) i wskażę `jakarta.persistence`.
- [ ] Wyjaśnię persistence context jako L1 cache + identity map i jego konsekwencje.
- [ ] Narysuję diagram stanów encji i opiszę każde przejście (persist/merge/remove/detach/find/refresh).
- [ ] Wyjaśnię dirty checking (snapshot) i czemu UPDATE działa bez save().
- [ ] Uzasadnię `SEQUENCE` vs `IDENTITY` w kontekście batch insert na PostgreSQL.
- [ ] Zdiagnozuję i naprawię `LazyInitializationException` (JOIN FETCH / EntityGraph / DTO).
- [ ] Poprawnie napiszę `equals`/`hashCode` dla encji z generowanym id.
- [ ] Wskażę stronę owning vs inverse i zsynchronizuję obie strony relacji.

### 🔲 Black-box check
- [ ] Co robi `merge` i czemu NIE modyfikuje przekazanego obiektu, tylko zwraca kopię?
- [ ] Kiedy dokładnie Hibernate wykonuje flush i czemu odracza SQL?
- [ ] Jak działa proxy w lazy loadingu i czemu rzuca `LazyInitializationException`?
- [ ] Czemu `@Enumerated` domyślnie jest ORDINAL i czemu to niebezpieczne?
- [ ] Czemu `IDENTITY` wyłącza batch insert, a `SEQUENCE` nie?
- [ ] Co gwarantuje identity map i dlaczego `a1 == a2` w jednej sesji?

### 🎤 Pytania rekrutacyjne
- [ ] „JPA vs Hibernate — różnica?"
- [ ] „Co to persistence context i cache pierwszego poziomu?"
- [ ] „Wymień stany encji i przejścia."
- [ ] „Czym różni się `persist` od `merge`?"
- [ ] „Co to `LazyInitializationException` i jak ją naprawić?"
- [ ] „Owning vs inverse side — gdzie jest klucz obcy?"
- [ ] „Jak zaimplementujesz `equals`/`hashCode` dla encji?"
- [ ] „Czemu widzisz N+1 i jak z tym walczysz?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/06-persystencja.csv`.*
```
JPA vs Hibernate?;JPA to specyfikacja (Jakarta Persistence), Hibernate to jej najpopularniejsza implementacja/provider.
Jak nazywa się pakiet JPA w Jakarta EE 9+ / Spring Boot 3?;jakarta.persistence (wcześniej javax.persistence).
Co rozwiązuje ORM?;Object-relational impedance mismatch — niezgodność modelu obiektowego (grafy, dziedziczenie) z relacyjnym (tabele, FK).
Czym jest persistence context?;Jednostka pracy EntityManagera: cache pierwszego poziomu (L1) + identity map.
Co gwarantuje identity map?;Jeden obiekt na parę (typ, id) w obrębie sesji — dwa find() zwracają tę samą referencję (a1 == a2).
Wymień 4 stany encji.;Transient, managed/persistent, detached, removed.
Co robi persist()?;Przenosi encję transient → managed i planuje INSERT (odroczony do flush).
Czym różni się persist od merge?;persist działa na transient i modyfikuje obiekt; merge działa na detached i ZWRACA nową managed kopię, oryginał zostaje detached.
Co to dirty checking?;Hibernate robi snapshot managed encji i przy flush porównuje stan — zmiany generują UPDATE bez wywołania save().
Kiedy Hibernate robi flush?;Przed zapytaniem JPQL (AUTO), przy commit transakcji, lub ręcznie em.flush().
Domyślny fetch dla @ManyToOne i @OneToOne?;EAGER.
Domyślny fetch dla @OneToMany i @ManyToMany?;LAZY.
Jaki fetch zaleca się w praktyce?;LAZY wszędzie, a potrzebne dane dociągać przez JOIN FETCH lub EntityGraph.
Jak Hibernate realizuje lazy loading?;Przez proxy (dla @ManyToOne/@OneToOne) i PersistentCollection (dla kolekcji); dane doładowuje SELECT-em przy pierwszym dostępie.
Co to LazyInitializationException?;Błąd przy dostępie do lazy relacji poza otwartą sesją/transakcją — proxy nie ma jak wykonać SELECT.
Jak naprawić LazyInitializationException?;JOIN FETCH / @EntityGraph w transakcji, mapowanie na DTO; NIE zmianą na EAGER ani OSIV.
Czemu SEQUENCE jest lepsza niż IDENTITY na PostgreSQL?;SEQUENCE z allocationSize pobiera id w partiach i umożliwia batch insert; IDENTITY (auto-increment) wymaga id po każdym INSERT i wyłącza batch.
Pułapka @Enumerated?;Domyślny EnumType.ORDINAL zapisuje indeks (0,1,2) i psuje dane przy zmianie enuma — używaj EnumType.STRING.
Strona owning vs inverse relacji?;Owning ma @JoinColumn i trzyma klucz obcy (zapisuje FK); inverse ma mappedBy i służy tylko do nawigacji.
Czemu trzeba synchronizować obie strony relacji?;Hibernate zapisuje FK tylko ze strony owning; helper (addBook) ustawia obie strony, by graf w pamięci zgadzał się z DB.
Pułapka equals/hashCode w encji?;id jest null dla transient i zmienia się po persist — łamie kolekcje hashujące; użyj klucza biznesowego lub stałego hashCode.
Co to orphanRemoval?;Usuwa dziecko z DB, gdy zostanie usunięte z kolekcji rodzica (odmienne od CascadeType.REMOVE).
Czym jest cache drugiego poziomu?;Opcjonalny cache współdzielony między sesjami (Ehcache/Infinispan), włączany świadomie; grozi nieświeżymi danymi.
Co robi cascade?;Propaguje operacje (PERSIST/MERGE/REMOVE...) z encji rodzica na powiązane encje.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/06-persystencja/transakcje]] · [[wiedza/06-persystencja/n-plus-1]] · [[wiedza/06-persystencja/spring-data-jpa]]
