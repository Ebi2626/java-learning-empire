---
temat: "Spring MVC i budowa REST API"
faza: 5
status: nieopanowany
priorytet: 🔴
tags: [java, spring, web]
powiazane: ["[[wiedza/05-spring/walidacja-bledy]]", "[[wiedza/07-api-design/rest]]", "[[wiedza/03-wspolbieznosc/virtual-threads]]", "[[wiedza/07-api-design/cors-bledy]]"]
---

# Spring MVC i budowa REST API

> **TL;DR:** Spring MVC to **synchroniczny, blokujący** framework webowy oparty na **Servlet API**. Wszystko przechodzi przez jeden **front controller** — `DispatcherServlet` — który dobiera **HandlerMapping** → **HandlerAdapter** → metodę kontrolera, a wynik serializuje przez **HttpMessageConverter** (Jackson dla JSON). W REST używasz `@RestController` (= `@Controller` + `@ResponseBody`), zwracasz `ResponseEntity` (status + nagłówki + body) i wiążesz wejście przez `@PathVariable`/`@RequestParam`/`@RequestBody`. Model wątkowy: **thread-per-request** — alternatywy to WebFlux i [[wiedza/03-wspolbieznosc/virtual-threads|virtual threads]].

## 1. Co — definicja i API

**Spring MVC** = warstwa web frameworka Spring zbudowana na **Servlet API** (Jakarta Servlet w Spring 6 / Boot 3). Realizuje wzorzec **Model-View-Controller** i wzorzec **Front Controller**: pojedynczy servlet (`DispatcherServlet`) przyjmuje *wszystkie* żądania i rozdziela je do kontrolerów.

**Dwa tryby kontrolera:**
- `@Controller` — klasyczny MVC. Metoda zwraca **nazwę widoku** (String) lub `ModelAndView`; `ViewResolver` renderuje HTML (Thymeleaf, JSP). Model danych trafia do szablonu.
- `@RestController` — **meta-adnotacja** = `@Controller` + `@ResponseBody`. Każda metoda zwraca **dane** (obiekt/DTO), nie widok. Wynik jest serializowany do body odpowiedzi (JSON) przez `HttpMessageConverter`. To domyślny wybór dla REST API.

**Mapowanie żądań** (którą metodę wywołać):
```java
@RequestMapping(path = "/api/users", method = RequestMethod.GET, produces = "application/json")
```
Skróty (composed annotations) — najczęściej używane:

| Adnotacja        | Metoda HTTP | Idempotentna? | Typowe użycie             |
|------------------|-------------|---------------|---------------------------|
| `@GetMapping`    | GET         | tak           | odczyt zasobu             |
| `@PostMapping`   | POST        | nie           | tworzenie / operacja      |
| `@PutMapping`    | PUT         | tak           | pełna podmiana zasobu     |
| `@PatchMapping`  | PATCH       | nie*          | częściowa aktualizacja    |
| `@DeleteMapping` | DELETE      | tak           | usunięcie                 |

**Wiązanie danych wejściowych (data binding):**
- `@PathVariable Long id` — segment ścieżki: `/users/{id}`.
- `@RequestParam String q` — query string / form field: `?q=...` (obsługuje `required`, `defaultValue`).
- `@RequestBody CreateUserDto dto` — **deserializacja body** (JSON → obiekt) przez `HttpMessageConverter`.
- `@RequestHeader("Authorization") String auth` — pojedynczy nagłówek.
- `@ModelAttribute` — wiąże parametry formularza/query do obiektu (klasyczny MVC, `application/x-www-form-urlencoded`), NIE do body JSON.

**Odpowiedź:**
- `@ResponseBody` — wynik metody idzie do body (już wliczone w `@RestController`).
- `ResponseEntity<T>` — pełna kontrola: **status + nagłówki + body** w jednym obiekcie.
- `@ResponseStatus(HttpStatus.CREATED)` — statyczny kod statusu na metodzie/wyjątku (mniej elastyczne niż `ResponseEntity`).
- Kody statusu HTTP: `200 OK`, `201 Created` (+ `Location`), `204 No Content`, `400 Bad Request`, `401/403`, `404 Not Found`, `409 Conflict`, `422 Unprocessable Entity`, `500`.

## 2. Jak — DispatcherServlet i konwertery pod spodem

**Przepływ pojedynczego żądania** (życie requestu w `DispatcherServlet.doDispatch()`):

```
                         HTTP request
                              │
                   Servlet Filter chain  (Filter, np. auth, logging, encoding)
                              │
                    ┌─────────▼─────────┐
                    │  DispatcherServlet │  ← FRONT CONTROLLER (jeden servlet na wszystko)
                    └─────────┬─────────┘
   1. HandlerMapping ─────────┤  „która metoda obsłuży to URL+method?"
      (RequestMappingHandlerMapping) → zwraca HandlerExecutionChain
                              │
   2. HandlerInterceptor.preHandle()  (0..n interceptorów)
                              │
   3. HandlerAdapter ─────────┤  wie JAK wywołać handler
      (RequestMappingHandlerAdapter)
        ├─ rozwiązuje argumenty (HandlerMethodArgumentResolver):
        │    @PathVariable, @RequestParam, @RequestBody → HttpMessageConverter.read()
        ├─ WYWOŁUJE metodę kontrolera  ← twój kod
        └─ przetwarza wynik (HandlerMethodReturnValueHandler):
             @ResponseBody / ResponseEntity → HttpMessageConverter.write()
                              │
   4. HandlerInterceptor.postHandle()
                              │
      (@Controller: ViewResolver + render; @RestController pomija ten krok)
                              │
   5. HandlerInterceptor.afterCompletion()
                              │
                        HTTP response
```

Kluczowe elementy „pod spodem":

- **HandlerMapping** — mapa `{URL pattern + HTTP method + produces/consumes}` → `HandlerMethod`. Budowana raz na starcie ze skanu adnotacji `@RequestMapping`. Wybiera *najbardziej specyficzne* dopasowanie.
- **HandlerAdapter** — abstrakcja „jak zawołać handler". Dla adnotowanych kontrolerów: `RequestMappingHandlerAdapter`, który używa łańcucha **ArgumentResolverów** (skąd wziąć każdy parametr) i **ReturnValueHandlerów** (co zrobić z wynikiem).
- **HttpMessageConverter** — dwukierunkowa (de)serializacja między body a obiektem Java. Wybór konwertera zależy od **Content-Type** (przy read) i **Accept** (przy write — patrz negocjacja treści). Dla JSON: `MappingJackson2HttpMessageConverter` (Jackson). Inne: form, byte[], String, XML.

**Negocjacja treści (content negotiation):** klient wysyła nagłówek `Accept: application/json`, Spring iteruje po zarejestrowanych konwerterach i wybiera pierwszy, który obsłuży dany typ i klasę zwracaną. `produces`/`consumes` w mapowaniu zawężają dopuszczalne typy (błąd `406 Not Acceptable` / `415 Unsupported Media Type`).

**Jackson — jak działa (de)serializacja:**
- Serializacja: `ObjectMapper` odczytuje **gettery / pola** obiektu i zapisuje JSON. Deserializacja: dopasowuje klucze JSON do właściwości i woła **konstruktor / settery**.
- `@JsonProperty("user_name")` — mapuje pole na inną nazwę w JSON (np. snake_case ↔ camelCase).
- `@JsonIgnore` — wyklucza pole z (de)serializacji (np. hasło).
- `@JsonFormat(pattern = "yyyy-MM-dd")` — format daty/liczby.
- Konfiguracja globalna: bean `ObjectMapper` lub `Jackson2ObjectMapperBuilderCustomizer`; typowo: `FAIL_ON_UNKNOWN_PROPERTIES=false`, rejestracja `JavaTimeModule` (Boot robi to auto), `WRITE_DATES_AS_TIMESTAMPS=false`.

**Filter vs HandlerInterceptor** — dwa różne poziomy przechwytywania:

| Cecha        | `Filter` (Servlet)                       | `HandlerInterceptor` (Spring MVC)          |
|--------------|------------------------------------------|--------------------------------------------|
| Poziom       | kontener servletowy, **przed** DispatcherServlet | wewnątrz DispatcherServlet, **po** HandlerMapping |
| Wie co do?   | nie zna handlera/kontrolera              | zna wybrany `HandlerMethod`                |
| Dostęp       | surowy `ServletRequest/Response`         | request + handler + `ModelAndView`         |
| Może zmienić body| tak (wrapping request/response)      | ogranicza się głównie do pre/post logiki   |
| Kiedy        | CORS, kompresja, security (Spring Security to filtr!), auth przed routingiem | logika zależna od kontrolera, np. audyt/logowanie na poziomie MVC |
| Kolejność    | `@Order` / rejestracja w chainie         | kolejność rejestracji w `WebMvcConfigurer` |

Reguła: co dotyczy **wszystkich** requestów niezależnie od routingu i musi być wcześnie (security, encoding) → **Filter**. Co potrzebuje wiedzy o wybranym kontrolerze → **Interceptor**.

## 3. Dlaczego / kiedy — model wątkowy, dobór klienta

**Model wątkowy: thread-per-request (blokujący).**
Spring MVC działa na Servlet: kontener (Tomcat) trzyma **pulę wątków**; każde żądanie zajmuje **jeden wątek** od początku do końca. Jeśli metoda robi blokujące I/O (JDBC, wołanie innego serwisu przez `RestClient`), wątek **czeka bezczynnie**. Przy dużej liczbie wolnych zależności pula (`server.tomcat.threads.max`, domyślnie 200) się wyczerpuje → żądania się kolejkują.

Kontrast:
- **WebFlux (reactive)** — nieblokujący, event-loop, mało wątków (Netty). Skaluje się na I/O-bound bez wysysania puli, ale programowanie w `Mono`/`Flux` jest trudniejsze i cała ścieżka musi być non-blocking.
- **Virtual threads (Java 21, Project Loom)** — zmieniają równanie: zachowujesz prosty, **blokujący** styl MVC, ale wątek platformowy nie jest blokowany podczas I/O. Spring Boot 3.2+ włącza je flagą `spring.threads.virtual.enabled=true`. To często **lepszy wybór niż WebFlux** dla klasycznego I/O-bound API — patrz [[wiedza/03-wspolbieznosc/virtual-threads]].

**Wołanie innych usług — dobór klienta HTTP:**
- **`RestClient`** (Spring 6.1 / Boot 3.2, **nowy**) — synchroniczny, fluent API. Domyślny wybór dla MVC dziś. Zastępuje `RestTemplate`.
- **`WebClient`** (z WebFlux) — reaktywny, nieblokujący; użyj gdy potrzebujesz strumieni/reactive lub w aplikacji WebFlux. Ma też tryb blokujący (`.block()`), ale to code smell w MVC.
- **`RestTemplate`** (**legacy**) — nadal działa, ale w trybie *maintenance*, nowych ficzerów nie dostanie. W nowym kodzie preferuj `RestClient`.

**Async w MVC (zajawka):** metoda może zwrócić `Callable<T>` (Spring wykona ją na osobnym wątku puli, zwalniając wątek Tomcata) lub `DeferredResult<T>` (wynik ustawiany z innego wątku, np. po zdarzeniu/kolejce). Pozwala uwolnić wątek servletu na czas długiej operacji bez pełnego WebFlux.

**CORS (zajawka):** `@CrossOrigin` na kontrolerze/metodzie lub globalna konfiguracja w `WebMvcConfigurer.addCorsMappings`. Realizowany przez preflight `OPTIONS` i nagłówki `Access-Control-Allow-*`. Szczegóły i pułapki: [[wiedza/07-api-design/cors-bledy]].

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Typowa warstwa web REST-owego API w Spring Boot 3 / Java 21 — kontroler z DTO i `ResponseEntity`:

```java
public record CreateUserRequest(
        @NotBlank String name,
        @Email String email) {}

public record UserResponse(Long id, String name, String email) {}

@RestController
@RequestMapping("/api/users")
class UserController {

    private final UserService service;

    UserController(UserService service) {
        this.service = service;
    }

    @GetMapping("/{id}")
    ResponseEntity<UserResponse> getById(@PathVariable Long id) {
        return service.find(id)
                .map(ResponseEntity::ok)                 // 200 + body
                .orElseGet(() -> ResponseEntity.notFound().build()); // 404
    }

    @GetMapping
    List<UserResponse> list(@RequestParam(defaultValue = "0") int page) {
        return service.page(page);                       // @ResponseBody domyślnie → JSON
    }

    @PostMapping
    ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserRequest req) {
        UserResponse created = service.create(req);
        return ResponseEntity
                .created(URI.create("/api/users/" + created.id())) // 201 + Location
                .body(created);
    }

    @DeleteMapping("/{id}")
    ResponseEntity<Void> delete(@PathVariable Long id) {
        service.delete(id);
        return ResponseEntity.noContent().build();       // 204
    }
}
```

Obsługa błędów żyje osobno w `@RestControllerAdvice` z `@ExceptionHandler` (mapowanie wyjątków na kody + ciało błędu, najlepiej `ProblemDetail` z RFC 7807) — patrz [[wiedza/05-spring/walidacja-bledy]].

---

## ✅ Kryteria opanowania
*Temat jest DOMKNIĘTY, gdy odpowiesz na wszystkie BEZ zaglądania.*

- [ ] Narysuję z pamięci przepływ żądania: request → HandlerMapping → HandlerAdapter → kontroler → HttpMessageConverter → response.
- [ ] Wyjaśnię, czym `@RestController` różni się od `@Controller` i z czego się składa.
- [ ] Poprawnie dobiorę adnotację wiązania wejścia (`@PathVariable`/`@RequestParam`/`@RequestBody`/`@RequestHeader`).
- [ ] Zbuduję `ResponseEntity` z właściwym statusem, nagłówkiem `Location` i body.
- [ ] Wyjaśnię thread-per-request i uzasadnię wybór virtual threads vs WebFlux.
- [ ] Dobiorę `RestClient` / `WebClient` / `RestTemplate` do sytuacji.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co konkretnie robi `DispatcherServlet` i dlaczego to „front controller"?
- [ ] Czym różni się HandlerMapping od HandlerAdapter?
- [ ] Jak Jackson zamienia obiekt na JSON i z powrotem? Rola `ObjectMapper`.
- [ ] Jak działa negocjacja treści (Accept vs produces/consumes)?
- [ ] Filter vs HandlerInterceptor — który jest wcześniej i który zna kontroler?
- [ ] Dlaczego blokujące I/O wyczerpuje pulę wątków Tomcata i co to zmieniają virtual threads?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Opisz drogę żądania HTTP przez Spring MVC."
- [ ] „`@RestController` vs `@Controller`?"
- [ ] „`@RequestParam` vs `@PathVariable` vs `@RequestBody`?"
- [ ] „Kiedy `ResponseEntity`, a kiedy `@ResponseStatus`?"
- [ ] „Filter czy Interceptor — gdzie zaimplementujesz X?"
- [ ] „MVC blokujący vs WebFlux vs virtual threads — kiedy co?"
- [ ] „`RestTemplate`, `WebClient`, `RestClient` — czym się różnią?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/05-spring.csv`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Co to DispatcherServlet?;Front controller Spring MVC — jeden servlet przyjmujący wszystkie żądania i rozdzielający je do kontrolerów.
Kolejność przepływu żądania w Spring MVC?;request → HandlerMapping → HandlerAdapter → metoda kontrolera → HttpMessageConverter → response.
Co robi HandlerMapping?;Na podstawie URL+metody HTTP wybiera właściwą metodę kontrolera (HandlerMethod).
Co robi HandlerAdapter?;Wie JAK wywołać handler — rozwiązuje argumenty i przetwarza wartość zwracaną.
@RestController = ?;@Controller + @ResponseBody — metody zwracają dane (JSON), nie widoki.
Czym różni się @Controller od @RestController?;@Controller zwraca nazwy widoków (renderowane przez ViewResolver), @RestController zwraca serializowane dane.
Do czego @PathVariable?;Wiąże segment ścieżki URL, np. /users/{id}.
Do czego @RequestParam?;Wiąże parametr query string lub pole formularza (?q=...).
Do czego @RequestBody?;Deserializuje body żądania (JSON) do obiektu przez HttpMessageConverter.
Do czego @RequestHeader?;Wiąże wartość nagłówka HTTP do parametru metody.
Do czego @ModelAttribute?;Wiąże parametry formularza/query do obiektu (form-urlencoded), NIE body JSON.
Po co ResponseEntity?;Pełna kontrola nad odpowiedzią: status HTTP + nagłówki + body w jednym obiekcie.
Kod HTTP dla utworzenia zasobu?;201 Created, zwykle z nagłówkiem Location wskazującym nowy zasób.
Kod HTTP przy braku ciała odpowiedzi (np. DELETE)?;204 No Content.
Co to HttpMessageConverter?;Komponent (de)serializujący body ↔ obiekt Java; dla JSON to Jackson (MappingJackson2HttpMessageConverter).
Jak działa negocjacja treści?;Spring wybiera konwerter na podstawie nagłówka Accept (write) i Content-Type (read); produces/consumes zawężają typy.
Co robi @JsonIgnore?;Wyklucza pole z serializacji i deserializacji Jacksona (np. hasło).
Co robi @JsonProperty?;Mapuje pole Javy na inną nazwę klucza w JSON (np. snake_case).
Filter vs HandlerInterceptor — który wcześniej?;Filter działa w kontenerze servletowym PRZED DispatcherServlet; Interceptor wewnątrz MVC, PO wyborze handlera.
Który zna wybrany kontroler: Filter czy Interceptor?;HandlerInterceptor (dostaje HandlerMethod); Filter operuje na surowym request/response.
Jaki model wątkowy ma Spring MVC?;Thread-per-request, blokujący, na Servlet API — jeden wątek z puli na całe żądanie.
Dlaczego virtual threads pomagają MVC?;Pozwalają pisać blokujący kod, nie blokując wątku platformowego podczas I/O (spring.threads.virtual.enabled=true).
Który klient HTTP jest nowy i zalecany w Spring 6.1?;RestClient (synchroniczny, fluent) — zastępuje przestarzały RestTemplate.
Do czego DeferredResult/Callable w MVC?;Async — zwalniają wątek servletu na czas długiej operacji, wynik ustawiany później/na innym wątku.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/05-spring/walidacja-bledy]] · [[wiedza/07-api-design/rest]] · [[wiedza/07-api-design/cors-bledy]] · [[wiedza/03-wspolbieznosc/virtual-threads]]
