---
temat: "Keycloak — Identity and Access Management (IdP)"
faza: 8
status: nieopanowany
priorytet: 🟡
tags: [java, bezpieczenstwo, keycloak, iam]
powiazane: ["[[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]]", "[[wiedza/09-architektura/multitenancy]]", "[[wiedza/10-cloud-native/konteneryzacja]]", "[[projekt-rekrutacja/docs/ARCHITEKTURA]]"]
---

# Keycloak — Identity and Access Management (IdP)

> **TL;DR:** **Keycloak** to open-source **IAM/Identity Provider** od Red Hat, który wdraża
> **OAuth2 / OIDC / SAML** i daje gotowe: uwierzytelnianie, **SSO**, MFA, social login, federację
> i zarządzanie użytkownikami. Nie piszesz auth sam — twój Spring Boot działa jako **OAuth2 Resource Server**,
> waliduje **JWT** publicznym kluczem z **JWKS** Keycloaka i mapuje `realm_access.roles` na `GrantedAuthority`.
> W multitenancy modelujesz tenanty jako **realm-per-tenant** (twarda izolacja) lub **jeden realm + grupy/role**.

## 1. Co — definicja i API

**Keycloak** = gotowy serwer **IdP (Identity Provider)** i **Authorization Server**. Zamiast implementować
logowanie, hasła, resetowanie, sesje SSO, tokeny i integrację z Google/GitHub — delegujesz to do Keycloaka,
a aplikacja tylko **konsumuje tokeny**. Keycloak jest kompletną implementacją standardów:

- **OAuth2** — framework autoryzacji (wydawanie access/refresh tokenów, flow: Authorization Code + PKCE, Client Credentials…).
- **OIDC (OpenID Connect)** — warstwa **uwierzytelniania** nad OAuth2 (kim jest user → `id_token`, `userinfo`).
- **SAML 2.0** — dla starszych enterprise SSO (federacja z korporacyjnymi IdP).

Szczegóły protokołów: [[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]].

### Po co gotowy IdP zamiast własnego auth
- **SSO** — jedno logowanie do wielu aplikacji (wspólna sesja w realmie).
- **Federacja** — LDAP/AD, brokering do zewnętrznych IdP (Google, Azure AD, GitHub — „social login").
- **MFA/OTP**, polityki haseł, brute-force detection, wygaszanie sesji — z pudełka.
- **Gotowe, przetestowane flow** OAuth2/OIDC — auth to obszar, gdzie **własny kod = luki bezpieczeństwa**.
  Reguła: *nie wymyślaj kryptografii ani auth sam*.

### Model pojęciowy (kluczowe encje)
- **REALM** — **izolowana przestrzeń** użytkowników, klientów, ról, grup i konfiguracji. Users z realmu A
  nie widzą realmu B; każdy realm ma **własny issuer i własne klucze podpisu**. To jednostka izolacji
  → **kluczowe dla MULTITENANCY** ([[wiedza/09-architektura/multitenancy]]). (`master` = realm administracyjny — nie trzymaj tam userów apki.)
- **CLIENT** — reprezentacja aplikacji/usługi w realmie:
  - **confidential** — ma sekret, potrafi go bezpiecznie przechować (backend, Spring Boot) → Authorization Code / Client Credentials.
  - **public** — bez sekretu (SPA, mobile) → wyłącznie Authorization Code **+ PKCE**.
  - konfiguracja: **redirect URIs** (whitelist adresów powrotu — ochrona przed open redirect), web origins (CORS), access type.
- **USERS** — konta w realmie (lokalne lub sfederowane).
- **ROLES** — uprawnienia:
  - **realm roles** — globalne w obrębie realmu (`admin`, `user`).
  - **client roles** — zawężone do konkretnego clienta (`account:manage-account`).
- **GROUPS** — hierarchiczne zbiory userów; do grupy przypina się role → user dziedziczy je z grup.
- **client scopes / scopes** — zestawy claimów/ról doklejanych do tokenu (np. scope `profile`, `email`).

## 2. Jak — realm, OIDC, walidacja JWT pod spodem

### Endpointy OIDC (discovery)
Keycloak publikuje **well-known discovery document** per realm:
```
GET  {base}/realms/{realm}/.well-known/openid-configuration
```
zawiera adresy pozostałych endpointów:
- **authorization_endpoint** (`/protocol/openid-connect/auth`) — start flow, przekierowanie usera do logowania.
- **token_endpoint** (`/protocol/openid-connect/token`) — wymiana `code`→tokeny lub Client Credentials.
- **userinfo_endpoint** (`/protocol/openid-connect/userinfo`) — dane profilu po access tokenie.
- **jwks_uri** (`/protocol/openid-connect/certs`) — **JWKS**: publiczne klucze (RSA/EC) do **weryfikacji podpisu JWT**.
- **issuer** — identyfikator wydawcy, np. `https://auth.example.com/realms/acme`.

### Jak Resource Server naprawdę waliduje token (black-box zabity)
Access token Keycloaka to **JWT** podpisany kluczem prywatnym realmu (`RS256`). Resource Server **nie pyta
Keycloaka o każdy request** — waliduje lokalnie:
1. Startowo (lub przy nieznanym `kid`) pobiera **JWKS** z `jwks_uri` i cache'uje klucze publiczne.
2. Dla każdego żądania: parsuje JWT, po nagłówku `kid` wybiera klucz, **weryfikuje podpis**.
3. Sprawdza `iss` (musi = skonfigurowany `issuer-uri`), `exp` (ważność), `aud`/`azp`, `nbf`.
4. Buduje `Authentication`; role są w claimach `realm_access.roles` i `resource_access.{client}.roles`.

Podpis asymetryczny = Resource Server zna tylko **klucz publiczny**, nie sekret → skalowalne, offline, bez round-tripu.

### Mappers i token exchange
- **Protocol mappers** — konfigurowalne reguły **wstrzykiwania claimów** do tokenu: dodaj atrybut usera,
  role, `groups`, custom claim (np. `tenant_id`). Bez mappera claim nie trafi do JWT — częsta przyczyna „brak roli".
- **Token exchange** — wymiana jednego tokenu na inny (impersonacja, przejście między klientami/audiencjami,
  service-to-service). Wymaga włączenia i uprawnień; używać ostrożnie.

### Integracja ze Spring Boot 3 (Java 21)
Dwie role aplikacji (często obie):
- **OAuth2 Resource Server** — backend API chroniony JWT (najczęstszy przypadek). Minimalna konfiguracja to
  **jedynie `issuer-uri`** — Spring sam pobierze discovery + JWKS:
  ```yaml
  spring:
    security:
      oauth2:
        resourceserver:
          jwt:
            issuer-uri: https://auth.example.com/realms/acme
  ```
- **OAuth2 Client / Login** — gdy sama apka loguje usera (server-side rendering, BFF) i przechowuje sesję.

Kluczowy krok integracji: **przemapować role Keycloaka na `GrantedAuthority`**, bo domyślny konwerter
czyta `scope`/`scp`, a role Keycloaka siedzą w `realm_access.roles` (patrz sekcja *Przykład w praktyce*).

### Uruchomienie (kontener)
Keycloak dostarcza się jako obraz OCI ([[wiedza/10-cloud-native/konteneryzacja]]):
```bash
docker run -p 8080:8080 \
  -e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
  -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:24.0 start-dev
```
`start-dev` tylko do developmentu (H2, HTTP). Produkcja: `start` + zewnętrzny **Postgres**, HTTPS/reverse-proxy,
`KC_HOSTNAME`, realmy importowane z JSON (`--import-realm`) lub provisionowane przez Admin REST API.

## 3. Dlaczego / kiedy — multitenancy, provisioning, pułapki

### Multitenancy: realm-per-tenant vs jeden realm
Sedno projektu rekrutacyjnego → [[wiedza/09-architektura/multitenancy]].

| Kryterium | **Realm per tenant** | **Jeden realm + grupy/role** |
|---|---|---|
| Izolacja userów/kluczy | **Twarda** (osobny issuer, klucze, konfiguracja) | Miękka (logiczna, w danych) |
| Custom branding / IdP per tenant | Łatwo (osobne ustawienia, login themes, brokering) | Trudno / niemożliwe |
| Provisioning nowego tenanta | Utwórz realm (cięższe) | Dodaj grupę + `tenant_id` (lekkie) |
| Skalowalność (setki/tysiące) | Kosztowna — każdy realm to narzut, wolniejszy start | **Skalowalna** — jeden realm |
| Cross-tenant admin / raportowanie | Trudniejsze (N issuerów) | Łatwiejsze |
| Ryzyko wycieku między tenantami | Minimalne | Zależy od dyscypliny w kodzie/claimach |

**Reguła kciuka:** mało tenantów, wymagana silna izolacja / osobne IdP → **realm per tenant**;
dużo tenantów, wspólna apka → **jeden realm**, tenant jako **grupa** i claim **`tenant_id`** w tokenie
(dodany mapperem), po którym filtrujesz dane. Resource Server przy realm-per-tenant musi rozpoznać issuera
per request (np. `JwtIssuerAuthenticationManagerResolver` — multi-issuer resolver).

### Provisioning tenantów programowo
- **Admin Console** — GUI do ręcznej administracji.
- **Admin REST API** (`/admin/realms/...`) — provisioning **z kodu**: tworzenie realmów/clientów/userów/ról
  przy onboardingu tenanta. Uwierzytelniasz się jako service account (Client Credentials) z rolami admina.
  Automatyzacja onboardingu = tworzysz realm lub grupę + rolę jednym wywołaniem z backendu.

### Pułapki (rekrutacyjne must-know)
- **Mapowanie ról** — bez custom `JwtAuthenticationConverter` `@PreAuthorize("hasRole('ADMIN')")` nie działa,
  bo role są w `realm_access.roles`, a nie w domyślnym `scope`. Pamiętaj o prefiksie `ROLE_`.
- **Issuer za reverse-proxy** — `iss` w tokenie musi **dokładnie** zgadzać się z `issuer-uri` w apce.
  Za proxy Keycloak potrafi wystawić issuer z wewnętrznym hostem/portem → walidacja pada. Ustaw `KC_HOSTNAME`
  i przekazuj `X-Forwarded-*` / `KC_PROXY_HEADERS`, żeby issuer był publicznym URL-em.
- **Czas życia tokenu** — access token krótki (minuty). Za długi → trudno odwołać (JWT ważny do `exp`).
  Za krótki → częste odświeżanie. Refresh token dłuższy, ale rotowany. Odwołanie natychmiast = introspection/blacklist.
- **`realm` vs `client` roles** — łatwo pomylić źródło; klienckie są w `resource_access.{client}.roles`.
- **Brak mappera** → oczekiwany claim (`groups`, `tenant_id`) nie pojawia się w tokenie.
- **Cache JWKS a rotacja kluczy** — po rotacji kluczy realmu Resource Server musi odświeżyć JWKS (nowy `kid`).

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

**Spring Boot 3 jako Resource Server + konwerter ról Keycloaka:**
```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

  @Bean
  SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
      .authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/admin/**").hasRole("ADMIN")
        .anyRequest().authenticated())
      .oauth2ResourceServer(oauth -> oauth
        .jwt(jwt -> jwt.jwtAuthenticationConverter(keycloakConverter())));
    return http.build();
  }

  // Wyciąga realm_access.roles z JWT i mapuje na ROLE_* (GrantedAuthority)
  private JwtAuthenticationConverter keycloakConverter() {
    JwtAuthenticationConverter conv = new JwtAuthenticationConverter();
    conv.setJwtGrantedAuthoritiesConverter(jwt -> {
      Map<String, Object> realmAccess = jwt.getClaim("realm_access");
      if (realmAccess == null || realmAccess.get("roles") == null) {
        return List.of();
      }
      @SuppressWarnings("unchecked")
      Collection<String> roles = (Collection<String>) realmAccess.get("roles");
      return roles.stream()
          .map(r -> new SimpleGrantedAuthority("ROLE_" + r))
          .collect(Collectors.toList());
    });
    return conv;
  }
}
```
```yaml
# application.yml — reszta (JWKS, walidacja podpisu/iss/exp) dzieje się automatycznie
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com/realms/acme
```

**Szkic realm-per-tenant (multi-issuer resolver):**
```java
// Każdy tenant = osobny realm/issuer; resolver dobiera walidator per token
@Bean
AuthenticationManagerResolver<HttpServletRequest> tenantResolver() {
  return new JwtIssuerAuthenticationManagerResolver(
      "https://auth.example.com/realms/acme",
      "https://auth.example.com/realms/globex"); // zaufana lista issuerów
}
// http.oauth2ResourceServer(o -> o.authenticationManagerResolver(tenantResolver()));
```
Onboarding tenanta: backend woła **Admin REST API** → tworzy nowy realm (lub grupę + rolę w jednym realmie)
i dopisuje go do listy zaufanych issuerów / konfiguracji. Kontekst projektowy: [[projekt-rekrutacja/docs/ARCHITEKTURA]].

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, czym jest Keycloak i **po co gotowy IdP** zamiast własnego auth.
- [ ] Rozróżnię REALM / CLIENT (confidential vs public) / USER / ROLE (realm vs client) / GROUP / scope.
- [ ] Opiszę endpointy OIDC i **jak Resource Server waliduje JWT** przez JWKS (bez round-tripu do Keycloaka).
- [ ] Skonfiguruję Spring Boot jako Resource Server i napiszę konwerter mapujący `realm_access.roles`.
- [ ] Uzasadnię **realm-per-tenant vs jeden realm** i wskażę kompromisy dla multitenancy.
- [ ] Wymienię pułapki: issuer za proxy, mapowanie ról, TTL tokenu, brak mappera.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Jak Resource Server sprawdza, że JWT jest autentyczny, **nie pytając** Keycloaka o każdy request? (JWKS + podpis RS256)
- [ ] Skąd apka zna adresy token/authorize/JWKS? (well-known/openid-configuration = discovery)
- [ ] Gdzie w tokenie siedzą role Keycloaka i czemu domyślnie nie działa `hasRole(...)`? (`realm_access.roles`, brak prefiksu `ROLE_`)
- [ ] Czemu za reverse-proxy walidacja issuera pada i jak to naprawić? (`iss` ≠ `issuer-uri`, `KC_HOSTNAME`/forwarded headers)
- [ ] Jak claim `tenant_id`/`groups` trafia do tokenu? (protocol mapper)

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Czym jest Keycloak i dlaczego nie pisać auth samemu?"
- [ ] „Jak zintegrujesz Keycloak ze Spring Boot?" (Resource Server, issuer-uri, JWKS, konwerter ról)
- [ ] „Multitenancy w Keycloak — realm per tenant czy jeden realm? Kompromisy?"
- [ ] „Confidential vs public client — kiedy PKCE?"
- [ ] „Jak zautomatyzujesz provisioning nowego tenanta?" (Admin REST API)
- [ ] „Realm role vs client role — różnica i gdzie w JWT?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/08-bezpieczenstwo.csv`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Czym jest Keycloak?;Open-source IAM/Identity Provider od Red Hat wdrażający OAuth2/OIDC/SAML — uwierzytelnianie, SSO, MFA, federacja, zarządzanie użytkownikami.
Po co gotowy IdP zamiast własnego auth?;Gotowe i przetestowane SSO, MFA, social login, federacja i flow OAuth2 — własny auth = luki bezpieczeństwa.
Co to REALM w Keycloak?;Izolowana przestrzeń użytkowników, klientów, ról i grup z własnym issuerem i kluczami podpisu — jednostka izolacji (kluczowa dla multitenancy).
Confidential vs public client?;Confidential ma sekret (backend) i używa Authorization Code/Client Credentials; public bez sekretu (SPA/mobile) używa wyłącznie Authorization Code + PKCE.
Realm role vs client role?;Realm role jest globalna w realmie (realm_access.roles); client role zawężona do klienta (resource_access.{client}.roles).
Do czego służą redirect URIs w client?;Whitelist dozwolonych adresów powrotu po logowaniu — ochrona przed open redirect / przechwyceniem code.
Co to JWKS i po co?;Endpoint z publicznymi kluczami realmu (jwks_uri) do lokalnej weryfikacji podpisu JWT bez round-tripu do Keycloaka.
Gdzie znajdziesz adresy endpointów OIDC?;W discovery document: /realms/{realm}/.well-known/openid-configuration.
Jak Resource Server waliduje access token?;Lokalnie: weryfikuje podpis JWT kluczem z JWKS (po kid) oraz sprawdza iss, exp, aud/azp — bez zapytania do Keycloaka.
Do czego służą protocol mappers?;Wstrzykują dodatkowe claimy do tokenu (role, atrybuty usera, tenant_id, groups) — bez mappera claim nie trafi do JWT.
Co to token exchange?;Wymiana jednego tokenu na inny (impersonacja, zmiana audiencji/klienta, service-to-service) — wymaga włączenia i uprawnień.
Jak zintegrować Keycloak ze Spring Boot API?;Jako OAuth2 Resource Server: ustawić issuer-uri, Spring pobierze JWKS i zwaliduje JWT automatycznie.
Czemu hasRole('ADMIN') nie działa domyślnie z Keycloak?;Bo role są w claimie realm_access.roles, a domyślny konwerter czyta scope — trzeba custom JwtAuthenticationConverter i prefiks ROLE_.
Realm-per-tenant vs jeden realm w multitenancy?;Realm-per-tenant = twarda izolacja i osobne IdP, ale kosztowny narzut; jeden realm + grupy/tenant_id = skalowalny przy wielu tenantach, izolacja logiczna.
Jak provisionować tenantów programowo?;Przez Admin REST API (/admin/realms/...) — tworzenie realmów/klientów/userów/ról z backendu jako service account.
Najczęstszy błąd konfiguracji issuera za proxy?;iss w tokenie różni się od issuer-uri w apce — trzeba ustawić KC_HOSTNAME i przekazywać X-Forwarded-* / KC_PROXY_HEADERS.
Jak uruchomić Keycloak lokalnie?;Kontener quay.io/keycloak/keycloak z komendą start-dev (tylko dev); produkcja: start + Postgres + HTTPS.
Czym różni się OIDC od OAuth2 w kontekście Keycloak?;OAuth2 to autoryzacja (access token); OIDC dokłada warstwę uwierzytelniania (id_token, userinfo — kim jest user).
Jak obsłużyć wielu issuerów (realm-per-tenant) w Spring?;JwtIssuerAuthenticationManagerResolver z listą zaufanych issuerów — resolver dobiera walidator per token.
Czemu access token robi się krótki?;JWT jest ważny do exp i trudno go odwołać; krótki TTL ogranicza okno nadużycia, a refresh token odnawia dostęp.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]] · [[wiedza/09-architektura/multitenancy]] · [[wiedza/10-cloud-native/konteneryzacja]] · [[projekt-rekrutacja/docs/ARCHITEKTURA]]
