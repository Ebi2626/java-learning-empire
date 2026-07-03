---
temat: "Walidacja danych i globalna obsługa błędów w Spring"
faza: 5
status: nieopanowany
priorytet: 🔴
tags: [java, spring, web]
powiazane: ["[[wiedza/05-spring/mvc-rest]]", "[[wiedza/01-jezyk/wyjatki]]", "[[wiedza/07-api-design/rest]]"]
---

# Walidacja danych i globalna obsługa błędów w Spring

> **TL;DR:** Deklaratywna **Bean Validation** (Jakarta Validation, impl. **Hibernate Validator**) + adnotacje
> ograniczeń na DTO; `@Valid` przy `@RequestBody` uruchamia walidację ciała (błąd → `MethodArgumentNotValidException`),
> `@Validated` na klasie włącza walidację `@PathVariable`/`@RequestParam` i metod (błąd → `ConstraintViolationException`).
> Wyjątki łapiesz centralnie w `@RestControllerAdvice` + `@ExceptionHandler` i mapujesz na spójny **ProblemDetail**
> (RFC 7807, `application/problem+json`) — nigdy stacktrace ani wewnętrzne szczegóły.

## 1. Co — definicja i API

**Bean Validation** to standard Jakarta (dawniej **JSR-380**, javax → jakarta) opisujący *deklaratywne* ograniczenia
przez adnotacje. **Hibernate Validator** to referencyjna *implementacja* (Spring Boot dokłada ją przez
`spring-boot-starter-validation`). Ograniczenia opisują *kontrakt* danych, a nie logikę — silnik je egzekwuje.

**Adnotacje ograniczeń** (na polach/parametrach DTO):

| Adnotacja | Znaczenie | Typy | Pułapka |
|---|---|---|---|
| `@NotNull` | wartość ≠ `null` | dowolny | `""` i `"  "` przechodzą |
| `@NotEmpty` | ≠ `null` **i** rozmiar > 0 | `String`, `Collection`, `Map`, tablica | `"  "` (spacje) **przechodzi** |
| `@NotBlank` | ≠ `null` i po `trim()` długość > 0 | **tylko `String`** | najostrzejsze dla stringów |
| `@Size(min,max)` | długość/rozmiar w zakresie | `String`, kolekcje | `null` przechodzi (użyj z `@NotNull`) |
| `@Min`/`@Max` | wartość liczbowa ≥/≤ | liczby | dla `BigDecimal` → `@DecimalMin/@DecimalMax` |
| `@Positive`/`@PositiveOrZero` | > 0 / ≥ 0 | liczby | `null` przechodzi |
| `@Email` | format e-mail | `String` | luźny regex — nie gwarantuje istnienia |
| `@Pattern(regexp)` | dopasowanie do regex | `String` | kosztowne regexy = ReDoS |
| `@Past`/`@Future` / `@PastOrPresent` | data w przeszłości/przyszłości | `LocalDate`, `Instant`… | zależne od strefy/zegara |

Kluczowa różnica: **`@NotNull` ⊂ `@NotEmpty` ⊂ `@NotBlank`** pod względem restrykcyjności dla `String`.
Dla stringów prawie zawsze chcesz `@NotBlank`.

```java
public record CreateUserRequest(
    @NotBlank @Size(max = 50) String name,
    @NotNull @Email String email,
    @Min(18) @Max(120) int age,
    @Past LocalDate birthDate,
    @Valid Address address   // kaskada na obiekt zagnieżdżony
) {}
```

**`@Valid` vs `@Validated`:**
- `@Valid` — z Jakarta Validation; uruchamia walidację obiektu, wspiera **kaskadę** (walidację zagnieżdżonych pól),
  **nie** obsługuje grup.
- `@Validated` — z Springa (`org.springframework.validation.annotation`); dodatkowo pozwala wskazać **grupy
  walidacji** oraz włącza **walidację na poziomie metody** (przez proxy) gdy postawisz ją na klasie/beanie.

## 2. Jak — gdzie walidacja się wyzwala pod spodem

Miejsce wyzwolenia decyduje o *typie wyjątku* — to fundament poprawnego mapowania błędów.

**a) `@Valid @RequestBody` w kontrolerze (ciało JSON):**
`RequestResponseBodyMethodProcessor` po zdeserializowaniu ciała woła `DataBinder` z `Validator`em. Naruszenia trafiają
do `BindingResult`. Jeśli argument tuż po DTO **nie** jest `BindingResult`/`Errors`, Spring rzuca
**`MethodArgumentNotValidException`** (rozszerza `BindException`). Zawiera `BindingResult` z listą `FieldError`
(pole + odrzucona wartość + komunikat) i `ObjectError` (błędy klasowe). Domyślnie → HTTP **400**.

**b) Kaskada (obiekty zagnieżdżone):**
Walidacja **nie schodzi automatycznie** do zagnieżdżonych obiektów/elementów kolekcji. Trzeba jawnie oznaczyć pole
`@Valid` (np. `@Valid Address address`, `List<@Valid Item> items`). Ograniczenia na typie generycznym (`List<@NotBlank String>`)
też wymagają, by kontener był walidowany.

**c) `@PathVariable` / `@RequestParam` (`@Validated` na klasie kontrolera):**
To są *parametry metody*, nie obiekt-DTO — walidację włącza **`@Validated` na klasie** kontrolera. Wtedy Spring tworzy
`MethodValidationInterceptor` (proxy AOP), który waliduje argumenty przez `ExecutableValidator`. Naruszenie →
**`ConstraintViolationException`** (inny wyjątek niż w pkt a!), niosący `Set<ConstraintViolation>` z `propertyPath`.
Bez własnego handlera domyślnie → HTTP **500**, więc trzeba go obsłużyć.

```java
@Validated                                   // włącza walidację parametrów metody
@RestController
class ProductController {
    @GetMapping("/products/{id}")
    Product get(@PathVariable @Positive Long id,
                @RequestParam @Size(max = 20) String q) { ... }
}
```

> Uwaga (Spring 6.1 / Boot 3.2+): tzw. *method validation for controllers* ujednolica to i potrafi rzucać
> `HandlerMethodValidationException` zamiast `ConstraintViolationException` dla parametrów kontrolera — sprawdź
> zachowanie w swojej wersji i obsłuż właściwy typ.

**d) Walidacja metod serwisu (przez proxy):**
`@Validated` na beanie serwisowym → Spring owija go proxy; wywołania metod z `@Valid`/ograniczeniami na parametrach
lub `@NotNull` na zwracanej wartości są walidowane przy każdym wywołaniu. **Ograniczenie proxy**: samowywołanie
(`this.method()`) omija proxy → walidacja nie zadziała (jak przy `@Transactional`).

**e) Własne ograniczenia (`@Constraint` + `ConstraintValidator`):**
```java
@Documented
@Constraint(validatedBy = StrongPasswordValidator.class)
@Target({FIELD, PARAMETER})
@Retention(RUNTIME)
public @interface StrongPassword {
    String message() default "{validation.password.weak}";  // klucz i18n
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

class StrongPasswordValidator implements ConstraintValidator<StrongPassword, String> {
    public boolean isValid(String value, ConstraintValidatorContext ctx) {
        if (value == null) return true;      // null zostaw dla @NotNull (separacja odpowiedzialności)
        return value.length() >= 12 && value.chars().anyMatch(Character::isDigit);
    }
}
```
Metody `message()`, `groups()`, `payload()` są **obowiązkowe** w adnotacji ograniczenia. Validator może być
wstrzykiwany Springiem (dostęp do beanów, np. sprawdzenie unikalności w bazie — ostrożnie z wydajnością).

**f) Grupy walidacji** — pozwalają walidować różne podzbiory ograniczeń w różnych kontekstach (np. `OnCreate`
wymaga `id == null`, `OnUpdate` wymaga `@NotNull id`):
```java
interface OnCreate {}
interface OnUpdate {}
record UserDto(@Null(groups=OnCreate.class) @NotNull(groups=OnUpdate.class) Long id, ...) {}

@PostMapping void create(@Validated(OnCreate.class) @RequestBody UserDto dto) { ... }
```
Grupy działają **tylko z `@Validated`**, nie z `@Valid`.

## 3. Dlaczego / kiedy — kontrakt błędu i pułapki

**Globalna obsługa błędów** = jedno miejsce mapujące wyjątki na odpowiedzi HTTP:

- **`@RestControllerAdvice`** (= `@ControllerAdvice` + `@ResponseBody`) — klasa z metodami `@ExceptionHandler`,
  która obejmuje wszystkie kontrolery. Tu tłumaczysz wyjątek → status + body.
- **`ResponseEntityExceptionHandler`** — bazowa klasa Springa obsługująca standardowe wyjątki MVC
  (`MethodArgumentNotValidException`, `HttpMessageNotReadableException`, 404 itd.). Dziedzicząc po niej i nadpisując
  `handle...`, dostajesz spójne mapowanie „za darmo”, dokładając własne handlery dla wyjątków domenowych.

**ProblemDetail i RFC 7807** (od Spring 6 / Boot 3): standardowy format błędu `application/problem+json` z polami
`type`, `title`, `status`, `detail`, `instance` + dowolne rozszerzenia (`properties`). Klasa `ProblemDetail` jest
wbudowana; włączenie `spring.mvc.problemdetails.enabled=true` sprawia, że `ResponseEntityExceptionHandler` sam zwraca
ProblemDetail. To zamiennik ad-hoc map błędów — szczegóły kontraktu → [[wiedza/07-api-design/rest]].

**Kontrakt błędu walidacji** — klient musi wiedzieć *które pole* jest złe i *dlaczego*. Buduj mapę `field → message`
z `BindingResult.getFieldErrors()` i wkładaj jako rozszerzenie ProblemDetail:

```java
@RestControllerAdvice
class ApiExceptionHandler extends ResponseEntityExceptionHandler {

    @Override                                  // ciało @RequestBody
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex, HttpHeaders h, HttpStatusCode s, WebRequest r) {
        var pd = ProblemDetail.forStatusAndDetail(BAD_REQUEST, "Walidacja nie powiodła się");
        pd.setTitle("Validation failed");
        pd.setType(URI.create("https://api.example.com/errors/validation"));
        Map<String, String> errors = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors()
          .forEach(fe -> errors.put(fe.getField(), fe.getDefaultMessage()));
        pd.setProperty("errors", errors);
        pd.setProperty("timestamp", Instant.now());
        return ResponseEntity.badRequest().contentType(APPLICATION_PROBLEM_JSON).body(pd);
    }

    @ExceptionHandler(ConstraintViolationException.class)   // @PathVariable/@RequestParam
    ProblemDetail handleConstraintViolation(ConstraintViolationException ex) {
        var pd = ProblemDetail.forStatusAndDetail(BAD_REQUEST, "Niepoprawne parametry");
        Map<String, String> errors = new LinkedHashMap<>();
        ex.getConstraintViolations()
          .forEach(v -> errors.put(v.getPropertyPath().toString(), v.getMessage()));
        pd.setProperty("errors", errors);
        return pd;
    }
}
```

**i18n komunikatów** — komunikat w adnotacji jako klucz `{validation.email.invalid}` rozwiązywany z
`messages.properties` (`ValidatorFactory` z `MessageSource`). Wersje `messages_pl.properties`, `messages_en.properties`
dobierane po `Accept-Language`. Wstrzyknij Springowy `MessageSource` do `LocalValidatorFactoryBean`, by ograniczenia
korzystały z tych samych plików co reszta aplikacji.

**Pułapki:**
- **Nie zwracaj stacktrace ani nazw klas/tabel** — to wyciek informacji (attack surface). Loguj szczegóły po stronie
  serwera z `traceId`, klientowi dawaj tylko `detail` + kod.
- **Dwa różne wyjątki** dla ciała (`MethodArgumentNotValidException`) i parametrów (`ConstraintViolationException`) —
  łatwo obsłużyć tylko jeden i dostać 500 na drugim.
- **`@Valid` bez kaskady** — zagnieżdżone DTO nie są walidowane bez `@Valid` na polu.
- **Samowywołanie omija proxy** — walidacja metod serwisu nie zadziała przy `this.foo()`.
- **`@Size`/`@Positive` na `null`** przechodzą — łącz z `@NotNull`.
- **Kolejność vs deserializacja** — `@Pattern`/`@Email` nie zabezpieczą przed błędem parsowania JSON
  (`HttpMessageNotReadableException`) — to osobny handler.
- **Nie waliduj encji JPA jako DTO** — trzymaj ograniczenia wejścia na warstwie API (DTO), nie mieszaj z modelem domeny.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

REST API rejestracji użytkownika: kontroler przyjmuje `@Valid @RequestBody CreateUserRequest`, ograniczenia
(`@NotBlank`, `@Email`, `@Past`) odrzucają złe dane *zanim* dotrą do serwisu. `@RestControllerAdvice` zamienia
`MethodArgumentNotValidException` na `ProblemDetail` z mapą `field→message` i statusem 400, a
`ConstraintViolationException` z endpointów `GET /users/{id}` (`@Positive`) — na analogiczny 400. Frontend czyta
`errors` i podświetla konkretne pola formularza. Komunikaty pochodzą z `messages_pl.properties`, więc UI dostaje
polskie teksty bez hardkodowania w kodzie.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię różnicę `@NotNull` / `@NotEmpty` / `@NotBlank` i kiedy który.
- [ ] Wiem, że `@Valid` (kaskada) ≠ `@Validated` (grupy + walidacja metod) i skąd pochodzą.
- [ ] Wskażę, który wyjątek leci dla `@RequestBody`, a który dla `@PathVariable`/`@RequestParam`.
- [ ] Napiszę `@RestControllerAdvice` zwracający ProblemDetail dla walidacji.
- [ ] Zbuduję własne ograniczenie (`@Constraint` + `ConstraintValidator`).
- [ ] Wiem, czemu nie zwracać stacktrace i jak działa i18n komunikatów.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Jak Spring uruchamia walidację `@RequestBody` i skąd bierze się `MethodArgumentNotValidException`?
- [ ] Dlaczego walidacja `@PathVariable` wymaga `@Validated` na klasie i daje inny wyjątek?
- [ ] Jak działa walidacja metod serwisu (proxy) i czemu samowywołanie ją omija?
- [ ] Czemu kaskada wymaga jawnego `@Valid` na zagnieżdżonym polu?
- [ ] Co daje `ResponseEntityExceptionHandler` i jak włączyć ProblemDetail?
- [ ] Jak komunikaty łączą się z `messages.properties` i `Accept-Language`?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „`@Valid` vs `@Validated` — różnice?”
- [ ] „Jak zrobisz spójny format błędów walidacji w API?” (RFC 7807 / ProblemDetail)
- [ ] „Gdzie kończy walidacja i jak globalnie łapiesz wyjątki?” (`@RestControllerAdvice`)
- [ ] „`@NotBlank` vs `@NotEmpty` vs `@NotNull`?”
- [ ] „Jak walidujesz `@RequestParam`/`@PathVariable`?” (`@Validated` + `ConstraintViolationException`)
- [ ] „Jak zrobisz własną adnotację walidacyjną?”

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Co to Bean Validation i jaka jest referencyjna implementacja?;Standard Jakarta Validation (dawniej JSR-380) do deklaratywnych ograniczeń przez adnotacje; impl. to Hibernate Validator.
Różnica @NotNull vs @NotEmpty vs @NotBlank?;@NotNull: nie null. @NotEmpty: nie null i rozmiar>0 (spacje przechodzą). @NotBlank: tylko String, nie null i po trim niepusty.
Czy @NotEmpty przepuszcza string ze spacjami?;Tak — "   " ma rozmiar>0. Do stringów użyj @NotBlank.
Czy @Size i @Positive odrzucają null?;Nie, null przechodzi — trzeba dołożyć @NotNull.
@Valid vs @Validated — źródło i różnica?;@Valid (Jakarta): walidacja + kaskada, bez grup. @Validated (Spring): dodatkowo grupy walidacji i walidacja metod przez proxy.
Który wyjątek leci przy @Valid @RequestBody?;MethodArgumentNotValidException (rozszerza BindException), zawiera BindingResult z FieldError.
Jak włączyć walidację @PathVariable/@RequestParam?;Postawić @Validated na klasie kontrolera — Spring waliduje parametry przez proxy.
Który wyjątek leci przy walidacji @RequestParam/@PathVariable?;ConstraintViolationException z Set<ConstraintViolation> (domyślnie 500, trzeba obsłużyć).
Dlaczego kaskada walidacji wymaga @Valid na polu?;Walidacja nie schodzi automatycznie do zagnieżdżonych obiektów — trzeba jawnie oznaczyć pole/element @Valid.
Jak działa walidacja metod serwisu?;@Validated na beanie owija go proxy AOP walidującym parametry/wynik; samowywołanie (this.foo()) omija proxy.
Jakie metody musi mieć własna adnotacja @Constraint?;message(), groups() i payload() (plus @Constraint(validatedBy=...)).
Co powinien zwrócić ConstraintValidator dla null?;Zwykle true — null zostaw dla @NotNull (separacja odpowiedzialności).
Co robi @RestControllerAdvice?;To @ControllerAdvice+@ResponseBody: centralna klasa z @ExceptionHandler mapująca wyjątki na odpowiedzi HTTP dla wszystkich kontrolerów.
Do czego ResponseEntityExceptionHandler?;Bazowa klasa obsługująca standardowe wyjątki MVC (np. MethodArgumentNotValidException); dziedziczysz i nadpisujesz handle...
Co to ProblemDetail / RFC 7807?;Standardowy format błędu application/problem+json z polami type,title,status,detail,instance + rozszerzenia; w Springu klasa ProblemDetail.
Jak zbudować kontrakt błędu walidacji?;Zmapować BindingResult.getFieldErrors() na mapę field->message i dodać jako property ProblemDetail.
Jak działa i18n komunikatów walidacji?;message jako klucz {klucz} rozwiązywany z messages.properties przez MessageSource; wersje _pl/_en dobierane po Accept-Language.
Główna pułapka odpowiedzi błędu?;Nie zwracać stacktrace ani nazw klas/tabel — wyciek informacji; loguj szczegóły z traceId, klientowi dawaj tylko detail+kod.
Do jakiego statusu HTTP mapuje się błąd walidacji wejścia?;400 Bad Request.
Czemu @Pattern/@Email nie chroni przed błędem parsowania JSON?;Bo błąd deserializacji to HttpMessageNotReadableException (osobny handler), przed uruchomieniem walidacji ograniczeń.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/05-spring/mvc-rest]] · [[wiedza/01-jezyk/wyjatki]] · [[wiedza/07-api-design/rest]]
