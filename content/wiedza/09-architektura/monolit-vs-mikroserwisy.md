---
temat: "Monolit vs mikroserwisy"
faza: 9
status: nieopanowany
priorytet: 🟡
tags: [java, architektura, mikroserwisy]
powiazane: ["[[wiedza/09-architektura/ddd-lite]]", "[[wiedza/09-architektura/komunikacja-async]]", "[[wiedza/10-cloud-native/obserwowalnosc]]", "[[projekt-rekrutacja/docs/ARCHITEKTURA]]"]
---

# Monolit vs mikroserwisy

> **TL;DR:** To nie „stary vs nowoczesny", tylko **kompromis**. **Monolit** = jeden deployowalny artefakt: prostota, lokalne transakcje **ACID**, wywołania **in-process** — ale sprzężenie i skalowanie całości. **Mikroserwisy** = niezależnie wdrażane usługi wokół zdolności biznesowych: autonomia i niezależne skalowanie — kupione za cenę **złożoności rozproszonej** (sieć zawodzi, latencja, spójność **eventual**, saga, tracing, DevOps). Domyślnie zaczynaj od **monolitu modularnego** (**MonolithFirst**) i wydzielaj usługi tylko z realnego powodu.

## 1. Co — definicja i API

Trzy style na jednej osi „ile procesów, ile deploymentów":

```
MONOLIT                      MONOLIT MODULARNY              MIKROSERWISY
┌───────────────────┐        ┌───────────────────┐         ┌──────┐ ┌──────┐ ┌──────┐
│ Orders  Payments  │        │ [Orders]│[Payments]│        │Orders│ │Paym. │ │Ship. │
│ Shipping  Users   │        │ [Shipping]│[Users] │        │  DB  │ │  DB  │ │  DB  │
│  ── jedna DB ──   │        │  moduły + jedna DB │        └──┬───┘ └──┬───┘ └──┬───┘
└───────────────────┘        └───────────────────┘           └───sieć──┴───────┘
1 artefakt, 1 proces         1 artefakt, jasne granice        N artefaktów, N procesów
```

- **Monolit** — cały system to **jeden deployowalny artefakt** (np. jeden fat-jar Spring Boot), jeden proces JVM, zwykle **jedna baza danych**. Moduły komunikują się przez zwykłe wywołania metod (**in-process**).
- **Monolit modularny (modular monolith)** — nadal **jeden artefakt**, ale wewnętrznie podzielony na **moduły z jasnymi granicami** (osobne pakiety/Gradle-moduły, publiczne API modułu vs implementacja `internal`). Moduły rozmawiają przez jawne interfejsy, nie przez współdzielone tabele. To **most** między monolitem a mikroserwisami — i **rekomendowany start**.
- **Mikroserwisy** — zestaw małych, **niezależnie wdrażanych usług**, każda ownuje swój **bounded context** (patrz [[wiedza/09-architektura/ddd-lite]]) i **własną bazę**. Rozmawiają przez sieć (REST/gRPC/eventy).

Kluczowa różnica nie jest w „liczbie linii kodu", tylko w **granicy deploymentu**: w monolicie zmiana w jednym module wymusza redeploy całości; w mikroserwisach każdą usługę wdrażasz osobno.

## 2. Jak — granice, komunikacja, dane pod spodem

### Granice (bounded context)
Poprawny podział na usługi **nie jest techniczny** („warstwa serwisów", „warstwa DAO"), tylko **biznesowy** — wokół zdolności (capabilities): `Orders`, `Payments`, `Shipping`. Każda usługa = jeden **bounded context** z DDD ([[wiedza/09-architektura/ddd-lite]]). Zła granica = usługi, które muszą się wołać przy każdej operacji → dostajesz najgorszy antywzorzec.

### „Distributed monolith" — antywzorzec
**Distributed monolith** = masz koszty rozproszenia (sieć, deployment N usług, tracing) **bez** korzyści, bo usługi są **silnie sprzężone**: nie da się ich wdrożyć niezależnie (release lockstep), jedna zmiana API łamie pięć innych, wspólna baza. To zwykle skutek zbyt wczesnego lub źle poprowadzonego podziału. **Gorzej niż monolit** — masz jego sprzężenie plus podatki sieci.

### Komunikacja: sync vs async
- **Synchronicznie (REST/gRPC):** proste, request-response, ale tworzy **temporal coupling** — jak `Orders` woła `Payments` przez HTTP, to gdy `Payments` padnie, `Orders` też się dławi (kaskada awarii bez circuit breakera).
- **Asynchronicznie (eventy, message broker):** `Orders` publikuje `OrderPlaced`, `Payments`/`Shipping` reagują. Luźne sprzężenie, izolacja awarii, ale **spójność eventual** i trudniejsze debugowanie. Szczegóły: [[wiedza/09-architektura/komunikacja-async]].

### Dane: baza per serwis
Żelazna zasada: **każdy serwis ma własną bazę i NIE współdzieli jej** z innymi. Współdzielona baza = ukryte sprzężenie przez schemat (zmiana kolumny łamie 3 usługi) i utrata autonomii — to znowu distributed monolith. Konsekwencja: **nie ma już lokalnej transakcji ACID** obejmującej kilka usług.

### Transakcje rozproszone i saga
W monolicie „zamów + pobierz płatność + zarezerwuj magazyn" to **jedna transakcja ACID** (`@Transactional`, rollback za darmo). Rozbite na usługi z osobnymi bazami — nie da się tego objąć jedną transakcją DB. Zamiast tego **saga**: ciąg lokalnych transakcji + **transakcje kompensujące** przy błędzie (np. „anuluj płatność", bo magazyn odmówił). To znaczy: **spójność eventual**, nie natychmiastowa. 2PC/XA istnieje, ale w praktyce się go unika (blokady, słaba skalowalność).

### Obserwowalność
Gdy jedno żądanie użytkownika przechodzi przez 6 usług, log w jednej z nich nic nie mówi. Potrzebujesz **distributed tracing** (trace-id propagowany między usługami, np. OpenTelemetry), zagregowanych logów i metryk — patrz [[wiedza/10-cloud-native/obserwowalnosc]]. W monolicie stacktrace pokazuje całą ścieżkę „za darmo".

### Prawo Conwaya
> „Organizacje projektują systemy, które są kopią ich własnej struktury komunikacji." — M. Conway

Architektura **odbija strukturę organizacji**. Trzy zespoły → naturalnie trzy usługi. Jeśli granice usług nie pasują do granic zespołów, dostajesz ciągłe konflikty i zmiany „na styk". **Inverse Conway Maneuver** = celowo ustaw zespoły tak, by wymusić pożądaną architekturę. Wniosek praktyczny: mikroserwisy to **decyzja organizacyjna**, nie tylko techniczna.

## 3. Dlaczego / kiedy — kompromisy i kiedy co

**Monolit — zalety:** prostota deploymentu (jeden artefakt), **lokalne transakcje ACID**, łatwe debugowanie (jeden stacktrace, jeden proces), szybkie wywołania **in-process** (brak latencji sieci), jedna baza (proste JOIN-y i raporty), prosty CI/CD.

**Monolit — wady przy skali:** skalujesz **całość, nie część** (nie zeskalujesz samego `Payments`), rosnące **coupling** i erozja granic (bez dyscypliny wszystko woła wszystko), **długi build** i długie testy, **jeden stack** technologiczny dla całości, jeden redeploy = ryzyko całościowe.

**Mikroserwisy — zalety:** **niezależny deployment** (mały blast radius), **niezależne skalowanie** (skalujesz tylko gorącą usługę), **niezależny tech** (właściwe narzędzie do problemu), **izolacja awarii** (padnięcie `Shipping` nie kładzie `Orders` — o ile async), **autonomia zespołów** (własny release cadence).

**Mikroserwisy — koszty/wyzwania:** **złożoność rozproszona** — 8 fallacji sieci (sieć zawodzi, ma latencję, nie jest niezawodna), **spójność eventual zamiast ACID**, **transakcje rozproszone/saga**, **obserwowalność/tracing**, **wersjonowanie API** (backward compatibility, bo nie wdrażasz wszystkiego naraz), **testowanie integracyjne** (contract testing zamiast prostego E2E), **DevOps overhead** (k8s, CI/CD × N, service discovery, konfiguracja).

### Tabela porównawcza

| Wymiar | Monolit | Monolit modularny | Mikroserwisy |
|---|---|---|---|
| Artefakt / deployment | 1 artefakt, redeploy całości | 1 artefakt, redeploy całości | N artefaktów, niezależny deployment |
| Granice | umowne, łatwo naruszyć | **jawne** (API modułu) | twarde (proces + sieć) |
| Komunikacja | in-process (metody) | in-process przez interfejsy | sieć: REST/gRPC/eventy |
| Transakcje | **ACID lokalny** | ACID lokalny | **saga / eventual** |
| Baza | jedna | jedna (logiczne schematy) | **jedna per serwis** |
| Skalowanie | całość | całość | **per usługa** |
| Tech stack | jeden | jeden | dowolny per usługa |
| Debugowanie | proste (1 stack) | proste | **tracing** wymagany |
| DevOps overhead | niski | niski | **wysoki** |
| Ryzyko | erozja granic | dyscyplina modułów | distributed monolith |
| Dobre dla | start, mały zespół, MVP | **domyślny start**, rosnący system | duża skala, wiele zespołów |

### Kiedy co — MonolithFirst
**Domyślnie zaczynaj od (modularnego) monolitu.** Martin Fowler — **„MonolithFirst"**: prawie każdy udany system mikroserwisowy zaczynał jako monolit, który urósł i został rozbity; prawie każdy system budowany od zera jako mikroserwisy popadał w kłopoty. Powód: **na starcie nie znasz właściwych granic** — a złe granice w mikroserwisach są bardzo drogie do poprawienia (zmiana granicy = zmiana kontraktów sieciowych i baz). W monolicie modularnym przesunięcie granicy to refaktor pakietów.

**Wydzielaj mikroserwis, gdy masz REALNY powód:**
- **skala** — konkretny moduł ma inny profil obciążenia i chcesz go skalować osobno,
- **zespoły** — prawo Conwaya, autonomiczne zespoły potrzebują niezależnego release,
- **różne wymagania** — inny SLA, inny tech (np. moduł ML w innym języku), inna izolacja awarii.

Jeśli powodem jest tylko „bo mikroserwisy są modne" — **nie rób tego**.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

**Projekt rekrutacyjny — uzasadnienie wyboru monolitu modularnego** (Java 21, Spring Boot 3): dla portfolio/rekrutacji wybieramy **monolit modularny**, bo (1) pokazuje dojrzałość — świadomy dobór, nie kult cargo, (2) zachowuje **jasne granice modułów** (`orders`, `payments`, `users` jako osobne pakiety/moduły Gradle z publicznym API i `internal`), więc gdyby zaszła potrzeba, wydzielenie usługi jest tanie, (3) daje **prosty deployment i lokalne transakcje ACID** (jeden `docker run`, brak sagi i brokera do utrzymania), (4) recruiter widzi Conwaya i MonolithFirst w praktyce. Pełne uzasadnienie i diagram granic: [[projekt-rekrutacja/docs/ARCHITEKTURA]]. W Spring Boot moduły izolujesz przez strukturę pakietów + Spring Modulith (weryfikacja granic w testach) i zdarzenia domenowe (`ApplicationEventPublisher`) jako in-process odpowiednik eventów — łatwe do zamiany na broker przy wydzieleniu.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię różnicę monolit / monolit modularny / mikroserwisy przez **granicę deploymentu**, nie liczbę linii.
- [ ] Wymienię zalety monolitu (ACID, in-process, prostota) i jego wady przy skali (coupling, skalowanie całości).
- [ ] Wiem, czym jest **distributed monolith** i czemu jest gorszy niż monolit.
- [ ] Uzasadnię **MonolithFirst** i podam realne kryteria wydzielania usługi.
- [ ] Wytłumaczę „baza per serwis" i dlaczego współdzielona baza łamie autonomię.
- [ ] Wyjaśnię przejście z ACID na sagę / spójność eventual.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co dokładnie znika, gdy rozbijasz monolit na usługi? (lokalne ACID, in-process call, jeden stacktrace)
- [ ] Czym jest bounded context i czemu granica ma być biznesowa, nie techniczna?
- [ ] Jak działa saga i transakcja kompensująca? Czemu nie 2PC?
- [ ] Dlaczego przy N usługach potrzebujesz distributed tracing?
- [ ] Na czym polega temporal coupling przy synchronicznym REST i jak go łagodzić?
- [ ] Jak prawo Conwaya wpływa na dobór architektury?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Monolit czy mikroserwisy — od czego byś zaczął w nowym projekcie i dlaczego?"
- [ ] „Jaki jest największy koszt mikroserwisów?" (złożoność rozproszona, nie kod)
- [ ] „Co to distributed monolith i jak go rozpoznać?"
- [ ] „Jak zapewnisz spójność między dwoma usługami bez transakcji rozproszonej?"
- [ ] „Dlaczego nie współdzielić bazy między serwisami?"
- [ ] „Czemu w Twoim projekcie wybrałeś monolit modularny?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/09-architektura.csv`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Czym jest monolit (deployment)?;Cały system jako jeden deployowalny artefakt, jeden proces, zwykle jedna baza; moduły wołają się in-process.
Główna zaleta monolitu przy transakcjach?;Lokalna transakcja ACID obejmująca wiele operacji (jeden @Transactional, rollback za darmo).
Główne wady monolitu przy skali?;Skalowanie całości nie części, rosnący coupling, długi build, jeden stack technologiczny.
Czym jest monolit modularny?;Jeden artefakt z modułami o jasnych granicach (publiczne API + internal); most między monolitem a mikroserwisami.
Czym są mikroserwisy?;Zestaw niezależnie wdrażanych usług, każda ownuje bounded context i własną bazę, komunikacja przez sieć.
Wokół czego dzieli się usługi?;Wokół zdolności biznesowych (bounded context z DDD), nie wokół warstw technicznych.
Główny koszt mikroserwisów?;Złożoność rozproszona: sieć zawodzi, latencja, spójność eventual, saga, tracing, DevOps overhead.
Co to distributed monolith?;Antywzorzec: usługi rozproszone ale silnie sprzężone (release lockstep, wspólna baza) — koszty rozproszenia bez korzyści.
Dlaczego distributed monolith jest gorszy od monolitu?;Ma sprzężenie monolitu plus podatki sieci (latencja, deployment N usług, tracing).
ACID vs mikroserwisy?;Nie ma jednej transakcji ACID przez wiele baz; zamiast tego saga i spójność eventual.
Czym jest saga?;Ciąg lokalnych transakcji z transakcjami kompensującymi przy błędzie zamiast jednej transakcji rozproszonej.
Zasada bazy w mikroserwisach?;Baza per serwis, nie współdziel bazy — współdzielenie tworzy ukryte sprzężenie przez schemat.
Sync REST vs async eventy?;REST prosty ale temporal coupling i kaskady awarii; eventy luźne sprzężenie i izolacja, ale eventual consistency.
Po co distributed tracing?;Jedno żądanie idzie przez wiele usług; trace-id (np. OpenTelemetry) łączy logi w jedną ścieżkę.
Prawo Conwaya?;Architektura systemu odbija strukturę komunikacji organizacji; granice usług dążą do granic zespołów.
Co to MonolithFirst (Fowler)?;Zaczynaj od monolitu; wydzielaj mikroserwisy dopiero gdy poznasz granice i masz realny powód.
Kiedy wydzielić mikroserwis?;Gdy jest realny powód: skala danego modułu, autonomia zespołu lub różne wymagania (SLA/tech/izolacja).
Czemu złe granice są droższe w mikroserwisach?;Zmiana granicy = zmiana kontraktów sieciowych i baz; w monolicie to tylko refaktor pakietów.
Co znika przy rozbiciu monolitu?;Lokalne ACID, szybkie wywołania in-process i jeden stacktrace do debugowania.
Dlaczego monolit modularny w projekcie rekrutacyjnym?;Prostota (ACID, jeden deploy) + jasne granice, więc łatwe późniejsze wydzielenie usług — dojrzały świadomy wybór.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/09-architektura/ddd-lite]] · [[wiedza/09-architektura/komunikacja-async]] · [[wiedza/10-cloud-native/obserwowalnosc]] · [[projekt-rekrutacja/docs/ARCHITEKTURA]]
