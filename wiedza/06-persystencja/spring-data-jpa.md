---
temat: "Spring Data JPA"
faza: 6
status: nieopanowany
priorytet: 🔴
tags: [java, spring, jpa, bazy]
powiazane: ["[[wiedza/06-persystencja/jpa-hibernate]]", "[[wiedza/06-persystencja/n-plus-1]]", "[[wiedza/05-spring/transakcje]]"]
---

# Spring Data JPA

> **TL;DR:** Spring Data JPA generuje w runtime **proxy** implementujące Twój interfejs repozytorium
> (backing: `SimpleJpaRepository`), więc CRUD, paginację i większość zapytań dostajesz bez pisania implementacji.
> Zapytania powstają z **nazwy metody** (derived queries) albo z `@Query` (JPQL/native). Cena za wygodę:
> łatwo o **N+1**, o niepotrzebne ładowanie całych encji (→ projekcje) i o nieczytelne, przekombinowane nazwy metod.

## 1. Co — definicja i API

**Spring Data JPA** to warstwa nad JPA/Hibernate ([[wiedza/06-persystencja/jpa-hibernate]]), której celem jest
**eliminacja boilerplate DAO**. Zamiast pisać `EntityManager em; em.persist(...); em.createQuery(...)` w każdej
klasie DAO, deklarujesz **interfejs** — a Spring dostarcza implementację.

### Hierarchia interfejsów (co dokłada każdy poziom)
```
Repository<T, ID>                     ← marker, zero metod (tylko typowanie T + ID)
  └─ CrudRepository<T, ID>            ← save, saveAll, findById, existsById, findAll, count, deleteById, delete...
       └─ ListCrudRepository<T, ID>   ← (Boot 3) warianty zwracające List zamiast Iterable
       └─ PagingAndSortingRepository  ← findAll(Sort), findAll(Pageable)  → paginacja + sortowanie
            └─ JpaRepository<T, ID>   ← flush(), saveAndFlush(), deleteAllInBatch(), getReferenceById(), List-owe findAll
```
- **`Repository`** — pusty marker; służy tylko do parametryzacji typem encji i typem klucza.
- **`CrudRepository`** — podstawowy CRUD.
- **`PagingAndSortingRepository`** — dokłada paginację i sortowanie (`Pageable`, `Sort`).
- **`JpaRepository`** — najbogatszy, JPA-specyficzny: `flush()`, operacje *batch* (`deleteAllInBatch`),
  `getReferenceById` (leniwe proxy zamiast `findById`), zwracanie `List` zamiast `Iterable`. W praktyce
  99% repozytoriów dziedziczy po **`JpaRepository`**.

### Minimalny przykład
```java
public interface UserRepository extends JpaRepository<User, Long> {
    // to wystarczy — CRUD + paginacja dostępne od ręki
    List<User> findByLastName(String lastName);   // derived query
}
```
Żadnej klasy `UserRepositoryImpl`. Spring Boot 3 wykrywa interfejs przez `@EnableJpaRepositories`
(auto-config skanuje pakiet aplikacji) i **wstrzykuje gotowy bean**.

## 2. Jak — proxy repo i parser nazw pod spodem

### Skąd bierze się implementacja — dynamic proxy
Nie ma pliku z implementacją. Podczas startu kontekstu:
1. `RepositoryFactoryBean` skanuje interfejsy rozszerzające `Repository`.
2. Dla każdego tworzy **JDK dynamic proxy** implementujące ten interfejs.
3. Proxy ma **backing object** — `SimpleJpaRepository<T, ID>` (domyślna implementacja `JpaRepository`,
   trzymająca `EntityManager`). Metody CRUD delegują wprost do niej (`findById` → `em.find`, `save` → persist/merge).
4. Metody, których `SimpleJpaRepository` nie ma (Twoje `findByLastName...`), przechodzą przez łańcuch
   **`QueryExecutorMethodInterceptor`**, który podpina wcześniej zbudowane `RepositoryQuery`.

Czyli: **wywołanie metody repo → proxy → albo delegacja do `SimpleJpaRepository`, albo wykonanie zapytania
zbudowanego z nazwy/`@Query`.**

### Derived query methods — parser nazw
Przy starcie (fail-fast!) `PartTree` **parsuje nazwę metody** na drzewo i buduje z niego JPQL.
Gramatyka: `[czasownik][Distinct][TopN]By[Property][Operator][And|Or]...OrderBy...`.

| Nazwa metody | Wygenerowane (uproszczone) |
|---|---|
| `findByLastNameAndAgeGreaterThan(String, int)` | `... where lastName = ?1 and age > ?2` |
| `findTop3ByOrderByCreatedDesc()` | `... order by created desc` + `LIMIT 3` |
| `findDistinctByStatus(Status)` | `select distinct ... where status = ?1` |
| `existsByEmail(String)` | `select count>0 ... where email = ?1` → `boolean` |
| `countByActiveTrue()` | `select count(*) ... where active = true` |
| `deleteByCreatedBefore(Instant)` | `delete ... where created < ?1` |

Słowa kluczowe: `And/Or`, `Between`, `LessThan/GreaterThan`, `Like/Containing/StartingWith`, `In`,
`IsNull/IsNotNull`, `True/False`, `IgnoreCase`, `OrderBy...Asc/Desc`.
**Fail-fast:** literówka w nazwie property (`findByLstName`) wywala aplikację **przy starcie**, nie w runtime.

**Kiedy nazwa staje się nieczytelna:** `findByFirstNameAndLastNameAndAgeBetweenAndStatusInOrderByCreatedDesc`
to sygnał, że trzeba przejść na `@Query`, **Specifications** albo Querydsl. Reguła kciuka: > ~3 warunki → nazwa
przestaje być samo-dokumentująca i staje się kruchym „stringiem w nazwie metody".

### `@Query` — JPQL i native
```java
@Query("select u from User u where u.email = :email")          // JPQL (nad encjami, nie tabelami)
Optional<User> findByEmail(@Param("email") String email);

@Query(value = "select * from users where email = ?1", nativeQuery = true)  // czysty SQL
Optional<User> findByEmailNative(String email);
```
- **Parametry pozycyjne** (`?1`, `?2`) vs **nazwane** (`:email` + `@Param("email")`). Nazwane są czytelniejsze
  i odporne na zmianę kolejności — preferuj je.
- **`nativeQuery = true`** — omija JPQL, jedzie prosto do dialektu bazy (funkcje specyficzne dla DB,
  hinty, `RETURNING`). Cena: brak przenośności i słabsze mapowanie.

### `@Modifying` — UPDATE/DELETE
```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("update User u set u.active = false where u.lastLogin < :cutoff")
int deactivateStale(@Param("cutoff") Instant cutoff);   // zwraca liczbę zmienionych wierszy
```
`@Modifying` mówi, że to zapytanie zmieniające (nie `SELECT`). Uwaga: **bulk update/delete omija persistence
context** — encje już załadowane w cache 1. poziomu **nie zostaną odświeżone**. `clearAutomatically = true`
czyści kontekst po zapytaniu (kolejny odczyt pobierze świeże dane); `flushAutomatically = true` wypycha
oczekujące zmiany przed wykonaniem. Wymaga aktywnej transakcji.

### Paginacja i sortowanie — Page vs Slice
```java
Page<User>  findByStatus(Status s, Pageable pageable);   // + osobny COUNT(*)
Slice<User> findByStatus(Status s, Pageable pageable);   // bez COUNT
List<User>  findByStatus(Status s, Sort sort);
```
- **`Pageable`** = numer strony + rozmiar + `Sort` (`PageRequest.of(0, 20, Sort.by("created").descending())`).
- **`Page`** zna `getTotalElements()`/`getTotalPages()` — bo wykonuje **dodatkowe zapytanie `COUNT(*)`**.
- **`Slice`** wie tylko, czy **jest następna strona** (`hasNext()`) — pobiera `size+1` wierszy, **bez COUNT**.
  Dla „infinite scroll" / dużych tabel `Slice` jest **znacznie tańszy** (COUNT na milionach wierszy potrafi boleć).

### Specifications — dynamiczne zapytania (Criteria API)
```java
public interface UserRepository extends JpaRepository<User, Long>,
                                        JpaSpecificationExecutor<User> { }

Specification<User> spec = (root, query, cb) ->
    cb.and(cb.equal(root.get("status"), Status.ACTIVE),
           cb.greaterThan(root.get("age"), 18));
List<User> result = userRepository.findAll(spec);   // + wersje z Pageable/Sort
```
`JpaSpecificationExecutor` daje `findAll(Specification)`. `Specification` buduje predykaty przez **Criteria API**
(typo-bezpieczne, programowe). Idealne, gdy **filtry są opcjonalne i dynamiczne** (formularz wyszukiwania:
raz filtrujesz po statusie, raz po wieku, raz po obu). Alternatywa: klejenie stringów JPQL — brzydkie i podatne
na błędy. Wadą Criteria jest gadatliwość; przy bardzo złożonych zapytaniach rozważ **Querydsl**.

### Projekcje — nie ładuj całych encji
Cel: pobrać tylko potrzebne kolumny (mniej I/O, brak inicjalizacji leniwych asocjacji → mniej **N+1**,
zob. [[wiedza/06-persystencja/n-plus-1]]).
```java
// 1) Interfejsowa CLOSED — Spring generuje proxy zwracające tylko te gettery (SELECT tylko tych kolumn)
interface UserView { String getFirstName(); String getLastName(); }

// 2) Interfejsowa OPEN — @Value/SpEL łączy pola; ŁADUJE CAŁĄ encję (traci optymalizację SELECT)
interface UserFull { @Value("#{target.firstName + ' ' + target.lastName}") String getFullName(); }

// 3) Klasowa (DTO) — mapowanie po KONSTRUKTORZE (kolejność i typy muszą pasować)
record UserDto(String firstName, String lastName) {}

List<UserView> findByStatus(Status s);            // closed projection
List<UserDto>  findAllProjectedBy();              // DTO
```
- **Closed** (tylko gettery pól) → Spring/Hibernate potrafi wygenerować **wąski `SELECT`** (tylko te kolumny).
- **Open** (`@Value`/SpEL na wyliczanych polach) → **ładuje całą encję**, więc traci zysk z wąskiego SELECT.
- **DTO klasowe** → w JPQL można wprost: `select new com.app.UserDto(u.firstName, u.lastName) from User u`.

### `@EntityGraph` — walka z N+1 na poziomie metody
```java
@EntityGraph(attributePaths = {"roles", "address"})
List<User> findByStatus(Status s);   // roles/address dociągnięte JOIN-em w jednym zapytaniu
```
Deklaruje, które asocjacje **fetch-ować od razu** (fetch join) dla tej konkretnej metody — leczy N+1 bez
zmieniania globalnego `FetchType` encji. Szczegóły problemu: [[wiedza/06-persystencja/n-plus-1]].

### Audyt
```java
@Configuration @EnableJpaAuditing
class JpaConfig { @Bean AuditorAware<String> auditor() { return () -> Optional.of(currentUser()); } }

@Entity @EntityListeners(AuditingEntityListener.class)
class Article {
    @CreatedDate     Instant createdAt;
    @LastModifiedDate Instant updatedAt;
    @CreatedBy       String  createdBy;   // z AuditorAware
    @LastModifiedBy  String  updatedBy;
}
```
`@EnableJpaAuditing` + `AuditingEntityListener` automatycznie wypełniają pola przy `persist`/`update`.
`@CreatedBy`/`@LastModifiedBy` biorą wartość z beana `AuditorAware` (zwykle z Spring Security).

### Transakcyjność metod repo (domyślnie)
Metody `SimpleJpaRepository` są **`@Transactional`** — odczyty jako `readOnly = true`, modyfikacje jako
zwykła transakcja. Jeśli wołasz repo **bez własnej transakcji**, każde wywołanie to osobna transakcja
(a leniwe asocjacje odczytane później rzucą `LazyInitializationException`). W realnych use-case'ach
transakcję otwierasz **na poziomie serwisu** (`@Transactional` na metodzie serwisowej — [[wiedza/05-spring/transakcje]]),
żeby wiele operacji repo dzieliło jeden persistence context i jeden commit/rollback.

## 3. Dlaczego / kiedy — projekcje, specyfikacje, pułapki

- **`save()` = persist LUB merge — zależnie od stanu encji.** Dla nowej encji (klucz `null`/oznaczona jako nowa)
  Hibernate robi `persist`; dla encji z ustawionym kluczem — `merge` (który **wykonuje SELECT** i **zwraca nową,
  zarządzaną kopię**). Stąd: zawsze używaj obiektu **zwróconego** przez `save()`, nie oryginalnego argumentu.
  Przy encjach z ręcznie przypisanym ID `save()` może zrobić zbędny SELECT (traktuje jako potencjalny update).
- **`deleteAll` w pętli / `deleteById` w pętli** → **N zapytań DELETE** (+ ewentualnie SELECT-y). Dla masowych
  usunięć użyj `deleteAllInBatch()` (jedno `DELETE`, ale **omija kaskady i lifecycle callbacks**) albo
  `@Modifying @Query("delete ...")`. Świadomie wybierz: batch = szybko ale bez kaskad; pętla = wolno ale z regułami JPA.
- **Nadużycie derived methods** — długie `findBy...And...And...` są kruche (refactor property = cichy błąd
  kompilacji tylko w Boot z przetwarzaniem nazw), nieczytelne i nie dają się złożyć dynamicznie. Granica:
  proste 1–3 warunki → derived; złożone/stałe → `@Query`; dynamiczne/opcjonalne → **Specifications**/Querydsl.
- **N+1 z automatu.** Domyślnie `@ManyToOne` jest EAGER, `@OneToMany` LAZY; pętla po encjach dotykająca
  leniwych asocjacji generuje lawinę zapytań. Leki: `@EntityGraph`, fetch join w `@Query`, projekcje DTO
  ([[wiedza/06-persystencja/n-plus-1]]).
- **Projekcje zamiast encji** — jeśli endpoint zwraca 3 pola, nie ładuj 20-kolumnowej encji z relacjami.
  Closed projection / DTO = mniej danych z bazy, brak dirty-checkingu, brak ryzyka lazy-load.
- **`Page` na wielkich tabelach** — `COUNT(*)` bywa najdroższą częścią zapytania; jeśli UI nie potrzebuje
  „strona X z Y", użyj `Slice`.
- **Bulk `@Modifying` a cache 1. poziomu** — omija persistence context; bez `clearAutomatically` odczytasz
  nieaktualne encje.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Endpoint „lista aktywnych użytkowników z paginacją, zwracany jako lekkie DTO":
```java
@Entity
class User {
    @Id @GeneratedValue Long id;
    String firstName, lastName, email;
    @Enumerated(EnumType.STRING) Status status;
    @ManyToOne(fetch = FetchType.LAZY) Department department;
    Instant created;
}

record UserSummary(String firstName, String lastName, String departmentName) {}

interface UserRepository extends JpaRepository<User, Long>, JpaSpecificationExecutor<User> {

    // derived query — proste, czytelne
    List<User> findByStatusAndLastNameStartingWithIgnoreCase(Status s, String prefix);

    // @Query z DTO + fetch, żeby nie było N+1 na department
    @Query("""
           select new com.app.UserSummary(u.firstName, u.lastName, u.department.name)
           from User u
           where u.status = :status
           """)
    Page<UserSummary> findActiveSummaries(@Param("status") Status status, Pageable pageable);

    @Modifying(clearAutomatically = true)
    @Query("update User u set u.status = :s where u.created < :cutoff")
    int archiveOlderThan(@Param("s") Status s, @Param("cutoff") Instant cutoff);
}

// serwis — transakcja tu, nie w repo
@Service
class UserService {
    private final UserRepository repo;
    UserService(UserRepository repo) { this.repo = repo; }

    @Transactional(readOnly = true)
    Page<UserSummary> active(int page) {
        Pageable p = PageRequest.of(page, 20, Sort.by("lastName").ascending());
        return repo.findActiveSummaries(Status.ACTIVE, p);
    }
}
```
Efekt: zero klasy `Impl`, wąski `SELECT` (tylko 3 kolumny w DTO, `department` bez osobnego zapytania),
paginacja z jednym `COUNT`, a masowa archiwizacja jednym `UPDATE` zamiast N `save()`-ów.

---

## ✅ Kryteria opanowania
- [ ] Narysuję hierarchię `Repository → CrudRepository → PagingAndSortingRepository → JpaRepository` i powiem, co dokłada każdy poziom.
- [ ] Wyjaśnię, że implementacji nie piszę — Spring tworzy proxy z backingiem `SimpleJpaRepository`.
- [ ] Zbuduję z głowy derived query i wiem, kiedy nazwa staje się nieczytelna.
- [ ] Rozróżnię `@Query` JPQL vs native, parametry pozycyjne vs nazwane, i wiem po co `@Modifying`/`clearAutomatically`.
- [ ] Wyjaśnię różnicę `Page` vs `Slice` (dodatkowy COUNT) i kiedy który.
- [ ] Wiem, kiedy sięgnąć po Specifications, a kiedy po projekcję DTO/`@EntityGraph`.
- [ ] Rozumiem, że `save` = persist LUB merge, i unikam `deleteAll` w pętli.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Skąd bierze się instancja repozytorium, skoro nie ma klasy `Impl`? (dynamic proxy + `SimpleJpaRepository`)
- [ ] Jak z `findByLastNameAndAgeGreaterThan` powstaje JPQL? (`PartTree`/parser nazw, przy starcie)
- [ ] Dlaczego literówka w nazwie property wywala aplikację przy starcie, a nie w runtime?
- [ ] Co realnie robi `Page` czego nie robi `Slice`? (dodatkowy `COUNT(*)`)
- [ ] Dlaczego bulk `@Modifying` bez `clearAutomatically` zwraca „stare" encje? (omija persistence context)
- [ ] Czemu trzeba używać obiektu zwróconego przez `save()`? (merge zwraca nową zarządzaną kopię)

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Kto pisze implementację `UserRepository`?" (nikt — proxy w runtime, backing `SimpleJpaRepository`)
- [ ] „Czym się różni `CrudRepository` od `JpaRepository`?"
- [ ] „`Page` vs `Slice` — kiedy który i dlaczego?"
- [ ] „Kiedy derived methods, a kiedy `@Query` / Specifications?"
- [ ] „`save()` robi INSERT czy UPDATE?" (persist vs merge zależnie od stanu encji)
- [ ] „Jak walczysz z N+1 na poziomie repozytorium?" (`@EntityGraph`, fetch join, projekcje DTO)

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Główny cel Spring Data JPA?;Eliminacja boilerplate DAO — deklarujesz interfejs, implementację dostarcza framework.
Kolejność hierarchii interfejsów Spring Data JPA?;Repository → CrudRepository → PagingAndSortingRepository → JpaRepository.
Co dokłada PagingAndSortingRepository?;Paginację i sortowanie: findAll(Sort) i findAll(Pageable).
Co dokłada JpaRepository ponad PagingAndSortingRepository?;flush/saveAndFlush, operacje batch (deleteAllInBatch), getReferenceById, zwracanie List zamiast Iterable.
Kto pisze implementację interfejsu repozytorium?;Nikt — Spring tworzy w runtime dynamic proxy z backingiem SimpleJpaRepository.
Czym jest SimpleJpaRepository?;Domyślna implementacja JpaRepository (trzyma EntityManager), do której proxy deleguje metody CRUD.
Jak z nazwy metody powstaje zapytanie?;Parser (PartTree) rozbija nazwę na drzewo predykatów i buduje JPQL — przy starcie kontekstu.
Co robi findTop3ByOrderByCreatedDesc?;Sortuje malejąco po created i zwraca 3 pierwsze rekordy (LIMIT 3).
existsBy... i countBy... zwracają?;existsBy → boolean (count>0), countBy → long (liczba rekordów).
Kiedy porzucić derived methods na rzecz @Query/Specifications?;Gdy nazwa robi się nieczytelna (>~3 warunki) lub gdy filtry są dynamiczne/opcjonalne.
JPQL vs nativeQuery=true?;JPQL działa nad encjami i jest przenośny; nativeQuery to czysty SQL dialektu bazy (mniej przenośny, DB-specific).
Parametry pozycyjne vs nazwane?;Pozycyjne ?1/?2 wg kolejności; nazwane :name + @Param — czytelniejsze i odporne na zmianę kolejności.
Po co @Modifying i clearAutomatically?;@Modifying oznacza UPDATE/DELETE; clearAutomatically czyści persistence context, by nie czytać nieaktualnych encji z cache 1. poziomu.
Różnica Page vs Slice?;Page wykonuje dodatkowy COUNT(*) i zna total; Slice wie tylko czy jest następna strona (size+1), bez COUNT — tańszy.
Kiedy użyć JpaSpecificationExecutor / Specification?;Do dynamicznych, opcjonalnych filtrów budowanych programowo przez Criteria API.
Closed vs open projection?;Closed = tylko gettery pól, generuje wąski SELECT; open = @Value/SpEL, ładuje całą encję (traci optymalizację).
Po co projekcje DTO?;Pobierać tylko potrzebne kolumny — mniej I/O, brak dirty-checkingu, mniejsze ryzyko N+1/lazy-load.
Do czego @EntityGraph na metodzie repo?;Deklaruje asocjacje do fetch-owania od razu (JOIN) dla tej metody — leczy N+1 bez zmiany globalnego FetchType.
Jak włączyć audyt i które adnotacje pól?;@EnableJpaAuditing + AuditingEntityListener; pola @CreatedDate/@LastModifiedDate/@CreatedBy/@LastModifiedBy (@CreatedBy z AuditorAware).
Czy metody repo są transakcyjne?;Tak, domyślnie — odczyty jako readOnly=true; w praktyce transakcję otwiera się na poziomie serwisu.
save() to INSERT czy UPDATE?;Zależnie od stanu encji: nowa → persist (INSERT), z kluczem → merge (SELECT+UPDATE), zwraca nową zarządzaną kopię.
Dlaczego deleteAll w pętli jest złe?;Generuje N zapytań DELETE; dla mas użyj deleteAllInBatch lub @Modifying delete (ale to omija kaskady/callbacki).
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/06-persystencja/jpa-hibernate]] · [[wiedza/06-persystencja/n-plus-1]] · [[wiedza/05-spring/transakcje]]
