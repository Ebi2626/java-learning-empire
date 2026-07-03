---
temat: "Projektowanie API REST — dobre praktyki"
faza: 7
status: nieopanowany
priorytet: 🔴
tags: [java, api, rest]
powiazane: ["[[wiedza/07-api-design/paginacja-filtrowanie]]", "[[wiedza/07-api-design/dto-mapstruct]]", "[[wiedza/05-spring/walidacja-bledy]]"]
---

# Projektowanie API REST — dobre praktyki

> **TL;DR:** **REST** to **styl architektoniczny** (Roy Fielding, 2000), nie protokół — narzuca *constraints*:
> **stateless**, client-server, **cacheable**, uniform interface, layered system. W praktyce: **zasoby** jako
> rzeczowniki w URI (`/orders/{id}/items`), **metody HTTP** z ustaloną semantyką (**safe**/**idempotent**),
> właściwe **kody statusu**, wersjonowanie i stabilny kontrakt. Prawdziwy REST = poziom 3 modelu Richardsona
> (**HATEOAS**), którego prawie nikt nie stosuje w pełni.

## 1. Co — styl architektoniczny i ograniczenia

**REST (REpresentational State Transfer)** to zbiór **ograniczeń architektonicznych** (constraints) opisanych
w rozprawie doktorskiej Fieldinga. To NIE standard ani framework — to sposób projektowania systemów
rozproszonych, najczęściej realizowany nad HTTP. Klient operuje na **reprezentacjach** zasobów (JSON, XML),
a nie na samych zasobach.

Sześć constraints (piąty — code-on-demand — jest opcjonalny):

- **Client–Server** — separacja odpowiedzialności: UI vs. przechowywanie danych. Ewoluują niezależnie.
- **Stateless** — **kluczowe dla skalowania.** Serwer NIE trzyma stanu sesji między żądaniami; **każde żądanie
  niesie cały kontekst** (token, parametry). Skutki: dowolny **load-balancer** może wysłać żądanie do
  dowolnej instancji (brak *sticky sessions*), łatwe skalowanie poziome, prostszy failover. Koszt: więcej
  danych w każdym żądaniu (np. JWT w nagłówku za każdym razem).
- **Cacheable** — odpowiedzi muszą jawnie deklarować cachowalność (`Cache-Control`, `ETag`, `Last-Modified`).
  Poprawia latencję i odciąża serwer; wymaga dyscypliny przy inwalidacji.
- **Uniform Interface** — jednolity, przewidywalny kontrakt: identyfikacja zasobów przez URI, manipulacja
  przez reprezentacje, samoopisujące się komunikaty, **HATEOAS**. To odróżnia REST od RPC nad HTTP.
- **Layered System** — klient nie wie, czy łączy się bezpośrednio z serwerem, czy przez proxy/gateway/cache.
  Umożliwia API gateway, WAF, load-balancery, CDN — bez wiedzy klienta.

### Zasoby i URI

**Zasób** (resource) to rzecz, którą API udostępnia (zamówienie, użytkownik). URI to jego **identyfikator**,
nie „ścieżka do metody".

- **Rzeczowniki, nie czasowniki.** Zła: `/getUser`, `/createOrder`, `/order/delete`. Dobra: `GET /users/42`,
  `POST /orders`, `DELETE /orders/42`. Czasownik niesie **metoda HTTP**, nie ścieżka.
- **Liczba mnoga** dla kolekcji: `/orders`, `/orders/42` (nie `/order/42`).
- **Hierarchia** dla relacji zawierania: `/orders/{id}/items`, `/orders/{id}/items/{itemId}`.
- **Bez rozszerzeń i bez czasowników w ścieżce**: nie `/orders.json`, nie `/orders/42/cancel` — a jeśli
  operacja naprawdę nie jest CRUD-em na zasobie, modeluj ją jako sub-zasób lub akcję świadomie (pragmatyczny
  wyjątek: `POST /orders/42/cancellation`).
- **kebab-case** w ścieżkach, spójnie w całym API.

```
✅ GET    /orders?status=PAID&page=2      ❌ GET  /getPaidOrders?p=2
✅ POST   /orders                         ❌ POST /orders/create
✅ GET    /orders/42/items                ❌ GET  /order/42/getItems
✅ DELETE /orders/42                      ❌ POST /orders/42/delete
```

## 2. Jak — semantyka metod, statusy, dojrzałość

### Semantyka metod HTTP — SAFE i IDEMPOTENT

Dwa pojęcia z RFC 9110, których mylenie to klasyczny błąd:

- **Safe** — metoda **nie zmienia stanu** serwera (read-only). Caching, prefetch i crawlery na tym polegają.
- **Idempotent** — **wielokrotne wykonanie tego samego żądania daje ten sam efekt na serwerze** co jednokrotne.
  Nie chodzi o identyczną *odpowiedź*, lecz o brak dodatkowych *skutków ubocznych*. Kluczowe przy **retry**
  (timeout, sieć): idempotentne żądanie można bezpiecznie powtórzyć.

| Metoda | Safe | Idempotent | Rola |
|--------|:----:|:----------:|------|
| **GET** | ✅ | ✅ | Pobiera reprezentację. Bez efektów ubocznych. |
| **POST** | ❌ | ❌ | **Tworzy** zasób / uruchamia akcję. Powtórzenie → drugi zasób. |
| **PUT** | ❌ | ✅ | **Zastępuje** cały zasób pełną reprezentacją. Powtórzenie → ten sam stan. |
| **PATCH** | ❌ | ⚠️ (zależy) | **Częściowa** zmiana. Idempotentny tylko jeśli operacja taka jest (`set x=5` tak; `increment` nie). |
| **DELETE** | ❌ | ✅ | Usuwa zasób. Powtórzenie → nadal usunięty (drugi raz zwykle 404, ale efekt ten sam). |

Uwaga: `POST` NIE jest idempotentny — stąd potrzeba **kluczy idempotentności** (sekcja 3). `PUT` używa się
do *create-or-replace pod znanym URI* (`PUT /users/42`), `POST` do *create pod URI kolekcji* (`POST /users`,
serwer nadaje id).

### Kody statusu — 2xx / 3xx / 4xx / 5xx

Kod to **część kontraktu**. Zwracaj precyzyjnie, nie „200 na wszystko".

- **2xx — sukces**
  - **200 OK** — udane GET/PUT/PATCH z ciałem odpowiedzi.
  - **201 Created** — po `POST`, który utworzył zasób; dodaj nagłówek **`Location`** z URI nowego zasobu.
  - **204 No Content** — sukces bez ciała (typowo `DELETE`, czasem `PUT`).
- **3xx — przekierowania / warunki**
  - **301/302/307/308** — przekierowania. **304 Not Modified** — cache: klient ma aktualną wersję
    (`ETag`/`If-None-Match`), oszczędza transfer.
- **4xx — błąd klienta** (nie retry-uj bez zmiany żądania)
  - **400 Bad Request** — źle sformułowane żądanie (zły JSON, brak wymaganego pola składniowo).
  - **401 Unauthorized** — brak/niepoprawne uwierzytelnienie (mylnie nazwane — chodzi o *authentication*).
  - **403 Forbidden** — uwierzytelniony, ale **bez uprawnień** (*authorization*).
  - **404 Not Found** — zasób nie istnieje (lub celowo ukrywamy istnienie zamiast 403).
  - **409 Conflict** — konflikt ze stanem: naruszenie unikalności, kolizja optymistycznego lockowania,
    zmiana niedozwolona w danym stanie (np. anulowanie już wysłanego zamówienia).
  - **422 Unprocessable Entity** — **składnia OK, ale semantyka/walidacja biznesowa nie** (np. email w złym
    formacie, kwota ujemna). Popularny wybór dla błędów walidacji zamiast 400.
- **5xx — błąd serwera** (klient może retry-ować, bo to nie jego wina)
  - **500 Internal Server Error** — nieobsłużony wyjątek. **502/503/504** — gateway/niedostępność/timeout.

```
POST /orders           → 201 Created + Location: /orders/42
GET  /orders/999       → 404 Not Found
POST /orders (zły JSON)→ 400 Bad Request
POST /orders (kwota<0) → 422 Unprocessable Entity
DELETE /orders/42      → 204 No Content
PUT  /orders/42 (cudze)→ 403 Forbidden
```

### Model dojrzałości Richardsona (RMM)

Skala „jak bardzo RESTful" jest Twoje API:

- **Poziom 0 — POX / RPC-over-HTTP.** Jeden endpoint (`POST /api`), tunelowanie wywołań (styl SOAP). To nie REST.
- **Poziom 1 — Zasoby.** Wiele URI reprezentujących zasoby, ale wciąż zwykle jedna metoda (np. `POST` na wszystko).
- **Poziom 2 — Metody HTTP + statusy.** Poprawne użycie GET/POST/PUT/DELETE i kodów statusu.
  **Tu żyje 95% realnych „REST API"** — i to w pełni akceptowalne, pragmatyczne.
- **Poziom 3 — HATEOAS.** Odpowiedzi zawierają **linki** sterujące dalszymi akcjami (hypermedia).
  Wg Fieldinga dopiero to jest „prawdziwy REST".

**HATEOAS** (Hypermedia As The Engine Of Application State): serwer w odpowiedzi zwraca dostępne przejścia
stanu jako linki, np. zamówienie w stanie `PENDING` niesie link do `cancel`, opłacone już nie. Zalety:
klient odkrywa API dynamicznie, mniej hardcodowania URI, kontrakt może ewoluować. Wady i **dlaczego rzadko
w pełni stosowany**: brak dobrego standardu/tooling po stronie klientów, wzrost rozmiaru odpowiedzi, większa
złożoność, a klienci i tak zwykle hardcodują URI. W praktyce częściowo: paginacja przez linki `next`/`prev`,
formaty HAL/JSON:API. Spring HATEOAS wspiera to w Spring Boot 3.

## 3. Dlaczego / kiedy — wersjonowanie, alternatywy, pułapki

### Wersjonowanie API — kompromisy

Kontraktu, którego używają zewnętrzni klienci, **nie wolno łamać**. Zmiany łamiące (breaking) → nowa wersja:

- **URI versioning** — `/v1/orders`, `/v2/orders`. **Najczęstsze i najprostsze**: widoczne, łatwe do
  routingu i cachowania, testowalne z przeglądarki. Wada: „nieczyste" wg purystów (wersja nie jest częścią
  identyfikatora zasobu), duplikacja ścieżek.
- **Nagłówek** — custom header, np. `X-API-Version: 2`. Czyste URI, ale niewidoczne, trudniejsze do
  cachowania i debugowania.
- **Media type / content negotiation** — `Accept: application/vnd.myapp.v2+json`. Teoretycznie najbardziej
  „RESTful", ale skomplikowane i słabo wspierane przez toolset i cache.

Praktyka: **URI versioning** dla API publicznych. Niezależnie od strategii — dbaj o **kompatybilność wsteczną**
(dodawanie pól = OK, usuwanie/zmiana typu = breaking; klienci powinni ignorować nieznane pola).

### Idempotentność i Idempotency-Key

`POST` nie jest idempotentny → przy retry (timeout, podwójne kliknięcie) grozi **duplikat** (dwa zamówienia,
podwójna płatność). Rozwiązanie: klient wysyła nagłówek **`Idempotency-Key: <UUID>`**. Serwer zapisuje wynik
pod tym kluczem; **powtórka z tym samym kluczem zwraca zapisany wynik** zamiast tworzyć nowy zasób. Wzorzec
znany ze Stripe. Wymaga przechowywania kluczy z TTL i obsługi wyścigów (unikalny indeks).

### Paginacja / filtrowanie / sortowanie

Kolekcje muszą być stronicowane, filtrowalne i sortowalne przez **query params**
(`?status=PAID&sort=createdAt,desc&page=2&size=20`), nigdy przez zwracanie wszystkiego.
Offset vs. cursor/keyset, `Link` header vs. koperta z metadanymi — szczegóły:
[[wiedza/07-api-design/paginacja-filtrowanie]].

### Format błędów — Problem Details (RFC 7807 / 9457)

Standaryzowany format błędów: `Content-Type: application/problem+json` z polami `type`, `title`, `status`,
`detail`, `instance` (+ pola rozszerzone). Spójny, maszynowo-czytelny, unika własnych ad-hoc formatów.
Spring Boot 3 wspiera to natywnie (`ProblemDetail`, `@ControllerAdvice`) — mapowanie walidacji i wyjątków
na Problem Details: [[wiedza/05-spring/walidacja-bledy]].

```json
{
  "type": "https://api.myapp.com/errors/insufficient-funds",
  "title": "Insufficient funds",
  "status": 422,
  "detail": "Saldo 30 zł jest niższe niż kwota 50 zł.",
  "instance": "/accounts/42/transactions"
}
```

### Alternatywy dla REST — kiedy nie REST

- **gRPC** — HTTP/2 + Protobuf, kontrakt-first, binarny, wydajny, streaming dwukierunkowy. **Komunikacja
  wewnętrzna między mikroserwisami**, niska latencja, silne typowanie. Słabszy dla przeglądarek/publicznego API.
- **GraphQL** — jedno query, klient **sam wybiera pola** (rozwiązuje over-/under-fetching), agregacja wielu
  źródeł. Dobre dla bogatych, zróżnicowanych klientów (mobile + web). Koszt: złożoność cachowania, ryzyko
  drogich zapytań (N+1, głębokie zagnieżdżenia), trudniejszy rate-limiting.
- **REST** wybieraj dla publicznych, prostych, cachowalnych API o zasobowym modelu i szerokim ekosystemie narzędzi.

### Dobre praktyki i pułapki

- **DTO, nie encje.** Nie eksponuj encji JPA w API — sprzęga schemat bazy z kontraktem, powoduje leaki
  (LazyInitializationException, wrażliwe pola) i cykliczne serializacje. Mapuj przez dedykowane DTO
  ([[wiedza/07-api-design/dto-mapstruct]]).
- **Spójność nazewnictwa** — jedna konwencja (mnoga, kebab-case, camelCase w JSON) w całym API.
- **Nie łam kontraktu** — deprecacja + wersjonowanie zamiast cichej zmiany. Publikuj kontrakt (OpenAPI).
- **Nie 200 na błąd** — anty-wzorzec „always 200 + pole `error` w ciele" łamie semantykę HTTP.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Kontroler Spring Boot 3 wystawiający zasób zamówień: `POST /v1/orders` przyjmuje `CreateOrderRequest` (DTO),
waliduje (`@Valid`), na sukces zwraca `201 Created` z `Location`, na błąd walidacji `422` z `ProblemDetail`,
na kolizję stanu `409`. Endpoint jest **stateless** (autoryzacja z JWT w każdym żądaniu), więc trzy repliki
za load-balancerem działają bez sticky sessions. Płatność (`POST /v1/orders/{id}/payments`) chroniona
`Idempotency-Key`, żeby retry po timeoucie nie obciążył karty dwa razy.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, że REST to styl (constraints Fieldinga), i wymienię je z pamięci.
- [ ] Uzasadnię, czemu **stateless** jest kluczowe dla skalowania i load-balancingu.
- [ ] Rozróżnię **safe** vs **idempotent** i przypiszę je metodom HTTP.
- [ ] Dobiorę poprawny kod statusu (201 vs 200, 401 vs 403, 400 vs 422, 409).
- [ ] Opiszę poziomy Richardsona i wyjaśnię, czym jest HATEOAS oraz czemu rzadko stosowany.
- [ ] Porównam strategie wersjonowania i wybiorę z uzasadnieniem.
- [ ] Wskażę, kiedy gRPC/GraphQL zamiast REST.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Czemu stateless umożliwia skalowanie poziome bez sticky sessions?
- [ ] Czym różni się safe od idempotent? Podaj metodę idempotentną, ale nie safe (PUT/DELETE).
- [ ] Czemu POST nie jest idempotentny i jak to naprawić (Idempotency-Key)?
- [ ] Kiedy 400, a kiedy 422? Kiedy 401, a kiedy 403? Kiedy 409?
- [ ] Co dokładnie znaczy poziom 3 Richardsona i czemu prawie nikt go w pełni nie wdraża?
- [ ] Czemu nie wystawiać encji JPA jako DTO?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Czy REST to protokół?" (nie — styl architektoniczny / constraints nad HTTP)
- [ ] „Co daje stateless i jaki ma koszt?"
- [ ] „PUT vs POST vs PATCH — różnice i idempotentność?"
- [ ] „Jak zabezpieczysz płatność przed podwójnym obciążeniem przy retry?"
- [ ] „REST vs gRPC vs GraphQL — kiedy co?"
- [ ] „Jak wersjonujesz API i jak nie łamiesz kontraktu?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/07-api-design.csv`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Czym jest REST?;Stylem architektonicznym (constraints Fieldinga), nie protokołem — najczęściej realizowanym nad HTTP na zasobach.
Wymień constraints REST.;Client-server, stateless, cacheable, uniform interface, layered system (+ opcjonalny code-on-demand).
Co znaczy stateless w REST?;Serwer nie trzyma stanu sesji między żądaniami; każde żądanie niesie cały kontekst.
Czemu stateless jest kluczowe dla skalowania?;Dowolna instancja obsłuży dowolne żądanie — load-balancing bez sticky sessions, łatwe skalowanie poziome i failover.
Co znaczy safe (metoda HTTP)?;Nie zmienia stanu serwera (read-only) — np. GET.
Co znaczy idempotent (metoda HTTP)?;Wielokrotne wykonanie daje ten sam efekt na serwerze co jednokrotne — bezpieczne przy retry.
Czy POST jest idempotentny?;Nie — powtórzenie tworzy kolejny zasób; stąd potrzeba Idempotency-Key.
Które metody HTTP są idempotentne?;GET, PUT, DELETE (i PATCH warunkowo); POST nie.
GET jest safe i idempotent?;Tak — oba; służy tylko do pobrania reprezentacji.
PUT vs POST przy tworzeniu?;PUT = create/replace pod znanym URI (idempotent); POST = create pod URI kolekcji, serwer nadaje id.
PATCH vs PUT?;PUT zastępuje cały zasób; PATCH zmienia częściowo (idempotentny tylko jeśli sama operacja taka jest).
Jak nazywasz URI zasobów?;Rzeczownikami w liczbie mnogiej, bez czasowników w ścieżce; np. GET /orders/42, nie /getOrder.
Kod po udanym POST tworzącym zasób?;201 Created + nagłówek Location z URI nowego zasobu.
Kod dla sukcesu bez ciała (np. DELETE)?;204 No Content.
401 vs 403?;401 = brak/niepoprawne uwierzytelnienie (authentication); 403 = uwierzytelniony, ale bez uprawnień (authorization).
400 vs 422?;400 = żądanie źle sformułowane składniowo; 422 = składnia OK, ale walidacja/semantyka biznesowa nie przechodzi.
Kiedy 409 Conflict?;Konflikt ze stanem: naruszenie unikalności, optimistic lock, operacja niedozwolona w danym stanie zasobu.
Poziomy modelu Richardsona?;0 = RPC-over-HTTP, 1 = zasoby, 2 = metody HTTP + statusy, 3 = HATEOAS.
Czym jest HATEOAS?;Odpowiedzi niosą linki sterujące dalszymi akcjami (hypermedia) — poziom 3 Richardsona, rzadko w pełni wdrażany.
Strategie wersjonowania API?;URI (/v1), nagłówek (X-API-Version), media type (Accept vnd...+json); najczęściej i najprościej URI.
Do czego Idempotency-Key?;By retry POST (timeout, podwójne kliknięcie) nie utworzył duplikatu — serwer zwraca zapisany wynik dla tego klucza.
Format błędów w API (standard)?;Problem Details RFC 7807/9457 — application/problem+json (type, title, status, detail, instance).
Kiedy gRPC zamiast REST?;Wewnętrzna komunikacja mikroserwisów: HTTP/2 + Protobuf, niska latencja, streaming, silne typowanie.
Kiedy GraphQL zamiast REST?;Gdy klient sam wybiera pola (over/under-fetching), agregacja źródeł, zróżnicowani klienci (mobile+web).
Czemu DTO, nie encje JPA w API?;Encja sprzęga schemat bazy z kontraktem, grozi leakiem pól, LazyInitException i cyklami; DTO oddziela kontrakt.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/07-api-design/paginacja-filtrowanie]] · [[wiedza/07-api-design/dto-mapstruct]] · [[wiedza/05-spring/walidacja-bledy]]
