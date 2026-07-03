---
temat: "JWT, OAuth2 i OpenID Connect"
faza: 8
status: nieopanowany
priorytet: 🔴
tags: [java, bezpieczenstwo, jwt, oauth]
powiazane: ["[[wiedza/08-bezpieczenstwo/spring-security]]", "[[wiedza/08-bezpieczenstwo/keycloak]]"]
---

# JWT, OAuth2 i OpenID Connect

> **TL;DR:** **JWT** to podpisany, samodzielnie weryfikowalny token (`header.payload.signature`, base64url) —
> *zakodowany, nie zaszyfrowany*, więc nie wkładasz w niego sekretów. **OAuth2** to protokół **delegacji autoryzacji**
> („zaloguj przez Google" bez oddawania hasła), **OIDC** dodaje na wierzchu **warstwę uwierzytelniania** (tożsamość:
> *ID Token* + endpoint `userinfo`). W Spring Boot 3: `oauth2-resource-server` waliduje JWT z Keycloaka po `issuer-uri`.

## 1. Co — definicje i API

Trzy powiązane, ale różne rzeczy — częsty błąd to mylenie ich:

| Rzecz | Co to jest | Warstwa |
|---|---|---|
| **JWT** | *format* tokenu (podpisany JSON) | struktura danych |
| **OAuth2** | protokół **autoryzacji** (delegacja dostępu) | „co możesz zrobić" |
| **OIDC** | nadbudowa nad OAuth2 dodająca **uwierzytelnianie** | „kim jesteś" |

**JWT (JSON Web Token)** — trzy części oddzielone kropkami, każda kodowana **base64url**:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImFiYyJ9   ← header
.eyJpc3MiOiJodHRwczovL2tjL3JlYWxtcy9hcHAiLCJzdWIiOiIx  ← payload (claims)
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c            ← signature
```

- **Header** — algorytm podpisu (`alg`: `RS256`, `HS256`, `ES256`) i `kid` (key id → wskazuje klucz w JWKS).
- **Payload** — **claimy**. Standardowe (registered): `iss` (issuer — kto wystawił), `sub` (subject — id usera),
  `aud` (audience — dla kogo token), `exp` (expiry, epoch seconds), `iat` (issued at), `nbf` (not before),
  `jti` (token id). W OAuth2 dodatkowo `scope`/`scp`; w Keycloaku role w `realm_access.roles` / `resource_access`.
- **Signature** — podpis `HMAC-SHA256(base64url(header)+"."+base64url(payload), secret)` **albo** RSA/EC
  (klucz prywatny podpisuje, publiczny weryfikuje).

> ⚠️ **JWT jest tylko ZAKODOWANY (base64url), NIE zaszyfrowany.** Każdy może odczytać payload (wklej na jwt.io).
> **Nie wkładaj sekretów** (haseł, PESEL, kluczy). Podpis chroni **integralność**, nie **poufność**.
> (Szyfrowany wariant to JWE — rzadziej używany; „gołe" JWT z podpisem to JWS.)

**OAuth2 — role (aktorzy):**
- **Resource Owner** — user (właściciel danych).
- **Client** — aplikacja chcąca dostępu (Twoje SPA / backend).
- **Authorization Server** — wydaje tokeny (Keycloak, Google, Auth0).
- **Resource Server** — API chronione tokenem (Twój Spring Boot backend).

## 2. Jak — budowa/walidacja JWT, flow OAuth2 pod spodem

### Walidacja JWT (resource server, przy każdym żądaniu — bez odpytywania AS)

To sedno **stateless**: serwer sprawdza token *lokalnie*, kryptograficznie, nie trzymając sesji.

1. **Podpis** — pobierz klucz publiczny z **JWKS** (`.../protocol/openid-connect/certs`) po `kid` z headera,
   zweryfikuj sygnaturę. (Klucze cache'owane i rotowane — dlatego RS256 > HS256 dla wielu serwisów: klucz
   publiczny może być wszędzie, prywatny zostaje w AS.)
2. **`exp`** — token nie wygasł (z małym clock skew).
3. **`nbf`/`iat`** — czasy poprawne.
4. **`iss`** — wystawiony przez zaufany issuer (Twój Keycloak realm).
5. **`aud`** — token przeznaczony dla tego API.
6. Dopiero po tym: mapowanie `scope`/`roles` → uprawnienia (authorities).

### OAuth2 grant types (flows)

- **Authorization Code + PKCE** — **dziś standard** dla SPA/Angular i mobile. PKCE (Proof Key for Code Exchange)
  chroni przed przechwyceniem `code` u klienta publicznego (bez client secret).
- **Client Credentials** — **machine-to-machine** (serwis↔serwis, cron, backend bez usera). Klient uwierzytelnia
  się swoim secretem, dostaje access token.
- **Implicit** i **Resource Owner Password Credentials** — **PRZESTARZAŁE** (deprecated w OAuth 2.1).
  Implicit zwracał token w URL fragmencie (wyciekał); password wymagał podania hasła klientowi.

### Diagram: Authorization Code Flow + PKCE

```
 [User]        [SPA / Angular]         [Authorization Server]      [Resource Server / API]
   │                 │                        (Keycloak)                    │
   │  klik "Login"   │                            │                        │
   │────────────────>│  1. generuj code_verifier  │                        │
   │                 │     code_challenge=S256(v)  │                        │
   │                 │  2. redirect /authorize?    │                        │
   │                 │     response_type=code&      │                        │
   │                 │     code_challenge=...&      │                        │
   │                 │     scope=openid ...        │                        │
   │                 │────────────────────────────>│                        │
   │   ekran logowania (login + hasło u AS, NIE u klienta)                  │
   │<───────────────────────────────────────────── │                        │
   │  login/hasło    │                            │                        │
   │────────────────────────────────────────────> │                        │
   │                 │  3. redirect ...?code=XYZ   │                        │
   │                 │<────────────────────────────│                        │
   │                 │  4. POST /token             │                        │
   │                 │     code=XYZ +              │                        │
   │                 │     code_verifier (dowód!)  │                        │
   │                 │────────────────────────────>│  weryfikuje:           │
   │                 │                             │  SHA256(verifier)==     │
   │                 │                             │  challenge             │
   │                 │  5. { access_token (JWT),   │                        │
   │                 │       id_token (JWT, OIDC), │                        │
   │                 │       refresh_token }       │                        │
   │                 │<────────────────────────────│                        │
   │                 │  6. GET /api  Authorization: Bearer <access_token>   │
   │                 │─────────────────────────────────────────────────────>│
   │                 │                             │   waliduje JWT lokalnie │
   │                 │  7. 200 OK + dane           │   (podpis/exp/iss/aud)  │
   │                 │<─────────────────────────────────────────────────────│
```

### OIDC (OpenID Connect) — warstwa tożsamości

OAuth2 sam w sobie mówi *co* możesz (autoryzacja), ale **nie standaryzuje „kim jesteś"**. OIDC dokłada:
- **ID Token** — JWT z tożsamością usera (`sub`, `name`, `email`, `nonce`). To on odpowiada na „kto się zalogował".
  Różnica: *access token* jest dla **API**, *ID token* dla **klienta** (nie wysyłaj ID tokenu do API jako Bearer).
- **scope `openid`** — magiczne słowo włączające OIDC; bez niego dostajesz czysty OAuth2.
- **endpoint `/userinfo`** — zwraca claimy o userze na podstawie access tokenu.
- **discovery** — `/.well-known/openid-configuration` (adresy endpointów, JWKS, issuer).

## 3. Dlaczego / kiedy — stateless, przechowywanie tokenu, pułapki

**Zalety (stateless):**
- Serwer **nie trzyma sesji** — cała informacja (kto, jakie role, do kiedy) jest w podpisanym tokenie.
- **Skaluje się poziomo** — każdy instancja API waliduje token samodzielnie, bez wspólnego session store.
  Idealne pod **load-balancing** (brak sticky sessions) i **mikroserwisy** (token przekazywany między serwisami).

**Wady / pułapki:**
- **Nie da się łatwo unieważnić** przed `exp` (token jest samowystarczalny — API go nie „odpytuje"). Skradziony
  token działa do wygaśnięcia. Mitigacja: **krótki access token (np. 5–15 min) + refresh token** + opcjonalnie
  denylist/introspection dla nagłych rewokacji.
- **Rozmiar** — JWT (zwłaszcza z rolami) jest większy niż session id; leci w każdym żądaniu (nagłówek).
- **Wyciek = dostęp** — kto ma token, ten ma dostęp (bearer). Zawsze HTTPS.

### Access + Refresh token (wzorzec)

- **Access token** — krótkożyjący JWT, wysyłany do API jako `Authorization: Bearer`.
- **Refresh token** — długożyjący, **tylko** do wymiany na nowy access token w AS (nie chodzi do API).
  Może być rotowany (refresh token rotation) i unieważniany po stronie AS.

### Gdzie trzymać token w SPA/Angular — kompromis (nie ma idealnej opcji)

| Miejsce | Zaleta | Podatność |
|---|---|---|
| **localStorage** | proste, przetrwa refresh strony | **XSS** — dowolny wstrzyknięty JS odczyta token |
| **httpOnly cookie** | JS **nie** ma dostępu (odporne na XSS-owy odczyt) | **CSRF** — trzeba `SameSite`/anty-CSRF token |
| **pamięć (zmienna JS)** | najtrudniejsze do wykradzenia | znika przy refreshu → potrzebny cichy refresh |

Współczesne zalecenie dla SPA: **Authorization Code + PKCE**, refresh token w **httpOnly + Secure + SameSite cookie**
(lub wzorzec BFF — Backend For Frontend trzymający tokeny po stronie serwera, klient dostaje tylko sesyjne cookie).

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Backend Spring Boot 3 jako **resource server** walidujący JWT z **Keycloaka** ([[wiedza/08-bezpieczenstwo/keycloak]]).

`build.gradle` / `pom.xml`: `spring-boot-starter-oauth2-resource-server`.

```yaml
# application.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # discovery + JWKS pobrane automatycznie z issuer-uri:
          issuer-uri: https://keycloak.example.com/realms/moja-appka
          # (opcjonalnie sam JWKS: jwk-set-uri: .../protocol/openid-connect/certs)
```

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity                       // dla @PreAuthorize("hasRole('ADMIN')")
class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(keycloakConverter())))
            .sessionManagement(s -> s
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)) // brak sesji!
            .csrf(csrf -> csrf.disable());   // OK dla bezstanowego API z Bearer
        return http.build();
    }

    // Keycloak trzyma role w realm_access.roles — mapujemy je na ROLE_*
    private JwtAuthenticationConverter keycloakConverter() {
        var conv = new JwtAuthenticationConverter();
        conv.setJwtGrantedAuthoritiesConverter(jwt -> {
            var realm = jwt.getClaimAsMap("realm_access");
            if (realm == null) return List.of();
            @SuppressWarnings("unchecked")
            var roles = (Collection<String>) realm.getOrDefault("roles", List.of());
            return roles.stream()
                .map(r -> new SimpleGrantedAuthority("ROLE_" + r))
                .collect(Collectors.toList());
        });
        return conv;
    }
}
```

Spring sam: pobiera JWKS z `issuer-uri`, waliduje podpis/`exp`/`iss`, buduje `JwtAuthenticationToken`
w `SecurityContext` ([[wiedza/08-bezpieczenstwo/spring-security]]). Dla klienta serwerowego (M2M / login przez
Google) użyłbyś zamiast tego `spring-boot-starter-oauth2-client` (`oauth2Login()` / `WebClient` z tokenem).

---

## ✅ Kryteria opanowania
- [ ] Wytłumaczę różnicę JWT vs OAuth2 vs OIDC jednym zdaniem każde.
- [ ] Wiem, że JWT jest zakodowany, nie zaszyfrowany — i co z tego wynika.
- [ ] Wymienię kroki walidacji JWT (podpis/exp/iss/aud) i skąd bierze się klucz (JWKS/kid).
- [ ] Uzasadnię stateless (skalowanie poziome) i jego cenę (rewokacja przed exp).
- [ ] Wyjaśnię wzorzec access + refresh token i przechowywanie w SPA (XSS vs CSRF).
- [ ] Naszkicuję Authorization Code + PKCE i powiem, po co PKCE.
- [ ] Skonfiguruję Spring Boot resource server pod Keycloak z głowy.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Z czego składa się JWT i co jest w każdej z trzech części?
- [ ] Dlaczego RS256 (asymetryczny) bywa lepszy niż HS256 dla wielu serwisów?
- [ ] Jak resource server waliduje token BEZ odpytywania authorization server?
- [ ] Czym różni się access token od ID tokenu i gdzie każdy trafia?
- [ ] Po co PKCE, skoro jest już redirect z code?
- [ ] Jak działa refresh token i dlaczego nie chodzi do API?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „OAuth2 vs OIDC — po co OIDC, skoro jest OAuth2?"
- [ ] „Czy w JWT można trzymać dane wrażliwe? Dlaczego nie?"
- [ ] „Jak unieważnić JWT przed wygaśnięciem?"
- [ ] „Gdzie trzymać token w Angularze — localStorage czy cookie?"
- [ ] „Który grant type dla SPA, a który dla komunikacji serwis-serwis?"
- [ ] „Co daje stateless auth i jaka jest jego cena?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/08-bezpieczenstwo.csv`. Format: `Pytanie;Odpowiedź`.*

```
Z czego składa się JWT?;Trzy części base64url oddzielone kropkami: header.payload.signature.
Czy JWT jest zaszyfrowany?;Nie — jest tylko zakodowany (base64url). Podpis chroni integralność, nie poufność; nie wkładaj sekretów.
Co chroni podpis JWT?;Integralność — że payload nie został zmieniony. NIE ukrywa treści (każdy ją odczyta).
Co oznacza claim iss?;Issuer — kto wystawił token (np. Twój Keycloak realm). Walidowany po stronie API.
Co oznacza claim aud?;Audience — dla kogo token jest przeznaczony (które API ma go zaakceptować).
Co oznacza claim sub?;Subject — identyfikator podmiotu/usera, którego dotyczy token.
Co oznacza claim exp?;Expiry — czas wygaśnięcia tokenu (epoch seconds); po nim token jest nieważny.
Jakie kroki obejmuje walidacja JWT?;Podpis (przez JWKS wg kid), exp/nbf, iss (zaufany wystawca), aud (dla tego API).
Skąd resource server bierze klucz do weryfikacji podpisu?;Z endpointu JWKS authorization servera; wybiera klucz po kid z headera.
HS256 vs RS256?;HS256 = symetryczny (wspólny sekret); RS256 = asymetryczny (prywatny podpisuje, publiczny weryfikuje) — lepszy dla wielu serwisów.
Główna zaleta JWT/stateless auth?;Serwer nie trzyma sesji — skaluje się poziomo, bez sticky sessions, idealne dla mikroserwisów i load-balancingu.
Główna wada JWT?;Trudno unieważnić przed exp — skradziony token działa do wygaśnięcia. Mitigacja: krótki access token + refresh.
Wzorzec access + refresh token?;Krótkożyjący access (do API jako Bearer) + długożyjący refresh (tylko do wymiany na nowy access w AS).
localStorage vs httpOnly cookie na token?;localStorage podatny na XSS (JS odczyta), httpOnly cookie odporny na XSS ale podatny na CSRF (potrzeba SameSite/anty-CSRF).
Po co jest OAuth2?;Delegacja autoryzacji — dostęp third-party do zasobów usera bez podawania mu hasła ("zaloguj przez Google").
Role w OAuth2?;Resource owner (user), client (aplikacja), authorization server (wydaje tokeny), resource server (chronione API).
Który grant type dla SPA/mobile?;Authorization Code + PKCE (klient publiczny bez secretu, PKCE chroni przechwycony code).
Który grant type dla machine-to-machine?;Client Credentials — klient uwierzytelnia się secretem, dostaje access token (bez usera).
Które grant types są przestarzałe?;Implicit i Resource Owner Password Credentials (deprecated w OAuth 2.1).
Po co PKCE?;Chroni Authorization Code Flow u klienta publicznego przed przechwyceniem code (code_verifier/code_challenge S256).
OAuth2 vs OIDC?;OAuth2 = autoryzacja ("co możesz"); OIDC = warstwa uwierzytelniania na OAuth2 dodająca tożsamość (ID Token, userinfo, scope openid).
Czym różni się ID token od access tokenu?;ID token (OIDC) niesie tożsamość usera i jest dla klienta; access token jest dla API. Nie wysyłaj ID tokenu do API jako Bearer.
Jak skonfigurować Spring Boot resource server pod Keycloak?;spring-boot-starter-oauth2-resource-server + issuer-uri realmu; Spring pobiera JWKS i waliduje JWT automatycznie.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/08-bezpieczenstwo/spring-security]] · [[wiedza/08-bezpieczenstwo/keycloak]]
