---
temat: "Architektura warstwowa vs hexagonalna (Ports & Adapters)"
faza: 9
status: nieopanowany
priorytet: 🟡
tags: [java, architektura]
powiazane: ["[[wiedza/09-architektura/ddd-lite]]", "[[wiedza/09-architektura/solid]]", "[[wiedza/07-spring/dependency-injection]]"]
---

# Architektura warstwowa vs hexagonalna (Ports & Adapters)

> **TL;DR:** **Layered** (controller → service → repository → entity) jest prosta i znana, ale domena
> *zależy w dół* od infrastruktury (JPA/Spring wyciekają do rdzenia). **Hexagonal** (Ports & Adapters)
> i pokrewna **Clean Architecture** *odwracają zależność* (**DIP**): domena definiuje **porty** (interfejsy),
> a infrastruktura (REST, JPA, kolejka) to **adaptery**, które od domeny zależą — nigdy odwrotnie.
> Zysk: testowalność i wymienialność. Koszt: więcej mapowania/boilerplate — dla prostego CRUD zwykle się nie opłaca.

## 1. Co — definicje i API

### Architektura warstwowa (layered / n-tier)
Klasyczny układ Spring Boot: żądanie płynie z góry na dół, zależności skierowane **w dół**.

```
┌─────────────────────────────────────────────┐
│  Controller  (REST, @RestController)         │  ← warstwa prezentacji
│        │ zależy w dół                        │
│  Service     (@Service, logika + @Transactional) │  ← warstwa aplikacji/domeny
│        │ zależy w dół                        │
│  Repository  (Spring Data JPA)               │  ← warstwa dostępu do danych
│        │ zależy w dół                        │
│  Entity      (@Entity — encje JPA)           │  ← model danych = model bazy
└─────────────────────────────────────────────┘
       ↓ wszystko wskazuje na infrastrukturę (bazę)
```

- **Zalety:** prosta, powszechnie znana, zero ceremonii — pasuje do większości aplikacji.
- **Wady:** logika domenowa **uzależnia się od infrastruktury** — `@Entity` z adnotacjami JPA jest
  jednocześnie modelem biznesowym; serwis operuje na typach z `jakarta.persistence`; `LAZY`/`@Transactional`
  wymuszają myślenie o bazie w warstwie „biznesowej". JPA **wycieka do góry**, testy wymagają bazy (albo H2),
  a wymiana Postgresa na coś innego dotyka całego stosu.

### Hexagonal (Ports & Adapters, Alistair Cockburn, 2005)
Rdzeń domenowy w środku heksagonu; komunikacja tylko przez **porty**.

```
        DRIVING / PRIMARY (kto woła aplikację)          DRIVEN / SECONDARY (co aplikacja woła)
     ┌──────────────┐                                 ┌──────────────────┐
     │ REST adapter │──▶ [Port IN: use case] ┐        │  JPA adapter     │
     │ (controller) │                        │        │ (implementacja)  │
     └──────────────┘                        ▼        └──────────────────┘
     ┌──────────────┐              ┌────────────────────┐        ▲
     │ CLI / test   │──▶ Port IN ─▶│   RDZEŃ DOMENOWY    │─▶ Port OUT (interfejs w domenie)
     └──────────────┘              │  (encje, use cases) │        ▲
                                   └────────────────────┘  ┌──────────────────┐
                                    NIE zna Springa ani     │ Kafka adapter    │
                                    bazy                    └──────────────────┘
```

- **Port** = interfejs należący do domeny. Dwa rodzaje:
  - **driving / primary / inbound** — API aplikacji, np. `PlaceOrderUseCase` (woła go controller);
  - **driven / secondary / outbound** — czego domena potrzebuje z zewnątrz, np. `OrderRepository`,
    `PaymentGateway` (interfejs **w domenie**, implementacja poza nią).
- **Adapter** = implementacja portu w infrastrukturze: `OrderRestController` (adapter portu IN),
  `JpaOrderRepository` (adapter portu OUT), `KafkaEventPublisher`.
- **Reguła:** domena **nie zna** frameworka ani bazy. Infrastruktura zależy od domeny — nie odwrotnie.

### Clean Architecture (Robert C. „Uncle Bob" Martin)
Koncentryczne kręgi; **The Dependency Rule**: zależności źródłowe wskazują tylko **do wewnątrz**.

```
( Frameworks & Drivers )  → web, DB, Spring
   ( Interface Adapters ) → controllers, presenters, gateways/repo impl
      ( Use Cases )       → logika aplikacyjna (application-specific)
         ( Entities )     → reguły biznesowe (enterprise-wide, najbardziej stabilne)
```
Pokrewna hexagonal: to samo odwrócenie zależności, inne słownictwo (kręgi zamiast portów). Entities/use cases
= rdzeń; adapters/frameworks = skorupa wymienialna.

## 2. Jak — porty, adaptery, inwersja zależności

### Fundament: Dependency Inversion Principle (DIP, „D" z SOLID)
> Moduły wysokopoziomowe **nie** powinny zależeć od niskopoziomowych — **oba** zależą od **abstrakcji**.
> Abstrakcje nie zależą od szczegółów — szczegóły zależą od abstrakcji.

W layered strzałka idzie `Service → Repository (impl JPA)` — wysoki poziom zależy od niskiego.
Hexagonal **odwraca** ją: domena definiuje `interface OrderRepository`, a `JpaOrderRepository` go **implementuje**.
W czasie kompilacji strzałka `JpaOrderRepository → OrderRepository` biegnie **do domeny**. To właśnie *inwersja*.

Mechanicznie umożliwia to **Dependency Injection** ([[wiedza/07-spring/dependency-injection]]): Spring wstrzykuje
konkretny adapter tam, gdzie domena widzi tylko interfejs port. Domena kompiluje się bez `spring-*` na classpath.

### Pakietowanie: by-layer vs by-feature
- **by-layer** (pakiety `controller/`, `service/`, `repository/`) — typowe dla layered; „poziome" cięcie.
- **by-feature / by-package** (`order/`, `payment/` — każdy z własnymi portami i adapterami) — sprzyja hexagonal,
  wysoka spójność (*high cohesion*), granice modułów widać w strukturze pakietów. Wewnątrz featury często:
  `domain/`, `application/` (porty + use cases), `adapter/in/`, `adapter/out/`.

### Mapowanie modeli
Cena odwrócenia: encja domenowa `Order` ≠ encja JPA `OrderEntity` ≠ DTO REST `OrderResponse`. Adapter mapuje
między nimi (MapStruct/ręcznie). Domena zostaje czysta, ale mapperów przybywa — to główny źródło boilerplate.

## 3. Dlaczego / kiedy — koszty vs korzyści, pragmatyzm

**Korzyści hexagonal/clean:**
- **Testowalność** — use case testujesz na fake/mock porcie OUT, bez bazy i bez web (szybkie testy jednostkowe
  domeny; Testcontainers tylko dla adapterów).
- **Wymienialność infrastruktury** — REST→gRPC, JPA→Mongo, sync→Kafka to podmiana adaptera, rdzeń bez zmian.
- **Ochrona logiki biznesowej** — framework nie „przecieka" do reguł; upgrade Springa nie dotyka domeny.

**Koszty:**
- więcej warstw, interfejsów, **mapowania i boilerplate**; więcej plików na jeden use case;
- narzut poznawczy — junior szybciej odnajdzie się w prostym layered;
- **kiedy się NIE opłaca:** proste **CRUD**, gdzie domena to „cienka warstwa" nad tabelą — porty tylko dublują
  Spring Data, a mapowanie DTO↔entity↔domena to czysty koszt bez zysku.

**Pragmatyzm:** większość aplikacji Spring Boot to layered — i to jest **OK**. Nie każdy serwis potrzebuje heksagonu.

**Kiedy sięgać po hexagonal:** złożona **domena** z regułami biznesowymi, długowieczność systemu (lata rozwoju),
podejście **DDD** ([[wiedza/09-architektura/ddd-lite]]), wiele adapterów wejścia/wyjścia (REST + kolejka + scheduler),
wysokie wymagania testowe. Można też **hybrydowo**: heksagon dla core-domain, layered dla prostych supporting modules.

## Przykład w praktyce (ten sam use case: „złóż zamówienie")

### Wersja warstwowa (JPA wycieka do serwisu)
```java
@Entity                                   // model biznesowy = model bazy
class Order { @Id Long id; String status; /* ... */ }

interface OrderRepository extends JpaRepository<Order, Long> {}   // z frameworka

@Service
class OrderService {
    private final OrderRepository repo;   // serwis zależy WPROST od JPA
    @Transactional
    Order place(OrderRequest req) {
        Order o = new Order();
        o.setStatus("NEW");
        return repo.save(o);              // logika spleciona z persystencją
    }
}
```

### Wersja hexagonal (port + adapter, DIP)
```java
// --- DOMENA (bez Springa, bez JPA na classpath) ---
package shop.order.domain;
public record Order(OrderId id, OrderStatus status) { /* reguły biznesowe */ }

// port OUT — interfejs W DOMENIE
package shop.order.application.port.out;
public interface SaveOrderPort { Order save(Order order); }

// port IN — API aplikacji
package shop.order.application.port.in;
public interface PlaceOrderUseCase { Order place(PlaceOrderCommand cmd); }

// use case — implementuje port IN, zależy tylko od portu OUT (abstrakcji)
package shop.order.application;
@Service
class PlaceOrderService implements PlaceOrderUseCase {
    private final SaveOrderPort saveOrder;                 // <-- abstrakcja, nie JPA
    PlaceOrderService(SaveOrderPort saveOrder) { this.saveOrder = saveOrder; }
    public Order place(PlaceOrderCommand cmd) {
        return saveOrder.save(Order.newOrder(cmd));        // czysta domena
    }
}

// --- ADAPTER OUT (infrastruktura zależy OD domeny) ---
package shop.order.adapter.out.persistence;
@Component
class OrderPersistenceAdapter implements SaveOrderPort {   // implementuje port domeny
    private final OrderJpaRepository jpa;                   // Spring Data ukryte tutaj
    public Order save(Order order) {
        return toDomain(jpa.save(toEntity(order)));         // mapowanie entity<->domena
    }
}

// --- ADAPTER IN ---
package shop.order.adapter.in.web;
@RestController
class OrderController {
    private final PlaceOrderUseCase useCase;               // widzi tylko port IN
}
```
Test domeny: `new PlaceOrderService(fakeSaveOrderPort).place(cmd)` — zero bazy, zero Springa, milisekundy.

---

## ✅ Kryteria opanowania
- [ ] Narysuję przepływ zależności w layered vs hexagonal i wskażę, gdzie następuje inwersja.
- [ ] Wyjaśnię DIP jednym zdaniem i pokażę go na parze port/adapter.
- [ ] Odróżnię port driving (IN) od driven (OUT) i podam po przykładzie adaptera każdego.
- [ ] Uzasadnię, dlaczego domena hexagonal nie ma Springa/JPA na classpath.
- [ ] Powiem, kiedy hexagonal się NIE opłaca (proste CRUD) i obronię layered jako pragmatyczny domyślny wybór.

### 🔲 Black-box check
- [ ] Jak dokładnie JPA „wycieka" do warstwy serwisowej w layered? (typy, `@Transactional`, LAZY)
- [ ] Czym różni się port od adaptera i który należy do domeny, a który do infrastruktury?
- [ ] Jak DI (Spring) technicznie realizuje inwersję zależności?
- [ ] Czym hexagonal różni się od Clean Architecture, a w czym są tożsame?
- [ ] Skąd bierze się boilerplate mapowania i czy zawsze jest konieczny?

### 🎤 Pytania rekrutacyjne
- [ ] „Layered vs hexagonal — kiedy co wybierasz?"
- [ ] „Co to Dependency Inversion Principle i jak go widać w Ports & Adapters?"
- [ ] „Jak przetestujesz logikę domenową bez uruchamiania bazy?"
- [ ] „Wady architektury warstwowej w dużym, długowiecznym systemie?"
- [ ] „by-layer vs by-feature packaging — co i dlaczego?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Kolejność warstw w architekturze warstwowej?;Controller → Service → Repository → Entity, zależności skierowane w dół (ku bazie).
Główna wada architektury warstwowej?;Domena zależy od infrastruktury — JPA/Spring wyciekają do rdzenia, utrudniając testy i wymianę technologii.
Co mówi Dependency Inversion Principle (DIP)?;Moduły wysokopoziomowe i niskopoziomowe zależą od abstrakcji, nie od siebie; szczegóły zależą od abstrakcji, nie odwrotnie.
Kto stworzył architekturę Ports & Adapters?;Alistair Cockburn (2005) — inna nazwa: hexagonal architecture.
Co to port w architekturze heksagonalnej?;Interfejs należący do domeny, opisujący punkt komunikacji z zewnątrzem.
Co to adapter w architekturze heksagonalnej?;Implementacja portu w infrastrukturze (np. REST controller, JPA repository, adapter Kafki).
Port driving/primary vs driven/secondary?;Driving (IN) = API aplikacji wołane z zewnątrz (np. use case); driven (OUT) = to, czego domena potrzebuje z zewnątrz (np. repozytorium).
Adapter dla portu IN a adapter dla portu OUT — przykłady?;IN: REST controller/CLI woła use case; OUT: JPA repository / gateway płatności implementuje port domeny.
Kluczowa reguła zależności w hexagonal?;Infrastruktura zależy od domeny — domena nie zna frameworka ani bazy.
Kto autorem Clean Architecture i jaka jej główna reguła?;Robert C. Martin (Uncle Bob); The Dependency Rule — zależności źródłowe wskazują tylko do wewnątrz.
Kręgi Clean Architecture od środka?;Entities → Use Cases → Interface Adapters → Frameworks & Drivers.
Jak hexagonal odnosi się do Clean Architecture?;Pokrewne — to samo odwrócenie zależności, inne słownictwo (porty vs koncentryczne kręgi).
Główne korzyści hexagonal/clean?;Testowalność domeny bez bazy/webu, wymienialność infrastruktury, ochrona logiki biznesowej przed frameworkiem.
Główny koszt architektury heksagonalnej?;Więcej warstw, interfejsów, mapowania i boilerplate — narzut poznawczy i więcej plików na use case.
Kiedy hexagonal się NIE opłaca?;Przy prostym CRUD, gdzie domena to cienka warstwa nad tabelą — porty tylko dublują Spring Data.
Kiedy sięgać po hexagonal?;Złożona domena, długowieczność systemu, DDD, wiele adapterów IN/OUT, wysokie wymagania testowe.
Czy większość aplikacji Spring to layered i czy to źle?;Tak, większość to layered — i to jest OK; nie każdy serwis potrzebuje heksagonu.
by-layer vs by-feature packaging?;by-layer = pakiety controller/service/repository (poziomo); by-feature = pakiet na moduł domenowy z własnymi portami i adapterami (wysoka spójność).
Jak Spring realizuje inwersję zależności?;Przez Dependency Injection — wstrzykuje konkretny adapter tam, gdzie domena widzi tylko interfejs portu.
Dlaczego trzeba mapować entity↔domena↔DTO w hexagonal?;Bo model domenowy, encja JPA i DTO REST to różne modele — rozdzielenie chroni domenę kosztem mapperów.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/09-architektura/ddd-lite]] · [[wiedza/09-architektura/solid]] · [[wiedza/07-spring/dependency-injection]]
