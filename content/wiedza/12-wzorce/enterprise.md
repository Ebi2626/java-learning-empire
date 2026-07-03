---
temat: "Wzorce enterprise / aplikacyjne (PoEAA) w Springu"
faza: 12
status: nieopanowany
priorytet: 🟡
tags: [java, wzorce, enterprise, spring]
powiazane: ["[[wiedza/06-persystencja/spring-data-jpa]]", "[[wiedza/06-persystencja/jpa-hibernate]]", "[[wiedza/05-spring/ioc-di]]", "[[wiedza/07-api-design/dto-mapstruct]]", "[[wiedza/09-architektura/ddd-lite]]"]
---

# Wzorce enterprise / aplikacyjne (PoEAA) w Springu

> **TL;DR:** Większość wzorców z *Patterns of Enterprise Application Architecture* (Fowler) w backendzie Spring
> **realizuje za Ciebie framework**: **Repository/DAO** (Spring Data), **Unit of Work + Identity Map + Lazy Loading**
> (Hibernate persistence context), **Dependency Injection** (kontener IoC), **Optimistic/Pessimistic Locking**
> (`@Version` / `PESSIMISTIC_WRITE`), **Mapper** (MapStruct). Twoja robota to złożyć je poprawnie w warstwach
> **Controller → Service (`@Transactional`) → Repository**, z **DTO** na granicy API — i **rozumieć, co dzieje się
> pod spodem**, żeby framework nie był black-boxem (skąd N+1, kiedy leci `UPDATE`, dlaczego `OptimisticLockException`).

## 1. Co — definicja i API

**PoEAA** to katalog wzorców organizacji logiki i dostępu do danych w aplikacjach warstwowych. W typowym
Spring Boot 3 / Java 21 backendzie spotykasz codziennie garść z nich — często nie zauważając, bo są schowane
w adnotacjach i interfejsach frameworka.

Trzy **style organizacji logiki biznesowej** (to fundamentalny wybór — reszta wzorców się do niego dopina):

- **Transaction Script** — jedna procedura na przypadek użycia, logika proceduralnie w Service, encje anemiczne
  (same gettery/settery). Prosto, świetne dla CRUD i prostych flow; puchnie przy złożonej domenie.
- **Domain Model** — logika **w encjach/agregatach** (metody na obiektach domeny), Service tylko orkiestruje
  i wyznacza transakcję. Bogaty model, skaluje się z złożonością — podstawa DDD ([[wiedza/09-architektura/ddd-lite]]).
- **Table Module** — jedna instancja obsługuje **całą tabelę** (nie jeden rekord); rzadkie w Javie/Spring,
  bliższe światu ADO.NET/DataSet.

**Value Object** — obiekt bez tożsamości, porównywany przez wartość, niemutowalny (np. `Money`, `Email`).
W Javie 21 idealny jako **`record`** (albo `@Embeddable` w JPA). Kontrast: **Entity** ma tożsamość (id) i cykl życia.

## 2. Jak — wzorce realizowane przez Spring/Hibernate

Sedno notatki: **te wzorce działają ZA CIEBIE**, dlatego trzeba wiedzieć, co robią pod spodem.

- **Repository** — abstrakcja dostępu do danych jako *kolekcji agregatów* (`save`, `findById`, `delete`).
  Akcent DDD: „kolekcja obiektów domeny", nie „opakowanie SQL". W Springu: `JpaRepository<T, ID>` —
  interfejs, którego implementację generuje Spring Data w locie z nazw metod (`findByStatusAndEmail`).
  Szczegóły: [[wiedza/06-persystencja/spring-data-jpa]].
- **DAO (Data Access Object)** — również enkapsuluje dostęp do danych, ale **akcent na źródło/technologię**
  (jedna tabela, jeden `EntityManager`, operacje techniczne). Różnica głównie w akcencie: DAO ≈ „obiekt do gadania
  z bazą", Repository ≈ „kolekcja agregatów w języku domeny". W praktyce Spring Data zaciera granicę — jeden interfejs
  gra obie role.
- **Unit of Work** — śledzi zmiany obiektów w ramach jednej logicznej operacji i wypycha je **jedną transakcją**
  (kolejność INSERT/UPDATE/DELETE, `flush` na końcu). Realizuje to **persistence context Hibernate**: modyfikujesz
  encję zarządzaną (managed) — Hibernate przy `flush`/commit sam wykrywa **dirty checking** i generuje `UPDATE`.
  Nie wołasz `save` po każdej zmianie. Szczegóły: [[wiedza/06-persystencja/jpa-hibernate]].
- **Identity Map** — gwarancja „jeden obiekt Java na jedno id w obrębie sesji". Realizuje to **cache L1** Hibernate
  (persistence context): dwa `findById(1L)` w jednej transakcji zwracają **tę samą instancję** (`==`), bez drugiego
  zapytania. Stąd też: encji nie da się zdublować w sesji.
- **Lazy Loading** — kolekcje/relacje ładowane dopiero przy pierwszym dostępie (proxy Hibernate). Domyślne dla
  `@OneToMany`/`@ManyToMany`. Zaleta: nie ciągniesz całego grafu; pułapka: **N+1 select** i `LazyInitializationException`
  poza sesją. Kontrola przez `JOIN FETCH` / `@EntityGraph`. Szczegóły: [[wiedza/06-persystencja/n-plus-1]].
- **Dependency Injection** — komponenty (`@Service`, `@Repository`, `@Component`) nie tworzą zależności same,
  dostają je z kontenera IoC (najlepiej **wstrzykiwanie przez konstruktor**). To sprzątanie po sobie po stronie
  frameworka: cykl życia beanów, singletony, proxy transakcyjne. Szczegóły: [[wiedza/05-spring/ioc-di]].
- **Mapper** — osobny obiekt tłumaczący między modelami (Entity ↔ DTO), żeby nie mieszać warstw. W Springu:
  **MapStruct** generuje implementację mappera w compile-time (bez refleksji, szybko). Szczegóły:
  [[wiedza/07-api-design/dto-mapstruct]].
- **DTO (Data Transfer Object)** — płaski obiekt do przenoszenia danych przez granicę (API/warstwy), żeby **nie
  wystawiać encji** (lazy proxy, cykle, przypadkowe pola, coupling z schematem bazy). W Javie 21 zwykle `record`.
  Szczegóły: [[wiedza/07-api-design/dto-mapstruct]].
- **Specification** — kryterium zapytania jako **obiekt** (składalne AND/OR), zamiast mnożenia metod
  `findByAAndBAndC`. W Springu: **Spring Data JPA Specifications** (`Specification<T>` + `JpaSpecificationExecutor`)
  budujące dynamiczne predykaty Criteria API.
- **Gateway / Adapter** — obiekt-fasada nad zewnętrznym systemem (REST innej usługi, kolejka, płatności),
  izolujący domenę od szczegółów integracji. W Springu: klient (`RestClient`/`WebClient`/Feign) opakowany
  we własny interfejs; wzorzec kluczowy w architekturze heksagonalnej (port + adapter).

## 3. Dlaczego / kiedy — dobór, locking, CQRS

**Active Record vs Repository (czemu w JPA raczej Repository).** *Active Record* to encja, która **sama** umie się
zapisać (`user.save()`, metody CRUD na obiekcie domeny — jak w Railsach). W Javie z JPA dominuje rozdzielenie:
**Entity = stan/logika domenowa**, **Repository = persystencja**. Powody: testowalność (mockujesz Repository),
brak mieszania odpowiedzialności, oraz to, że Hibernate i tak zarządza cyklem życia encji przez persistence context —
encja nie musi znać `EntityManager`. (Spring Data ma `@DomainEvents`/`AbstractAggregateRoot`, ale to nie klasyczny
Active Record.)

**Dobór stylu:** prosty CRUD/raporty → **Transaction Script** (nie przeinżynieruj). Złożone reguły, inwarianty,
agregaty → **Domain Model** + DDD. Nie zaczynaj od bogatej domeny „na zapas".

**Locking — spójność przy współbieżności** (szczegóły SQL: [[wiedza/06-persystencja/sql-podstawy]], granice:
[[wiedza/06-persystencja/transakcje]]):

- **Optimistic Locking** — zakładasz brak konfliktu; kolumna **`@Version`** (int/timestamp). Przy `UPDATE`
  Hibernate dokłada `WHERE version = ?` i podbija wersję. Jeśli ktoś zmienił rekord w międzyczasie → 0 wierszy →
  **`OptimisticLockException`** (retry/409). Tanie, bez blokad w bazie; dobre dla większości aplikacji webowych
  (mało realnych kolizji, długie „read-modify-write" przez UI).
- **Pessimistic Locking** — blokujesz rekord od razu: `SELECT ... FOR UPDATE`
  (`@Lock(PESSIMISTIC_WRITE)` na metodzie repozytorium lub `LockModeType`). Inne transakcje czekają. Dla
  krótkich, sekwencyjnych operacji o wysokiej kontencji (np. rezerwacja ostatniej sztuki). Koszt: blokady,
  ryzyko deadlocków, gorsza skalowalność.

**CQRS (Command Query Responsibility Segregation)** — rozdzielenie **modelu zapisu** (komendy, bogaty
Domain Model, transakcje) od **modelu odczytu** (zapytania, płaskie read-model/projekcje, często natywny SQL
lub osobna baza). Warto, gdy odczyty i zapisy mają **radykalnie różne wymagania** (skalowanie, kształt danych,
złożone raporty). Nie warto na starcie w prostym CRUD — dokłada spójność ostateczną i infrastrukturę.
Lekka wersja: te same encje do zapisu, ale osobne **DTO/projekcje** (interfejsowe/`record`) do odczytu.

**Główna teza:** framework robi Repository/UoW/Identity Map/Lazy/DI za Ciebie — ale to znaczy, że musisz umieć
**zajrzeć do środka**: kiedy leci `flush`, skąd N+1, czemu encja jest „stale", dlaczego `LazyInitializationException`.
Bez tego debugowanie produkcji to zgadywanka.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Klasyczny przepływ Spring Boot 3 / Java 21 — **Controller → Service(`@Transactional`) → Repository**, z DTO i mapperem:

```java
// --- Value Object jako record (immutable) ---
public record Money(BigDecimal amount, String currency) {}

// --- DTO na granicy API (nie wystawiamy encji) ---
public record OrderDto(Long id, String status, Money total) {}

// --- Mapper (MapStruct generuje impl w compile-time) ---
@Mapper(componentModel = "spring")
public interface OrderMapper {
    OrderDto toDto(Order order);
}

// --- Repository (Spring Data implementuje interfejs) ---
public interface OrderRepository
        extends JpaRepository<Order, Long>, JpaSpecificationExecutor<Order> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)          // SELECT ... FOR UPDATE
    Optional<Order> findWithLockById(Long id);
}

// --- Service = Service Layer + granica transakcji (Unit of Work) ---
@Service
@RequiredArgsConstructor                            // DI przez konstruktor
public class OrderService {

    private final OrderRepository orders;           // wstrzyknięte przez kontener IoC
    private final OrderMapper mapper;

    @Transactional                                  // start UoW / persistence context
    public OrderDto confirm(Long id) {
        Order order = orders.findById(id)           // Identity Map (L1) + Lazy proxy
                .orElseThrow(OrderNotFound::new);
        order.confirm();                            // Domain Model: logika w encji
        // brak jawnego save() — dirty checking wygeneruje UPDATE przy commit
        // @Version → optimistic lock: WHERE version=? , przy konflikcie OptimisticLockException
        return mapper.toDto(order);                 // Entity → DTO, zanim sesja się zamknie
    }
}

@RestController
@RequiredArgsConstructor
class OrderController {
    private final OrderService service;

    @PostMapping("/orders/{id}/confirm")
    OrderDto confirm(@PathVariable Long id) {        // dostaje/oddaje DTO, nie encję
        return service.confirm(id);
    }
}
```

Co tu robi framework **za Ciebie**: `@Transactional` otwiera Unit of Work (persistence context) i domyka transakcję;
`findById` korzysta z Identity Map (L1); zmiana `order.confirm()` trafia do bazy przez dirty checking bez `save()`;
`@Version` daje optimistic locking; MapStruct mapuje na DTO; kontener IoC wstrzykuje zależności.

---

## ✅ Kryteria opanowania
- [ ] Wymienię wzorce PoEAA realizowane przez Spring/Hibernate i wskażę, **gdzie** (adnotacja/interfejs).
- [ ] Wyjaśnię różnicę **Repository vs DAO** (akcent domenowy vs techniczny).
- [ ] Opiszę, jak **Unit of Work + Identity Map + Lazy Loading** = persistence context Hibernate.
- [ ] Uzasadnię **Optimistic vs Pessimistic locking** i wskażę mechanizm (`@Version` / `FOR UPDATE`).
- [ ] Wybiorę styl **Transaction Script vs Domain Model** dla danego przypadku.
- [ ] Wyjaśnię, czemu w JPA **Repository, nie Active Record** i po co **DTO** na granicy.
- [ ] Powiem, kiedy **CQRS** się opłaca, a kiedy to overengineering.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Kiedy Hibernate wykonuje `flush` i jak działa **dirty checking** (czemu nie trzeba `save`)?
- [ ] Jak L1 cache realizuje **Identity Map** — pokaż, że `findById` dwa razy = `==`.
- [ ] Skąd bierze się **N+1** przy Lazy Loading i jak go usunąć (`JOIN FETCH`/`@EntityGraph`)?
- [ ] Co dokładnie generuje **optimistic lock** w SQL i kiedy leci `OptimisticLockException`?
- [ ] Jak Spring Data **implementuje** interfejs Repository (proxy + query derivation)?
- [ ] Czemu wystawianie **encji** przez API jest ryzykowne (proxy, cykle, coupling)?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Repository vs DAO — różnica?"
- [ ] „Co to persistence context i jakie wzorce PoEAA realizuje?"
- [ ] „Optimistic czy pessimistic locking — kiedy które?"
- [ ] „Po co DTO, skoro mam encje?"
- [ ] „Transaction Script vs Domain Model — jak wybierasz?"
- [ ] „Czemu w Javie nie stosuje się Active Record jak w Railsach?"
- [ ] „Kiedy sięgasz po CQRS?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/12-wzorce.csv`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
PoEAA to?;Katalog wzorców enterprise Martina Fowlera (Patterns of Enterprise Application Architecture) — organizacja logiki i dostępu do danych.
Wzorzec Repository — problem i rozwiązanie?;Ukrywa dostęp do danych jako kolekcję agregatów domeny; w Springu JpaRepository<T,ID>.
Repository vs DAO — różnica?;Ten sam cel, inny akcent: Repository mówi językiem domeny (kolekcja agregatów), DAO akcentuje technologię/źródło (jedna tabela, EntityManager).
Który wzorzec realizuje persistence context jako "śledzenie zmian i jedna transakcja"?;Unit of Work — Hibernate wykrywa zmiany (dirty checking) i wypycha je jednym commit.
Co to Identity Map i co ją realizuje w Hibernate?;Gwarancja jeden obiekt na jedno id w sesji; realizuje cache L1 (persistence context) — dwa findById zwracają tę samą instancję.
Co to Lazy Loading i jaka jest jego główna pułapka?;Ładowanie relacji dopiero przy dostępie (proxy); pułapka to N+1 select i LazyInitializationException poza sesją.
Czemu nie trzeba wołać save() po zmianie encji zarządzanej?;Bo Unit of Work robi dirty checking — Hibernate porównuje stan i generuje UPDATE przy flush/commit.
Wzorzec Service Layer — po co?;Warstwa przypadków użycia i granica transakcji (@Transactional) między Controllerem a Repository.
Wzorzec DTO — problem i rozwiązanie?;Nie wystawiaj encji (proxy, cykle, coupling ze schematem) — przenoś dane płaskim obiektem (record) przez granicę API.
Wzorzec Mapper w Springu?;Osobny obiekt Entity↔DTO; MapStruct generuje implementację w compile-time (bez refleksji).
Wzorzec Specification w Spring Data?;Kryterium zapytania jako składalny obiekt (Specification<T> + JpaSpecificationExecutor) zamiast mnożenia metod findByAAndB.
Dependency Injection — na czym polega?;Komponent dostaje zależności z kontenera IoC (najlepiej przez konstruktor) zamiast tworzyć je sam.
Transaction Script vs Domain Model?;Transaction Script = logika proceduralnie w Service (encje anemiczne); Domain Model = logika w encjach/agregatach, Service tylko orkiestruje.
Table Module to?;Jedna instancja obsługuje całą tabelę (nie jeden rekord); rzadkie w Javie, bliskie światu ADO.NET.
Active Record vs Repository — czemu w JPA Repository?;Active Record: encja sama się zapisuje; w JPA rozdziela się encję (stan/logika) od Repository (persystencja) — testowalność, brak mieszania odpowiedzialności.
Value Object — czym jest i jak w Javie 21?;Obiekt bez tożsamości, porównywany przez wartość, niemutowalny; idealnie record albo @Embeddable.
Optimistic locking — jak działa?;Kolumna @Version; UPDATE dostaje WHERE version=? i podbija wersję; konflikt = 0 wierszy = OptimisticLockException.
Pessimistic locking — jak i kiedy?;SELECT ... FOR UPDATE (@Lock(PESSIMISTIC_WRITE)); blokuje rekord — dla krótkich operacji o wysokiej kontencji.
Optimistic czy pessimistic domyślnie w webie?;Zwykle optimistic — brak blokad w bazie, tanie, kolizje rzadkie przy read-modify-write przez UI.
Gateway/Adapter — po co?;Fasada nad zewnętrznym systemem (REST, kolejka) izolująca domenę od szczegółów integracji; port+adapter w architekturze heksagonalnej.
CQRS — na czym polega i kiedy warto?;Rozdzielenie modelu zapisu (komendy) i odczytu (projekcje); warto gdy odczyty/zapisy mają radykalnie różne wymagania — nie w prostym CRUD.
Które wzorce PoEAA Spring/Hibernate robi za Ciebie?;Repository/DAO (Spring Data), Unit of Work+Identity Map+Lazy Loading (persistence context), DI (IoC), locking (@Version/FOR UPDATE), Mapper (MapStruct).
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/06-persystencja/spring-data-jpa]] · [[wiedza/06-persystencja/jpa-hibernate]] · [[wiedza/06-persystencja/transakcje]] · [[wiedza/06-persystencja/n-plus-1]] · [[wiedza/06-persystencja/sql-podstawy]] · [[wiedza/05-spring/ioc-di]] · [[wiedza/07-api-design/dto-mapstruct]] · [[wiedza/09-architektura/ddd-lite]]
