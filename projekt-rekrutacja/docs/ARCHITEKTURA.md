# Projekt: System rekrutacji dla stowarzyszeń

System SaaS, w którym **stowarzyszenia (tenanci)** prowadzą procesy rekrutacji członków/wolontariuszy.
Rdzeniem jest **silnik procesów Flowable (BPMN)**, dzięki czemu admin danego tenantu może
**dopracować/skonfigurować własny proces** rekrutacji bez zmian w kodzie.

> Status: **faza projektowa**. Kod powstaje od fazy 5 roadmapy. Ten dokument to żywa architektura —
> decyzje zapisujemy jako [[#ADR — rejestr decyzji|ADR-y]].

## Cel projektu (po co istnieje)
Portfolio dowodzące kompetencji w: konteneryzacji, load-balancingu, **multitenancy**, cloud-native,
oraz w osadzeniu **konfigurowalnego silnika procesów** w realnym produkcie.

## Kluczowe wymagania
- **Multitenancy** — pełna izolacja danych między stowarzyszeniami.
- **Konfigurowalność procesu** — admin tenantu modeluje/wersjonuje własny proces (BPMN + DMN reguły).
- **Cloud-native** — konteneryzacja, skalowanie poziome, bezstanowość, observability.
- **Bezpieczeństwo** — OIDC (Keycloak), role per tenant, izolacja.

## Aktorzy
- **Kandydat** — składa aplikację, wykonuje zadania (formularze) przypisane przez proces.
- **Rekruter (tenant)** — obsługuje zadania użytkownika (user tasks) w procesie.
- **Admin tenantu** — konfiguruje definicję procesu, formularze, reguły decyzyjne.
- **Operator platformy** — zarządza tenantami, monitoruje system.

## Architektura logiczna (start: monolit modularny)

```
                 Angular SPA  ──(JWT/OIDC)──►  API Gateway / Ingress (L7 LB)
                                                       │
                                       ┌───────────────┴───────────────┐
                                       │      Spring Boot (monolit       │
                                       │      modularny, skalowany       │
                                       │      poziomo, bezstanowy)       │
                                       │  ┌─────────────────────────┐    │
                                       │  │ Moduł: Tenant & Auth     │    │
                                       │  │ Moduł: Rekrutacja (API)  │    │
                                       │  │ Moduł: Proces (Flowable) │    │
                                       │  │ Moduł: Formularze/DMN    │    │
                                       │  └─────────────────────────┘    │
                                       └───────┬─────────────┬───────────┘
                                               │             │
                                       PostgreSQL        Keycloak (OIDC)
                                    (multitenant:       (realm per tenant
                                     schema/dyskrym.)    lub grupy)
                                               │
                                        Observability:
                                     Micrometer→Prometheus→Grafana, OTel
```

> **Decyzja świadoma:** zaczynamy od **monolitu modularnego**, nie mikroserwisów — patrz [[wiedza/09-architektura/monolit-vs-mikroserwisy]].
> Granice modułów projektujemy tak, by *dało się* wydzielić serwis później, jeśli zajdzie potrzeba.

## Multitenancy — strategia (do rozstrzygnięcia w ADR-002)
Trzy klasyczne podejścia (szczegóły: [[wiedza/09-architektura/multitenancy]]):
1. **Baza per tenant** — najlepsza izolacja, najdroższe operacyjnie.
2. **Schemat per tenant** (PostgreSQL schemas) — dobry kompromis izolacja/koszt. **Kandydat domyślny.**
3. **Wspólny schemat + kolumna `tenant_id`** (dyskryminator) — najtańsze, najsłabsza izolacja, ryzyko wycieku danych.

Flowable wspiera multitenancy natywnie (`tenantId` na deploymentach/procesach) — to wpływa na wybór.

## Flowable — rola
- Definicje procesów BPMN **deployowane per `tenantId`** → każdy tenant ma własną wersję procesu.
- **User tasks** = zadania rekruterów; **service tasks** = automatyka (mejle, scoring); **DMN** = reguły kwalifikacji.
- Admin tenantu edytuje proces (Flowable Modeler lub własne UI w Angularze) → nowy **deployment z wersjonowaniem**.
- Szczegóły: [[wiedza/11-flowable-bpmn/flowable-spring]] · [[wiedza/11-flowable-bpmn/multitenant-procesy]]

## Stos technologiczny (planowany)
| Warstwa | Wybór | Uzasadnienie |
|---|---|---|
| Frontend | Angular | Twoja mocna strona; klient TS generowany z OpenAPI |
| Backend | Java 21 (LTS) + Spring Boot 3 | Standard rynkowy |
| Proces | Flowable (embedded) | Wymóg projektu; konfigurowalność |
| Baza | PostgreSQL | Schematy = multitenancy; dojrzała |
| Auth | Keycloak (OIDC) | Realmy/role per tenant |
| Migracje | Flyway | Wersjonowanie schematu |
| Konteneryzacja | Podman/Docker, multi-stage | Lokalnie masz Podman |
| Orkiestracja | Kubernetes (kind lokalnie) | Skalowanie, LB, probes |
| Observability | Micrometer + Prometheus + Grafana + OpenTelemetry | Cloud-native standard |
| CI/CD | GitHub Actions | Build→test→obraz→deploy |

## Roadmapa projektu (kamienie milowe)
1. **MVP monolit** — encje tenant/aplikacja, REST CRUD, Flyway, Postgres w Testcontainers. *(po fazie 6)*
2. **Proces** — Flowable embedded, prosty proces rekrutacji (1 user task + 1 bramka). *(faza 11)*
3. **Multitenancy** — izolacja danych (schema per tenant) + `tenantId` we Flowable. *(faza 9)*
4. **Auth** — Keycloak OIDC, role per tenant, method security. *(faza 8)*
5. **Konfigurowalność** — admin edytuje/wersjonuje proces. *(faza 11)*
6. **Cloud-native** — Dockerfile, k8s (deployment+ingress+HPA), probes, metryki. *(faza 10)*
7. **CI/CD + observability** — pipeline, Grafana dashboard, tracing. *(faza 10)*

## ADR — rejestr decyzji
Każda istotna decyzja architektoniczna = osobny plik `docs/adr/NNNN-tytul.md` (szablon: `docs/adr/_TEMPLATE.md`).
- ADR-001 — Monolit modularny zamiast mikroserwisów *(do spisania)*
- ADR-002 — Strategia multitenancy *(do spisania)*
- ADR-003 — Embedded Flowable vs osobny serwis procesowy *(do spisania)*

## Otwarte pytania (do rozstrzygnięcia w trakcie nauki)
- Schema-per-tenant vs dyskryminator — jak to gra z Hibernate i Flowable jednocześnie?
- Keycloak: realm per tenant czy jeden realm + grupy? (kompromis izolacja/operacje)
- Edycja procesu: Flowable Modeler (gotowy) czy własne UI w Angularze (więcej pracy, lepsze portfolio)?
