---
temat: "DTO i mapowanie encja↔DTO (MapStruct)"
faza: 7
status: nieopanowany
priorytet: 🔴
tags: [java, api, mapowanie]
powiazane: ["[[wiedza/06-persystencja/jpa-hibernate]]", "[[wiedza/06-persystencja/n-plus-1]]", "[[wiedza/01-jezyk/record-sealed-enum]]"]
---

# DTO i mapowanie encja↔DTO (MapStruct)

> **TL;DR:** **DTO** (Data Transfer Object) to kształt danych na granicy systemu (API), oddzielony od **encji** JPA — bo encja jest modelem bazy, a nie kontraktem API. **Nigdy nie eksponuj encji w kontrolerze** (`LazyInitializationException`, wyciek pól, mass assignment, sprzężenie API↔schemat). Rekordy Javy = idealne, niemutowalne DTO. **MapStruct** generuje mappery jako **zwykły kod Java w czasie kompilacji** (annotation processor, `@Mapper`) — zero refleksji w runtime, typobezpiecznie, błędy wychodzą przy `javac`, a nie na produkcji.

## 1. Co — definicja i API

**DTO (Data Transfer Object)** to prosty obiekt służący wyłącznie do **przenoszenia danych** przez granicę (HTTP, kolejka, warstwa serwisu). Nie ma logiki domenowej, nie jest zarządzany przez `EntityManager`, nie ma tożsamości z bazą — to „płaski" kontrakt.

**Encja** (`@Entity`) natomiast jest częścią modelu persystencji: ma tożsamość (`@Id`), relacje (`@OneToMany`, lazy proxy), stan zarządzany (managed/detached) i cykl życia związany z transakcją. To **model bazy danych**, nie model API.

Rekord Javy 21 jest idealnym nośnikiem DTO — niemutowalny, zwięzły, z automatycznym `equals`/`hashCode`/`toString` (→ [[wiedza/01-jezyk/record-sealed-enum]]):

```java
// Encja JPA — model bazy, NIE eksponowana w API
@Entity
class User {
    @Id @GeneratedValue Long id;
    String email;
    String passwordHash;      // pole wrażliwe — nigdy do API
    Instant createdAt;
    @OneToMany(fetch = LAZY) List<Order> orders;  // lazy proxy
}

// DTO odpowiedzi — kontrakt API, tylko to, co klient ma zobaczyć
public record UserResponse(Long id, String email, Instant createdAt) {}

// DTO żądania — osobny kształt dla tworzenia
public record CreateUserRequest(String email, String password) {}
```

**Rodzaje DTO** — rozdzielone świadomie:
- **Request vs Response** — inne pola (request nie ma `id`/`createdAt`; response nie ma `password`).
- **Osobne DTO per operacja** — `CreateUserRequest` ≠ `UpdateUserRequest` (przy update część pól opcjonalna/niezmienialna).
- Dzięki temu jeden endpoint nie „przecieka" pól innego.

## 2. Jak — MapStruct jako annotation processor pod spodem

**Problem mapowania:** ręczne przepisywanie pól (`dto.setEmail(entity.getEmail())`, ...) jest żmudne, rozwlekłe i błędogenne — łatwo pominąć pole, pomylić kierunek, zapomnieć zaktualizować po dodaniu kolumny. Kompilator tego nie wyłapie.

**MapStruct** rozwiązuje to **generacją kodu w czasie kompilacji**. Definiujesz **interfejs** (lub klasę abstrakcyjną) z `@Mapper`, a MapStruct — jako **annotation processor** wpięty w `javac` — generuje implementację jako zwykły plik `.java` (`UserMapperImpl`) w `target/generated-sources`:

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserResponse toResponse(User user);            // encja → DTO
    User toEntity(CreateUserRequest req);          // DTO → encja
    List<UserResponse> toResponseList(List<User> users);  // kolekcja
}
```

Wygenerowana implementacja to po prostu **jawne przypisania** — bez refleksji:

```java
// UserMapperImpl.java (wygenerowane)
@Component
public class UserMapperImpl implements UserMapper {
    public UserResponse toResponse(User user) {
        if (user == null) return null;
        return new UserResponse(user.getId(), user.getEmail(), user.getCreatedAt());
    }
    // ...
}
```

**Co to znaczy „pod spodem":**
- **Zero refleksji w runtime** — wygenerowany kod to zwykłe wywołania getterów/konstruktorów. Stąd jest **szybki** (JIT go optymalizuje jak każdy inny kod) i **typobezpieczny**.
- **Błędy przy kompilacji** — jeśli pole `target` nie ma źródła i nie jest zignorowane, `javac` **przerywa build** z komunikatem. Niedopasowanie typów wychodzi natychmiast, nie na produkcji.
- **`componentModel = "spring"`** sprawia, że `UserMapperImpl` dostaje `@Component` → jest **wstrzykiwalnym beanem** (`@Autowired`/konstruktor). Bez tego mapper jest dostępny przez `Mappers.getMapper(...)`.

**`@Mapping` — sterowanie mapowaniem** gdy nazwy/kształty się różnią:

```java
@Mapper(componentModel = "spring")
public interface OrderMapper {

    @Mapping(source = "customer.email", target = "customerEmail")  // zagnieżdżone
    @Mapping(target = "total", expression = "java(order.getPrice().multiply(order.getQty()))")
    @Mapping(target = "internalNote", ignore = true)               // nie eksponuj
    OrderResponse toResponse(Order order);

    @Mapping(target = "createdAt", qualifiedByName = "nowUtc")      // custom logika
    Order toEntity(CreateOrderRequest req);

    @Named("nowUtc")
    default Instant nowUtc(String ignored) { return Instant.now(); }
}
```

- `source`/`target` — mapowanie **różnych nazw** pól.
- `ignore = true` — jawne pominięcie (świetne do wrażliwych/wewnętrznych pól).
- `expression = "java(...)"` — wstawienie kodu Java.
- `qualifiedByName` / `@Named` — wybór konkretnej metody pomocniczej (np. konwersja typu).

**Aktualizacja istniejącego obiektu** (`@MappingTarget`) — nie tworzy nowego, tylko nadpisuje pola przekazanej instancji (kluczowe przy PATCH/PUT encji zarządzanej przez Hibernate):

```java
void updateEntity(UpdateUserRequest req, @MappingTarget User user);
// wywołane w transakcji na managed encji → dirty checking zapisze zmiany
```

## 3. Dlaczego / kiedy — DTO obowiązkowe, pułapki mapowania

**Dlaczego DTO jest obowiązkowe (nigdy encja w API):**
- **Rozdzielenie kontraktu API od modelu bazy** — zmiana kolumny/relacji nie łamie klientów; API ewoluuje niezależnie od schematu.
- **`LazyInitializationException`** — serializacja encji poza transakcją dotyka lazy relacji (proxy) → wyjątek albo, jeszcze gorzej, **niezamierzone doładowanie** całych grafów. DTO ma tylko dane już wczytane.
- **Bezpieczeństwo — wyciek pól:** encja ma `passwordHash`, flagi wewnętrzne, audyt. Serializacja encji wprost ujawni je w JSON.
- **Bezpieczeństwo — mass assignment:** deserializacja żądania wprost do encji pozwala atakującemu ustawić pola, których nie powinien (`role`, `id`, `enabled`). DTO żądania zawiera **tylko** pola dozwolone do zapisu.
- **Wersjonowanie API** — możesz mieć `UserResponseV1` i `V2` na tym samym modelu bazy.
- **Kontrola kształtu odpowiedzi** — spłaszczasz, agregujesz, nazywasz pola po swojemu, niezależnie od struktury tabel.

**Pułapki mapowania (tu leży prawdziwy koszt):**
- **N+1 przy mapowaniu** — mapper czytający lazy kolekcję (`order.getItems()`) **doładowuje** ją przy każdym elemencie listy → klasyczny problem N+1 (→ [[wiedza/06-persystencja/n-plus-1]]). Lekarstwo: pobierz dane z `JOIN FETCH`/`@EntityGraph` **przed** mapowaniem, albo projektuj wprost do DTO (`SELECT new ...` / interfejsowe projekcje) i w ogóle nie mapuj z encji.
- **Mapowanie cykliczne** — encje z relacją dwukierunkową (`User ↔ Order`) dają nieskończoną rekurencję w mapperze/serializacji. Rozwiąż mapując tylko jedną stronę lub płaski DTO.
- **Mapowanie musi być poza granicą lenistwa** — jeśli mapujesz encję poza transakcją, lazy pola rzucą wyjątek; mapuj w warstwie serwisu, w transakcji, albo mapuj już wczytane dane.

**Alternatywy:**
- **ModelMapper** — dopasowuje pola przez **refleksję w runtime**: mniej kodu konfiguracji, ale wolniejszy, „magiczny" (błędy dopiero w runtime), trudniejszy do debugowania. MapStruct wygrywa wydajnością i bezpieczeństwem typów.
- **Ręcznie** — pełna kontrola, ale żmudne i błędogenne przy wielu polach; sensowne tylko dla pojedynczych, nietypowych konwersji.
- **Projekcje na poziomie zapytania** (JPQL `SELECT new`, projekcje interfejsowe, DTO-native query) — omijają mapowanie encji i unikają N+1; idealne dla odczytów read-only.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Typowa warstwa serwisowa Spring Boot 3 — mapper wstrzyknięty jako bean, mapowanie w transakcji:

```java
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository repo;
    private final UserMapper mapper;   // wstrzyknięty bean (componentModel="spring")

    @Transactional(readOnly = true)
    public UserResponse getUser(Long id) {
        User user = repo.findById(id).orElseThrow();
        return mapper.toResponse(user);         // encja → DTO, w transakcji
    }

    @Transactional
    public UserResponse create(CreateUserRequest req) {
        User user = mapper.toEntity(req);       // DTO żądania → encja
        user.setPasswordHash(hash(req.password()));
        return mapper.toResponse(repo.save(user));
    }

    @Transactional
    public UserResponse update(Long id, UpdateUserRequest req) {
        User user = repo.findById(id).orElseThrow();
        mapper.updateEntity(req, user);         // @MappingTarget — nadpisuje pola
        return mapper.toResponse(user);         // dirty checking zapisze przy commit
    }
}
```

Kontroler przyjmuje i zwraca **wyłącznie DTO** — encja `User` nigdy nie opuszcza warstwy serwisu.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, czym różni się DTO od encji i podam 4 powody, by nie eksponować encji w API.
- [ ] Wiem, że MapStruct generuje kod przy kompilacji (annotation processor) i że nie używa refleksji.
- [ ] Potrafię napisać `@Mapper` z `@Mapping` (source/target, ignore, expression) i użyć `@MappingTarget`.
- [ ] Rozumiem, jak mapowanie encji powoduje N+1 i `LazyInitializationException` oraz jak temu zapobiec.
- [ ] Umiem uzasadnić MapStruct vs ModelMapper vs ręczne mapowanie.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co dokładnie robi annotation processor MapStruct podczas `javac` i gdzie ląduje wynik?
- [ ] Dlaczego wygenerowany mapper jest szybszy niż ModelMapper? (kod jawny vs refleksja)
- [ ] Jak `componentModel="spring"` zmienia wygenerowaną klasę? (`@Component` → bean)
- [ ] Dlaczego serializacja encji poza transakcją rzuca `LazyInitializationException`?
- [ ] Na czym polega atak mass assignment i jak DTO żądania go blokuje?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Czemu nie zwracać encji JPA wprost z kontrolera?"
- [ ] „Czym MapStruct różni się od ModelMapper?" (kompilacja/kod vs runtime/refleksja)
- [ ] „Jak mapowanie DTO może wywołać N+1 i jak temu zapobiec?"
- [ ] „Jak zaktualizować istniejącą encję mapperem?" (`@MappingTarget`)
- [ ] „Dlaczego rekord to dobry DTO?" (niemutowalny, zwięzły, `equals`/`hashCode`)

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Co to DTO?;Data Transfer Object — płaski obiekt do przenoszenia danych przez granicę systemu (API), bez logiki domenowej i bez zarządzania przez EntityManager.
Dlaczego nie eksponować encji JPA w API?;Sprzęga API ze schematem bazy, grozi LazyInitializationException, wyciekiem pól wrażliwych i mass assignment, utrudnia wersjonowanie i kontrolę kształtu odpowiedzi.
Czym różni się DTO od encji?;Encja to model bazy (tożsamość @Id, relacje, stan zarządzany, lazy proxy); DTO to kontrakt na granicy systemu bez powiązania z persystencją.
Dlaczego rekord Javy to dobry DTO?;Jest niemutowalny, zwięzły i ma automatyczne equals/hashCode/toString — idealny nośnik danych.
Kiedy MapStruct generuje kod mappera?;W czasie kompilacji, jako annotation processor wpięty w javac — tworzy klasę Impl w generated-sources.
Czy MapStruct używa refleksji w runtime?;Nie — generuje zwykły kod Java (jawne przypisania getterów/konstruktorów), dlatego jest szybki i typobezpieczny.
Kiedy błąd mapowania MapStruct się ujawnia?;Podczas kompilacji — javac przerywa build, gdy pole target nie ma źródła i nie jest zignorowane, lub przy niedopasowaniu typów.
Do czego służy @Mapping w MapStruct?;Do sterowania mapowaniem: source/target dla różnych nazw pól, ignore, expression, qualifiedByName.
Co robi @MappingTarget?;Nadpisuje pola istniejącej instancji zamiast tworzyć nowy obiekt — używane do aktualizacji managed encji (PATCH/PUT).
Co daje componentModel="spring" w @Mapper?;Wygenerowany mapper dostaje @Component i jest wstrzykiwalnym beanem Springa.
MapStruct vs ModelMapper?;MapStruct generuje kod przy kompilacji (bez refleksji, błędy w javac, szybko); ModelMapper mapuje refleksją w runtime (wolniej, błędy dopiero w runtime).
Jak mapowanie DTO powoduje N+1?;Mapper czytający lazy kolekcję encji doładowuje ją dla każdego elementu listy — zapobiega temu JOIN FETCH/@EntityGraph lub projekcja wprost do DTO.
Czemu serializacja encji poza transakcją rzuca LazyInitializationException?;Dotyka lazy proxy relacji, gdy sesja Hibernate jest już zamknięta — nie ma jak doładować danych.
Co to mass assignment i jak DTO go blokuje?;Atak, w którym deserializacja żądania wprost do encji ustawia niedozwolone pola (role, id); DTO żądania zawiera tylko pola dozwolone do zapisu.
Po co osobne DTO dla request i response?;Mają różne pola (request bez id/audytu, response bez haseł) — zapobiega wyciekom i mass assignment.
Jak MapStruct mapuje kolekcje i pola zagnieżdżone?;Kolekcje przez metodę List→List (iteruje pojedynczy mapper), zagnieżdżone przez @Mapping(source="a.b", target="c").
Jak dodać własną logikę w mapowaniu MapStruct?;Przez expression="java(...)", metodę default z @Named + qualifiedByName, albo klasę abstrakcyjną z metodami pomocniczymi.
Gdzie należy mapować encję na DTO?;W warstwie serwisu, w obrębie transakcji (lub na już wczytanych danych) — inaczej lazy pola rzucą LazyInitializationException.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/06-persystencja/jpa-hibernate]] · [[wiedza/06-persystencja/n-plus-1]] · [[wiedza/01-jezyk/record-sealed-enum]]
