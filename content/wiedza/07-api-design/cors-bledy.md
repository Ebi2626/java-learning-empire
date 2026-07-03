---
temat: "CORS i kontrakt błędów w API"
faza: 7
status: nieopanowany
priorytet: 🔴
tags: [java, api, cors, angular]
powiazane: ["[[wiedza/08-bezpieczenstwo/spring-security]]", "[[wiedza/05-spring/walidacja-bledy]]"]
---

# CORS i kontrakt błędów w API

> **TL;DR:** **CORS** to mechanizm **przeglądarki** (nie serwera!) egzekwujący **Same-Origin Policy** — Angular na
> `localhost:4200` woła Spring na `localhost:8080` = **inny origin**, więc przeglądarka wysyła **preflight** (`OPTIONS`)
> i sprawdza nagłówki `Access-Control-Allow-*`. Osobno: **kontrakt błędów** — spójny format (**Problem Details, RFC 7807**,
> `application/problem+json`), właściwe kody statusu i **trace id**, bez wycieku stacktrace.

## 1. Co — czym jest CORS i skąd się bierze

**CORS** = *Cross-Origin Resource Sharing*. To rozluźnienie **Same-Origin Policy (SOP)** — reguły bezpieczeństwa
**przeglądarki**, która domyślnie blokuje skryptowi z jednego *origin* odczyt odpowiedzi z innego *origin*.

**Origin = schemat + host + port** (`https://app.example.com:443`). Wystarczy, że jedno się różni → inny origin:

```
http://localhost:4200   (Angular dev server)   ┐
http://localhost:8080   (Spring Boot API)       ├─ RÓŻNE ORIGINY (inny port)
                                                 │  → przeglądarka egzekwuje CORS
https://localhost:4200  vs http://localhost:4200 → inny schemat = inny origin
```

**Kluczowe (i najczęściej mylone):** CORS jest egzekwowany **wyłącznie przez przeglądarkę**.
- **curl / Postman / testy integracyjne backendu NIE mają CORS** — żądanie przejdzie, bo to nie przeglądarka.
- „Działa w Postmanie, nie działa w Angularze" = klasyczny objaw problemu CORS.
- Serwer **nie blokuje** — on tylko *deklaruje* nagłówkami, komu wolno; **decyzję o zablokowaniu podejmuje przeglądarka**.

Minimalny kontrakt nagłówków (odpowiedź serwera):
```http
Access-Control-Allow-Origin: http://localhost:4200
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 3600
```

## 2. Jak — SOP, preflight i nagłówki pod spodem

**Simple request vs non-simple.** Przeglądarka rozróżnia:
- **Simple request** — `GET`/`HEAD`/`POST` z „bezpiecznymi" nagłówkami i `Content-Type` ∈
  {`text/plain`, `application/x-www-form-urlencoded`, `multipart/form-data`}. Wysyłany **od razu**;
  przeglądarka dopiero **po** odpowiedzi sprawdza `Access-Control-Allow-Origin` i ewentualnie ukrywa wynik.
- **Non-simple request** — np. `Content-Type: application/json`, `Authorization`, metoda `PUT`/`DELETE`/`PATCH`.
  Tu przeglądarka wysyła **preflight**.

**Preflight** — automatyczne żądanie `OPTIONS` **przed** właściwym, pytające serwer „czy mi wolno?":
```http
OPTIONS /api/orders HTTP/1.1
Origin: http://localhost:4200
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type, authorization
```
Serwer odpowiada `2xx` z nagłówkami `Access-Control-Allow-*`. Dopiero gdy przeglądarka uzna odpowiedź za zgodną,
puszcza **właściwe** żądanie. `Access-Control-Max-Age` mówi, jak długo cache'ować wynik preflightu (mniej `OPTIONS`).

```
Angular          Przeglądarka                 Spring
  fetch(POST) ──▶ [non-simple?] ──▶ OPTIONS ─────────▶ 200 + Allow-* 
                        │  ◀───────────────────────────┘
                        │  [zgodne?] ──▶ POST /api/orders ─▶ 200 + body
                        │              ◀──────────────────────┘
                        └─ [niezgodne] → BLOCK, "blocked by CORS policy"
```

**Nagłówki — pułapki:**
- `Access-Control-Allow-Origin` — **konkretny origin** albo `*`. Wartość *odbija* origin żądania (przy wielu dozwolonych).
- **`Allow-Credentials: true` NIE działa z `*`** — przy `withCredentials`/ciasteczkach/`Authorization` **musisz** podać
  konkretny origin, nie wildcard. To najczęstszy błąd konfiguracji.
- `Allow-Methods` / `Allow-Headers` — muszą obejmować to, o co pyta preflight (`Access-Control-Request-*`).
- **Odpowiedź błędu (4xx/5xx) też musi nieść nagłówki CORS** — inaczej przeglądarka ukryje ciało błędu i zobaczysz
  tylko „CORS", zamiast realnego `400`/`401`.

## 3. Dlaczego / kiedy — konfiguracja, diagnoza, kontrakt błędu

### Konfiguracja w Spring Boot 3

**a) `@CrossOrigin`** — punktowo na kontrolerze/metodzie (szybkie, ale rozproszone):
```java
@CrossOrigin(origins = "http://localhost:4200")
@RestController
class OrderController { /* ... */ }
```

**b) Globalnie przez `WebMvcConfigurer`** — jedno miejsce prawdy:
```java
@Configuration
class CorsConfig implements WebMvcConfigurer {
    @Override public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("http://localhost:4200")   // NIE "*" przy credentials
                .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }
}
```

**c) Ze Spring Security** — najważniejszy przypadek. CORS musi zadziałać **w łańcuchu filtrów, PRZED autoryzacją**,
bo preflight `OPTIONS` **nie niesie** nagłówka `Authorization`. Jeśli Security zablokuje `OPTIONS`, dostaniesz
**preflight 401/403** i przeglądarka zgłosi „CORS", choć realnie to problem autoryzacji.
→ [[wiedza/08-bezpieczenstwo/spring-security]] (kolejność filtrów).
```java
@Bean
SecurityFilterChain chain(HttpSecurity http) throws Exception {
    http.cors(Customizer.withDefaults())   // włącza CorsFilter PRZED autoryzacją
        .csrf(csrf -> csrf.disable())
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()  // przepuść preflight
            .anyRequest().authenticated());
    return http.build();
}

@Bean
CorsConfigurationSource corsConfigurationSource() {
    var cfg = new CorsConfiguration();
    cfg.setAllowedOrigins(List.of("http://localhost:4200"));
    cfg.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    cfg.setAllowedHeaders(List.of("Content-Type", "Authorization"));
    cfg.setAllowCredentials(true);
    var source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", cfg);
    return source;
}
```

### Typowe błędy i diagnoza
| Objaw | Przyczyna | Fix |
|---|---|---|
| `blocked by CORS policy: No 'Access-Control-Allow-Origin'` | serwer nie zwraca nagłówka (brak konfiguracji / błędna ścieżka) | dodaj mapping dla właściwego path |
| `credentials mode ... Allow-Origin '*'` | `allowCredentials(true)` + wildcard | podaj **konkretny** origin |
| preflight `401`/`403` | Security blokuje `OPTIONS` przed CORS | `http.cors(...)` + `permitAll` na `OPTIONS` |
| „CORS", a to `500` | odpowiedź błędu bez nagłówków CORS | upewnij się, że `@ControllerAdvice` też przechodzi przez `CorsFilter` |
| działa w Postmanie, nie w przeglądarce | to zawsze CORS — Postman go nie egzekwuje | diagnozuj w DevTools → Network → nagłówki odpowiedzi `OPTIONS` |

### Kontrakt błędów (osobny, równie ważny temat)
Front potrzebuje **przewidywalnego** formatu błędu, nie surowego stacktrace. Standard: **Problem Details, RFC 7807**
(`Content-Type: application/problem+json`) — w Spring 6/Boot 3 wsparte przez `ProblemDetail` i `@ControllerAdvice`.
→ szczegóły walidacji: [[wiedza/05-spring/walidacja-bledy]].

Zasady dobrego kontraktu:
- **Właściwy kod statusu**: `400` walidacja, `401` brak auth, `403` brak uprawnień, `404` nie ma zasobu,
  `409` konflikt (np. optimistic lock / duplikat), `422` semantycznie błędne dane, `5xx` awaria serwera.
- **Korelacja**: `traceId` w ciele błędu (link do logów/tracingu) — front pokazuje go userowi, ty znajdujesz w logach.
- **Nie ujawniaj** stacktrace, nazw klas, zapytań SQL, ścieżek — to wektor ataku i szum dla frontu.
- **Komunikaty zrozumiałe** dla frontu i stabilne (kontrakt): pole `title`/`detail`, opcjonalnie kod błędu domenowego.
- **Mapowanie wyjątków** w jednym miejscu (`@RestControllerAdvice`), nie ad hoc w kontrolerach.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Angular (`localhost:4200`) z `withCredentials: true` woła Spring API (`localhost:8080`). Globalny `CorsConfigurationSource`
odbija origin i pozwala na `Authorization`; Security przepuszcza `OPTIONS`. Błędy walidacji lecą jako `problem+json`:

```java
@RestControllerAdvice
class ApiExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail onValidation(MethodArgumentNotValidException ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);   // 400
        pd.setTitle("Validation failed");
        pd.setDetail("Request contains invalid fields");
        pd.setProperty("traceId", MDC.get("traceId"));
        pd.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(FieldError::getField, FieldError::getDefaultMessage)));
        return pd;
    }

    @ExceptionHandler(OptimisticLockingFailureException.class)
    ProblemDetail onConflict(OptimisticLockingFailureException ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);      // 409
        pd.setTitle("Resource conflict");
        pd.setDetail("The resource was modified by another request. Reload and retry.");
        pd.setProperty("traceId", MDC.get("traceId"));
        return pd;
    }
}
```

Odpowiedź `400`:
```http
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json
Access-Control-Allow-Origin: http://localhost:4200
```
```json
{
  "type": "about:blank",
  "title": "Validation failed",
  "status": 400,
  "detail": "Request contains invalid fields",
  "traceId": "a1b2c3d4e5f6",
  "errors": { "email": "must be a well-formed email address", "amount": "must be positive" }
}
```
Odpowiedź `409` (konflikt):
```json
{ "title": "Resource conflict", "status": 409,
  "detail": "The resource was modified by another request. Reload and retry.",
  "traceId": "9f8e7d6c5b4a" }
```

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, że CORS to mechanizm **przeglądarki** i czemu curl/Postman go nie widzą.
- [ ] Zdefiniuję **origin** (schemat+host+port) i wskażę, czemu 4200 vs 8080 to inny origin.
- [ ] Opiszę **preflight** (`OPTIONS`, `Access-Control-Request-*` → `Allow-*`) i kiedy się pojawia.
- [ ] Wiem, czemu `Allow-Credentials: true` nie współgra z `*`.
- [ ] Skonfiguruję CORS globalnie i z Spring Security (kolejność filtrów, `OPTIONS` `permitAll`).
- [ ] Zaprojektuję kontrakt błędu w **RFC 7807** z `traceId`, bez wycieku szczegółów.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Kto dokładnie blokuje żądanie przy CORS — serwer czy przeglądarka?
- [ ] Czym różni się *simple* od *non-simple* request i co wywołuje preflight?
- [ ] Czemu preflight nie niesie `Authorization` i co to znaczy dla Security?
- [ ] Czemu odpowiedź `5xx` bywa raportowana w przeglądarce jako „CORS"?
- [ ] Po co `traceId` w odpowiedzi błędu i gdzie się go generuje (MDC/tracing)?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Działa w Postmanie, nie w Angularze — czemu?"
- [ ] „Jak skonfigurować CORS w Spring z Security?"
- [ ] „Czemu wildcard `*` nie działa z ciasteczkami/credentials?"
- [ ] „Jak wygląda dobry kontrakt błędów w REST API?" (RFC 7807, statusy, korelacja)
- [ ] „Preflight zwraca 401 — od czego zaczniesz diagnozę?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Co to CORS?;Cross-Origin Resource Sharing — mechanizm przeglądarki rozluźniający Same-Origin Policy dla żądań między różnymi originami.
Kto egzekwuje CORS — serwer czy przeglądarka?;Wyłącznie przeglądarka; serwer tylko deklaruje nagłówkami Access-Control-Allow-*, decyzję o blokadzie podejmuje przeglądarka.
Czemu curl i Postman "nie mają CORS"?;CORS jest egzekwowany tylko przez przeglądarkę; klienty spoza przeglądarki ignorują nagłówki Access-Control-*.
Co to origin w kontekście SOP?;Trójka schemat + host + port; różnica w którymkolwiek = inny origin.
Czemu localhost:4200 i localhost:8080 to różne originy?;Różnią się portem, a port jest częścią origin.
Co to preflight?;Automatyczne żądanie OPTIONS wysyłane przez przeglądarkę przed non-simple requestem, pytające serwer o dozwolone metody/nagłówki.
Kiedy przeglądarka wysyła preflight?;Dla non-simple requestów — np. Content-Type application/json, nagłówek Authorization, metody PUT/DELETE/PATCH.
Jakie nagłówki niesie preflight?;Origin, Access-Control-Request-Method, Access-Control-Request-Headers.
Czym odpowiada serwer na preflight?;2xx z Access-Control-Allow-Origin/Methods/Headers oraz opcjonalnie Max-Age.
Do czego służy Access-Control-Max-Age?;Mówi przeglądarce, jak długo cache'ować wynik preflightu, by nie wysyłać OPTIONS przy każdym żądaniu.
Czemu Allow-Credentials: true nie działa z Allow-Origin *?;Przy credentials (ciasteczka/Authorization) spec wymaga konkretnego origin; wildcard jest wtedy odrzucany przez przeglądarkę.
Jak włączyć CORS w Spring Security?;http.cors(...) w SecurityFilterChain + bean CorsConfigurationSource; CorsFilter działa przed autoryzacją.
Czemu preflight bywa 401/403 w Spring Security?;Security blokuje OPTIONS przed CORS-em, a preflight nie niesie Authorization; trzeba przepuścić OPTIONS (permitAll) i włączyć cors().
Czemu błąd 500 bywa pokazany jako "CORS"?;Bo odpowiedź błędu nie zawiera nagłówków CORS, więc przeglądarka ukrywa ciało i raportuje tylko naruszenie CORS.
Jaki standard formatu błędów w REST?;Problem Details (RFC 7807), Content-Type application/problem+json; w Spring 6 klasa ProblemDetail.
Po co traceId w odpowiedzi błędu?;Korelacja odpowiedzi z logami/tracingiem; front pokazuje go userowi, backend odnajduje w logach.
Jaki status HTTP dla konfliktu zasobu?;409 Conflict — np. optimistic locking lub duplikat.
Czego NIE ujawniać w odpowiedzi błędu?;Stacktrace, nazw klas, zapytań SQL i ścieżek — to wektor ataku i szum dla frontu.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/08-bezpieczenstwo/spring-security]] · [[wiedza/05-spring/walidacja-bledy]]
