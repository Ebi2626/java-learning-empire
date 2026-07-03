---
temat: "Spring Security — architektura uwierzytelniania i autoryzacji"
faza: 8
status: nieopanowany
priorytet: 🔴
tags: [java, spring, bezpieczenstwo]
powiazane: ["[[wiedza/05-spring/proxy-aop]]", "[[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]]", "[[wiedza/07-api-design/cors-bledy]]"]
---

# Spring Security — architektura uwierzytelniania i autoryzacji

> **TL;DR:** Spring Security to **łańcuch filtrów serwletowych** (`SecurityFilterChain`) wpięty **PRZED** `DispatcherServlet`.
> **Authentication** (kim jesteś) rozwiązuje `AuthenticationManager → AuthenticationProvider → UserDetailsService`, wynik ląduje
> w `SecurityContextHolder` (ThreadLocal). **Authorization** (co możesz) sprawdzają `authorizeHttpRequests` (per-request) i method
> security (`@PreAuthorize`, przez proxy). W Spring Security 6 konfigurujesz to jednym beanem `SecurityFilterChain` + lambda DSL —
> `WebSecurityConfigurerAdapter` **usunięty**.

## 1. Co — authentication vs authorization, i API

Dwa rozłączne pojęcia, których nie wolno mylić:

- **Authentication (uwierzytelnianie) — „kim jesteś?"** Weryfikacja tożsamości: login+hasło, token JWT, certyfikat, OAuth2.
  Wynik: obiekt `Authentication` z ustawionym `authenticated=true`.
- **Authorization (autoryzacja) — „co ci wolno?"** Decyzja, czy *już uwierzytelniony* podmiot ma dostęp do zasobu/metody.
  Bazuje na `GrantedAuthority` (uprawnienia/role) niesionych przez `Authentication`.

Kolejność jest zawsze taka sama: **najpierw authN, potem authZ**. Nie da się autoryzować kogoś nieznanego (choć „anonymous"
to też forma uwierzytelnienia — pseudo-użytkownik `anonymousUser`).

Kluczowe typy domenowe:

```
Authentication            // "paszport" bieżącego żądania
 ├─ getPrincipal()        // KTO — zwykle UserDetails (po zalogowaniu) lub String (przed)
 ├─ getCredentials()      // hasło/token — czyszczone po udanym authN
 ├─ getAuthorities()      // Collection<GrantedAuthority> — role i uprawnienia
 └─ isAuthenticated()

GrantedAuthority          // pojedyncze uprawnienie, np. "ROLE_ADMIN" albo "READ_PRODUCT"
UserDetails               // kontrakt "użytkownik z bazy": getUsername/getPassword/getAuthorities + flagi (enabled, locked...)
UserDetailsService        // loadUserByUsername(String) -> UserDetails   (jedno-metodowy interfejs)
PasswordEncoder           // encode() / matches() — hashowanie i weryfikacja hasła
```

**Principal vs Authentication vs UserDetails** — częsta pułapka: `Principal` to najogólniejszy interfejs z `java.security`
(ma tylko `getName()`); `Authentication` go rozszerza i dokłada `authorities`+`credentials`; `getPrincipal()` zwraca zwykle
`UserDetails` (Twój obiekt użytkownika). Czyli: `Authentication.getPrincipal()` → `UserDetails`.

Nowoczesna konfiguracja (Security 6, bez adaptera):

```java
@Configuration
@EnableWebSecurity
class SecurityConfig {
    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .formLogin(Customizer.withDefaults())
            .build();
    }
}
```

## 2. Jak — łańcuch filtrów i SecurityContext pod spodem

**Spring Security to NIE magia adnotacji — to zestaw filtrów serwletowych (`jakarta.servlet.Filter`).** Cały ruch przechodzi
przez łańcuch filtrów, który stoi **przed** `DispatcherServlet` (a więc przed kontrolerami MVC). Punkt wejścia to jeden filtr
w kontenerze serwletowym — `DelegatingFilterProxy` → `FilterChainProxy` — który wybiera pasujący `SecurityFilterChain`
i przepuszcza żądanie przez jego wewnętrzną listę filtrów.

```
HTTP request
   │
   ▼
[ Servlet Container ]
   │  DelegatingFilterProxy  → FilterChainProxy  (springSecurityFilterChain)
   ▼
┌─────────────────── SecurityFilterChain (kolejność MA znaczenie) ───────────────────┐
│ 1. DisableEncodeUrlFilter                                                           │
│ 2. WebAsyncManagerIntegrationFilter                                                 │
│ 3. SecurityContextHolderFilter   ← ładuje SecurityContext (z sesji lub pusty)       │
│ 4. HeaderWriterFilter            ← nagłówki bezpieczeństwa (X-Frame-Options...)     │
│ 5. CorsFilter                    ← CORS ZANIM authN (preflight OPTIONS nie ma tokena)│
│ 6. CsrfFilter                    ← weryfikacja tokena CSRF (dla mutujących metod)   │
│ 7. LogoutFilter                                                                     │
│ 8. UsernamePasswordAuthenticationFilter  ← form login (POST /login)                │
│    (tu wpinasz WŁASNY JwtAuthenticationFilter dla REST API)                         │
│ 9. ... BasicAuthenticationFilter, RequestCacheAwareFilter ...                       │
│10. AnonymousAuthenticationFilter ← jeśli nikt nie uwierzytelnił → anonymousUser     │
│11. ExceptionTranslationFilter    ← łapie AuthenticationException/AccessDenied,      │
│                                     zwraca 401 (entry point) / 403                  │
│12. AuthorizationFilter           ← OSTATNI: sprawdza authorizeHttpRequests          │
└────────────────────────────────────────────────────────────────────────────────────┘
   │  (jeśli przejdzie)
   ▼
DispatcherServlet → Controller
```

**SecurityContext i SecurityContextHolder.** Uwierzytelniony `Authentication` żyje w `SecurityContext`, który
`SecurityContextHolder` trzyma w **`ThreadLocal`** (domyślna strategia `MODE_THREADLOCAL`). Dzięki temu w dowolnym miejscu
warstwy serwisowej — na tym samym wątku żądania — możesz zrobić:

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
```

Konsekwencje ThreadLocal, o które pytają na rozmowach:
- **`@Async` / nowe wątki** kontekstu NIE dziedziczą (chyba że `MODE_INHERITABLETHREADLOCAL` lub `DelegatingSecurityContextExecutor`).
- Po zakończeniu żądania kontener zwykle **reużywa wątek** z puli — dlatego `SecurityContextHolderFilter` **czyści** kontekst
  po przejściu łańcucha (inaczej wyciek tożsamości między żądaniami).
- **Reactive (WebFlux)** nie używa ThreadLocal — tam jest `ReactiveSecurityContextHolder` na `Context` Reactora.

**Przepływ authentication pod spodem (form login / DAO):**

```
UsernamePasswordAuthenticationFilter
   │  buduje UsernamePasswordAuthenticationToken(login, hasło)  (authenticated=false)
   ▼
AuthenticationManager  (impl: ProviderManager)
   │  iteruje po providerach, pyta "obsłużysz ten typ tokenu?"
   ▼
AuthenticationProvider  (np. DaoAuthenticationProvider)
   │  1) UserDetailsService.loadUserByUsername(login) → UserDetails (z bazy)
   │  2) PasswordEncoder.matches(surowe, zahashowane z UserDetails)
   │  3) sprawdza flagi (enabled, accountNonLocked, credentialsNonExpired...)
   ▼
zwraca w pełni uwierzytelniony Authentication (authenticated=true, z authorities, credentials wyczyszczone)
   ▼
trafia do SecurityContextHolder.getContext().setAuthentication(...)
```

**PasswordEncoder — dlaczego BCrypt jest *celowo wolny* i z solą.**
- Hasła nigdy nie trzyma się jawnie ani „szybkim" hashem (MD5/SHA-256) — te są za szybkie, więc atakujący liczy miliardy
  prób/s (brute force, tęczowe tablice).
- **BCrypt** ma wbudowany **work factor** (`strength`, domyślnie 10 → 2^10 iteracji): każde sprawdzenie hasła jest kosztowne
  (dziesiątki ms), co ogranicza brute-force. Ma też **wbudowaną, losową sól** zapisaną w samym hashu → dwaj użytkownicy z tym
  samym hasłem mają różne hashe, a tęczowe tablice są bezużyteczne.
- **`DelegatingPasswordEncoder`** (domyślny od Spring Security 5.7+, tworzony przez `PasswordEncoderFactories.createDelegatingPasswordEncoder()`)
  zapisuje hash z prefiksem algorytmu: `{bcrypt}$2a$10$...`. Dzięki temu można **migrować** algorytmy bez wygaszania starych
  haseł — encoder czyta prefiks i wybiera właściwą implementację. Stąd błąd `There is no PasswordEncoder mapped for the id "null"`,
  gdy hasło w bazie nie ma prefiksu.

## 3. Dlaczego / kiedy — stateless vs stateful, method security, CSRF/CORS

**Stateful (sesja + `JSESSIONID`) vs stateless (JWT).**
- **Stateful:** po zalogowaniu serwer trzyma `SecurityContext` w `HttpSession`, klient dostaje cookie `JSESSIONID`.
  Każde kolejne żądanie niesie cookie, `SecurityContextHolderFilter` odtwarza kontekst z sesji. Prosto, ale **skalowanie**
  wymaga sticky sessions albo współdzielonego magazynu sesji (Spring Session + Redis). Domyślne dla klasycznych aplikacji z UI.
- **Stateless:** brak sesji (`SessionCreationPolicy.STATELESS`). Tożsamość jest w **każdym** żądaniu, zwykle jako **JWT**
  w nagłówku `Authorization: Bearer ...` — patrz [[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]]. Serwer nic nie pamięta między
  żądaniami → łatwe skalowanie poziome, naturalne dla REST API i mikroserwisów. Kosztem: token trudno unieważnić przed
  wygaśnięciem (potrzeba krótkiego TTL + refresh/blacklist).

**Authorization — konfiguracja żądań (`authorizeHttpRequests`).**
- `requestMatchers("/admin/**").hasRole("ADMIN")` — dopasowanie ścieżek do reguł; reguły sprawdzane **z góry na dół**,
  pierwsza pasująca wygrywa → bardziej szczegółowe muszą być **przed** ogólnymi, a `anyRequest()` zawsze na końcu.
- **`hasRole` vs `hasAuthority` — kluczowa różnica z prefiksem `ROLE_`:** `hasRole("ADMIN")` sprawdza authority
  **`ROLE_ADMIN`** (Spring dokłada prefiks automatycznie), a `hasAuthority("ROLE_ADMIN")` sprawdza dosłownie `ROLE_ADMIN`.
  Czyli role to po prostu authorities z konwencją prefiksu `ROLE_`. Jeśli w bazie trzymasz uprawnienia bez prefiksu
  (`READ_PRODUCT`), używaj `hasAuthority`, nie `hasRole`.

**Method security (`@EnableMethodSecurity`).**
- Włącza adnotacje na metodach: `@PreAuthorize` (przed wywołaniem), `@PostAuthorize` (po — może badać zwracany obiekt),
  `@PreFilter`/`@PostFilter` (filtrowanie kolekcji). W wyrażeniach używa **SpEL**:
  ```java
  @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
  User getUser(Long userId) { ... }
  ```
- **Jak to działa: przez proxy AOP** — patrz [[wiedza/05-spring/proxy-aop]]. Spring owija bean w proxy, które przed metodą
  odpytuje `AuthorizationManager`. Stąd znane pułapki proxy: **wywołanie wewnętrzne** (`this.getUser(...)`) omija proxy,
  więc adnotacja NIE zadziała; adnotacja na metodzie prywatnej/`final` nie zadziała. `@EnableMethodSecurity` zastąpiło stare
  `@EnableGlobalMethodSecurity` (domyślnie `prePostEnabled=true`).

**CSRF — kiedy jest potrzebny.**
- CSRF (Cross-Site Request Forgery) grozi, gdy przeglądarka **automatycznie dołącza poświadczenia** — czyli przy uwierzytelnianiu
  **cookie/sesją** i formularzach. Atakujący z innej strony może wywołać mutujące żądanie z Twoim `JSESSIONID`. Dlatego dla
  aplikacji stateful CSRF jest **domyślnie włączony** (token synchronizujący dla POST/PUT/DELETE).
- **Przy stateless JWT CSRF zwykle się wyłącza** (`http.csrf(csrf -> csrf.disable())`), bo token nosisz **ręcznie** w nagłówku
  `Authorization`, którego przeglądarka nie dokłada automatycznie do cross-site żądań → wektor CSRF znika. (Uwaga: jeśli JWT
  trzymasz w cookie, CSRF wraca do gry.)

**CORS a Security — kolejność.** CORS i CSRF to różne rzeczy (patrz [[wiedza/07-api-design/cors-bledy]]). W łańcuchu
`CorsFilter` musi stać **przed** filtrami authN/authZ, bo preflight `OPTIONS` nie niesie poświadczeń — gdyby najpierw zadziałała
autoryzacja, preflight dostałby 401. Dlatego włączasz CORS przez `http.cors(...)` (bean `CorsConfigurationSource`), a nie ręcznie.

## Przykład w praktyce (REST API, stateless, JWT)

Typowa konfiguracja bezpiecznego REST API na Spring Boot 3 / Security 6:

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity                      // włącza @PreAuthorize itd.
class SecurityConfig {

    private final JwtAuthenticationFilter jwtFilter;   // Twój filtr czytający Bearer token
    SecurityConfig(JwtAuthenticationFilter jwtFilter) { this.jwtFilter = jwtFilter; }

    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())                                   // stateless + Bearer → CSRF zbędny
            .cors(Customizer.withDefaults())                                // korzysta z beanu CorsConfigurationSource
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.POST, "/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.GET, "/api/products/**").hasAuthority("READ_PRODUCT")
                .anyRequest().authenticated())
            // wpinamy JWT filtr PRZED filtrem form-login
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();  // {bcrypt}...
    }

    // AuthenticationManager potrzebny np. w endpointcie logowania wystawiającym JWT
    @Bean
    AuthenticationManager authenticationManager(AuthenticationConfiguration cfg) throws Exception {
        return cfg.getAuthenticationManager();
    }
}
```

`UserDetailsService` ładujący użytkownika z bazy (mapowanie encji → `UserDetails`):

```java
@Service
class DbUserDetailsService implements UserDetailsService {
    private final UserRepository users;
    DbUserDetailsService(UserRepository users) { this.users = users; }

    @Override
    public UserDetails loadUserByUsername(String username) {
        UserEntity u = users.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(username));
        var authorities = u.getRoles().stream()
            .map(r -> new SimpleGrantedAuthority("ROLE_" + r.name()))   // prefiks ROLE_ dla hasRole
            .toList();
        return User.withUsername(u.getUsername())
            .password(u.getPasswordHash())                             // już zahashowane {bcrypt}...
            .authorities(authorities)
            .disabled(!u.isActive())
            .build();
    }
}
```

Zabezpieczenie metody serwisu (autoryzacja domenowa przez proxy):

```java
@Service
class OrderService {
    @PreAuthorize("hasRole('ADMIN') or #ownerId == authentication.principal.username")
    public List<Order> getOrders(String ownerId) { ... }
}
```

---

## ✅ Kryteria opanowania
- [ ] Odróżniam authentication od authorization i wiem, że authN jest zawsze pierwsze.
- [ ] Wiem, że Spring Security to łańcuch filtrów serwletowych przed `DispatcherServlet`.
- [ ] Rozumiem przepływ `AuthenticationManager → AuthenticationProvider → UserDetailsService → UserDetails`.
- [ ] Wyjaśnię `SecurityContextHolder`/ThreadLocal i skutki dla `@Async` i reużycia wątków.
- [ ] Znam różnicę `hasRole` vs `hasAuthority` (prefiks `ROLE_`).
- [ ] Wiem, czemu BCrypt jest wolny i z solą oraz po co `DelegatingPasswordEncoder`.
- [ ] Umiem uzasadnić stateless (JWT) vs stateful (sesja) i skonfigurować `SecurityFilterChain` dla REST API.
- [ ] Rozumiem, kiedy CSRF jest potrzebny i czemu przy stateless JWT bywa wyłączany.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co dokładnie robi `DelegatingFilterProxy` / `FilterChainProxy` i gdzie stoi względem `DispatcherServlet`?
- [ ] Dlaczego kolejność filtrów ma znaczenie? Gdzie wpinasz własny `JwtAuthenticationFilter` i czemu `addFilterBefore`?
- [ ] Jak `@PreAuthorize` jest egzekwowane pod spodem i czemu wywołanie wewnętrzne (`this.metoda()`) je omija?
- [ ] Co dokładnie zwraca `SecurityContextHolder.getContext().getAuthentication().getPrincipal()`?
- [ ] Czemu `SecurityContextHolderFilter` czyści kontekst po żądaniu?
- [ ] Skąd bierze się błąd `There is no PasswordEncoder mapped for the id "null"`?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Jak działa Spring Security — od żądania HTTP do kontrolera?"
- [ ] „`hasRole('ADMIN')` vs `hasAuthority('ADMIN')` — jaka różnica?"
- [ ] „Czemu do haseł używamy BCrypt, a nie SHA-256?"
- [ ] „Kiedy wyłączyć CSRF i dlaczego?"
- [ ] „Stateless czy stateful — co wybierzesz dla REST API i czemu?"
- [ ] „Czym różni się `Authentication` od `Principal` i `UserDetails`?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Authentication vs authorization?;Authentication = "kim jesteś" (weryfikacja tożsamości), authorization = "co ci wolno" (dostęp już uwierzytelnionego). Najpierw authN, potem authZ.
Czym jest Spring Security pod spodem?;Łańcuchem filtrów serwletowych (SecurityFilterChain) wpiętym PRZED DispatcherServlet, wybieranym przez FilterChainProxy.
Jak konfiguruje się Spring Security 6?;Beanem SecurityFilterChain + lambda DSL na HttpSecurity; WebSecurityConfigurerAdapter został USUNIĘTY.
Gdzie przechowywany jest Authentication bieżącego użytkownika?;W SecurityContext trzymanym przez SecurityContextHolder w ThreadLocal (domyślnie MODE_THREADLOCAL).
Czemu SecurityContext jest czyszczony po żądaniu?;Bo kontener reużywa wątki z puli — bez czyszczenia doszłoby do wycieku tożsamości między żądaniami.
Co zwraca Authentication.getPrincipal()?;Zwykle obiekt UserDetails (reprezentuje zalogowanego użytkownika); przed authN bywa to String (login).
Różnica Principal, Authentication, UserDetails?;Principal (java.security) to tylko getName(); Authentication go rozszerza o authorities+credentials; UserDetails to "użytkownik z bazy" zwracany jako principal.
Przepływ uwierzytelniania w DAO?;AuthenticationManager (ProviderManager) → AuthenticationProvider (DaoAuthenticationProvider) → UserDetailsService.loadUserByUsername → UserDetails; potem PasswordEncoder.matches.
Do czego służy UserDetailsService?;Ma jedną metodę loadUserByUsername(String) zwracającą UserDetails — ładuje użytkownika (np. z bazy) na potrzeby uwierzytelnienia.
Czemu BCrypt jest celowo wolny?;Ma work factor (2^strength iteracji), co spowalnia każde sprawdzenie hasła i utrudnia brute-force; szybkie hashe (MD5/SHA-256) są niebezpieczne.
Po co sól w hashu hasła?;Losowa sól sprawia, że identyczne hasła mają różne hashe i unieważnia tęczowe tablice; BCrypt ma sól wbudowaną w hash.
Co robi DelegatingPasswordEncoder?;Zapisuje hash z prefiksem algorytmu, np. {bcrypt}..., dzięki czemu można migrować algorytmy bez wygaszania starych haseł.
hasRole vs hasAuthority?;hasRole('ADMIN') sprawdza authority ROLE_ADMIN (Spring dokłada prefiks ROLE_); hasAuthority('ROLE_ADMIN') sprawdza dosłownie. Role to authorities z konwencją ROLE_.
Jak działa @PreAuthorize?;Przez proxy AOP — Spring owija bean, przed metodą odpytuje AuthorizationManager (SpEL). Wywołanie wewnętrzne this.metoda() omija proxy i adnotacja nie zadziała.
Co włącza @EnableMethodSecurity?;Adnotacje method security: @PreAuthorize/@PostAuthorize/@PreFilter/@PostFilter z wyrażeniami SpEL; zastąpiło @EnableGlobalMethodSecurity.
Stateless vs stateful w Spring Security?;Stateful = SecurityContext w HttpSession + cookie JSESSIONID; stateless = brak sesji, tożsamość (JWT) w każdym żądaniu w nagłówku Authorization → łatwe skalowanie.
Kiedy CSRF jest potrzebny?;Gdy przeglądarka automatycznie dołącza poświadczenia — czyli przy sesji/cookie i formularzach. Domyślnie włączony dla aplikacji stateful.
Czemu przy stateless JWT często wyłącza się CSRF?;Bo token nosi się ręcznie w nagłówku Authorization, którego przeglądarka nie dokłada do cross-site żądań, więc wektor CSRF znika (chyba że JWT jest w cookie).
Czemu CorsFilter musi być przed authN w łańcuchu?;Bo preflight OPTIONS nie niesie poświadczeń — gdyby najpierw zadziałała autoryzacja, preflight dostałby 401.
Jak wpiąć własny filtr JWT?;http.addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class) — filtr czyta Bearer token i ustawia Authentication w SecurityContext.
Co robi ExceptionTranslationFilter?;Łapie AuthenticationException i AccessDeniedException z dalszej części łańcucha i mapuje na 401 (entry point) lub 403.
Jak włączyć STATELESS w Security 6?;http.sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS)) — Spring nie tworzy ani nie używa HttpSession.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/05-spring/proxy-aop]] · [[wiedza/08-bezpieczenstwo/jwt-oauth-oidc]] · [[wiedza/07-api-design/cors-bledy]]
