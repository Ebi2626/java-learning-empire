---
temat: "Multitenant i wersjonowane procesy Flowable"
faza: 11
status: nieopanowany
priorytet: 🟢
tags: [flowable, bpmn, multitenancy, procesy]
powiazane: ["[[wiedza/09-architektura/multitenancy]]", "[[wiedza/08-bezpieczenstwo/spring-security]]", "[[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]]", "[[projekt-rekrutacja/docs/ARCHITEKTURA]]"]
---

# Multitenant i wersjonowane procesy Flowable

> **TL;DR:** Każde stowarzyszenie (tenant) ma **własną wersję procesu rekrutacji**. We Flowable robisz to przez `tenantId` na **deploymencie** i **definicji procesu** — procesy są izolowane per tenant, a start przez `startProcessInstanceByKeyAndTenantId`. Flowable **automatycznie wersjonuje** definicję przy każdym deploymencie (v1, v2, v3…); **instancje już uruchomione zostają na swojej wersji** (in-flight nie migrują same), nowe startują na najnowszej. Admin tenantu edytuje **tylko swój** proces (autoryzacja z JWT → `tenantId`), a service taski trzymamy w **sandboxie** predefiniowanych klocków — nigdy dowolny Java/skrypt od tenanta.

## 1. Co — wymóg projektu i API

**Wymóg projektu rekrutacyjnego:** system jest wielo-tenantowy (SaaS dla stowarzyszeń). Każdy tenant musi mieć **własną, edytowalną wersję procesu rekrutacji** (BPMN), którą jego admin dopracowuje **bez zmian w kodzie** i bez redeployu aplikacji. To odróżnia „silnik procesów" (Flowable) od procesu zaszytego na sztywno w `if`-ach — logika biznesowa mieszka w definicjach BPMN w bazie, nie w Javie.

Kluczowe pojęcia Flowable:

- **Process definition** — sparsowany BPMN XML (klucz `key`, np. `rekrutacja`, `version`, `deploymentId`, opcjonalnie `tenantId`). To *szablon*.
- **Process instance** — konkretne uruchomienie definicji (jedna aplikacja kandydata = jedna instancja).
- **Deployment** — jednostka wdrożenia: paczka jednego lub wielu plików BPMN wrzucona do silnika. **Wersjonowanie dzieje się na poziomie deploymentu.**
- **`tenantId`** — dyskryminator izolacji. Nadawany na deploymencie i propagowany na definicje/instancje/taski.

Serwisy Flowable (Spring Boot 3, Flowable 7):

- `RepositoryService` — deploy definicji, odpytywanie o wersje (`ProcessDefinitionQuery`).
- `RuntimeService` — start i sterowanie instancjami (`startProcessInstanceByKeyAndTenantId`).
- `TaskService` — zadania użytkownika (user tasks).
- `ProcessMigrationService` — świadoma migracja in-flight instancji między wersjami.

## 2. Jak — tenantId, wersjonowanie, edycja pod spodem

### Multitenancy: tenantId na deploymencie i definicji

Flowable oferuje kilka strategii multitenancy; **najprostsza i domyślna to „shared database, discriminator column"**: wszystkie tenanty w tych samych tabelach `ACT_*`, a rozróżnia je kolumna `TENANT_ID_`.

```java
// Deploy definicji dla konkretnego tenanta
repositoryService.createDeployment()
    .addClasspathResource("processes/rekrutacja.bpmn20.xml")
    .name("rekrutacja-" + tenantId)
    .tenantId(tenantId)              // ⟵ wszystko z tej paczki dostaje TENANT_ID_
    .enableDuplicateFiltering()      // ⟵ nie deployuj ponownie identycznego XML (patrz pułapki)
    .deploy();
```

Po deployu w `ACT_RE_PROCDEF` powstaje wiersz z `KEY_='rekrutacja'`, `TENANT_ID_='assoc-42'`, `VERSION_=1`. Ten sam klucz `rekrutacja` **może istnieć równolegle** dla wielu tenantów — bo para (`key`, `tenantId`) jest osią wersjonowania. Dwa tenanty mają dwie niezależne linie wersji tej samej „nazwy" procesu.

Start instancji **zawsze** z tenantId, żeby trafić we właściwą linię:

```java
runtimeService.startProcessInstanceByKeyAndTenantId(
    "rekrutacja",                    // key
    businessKey,                     // np. id aplikacji kandydata
    variables,                       // dane startowe
    tenantId                         // ⟵ wybiera najnowszą wersję definicji tego tenanta
);
```

Sam `startProcessInstanceByKey("rekrutacja")` (bez tenantId) szukałby definicji **bez** tenanta (`TENANT_ID_` NULL) — i albo wystartuje globalną, albo rzuci wyjątkiem. W środowisku multitenant to klasyczny błąd: „nie znaleziono definicji" mimo że jest — bo pytasz o pustego tenanta.

Ponadto silnik potrafi „widzieć tenanty przezroczyście" — jeśli ustawisz `TenantInfoHolder` / obsługę tenanta w konfiguracji, część zapytań filtruje po bieżącym tenancie automatycznie. W praktyce jednak jawne `...AndTenantId` jest czytelniejsze i mniej podatne na wycieki między tenantami.

### Relacja do strategii multitenancy danych

Izolacja procesów jest **projekcją** ogólnej strategii multitenancy danych (→ [[wiedza/09-architektura/multitenancy]]) na tabele `ACT_*`:

| Strategia | Jak w Flowable | Kompromis |
|---|---|---|
| **Discriminator column** (`TENANT_ID_`) | Domyślne: wspólne tabele `ACT_*`, filtr po kolumnie | Najtaniej operacyjnie (1 baza, 1 migracja schematu), ale słabsza izolacja — błąd w filtrze = wyciek danych między tenantami; wszyscy dzielą pule/indeksy (noisy neighbor). |
| **Schema-per-tenant** | `MultiSchemaMultiTenantProcessEngineConfiguration` — osobny schemat/`DataSource` per tenant | Silniejsza izolacja, łatwiejszy per-tenant backup; koszt: N schematów do migrowania, cięższe zarządzanie, connection pools. |
| **Database-per-tenant** | Osobny `ProcessEngine`/`DataSource` per tenant | Najsilniejsza izolacja i „blast radius"; najdroższe operacyjnie, gorzej się skaluje przy wielu małych tenantach. |

Dla projektu rekrutacyjnego (wiele małych stowarzyszeń) **discriminator column** jest zwykle rozsądnym startem — z pełną świadomością, że izolacja stoi na poprawności filtrów aplikacyjnych, nie na barierze bazy.

### Wersjonowanie — automatyczne przy każdym deploymencie

Za każdym razem, gdy admin wdroży zmodyfikowany BPMN o **tym samym `key`** (i tym samym `tenantId`), Flowable **inkrementuje `VERSION_`**: v1 → v2 → v3… Kluczowe konsekwencje dla edycji procesu przez admina:

1. **Instancje już uruchomione zostają na swojej wersji.** Kandydat, który wystartował na v1, **dokończy rekrutację według v1**, nawet po wdrożeniu v2. Flowable trzyma `PROC_DEF_ID_` (konkretną wersję) w wierszu instancji w `ACT_RU_EXECUTION`. To celowe — nie chcemy w połowie procesu podmienić kandydatowi reguł.
2. **Nowe instancje startują na najnowszej wersji** (dla `...ByKeyAndTenantId` — bez podania konkretnej definicji).
3. **In-flight NIE migrują automatycznie.** Jeśli admin dodał krok, który powinien dotyczyć trwających aplikacji, sama zmiana definicji tego nie załatwi — potrzebna świadoma migracja.

To bezpośrednio kształtuje UX admina: „zapisz nową wersję procesu" ≠ „zmień proces wszystkim tu i teraz". Trzeba to komunikować (np. „zmiana obejmie nowe zgłoszenia; trwające dokończą się po staremu").

### Migracja instancji między wersjami

Gdy jednak trzeba przenieść trwające instancje na nową definicję (np. krytyczna poprawka, usunięty krok), używa się `ProcessMigrationService` / `ProcessInstanceMigration`:

```java
ProcessInstanceMigrationBuilder builder = processMigrationService
    .createProcessInstanceMigrationBuilder()
    .migrateToProcessDefinition(targetProcDefId);   // konkretna wersja docelowa
// opcjonalny mapping activity, gdy zmieniły się identyfikatory kroków:
// .addActivityMigrationMapping("staraNazwaTaska", "nowaNazwaTaska")
builder.migrate(processInstanceId);
```

Migracja jest **nietrywialna**, gdy zmieniła się topologia (usunięto/przesunięto activity z aktywnymi tokenami) — trzeba jawnie zmapować, gdzie „przełożyć" token. Świadomość „kiedy jest potrzebna" ważniejsza niż implementacja: rutynowe zmiany (nowy proces dla nowych zgłoszeń) → migracja niepotrzebna; wymuszona zmiana dla trwających → migracja + mapping.

### Edycja procesu przez admina — Modeler vs własne UI

Admin ma zmienić proces **bez kodu**. Dwie drogi:

- **Flowable Modeler** (gotowy edytor graficzny, część Flowable UI Apps) — działa od ręki, zapisuje/deployuje BPMN. Zaleta: zero pracy, kompletny. Wada: obce UI, słabo integruje się z Twoim Angularem, trudniej egzekwować per-tenant sandbox i branding — słabszy „portfolio value".
- **Własne UI w Angularze na BPMN.js** — [BPMN.js](https://bpmn.io) renderuje i edytuje diagram po stronie frontu; użytkownik przeciąga klocki, front **eksportuje BPMN 2.0 XML** i wysyła go do backendu, który przez `RepositoryService` robi `deploy().tenantId(tenant)`. Zaleta: pełna kontrola (możesz ograniczyć paletę do dozwolonych klocków, dodać walidację, spójny UX), świetne na portfolio. Wada: więcej pracy, sam odpowiadasz za walidację i bezpieczeństwo.

Szkic przepływu (własne UI):

```
Angular (BPMN.js) ── edycja diagramu ──▶ export XML
        │
        ▼  PUT /api/tenants/{tenantId}/process  (body: BPMN XML)
Backend (Spring Boot):
  1. AuthZ: JWT.tenantId == {tenantId}?           ← Spring Security
  2. Walidacja XML: parsuje? tylko dozwolone elementy? brak zakazanych service task/script?
  3. repositoryService.createDeployment()
        .tenantId(tenantId).enableDuplicateFiltering()
        .addString("rekrutacja.bpmn20.xml", xml).deploy();
  4. ⟹ nowa VERSION_ dla (key='rekrutacja', tenantId)
```

### Bezpieczeństwo i sandbox — pod spodem najważniejsze

Skoro tenant przysyła BPMN, który silnik **wykona**, to jest to zdalne wykonanie logiki. Zagrożenia i obrona:

- **AuthZ na tenant** (→ [[wiedza/08-bezpieczenstwo/spring-security]]): admin edytuje/deployuje **wyłącznie** proces swojego tenantu. `tenantId` z path/body **musi** być porównany z `tenantId` z tokenu — nigdy zaufać wartości z requestu.
- **Sandbox service tasków — kluczowe.** BPMN pozwala na `serviceTask` z `flowable:class`, `flowable:expression`, `flowable:delegateExpression` i `scriptTask` (np. Groovy/JavaScript). Gdyby tenant mógł wpisać dowolną klasę/expression/skrypt, to **wykona dowolny kod na Twoim serwerze** (RCE) albo dowolny EL-expression (dostęp do beanów, `Runtime.exec`…). Dlatego:
  - **NIE** pozwalaj na dowolny Java class / script task / dowolny expression od tenanta.
  - Udostępnij **paletę predefiniowanych „klocków"** (własne, zaudytowane delegaty/beany, np. `wyslijMailaDoKandydata`, `oznaczEtap`), które użytkownik tylko konfiguruje parametrami.
  - Waliduj przysłany XML po stronie backendu: whitelist dozwolonych elementów i wartości `flowable:class`/`delegateExpression`; odrzucaj `scriptTask` i nieznane klasy. Rozważ wyłączenie/ograniczenie ekspresji i script engines w konfiguracji silnika.
- **Identyfikacja tenanta z JWT** (→ [[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]]): `tenantId` (np. custom claim / mapowanie z `org`) wyciągasz z tokenu w filtrze Spring Security i **propagujesz** do warstwy Flowable (np. przez `TenantInfoHolder` w request-scope albo jawnie do `...AndTenantId`). Cała ścieżka od JWT po zapytanie do `ACT_*` musi nieść ten sam tenant.

## 3. Dlaczego / kiedy — izolacja, bezpieczeństwo, pułapki

- **Kiedy discriminator vs osobny schemat/baza:** wiele małych tenantów, jeden zespół ops → discriminator column. Wymogi regulacyjne / silna izolacja / per-tenant backup → schema/DB-per-tenant. To ten sam kompromis co dla danych domenowych — trzymaj strategię `ACT_*` **spójną** ze strategią reszty danych (→ [[wiedza/09-architektura/multitenancy]]).
- **Kiedy migracja instancji:** domyślnie **nie** — nowa wersja obsłuży nowe zgłoszenia, trwające dokończą po staremu. Migruj tylko przy krytycznych zmianach dotyczących in-flight, ze świadomym mappingiem activity.
- **Modeler vs własne UI:** MVP/mało czasu → Modeler; portfolio + kontrola + sandbox → własne BPMN.js.

**Pułapki:**

- **Klucz procesu w kontekście tenanta:** unikalność definicji to para (`key`, `tenantId`), nie sam `key`. Start bez `tenantId` albo z pustym tenantem wskaże złą (lub żadną) definicję.
- **Duplicate filtering:** `enableDuplicateFiltering()` porównuje zasób w ramach tego samego (name, tenant) — bez niego **każdy zapis** (nawet identyczny) bije nową wersję, zaśmiecając linię wersji. Z nim: identyczny XML nie tworzy v+1.
- **Czyszczenie starych wersji:** wersje **narastają w nieskończoność** (i puchną `ACT_*`). Usunięcie definicji (`deleteDeployment(id, cascade=false)`) **nie usunie**, dopóki są aktywne instancje na tej wersji; `cascade=true` skasuje też instancje (uważaj!). Potrzebna polityka retencji + `ACT_HI_*` (history) i history cleanup jobs, żeby historia nie rosła bez końca.
- **Wycieki między tenantami:** przy discriminator column każde zapytanie „gołym" `RuntimeService`/`TaskService` bez tenanta może zwrócić dane innych tenantów. Zawsze filtruj po tenancie (jawnie lub przez `TenantInfoHolder`).
- **Timery/joby a tenant:** async jobs i timery też noszą tenant — pilnuj konfiguracji async executora, by nie „gubił" kontekstu tenanta.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

W projekcie rekrutacyjnym: stowarzyszenie „ASSOC-42" loguje się, admin w Angularze (BPMN.js) dorzuca do procesu krok „rozmowa kwalifikacyjna" i klika „Zapisz". Front eksportuje BPMN XML, wysyła `PUT /api/tenants/assoc-42/process`. Backend sprawdza `JWT.tenantId == "assoc-42"`, waliduje XML (whitelist klocków, brak script tasków), i:

```java
Deployment d = repositoryService.createDeployment()
    .tenantId("assoc-42").enableDuplicateFiltering()
    .addString("rekrutacja.bpmn20.xml", xml).deploy();
// ⟹ ACT_RE_PROCDEF: key=rekrutacja, tenant=assoc-42, version=3
```

Kandydat, który aplikował wczoraj (na v2), dokończy proces bez „rozmowy". Nowy kandydat startuje na v3:

```java
runtimeService.startProcessInstanceByKeyAndTenantId(
    "rekrutacja", "app-9981", Map.of("email", "kandydat@x.pl"), "assoc-42");
```

Równolegle „ASSOC-7" ma swoją niezależną linię `rekrutacja` v1 — inny proces, te same tabele `ACT_*`, rozdzielone `TENANT_ID_`.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, dlaczego proces siedzi w BPMN w bazie (edytowalny per tenant), a nie w kodzie.
- [ ] Zadeployuję i wystartuję proces per tenant (`tenantId` + `startProcessInstanceByKeyAndTenantId`).
- [ ] Wyjaśnię, że para (`key`, `tenantId`) to oś wersjonowania.
- [ ] Rozumiem, że in-flight zostają na swojej wersji, a nowe startują na najnowszej.
- [ ] Wiem, kiedy potrzebna jest migracja instancji, a kiedy nie.
- [ ] Uzasadnię discriminator column vs schema/DB-per-tenant dla `ACT_*`.
- [ ] Wskażę ryzyko RCE w service/script taskach i zaprojektuję sandbox (predefiniowane klocki).
- [ ] Powiążę tenant z JWT aż do zapytania Flowable.

### 🔲 Black-box check
- [ ] Gdzie fizycznie leży `tenantId` po deployu? (`ACT_RE_DEPLOYMENT`/`ACT_RE_PROCDEF.TENANT_ID_`)
- [ ] Co dokładnie bumpuje `VERSION_` i jak temu zapobiega duplicate filtering?
- [ ] Skąd instancja „wie", której wersji definicji używa? (`PROC_DEF_ID_` w `ACT_RU_EXECUTION`)
- [ ] Co się stanie przy `startProcessInstanceByKey` bez tenantId w środowisku multitenant?
- [ ] Jak `flowable:class`/`expression`/`scriptTask` mogą doprowadzić do RCE i jak to zablokować?
- [ ] Dlaczego usunięcie deploymentu bez `cascade` może się nie udać?

### 🎤 Pytania rekrutacyjne
- [ ] „Jak zrobisz, żeby każdy tenant miał własny, edytowalny proces bez zmian w kodzie?"
- [ ] „Co się dzieje z trwającymi procesami, gdy admin wdroży nową wersję?"
- [ ] „Modeler czy własny edytor na BPMN.js — co wybierzesz i dlaczego?"
- [ ] „Jak zabezpieczysz się przed tym, że tenant wgra proces wykonujący dowolny kod?"
- [ ] „Discriminator column vs osobna baza per tenant — kompromisy?"
- [ ] „Jak propagujesz tenanta z JWT do Flowable?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/11-flowable-bpmn.csv`.*

```
Jak we Flowable izolujesz procesy per tenant?;Nadajesz tenantId na deploymencie (.tenantId(...)), co propaguje TENANT_ID_ na definicje/instancje/taski.
Czym jest oś wersjonowania definicji w multitenancy Flowable?;Para (key, tenantId) — ten sam key może istnieć niezależnie dla wielu tenantów, każdy z własną linią wersji.
Jak wystartować instancję dla konkretnego tenanta?;runtimeService.startProcessInstanceByKeyAndTenantId(key, businessKey, vars, tenantId).
Kiedy Flowable inkrementuje VERSION_ definicji?;Przy każdym deploymencie BPMN o tym samym (key, tenantId) — chyba że enableDuplicateFiltering wykryje identyczny zasób.
Co robi enableDuplicateFiltering()?;Nie tworzy nowej wersji, jeśli deployowany zasób jest identyczny z ostatnim (name, tenant) — chroni przed zaśmiecaniem wersji.
Co dzieje się z uruchomionymi instancjami po wdrożeniu nowej wersji procesu?;Nic — dokańczają na swojej wersji (in-flight nie migrują automatycznie); tylko nowe instancje startują na najnowszej.
Skąd instancja wie, której wersji definicji używa?;Trzyma PROC_DEF_ID_ (konkretną wersję) w ACT_RU_EXECUTION, ustawiony przy starcie.
Kiedy potrzebna jest migracja instancji (ProcessInstanceMigration)?;Gdy trwające instancje trzeba świadomie przenieść na nową definicję; przy zmianie topologii wymaga mappingu activity.
Domyślna strategia multitenancy Flowable dla bazy?;Shared database, discriminator column — wspólne tabele ACT_* z kolumną TENANT_ID_.
Kompromis discriminator column vs schema/DB-per-tenant?;Discriminator: tanio operacyjnie, słabsza izolacja (stoi na filtrach). Schema/DB-per-tenant: silna izolacja, droższe operacyjnie.
Modeler vs własne UI (BPMN.js) do edycji procesu?;Modeler: gotowy, szybki, obce UI/słabsza kontrola. BPMN.js: więcej pracy, pełna kontrola i sandbox, lepsze portfolio.
Jak admin edytuje proces bez kodu przy własnym UI?;BPMN.js eksportuje BPMN XML → backend przez RepositoryService.deploy().tenantId(tenant) tworzy nową wersję.
Główne ryzyko bezpieczeństwa przy przyjmowaniu BPMN od tenanta?;serviceTask (flowable:class/expression) i scriptTask mogą wykonać dowolny kod (RCE) na serwerze.
Jak zabezpieczyć service taski przed RCE?;Zakazać dowolnych klas/expression/script tasków; udostępnić tylko predefiniowane, zaudytowane klocki (beany/delegaty) konfigurowane parametrami.
Jak zapewnić, że admin edytuje tylko swój proces?;Porównać tenantId z requestu z tenantId z JWT (Spring Security); nigdy nie ufać tenantId z body/path.
Skąd bierzesz tenantId dla Flowable?;Z JWT (custom claim / mapowanie org) w filtrze Spring Security, propagowany do warstwy Flowable (TenantInfoHolder lub jawnie ...AndTenantId).
Co się stanie przy startProcessInstanceByKey bez tenantId w środowisku multitenant?;Szuka definicji bez tenanta (TENANT_ID_ NULL) — trafia w złą/żadną i może rzucić wyjątkiem „definicja nie znaleziona".
Dlaczego deleteDeployment bez cascade może się nie udać?;Bo istnieją aktywne instancje na tej wersji; cascade=true usuwa też instancje/historię (ryzykowne).
Jak nie utopić się w starych wersjach definicji?;Polityka retencji + czyszczenie deploymentów oraz history cleanup jobs dla ACT_HI_*, bo wersje i historia narastają w nieskończoność.
Dlaczego proces trzymamy w BPMN w bazie, a nie w kodzie?;Żeby admin tenantu edytował go bez zmian w kodzie i redeployu — logika biznesowa w definicjach, nie w if-ach.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[projekt-rekrutacja/docs/ARCHITEKTURA]]
- [[wiedza/09-architektura/multitenancy]] — strategie izolacji danych (dyskryminator vs schemat/baza per tenant)
- [[wiedza/08-bezpieczenstwo/spring-security]] — autoryzacja: admin edytuje tylko swój tenant
- [[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]] — tenantId z JWT i propagacja do Flowable
