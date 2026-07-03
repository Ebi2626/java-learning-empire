---
temat: "Domain-Driven Design (DDD) lite"
faza: 9
status: nieopanowany
priorytet: 🟡
tags: [java, architektura, ddd]
powiazane: ["[[wiedza/09-architektura/warstwy-hexagonal]]", "[[wiedza/01-jezyk/record-sealed-enum]]", "[[wiedza/06-persystencja/spring-data-jpa]]", "[[wiedza/09-architektura/komunikacja-async]]"]
---

# Domain-Driven Design (DDD) lite

> **TL;DR:** DDD (Eric Evans) to podejście do **złożonej domeny biznesowej**: modelujesz ją w kodzie tak, jak
> mówi o niej biznes (**ubiquitous language**), dzielisz na **bounded contexts** (granice modelu), a wewnątrz
> budujesz z klocków taktycznych — **entity**, **value object**, **aggregate** (z **aggregate root** jako
> jedynym wejściem), **repository**, **domain service**, **domain event**. Sedno: **rich domain model**
> (zachowanie w encjach), nie **anemic** (same gettery/settery + logika w serwisach). „Lite" = bierzesz taktykę
> tam, gdzie domena jest złożona, a zwykły CRUD zostawiasz w spokoju.

## 1. Co — definicja i mapa pojęć

**DDD** to zestaw praktyk projektowania oprogramowania wokół **modelu domeny** — nie wokół bazy danych czy
framoworka. Dzieli się na dwie warstwy:

**STRATEGICZNE DDD** (jak pociąć duży system):
- **Ubiquitous language** — jeden, wspólny język biznesu i programistów. Ta sama nazwa w rozmowie, w kodzie,
  w klasie, w metodzie. Jeśli biznes mówi „kandydat aplikuje na ofertę", to w kodzie jest `candidate.applyFor(offer)`,
  a nie `applicationService.create(dto)`. Język żyje **w obrębie jednego kontekstu**.
- **Bounded context** — **granica**, w której model i język są spójne. Ten sam termin znaczy co innego w różnych
  kontekstach: `User` w kontekście *tożsamości* (login, hasło, tenant) to nie ten sam `User` co *kandydat* w
  kontekście *rekrutacji* (CV, doświadczenie, statusy procesu). Nie próbuj robić jednej „God-class" `User` dla
  całego systemu — rozdziel na modele per kontekst.
- **Context map** — mapa relacji między kontekstami (kto od kogo zależy, gdzie są tłumaczenia/anti-corruption
  layer, gdzie shared kernel). Odpowiada na pytanie „jak konteksty ze sobą gadają".
- **Subdomeny** — części domeny biznesowej:
  - **core domain** — to, co daje przewagę konkurencyjną, gdzie inwestujesz najwięcej DDD (np. algorytm
    matchowania kandydatów);
  - **supporting** — potrzebne, ale nieunikalne (np. workflow procesu rekrutacji);
  - **generic** — kupujesz/bierzesz gotowe (autoryzacja, wysyłka maili, płatności).

**TAKTYCZNE DDD** (klocki wewnątrz jednego kontekstu) — sekcja 2.

Kluczowe rozróżnienie: **bounded context** to pojęcie *rozwiązania/modelu* (kod, zespół), **subdomena** to
pojęcie *problemu/biznesu*. Idealnie mapują się 1:1, ale nie zawsze.

## 2. Jak — agregaty, granice, klocki taktyczne

Klocki taktyczne to budulec modelu wewnątrz kontekstu:

**Entity** — obiekt z **tożsamością** (identity) i **cyklem życia**. Dwie encje o różnych atrybutach, ale tym
samym ID to ta sama encja; równość liczona po ID, nie po polach. Ma stan, który zmienia się w czasie
(`process.moveToStage(...)`).

**Value object** — obiekt **bez tożsamości**, **niemutowalny**, definiowany wyłącznie przez swoje **atrybuty**.
Dwa VO o tych samych polach są równe. To idealne zastosowanie **rekordu Javy** ([[wiedza/01-jezyk/record-sealed-enum]]):
niemutowalny, `equals`/`hashCode` po wartości z automatu, zwięzły. Przykłady: `Money`, `Email`, `DateRange`,
`Address`. VO może (i powinien) zawierać walidację w kanonicznym konstruktorze i zachowanie (`money.add(...)`).

**Aggregate** — **klaster** encji i value objectów traktowany jako **jedna całość** w kontekście spójności i
transakcji. **Aggregate root** to jedna encja będąca **jedynym punktem wejścia** — świat zewnętrzny trzyma
referencję tylko do roota i tylko przez niego modyfikuje wnętrze. Reguły:
- **Granica transakcyjna i spójności**: jeden agregat = jedna transakcja = jedna granica **invariantów**
  (niezmienników). W obrębie agregatu spójność jest natychmiastowa (strong consistency); między agregatami —
  **eventual consistency** (przez domain events).
- **Referencje między agregatami po ID, nie po obiekcie**: `Process` trzyma `CandidateId`, a nie całe
  `Candidate`. To utrzymuje małe granice transakcyjne, upraszcza ładowanie i zapobiega „wciąganiu" pół bazy
  do pamięci. Duże agregaty = kontencja, deadlocki, problemy z lazy loadingiem.
- Rób agregaty **małe** — najlepiej root + kilka VO.

**Repository** — kolekcjo-podobny interfejs do **odczytu i zapisu agregatów** (jeden repo na aggregate root,
nie na każdą encję). Ukrywa persystencję; w Springu realizowany zwykle przez Spring Data JPA
([[wiedza/06-persystencja/spring-data-jpa]]). W hexagonalu interfejs repo to **port** w domenie, a
implementacja JPA to **adapter**.

**Domain service** — logika domenowa, która **nie pasuje** naturalnie do żadnej encji ani VO (bo dotyczy
kilku agregatów lub jest bezstanowa), a wciąż jest częścią domeny — np. `TransferService`, `MatchingService`.
To NIE to samo co springowy `@Service` warstwy aplikacji; domain service jest czystym obiektem domeny.

**Domain event** — fakt, że **coś się wydarzyło** w domenie (nazwa w czasie przeszłym): `CandidateApplied`,
`ProcessAdvanced`. Publikowany przez agregat, konsumowany w innym miejscu/kontekście — to sposób na
**eventual consistency** i luźne sprzężenie ([[wiedza/09-architektura/komunikacja-async]]). W Springu:
`ApplicationEventPublisher` synchronicznie lub outbox + broker asynchronicznie.

**Factory** — hermetyzuje **złożone tworzenie** agregatu/VO, gdy konstruktor to za mało (np. rekonstrukcja z
zdarzeń, złożone invarianty). Może być metodą statyczną na agregacie (`Process.start(...)`).

Przepływ w hexagonalu ([[wiedza/09-architektura/warstwy-hexagonal]]): use case (application service) ładuje
agregat przez repository-port → wywołuje **metodę domenową na agregacie** (tam jest logika i invarianty) →
zapisuje agregat → publikuje domain events. Warstwa aplikacji orkiestruje, domena decyduje.

## 3. Dlaczego / kiedy — anemic vs rich, kiedy DDD

**Anemic domain model (antywzorzec, Fowler):** encje to gołe struktury danych — same gettery/settery, zero
zachowania — a cała logika siedzi w „grubych" serwisach. Wygląda obiektowo, ale nim nie jest: łamie
enkapsulację (każdy może wprowadzić obiekt w niespójny stan przez settery), logika się rozprasza i duplikuje,
invarianty nie mają jednego strażnika.

**Rich domain model:** zachowanie mieszka **w encjach/agregatach** obok danych, które chroni. Zamiast
`process.setStatus(REJECTED)` masz `process.reject(reason)`, która pilnuje, czy przejście jest dozwolone.
Settery znikają; stan zmienia się tylko przez metody wyrażające intencję biznesową.

**Debata i kompromis z JPA.** Encje JPA ciągną w stronę anemii: wymagają konstruktora bezargumentowego,
mutowalnych pól, gettery/settery bywają wygodne dla proxy/lazy loadingu, a `@Entity` wiąże domenę z ORM.
Dwie szkoły:
- **Pragmatyczna (lite):** encje JPA są jednocześnie encjami domenowymi, ale zachowanie i tak wkładziesz na
  nich (metody biznesowe zamiast setterów), a settery robisz `protected`/prywatne. Prosto, jeden zestaw klas.
- **Purystyczna:** czysty model domeny bez adnotacji + osobne encje persystencji + mapowanie (anti-corruption).
  Więcej kodu, ale domena nie wie o JPA. Sensowne, gdy core jest naprawdę złożony.
  Praktyczny środek: bogaty model, prywatne settery, VO jako `@Embeddable` (rekordy w JPA są ograniczone),
  a jeśli koszt mapowania jest do przełknięcia — separacja.

**Kiedy DDD:**
- **TAK** — złożona domena z bogatymi regułami, dużo zachowania, długi horyzont, wielu ludzi/zespołów,
  język biznesu jest istotny (rekrutacja, finanse, ubezpieczenia).
- **NIE / przesada** — proste **CRUD**, „przeklep formularza do tabeli", mało reguł. Wtedy agregaty,
  repozytoria-porty i eventy to zbędny narzut; wystarczy prosta warstwa serwisów. DDD ma **koszt** (nauka,
  boilerplate, dyscyplina) — płać go tam, gdzie złożoność go zwraca.
- Możesz mieszać: **core** w pełnym DDD, subdomeny **supporting/generic** jako zwykły CRUD w tym samym systemie.

## Przykład w praktyce (projekt rekrutacyjny)

Przykładowe **bounded contexts**: **Rekrutacja** (kandydat, oferta, aplikacja — core), **Tożsamość/Tenant**
(user, uprawnienia, multi-tenancy — supporting/generic), **Proces** (workflow etapów rekrutacji — supporting).
`User` z kontekstu tożsamości i `Candidate` z rekrutacji to osobne modele, powiązane po ID.

Agregat z zachowaniem + value object jako rekord (Java 21, Spring Boot 3):

```java
// Value object — rekord: niemutowalny, equals/hashCode po wartości, walidacja w konstruktorze
public record Email(String value) {
    public Email {
        if (value == null || !value.contains("@"))
            throw new IllegalArgumentException("Nieprawidłowy email: " + value);
    }
}

// Aggregate root — jedyne wejście, chroni invarianty, publikuje domain event
public class Process {                       // encja: tożsamość + cykl życia
    private final ProcessId id;
    private final CandidateId candidateId;   // referencja do innego agregatu PO ID, nie po obiekcie
    private Stage stage;
    private final List<DomainEvent> events = new ArrayList<>();

    public Process(ProcessId id, CandidateId candidateId) {
        this.id = id;
        this.candidateId = candidateId;
        this.stage = Stage.APPLIED;
    }

    // zachowanie w agregacie (rich model), nie setter — pilnuje reguły przejścia
    public void advance() {
        if (stage == Stage.HIRED || stage == Stage.REJECTED)
            throw new IllegalStateException("Proces zakończony, nie można awansować");
        this.stage = stage.next();
        events.add(new ProcessAdvanced(id, stage));   // domain event
    }

    public List<DomainEvent> pullEvents() { var e = List.copyOf(events); events.clear(); return e; }
    // brak publicznych setterów — stan zmienia się tylko przez metody intencji
}

// Repository per aggregate root (port; adapter = Spring Data JPA)
public interface ProcessRepository {
    Optional<Process> findById(ProcessId id);
    Process save(Process process);
}
```

Warstwa aplikacji tylko orkiestruje: `repo.findById(id) → process.advance() → repo.save(process) →`
opublikuj `pullEvents()`. Logika (kiedy wolno awansować) żyje w agregacie.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię DDD i różnicę strategiczne vs taktyczne jednym akapitem.
- [ ] Rozróżnię entity vs value object i uzasadnię rekord jako VO.
- [ ] Wytłumaczę aggregate, aggregate root i po co referencje po ID.
- [ ] Odróżnię anemic od rich model i wskażę kompromis z JPA.
- [ ] Powiem, kiedy DDD to przesada (prosty CRUD) a kiedy się zwraca.

### 🔲 Black-box check
- [ ] Co to bounded context i czemu ten sam termin znaczy co innego w dwóch kontekstach?
- [ ] Dlaczego aggregate root ma być jedynym punktem wejścia i co to daje transakcyjnie?
- [ ] Czemu referencje między agregatami idą po ID, a nie po obiekcie?
- [ ] Gdzie w hexagonalu leży repository-port, a gdzie adapter JPA?
- [ ] Jak domain events dają eventual consistency między agregatami?

### 🎤 Pytania rekrutacyjne
- [ ] „Czym różni się entity od value object?"
- [ ] „Co to aggregate root i po co on jest?"
- [ ] „Anemic vs rich domain model — co wybierasz i czemu?"
- [ ] „Jak pogodzić czystą domenę DDD z encjami JPA?"
- [ ] „Kiedy DDD to przerost formy?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/09-architektura.csv`.*
```
Kto sformułował DDD?;Eric Evans (książka "Domain-Driven Design", 2003) — podejście do złożonej domeny biznesowej.
Co to ubiquitous language?;Wspólny język biznesu i programistów, spójny w rozmowie i w kodzie, w obrębie jednego bounded context.
Co to bounded context?;Granica, w której model i język są spójne; ten sam termin może znaczyć co innego w innym kontekście.
Co to context map?;Mapa relacji i zależności między bounded contexts (kto od kogo zależy, gdzie tłumaczenia/ACL).
Subdomeny w DDD?;Core (przewaga konkurencyjna), supporting (potrzebne, nieunikalne), generic (gotowe/kupowane).
Czym różni się entity od value object?;Entity ma tożsamość i cykl życia (równość po ID); value object nie ma tożsamości, jest niemutowalny, równość po atrybutach.
Czemu rekord Javy pasuje na value object?;Niemutowalny, equals/hashCode po wartości z automatu, zwięzły, walidacja w konstruktorze kanonicznym.
Co to aggregate?;Klaster encji i value objectów traktowany jako całość w granicy spójności i transakcji.
Co to aggregate root?;Jedyny punkt wejścia do agregatu; świat zewnętrzny modyfikuje wnętrze tylko przez root.
Czemu referencje między agregatami po ID, nie po obiekcie?;Utrzymuje małe granice transakcyjne, upraszcza ładowanie, unika wciągania pół bazy do pamięci.
Granica transakcyjna w DDD?;Jeden agregat = jedna transakcja = jedna granica invariantów; między agregatami eventual consistency.
Co to repository w DDD?;Kolekcjo-podobny interfejs do zapisu/odczytu agregatów, jeden na aggregate root; w Springu Spring Data JPA.
Co to domain service?;Logika domenowa niepasująca do żadnej encji/VO (dotyczy kilku agregatów), wciąż część domeny; nie to samo co @Service aplikacji.
Co to domain event?;Fakt, że coś się wydarzyło w domenie (nazwa w czasie przeszłym), np. CandidateApplied; nośnik eventual consistency.
Co to factory w DDD?;Hermetyzuje złożone tworzenie/rekonstrukcję agregatu lub VO, gdy konstruktor to za mało.
Co to anemic domain model?;Antywzorzec: encje to gołe gettery/settery bez zachowania, logika rozproszona w serwisach; łamie enkapsulację.
Co to rich domain model?;Zachowanie i invarianty żyją w encjach/agregatach obok chronionych danych; zamiast setStatus() metoda intencji reject().
Problem DDD z JPA?;Encje JPA ciągną w anemię (bezarg. konstruktor, mutowalność, sprzężenie z ORM); rozwiązanie: bogaty model z prywatnymi setterami lub osobne encje persystencji.
Kiedy DDD to przesada?;Przy prostym CRUD z mało regułami; agregaty i porty to wtedy zbędny narzut — wystarczy prosta warstwa serwisów.
Jak DDD łączy się z hexagonal?;Repository-port i model domeny w rdzeniu, adaptery (JPA, REST) na zewnątrz; use case orkiestruje, agregat decyduje.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/09-architektura/warstwy-hexagonal]] · [[wiedza/01-jezyk/record-sealed-enum]] · [[wiedza/06-persystencja/spring-data-jpa]] · [[wiedza/09-architektura/komunikacja-async]]
