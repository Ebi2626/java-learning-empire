---
temat: "OWASP Top 10 i bezpieczeństwo aplikacji webowej"
faza: 8
status: nieopanowany
priorytet: 🔴
tags: [java, bezpieczenstwo, owasp]
powiazane: ["[[wiedza/08-bezpieczenstwo/spring-security]]", "[[wiedza/09-architektura/multitenancy]]", "[[wiedza/06-persystencja/jdbc-hikari]]", "[[wiedza/05-spring/actuator]]", "[[wiedza/04-narzedzia/logowanie]]"]
---

# OWASP Top 10 i bezpieczeństwo aplikacji webowej

> **TL;DR:** **OWASP** to fundacja non-profit, a **OWASP Top 10** to ranking 10 najczęstszych/najgroźniejszych
> kategorii podatności aplikacji webowych (aktualizowany co kilka lat, edycja 2021). Dla backendu Java/Spring
> najważniejsze to **Broken Access Control** (m.in. IDOR — pilnuj `@PreAuthorize` i właściciela zasobu, zwłaszcza
> w multitenancy), **Injection** (SQL — zawsze `PreparedStatement`/parametryzacja JPA, nigdy konkatenacja; XSS —
> escaping), **Cryptographic Failures** (TLS, hasła przez BCrypt/Argon2 z solą) i **Security Misconfiguration**
> (odsłonięty actuator, stacktrace w odpowiedzi, domyślne hasła). Sekrety trzymaj poza repo (env/Vault/k8s Secrets),
> zależności skanuj (Dependency-Check/Snyk/Dependabot).

## 1. Co — definicja i API

**OWASP** (*Open Worldwide Application Security Project*) — fundacja non-profit publikująca darmowe standardy,
narzędzia i dokumenty dot. bezpieczeństwa aplikacji (m.in. **ASVS** — Application Security Verification Standard,
**Cheat Sheet Series**, narzędzie **ZAP**, **Dependency-Check**).

**OWASP Top 10** — flagowy dokument: lista 10 najczęstszych **kategorii** ryzyka webowego, uszeregowana wg
częstości i wpływu, oparta na danych z realnych aplikacji. To **kategorie**, nie pojedyncze bugi — punkt startowy
świadomości, nie kompletna lista. Wydanie **2021** (kolejne w toku):

1. **A01 Broken Access Control** — użytkownik robi coś, do czego nie ma uprawnień (IDOR, brak sprawdzenia właściciela,
   modyfikacja `role` w żądaniu). *Nr 1 na liście* — najczęstsza kategoria.
2. **A02 Cryptographic Failures** — brak/zły TLS, słabe hashowanie haseł, sekrety plaintext, słabe algorytmy.
3. **A03 Injection** — SQL/NoSQL/OS/LDAP injection oraz **XSS** (dane użytkownika interpretowane jako kod/markup).
4. **A04 Insecure Design** — błąd na poziomie *projektu*, nie implementacji (brak threat modelingu, brak limitów).
5. **A05 Security Misconfiguration** — domyślne hasła, nadmierne uprawnienia, odsłonięty `actuator`, stacktrace,
   niewyłączone funkcje debug.
6. **A06 Vulnerable and Outdated Components** — biblioteki z CVE (np. **Log4Shell**).
7. **A07 Identification and Authentication Failures** — słabe hasła, brak rate-limiting, session fixation, brak MFA.
8. **A08 Software and Data Integrity Failures** — niezaufana **deserializacja**, brak weryfikacji integralności
   artefaktów/aktualizacji (CI/CD supply chain).
9. **A09 Security Logging and Monitoring Failures** — brak logów zdarzeń bezpieczeństwa → ataki niewidoczne.
10. **A10 Server-Side Request Forgery (SSRF)** — serwer zmuszony do wysłania żądania w miejsce wskazane przez atakującego.

> **CSRF** wypadł z Top 10 (frameworki chronią domyślnie), ale dalej istotny — patrz sekcja 3.

## 2. Jak — mechanika kluczowych ataków (injection, XSS, IDOR)

### SQL Injection
Powstaje, gdy **dane wejściowe trafiają do zapytania jako kod SQL**, nie jako parametr. Klasyk — konkatenacja:

```java
// PODATNE — string budowany z danych użytkownika
String sql = "SELECT * FROM users WHERE email = '" + email + "'";
// email = "x' OR '1'='1"  → zwraca wszystkich; "'; DROP TABLE users; --" → katastrofa
```

Pod spodem: parser DB dostaje jeden string i **nie odróżnia** intencji dewelopera od wstrzykniętego SQL.
**Obrona — parametryzacja** (`?`): sterownik wysyła *strukturę* zapytania i *wartości osobno*, więc dane nigdy
nie są parsowane jako SQL:

```java
// PreparedStatement — zob. [[wiedza/06-persystencja/jdbc-hikari]]
var ps = conn.prepareStatement("SELECT * FROM users WHERE email = ?");
ps.setString(1, email);          // wartość, nie fragment SQL
```

W JPA/Spring Data: **named/positional parameters**, nigdy konkatenacja w `@Query`:

```java
@Query("select u from User u where u.email = :email")   // OK
User findByEmail(@Param("email") String email);
// derived query findByEmail(String) też jest bezpieczne
```

Uwaga: `@Query` z SpEL / `nativeQuery` sklejany ręcznie = ponownie ryzyko. Nazwy kolumn/tabel nie da się
zparametryzować — trzeba whitelisty.

### XSS (Cross-Site Scripting)
Atakujący wstrzykuje `<script>` do treści, którą **przeglądarka innego użytkownika** wyrenderuje jako HTML/JS.
Skutek: kradzież sesji/tokenu, akcje w imieniu ofiary. Typy: **stored** (zapisany w DB), **reflected**
(odbity z parametru), **DOM-based**.

Mechanika obrony — **kontekstowy escaping**: przy wstawianiu danych do HTML zamień `<` → `&lt;` itd.
- **API zwracające JSON** — mniejszy problem: `Content-Type: application/json` nie jest renderowany jako HTML,
  a serializer (Jackson) sam escape'uje. Klucz: **poprawny Content-Type**, brak zwracania user-inputu jako `text/html`.
- **Frontend Angular** — domyślnie traktuje interpolację `{{ }}` jako tekst i escape'uje; niebezpieczne jest
  dopiero `[innerHTML]`/`bypassSecurityTrust*`. Czyli XSS to głównie problem warstwy widoku.

### IDOR / Broken Access Control
**IDOR** (*Insecure Direct Object Reference*): endpoint `GET /api/orders/42` zwraca zamówienie, a kod
**sprawdza tylko, że jesteś zalogowany**, nie czy `42` należy do Ciebie. Atakujący iteruje `41, 43, ...`
i czyta cudze dane.

```java
// PODATNE — brak sprawdzenia właściciela
@GetMapping("/orders/{id}")
public Order get(@PathVariable Long id) {
    return repo.findById(id).orElseThrow();   // czyj to order? nieważne...
}
```

Pod spodem chodzi o **authorization per obiekt** (nie tylko per rola/endpoint). Obrona: sprawdź właściciela.

```java
@PreAuthorize("hasRole('USER')")
@GetMapping("/orders/{id}")
public Order get(@PathVariable Long id, @AuthenticationPrincipal UserDetails me) {
    Order o = repo.findById(id).orElseThrow(NotFound::new);
    if (!o.getOwnerId().equals(userId(me))) throw new AccessDeniedException("nie Twój zasób");
    return o;
    // lepiej: repo.findByIdAndOwnerId(id, userId) — 404 zamiast 403 (nie zdradzaj istnienia)
}
```

W **multitenancy** ([[wiedza/09-architektura/multitenancy]]) to *krytyczne*: każde zapytanie musi być
przefiltrowane po `tenant_id`, inaczej tenant A widzi dane tenanta B. Wymuszaj izolację centralnie
(Hibernate filter / `@Where` / RLS w DB / interceptor), a nie „ręcznie w każdym query".

## 3. Dlaczego / kiedy — obrona w Spring, checklista

Krótko po kategorii — **czym jest + jak bronić w Spring Boot 3 / Java 21**:

- **A01 Broken Access Control** → `@EnableMethodSecurity`, `@PreAuthorize`, sprawdzanie właściciela zasobu
  (filtry po `tenant_id`/`ownerId`). *Deny by default* w `SecurityFilterChain`. Nie ufaj polom z requestu
  (`role`, `userId`) — bierz tożsamość z `SecurityContext`. Zob. [[wiedza/08-bezpieczenstwo/spring-security]].
- **A02 Cryptographic Failures** → wymuś **HTTPS/TLS** (`requiresChannel().anyRequest().requiresSecure()`,
  za proxy: `server.forward-headers-strategy`). Hasła: `BCryptPasswordEncoder` lub **Argon2** (adaptacyjne,
  z **solą** wbudowaną w hash) — nigdy MD5/SHA-1 „na goło". **Nie trzymaj sekretów w kodzie ani w payloadzie JWT**
  (JWT jest tylko *podpisany*, nie szyfrowany — każdy odczyta `payload`).
- **A03 Injection** → parametryzacja (wyżej). XSS: poprawny `Content-Type`, escaping w widoku, unikać
  `bypassSecurityTrust*` w Angularze. Walidacja wejścia (`@Valid` + Bean Validation) jako obrona w głąb.
- **A04 Insecure Design** → **threat modeling** przed kodem, limity biznesowe (max kwota, rate limity),
  bezpieczne defaulty. Tego nie „załata się" biblioteką.
- **A05 Security Misconfiguration** → **actuator**: wystaw tylko `health`/`info`, resztę za auth
  (`management.endpoints.web.exposure.include`), najlepiej osobny port/sieć — [[wiedza/05-spring/actuator]].
  Wyłącz stacktrace w odpowiedzi (`server.error.include-stacktrace=never`), globalny `@ControllerAdvice`
  zwracający generyczne błędy. Usuń domyślne konta/hasła.
- **A06 Vulnerable/Outdated Components** → skanuj zależności: **OWASP Dependency-Check**, **Snyk**,
  **Dependabot**/Renovate w CI. **Log4Shell** (CVE-2021-44228) to sztandarowy przykład — RCE przez podatną
  wersję biblioteki logującej. Aktualizuj Spring Boot BOM.
- **A07 Auth Failures** → polityka silnych haseł, **rate-limiting** / lockout na logowaniu (Bucket4j / Redis),
  MFA, `changeSessionId()` po logowaniu (obrona przed **session fixation** — Spring Security robi to domyślnie),
  krótki TTL tokenów + refresh.
- **A08 Integrity Failures** → **nie deserializuj niezaufanych danych** natywnym Java serialization
  (`ObjectInputStream` → RCE przez gadget chains). Preferuj JSON z jawnym typowaniem; wyłącz
  Jackson `enableDefaultTyping`/polymorphic bez whitelisty. Weryfikuj integralność artefaktów w CI/CD.
- **A09 Logging/Monitoring** → **loguj zdarzenia bezpieczeństwa** (logowania nieudane, odmowy dostępu,
  zmiany uprawnień), ale **nigdy nie loguj sekretów/haseł/tokenów/PII** — [[wiedza/04-narzedzia/logowanie]].
  Correlation ID, alerty na anomalie.
- **A10 SSRF** → waliduj/whitelistuj URL-e, do których serwer sam wykonuje request (webhooki, pobieranie obrazków);
  blokuj adresy wewnętrzne (`169.254.169.254` — metadata cloud, `localhost`, RFC1918).

### CSRF — kiedy dotyczy
- **Sesja + cookie** (stateful) → **CSRF groźne**: przeglądarka automatycznie dołącza cookie do żądania z obcej
  strony. Włącz ochronę CSRF Spring Security (domyślnie ON dla non-GET).
- **Stateless JWT w nagłówku `Authorization`** → CSRF praktycznie **nie dotyczy** (token nie jest wysyłany
  automatycznie przez przeglądarkę). Wtedy zwykle `csrf().disable()`. Uwaga: jeśli JWT trzymasz w cookie — wraca ryzyko.

### Zarządzanie sekretami
Nigdy w repo/`application.yml` commitowanym do gita. Kolejno: **zmienne środowiskowe** → **HashiCorp Vault** /
**k8s Secrets** (najlepiej z szyfrowaniem at-rest + Sealed Secrets/External Secrets) → Spring Cloud Config z
szyfrowaniem. Rotacja i zasada least privilege.

### Nagłówki bezpieczeństwa (Spring Security ustawia część domyślnie)
- **HSTS** (`Strict-Transport-Security`) — wymuś HTTPS po stronie przeglądarki.
- **CSP** (`Content-Security-Policy`) — ogranicz źródła skryptów → obrona w głąb przed XSS (nie domyślne, ustaw ręcznie).
- **X-Content-Type-Options: nosniff** — zablokuj MIME sniffing.
- **X-Frame-Options / frame-ancestors** — anty-clickjacking.

### Praktyczna checklista bezpieczeństwa API
- [ ] HTTPS wymuszony end-to-end, HSTS włączony.
- [ ] `SecurityFilterChain` = deny by default; każdy endpoint świadomie autoryzowany.
- [ ] Autoryzacja **per obiekt** (właściciel/tenant), nie tylko „zalogowany".
- [ ] Wszystkie query parametryzowane; brak konkatenacji SQL.
- [ ] Walidacja wejścia (`@Valid`), rozmiary/limity, whitelisty.
- [ ] Hasła: BCrypt/Argon2 z solą; sekrety poza repo (env/Vault/k8s).
- [ ] Rate-limiting na logowaniu i wrażliwych endpointach.
- [ ] Actuator ograniczony i zabezpieczony; brak stacktrace/wersji w błędach.
- [ ] Skan zależności w CI (Dependency-Check/Snyk/Dependabot); aktualny Boot BOM.
- [ ] Nagłówki: CSP, X-Content-Type-Options, X-Frame-Options.
- [ ] CSRF: włączony dla cookie/sesji, świadomie wyłączony dla stateless JWT.
- [ ] Logi zdarzeń bezpieczeństwa; brak sekretów/PII w logach.
- [ ] Brak natywnej deserializacji niezaufanych danych.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)
Multitenantowe API SaaS (Spring Boot 3, Java 21). Endpoint `GET /api/v1/invoices/{id}`: bez sprawdzenia
`tenant_id` konsultant firmy A pobiera faktury firmy B (IDOR + broken tenant isolation — A01). W code review
łapiesz też `@Query` z konkatenacją filtra (A03), odsłonięty `/actuator/env` zdradzający `spring.datasource.password`
(A05 + A02), oraz `log.info("login " + rawPassword)` (A09). Fix: filtr po tenancie w repozytorium,
parametryzacja, actuator za auth + `include-stacktrace=never`, usunięcie sekretu z logu i przeniesienie do Vault.
Do CI dokładasz OWASP Dependency-Check, który przy okazji flaguje starą wersję biblioteki z CVE (A06).

---

## ✅ Kryteria opanowania
- [ ] Wymienię, czym jest OWASP i OWASP Top 10, i że to *kategorie*, nie pojedyncze bugi.
- [ ] Wyjaśnię mechanikę SQL injection, XSS i IDOR — i dokładnie, jak każdemu zapobiec w Spring.
- [ ] Wiem, czemu parametryzacja (`?`) zapobiega injection na poziomie parsera DB.
- [ ] Rozróżnię, kiedy CSRF jest groźne (cookie/sesja) a kiedy nie (stateless JWT w nagłówku).
- [ ] Wskażę, gdzie i jak trzymać sekrety oraz jak skanować zależności.
- [ ] Odtworzę z głowy checklistę bezpieczeństwa API.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Dlaczego `PreparedStatement` jest bezpieczny, a konkatenacja nie? (struktura vs wartości, parser DB)
- [ ] Czemu autoryzacja per obiekt jest osobnym problemem niż per rola? (IDOR)
- [ ] Dlaczego JWT nie chroni sekretów w payloadzie? (podpis ≠ szyfrowanie)
- [ ] Czemu API-JSON jest mniej podatne na XSS niż widok HTML?
- [ ] Jak izoluje się tenantów, żeby jeden nie widział danych drugiego?
- [ ] Czemu natywna deserializacja Javy bywa RCE?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Wymień kilka pozycji z OWASP Top 10 i jak się przed nimi bronisz w Spring."
- [ ] „Jak zapobiegasz SQL injection?" (parametryzacja, JPA params, nigdy konkatenacja)
- [ ] „Co to IDOR i jak go łapiesz?" (sprawdzenie właściciela/tenanta, `@PreAuthorize`)
- [ ] „Jak przechowujesz hasła i sekrety?" (BCrypt/Argon2 + sól; env/Vault/k8s Secrets)
- [ ] „Kiedy potrzebujesz ochrony CSRF, a kiedy nie?"
- [ ] „Co poszło źle w Log4Shell i jak się przed tym chronić?" (A06, skan zależności)

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/08-bezpieczenstwo.csv`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Czym jest OWASP?;Fundacja non-profit publikująca darmowe standardy i narzędzia bezpieczeństwa aplikacji (Top 10, ASVS, ZAP, Dependency-Check).
Czym jest OWASP Top 10?;Rankingiem 10 najczęstszych KATEGORII podatności aplikacji webowych (edycja 2021).
Która kategoria jest nr 1 w OWASP Top 10 2021?;Broken Access Control (A01).
Co to IDOR?;Insecure Direct Object Reference — dostęp do cudzego zasobu przez podmianę ID, bo brak sprawdzenia właściciela.
Jak bronić się przed IDOR w Spring?;@PreAuthorize + sprawdzenie właściciela/tenanta zasobu (np. findByIdAndOwnerId), nie tylko "zalogowany".
Dlaczego IDOR jest krytyczny w multitenancy?;Bez filtra po tenant_id jeden tenant widzi dane drugiego — izolację wymuszaj centralnie (Hibernate filter / RLS).
Jak zapobiegać SQL injection?;Parametryzacja: PreparedStatement (?) lub named params w JPA; NIGDY konkatenacja stringów.
Dlaczego parametryzacja chroni przed injection?;Sterownik wysyła strukturę zapytania i wartości osobno — dane nigdy nie są parsowane jako SQL.
Co to XSS?;Wstrzyknięcie skryptu w treść renderowaną w przeglądarce innego użytkownika (stored/reflected/DOM).
Czemu API zwracające JSON jest mniej podatne na XSS?;Content-Type application/json nie jest renderowany jako HTML, a Jackson escape'uje; kluczowy jest poprawny Content-Type.
Jak przechowywać hasła użytkowników?;Hash adaptacyjny z solą: BCrypt lub Argon2 — nigdy MD5/SHA-1 na goło.
Dlaczego nie trzymać sekretów w payloadzie JWT?;JWT jest tylko podpisany, nie szyfrowany — payload może odczytać każdy.
Co to Security Misconfiguration (A05)?;Domyślne hasła, nadmierne uprawnienia, odsłonięty actuator, stacktrace w odpowiedzi.
Jak zabezpieczyć actuator w Spring Boot?;Wystawić tylko health/info, resztę za auth (exposure.include), najlepiej osobny port/sieć.
Czym był Log4Shell?;CVE-2021-44228 — RCE przez podatną wersję Log4j; przykład A06 (vulnerable components).
Jak skanować zależności pod kątem CVE?;OWASP Dependency-Check, Snyk, Dependabot/Renovate w CI + aktualny Spring Boot BOM.
Kiedy CSRF jest groźne?;Gdy uwierzytelniasz sesją+cookie (przeglądarka dołącza cookie automatycznie); przy stateless JWT w nagłówku zwykle nie.
Co to session fixation i jak bronić?;Atakujący narzuca ID sesji przed logowaniem; obrona: zmiana ID sesji po logowaniu (Spring Security domyślnie).
Czemu niebezpieczna jest deserializacja niezaufanych danych?;Java native deserialization pozwala na RCE przez gadget chains — nie deserializuj niezaufanego wejścia.
Co to SSRF?;Server-Side Request Forgery — serwer wykonuje request pod adres wskazany przez atakującego; blokuj adresy wewnętrzne i metadata cloud.
Gdzie trzymać sekrety zamiast repo?;Zmienne środowiskowe, HashiCorp Vault, k8s Secrets — nigdy w gicie; z rotacją i least privilege.
Wymień 3 nagłówki bezpieczeństwa.;HSTS (wymuś HTTPS), CSP (ogranicz źródła skryptów), X-Content-Type-Options: nosniff.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/08-bezpieczenstwo/spring-security]] · [[wiedza/09-architektura/multitenancy]] · [[wiedza/06-persystencja/jdbc-hikari]] · [[wiedza/05-spring/actuator]] · [[wiedza/04-narzedzia/logowanie]]
