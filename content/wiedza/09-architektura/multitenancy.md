---
temat: "Multitenancy (wielodostępność)"
faza: 9
status: nieopanowany
priorytet: 🟢
tags: [java, architektura, multitenancy, saas]
powiazane: ["[[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]]", "[[wiedza/08-bezpieczenstwo/owasp]]", "[[wiedza/06-persystencja/flyway]]", "[[projekt-rekrutacja/docs/ARCHITEKTURA]]"]
---

# Multitenancy (wielodostępność)

> **TL;DR:** Jedna instancja aplikacji (jeden proces JVM, jeden deployment) obsługuje **wielu izolowanych tenantów**
> (najemców) tak, że dane jednego są niewidoczne dla drugiego. Klucz to **izolacja danych** — realizowana na trzech
> poziomach: **database-per-tenant** (najsilniejsza izolacja, najdroższa operacyjnie), **schema-per-tenant**
> (złoty środek na PostgreSQL) i **shared schema + discriminator `tenant_id`** (najtańsza i najskalowalniejsza, ale
> każdy błąd w filtrowaniu = **wyciek danych między tenantami**). Tenant identyfikujesz z żądania (subdomena / nagłówek /
> claim w JWT), propagujesz przez `TenantContext` (uwaga na wątki/async), a wymusza go Hibernate 6
> (`CurrentTenantIdentifierResolver` + `MultiTenantConnectionProvider`) lub PostgreSQL **Row-Level Security (RLS)**.

## 1. Co — definicja i API

**Tenant** (najemca) = logicznie odrębny klient systemu: w projekcie rekrutacyjnym to jedno **stowarzyszenie**,
którego kandydaci, procesy i dane nie mogą się mieszać z danymi innego stowarzyszenia.

**Multitenancy** = architektura, w której **jedna instancja aplikacji** obsługuje wielu tenantów przy zachowaniu
**izolacji** (danych, a często i konfiguracji/brandingu). Przeciwieństwo to **single-tenant** — osobna kopia aplikacji
i bazy per klient (prosta izolacja, ale koszt i utrzymanie rosną liniowo z liczbą klientów).

```
                       ┌──────────────────────────────────────────────┐
   tenant A ──┐        │           JEDNA instancja aplikacji           │
   tenant B ──┼──HTTP──▶  (Spring Boot 3, jeden proces JVM 21)         │
   tenant C ──┘        │   TenantContext = "B"  (z JWT/subdomeny)      │
                       └───────────────────┬──────────────────────────┘
                                           │ wybór źródła danych wg tenanta
              ┌────────────────────────────┼────────────────────────────┐
              ▼                            ▼                            ▼
   DATABASE-PER-TENANT           SCHEMA-PER-TENANT          SHARED + DISCRIMINATOR
   db_a  db_b  db_c              schema_a | schema_b        jedna tabela, kolumna
   (osobne bazy)                 (jedna baza PG)            tenant_id = 'B' w WHERE
```

**Po co (SaaS):** wielodostępność to fundament modelu **SaaS** — zamiast utrzymywać N kopii aplikacji, utrzymujesz
jedną. Zyskujesz **efektywność kosztową** (współdzielone CPU/RAM/połączenia, jeden deployment, jedno wdrożenie zmian),
prostszy operations i ekonomię skali. Płacisz za to **słabszą izolacją** i większą złożonością warstwy danych — cały
temat to negocjacja na osi **izolacja ⇄ koszt/skalowalność**.

## 2. Jak — strategie, identyfikacja, Hibernate/RLS pod spodem

### 2.1. Trzy strategie izolacji danych

**(1) Database-per-tenant** — każdy tenant ma **własną, fizycznie osobną bazę danych**.
- ✅ **Najsilniejsza izolacja i bezpieczeństwo** — brak fizycznej możliwości, by zapytanie tenanta A dotknęło danych B.
- ✅ **Backup/restore/PITR per tenant** trywialne; łatwe **geo-residency** (baza tenanta w jego regionie); łatwe usunięcie
  danych (GDPR „prawo do bycia zapomnianym" = `DROP DATABASE`).
- ❌ **Kosztowne operacyjnie:** dużo instancji baz/schematów, **N × migracje** (Flyway musi przejść po każdej bazie),
  **N × pule połączeń** — przy tysiącach tenantów liczba connection pooli i otwartych połączeń staje się wąskim gardłem.
- ❌ Wolne **onboarding** nowego tenanta (provisioning bazy) i trudny **cross-tenant reporting**.

**(2) Schema-per-tenant** — **jedna baza PostgreSQL**, ale **osobny `schema`** (namespace tabel) na tenanta.
- ✅ **Dobry kompromis:** dane fizycznie w osobnych schematach (mocna separacja w ramach jednej bazy), a jednocześnie
  **jedna instancja bazy** i jedna główna pula połączeń.
- ✅ Backup selektywny po schemacie (`pg_dump -n schema_x`), migracje **per schemat** (Flyway iteruje po `search_path`).
- ❌ Migracje wciąż **N-krotne** (każdy schemat trzeba przemigrować), katalog PostgreSQL rośnie (tysiące schematów ×
  tabele = duży `pg_catalog`, wolniejsze `\dt`, presja na shared memory). Realny sufit to setki–niskie tysiące tenantów.
- To zwykle **rekomendowany start** dla średniej liczby średnio-dużych tenantów (jak stowarzyszenia).

**(3) Shared schema + discriminator** — **wspólne tabele**, a przynależność rozróżnia **kolumna `tenant_id`**.
- ✅ **Najtańsze i najbardziej skalowalne** — jeden schemat, jeden zestaw tabel, **jedna migracja** dla wszystkich,
  natychmiastowy onboarding (wstaw wiersz z nowym `tenant_id`). Idealne przy **tysiącach** małych tenantów.
- ❌ **Najsłabsza izolacja** i **krytyczne ryzyko wycieku**: jeśli którekolwiek zapytanie zapomni `WHERE tenant_id = ?`,
  jeden tenant zobaczy dane innych (to jest wprost **Broken Access Control** — [[wiedza/08-bezpieczenstwo/owasp]]).
  **Każde** zapytanie musi filtrować po tenancie — dlatego filtruje się **automatycznie** (Hibernate `@TenantId`,
  filtry, albo bazowo **PostgreSQL RLS**), a nie ręcznie w każdym repozytorium.
- ❌ Wspólne tabele, indeksy i statystyki („noisy neighbor" — duży tenant psuje plany zapytań innym); trudny per-tenant
  backup; usunięcie tenanta to duży `DELETE ... WHERE tenant_id = ?`.

#### Tabela porównawcza

| Kryterium            | Database-per-tenant | Schema-per-tenant     | Shared + discriminator |
|----------------------|---------------------|-----------------------|------------------------|
| **Izolacja danych**  | 🟢 najsilniejsza    | 🟡 dobra              | 🔴 najsłabsza (ryzyko wycieku) |
| **Koszt / zasoby**   | 🔴 najwyższy        | 🟡 średni             | 🟢 najniższy           |
| **Skalowalność (liczba tenantów)** | 🔴 setki | 🟡 setki–tysiące   | 🟢 dziesiątki tysięcy  |
| **Migracje**         | 🔴 N × baza         | 🔴 N × schemat        | 🟢 1 × wspólny schemat |
| **Backup/restore per tenant** | 🟢 trywialny | 🟢 `pg_dump -n`      | 🔴 trudny (filtrowany)  |
| **Złożoność aplikacji** | 🟡 routing DataSource | 🟡 routing search_path | 🔴 filtr w KAŻDYM zapytaniu / RLS |
| **Onboarding nowego tenanta** | 🔴 provisioning bazy | 🟡 `CREATE SCHEMA` + migracja | 🟢 wiersz z tenant_id |

### 2.2. Identyfikacja tenanta w żądaniu

Zanim cokolwiek zrobisz z danymi, musisz wiedzieć **kto pyta**. Źródła identyfikatora tenanta:
- **Subdomena** — `stowarzyszenie-a.rekrutacja.pl` → tenant `stowarzyszenie-a` (czytelne, dobre dla brandingu).
- **Nagłówek HTTP** — np. `X-Tenant-ID: a` (proste dla API-to-API, ale trzeba pilnować, by klient nie mógł go podszyć).
- **Claim w JWT** — najbezpieczniejsze dla systemu z logowaniem: `tenant_id` (lub `org`) jest **podpisanym** claimem
  w tokenie, więc użytkownik **nie może go sfałszować** bez złamania podpisu. Zob. [[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]].
  **Zasada:** ufaj tenantowi z **zaufanego, zweryfikowanego** źródła (claim po walidacji podpisu), nie z surowego
  nagłówka, który klient wpisał sam.

### 2.3. Propagacja tenanta — TenantContext

Warstwa danych (Hibernate/JDBC) nie ma dostępu do `HttpServletRequest`. Rozwiązanie: **na wejściu** (filtr / interceptor)
wyciągasz tenanta i wkładasz do `TenantContext` opartego o **`ThreadLocal`**, a **na końcu żądania czyścisz** (`finally`).

```java
public final class TenantContext {
    private static final ThreadLocal<String> CURRENT = new ThreadLocal<>();
    public static void set(String tenant) { CURRENT.set(tenant); }
    public static String get() { return CURRENT.get(); }
    public static void clear() { CURRENT.remove(); }   // ZAWSZE w finally!
}
```

**Pułapki wątkowości (bardzo ważne):**
- `ThreadLocal` żyje w **jednym** wątku. Gdy zlecasz pracę **async** (`@Async`, `CompletableFuture`, executor, reactive),
  kontekst **NIE** przechodzi automatycznie do wątku roboczego → tenant „znika" albo, gorzej, **wraca cudzy** z puli.
  Trzeba go **jawnie skopiować** do zadania (dekorator `TaskDecorator`, `MDC`-podobny wrapper, `ContextSnapshot`).
- **Virtual threads (Java 21):** wciąż wątki, więc `ThreadLocal` działa, ale ich krótkotrwałość zniechęca do dużych
  `ThreadLocal`; nowocześnie rozważ **`ScopedValue`** (Java 21 preview) jako czystsze, niemutowalne, dziedziczone
  przez `StructuredTaskScope` przenoszenie kontekstu. Kluczowa zasada niezmienna: **czyść ThreadLocal**, bo wątki (także
  wirtualne montowane na nośnych) i wątki z puli **są recyklingowane** — brak `clear()` = wyciek tenanta.

### 2.4. Wsparcie Hibernate 6 dla multitenancy

Hibernate ma **wbudowane** multitenancy dla trybów DATABASE i SCHEMA — trzeba dostarczyć dwie strategie:

- **`CurrentTenantIdentifierResolver<String>`** — mówi Hibernate „kto jest bieżącym tenantem" (czyta z `TenantContext`).
- **`MultiTenantConnectionProvider<String>`** — dostarcza właściwe **`Connection`**:
  - tryb **DATABASE** → wybiera odpowiedni `DataSource` (routing baza per tenant),
  - tryb **SCHEMA** → bierze jedno połączenie i ustawia `SET search_path TO <schema_tenanta>` (przy zwrocie do puli
    **resetuje** `search_path` do domyślnego — inaczej połączenie wróci „skażone").

```
hibernate.tenant_identifier_resolver = <bean resolvera>
hibernate.multi_tenant_connection_provider = <bean providera>
# tryb wykrywany z beanów; historyczne hibernate.multiTenancy = SCHEMA|DATABASE
```

- **Dla strategii dyskryminatora** Hibernate 6 daje adnotację **`@TenantId`** na polu encji: Hibernate **sam** dokłada
  `tenant_id` do `INSERT`ów i do warunku `WHERE` `SELECT`ów/`UPDATE`ów — eliminując ręczne filtrowanie. Alternatywy:
  **`@Filter`** włączany per sesja, albo (najmocniej) **PostgreSQL RLS** pod spodem.

### 2.5. PostgreSQL Row-Level Security (RLS) — sieć bezpieczeństwa dla dyskryminatora

RLS to mechanizm samej **bazy**: definiujesz **POLICY**, a PostgreSQL **automatycznie** dokłada warunek do każdego
`SELECT/INSERT/UPDATE/DELETE`. Ochrona działa nawet jeśli aplikacja zapomni `WHERE tenant_id` — **baza i tak nie odda
cudzych wierszy**. To zmienia izolację strategii (3) z „ufamy każdemu programiście" na „wymusza silnik bazy". RLS jest
kompletne dopiero, gdy aplikacja łączy się rolą **BEZ** `BYPASSRLS` i nie jest właścicielem tabeli (właściciel domyślnie
omija policy).

## 3. Dlaczego / kiedy — dobór strategii, izolacja vs koszt

- **Izolacja vs koszt to główny kompromis.** Im silniejsza izolacja (osobne bazy), tym większy narzut operacyjny;
  im tańsze i skalowalniejsze (wspólny schemat), tym łatwiej o **wyciek** i „noisy neighbor".
- **Kiedy database-per-tenant:** mało tenantów o wysokich wymaganiach (regulacje, data-residency, gwarancje izolacji,
  odrębny backup/SLA), gdzie klient płaci za dedykację. NIE przy tysiącach małych tenantów (utoniesz w migracjach i pulach).
- **Kiedy schema-per-tenant:** średnia liczba średnich tenantów, potrzeba **wyraźnej** separacji bez kosztu N baz — częsty
  „domyślny wybór" dla B2B SaaS.
- **Kiedy shared + discriminator (+RLS):** bardzo dużo małych tenantów, priorytet: koszt i skalowalność; izolację
  **koniecznie** wzmacniasz RLS lub `@TenantId`, bo sam kod nie wystarcza.
- **Bezpieczeństwo:** cross-tenant data leak to **Broken Access Control** (OWASP Top 10 A01) — jeden z najgroźniejszych
  błędów w SaaS. Testuj to jawnie: uwierzytelniony tenant A **nie może** odczytać/zmodyfikować zasobu tenanta B (nawet
  znając ID). Zob. [[wiedza/08-bezpieczenstwo/owasp]].
- **Migracje:** przy database/schema-per-tenant [[wiedza/06-persystencja/flyway]] musi przejść **po każdej bazie/schemacie**
  (pętla migracji przy starcie / krok w CI/CD), a nowe tenanty domigrować przy provisioningu. Wspólny schemat = jedna migracja.
- **Pula połączeń:** database-per-tenant zwykle oznacza **wiele pul** (albo dynamiczny routing DataSource) — łatwo przekroczyć
  `max_connections` PostgreSQL; ratunek to warstwa poolingu (np. PgBouncer) lub schema/discriminator, gdzie wystarczy
  jedna pula.

### Rekomendacja dla projektu rekrutacyjnego (stowarzyszenia)
Tenant = **stowarzyszenie**. Spodziewana skala to dziesiątki–setki średnich tenantów, z realnym wymogiem izolacji danych
kandydatów.
- **Kandydat główny: schema-per-tenant** — mocna separacja, jedna baza PostgreSQL, sensowny koszt, wykonalne migracje
  Flyway per schemat. Dobrze mapuje się na Hibernate SCHEMA mode.
- **Alternatywa: shared schema + `tenant_id` + PostgreSQL RLS** — jeśli liczba tenantów urośnie do tysięcy lub priorytetem
  stanie się koszt; RLS domyka lukę izolacji.
- **Współpraca z Flowable (silnik procesów):** Flowable natywnie zna pojęcie **`tenantId`** — encje procesowe
  (definicje, instancje, taski) da się znakować `tenantId` spójnym z `TenantContext`, więc silnik procesów rekrutacji
  respektuje ten sam podział najemców co warstwa danych. Szczegóły w [[projekt-rekrutacja/docs/ARCHITEKTURA]].

## Przykład w praktyce

**A) `CurrentTenantIdentifierResolver` czytający tenanta z JWT (przez `TenantContext`):**

```java
// Filtr wejściowy: wyciąga zweryfikowany claim z JWT i ustawia kontekst
@Component
class TenantFilter extends OncePerRequestFilter {
    @Override protected void doFilterInternal(HttpServletRequest req,
            HttpServletResponse res, FilterChain chain) throws ServletException, IOException {
        var auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth instanceof JwtAuthenticationToken jwt) {
            TenantContext.set(jwt.getToken().getClaimAsString("tenant_id")); // claim PO walidacji podpisu
        }
        try { chain.doFilter(req, res); }
        finally { TenantContext.clear(); }   // sprzątanie ThreadLocal — obowiązkowe
    }
}

// Hibernate pyta o bieżącego tenanta
@Component
class JwtTenantResolver implements CurrentTenantIdentifierResolver<String> {
    private static final String DEFAULT = "public";
    @Override public String resolveCurrentTenantIdentifier() {
        var t = TenantContext.get();
        return t != null ? t : DEFAULT;                  // fallback dla operacji poza żądaniem (np. bootstrap)
    }
    @Override public boolean validateExistingCurrentSessions() { return true; }
}
```

**B) PostgreSQL Row-Level Security dla strategii dyskryminatora:**

```sql
-- 1) tabela ze znacznikiem tenanta
CREATE TABLE candidate (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id   text NOT NULL,
    full_name   text NOT NULL
);

-- 2) włącz RLS na tabeli
ALTER TABLE candidate ENABLE ROW LEVEL SECURITY;
ALTER TABLE candidate FORCE ROW LEVEL SECURITY;   -- egzekwuj też wobec właściciela

-- 3) policy: widoczne tylko wiersze bieżącego tenanta (z ustawienia sesji)
CREATE POLICY tenant_isolation ON candidate
    USING     (tenant_id = current_setting('app.current_tenant', true))
    WITH CHECK (tenant_id = current_setting('app.current_tenant', true));
```

```java
// Aplikacja ustawia tenanta na połączeniu przed zapytaniami (np. w MultiTenantConnectionProvider):
//   SET app.current_tenant = 'stowarzyszenie-a';
// Od teraz KAŻDE zapytanie na 'candidate' zwróci tylko wiersze tego tenanta —
// nawet gdy kod pominie WHERE tenant_id. To wymusza baza, nie programista.
```

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, czym jest multitenancy i dlaczego izolacja danych jest jej sednem.
- [ ] Wymienię trzy strategie i ich kompromisy (izolacja / koszt / skalowalność / migracje).
- [ ] Uzasadnię dobór strategii dla konkretnej skali i wymagań (np. stowarzyszenia).
- [ ] Wiem, jak identyfikować tenanta (subdomena / nagłówek / JWT) i czemu claim > nagłówek.
- [ ] Rozumiem propagację przez `ThreadLocal` i pułapki async / virtual threads / brak `clear()`.
- [ ] Skonfiguruję Hibernate 6 (`CurrentTenantIdentifierResolver` + `MultiTenantConnectionProvider`) i napiszę RLS policy.

### 🔲 Black-box check
- [ ] Co dokładnie robi `MultiTenantConnectionProvider` w trybie SCHEMA vs DATABASE?
- [ ] Dlaczego shared+discriminator to największe ryzyko wycieku i jak je domyka `@TenantId`/RLS?
- [ ] Czemu `ThreadLocal` nie wystarcza przy `@Async`/`CompletableFuture` i co robi `ScopedValue`?
- [ ] Jak PostgreSQL RLS wymusza izolację, gdy aplikacja zapomni `WHERE tenant_id`? (POLICY, `FORCE`, rola bez `BYPASSRLS`)
- [ ] Jak migrujesz schemat przy 500 schematach-per-tenant? (pętla Flyway per `search_path`)

### 🎤 Pytania rekrutacyjne
- [ ] „Jakie znasz strategie multitenancy i który kompromis wybierają?"
- [ ] „Jak zapewnisz, że tenant A nie zobaczy danych tenanta B?" (RLS/@TenantId + testy cross-tenant, OWASP A01)
- [ ] „Skąd bierzesz identyfikator tenanta i jak go propagujesz do warstwy danych?"
- [ ] „Jak wygląda migracja bazy przy tysiącach tenantów?"
- [ ] „Kiedy wybrałbyś database-per-tenant, a kiedy shared schema?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/09-architektura.csv`.*
```
Co to jest multitenancy?;Jedna instancja aplikacji obsługuje wielu izolowanych tenantów (najemców) z izolacją danych między nimi.
Co jest sednem multitenancy?;Izolacja danych — dane jednego tenanta muszą być niewidoczne dla innych.
Jaki jest główny kompromis w multitenancy?;Izolacja vs koszt/skalowalność — im silniejsza izolacja, tym większy narzut operacyjny.
Trzy strategie izolacji danych?;Database-per-tenant, schema-per-tenant, shared schema + discriminator (tenant_id).
Zalety database-per-tenant?;Najsilniejsza izolacja i bezpieczeństwo, łatwy backup/restore i usunięcie danych per tenant.
Wady database-per-tenant?;Kosztowne operacyjnie: N x migracje, N x pule połączeń, słabe przy tysiącach tenantów.
Na czym polega schema-per-tenant?;Jedna baza PostgreSQL, osobny schema (namespace tabel) na tenanta — kompromis izolacja/koszt.
Wady schema-per-tenant?;Migracje wciąż N-krotne (per schemat) i puchnący pg_catalog przy tysiącach schematów.
Na czym polega shared schema + discriminator?;Wspólne tabele, kolumna tenant_id rozróżnia właściciela wiersza.
Największe ryzyko strategii discriminator?;Wyciek danych między tenantami, gdy zapytanie pominie WHERE tenant_id (Broken Access Control).
Która strategia jest najtańsza i najskalowalniejsza?;Shared schema + discriminator (jedna migracja, natychmiastowy onboarding, tysiące tenantów).
Jak identyfikować tenanta w żądaniu?;Subdomena, nagłówek HTTP, albo (najbezpieczniej) podpisany claim tenant_id w JWT.
Czemu claim w JWT jest lepszy niż nagłówek do identyfikacji tenanta?;Claim jest podpisany — klient nie może go sfałszować, surowy nagłówek klient wpisuje sam.
Jak propaguje się tenanta do warstwy danych?;Przez TenantContext oparty o ThreadLocal, ustawiany na wejściu żądania i czyszczony w finally.
Dlaczego trzeba czyścić ThreadLocal tenanta?;Wątki (z puli i wirtualne) są recyklingowane — brak clear() = następne żądanie dostanie cudzego tenanta.
Problem z ThreadLocal przy @Async/CompletableFuture?;Kontekst nie przechodzi do wątku roboczego automatycznie — trzeba go jawnie skopiować (TaskDecorator/ContextSnapshot).
Jakie dwa beany dostarcza się Hibernate dla multitenancy?;CurrentTenantIdentifierResolver (kto jest tenantem) i MultiTenantConnectionProvider (właściwe Connection).
Co robi MultiTenantConnectionProvider w trybie SCHEMA?;Ustawia SET search_path na schemat tenanta (i resetuje go przy zwrocie połączenia do puli).
Do czego służy adnotacja @TenantId w Hibernate 6?;Hibernate sam dokłada tenant_id do INSERTów i warunku WHERE — dla strategii discriminator, bez ręcznego filtrowania.
Co to PostgreSQL Row-Level Security (RLS)?;Mechanizm bazy dokładający POLICY do każdego zapytania, wymuszający izolację nawet gdy aplikacja pominie WHERE tenant_id.
Co musi być spełnione, by RLS działało?;Aplikacja łączy się rolą bez BYPASSRLS i nie jako właściciel tabeli (lub FORCE ROW LEVEL SECURITY).
Rekomendacja multitenancy dla projektu rekrutacyjnego stowarzyszeń?;Schema-per-tenant jako kandydat główny; shared schema + discriminator + RLS jako alternatywa przy dużej skali.
Jak multitenancy gra z Flowable?;Flowable natywnie zna tenantId — encje procesowe znakuje się tenantId spójnym z TenantContext.
Cross-tenant data leak to który błąd OWASP?;Broken Access Control (A01) — krytyczny błąd w systemach SaaS.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]] · [[wiedza/08-bezpieczenstwo/owasp]] · [[wiedza/06-persystencja/flyway]] · [[projekt-rekrutacja/docs/ARCHITEKTURA]]
