---
temat: "Paginacja, filtrowanie i sortowanie w REST API"
faza: 7
status: nieopanowany
priorytet: 🟡
tags: [java, api, wydajnosc]
powiazane: ["[[wiedza/06-persystencja/spring-data-jpa]]", "[[wiedza/06-persystencja/n-plus-1]]"]
---

# Paginacja, filtrowanie i sortowanie w REST API

> **TL;DR:** Nigdy nie zwracaj całej tabeli — tnij wyniki na strony. **Offset-based** (`LIMIT/OFFSET`, `?page&size`)
> jest proste i pozwala skoczyć do strony N, ale drogie dla dużych offsetów i **niestabilne** przy zmianach danych.
> **Keyset/cursor** (`WHERE id > :last ORDER BY id LIMIT n`) jest szybkie i stabilne — idealne do infinite scroll,
> kosztem braku skoku do dowolnej strony. W Spring Data: `Page` (z `COUNT`, zna `total`) vs `Slice` (bez `COUNT`, tańsze).

## 1. Co — definicja i API

**Paginacja (pagination)** to zwracanie wyników porcjami zamiast wszystkich naraz. Powody:

- **Wydajność** — `SELECT * FROM orders` na 10 mln wierszy zabije bazę, sieć i serializację JSON.
- **Pamięć** — po stronie serwera (cała lista w heapie → OOM) i klienta (przeglądarka nie zrenderuje 100k wierszy).
- **UX** — użytkownik i tak patrzy na 20 rekordów; reszta to marnotrawstwo latencji.

Trzy powiązane operacje, prawie zawsze razem:

| Operacja | Sens | Query param |
|---|---|---|
| **Pagination** | która porcja | `?page=0&size=20` lub `?cursor=...` |
| **Sorting** | w jakiej kolejności | `?sort=name,asc&sort=createdAt,desc` |
| **Filtering** | które wiersze | `?status=ACTIVE&minPrice=10` |

W Spring MVC dostajesz to niemal za darmo — kontroler przyjmuje `Pageable`, a resolver mapuje `page/size/sort`:

```java
@GetMapping("/orders")
Page<OrderDto> list(@PageableDefault(size = 20) Pageable pageable) {
    return orderRepository.findAll(pageable).map(OrderDto::from);
}
```

## 2. Jak — offset vs keyset pod spodem

### Offset-based pagination

`page=3, size=20` → SQL:

```sql
SELECT * FROM orders
ORDER BY created_at DESC, id DESC   -- kolejność MUSI być deterministyczna (tie-breaker po id!)
LIMIT 20 OFFSET 60;
```

Mechanika i **wady**:

- **Kosztowne dla dużych offsetów.** `OFFSET 1000000` nie jest "przeskoczeniem" — baza **skanuje i odrzuca**
  pierwszy milion wierszy, żeby wyprodukować 20. Koszt rośnie liniowo z numerem strony (`O(offset)`).
  To klasyczny "deep pagination problem" — ostatnie strony są dramatycznie wolniejsze niż pierwsze.
- **Niestabilne przy zmianach danych.** Między pobraniem strony 1 i 2 ktoś wstawił/usunął wiersz na początku:
  wszystko się **przesuwa**. Efekt: użytkownik zobaczy ten sam rekord dwa razy (**duplikat**) albo pominie jeden
  (**gap**). Offset liczy pozycję, a pozycja się zmienia.

Zaleta jedyna, ale ważna: **skok do dowolnej strony** (`page=42`) i znajomość `totalPages` (pasek stron 1..N).

### Keyset / cursor pagination (seek method)

Zamiast pozycji zapamiętujesz **wartość ostatnio widzianego klucza** i pytasz o "co po nim":

```sql
SELECT * FROM orders
WHERE (created_at, id) < (:last_created_at, :last_id)  -- kompozytowy kursor
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

- **Wydajne** — z indeksem na `(created_at, id)` baza robi **index seek** wprost do miejsca startu, koszt
  jest stały niezależnie od głębokości (`O(log n)` + skan strony). Nie skanuje odrzuconych wierszy.
- **Stabilne** — kursor to konkretna wartość danych, nie pozycja. Wstawki/usunięcia wcześniej w zbiorze nie
  powodują duplikatów ani gapów. Dlatego to standard dla **infinite scroll** i feedów (Twitter, GitHub API).
- **Wady:** brak skoku do strony N (idziesz tylko naprzód/wstecz), zwykle brak taniego `totalPages`,
  sortowanie musi pokrywać się z kluczem kursora, a klucz musi być **unikalny i deterministyczny**
  (stąd `id` jako tie-breaker, gdy `created_at` może się powtórzyć).

Kursor często koduje się do nieprzezroczystego (opaque) tokena Base64, żeby klient nie polegał na jego strukturze:
`?cursor=eyJjcmVhdGVkQXQiOiIuLi4iLCJpZCI6MTIzfQ==`.

### Spring Data: Page vs Slice

Zobacz [[wiedza/06-persystencja/spring-data-jpa]]. Repozytorium może zwrócić trzy typy:

| Typ | Dodatkowe zapytanie | Wie ile jest total? | Kiedy |
|---|---|---|---|
| `List<T>` | — | nie | proste, sam zarządzasz |
| `Slice<T>` | **brak `COUNT`** | nie — tylko `hasNext()` | infinite scroll, gdy total zbędny |
| `Page<T>` | **osobny `SELECT COUNT(*)`** | tak — `getTotalElements()` | UI z paskiem stron 1..N |

`Slice` pobiera `size + 1` wierszy: jeśli przyszło o jeden więcej, wie, że jest następna strona — **bez drogiego
`COUNT`**. Na dużych, filtrowanych tabelach `COUNT(*)` bywa wolniejszy niż samo pobranie strony, więc `Slice`
(albo keyset) to realna oszczędność. Wybieraj `Page` tylko wtedy, gdy front faktycznie renderuje `totalPages`.

## 3. Dlaczego / kiedy — dobór, limity, pułapki

**Dobór strategii:**

- Klasyczne UI z numerami stron, dane raczej stabilne, małe/średnie zbiory → **offset** (`Page`).
- Infinite scroll, feed, duże tabele, głęboka paginacja, częste zmiany → **keyset** (`Slice`/własny cursor).

**Filtrowanie — od prostego do dynamicznego:**

- **Proste query params** — stała lista pól: `?status=ACTIVE&category=BOOKS`. Mapujesz je na derived query
  (`findByStatusAndCategory(...)`). Czytelne, ale nie skaluje się kombinatorycznie.
- **Dynamiczne** — `Specification` / Criteria API ([[wiedza/06-persystencja/spring-data-jpa]]) budują `WHERE`
  z opcjonalnych predykatów w locie; repozytorium rozszerza `JpaSpecificationExecutor`. Dla "N filtrów, każdy
  opcjonalny" to jedyne sensowne wyjście.
- **RSQL / QueryDSL** (świadomość) — pozwalają klientowi wyrażać złożone filtry w URL
  (`?filter=price>10;status==ACTIVE`). Elastyczne, ale to **ryzyko** — trzeba whitelistować pola i operatory,
  inaczej otwierasz drogę do nadmiernych/kosztownych zapytań.

**Walidacja i LIMITY rozmiaru strony (KRYTYCZNE):**

Nigdy nie ufaj `size` od klienta. `?size=1000000` to wektor DoS — jedno żądanie ładuje całą tabelę do pamięci.
Ustaw **twardy sufit** i domyślny rozmiar:

```java
// application.yml
spring.data.web.pageable.default-page-size: 20
spring.data.web.pageable.max-page-size: 100   // Spring UTNIE do 100, nie odrzuci — pamiętaj o tym
```

Dla własnych parametrów waliduj jawnie (`@Max(100) @Min(1) int size`). Waliduj też **pola sortowania** — zezwól
tylko na whitelistę kolumn, bo `?sort=<cokolwiek>` po niezindeksowanej/nieistniejącej kolumnie to błąd lub full scan.

**Kształt odpowiedzi (response shape):**

Dwa style, oba akceptowalne — wybierz jeden i trzymaj się go w całym API:

1. **Metadane w body** (styl Spring `Page`): koperta z zawartością + paginacją.
   ```json
   { "content": [ ... ],
     "page": { "number": 0, "size": 20, "totalElements": 1543, "totalPages": 78 } }
   ```
2. **Nagłówki HTTP**: `X-Total-Count: 1543` oraz `Link: <...&page=1>; rel="next", <...&page=77>; rel="last"`
   (styl GitHub). Body zostaje czystą tablicą. Dla keyset zwraca się `next_cursor` w body lub `Link`.

**Pułapki (traps):**

- **Paginacja + `JOIN FETCH` kolekcji = paginacja w pamięci!** Gdy fetchujesz kolekcję `@OneToMany`, jeden
  encja daje N wierszy w wyniku — Hibernate **nie może** zastosować `LIMIT` na poziomie SQL (uciąłby część
  kolekcji encji), więc **pobiera WSZYSTKO i pobiera stronę w Javie**. W logu zobaczysz osławione
  `HHH000104: firstResult/maxResults specified with collection fetch; applying in memory`. To cichy zabójca
  wydajności. Rozwiązanie: paginuj po encjach głównych (osobne zapytanie po id), potem dociągnij kolekcje —
  zobacz [[wiedza/06-persystencja/n-plus-1]] (`@EntityGraph`, `@BatchSize`, dwustopniowe pobranie).
- **Sortowanie po niezindeksowanej kolumnie** → `ORDER BY` wymusza pełny skan + sortowanie na dysku. Każda
  kolumna dostępna w `?sort=` powinna mieć indeks (dla keyset — indeks kompozytowy pokrywający cały klucz).
- **Brak deterministycznego tie-breakera** w `ORDER BY` → przy równych wartościach kolejność jest niezdefiniowana,
  co psuje zarówno offset (drżenie stron), jak i keyset (kursor może pomijać wiersze). Zawsze dokładaj `, id`.
- **`COUNT(*)` na dużej filtrowanej tabeli** bywa droższy niż sama strona — nie żądaj `Page`, jeśli nie musisz.

**Spójność z frontem (Angular):** klient TypeScript jest zwykle **generowany z OpenAPI**. Jeśli backend zwraca
`Page<T>`, wygenerowany model musi mieć te same pola (`content`, `totalElements`...). Dlatego warto zwracać
**własne, stabilne DTO paginacji** zamiast wewnętrznej struktury Spring `PageImpl` (jej serializacja zmieniała się
między wersjami Boota — w Spring Boot 3 doszła jawna warstwa `PagedModel`, żeby ustabilizować kontrakt JSON).

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Kontroler z `Pageable`, filtrem i limitem — plus porównanie SQL offset vs keyset.

```java
@RestController
@RequestMapping("/api/orders")
class OrderController {

    private final OrderRepository repo;

    // GET /api/orders?status=ACTIVE&page=0&size=20&sort=createdAt,desc
    @GetMapping
    PagedModel<OrderDto> list(
            @RequestParam(required = false) OrderStatus status,
            @PageableDefault(size = 20)
            @SortDefault(sort = "createdAt", direction = Sort.Direction.DESC) Pageable pageable) {

        // Specification buduje WHERE dynamicznie z opcjonalnego filtra
        Specification<Order> spec = (root, q, cb) ->
                status == null ? cb.conjunction() : cb.equal(root.get("status"), status);

        Page<OrderDto> page = repo.findAll(spec, pageable).map(OrderDto::from);
        return new PagedModel<>(page);   // stabilny kontrakt JSON dla OpenAPI/Angular
    }
}
```

Porównanie tego samego "daj 20 najnowszych po ostatnio widzianym":

```sql
-- OFFSET: dla page=5000 baza skanuje 100 000 wierszy i wyrzuca do kosza
SELECT * FROM orders
ORDER BY created_at DESC, id DESC
LIMIT 20 OFFSET 100000;                       -- wolne, rośnie z numerem strony

-- KEYSET: index seek prosto do miejsca, koszt niezależny od głębokości
SELECT * FROM orders
WHERE (created_at, id) < ('2026-07-03 10:00', 8421)
ORDER BY created_at DESC, id DESC
LIMIT 20;                                      -- szybkie i stabilne; wymaga indeksu (created_at, id)
```

Kontekst: Java 21 + Spring Boot 3 + PostgreSQL. W PostgreSQL keyset korzysta z indeksu b-tree
`CREATE INDEX ON orders (created_at DESC, id DESC)`; offset nie ma jak przyspieszyć poza cache'em.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, po co paginacja (wydajność, pamięć, UX) — jednym zdaniem.
- [ ] Napiszę SQL offset i keyset i wskażę, czemu keyset jest szybszy dla dużych offsetów.
- [ ] Wymienię wady offsetu: koszt deep pagination + niestabilność (duplikaty/gapy).
- [ ] Rozróżnię `Page` (z `COUNT`) od `Slice` (bez `COUNT`) i uzasadnię wybór.
- [ ] Wiem, czemu paginacja z `JOIN FETCH` kolekcji ląduje w pamięci i jak to naprawić.
- [ ] Wiem, czemu trzeba limitować `size` i whitelistować pola sortowania.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co robi baza przy `LIMIT 20 OFFSET 1000000`? (skanuje i odrzuca milion wierszy)
- [ ] Dlaczego keyset jest stabilny, a offset nie? (wartość klucza vs pozycja)
- [ ] Skąd `Slice` wie, że jest następna strona, bez `COUNT`? (pobiera size+1)
- [ ] Czemu `ORDER BY` potrzebuje tie-breakera po `id`?
- [ ] Co oznacza `HHH000104 ... applying in memory` i czym grozi?
- [ ] Czemu `?size=1000000` jest niebezpieczne i jak Spring to ogranicza?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Offset vs cursor pagination — kiedy co i dlaczego?"
- [ ] „Masz feed z infinite scroll na 50 mln rekordów — jak paginujesz?"
- [ ] „Czym różni się `Page` od `Slice` w Spring Data i po co `Slice`?"
- [ ] „Jak zrobisz dynamiczne filtrowanie po wielu opcjonalnych polach?"
- [ ] „Paginacja przestała działać wydajnie po dodaniu `JOIN FETCH` — dlaczego?"
- [ ] „Jak zabezpieczysz endpoint listujący przed nadużyciem parametrów?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Po co paginacja w REST API?;Żeby nie zwracać całej tabeli — chroni wydajność bazy/sieci, pamięć serwera i klienta oraz UX.
Jak wygląda offset-based pagination w SQL?;LIMIT n OFFSET m z deterministycznym ORDER BY — zwraca n wierszy pomijając pierwsze m.
Główna wada offsetu dla dużych stron?;Baza skanuje i odrzuca wszystkie wiersze przed offsetem — koszt rośnie liniowo z numerem strony (deep pagination).
Czemu offset jest niestabilny?;Liczy pozycję; wstawka/usunięcie wcześniej przesuwa dane, dając duplikaty lub pominięte wiersze między stronami.
Jak wygląda keyset/cursor pagination?;WHERE (klucz) < :ostatnio_widziany ORDER BY klucz LIMIT n — pytasz o "co po ostatnio widzianym".
Główna zaleta keyset pagination?;Index seek wprost do miejsca startu — koszt stały niezależnie od głębokości, i stabilność (bez duplikatów/gapów).
Główna wada keyset pagination?;Brak skoku do dowolnej strony N (tylko naprzód/wstecz) i zwykle brak taniego totalPages.
Kiedy wybrać keyset zamiast offset?;Infinite scroll, feedy, duże tabele, głęboka paginacja i częste zmiany danych.
Page vs Slice w Spring Data?;Page robi dodatkowy SELECT COUNT i zna totalElements/totalPages; Slice pomija COUNT i wie tylko hasNext — taniej.
Skąd Slice wie, że jest następna strona?;Pobiera size+1 wierszy — jeśli przyszedł jeden nadmiarowy, następna strona istnieje.
Parametry paginacji/sortowania w Spring MVC?;?page=0&size=20&sort=name,asc — mapowane automatycznie na Pageable.
Proste vs dynamiczne filtrowanie?;Proste = stałe query params i derived query; dynamiczne = Specification/Criteria budujące WHERE z opcjonalnych predykatów.
Czemu trzeba limitować rozmiar strony?;?size=1000000 to wektor DoS ładujący całą tabelę do pamięci; ustaw max-page-size i waliduj.
Jak przekazać metadane paginacji w odpowiedzi?;W body (content + totalElements/totalPages/number) albo w nagłówkach X-Total-Count i Link (rel=next/last).
Pułapka: paginacja + JOIN FETCH kolekcji?;Hibernate nie może dać LIMIT w SQL, pobiera wszystko i tnie stronę w pamięci (HHH000104) — paginuj po encjach głównych, potem dociągaj kolekcje.
Czemu ORDER BY potrzebuje tie-breakera po id?;Przy równych wartościach kolejność jest niezdefiniowana — psuje to stabilność offsetu i poprawność kursora keyset.
Ryzyko sortowania po niezindeksowanej kolumnie?;ORDER BY wymusza pełny skan i sortowanie na dysku — whitelistuj pola sortowania i indeksuj je.
Czemu zwracać własne DTO paginacji zamiast PageImpl?;Serializacja PageImpl zmieniała się między wersjami Boota; stabilne DTO/PagedModel daje przewidywalny kontrakt dla klienta z OpenAPI (Angular).
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/06-persystencja/spring-data-jpa]] · [[wiedza/06-persystencja/n-plus-1]]
