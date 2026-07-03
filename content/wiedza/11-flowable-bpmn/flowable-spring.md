---
temat: "Flowable — silnik BPMN w Spring Boot"
faza: 11
status: nieopanowany
priorytet: 🟢
tags: [flowable, bpmn, spring, procesy]
powiazane: ["[[wiedza/11-flowable-bpmn/bpmn-podstawy]]", "[[wiedza/05-spring/autokonfiguracja]]", "[[wiedza/11-flowable-bpmn/multitenant-procesy]]", "[[wiedza/11-flowable-bpmn/dmn-reguly]]"]
---

# Flowable — silnik BPMN w Spring Boot

> **TL;DR:** **Flowable** to lekki, open-source silnik procesów **BPMN 2.0 / DMN / CMMN** napisany w Javie (fork **Activiti**).
> W projekcie rekrutacyjnym działa **embedded** w Spring Boot — starter go autokonfiguruje, silnik **współdzieli `DataSource` i transakcje**
> aplikacji, a stan procesów trzyma we **własnych tabelach `ACT_*`**. Proces modelujesz w `.bpmn20.xml`, wdrażasz przez `RepositoryService`,
> uruchamiasz instancje przez `RuntimeService`, a **service task** wpina Twój kod Springa przez `JavaDelegate` / delegate expression.

## 1. Co — definicja i API

**Flowable** wykonuje modele procesów zapisane w standardzie **BPMN 2.0** (XML). Oprócz BPMN obsługuje **DMN** (tablice decyzyjne — reguły biznesowe,
[[wiedza/11-flowable-bpmn/dmn-reguly]]) i **CMMN** (case management). Powstał w 2016 jako fork **Activiti** przez oryginalnych autorów; jest bezpośrednim
konkurentem **Camundy 7**. Kontekst tu: **Java 21, Spring Boot 3, Flowable 7**.

Rdzeniem jest **`ProcessEngine`** — fabryka serwisów, przez które robisz wszystko:

| Serwis | Odpowiada za |
|---|---|
| **`RepositoryService`** | **deployment** definicji procesów (wdrożenie `.bpmn20.xml`), zarządzanie definicjami (wersje), suspend/activate. |
| **`RuntimeService`** | **uruchamianie instancji** (`startProcessInstanceByKey`), zmienne procesowe, sygnały (`signalEventReceived`) i wiadomości (`messageEventReceived`). |
| **`TaskService`** | **zadania użytkownika** (user task): pobieranie kolejek, `claim` (przypisanie), `complete`, `setAssignee`, komentarze. |
| **`HistoryService`** | **historia / audyt** — zakończone instancje, aktywności, zmienne, zadania (tabele `ACT_HI_*`). |
| **`ManagementService`** | operacje administracyjne: **jobs** (async executor, timery), tabele, metadane, purge. |
| **`FormService` / `DecisionService`** | formularze (Flowable Forms) i wykonywanie decyzji DMN. |

W Spring Boot wszystkie te serwisy są beanami — wstrzykujesz je jak każdy inny (`@Autowired RuntimeService`).

**Cykl życia:**
```
plik .bpmn20.xml  ──(RepositoryService.deploy)──▶  ProcessDefinition (WERSJONOWANA: key + version)
                                                          │  startProcessInstanceByKey("key")
                                                          ▼
                                                  ProcessInstance (żywy stan: tokeny, zmienne)
```
Kluczowe rozróżnienie: **definicja** procesu (szablon, immutable, wersjonowana) vs **instancja** (konkretne, żyjące wykonanie). Ten sam `.bpmn`
wdrożony ponownie = **nowa wersja** definicji; biegnące instancje kończą się na starej wersji, nowe startują na najnowszej.

## 2. Jak — serwisy, deployment i service task pod spodem

### Osadzenie (embedded) w Spring Boot
Dodajesz `flowable-spring-boot-starter`. Jego **autokonfiguracja** ([[wiedza/05-spring/autokonfiguracja]]) wykrywa `DataSource` i `PlatformTransactionManager`
w kontekście i buduje `SpringProcessEngineConfiguration` → `ProcessEngine`, rejestrując wszystkie serwisy jako beany. **Kluczowe:** Flowable **nie stawia
własnej bazy ani własnego transaction managera** — wpina się w te aplikacji. Dzięki temu zapis Twojej encji JPA i commit stanu procesu dzieją się
w **tej samej transakcji** — albo oba się utrwalą, albo oba wycofają (atomowość biznes + proces).

### Skąd tabele ACT_*
Silnik jest **stateful** — stan procesów trzyma w bazie relacyjnej w tabelach z prefiksem **`ACT_`**:
- `ACT_RE_*` — **RE**pository: definicje, deploymenty.
- `ACT_RU_*` — **RU**ntime: żywe instancje, tokeny (executions), zadania, zmienne, joby.
- `ACT_HI_*` — **HI**story: audyt zakończonych.
- `ACT_GE_*`, `ACT_ID_*` — general (bytearray), identity (users/groups).

Domyślnie Flowable **sam tworzy/migruje schemat** (`database-schema-update: true` — liquibase pod spodem). W produkcji to się **wyłącza**
(`false`) i schemat wdraża się kontrolowanie przez **Flyway** ([[wiedza/06-persystencja/flyway]]) — inaczej auto-DDL kłóci się z migracjami i psuje
powtarzalność deploymentu. Flowable dostarcza gotowe skrypty SQL per baza, które kopiujesz do `db/migration`.

### Service task — most między BPMN a kodem
W BPMN service task to krok wykonywany automatycznie przez silnik. Dwa główne sposoby wpięcia kodu:
- **`JavaDelegate`** — klasa `implements JavaDelegate` z metodą `execute(DelegateExecution)`; w XML wskazana przez `flowable:class="..."`.
  Instancjonowana przez Flowable (uwaga: nie jest to bean Springa, więc **brak wstrzykiwania** zależności).
- **Delegate expression / bean Springa** — `flowable:delegateExpression="${mojDelegate}"`, gdzie `mojDelegate` to **`@Component`** bean.
  **To preferowane w Spring Boot** — bean jest w pełni zarządzany (może mieć wstrzyknięte repozytoria, `@Transactional`, itp.). To właśnie **most**:
  proces BPMN woła Twój serwis Springa.

Wewnątrz delegata czytasz/piszesz **zmienne procesowe** (`execution.getVariable/setVariable`). Zmienne są **serializowane** do `ACT_RU_VARIABLE`
wg typu: prymitywy/`String`/`Date` mają dedykowane kolumny; obiekty POJO idą jako **`SERIALIZABLE`** (Java serialization → bytearray, kruche przy
zmianie klasy) albo lepiej jako **JSON** (Flowable ma `JsonType` dla Jackson `JsonNode`). Trzymaj zmienne małe i płaskie — to nie magazyn danych.

### Listenery
- **Execution listeners** — na zdarzeniach `start`/`end` aktywności lub `take` na sekwencji (np. audyt, logowanie).
- **Task listeners** — na `create`/`assignment`/`complete` user taska (np. wysłanie powiadomienia po przypisaniu).

### User task — praca człowieka
Zadanie czeka na akcję użytkownika. Przypisanie:
- **`assignee`** — konkretny użytkownik (zadanie „moje").
- **`candidateGroups` / `candidateUsers`** — pula; zadanie widoczne w **kolejce grupy**, dopóki ktoś go nie **`claim`**-nie (przejmie).
Obsługa przez API: `taskService.createTaskQuery()...list()` → `claim(taskId, user)` → `complete(taskId, variables)`. Formularze definiuje się
(Flowable Forms) lub własne UI renderuje pola i wysyła zmienne przy `complete`.

### Async executor (job executor)
Timery (boundary/intermediate timer events), zadania oznaczone `flowable:async="true"` i retry-e wykonuje **async executor** — pula wątków
odpytująca tabelę jobów (`ACT_RU_JOB`). To on „budzi" procesy czekające na timer i odciąża wątek żądania. W klastrze każdy węzeł ma executor;
joby są blokowane optymistycznie, żeby dwa węzły nie wykonały tego samego.

### Multitenancy
Flowable wspiera **`tenantId`** na deploymentach i instancjach — deployment z tenantem izoluje definicje procesów per najemca (ten sam `key` może
mieć różne wersje dla różnych tenantów). Szczegóły i strategie (shared schema vs osobne DataSource) w [[wiedza/11-flowable-bpmn/multitenant-procesy]].

## 3. Dlaczego / kiedy — embedded, tabele ACT, alternatywy

- **Kiedy embedded (Flowable/Camunda 7):** silnik **w tym samym JVM** co aplikacja → jedna transakcja z biznesem, brak sieciowego hopa, prosty
  deployment (jeden JAR). Idealne, gdy proces jest sercem monolitu/serwisu i mocno splata się z danymi domenowymi (jak tu — rdzeń rekrutacji).
- **Koszt embedded:** silnik **dzieli zasoby** (wątki async executora, connections do bazy) z aplikacją; tabele `ACT_*` żyją w Twojej bazie
  (obciążenie, backup, migracje). Skalowanie = skalowanie aplikacji, nie osobno silnika.
- **Serializacja zmiennych:** POJO jako `SERIALIZABLE` to pułapka — zmiana pola klasy psuje deserializację starych instancji. Preferuj JSON/proste typy.
- **Auto-DDL vs migracje:** wygodne na dev, **niebezpieczne na prod** — wyłącz i użyj Flyway.
- **Modelowanie:** **Flowable Modeler / Flowable UI** to graficzny edytor BPMN/DMN + web-app do zadań — przydatny dla **admina tenanta** do
  rysowania i wdrażania procesów. W kodzie i tak trzymasz `.bpmn20.xml` w repo (source of truth), a UI może być własne (React woła Twoje API nad `TaskService`).
- **Alternatywy — świadomość różnic:**
  - **Camunda 7** — bardzo podobne (też fork Activiti, embedded, ekosystem Spring). Główny konkurent.
  - **Camunda 8 / Zeebe** — **inna architektura**: **zewnętrzny, rozproszony silnik** (broker Zeebe), komunikacja przez gRPC, workery pobierają joby
    (job workers), brak wspólnej transakcji z aplikacją, brak relacyjnych tabel — event-sourcing. Skaluje się horyzontalnie, ale **traci prostotę
    embedded** i atomowość biznes+proces. Camunda 7 dobiega end-of-life, co pcha rynek ku Camunda 8 lub Flowable.
  - **Reguły biznesowe:** zamiast `if` w kodzie → **business rule task** z **DMN** ([[wiedza/11-flowable-bpmn/dmn-reguly]]).

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

W projekcie rekrutacyjnym proces „obsługa kandydata" żyje jako `.bpmn20.xml`; service task woła bean Springa, a rekruter domyka user taski przez API.

**1. Starter + konfiguracja (`application.yml`):**
```yaml
flowable:
  database-schema-update: false   # prod: schemat przez Flyway, nie auto-DDL
  async-executor-activate: true   # włącz job executor (timery, async)
  history-level: audit            # ile historii zapisywać (none/activity/audit/full)
```

**2. Service task jako bean Springa (delegate expression):**
```java
@Component("wyslijOferteDelegate")
class WyslijOferteDelegate implements JavaDelegate {
    private final EmailService email;
    WyslijOferteDelegate(EmailService email) { this.email = email; } // DI działa — to bean!

    @Override public void execute(DelegateExecution execution) {
        String kandydatId = (String) execution.getVariable("kandydatId");
        email.wyslijOferte(kandydatId);
        execution.setVariable("ofertaWyslana", true);
    }
}
```
W BPMN: `<serviceTask id="wyslijOferte" flowable:delegateExpression="${wyslijOferteDelegate}"/>`.

**3. Deployment i start instancji przez API:**
```java
repositoryService.createDeployment()
    .addClasspathResource("processes/rekrutacja.bpmn20.xml")
    .tenantId("tenant-acme")          // multitenancy
    .deploy();

Map<String,Object> vars = Map.of("kandydatId", "K-42");
ProcessInstance pi = runtimeService
    .startProcessInstanceByKey("procesRekrutacji", vars);
```

**4. Obsługa user taska (rekruter):**
```java
Task task = taskService.createTaskQuery()
    .processInstanceId(pi.getId())
    .taskCandidateGroup("rekruterzy")
    .singleResult();

taskService.claim(task.getId(), "recruiter-1");                 // przejęcie z kolejki grupy
taskService.complete(task.getId(), Map.of("decyzja", "ZAPROSZONY")); // domknięcie → proces rusza dalej
```

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, czym jest Flowable i co znaczy „embedded w Spring Boot" (wspólny DataSource + transakcja).
- [ ] Wymienię serwisy (`RepositoryService`/`RuntimeService`/`TaskService`/`HistoryService`/`ManagementService`) i ich rolę.
- [ ] Rozróżnię **definicję** (wersjonowaną) od **instancji** procesu.
- [ ] Wpięcie service task: `JavaDelegate` vs delegate expression/bean — i czemu bean jest lepszy w Spring Boot.
- [ ] Wiem, skąd tabele `ACT_*` i czemu na prod używa się Flyway zamiast auto-DDL.
- [ ] Odróżnię Flowable/Camunda 7 (embedded) od Camunda 8/Zeebe (zewnętrzny silnik).

### 🔲 Black-box check
- [ ] Co dokładnie robi autokonfiguracja startera z `DataSource` i transaction managerem aplikacji?
- [ ] Jak zmienna procesowa (POJO) trafia do bazy i czemu `SERIALIZABLE` jest kruche?
- [ ] Jak działa async executor — kto „budzi" proces czekający na timer?
- [ ] Co się dzieje z biegnącymi instancjami, gdy ponownie wdrożę zmieniony `.bpmn`?
- [ ] Różnica `assignee` vs `candidateGroups` i po co `claim`?

### 🎤 Pytania rekrutacyjne
- [ ] „Czym jest Flowable i jak osadza się w Spring Boot?"
- [ ] „Jak wywołać kod Springa z procesu BPMN?" (service task + delegate expression → bean)
- [ ] „Gdzie Flowable trzyma stan i jak zarządzać jego schematem?" (tabele `ACT_*`, Flyway)
- [ ] „Flowable vs Camunda 8 — kiedy embedded, a kiedy zewnętrzny silnik?"
- [ ] „Jak obsłużyć zadanie użytkownika programowo?" (query → claim → complete)

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/11-flowable-bpmn.csv`.*
```
Czym jest Flowable?;Lekki, open-source silnik procesów BPMN 2.0 / DMN / CMMN w Javie, fork Activiti; konkurent Camunda 7.
Co znaczy "embedded" w Spring Boot?;Silnik działa w tym samym JVM, współdzieli DataSource i transakcje aplikacji (stan biznesu i procesu w jednej transakcji).
Co daje flowable-spring-boot-starter?;Autokonfigurację: buduje ProcessEngine na bazie DataSource i transaction managera aplikacji i rejestruje serwisy jako beany.
Za co odpowiada RepositoryService?;Deployment definicji procesów (.bpmn20.xml), zarządzanie wersjami definicji, suspend/activate.
Za co odpowiada RuntimeService?;Uruchamianie instancji (startProcessInstanceByKey), zmienne procesowe, sygnały i wiadomości.
Za co odpowiada TaskService?;Zadania użytkownika: kolejki, claim (przejęcie), complete, setAssignee, komentarze.
Za co odpowiada HistoryService?;Historia i audyt zakończonych instancji, aktywności, zmiennych i zadań (tabele ACT_HI_*).
Za co odpowiada ManagementService?;Operacje administracyjne: joby (async executor, timery), tabele, metadane, purge.
Różnica: definicja vs instancja procesu?;Definicja to wersjonowany szablon (immutable); instancja to żywe wykonanie z tokenami i zmiennymi.
Co się dzieje przy ponownym deploy zmienionego .bpmn?;Powstaje nowa wersja definicji; biegnące instancje kończą na starej, nowe startują na najnowszej.
Dwa sposoby wpięcia kodu w service task?;JavaDelegate (flowable:class, bez DI) lub delegate expression/@Component bean (${bean}, pełne DI Springa).
Czemu delegate expression lepszy niż JavaDelegate w Spring Boot?;Bean jest zarządzany przez Spring — ma wstrzyknięte zależności i @Transactional; JavaDelegate instancjonuje Flowable bez DI.
Jak zmienne POJO trafiają do bazy i jaka pułapka?;Jako SERIALIZABLE (Java serialization → bytearray) lub JSON; SERIALIZABLE psuje się przy zmianie klasy — preferuj JSON/proste typy.
Skąd tabele ACT_* i ich prefiksy?;Flowable trzyma stan w bazie: ACT_RE_ (repo), ACT_RU_ (runtime), ACT_HI_ (historia), ACT_GE_/ACT_ID_.
Auto-DDL vs Flyway w Flowable?;database-schema-update: true tworzy schemat sam (dev); na prod wyłączyć i wdrażać skrypty ACT_* przez Flyway.
Do czego async executor (job executor)?;Wykonuje timery, zadania async (flowable:async) i retry-e; pula wątków odpytuje ACT_RU_JOB i budzi procesy.
assignee vs candidateGroups?;assignee to konkretny przypisany user; candidateGroups to pula/kolejka, z której ktoś przejmuje zadanie przez claim.
Cykl obsługi user taska przez API?;createTaskQuery -> claim(taskId, user) -> complete(taskId, variables); complete popycha proces dalej.
Multitenancy we Flowable?;tenantId na deploymentach i instancjach izoluje definicje procesów per najemca (ten sam key, różne wersje per tenant).
Flowable vs Camunda 8/Zeebe?;Flowable/Camunda 7 to embedded silnik z tabelami i wspólną transakcją; Camunda 8/Zeebe to zewnętrzny, rozproszony silnik (gRPC, job workers, event-sourcing).
Co to business rule task?;Krok BPMN wykonujący regułę DMN (tablica decyzyjna) zamiast if-ów w kodzie.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/11-flowable-bpmn/bpmn-podstawy]] · [[wiedza/11-flowable-bpmn/dmn-reguly]] · [[wiedza/11-flowable-bpmn/multitenant-procesy]]
- [[wiedza/05-spring/autokonfiguracja]] · [[wiedza/06-persystencja/flyway]] · [[projekt-rekrutacja/docs/ARCHITEKTURA]]
