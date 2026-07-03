---
temat: "Kubernetes — orkiestracja kontenerów"
faza: 10
status: nieopanowany
priorytet: 🔴
tags: [java, cloud, kubernetes]
powiazane: ["[[wiedza/10-cloud-native/konteneryzacja]]", "[[wiedza/10-cloud-native/load-balancing]]", "[[wiedza/05-spring/actuator]]", "[[wiedza/05-spring/konfiguracja-profile]]"]
---

# Kubernetes — orkiestracja kontenerów

> **TL;DR:** **Kubernetes** to **container orchestrator**: opisujesz **stan pożądany** (desired state)
> w YAML, a klaster w pętli **reconciliation loop** dąży, by rzeczywistość mu odpowiadała — sam robi
> **scaling**, **self-healing**, **rolling updates**, **service discovery** i **load balancing**.
> Jako developer backendu nie zarządzasz kontenerami ręcznie: deklarujesz **Deployment + Service + probes + ConfigMap**,
> a k8s pilnuje, żeby *N* zdrowych replik Twojego Spring Boota żyło i było osiągalne.

## 1. Co — po co orkiestracja i architektura

Jeden kontener uruchomisz `docker run`. Ale produkcja to dziesiątki kontenerów na wielu maszynach, które muszą:
się **skalować** pod obciążeniem, **wstać z powrotem** po awarii (self-healing), **aktualizować bez downtime**
(rolling update), **znajdować się nawzajem** (service discovery) i **rozkładać ruch** (load balancing). Robienie tego
ręcznie nie skaluje się. **Orkiestrator** = warstwa, która to automatyzuje.

**Architektura klastra** — dwie warstwy:

```
┌──────────────────── CONTROL PLANE (mózg) ───────────────────┐
│  kube-apiserver   ← jedyna brama (REST); wszystko przez nią   │
│  etcd             ← rozproszony key-value store: stan klastra │
│  kube-scheduler   ← przypisuje Pody do węzłów (bin-packing)   │
│  controller-mgr   ← pętle kontrolerów (reconciliation)        │
└──────────────────────────────────────────────────────────────┘
        │  (apiserver ⇄ węzły)
┌────── WORKER NODE ──────┐  ┌────── WORKER NODE ──────┐
│ kubelet   (agent węzła) │  │ kubelet                 │
│ kube-proxy (sieć/Svc)   │  │ kube-proxy              │
│ container runtime       │  │ container runtime       │
│  (containerd/CRI-O)     │  │                         │
│  [Pod] [Pod] [Pod]      │  │  [Pod] [Pod]            │
└─────────────────────────┘  └─────────────────────────┘
```

- **API server** — jedyny punkt wejścia (`kubectl`, kontrolery, kubelet gadają tylko z nim). Waliduje i zapisuje do etcd.
- **etcd** — źródło prawdy o całym klastrze (spójny, rozproszony store). Utrata etcd = utrata stanu.
- **scheduler** — decyduje, na *który* węzeł trafi nowy Pod (uwzględnia requests, taints/tolerations, affinity).
- **controller manager** — zbiór **kontrolerów**, każdy pilnuje jednego typu obiektu (Deployment→ReplicaSet→Pody).
- **kubelet** — agent na węźle: uruchamia kontenery (przez CRI), raportuje zdrowie, wykonuje **probes**.
- **kube-proxy** — programuje reguły sieciowe (iptables/IPVS) realizujące **Service** i load balancing.
- **container runtime** — faktycznie uruchamia kontenery (containerd, CRI-O; Docker już nie bezpośrednio).

## 2. Jak — model deklaratywny, obiekty, probes pod spodem

### Model deklaratywny i reconciliation loop
Nie mówisz „uruchom kontener" (imperatyw). Mówisz „chcę 3 repliki tego obrazu" (**deklaracja**). Zapisujesz to
w etcd przez apiserver. Odpowiedni **kontroler** działa w nieskończonej pętli:

```
loop forever:
  observed = obecny stan (z etcd / węzłów)
  desired  = stan zadeklarowany (spec)
  if observed != desired: podejmij działania korygujące (twórz/usuwaj Pody)
```

To jest **reconciliation loop** — serce k8s. Dlatego gdy Pod padnie, kontroler *sam* tworzy nowy: obserwuje
2 repliki zamiast 3 i koryguje. **Self-healing to konsekwencja modelu deklaratywnego**, nie osobny mechanizm.

### Obiekty (od najmniejszego do złożonych)
- **Pod** — najmniejsza jednostka wdrożenia: **1+ kontenerów** dzielących sieć (jeden IP) i wolumeny.
  Zwykle 1 kontener główny + ewentualne **sidecary** (np. proxy, log-shipper). Pod jest **efemeryczny** —
  ginie i nie wraca „ten sam"; dostaje nowy IP. Dlatego nie łączysz się nigdy po IP Poda.
- **ReplicaSet** — pilnuje, by żyło *N* identycznych Podów. Rzadko używasz go wprost.
- **Deployment** — zarządza ReplicaSet-ami; daje **rolling update** (nowy RS rośnie, stary maleje — bez downtime)
  i **rollback** (`kubectl rollout undo`). To główny obiekt dla **stateless** usług (typowy Spring Boot API).
- **Service** — **stabilny endpoint** (stały IP + nazwa DNS) przed efemerycznymi Podami + **load balancing**.
  Typy: **ClusterIP** (wewnątrz klastra, domyślny), **NodePort** (port na każdym węźle), **LoadBalancer**
  (zewnętrzny LB od chmury). Service kieruje ruch tylko do Podów **Ready** (patrz readiness probe).
- **Ingress** — **L7 routing HTTP(S)**: reguły host/path → różne Service, terminacja **TLS**, jeden wejściowy LB
  dla wielu usług. Wymaga Ingress Controllera (nginx, Traefik). Szczegóły warstwy 7 → [[wiedza/10-cloud-native/load-balancing]].
- **ConfigMap / Secret** — konfiguracja niewrażliwa / wrażliwa. Wstrzykiwana jako **env** albo **wolumin** (pliki).
  Secret jest tylko base64 (nie szyfrowanie!) — włącz encryption-at-rest / użyj zewnętrznego vaulta.
  Mapuje się to na profile Springa — → [[wiedza/05-spring/konfiguracja-profile]].
- **Namespace** — logiczna izolacja (dev/test/prod, zespoły); zakres nazw, quot, RBAC.
- **PersistentVolume (PV) / PersistentVolumeClaim (PVC)** — trwały storage niezależny od cyklu życia Poda.
  PVC = „żądanie" storage, PV = faktyczny zasób (dysk chmury). Konieczne, gdy dane mają przeżyć restart.
- **StatefulSet** — dla **stateful** workloadów (bazy, Kafka): stabilna tożsamość (`pod-0`, `pod-1`), stabilny
  storage per Pod, uporządkowany rollout. Twój Spring Boot zwykle NIE jest StatefulSetem — bazę trzymaj poza.
- **Job / CronJob** — zadania jednorazowe (batch, migracja) / cykliczne (jak `cron`, np. nocny raport).

### Probes — jak k8s wie, czy Pod żyje i jest gotowy
kubelet odpytuje kontener (HTTP/TCP/exec). Trzy typy — **nie mylić**:
- **liveness** — „czy proces się nie zawiesił?" Fail → kubelet **restartuje** kontener. (np. deadlock)
- **readiness** — „czy gotowy przyjąć ruch?" Fail → Pod **usuwany z Service** (przestaje dostawać żądania),
  ale NIE restart. Kluczowe podczas startu, warm-upu JIT, chwilowego przeciążenia.
- **startup** — „czy skończył się **wolny start**?" Wstrzymuje liveness/readiness do końca startu (JVM startuje
  wolno). Zapobiega temu, że liveness zabije kontener, zanim ten w ogóle wstanie.

W Spring Boot 3 mapują się na **Actuator health groups**: `/actuator/health/liveness` i `/actuator/health/readiness`
(włącz `management.endpoint.health.probes.enabled=true`; na k8s auto-detekcja). Szczegóły → [[wiedza/05-spring/actuator]].

## 3. Dlaczego / kiedy — skalowanie, zasoby, pułapki JVM

- **Skalowanie ręczne vs HPA.** `kubectl scale` ustawia repliki ręcznie. **HPA (Horizontal Pod Autoscaler)**
  robi to **automatycznie** wg metryk (CPU/memory z metrics-server, albo custom/RPS). HPA obserwuje obciążenie
  i zmienia `replicas` Deploymentu — znów reconciliation. (Osobno: VPA skaluje *w górę* zasoby jednego Poda.)

- **resources: requests vs limits.** `requests` = ile scheduler *rezerwuje* (podstawa decyzji o umieszczeniu +
  QoS). `limits` = twardy sufit. Przekroczenie **memory limit → kontener OOMKilled** (SIGKILL, kod 137).
  Przekroczenie **CPU limit → throttling** (spowolnienie, nie zabicie).

- **JVM w cgroup — najważniejsza pułapka backendowca Java (→ [[wiedza/10-cloud-native/konteneryzacja]]).**
  JVM ustala domyślny heap jako % pamięci *widzianej*. Nowoczesne JDK (11+/17/21) są **container-aware** —
  czytają limit z cgroup. Ale: (1) **heap ≠ cały RSS** (metaspace, stosy, bufory direct, kod JIT też liczą się
  do limitu), więc ustawiaj `-XX:MaxRAMPercentage` (np. 70–75), nie oddawaj 100% RAM heapowi — inaczej **OOMKilled**
  mimo że „heap się mieści". (2) Zbyt niski CPU limit → JIT/GC głodzą, throughput leci.

- **Graceful shutdown.** Przy skalowaniu w dół / rolling update k8s wysyła **SIGTERM**, czeka
  `terminationGracePeriodSeconds` (domyślnie 30s), potem **SIGKILL**. Spring Boot ma **graceful shutdown**
  (`server.shutdown=graceful`, `spring.lifecycle.timeout-per-shutdown-phase`): przestaje przyjmować nowe
  żądania, dokańcza bieżące. Ustaw grace period k8s > timeout Springa, inaczej zabijesz in-flight requesty.
  Uwaga: readiness powinno failować *przed* zamknięciem, by Service przestał kierować ruch.

- **Ekosystem (świadomość, nie trzeba wdrażać sam).**
  - **Helm** — menedżer pakietów k8s: „chart" = sparametryzowany szablon (values.yaml) → generuje YAML-e. Dla reuse/wersjonowania.
  - **kustomize** — nakładanie „patchy" na bazowe manifesty (overlays dev/prod) bez szablonów; wbudowane w `kubectl -k`.
  - **kind / minikube** — lokalny klaster k8s do dev/testów (kind = k8s w kontenerach Dockera).
  - **GitOps / ArgoCD** — Git jako źródło prawdy; ArgoCD **synchronizuje** stan klastra do repo (deklaratywnie, audytowalnie).

## Przykład w praktyce
Deployment + Service + probes + ConfigMap dla Spring Boot (Java 21, Spring Boot 3):

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: orders-config }
data:
  SPRING_PROFILES_ACTIVE: "prod"
  APP_GREETING: "witaj z k8s"
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: orders }
spec:
  replicas: 3                     # desired state — 3 repliki
  selector: { matchLabels: { app: orders } }
  template:
    metadata: { labels: { app: orders } }
    spec:
      terminationGracePeriodSeconds: 40   # > shutdown-timeout Springa
      containers:
        - name: orders
          image: registry.example.com/orders:1.4.2
          ports: [{ containerPort: 8080 }]
          envFrom: [{ configMapRef: { name: orders-config } }]
          env:
            - name: JAVA_TOOL_OPTIONS
              value: "-XX:MaxRAMPercentage=75.0"   # heap = 75% limitu, reszta na metaspace/stosy/GC
          resources:
            requests: { cpu: "500m", memory: "512Mi" }
            limits:   { cpu: "1",    memory: "768Mi" }   # memory>heap: bufor na non-heap
          startupProbe:                    # daj JVM wstać (wolny start)
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            failureThreshold: 30
            periodSeconds: 5
          livenessProbe:                   # zawieszony → restart
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            periodSeconds: 10
          readinessProbe:                  # niegotowy → poza Service
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata: { name: orders }
spec:
  selector: { app: orders }        # kieruje na Pody z tym labelem, tylko Ready
  ports: [{ port: 80, targetPort: 8080 }]
  type: ClusterIP                  # inne usługi wołają http://orders
```

Wdrożenie: `kubectl apply -f orders.yaml`. Nowa wersja obrazu → zmiana `image` + `apply` = **rolling update**
(Deployment wymienia Pody stopniowo). Coś nie tak → `kubectl rollout undo deployment/orders`. Inne serwisy
łączą się przez `http://orders` (service discovery po DNS), nie po IP Poda.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię model deklaratywny i **reconciliation loop** oraz że self-healing z niego wynika.
- [ ] Naszkicuję architekturę: control plane (apiserver/etcd/scheduler/controller-mgr) vs węzeł (kubelet/kube-proxy/runtime).
- [ ] Odróżnię Pod / ReplicaSet / Deployment i wiem, po co Deployment (rolling update + rollback).
- [ ] Wyjaśnię Service (ClusterIP/NodePort/LoadBalancer) i Ingress oraz różnicę L4 vs L7.
- [ ] Rozróżnię liveness / readiness / startup i zmapuję je na Actuator health groups.
- [ ] Skonfiguruję requests/limits i wiem, dlaczego JVM potrzebuje `MaxRAMPercentage` (OOMKilled).
- [ ] Wiem, co robi HPA i jak działa graceful shutdown (SIGTERM → grace period → SIGKILL).

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co się dzieje krok po kroku po `kubectl apply -f deploy.yaml`? (apiserver→etcd→controller→scheduler→kubelet)
- [ ] Dlaczego nie wolno łączyć się po IP Poda, a wolno przez Service? (efemeryczność vs stabilny endpoint)
- [ ] Czym różni się fail liveness od fail readiness? (restart kontenera vs usunięcie z Service)
- [ ] Dlaczego kontener dostaje OOMKilled mimo że „heap się mieści"? (non-heap wlicza się do memory limit)
- [ ] Jak Deployment robi rolling update bez downtime? (dwa ReplicaSety, jeden rośnie, drugi maleje + readiness)
- [ ] Czemu StatefulSet dla bazy, a Deployment dla stateless API?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Czym różni się Pod od Deploymentu i po co warstwa ReplicaSet?"
- [ ] „Jakie masz typy Service i kiedy Ingress zamiast LoadBalancer?"
- [ ] „Jak skonfigurujesz probes dla Spring Boota i jak mapują się na Actuator?"
- [ ] „Aplikacja Java jest OOMKilled na k8s — jak diagnozujesz i naprawiasz?"
- [ ] „Jak zapewnisz zero-downtime deploy i graceful shutdown in-flight requestów?"
- [ ] „ConfigMap vs Secret — czym Secret NIE jest?" (nie szyfrowanie, tylko base64)

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/10-cloud-native.csv`.*

```
Czym jest Kubernetes w jednym zdaniu?;Container orchestrator — deklarujesz stan pożądany, a klaster w reconciliation loop dąży do niego (scaling, self-healing, rolling updates, service discovery, load balancing).
Co to model deklaratywny w k8s?;Opisujesz stan POŻĄDANY (spec w YAML), nie kroki; kontrolery same doprowadzają rzeczywistość do tego stanu.
Co to reconciliation loop?;Nieskończona pętla kontrolera: porównuje stan obserwowany z pożądanym i koryguje różnicę; stąd self-healing.
Z czego składa się control plane?;kube-apiserver, etcd, kube-scheduler, kube-controller-manager.
Do czego służy etcd?;Rozproszony, spójny key-value store — źródło prawdy o całym stanie klastra.
Co robi kube-scheduler?;Przypisuje nowe Pody do węzłów wg requests, taints/tolerations, affinity.
Co jest na węźle roboczym?;kubelet (agent, uruchamia kontenery i probes), kube-proxy (sieć/Service), container runtime (containerd/CRI-O).
Co to Pod?;Najmniejsza jednostka wdrożenia — 1+ kontenerów dzielących sieć i wolumeny; efemeryczny, dostaje nowy IP przy odtworzeniu.
Czym różni się Deployment od ReplicaSet?;Deployment zarządza ReplicaSet-ami i daje rolling update oraz rollback; ReplicaSet tylko pilnuje N replik.
Po co Service, skoro Pody mają IP?;Pody są efemeryczne (zmienne IP); Service to stabilny IP+DNS z load balancingiem, kierujący ruch tylko do Podów Ready.
Typy Service?;ClusterIP (wewnątrz klastra, domyślny), NodePort (port na węzłach), LoadBalancer (zewnętrzny LB chmury).
Czym jest Ingress?;L7 routing HTTP(S): reguły host/path do różnych Service, terminacja TLS, jeden wejściowy LB (wymaga Ingress Controllera).
ConfigMap vs Secret?;Oba wstrzykiwane jako env lub wolumin; Secret na wrażliwe dane, ale to tylko base64 (nie szyfrowanie) — trzeba encryption-at-rest.
Liveness probe — co robi fail?;kubelet RESTARTUJE kontener (uznaje go za zawieszony).
Readiness probe — co robi fail?;Pod jest usuwany z Service (przestaje dostawać ruch), bez restartu.
Po co startup probe?;Wstrzymuje liveness/readiness do końca wolnego startu (np. JVM), by liveness nie zabił wstającego kontenera.
Jak probes mapują się na Spring Boot?;Actuator health groups: /actuator/health/liveness i /actuator/health/readiness (auto na k8s).
requests vs limits?;requests = rezerwacja dla schedulera i QoS; limits = twardy sufit (przekroczenie memory→OOMKilled, CPU→throttling).
Dlaczego JVM w kontenerze bywa OOMKilled mimo mieszczącego się heapu?;Do memory limit wlicza się też non-heap (metaspace, stosy, direct, kod JIT); ustaw MaxRAMPercentage ~75, nie 100.
Co robi HPA?;Horizontal Pod Autoscaler — automatycznie zmienia liczbę replik Deploymentu wg metryk (CPU/memory/custom).
Jak wygląda graceful shutdown na k8s?;SIGTERM → czekanie terminationGracePeriodSeconds → SIGKILL; Spring z server.shutdown=graceful dokańcza in-flight requesty.
Helm vs kustomize?;Helm = pakiety/szablony z values.yaml; kustomize = nakładki (overlays) patchujące bazowe manifesty bez szablonów.
Co to GitOps/ArgoCD?;Git jako źródło prawdy o stanie klastra; ArgoCD deklaratywnie synchronizuje klaster do repo.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/10-cloud-native/konteneryzacja]] · [[wiedza/10-cloud-native/load-balancing]] · [[wiedza/05-spring/actuator]] · [[wiedza/05-spring/konfiguracja-profile]]
