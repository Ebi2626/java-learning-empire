---
temat: "Transakcje w Spring (@Transactional)"
faza: 6
status: nieopanowany
priorytet: 🔴
tags: [java, spring, transakcje, bazy]
powiazane: ["[[wiedza/06-persystencja/sql-podstawy]]", "[[wiedza/05-spring/proxy-aop]]", "[[wiedza/06-persystencja/jpa-hibernate]]"]
---

# Transakcje w Spring (@Transactional)

> **TL;DR:** `@Transactional` to **deklaratywne** zarządzanie transakcją — Spring owija bean **proxy** (AOP),
> które **otwiera transakcję** przed metodą, **commit** po sukcesie i **rollback** po wyjątku. Klucz: rollback
> domyślnie **tylko dla `RuntimeException`/`Error`** (checked exception → potrzebny `rollbackFor`), a **self-invocation**
> i metody `private`/`final` **omijają proxy** — adnotacja wtedy nie działa. Granica transakcji = **warstwa serwisu**.

## 1. Co — czym jest transakcja i API

**Transakcja** to **jednostka pracy** (unit of work) — grupa operacji, które muszą wykonać się **atomowo**: albo
wszystkie razem (`commit`), albo żadna (`rollback`). Gwarantuje własności **ACID** ([[wiedza/06-persystencja/sql-podstawy]]):

- **Atomicity** — całość albo nic.
- **Consistency** — baza przechodzi między poprawnymi stanami (constraints, FK).
- **Isolation** — równoległe transakcje nie widzą swoich niezacommitowanych zmian (poziom zależy od `isolation`).
- **Durability** — po `commit` zmiany są trwałe (WAL / redo log).

W Springu masz dwa style:

**Deklaratywny (`@Transactional`)** — dominujący, oparty na AOP:
```java
@Service
public class OrderService {

    private final OrderRepository orders;
    private final PaymentClient payments;

    @Transactional  // proxy: begin → metoda → commit / rollback
    public void placeOrder(OrderCmd cmd) {
        Order o = orders.save(new Order(cmd));   // 1
        payments.charge(cmd.card(), o.total());  // 2 — jeśli rzuci, cofamy też save
    }
}
```

**Programistyczny (`TransactionTemplate`)** — pełna kontrola, gdy potrzebujesz warunkowego commitu/rollbacku
albo bardzo wąskiej granicy:
```java
private final TransactionTemplate tx;  // wstrzyknięty, opakowuje PlatformTransactionManager

public void placeOrder(OrderCmd cmd) {
    tx.executeWithoutResult(status -> {
        orders.save(new Order(cmd));
        if (invalid(cmd)) status.setRollbackOnly();  // ręczne markowanie
    });
}
```
Deklaratywny jest czytelniejszy i mniej boilerplate; programistyczny daje kontrolę i działa bez ograniczeń proxy.

## 2. Jak — proxy, propagacja, rollback pod spodem

### Proxy (AOP) — sedno mechanizmu
`@Transactional` **nie jest magią kompilatora** — to **AOP przez proxy** ([[wiedza/05-spring/proxy-aop]]). Przy starcie
kontekstu Spring wykrywa bean z `@Transactional` i podmienia go na **proxy** (JDK dynamic proxy jeśli jest interfejs,
inaczej CGLIB — podklasa). Proxy owija każde publiczne wywołanie w advice (`TransactionInterceptor`):

```
klient → [PROXY]  ──► begin tx (getConnection, setAutoCommit(false))
                  ──► TWOJA metoda serwisu
                  ──► sukces  → commit
                  ──► wyjątek → rollback (wg reguł) → rethrow
```

Pod spodem stoi `PlatformTransactionManager` (np. `DataSourceTransactionManager` dla JDBC,
`JpaTransactionManager` dla JPA/Hibernate). To on wywołuje `begin`/`commit`/`rollback` na fizycznym połączeniu.

### Związanie z połączeniem i wątkiem (ThreadLocal)
Kluczowe: **transakcja jest związana z bieżącym wątkiem** przez `TransactionSynchronizationManager`, który trzyma
w **`ThreadLocal`** aktualne połączenie / `EntityManager`. Dzięki temu `repository.save()` wywołane głębiej w tej
samej metodzie **dostaje to samo połączenie** — nie zaczyna nowej transakcji. **Konsekwencja:** transakcja **nie
przechodzi między wątkami** (nowy wątek / `@Async` / reactive = nowa lub brak transakcji).

### Propagacja (propagation) — co się dzieje, gdy metoda `@Transactional` woła inną `@Transactional`

| Tryb | Zachowanie | Scenariusz |
|---|---|---|
| **REQUIRED** (domyślny) | Jest transakcja → **dołącz**; brak → **utwórz** nową. | Standard — jedna logiczna transakcja na cały flow serwisu. |
| **REQUIRES_NEW** | **Zawsze nowa**, bieżącą **zawiesza** (suspend). Commit/rollback niezależny. | **Audyt / log**, który ma zostać zapisany **nawet gdy** transakcja główna zrobi rollback. |
| **NESTED** | **Savepoint** wewnątrz bieżącej transakcji. Rollback cofa tylko do savepointu. | Częściowy rollback fragmentu bez zabijania całości (wymaga wsparcia JDBC savepoints). |
| **SUPPORTS** | Jest transakcja → użyj; brak → działaj **bez** transakcji. | Metoda odczytowa, której transakcja nie jest konieczna, ale chętnie dołączy. |
| **MANDATORY** | **Musi** istnieć transakcja, inaczej **wyjątek**. | Metoda, która wyłącznie jako część większej jednostki pracy ma sens. |
| **NEVER** | **Nie może** być transakcji, inaczej **wyjątek**. | Kod, który świadomie nie może działać transakcyjnie. |
| **NOT_SUPPORTED** | **Zawiesza** bieżącą i działa **bez** transakcji. | Długa operacja raportowa, która nie ma trzymać transakcji/locka. |

**Uwaga do REQUIRES_NEW i NESTED:** oba działają tylko **przez proxy** — czyli gdy wołasz je **z innego beana**
(inaczej self-invocation, patrz niżej). REQUIRES_NEW bierze **drugie połączenie** z puli — przy małej puli grozi
zakleszczeniem/wyczerpaniem połączeń.

### Rollback rules — KLUCZOWA PUŁAPKA
Domyślnie Spring robi **rollback tylko dla `RuntimeException` (unchecked) i `Error`**. Dla **checked exception
(`Exception`) commituje!** To spadek po EJB i najczęstsze źródło „dlaczego dane się zapisały mimo błędu".

```java
@Transactional                                   // IOException (checked) → COMMIT (!)
public void a() throws IOException { ... }

@Transactional(rollbackFor = Exception.class)    // teraz rollback też dla checked
public void b() throws IOException { ... }

@Transactional(noRollbackFor = BusinessWarn.class) // wyjątek biznesowy bez rollbacku
public void c() { ... }
```

### readOnly = true
`@Transactional(readOnly = true)` to **hint** dla warstwy niżej. W Hibernate ustawia `FlushMode.MANUAL` → **brak
dirty checking / flushowania** na końcu, mniej pracy i pamięci. Baza może optymalizować (np. read-only routing do repliki).
Nie egzekwuje zakazu zapisu na poziomie SQL — to optymalizacja, nie zabezpieczenie. Stosuj do metod czysto odczytowych.

### Isolation
`@Transactional(isolation = Isolation.REPEATABLE_READ)` **deleguje do bazy** — Spring tłumaczy to na
`SET TRANSACTION ISOLATION LEVEL`. Poziomy (rosnąca izolacja, malejąca współbieżność):
`READ_UNCOMMITTED` → `READ_COMMITTED` → `REPEATABLE_READ` → `SERIALIZABLE`. `DEFAULT` = poziom bazy
(Postgres: `READ_COMMITTED`). Szczegóły anomalii (dirty/non-repeatable/phantom read) → [[wiedza/06-persystencja/sql-podstawy]].

### Rollback-only
Gdy w zagnieżdżonej metodzie (propagacja REQUIRED, ta sama transakcja fizyczna) poleci wyjątek i zostanie złapany
wyżej, transakcja jest już **oznaczona jako `rollback-only`**. Wtedy zewnętrzny `commit` rzuci
`UnexpectedRollbackException` — „Transaction silently rolled back because it has been marked as rollback-only".
To mechanizm bezpieczeństwa: raz zepsutej transakcji nie da się „uratować" commitem.

## 3. Dlaczego / kiedy — granice i pułapki

**Gdzie stawiać granicę:** na **warstwie serwisu** (use-case = jedna transakcja), **nie** w repozytorium (za wąsko —
każda metoda osobno) i **nie** w kontrolerze (za szeroko — transakcja żyje przez serializację JSON, walidację itd.).

### Pułapki (must-know na rekrutacji)

1. **Self-invocation** — wywołanie metody `@Transactional` **z tej samej klasy** omija proxy → **adnotacja nie działa**.
   ```java
   @Service
   class ReportService {
       public void run() {
           doAudit();  // WYWOŁANIE WEWNĘTRZNE → this.doAudit(), nie przez proxy → BEZ nowej tx
       }
       @Transactional(propagation = Propagation.REQUIRES_NEW)
       public void doAudit() { ... }   // REQUIRES_NEW ignorowane!
   }
   ```
   Fix: wydziel do **osobnego beana**, wstrzyknij self-proxy, albo użyj `TransactionTemplate`.

2. **`private` / `final` / `static`** — proxy (CGLIB) nadpisuje metody przez dziedziczenie; nie da się przechwycić
   metody `private` ani `final`. `@Transactional` na takiej metodzie jest **cicho ignorowane**. Metoda musi być
   `public` (i nie-`final`) na proxowanym beanie.

3. **Połknięty wyjątek = brak rollbacku** — jeśli w metodzie transakcyjnej złapiesz wyjątek i **nie rzucisz go
   dalej**, interceptor go nie zobaczy → **commit** mimo błędu (lub `rollback-only` przy zagnieżdżeniu).

4. **Checked exception bez `rollbackFor`** — patrz sekcja 2: commit zamiast rollbacku.

5. **`LazyInitializationException`** — encja jest **managed tylko w obrębie transakcji / otwartego
   `EntityManager`** ([[wiedza/06-persystencja/jpa-hibernate]]). Dostęp do leniwej kolekcji **poza** transakcją
   (np. w kontrolerze/serializacji DTO) → `LazyInitializationException: could not initialize proxy - no Session`.
   Fix: pobierz dane w transakcji (`JOIN FETCH`, `@EntityGraph`, mapowanie do DTO w serwisie) — **nie** przez
   `OpenSessionInView` (antywzorzec, przecieka transakcję do widoku).

6. **Długie transakcje** — trzymają **połączenie z puli** i **locki** przez cały czas. Wołanie zewnętrznego API,
   wysyłka maila czy `Thread.sleep` w transakcji = wyczerpanie puli i zakleszczenia. Trzymaj transakcje krótkie;
   I/O na zewnątrz transakcji.

## Przykład w praktyce
Serwis zamówień: główny flow w jednej transakcji, ale **audyt niezależny** od jej wyniku.

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orders;
    private final AuditService audit;      // OSOBNY bean → REQUIRES_NEW zadziała przez proxy

    @Transactional(rollbackFor = PaymentException.class)  // checked też cofa
    public void placeOrder(OrderCmd cmd) throws PaymentException {
        audit.log("PLACE_ORDER attempt " + cmd.id());     // zapisze się nawet przy rollbacku niżej
        Order o = orders.save(new Order(cmd));
        charge(cmd);   // rzuca PaymentException → rollback save(), ale log audytu ZOSTAJE
    }
}

@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)  // własna, niezależna transakcja
    public void log(String msg) { /* insert do tabeli audit */ }
}
```
Gdyby `AuditService.log` było metodą tej samej klasy `OrderService` — **self-invocation** — audyt trafiłby do tej
samej transakcji i **zniknął** przy rollbacku. Stąd wydzielenie do osobnego beana.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, że `@Transactional` działa przez **proxy (AOP)**, i co proxy robi (begin/commit/rollback).
- [ ] Wymienię wszystkie tryby **propagacji** ze scenariuszem dla REQUIRES_NEW i NESTED.
- [ ] Wiem, że domyślny rollback dotyczy **tylko** `RuntimeException`/`Error` i jak to zmienić.
- [ ] Rozpoznam i naprawię **self-invocation** oraz `@Transactional` na `private`/`final`.
- [ ] Wyjaśnię związek transakcji z **połączeniem i wątkiem** (`ThreadLocal`) i skąd `LazyInitializationException`.

### 🔲 Black-box check
- [ ] Co dokładnie robi Spring przy starcie z beanem `@Transactional`? (podmiana na proxy CGLIB/JDK)
- [ ] Kto faktycznie woła `commit`/`rollback`? (`PlatformTransactionManager` na połączeniu)
- [ ] Gdzie trzymane jest bieżące połączenie/transakcja? (`ThreadLocal` w `TransactionSynchronizationManager`)
- [ ] Czemu REQUIRES_NEW wywołane z tej samej klasy nie tworzy nowej transakcji?
- [ ] Co to `UnexpectedRollbackException` i skąd „rollback-only"?

### 🎤 Pytania rekrutacyjne
- [ ] „Jak działa `@Transactional` pod spodem?" (proxy + AOP + transaction manager + ThreadLocal)
- [ ] „Rzucam checked exception w metodzie transakcyjnej — co się stanie z danymi?" (commit, bez `rollbackFor`)
- [ ] „Czemu `@Transactional` na metodzie prywatnej / wołanej z tej samej klasy nie działa?"
- [ ] „Kiedy REQUIRES_NEW, a kiedy NESTED?" (niezależny audyt vs savepoint)
- [ ] „Skąd `LazyInitializationException` i jak go uniknąć?"
- [ ] „Gdzie stawiasz granicę transakcji i dlaczego nie w kontrolerze?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/06-persystencja.csv`.*
```
Czym jest transakcja?;Jednostką pracy (unit of work) wykonaną atomowo — commit wszystko albo rollback nic; gwarantuje ACID.
Co oznacza ACID?;Atomicity, Consistency, Isolation, Durability.
Jak działa @Transactional pod spodem?;Spring owija bean proxy (AOP), które otwiera transakcję przed metodą, commituje po sukcesie i robi rollback po wyjątku.
Jaki proxy stosuje Spring do @Transactional?;JDK dynamic proxy (gdy jest interfejs) albo CGLIB (podklasa) — bez interfejsu.
Kto faktycznie robi commit/rollback?;PlatformTransactionManager (np. DataSourceTransactionManager, JpaTransactionManager) na fizycznym połączeniu.
Z czym związana jest transakcja i jak?;Z bieżącym wątkiem — połączenie/EntityManager trzymane w ThreadLocal (TransactionSynchronizationManager).
Deklaratywne vs programistyczne zarządzanie transakcją?;Deklaratywne = @Transactional (AOP, mniej kodu); programistyczne = TransactionTemplate (pełna kontrola, bez ograniczeń proxy).
Domyślny tryb propagacji?;REQUIRED — dołącz do istniejącej transakcji lub utwórz nową.
Co robi REQUIRES_NEW?;Zawsze tworzy nową transakcję, zawieszając bieżącą; commit/rollback niezależny — np. audyt niezależny od rollbacku głównego flow.
Co robi propagacja NESTED?;Tworzy savepoint w bieżącej transakcji; rollback cofa tylko do savepointu (wymaga savepoints w JDBC).
Co robi MANDATORY?;Wymaga istniejącej transakcji — bez niej rzuca wyjątek.
Co robi NEVER?;Rzuca wyjątek, jeśli transakcja istnieje — kod nie może działać transakcyjnie.
Co robi NOT_SUPPORTED?;Zawiesza bieżącą transakcję i wykonuje metodę bez transakcji.
Dla jakich wyjątków @Transactional robi rollback domyślnie?;Tylko dla RuntimeException (unchecked) i Error; dla checked exception COMMITuje.
Jak wymusić rollback dla checked exception?;@Transactional(rollbackFor = Exception.class).
Co daje readOnly=true?;Hint optymalizacyjny — w Hibernate FlushMode.MANUAL (brak dirty checking), baza może routować do repliki; to nie zabezpieczenie.
Do czego służy isolation w @Transactional?;Deleguje do bazy poziom izolacji (READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE...) — Spring wykonuje SET TRANSACTION ISOLATION LEVEL.
Czemu self-invocation łamie @Transactional?;Wywołanie metody z tej samej klasy idzie przez this, nie przez proxy — advice transakcyjne się nie uruchamia.
Czemu @Transactional na metodzie private/final nie działa?;Proxy (CGLIB) nadpisuje metody dziedzicząc — nie może przechwycić private ani final; metoda musi być public i nie-final.
Skąd LazyInitializationException?;Encja jest managed tylko w transakcji/otwartej sesji; dostęp do leniwej relacji poza transakcją rzuca ten wyjątek.
Co to UnexpectedRollbackException?;Zewnętrzny commit na transakcji oznaczonej rollback-only (bo zagnieżdżony wyjątek ją zepsuł) — nie da się jej uratować commitem.
Gdzie stawiać granicę transakcji?;Na warstwie serwisu (use-case = jedna transakcja), nie w repozytorium (za wąsko) ani w kontrolerze (za szeroko).
Czemu długie transakcje są groźne?;Trzymają połączenie z puli i locki — I/O/API/sleep w transakcji wyczerpuje pulę i powoduje zakleszczenia.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/06-persystencja/sql-podstawy]] · [[wiedza/05-spring/proxy-aop]] · [[wiedza/06-persystencja/jpa-hibernate]]
