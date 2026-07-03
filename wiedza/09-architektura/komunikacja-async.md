---
temat: "Komunikacja asynchroniczna i architektura zdarzeniowa"
faza: 9
status: nieopanowany
priorytet: 🟡
tags: [java, architektura, messaging, kafka]
powiazane: ["[[wiedza/06-persystencja/transakcje]]", "[[wiedza/09-architektura/mikroserwisy]]", "[[wiedza/09-architektura/rest-api]]"]
---

# Komunikacja asynchroniczna i architektura zdarzeniowa

> **TL;DR:** W komunikacji **synchronicznej** (REST — żądanie/odpowiedź) nadawca **blokuje się** i czeka, tworząc
> **silne sprzężenie czasowe** (odbiorca musi żyć *teraz*). W **asynchronicznej** nadawca wrzuca **komunikat/zdarzenie**
> do **brokera** (RabbitMQ = kolejka z routingiem, Kafka = trwały, partycjonowany log zdarzeń) i idzie dalej —
> luźne sprzężenie, bufor obciążenia, odporność na chwilową niedostępność. Cena: **eventual consistency**,
> duplikaty (**at-least-once** → **idempotentni konsumenci**), problem **dual-write** (→ **outbox pattern**)
> i transakcje rozproszone (→ **saga**).

## 1. Co — synchronicznie vs asynchronicznie

**Synchronicznie (REST, gRPC blocking):** klient wysyła żądanie i **czeka na odpowiedź w tym samym wątku wywołania**.
```
Client ──HTTP request──▶ Service ──(przetwarza)──▶ ──HTTP 200 + body──▶ Client (odblokowany)
```
- **Sprzężenie czasowe (temporal coupling):** obie strony muszą być *jednocześnie* dostępne. Jak `Service` padł
  albo jest wolny — klient dostaje błąd / timeout. Awarie **propagują się** (kaskada, patrz circuit breaker).
- Proste, natychmiastowa odpowiedź, łatwe do debugowania (jeden trace synchroniczny).

**Asynchronicznie (messaging / events):** nadawca (*producer*) publikuje **komunikat** do **brokera** i **od razu
wraca** (fire-and-forget). Odbiorca (*consumer*) przetwarza **później**, we własnym tempie.
```
Producer ──publish──▶ [ BROKER: queue/topic ] ──deliver──▶ Consumer (własne tempo)
   (wraca od razu)         bufor / trwałość
```
- **Luźne sprzężenie:** nadawca **nie zna** odbiorców, nie czeka na nich, nie wie ilu ich jest.
- **Bufor obciążenia (load leveling):** szpila ruchu ląduje w kolejce, konsument przerabia w stałym tempie
  (nie przewraca się pod zalewem — *backpressure* naturalny).

### Message vs Event vs Command — kluczowe rozróżnienie
- **Message** — najogólniej: dowolna porcja danych przekazana przez broker (parasol na command i event).
- **Command** — „**zrób to**": intencja skierowana do **konkretnego** odbiorcy, oczekuje wykonania
  (`ChargePayment`, `SendEmail`). Sprzężenie z odbiorcą wyższe.
- **Event** — „**to się stało**": fakt z przeszłości, **czas przeszły** (`OrderPlaced`, `PaymentCaptured`).
  Nadawca **nie wie** kto zareaguje ani czy ktokolwiek. Najluźniejsze sprzężenie → serce EDA.

### Topic / Queue, partycje, consumer groups
- **Queue (RabbitMQ)** — komunikat konsumowany przez **jednego** konsumenta i **znika** po ack (work queue —
  rozdzielanie zadań między workery).
- **Topic / log (Kafka)** — zdarzenia **trwale zapisane**; **wielu** niezależnych konsumentów czyta te same dane,
  każdy własnym tempem (event streaming).
- **Partycje (Kafka)** — topic dzieli się na partycje = jednostki równoległości i **kolejności**. Klucz komunikatu
  → hash → partycja (te same klucze trafiają w tę samą partycję).
- **Consumer group (Kafka)** — grupa instancji tej samej aplikacji; Kafka przypisuje **każdą partycję dokładnie
  jednemu** konsumentowi w grupie → skalowanie horyzontalne + gwarancja kolejności *per partycja*.

## 2. Jak — brokery, dostarczanie, outbox pod spodem

### RabbitMQ — smart broker, model exchange → binding → queue
Producent publikuje do **exchange** (nie bezpośrednio do kolejki). Exchange wg **routing key** i typu
(`direct` / `topic` / `fanout` / `headers`) rozdziela komunikaty do powiązanych (**binding**) **kolejek**.
```
Producer ─▶ Exchange ──(routing key)──▶ Queue A ─▶ Consumer 1
                    └──────────────────▶ Queue B ─▶ Consumer 2
```
- **Model push**, konsument **acknowledguje** (`ack`) po przetworzeniu; brak ack (lub `nack`/timeout) → **redelivery**.
- Komunikat **usuwany** po ack. Świetne do **zadań w tle, RPC (reply-to), routingu**. Logika w brokerze.

### Kafka — dumb broker, smart consumer, trwały log
Broker to **append-only log** — zdarzenia dopisywane na koniec partycji, **nie kasowane** po odczycie
(retencja czasowa/rozmiarowa). Konsument trzyma **offset** (dokąd doczytał).
```
Partition 0: [e0][e1][e2][e3][e4]... ◀── offset konsumenta wskazuje pozycję
```
- **Wysoka przepustowość** (sekwencyjne I/O, batch, zero-copy), **replay** (przewiń offset i przetwarzaj od nowa —
  np. po naprawie buga albo dla nowego konsumenta).
- **Model pull**, konsument **commituje offset**. Logika po stronie konsumenta.

| | RabbitMQ | Kafka |
|---|---|---|
| Model | kolejka (exchange→queue) | trwały log zdarzeń, partycje |
| Po konsumpcji | komunikat znika (ack) | zostaje (retencja), offset |
| Wielu konsumentów tych samych danych | nie (fanout kopiuje) | tak (natywnie, każdy swój offset) |
| Replay | nie | tak |
| Kolejność | per queue | per partition |
| Najlepsze do | zadania, RPC, routing | event streaming, event sourcing, duży throughput |

### Gwarancje dostarczania
- **At-most-once** — 0 lub 1 raz; komunikat może **zginąć**, brak retry. Najszybsze, dla nieważnych danych (metryki).
- **At-least-once** — **≥1 raz**; ack po przetworzeniu → padnięcie przed ackiem = **redelivery = duplikaty**.
  **Domyślny, praktyczny wybór.** Wymusza **idempotentnych konsumentów**.
- **Exactly-once** — dokładnie raz. **Trudne i kosztowne** end-to-end (potrzeba transakcji/dedup na każdym styku).
  Kafka ma EOS (idempotent producer + transactions) *w obrębie Kafki*, ale side-effects (mail, płatność) i tak
  wymagają idempotencji. **W praktyce: at-least-once + idempotencja**, a nie prawdziwe exactly-once.

**Idempotentny konsument** — przetworzenie tego samego komunikatu **N razy daje ten sam efekt co raz**. Jak:
tabela `processed_messages(message_id)` z UNIQUE (sprawdź→pomiń jeśli był), upsert zamiast insert, naturalny klucz
biznesowy. Konieczne, bo at-least-once **gwarantuje** duplikaty.

**Kolejność:** globalnej kolejności zwykle nie ma. Kafka gwarantuje kolejność **tylko per partycja** — dlatego
zdarzenia jednej encji (np. `orderId`) publikuj z **tym samym kluczem**, by trafiały w jedną partycję.

### DLQ i retry
Komunikat, którego nie da się przetworzyć (poison message), po N nieudanych próbach ląduje w **Dead Letter Queue**
zamiast blokować kolejkę w nieskończonej pętli redelivery. Strategie: **retry z backoffem** (kolejno rosnące opóźnienia,
osobna retry-queue), potem **DLQ** do inspekcji/ręcznego reprocessingu. Bez DLQ jeden trujący komunikat potrafi
zatrzymać cały strumień.

### Outbox pattern — spójność zapisu do bazy z publikacją zdarzenia
**Problem dual-write:** w jednym handlerze chcesz (1) zapisać dane do bazy i (2) opublikować event do brokera.
To **dwa różne systemy bez wspólnej transakcji** — jak padniesz między nimi, dostajesz niespójność
(zapis jest, eventu nie ma — albo odwrotnie). Rollback brokera nie cofnie commitu bazy.

**Rozwiązanie:** event zapisujesz do tabeli **`outbox` w TEJ SAMEJ transakcji lokalnej** co dane biznesowe
(atomowo — albo jedno i drugie, albo nic, patrz [[wiedza/06-persystencja/transakcje]]). Osobny proces **relay**
(poller albo **CDC** np. Debezium czytający WAL/binlog) czyta wiersze outbox i **publikuje** do brokera,
po potwierdzeniu oznacza jako wysłane.
```
@Transactional {
   orderRepo.save(order);            // dane
   outboxRepo.save(orderPlacedEvent) // event — TA SAMA transakcja → atomowo
}
────────────── (osobno, asynchronicznie) ──────────────
Relay: SELECT z outbox → publish do Kafki → mark sent   // at-least-once → konsument idempotentny
```
Relay daje **at-least-once** (może opublikować, paść przed markiem, powtórzyć) — więc konsument i tak musi być
idempotentny. Outbox zamienia trudny „exactly-once dual-write" na łatwe „lokalna transakcja + at-least-once".

## 3. Dlaczego / kiedy — kompromisy, eventual consistency

**Po co async:**
- **Decoupling** — nadawca i odbiorca ewoluują niezależnie; dodajesz nowego konsumenta **nie ruszając** producenta.
- **Skalowanie** — konsumentów dokładasz niezależnie; partycje/kolejki rozdzielają pracę.
- **Odporność** — odbiorca chwilowo niedostępny? Komunikaty **czekają** w brokerze, nie giną; brak kaskady awarii.
- **Load leveling** — bufor absorbuje szpile ruchu.
- **Przetwarzanie w tle** — długie zadania (raporty, maile, obróbka obrazów) poza ścieżką żądania użytkownika.
- **Integracja** — jeden event, wielu odbiorców (analityka, mail, audyt) bez wiedzy producenta.

**Wzorce zdarzeniowe:**
- **Event-Driven Architecture (EDA)** — komponenty reagują na zdarzenia zamiast wołać się bezpośrednio.
- **Event notification** — event niesie tylko fakt + id (`OrderPlaced{orderId}`); konsument **dowoła** REST-em
  po szczegóły. Mały event, ale przywraca sprzężenie i ruch zwrotny.
- **Event-carried state transfer** — event niesie **cały potrzebny stan** (`OrderPlaced{orderId, items, total,...}`);
  konsument nie musi dowoływać, ma lokalną kopię. Większe eventy, ale pełne decoupling i autonomia.
- **Saga** — transakcja rozproszona jako **sekwencja lokalnych transakcji** + **kompensacje** (cofnięcie
  logiczne, np. `RefundPayment` gdy magazyn odmówi) zamiast globalnego 2PC.
  - **Choreografia** — serwisy reagują na eventy nawzajem, brak koordynatora (luźno, ale trudne do prześledzenia).
  - **Orkiestracja** — centralny **orchestrator** steruje krokami i kompensacjami (jasny przepływ, punkt centralny).
- **CQRS** (zajawka) — rozdzielenie modelu **zapisu** (commands) od **odczytu** (queries/read model), często
  read model budowany z eventów (eventual consistency między nimi). Osobny temat.

**Eventual consistency i UX:** async oznacza, że stan globalny **zbiega się z opóźnieniem**, nie natychmiast.
Po `POST /order` zwracasz `202 Accepted` — zamówienie „w trakcie", nie „gotowe". UX musi to obsłużyć:
statusy pending/processing, brak natychmiastowej gwarancji, komunikaty „przetwarzamy". Nie ma
„read-your-writes" za darmo — użytkownik może przez chwilę nie widzieć własnej zmiany.

**Kiedy async NIE jest potrzebne:** monolit lub prosty CRUD, gdzie potrzebujesz **natychmiastowej odpowiedzi
i silnej spójności**; mały zespół; brak realnych szpil obciążenia. **Prostota > rozproszenie.** Async dokłada:
brokera do utrzymania, monitoringu, śledzenia (distributed tracing), obsługi duplikatów, DLQ, eventual consistency.
Nie rozpraszaj przedwcześnie — dużo problemów rozwiązuje synchroniczne wywołanie w jednym procesie.

**W Springu:**
- **Spring Events** — zdarzenia **in-process** w obrębie jednego JVM: `ApplicationEventPublisher.publishEvent(...)`
  + `@EventListener`. Domyślnie **synchroniczne** (w wątku publikującego); `@Async` robi je asynchronicznymi.
  **`@TransactionalEventListener(phase = AFTER_COMMIT)`** — listener odpala się **dopiero po commit** transakcji
  (nie publikuj do brokera przed commitem — inaczej odbiorca zobaczy stan, którego rollback cofnął).
- **Spring for Apache Kafka** — `KafkaTemplate` (producer), `@KafkaListener` (consumer), konfiguracja consumer group,
  DLT (`@RetryableTopic` / `DeadLetterPublishingRecoverer`).
- **Spring AMQP (RabbitMQ)** — `RabbitTemplate` + `@RabbitListener`, deklaracja exchange/queue/binding.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Serwis zamówień: po złożeniu zamówienia trzeba powiadomić serwis płatności i wysyłki — **bez dual-write**.

```java
// 1) Producer: zapis danych + eventu do outbox w JEDNEJ transakcji lokalnej
@Service
class OrderService {
    @Transactional
    public void placeOrder(OrderRequest req) {
        Order order = orderRepo.save(Order.from(req));
        OutboxEvent evt = OutboxEvent.of(
            "OrderPlaced",
            order.getId().toString(),                 // klucz = kolejność per zamówienie
            json.write(new OrderPlaced(order.getId(), order.total())));
        outboxRepo.save(evt);   // TA SAMA transakcja co order → atomowo, brak dual-write
    }
}

// 2) Relay: poller publikuje do Kafki (at-least-once), klucz = aggregateId → partycja
@Scheduled(fixedDelay = 500)
@Transactional
void relay() {
    for (OutboxEvent e : outboxRepo.findUnsentOrderByIdAsc(100)) {
        kafkaTemplate.send("orders", e.getAggregateId(), e.getPayload());
        e.markSent();  // mógł paść przed tym → duplikat → konsument idempotentny
    }
}

// 3) Idempotentny konsument: dedup po eventId, pomija duplikaty
@KafkaListener(topics = "orders", groupId = "shipping")
@Transactional
public void onOrderPlaced(ConsumerRecord<String, String> rec) {
    OrderPlaced evt = json.read(rec.value(), OrderPlaced.class);
    if (!processed.markIfNew(evt.eventId())) {   // UNIQUE(eventId): już był → skip
        return;                                   // at-least-once = duplikaty są normą
    }
    shipmentService.schedule(evt.orderId());
}
```
Efekt: zamówienie i event są **atomowe** (outbox), publikacja jest **at-least-once** (relay), a duplikaty są
**niegroźne** (idempotencja). Zamówienie i wysyłka są spójne **ewentualnie**, nie natychmiast — UI pokazuje status.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię sync (REST, sprzężenie czasowe, blokada) vs async (broker, luźne sprzężenie, bufor) jednym zdaniem.
- [ ] Odróżnię **message / event / command** i podam po co jest event notification vs event-carried state transfer.
- [ ] Wiem, czym różni się **RabbitMQ (kolejka, ack, znika)** od **Kafki (log, offset, replay, partycje)**.
- [ ] Rozumiem **at-most/at-least/exactly-once** i dlaczego at-least-once wymusza **idempotentnych konsumentów**.
- [ ] Wyjaśnię problem **dual-write** i jak rozwiązuje go **outbox pattern**.
- [ ] Rozumiem **sagę** (choreografia vs orkiestracja, kompensacja) i **eventual consistency** w UX.
- [ ] Wiem, **kiedy async NIE** i że prostota bywa ważniejsza niż rozproszenie.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co dokładnie znaczy „sprzężenie czasowe" i czemu async je zdejmuje?
- [ ] Jak Kafka gwarantuje kolejność i skalowanie jednocześnie? (partycje + consumer group + klucz)
- [ ] Czemu w RabbitMQ komunikat znika, a w Kafce zostaje — i co to daje (replay)?
- [ ] Dlaczego exactly-once end-to-end jest praktycznie nieosiągalny i co robimy zamiast?
- [ ] Jak działa relay w outbox i dlaczego to nadal at-least-once?
- [ ] Czemu `@TransactionalEventListener(AFTER_COMMIT)` a nie zwykły `@EventListener` przy publikacji do brokera?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Kafka vs RabbitMQ — kiedy co wybierzesz?"
- [ ] „Jak zapewnisz spójność między zapisem do bazy a publikacją zdarzenia?" (outbox, dual-write)
- [ ] „At-least-once daje duplikaty — jak sobie z tym radzisz?" (idempotencja, dedup po id)
- [ ] „Czym jest saga i czym różni się choreografia od orkiestracji?"
- [ ] „Co to eventual consistency i jak wpływa na UX?"
- [ ] „Kiedy NIE użyłbyś messagingu?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/09-architektura.csv`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Sprzężenie czasowe (temporal coupling) w REST?;Nadawca i odbiorca muszą być jednocześnie dostępni — odbiorca musi żyć TERAZ, inaczej timeout/błąd.
Główna zaleta komunikacji asynchronicznej?;Luźne sprzężenie — nadawca nie zna/nie czeka na odbiorców, broker buforuje i daje odporność na chwilową niedostępność.
Message vs Event vs Command?;Message = ogólna porcja danych; Command = "zrób to" do konkretnego odbiorcy; Event = "to się stało" (fakt, przeszły), nadawca nie wie kto zareaguje.
Model RabbitMQ?;Producer → Exchange → (routing key/binding) → Queue → Consumer; komunikat znika po ack (kolejka).
Model Kafki?;Trwały append-only log podzielony na partycje; konsument trzyma offset; zdarzenia zostają (retencja), możliwy replay.
Czym jest partycja w Kafce?;Jednostka równoległości i kolejności w topicu; klucz komunikatu → hash → partycja (te same klucze w tej samej partycji).
Consumer group w Kafce?;Grupa instancji tej samej aplikacji; każda partycja przypisana dokładnie jednemu konsumentowi w grupie → skalowanie + kolejność per partycja.
RabbitMQ vs Kafka — retencja?;RabbitMQ usuwa komunikat po ack; Kafka trzyma zdarzenia (retencja czasowa/rozmiarowa), umożliwia replay i wielu niezależnych konsumentów.
At-most-once?;Dostarczenie 0 lub 1 raz; komunikat może zginąć, brak retry — dla nieważnych danych (np. metryki).
At-least-once?;Dostarczenie co najmniej raz; ack po przetworzeniu, padnięcie przed ackiem → redelivery = duplikaty. Domyślny wybór, wymusza idempotencję.
Dlaczego exactly-once jest trudne?;End-to-end wymaga transakcji/dedup na każdym styku i side-effects (mail, płatność) nie da się cofnąć — w praktyce robi się at-least-once + idempotencja.
Idempotentny konsument?;Przetworzenie tego samego komunikatu N razy daje ten sam efekt co raz — np. dedup po message_id (UNIQUE), upsert, naturalny klucz.
Co to DLQ (dead letter queue)?;Kolejka na komunikaty nieprzetwarzalne po N próbach (poison message) — zamiast nieskończonej pętli redelivery blokującej strumień.
Problem dual-write?;Zapis do bazy i publikacja eventu to dwa systemy bez wspólnej transakcji — padnięcie między nimi daje niespójność (są dane, brak eventu lub odwrotnie).
Outbox pattern?;Event zapisujesz do tabeli outbox w tej samej lokalnej transakcji co dane; osobny relay (poller/CDC) publikuje do brokera. Usuwa dual-write, daje at-least-once.
Event notification vs event-carried state transfer?;Notification niesie tylko fakt+id (konsument dowołuje po dane); state transfer niesie cały stan (konsument nie dowołuje, pełny decoupling, większy event).
Co to saga?;Transakcja rozproszona jako sekwencja lokalnych transakcji + kompensacje (logiczne cofnięcia) zamiast globalnego 2PC.
Choreografia vs orkiestracja sagi?;Choreografia — serwisy reagują na eventy nawzajem, bez koordynatora; orkiestracja — centralny orchestrator steruje krokami i kompensacjami.
Eventual consistency — skutek dla UX?;Stan zbiega się z opóźnieniem, nie natychmiast — zwracasz 202/pending, brak read-your-writes, UI pokazuje statusy "przetwarzamy".
@TransactionalEventListener(AFTER_COMMIT) — po co?;Listener odpala się dopiero po commit transakcji, by nie publikować eventu o stanie, który rollback może cofnąć.
Kiedy NIE używać messagingu?;Gdy potrzebna natychmiastowa odpowiedź i silna spójność, prosty CRUD/monolit, brak szpil ruchu — prostota ważniejsza niż rozproszenie.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/06-persystencja/transakcje]] · [[wiedza/09-architektura/mikroserwisy]] · [[wiedza/09-architektura/rest-api]]
