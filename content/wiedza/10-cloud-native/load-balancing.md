---
temat: "Load balancing i skalowanie poziome"
faza: 10
status: nieopanowany
priorytet: 🟡
tags: [java, cloud, load-balancing, skalowanie]
powiazane: ["[[wiedza/10-cloud-native/12-factor]]", "[[wiedza/10-cloud-native/kubernetes]]", "[[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]]", "[[wiedza/05-spring/actuator]]"]
---

# Load balancing i skalowanie poziome

> **TL;DR:** **Load balancer** rozkłada ruch na wiele instancji → większa **przepustowość**, **dostępność**
> i brak **single point of failure**. Kluczem jest **skalowanie poziome** (więcej instancji), a to wymaga
> aplikacji **bezstanowej** (stateless — [[wiedza/10-cloud-native/12-factor]]): stąd **JWT** zamiast sesji
> serwerowej. LB działa w **L4** (TCP/UDP, szybki) lub **L7** (HTTP, routing/TLS/Ingress); w Kubernetes
> to **Service** (L4 przez kube-proxy) + **Ingress** (L7) + **HPA** (autoskalowanie).

## 1. Co — definicja i API

**Load balancing** to rozdzielanie przychodzących żądań pomiędzy wiele identycznych instancji tej samej aplikacji.
Po co:

- **Przepustowość (throughput):** jedna instancja obsłuży N żądań/s; dziesięć instancji — teoretycznie 10×N.
- **Dostępność (availability):** gdy jedna instancja padnie, LB kieruje ruch do pozostałych — **brak single point
  of failure** (o ile sam LB też jest redundantny; zwykle jest w klastrze/HA).
- **Elastyczność:** dokładasz i zabierasz instancje pod obciążenie (autoskalowanie).

Dwa kierunki skalowania:

```
Skalowanie PIONOWE (scale up)          Skalowanie POZIOME (scale out)
  ┌──────────────┐                       ┌────┐ ┌────┐ ┌────┐ ┌────┐
  │  1 instancja │  → mocniejsza →        │ i1 │ │ i2 │ │ i3 │ │ i4 │
  │  2 vCPU 4GB  │     8 vCPU 32GB        └────┘ └────┘ └────┘ └────┘
  └──────────────┘                              ▲   Load Balancer  ▲
```

- **PIONOWE (vertical / scale up):** mocniejsza maszyna (więcej CPU/RAM). Prosto, bez zmian w aplikacji, ale:
  ma **twardy sufit** (największa maszyna ma granicę), jest **dyskretne i drogie** (rośnie nieliniowo),
  wymaga zwykle **restartu** (downtime) i **nie usuwa SPOF** — to wciąż jedna instancja.
- **POZIOME (horizontal / scale out):** więcej instancji za load balancerem. Skaluje się (prawie) **liniowo
  i bez limitu**, zapewnia **odporność na awarie**, pozwala na **rolling updates** bez downtime. Koszt:
  **złożoność** (potrzebny LB, service discovery) i **twardy warunek bezstanowości**.

**W cloud-native preferujemy poziome**, bo pasuje do commodity hardware / kontenerów, daje HA i elastyczne
autoskalowanie (płacisz za to, czego używasz). To wprost realizacja zasady z [[wiedza/10-cloud-native/12-factor]]:
procesy są **bezstanowe i wymienne** (disposable).

## 2. Jak — L4 vs L7, algorytmy, k8s pod spodem

### Warstwy LB: L4 vs L7 (model OSI)

| | **L4 (transport)** | **L7 (aplikacyjny)** |
|---|---|---|
| Operuje na | TCP/UDP, IP:port | HTTP/HTTPS (ścieżka, nagłówki, host, cookie) |
| Zna treść HTTP? | Nie (tylko pakiety/połączenia) | Tak (rozumie żądanie) |
| Możliwości | tylko przekierowanie połączenia | **routing po ścieżce/hoście/nagłówku**, **terminacja TLS**, rewrite, sticky sessions, rate-limit |
| Wydajność | bardzo szybki, mały narzut | wolniejszy (parsuje HTTP), ale bogatszy |
| Przykłady | AWS NLB, `kube-proxy`, HAProxy w trybie TCP | **Nginx**, **HAProxy** (http), Envoy, **Ingress** ([[wiedza/10-cloud-native/kubernetes]]), API Gateway |

**L4** nie wie, że to HTTP — po prostu rozdziela połączenia TCP (świetny do baz, gRPC, protokołów binarnych).
**L7** widzi żądanie i potrafi np. wysłać `/api/*` do serwisu A, a `/img/*` do serwisu B, terminować TLS
(deszyfruje HTTPS raz na wejściu — **TLS termination**) i wstrzykiwać nagłówki (`X-Forwarded-For`).

### Algorytmy dystrybucji

- **Round-robin** — po kolei, instancja po instancji. Domyślny, prosty; zakłada podobne obciążenie żądań.
- **Least connections** — do instancji z najmniejszą liczbą aktywnych połączeń. Lepszy przy nierównych,
  długo trwających żądaniach.
- **Weighted (round-robin / least-conn)** — instancjom nadaje się wagi (mocniejsza maszyna → więcej ruchu);
  przydatne w **canary** (np. 5% ruchu do nowej wersji).
- **IP hash** — hash adresu klienta → ta sama instancja (prymitywne "przyklejenie" bez cookie).

### Health checks i sticky sessions

- **Health checks:** LB okresowo odpytuje instancje i kieruje ruch **tylko do zdrowych**. W Spring Boot to
  **readiness probe** — endpoint `/actuator/health/readiness` ([[wiedza/05-spring/actuator]]). Różnica:
  **liveness** = "czy żyje? jeśli nie — restart"; **readiness** = "czy gotowa przyjmować ruch? jeśli nie —
  wypnij z LB, ale nie restartuj" (np. rozgrzewa się, przeciążona).
- **Sticky sessions (session affinity):** LB przypina klienta do jednej instancji (przez cookie w L7 lub IP hash
  w L4). Potrzebne **tylko** gdy stan trzymamy w pamięci instancji — czego chcemy uniknąć (patrz sekcja 3).

### Pod spodem w Kubernetes

- **Service (ClusterIP):** stabilny wirtualny IP + DNS dla zbioru podów (selektor po labelach). Ruch jest
  load-balansowany **po podach** — realizuje to **`kube-proxy`** (iptables/IPVS lub eBPF w Cilium), więc to
  w istocie **L4**, round-robin/losowo. `endpoints`/`EndpointSlice` zawierają tylko **gotowe** pody
  (readiness!).
- **Ingress:** obiekt **L7** — reguły routingu HTTP (host/ścieżka) obsługiwane przez **Ingress Controller**
  (Nginx, Traefik, HAProxy, Envoy/Gateway API). To on terminuje TLS i kieruje do właściwego Service.
- **HPA (HorizontalPodAutoscaler):** automatycznie zmienia liczbę replik Deploymentu wg metryk (CPU, pamięć,
  custom). To **skalowanie poziome w akcji** ([[wiedza/10-cloud-native/kubernetes]]).

### Client-side vs server-side LB

- **Server-side:** klient uderza w jeden adres LB, LB wybiera instancję (Nginx, Ingress, Service). Klient nic
  nie wie o topologii.
- **Client-side:** klient sam zna listę instancji (z service discovery) i sam wybiera. W ekosystemie Spring:
  **Spring Cloud LoadBalancer** (następca **Ribbon**, dziś deprecated), integracja z `WebClient`/`RestClient`
  i discovery (Eureka/Consul). Zaleta: brak dodatkowego hopa; wada: logika w kliencie. W k8s zwykle
  wystarcza Service, więc client-side LB jest rzadziej potrzebny.

### Reverse proxy, DNS, CDN (świadomość)

- **Reverse proxy** (Nginx / API Gateway) stoi przed aplikacją: terminuje TLS, buforuje, kompresuje, robi
  rate-limiting, auth, routing — często pełni też rolę L7 LB.
- **DNS load balancing:** wiele rekordów A dla jednej nazwy → klient dostaje różne IP. Tanie i globalne, ale
  **grube** (cachowanie DNS, brak health-checków w klasycznym DNS, brak kontroli sesji).
- **CDN:** rozprasza treści statyczne (i coraz częściej dynamiczne przez edge) geograficznie bliżej użytkownika
  — odciąża origin i skraca latencję.

## 3. Dlaczego / kiedy — bezstanowość i pułapki

**Twardy warunek skalowania poziomego: aplikacja BEZSTANOWA (stateless).** Skoro kolejne żądania tego samego
użytkownika mogą trafić do **różnych instancji**, żadna instancja nie może przechowywać stanu sesji lokalnie
w pamięci — bo inna instancja go nie zobaczy (użytkownik "wylogowuje się" losowo, koszyk znika).

Konsekwencje projektowe:

- **JWT zamiast sesji serwerowej:** stan sesji (tożsamość, role) trzymany jest **po stronie klienta** w
  podpisanym tokenie ([[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]]). Każda instancja waliduje token
  **niezależnie** (podpisem/kluczem publicznym), bez wspólnego magazynu sesji → idealne do scale-out.
- **Gdyby jednak sesja w pamięci:** masz dwie gorsze opcje:
  1. **Sticky sessions** — przypnij klienta do instancji. Wady: **nierównomierne obciążenie** (długie sesje
     przeciążają jeden pod), utrata sesji przy śmierci/restarcie instancji, kłopot z rolling updates i
     autoskalowaniem (nowe pody nie przejmują istniejących sesji).
  2. **Zewnętrzny session store (Redis)** — Spring Session przenosi sesję do współdzielonego Redisa.
     Instancje pozostają "stateless" wobec siebie, ale: dodajesz **sieciowy hop** i **zależność** (Redis
     staje się nowym SPOF, jeśli nie w HA), rośnie **latencja** i złożoność operacyjna. Działa, lecz JWT
     jest zwykle prostszy dla API.

**Kiedy pionowe mimo wszystko:** stanowe/monolityczne systemy, bazy danych (trudno skalować poziomo zapisy),
gdy koszt refaktoryzacji > korzyść. Zwykle skaluje się poziomo warstwę aplikacji, a pionowo/przez sharding — bazę.

### Pułapki

- **Stan w pamięci przy skalowaniu** — najczęstszy błąd; działa na 1 replice, sypie się na wielu (sesje,
  cache lokalny, liczniki, `@Scheduled` odpalany na każdej instancji → potrzeba ShedLock).
- **Brak health-checków / źle skonfigurowany readiness** — LB kieruje ruch do instancji, która jeszcze się
  nie rozgrzała lub już umiera → błędy 5xx dla użytkowników.
- **Brak connection draining (graceful shutdown)** — przy wdrożeniu/scale-in pod jest zabijany w trakcie
  obsługi żądań. Trzeba: readiness=false → LB przestaje słać nowy ruch → dokończ trwające żądania →
  `preStop` hook + `terminationGracePeriodSeconds` + `server.shutdown=graceful` w Spring Boot.
- **Thundering herd** — po awarii/restarcie wszystkie klienty jednocześnie się reconnectują / cache pada naraz
  → lawina na backend/DB. Łagodzenie: jitter, exponential backoff, cache stampede protection, rate limiting.
- **Sesje TLS / koszt** — brak terminacji TLS na LB przerzuca deszyfrowanie na każdą instancję.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Bezstanowe API w Spring Boot 3 / Java 21, wdrożone jako **Deployment** i wystawione przez **Service + Ingress**.
Skalujesz do N replik jedną komendą — LB (Service) rozdziela ruch po wszystkich podach:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: orders-api }
spec:
  replicas: 4                       # ← skalowanie POZIOME: 4 instancje
  selector: { matchLabels: { app: orders-api } }
  template:
    metadata: { labels: { app: orders-api } }
    spec:
      terminationGracePeriodSeconds: 30   # czas na connection draining
      containers:
        - name: app
          image: registry/orders-api:1.4.0
          ports: [ { containerPort: 8080 } ]
          readinessProbe:                 # LB kieruje ruch TYLKO do gotowych
            httpGet: { path: /actuator/health/readiness, port: 8080 }
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
---
apiVersion: v1
kind: Service                        # L4 LB po podach (kube-proxy, round-robin)
metadata: { name: orders-api }
spec:
  selector: { app: orders-api }
  ports: [ { port: 80, targetPort: 8080 } ]
---
apiVersion: networking.k8s.io/v1
kind: Ingress                        # L7: routing po hoście/ścieżce + TLS
metadata: { name: orders-api }
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /orders
            pathType: Prefix
            backend: { service: { name: orders-api, port: { number: 80 } } }
```

```bash
kubectl scale deployment/orders-api --replicas=8   # scale-out w 1s
kubectl autoscale deployment/orders-api --min=4 --max=20 --cpu-percent=70   # HPA
```

**Dlaczego bezstanowość jest konieczna:** żądanie `GET /orders` od Anny raz trafi do poda #2, za chwilę do
poda #5 (round-robin). Gdyby poda #2 trzymał jej sesję w pamięci — pod #5 uznałby ją za niezalogowaną. Dlatego
tożsamość niesie **JWT** ([[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]]) walidowany lokalnie przez każdy pod,
a stan biznesowy leży w bazie/Redisie. Wtedy `replicas` możesz podnosić dowolnie — i o to chodzi.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, po co load balancing (throughput, availability, brak SPOF).
- [ ] Porównam skalowanie poziome vs pionowe i uzasadnię, czemu cloud-native woli poziome.
- [ ] Wytłumaczę, czemu skalowanie poziome wymaga bezstanowości i jak to spełnia JWT.
- [ ] Odróżnię L4 od L7 i podam po co każdy (TLS termination, routing L7).
- [ ] Wymienię algorytmy (round-robin, least connections, IP hash, weighted) i kiedy który.
- [ ] Wskażę, jak to działa w k8s: Service (kube-proxy) + Ingress + HPA + readiness.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co realnie robi `kube-proxy` przy Service ClusterIP? (iptables/IPVS, L4, round-robin/losowo)
- [ ] Czym różni się liveness od readiness probe i po co LB potrzebuje readiness?
- [ ] Dlaczego sticky sessions i Redis session store są gorsze od JWT przy scale-out?
- [ ] Co to connection draining i jak zrobić graceful shutdown w Spring Boot na k8s?
- [ ] Client-side vs server-side LB — czym jest Spring Cloud LoadBalancer (i Ribbon)?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Poziome vs pionowe skalowanie — kompromisy?"
- [ ] „Czemu aplikacja musi być stateless, żeby skalować poziomo?"
- [ ] „L4 vs L7 load balancing — różnica i przykłady?"
- [ ] „Jak Kubernetes rozkłada ruch na pody?" (Service/kube-proxy + Ingress + HPA)
- [ ] „Sesja serwerowa czy JWT dla API pod load balancerem — czemu?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Po co load balancing?;Rozkłada ruch na wiele instancji → większa przepustowość, dostępność i brak single point of failure.
Skalowanie poziome (scale out) to?;Dokładanie kolejnych instancji za load balancerem (zamiast wzmacniania jednej maszyny).
Skalowanie pionowe (scale up) to?;Mocniejsza pojedyncza maszyna (więcej CPU/RAM); prosto, ale ma sufit, bywa drogie i nie usuwa SPOF.
Czemu cloud-native woli skalowanie poziome?;Skaluje liniowo bez limitu, daje HA, rolling updates i elastyczne autoskalowanie na commodity hardware.
Jaki warunek musi spełnić aplikacja, by skalować poziomo?;Musi być bezstanowa (stateless) — żadna instancja nie trzyma sesji w pamięci lokalnie.
Czemu JWT zamiast sesji serwerowej pod load balancerem?;Stan tożsamości niesie klient w podpisanym tokenie, walidowanym niezależnie przez każdą instancję — brak wspólnego session store.
Co to sticky sessions i czemu bywają gorsze?;Przypięcie klienta do jednej instancji; powoduje nierówne obciążenie i utratę sesji przy restarcie/scale-in.
Wada zewnętrznego session store (Redis)?;Dodatkowy hop i zależność (nowy potencjalny SPOF), większa latencja i złożoność; JWT dla API zwykle prostszy.
Load balancer L4 działa na?;Warstwie transportowej (TCP/UDP, IP:port); szybki, nie zna treści HTTP.
Load balancer L7 działa na?;Warstwie aplikacyjnej (HTTP) — routing po ścieżce/hoście/nagłówku, terminacja TLS; np. Nginx, HAProxy, Ingress.
Algorytm round-robin?;Kieruje żądania kolejno do każdej instancji po kolei.
Algorytm least connections?;Kieruje do instancji z najmniejszą liczbą aktywnych połączeń — dobre przy nierównych żądaniach.
Po co health checks w LB?;By kierować ruch tylko do zdrowych instancji; w Spring Boot to readiness probe z Actuatora.
Liveness vs readiness probe?;Liveness = czy żyje (jeśli nie, restart); readiness = czy gotowa na ruch (jeśli nie, wypnij z LB bez restartu).
Jak Kubernetes rozkłada ruch na pody?;Service (ClusterIP) load-balansuje L4 przez kube-proxy po gotowych podach; Ingress dodaje routing L7.
Co robi HPA?;HorizontalPodAutoscaler automatycznie zmienia liczbę replik Deploymentu wg metryk (CPU/pamięć/custom) — skalowanie poziome.
Client-side vs server-side load balancing?;Server-side: LB wybiera instancję (Nginx/Service). Client-side: klient sam zna listę i wybiera (Spring Cloud LoadBalancer, dawniej Ribbon).
Co to connection draining?;Przy wdrożeniu/scale-in: przestań kierować nowy ruch (readiness=false), dokończ trwające żądania, potem zabij pod (graceful shutdown).
Co to thundering herd?;Nagły zsynchronizowany napływ (reconnect/cache pada naraz) przeciążający backend; łagodzą jitter, backoff i rate limiting.
Czemu terminacja TLS na L7 LB się opłaca?;Deszyfrujesz HTTPS raz na wejściu zamiast na każdej instancji — mniej CPU i prostsze certyfikaty.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/10-cloud-native/12-factor]] · [[wiedza/10-cloud-native/kubernetes]] · [[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]] · [[wiedza/05-spring/actuator]]
