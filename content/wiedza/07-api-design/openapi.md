---
temat: "OpenAPI / Swagger — kontrakt API i generowanie klienta"
faza: 7
status: nieopanowany
priorytet: 🔴
tags: [java, api, openapi, angular]
powiazane: ["[[wiedza/07-api-design/rest]]", "[[wiedza/07-api-design/dto-mapstruct]]", "[[wiedza/07-api-design/rest]]"]
---

# OpenAPI / Swagger — kontrakt API i generowanie klienta

> **TL;DR:** **OpenAPI** to standard (JSON/YAML) opisujący REST API — jego `paths`, `operations`, `schemas`, `parameters`, `responses`, `security`. Z tego jednego dokumentu-**kontraktu** dostajesz **Swagger UI** (interaktywna dokumentacja) i **generowanego klienta** (np. typy + serwisy Angular). W fullstacku Angular+Java to eliminuje ręczne przepisywanie modeli i **rozjazd front-back**.

## 1. Co — specyfikacja, dokument i narzędzia

**OpenAPI Specification (OAS)** — formalny, języko-niezależny opis REST API. Historycznie nazywany **Swagger** (do wersji 2.0); od 3.0 nazwa to **OpenAPI**, a „Swagger" oznacza dziś **rodzinę narzędzi** (Swagger UI, Swagger Editor, Swagger Codegen) wokół tej specyfikacji. Bieżące wersje to OpenAPI 3.0.x / 3.1.x (3.1 jest w pełni zgodne z JSON Schema).

Dokument to plik **JSON lub YAML** o ustalonej strukturze:

```yaml
openapi: 3.0.3
info:              # metadane: tytuł, wersja API, opis
  title: Orders API
  version: 1.2.0
servers:           # bazowe URL-e (dev/prod)
  - url: https://api.example.com
paths:             # ścieżki → operacje (GET/POST/PUT/DELETE...)
  /orders/{id}:
    get:
      operationId: getOrder          # nazwa metody w generowanym kliencie
      parameters:                    # path/query/header/cookie
        - name: id
          in: path
          required: true
          schema: { type: integer, format: int64 }
      responses:                     # kody odpowiedzi + schematy body
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '404': { description: Not found }
      security:
        - bearerAuth: []
components:          # reużywalne definicje
  schemas:          # modele danych (DTO) — źródło typów TS
    Order:
      type: object
      required: [id, status]
      properties:
        id:     { type: integer, format: int64 }
        status: { type: string, enum: [NEW, PAID, SHIPPED] }
  securitySchemes:  # sposoby uwierzytelniania
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

Kluczowe pojęcia:
- **paths / operations** — mapa `URL → metoda HTTP → operacja`. `operationId` staje się nazwą metody w wygenerowanym kliencie (dlatego warto go ustawiać ręcznie i sensownie).
- **components** — słownik reużywalnych elementów (schematy, parametry, odpowiedzi, `securitySchemes`), wskazywanych przez `$ref`. Unika duplikacji.
- **schemas** — modele danych oparte na JSON Schema (typy, `enum`, `required`, `format`, `nullable`). To one generują interfejsy TypeScript i klasy DTO.
- **security schemes** — deklaracja mechanizmów auth (HTTP bearer/basic, `apiKey`, OAuth2, OpenID Connect); wskazywane per-operacja lub globalnie.

**Swagger UI** — renderuje spec do interaktywnej strony HTML: wykaz endpointów, schematy, i przycisk **„Try it out"** wykonujący realne żądania z przeglądarki. **Swagger/OpenAPI Generator** — narzędzia CLI generujące ze spec kod: serwery (stuby), klienty (Java, TypeScript, itd.) i dokumentację.

## 2. Jak — springdoc-openapi i generowanie klienta pod spodem

### Strona Java (Spring Boot 3): springdoc-openapi

W Spring Boot 3 (jakarta.*) standardem jest **springdoc-openapi** (Springfox jest martwy). Dodajesz zależność:

```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.x</version>
</dependency>
```

Startuje i masz „za darmo":
- **spec** pod `/v3/api-docs` (JSON) oraz `/v3/api-docs.yaml`,
- **Swagger UI** pod `/swagger-ui.html`.

**Pod spodem:** springdoc skanuje kontekst Springa — kontrolery (`@RestController`), mapowania (`@GetMapping`…), typy zwracane, parametry — i **na starcie / przy pierwszym żądaniu** buduje model OpenAPI przez introspekcję (reflection). Adnotacje `@Operation`, `@Schema`, `@ApiResponse`, `@Parameter` (z pakietu `io.swagger.v3.oas.annotations`) tylko **wzbogacają** to, czego nie da się wywnioskować z sygnatury: opisy, przykłady, kody błędów, formaty. To jest **code-first**: kod jest źródłem, spec jest produktem.

### Strona Angular: generowanie klienta TS (killer feature)

Z `openapi.json` (pobranego z `/v3/api-docs`) generujesz **typy i serwisy Angular**. Dwa popularne narzędzia:

- **openapi-generator** (generator `typescript-angular`) — pełny, konfigurowalny, generuje `*.service.ts` (na `HttpClient`), modele, `ApiModule`, konfigurację.
- **ng-openapi-gen** — lżejszy, ukierunkowany wyłącznie na Angular; czytelny output, dobrze traktuje `enum` i grupowanie po tagach.

**Efekt:** DTO backendu i modele frontu pochodzą z **jednego źródła**. Zmieniasz pole w Javie → regenerujesz klienta → **kompilator TypeScript** natychmiast pokazuje, gdzie front się rozjechał. Koniec z ręcznym przepisywaniem interfejsów i cichym driftem typów.

**Pod spodem** generator: (1) parsuje i waliduje spec, (2) rozwija `$ref`, (3) mapuje typy OpenAPI → TypeScript (`integer`→`number`, `string+enum`→union/`enum`, `format: date-time`→`string`/`Date`), (4) tworzy per-tag serwis z metodą na każde `operationId`, (5) składa URL z `paths` i wstrzykuje `HttpClient`.

### Wersjonowanie kontraktu i zgodność

Spec ma `info.version` (wersja API, semantyczna). Zmiany dziel na **backward-compatible** (dodanie opcjonalnego pola, nowego endpointu) i **breaking** (usunięcie pola, zmiana typu, nowe `required`) — te ostatnie wymagają nowej wersji API ([[wiedza/07-api-design/rest]]). Wersjonuj sam plik spec w repo (git), bo to on jest kontraktem.

**Contract testing** (świadomość): weryfikacja, że implementacja **nie odbiega** od kontraktu — np. narzędzia jak Pact (consumer-driven) albo walidatory sprawdzające odpowiedzi względem schematów OpenAPI w testach. Chroni przed sytuacją, gdy spec mówi jedno, a serwer zwraca drugie.

## 3. Dlaczego / kiedy — code-first vs contract-first, pułapki

Dwa fundamentalnie różne podejścia:

| | **Code-first** | **Contract-first / design-first** |
|---|---|---|
| Źródło prawdy | kod Javy | ręcznie pisany plik `openapi.yaml` |
| Kierunek | kod → spec (springdoc) | spec → kod/interfejsy (generator) |
| Zalety | szybki start, mniej narzędzi, spec zawsze „z kodu" | **kontrakt jako źródło prawdy**, front i back pracują **równolegle** (front generuje mocki/typy zanim back gotowy), łatwy review kontraktu, spójność w wielu serwisach |
| Wady | spec **rozjeżdża się** z intencją bez dyscypliny adnotacji; trudniej uzgodnić z zespołem front | więcej narzędzi/dyscypliny; trzeba pilnować, by implementacja trzymała się spec |

**Kiedy co:**
- **Code-first** — pojedynczy zespół, małe/średnie API, szybka iteracja, spec głównie do dokumentacji.
- **Contract-first** — zespoły rozdzielone (front/back), publiczne API, wiele konsumentów, gdy kontrakt musi być stabilny i uzgodniony **zanim** ktokolwiek napisze kod.

**Pułapki:**
- **Code-first bez dyscypliny** — spec formalnie się generuje, ale opisy/kody błędów/przykłady są puste albo błędne; typy generyczne (`Object`, surowe mapy, `Page<T>` bez konfiguracji) generują bezużyteczny klient. Spec „istnieje", ale **nie odzwierciedla rzeczywistej semantyki**.
- **Brak `operationId`** → generator wymyśla brzydkie nazwy metod (`ordersIdGet`), które psują się przy zmianie ścieżki.
- **Rozjazd typów daty/liczb** — `LocalDateTime` bez `format: date-time`, `BigDecimal` jako `string` vs `number`; front dostaje zły typ.
- **Zapominanie o regeneracji** klienta w CI → front używa nieaktualnego kontraktu (dlatego generowanie warto wpiąć w pipeline).
- **Ujawnianie encji JPA** jako schematów zamiast dedykowanych DTO — wyciek pól, cykle, lazy proxy w spec ([[wiedza/07-api-design/dto-mapstruct]]).

**Dobre praktyki:** dokumentuj **wszystkie** istotne kody błędów (`400/401/403/404/409/500`) przez `@ApiResponse`; dodawaj **przykłady** (`@ExampleObject`) i **opisy** (`description`) do pól i operacji; ustawiaj `operationId`; używaj `enum` zamiast wolnych stringów; trzymaj spec w repo i wersjonuj.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Fullstack Angular + Spring Boot. Backend (code-first ze springdoc) wystawia `/v3/api-docs`. W buildzie frontu odpalasz generator klienta TS — CI regeneruje modele i serwisy przy każdej zmianie kontraktu. Deweloper Angulara woła `orderService.getOrder(id)` z pełnym typowaniem, a nie ręcznie klepie `HttpClient.get<any>()`.

**Fragment kontrolera ze springdoc:**

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @Operation(summary = "Pobierz zamówienie po ID",
               description = "Zwraca pojedyncze zamówienie lub 404, gdy nie istnieje.")
    @ApiResponse(responseCode = "200", description = "Znaleziono",
        content = @Content(schema = @Schema(implementation = OrderDto.class)))
    @ApiResponse(responseCode = "404", description = "Brak zamówienia", content = @Content)
    @GetMapping("/{id}")
    public OrderDto getOrder(
            @Parameter(description = "Identyfikator zamówienia", example = "42")
            @PathVariable Long id) {
        return orderService.get(id);
    }
}

public record OrderDto(
    @Schema(description = "ID zamówienia", example = "42") Long id,
    @Schema(description = "Status", example = "PAID") OrderStatus status
) {}
```

**Generowanie klienta TypeScript dla Angulara** (z uruchomionego backendu):

```bash
# openapi-generator (generator typescript-angular)
openapi-generator-cli generate \
  -i http://localhost:8080/v3/api-docs \
  -g typescript-angular \
  -o ./src/app/api \
  --additional-properties=ngVersion=17,useSingleRequestParameter=true

# albo ng-openapi-gen (config: openapi wskazuje na spec, output na katalog)
npx ng-openapi-gen --input http://localhost:8080/v3/api-docs --output ./src/app/api
```

---

## ✅ Kryteria opanowania
*Temat jest DOMKNIĘTY, gdy odpowiesz na wszystkie BEZ zaglądania.*

- [ ] Wyjaśnię, czym jest OpenAPI i jak ma się do Swaggera (spec vs narzędzia).
- [ ] Wymienię główne sekcje dokumentu: `paths`, `components/schemas`, `parameters`, `responses`, `securitySchemes`.
- [ ] Skonfiguruję springdoc-openapi i wskażę, gdzie jest spec i Swagger UI.
- [ ] Wygeneruję klienta TypeScript dla Angulara z `openapi.json`.
- [ ] Porównam code-first vs contract-first i wybiorę właściwe dla scenariusza.
- [ ] Wymienię co najmniej trzy pułapki (drift, brak `operationId`, złe typy dat).

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Skąd springdoc bierze spec — skanowanie kontekstu i introspekcja kontrolerów, adnotacje tylko wzbogacają.
- [ ] Jak generator mapuje typy OpenAPI na TypeScript (`integer`→`number`, `enum`→union) i skąd nazwy metod (`operationId`).
- [ ] Do czego służy `$ref` i `components` (reużycie, brak duplikacji).
- [ ] Co czyni zmianę kontraktu breaking, a co backward-compatible.
- [ ] Czym jest contract testing i przed czym chroni.

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Czym różni się OpenAPI od Swaggera?"
- [ ] „Code-first czy contract-first — co i kiedy?"
- [ ] „Jak zapewniasz zgodność modeli front-back w Angular+Java?" (generowany klient TS z jednego kontraktu)
- [ ] „Jak dokumentujesz błędy i przykłady w springdoc?"
- [ ] „Jak wersjonujesz API i co robisz przy zmianie breaking?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Czym jest OpenAPI?;Języko-niezależną specyfikacją (JSON/YAML) opisującą REST API: paths, operations, schemas, parameters, responses, security.
OpenAPI a Swagger — różnica?;OpenAPI to specyfikacja (dawniej zwana Swagger do 2.0); Swagger to dziś rodzina narzędzi (UI, Editor, Codegen) wokół niej.
Do czego służy sekcja components w OpenAPI?;Trzyma reużywalne definicje (schemas, parameters, responses, securitySchemes) wskazywane przez $ref, unikając duplikacji.
Co to Swagger UI?;Interaktywna dokumentacja HTML renderowana ze spec, z opcją "Try it out" wykonującą realne żądania.
Jakie narzędzie generuje spec i Swagger UI w Spring Boot 3?;springdoc-openapi (Springfox jest martwy dla Boot 3 / jakarta.*).
Pod jakimi ścieżkami springdoc wystawia spec i UI?;Spec pod /v3/api-docs (i .yaml), Swagger UI pod /swagger-ui.html.
Do czego służą adnotacje @Operation, @Schema, @ApiResponse, @Parameter?;Wzbogacają generowaną spec o opisy, przykłady, kody błędów i formaty, których nie da się wywnioskować z sygnatury.
Czym jest podejście code-first?;Kod Javy jest źródłem prawdy, a spec jest z niego generowana (springdoc).
Czym jest podejście contract-first / design-first?;Najpierw ręcznie piszesz spec YAML (kontrakt), a z niej generujesz kod/interfejsy.
Główna zaleta contract-first?;Kontrakt jest źródłem prawdy, front i back pracują równolegle, łatwy review i spójność między serwisami.
Killer feature OpenAPI dla fullstacka Angular+Java?;Z openapi.json generujesz typy i serwisy Angular (openapi-generator / ng-openapi-gen) — koniec ręcznego przepisywania modeli i driftu front-back.
Do czego służy operationId w OpenAPI?;Staje się nazwą metody w generowanym kliencie; warto ustawiać go ręcznie i sensownie.
Główna pułapka code-first bez dyscypliny?;Spec formalnie się generuje, ale rozjeżdża się z semantyką: puste opisy, brak kodów błędów, bezużyteczne typy generyczne.
Czym jest contract testing?;Weryfikacją, że implementacja nie odbiega od kontraktu (np. Pact, walidacja odpowiedzi względem schematów OpenAPI).
Co czyni zmianę kontraktu breaking?;Usunięcie pola, zmiana typu lub dodanie nowego required — wymaga nowej wersji API.
Jak generator mapuje typy OpenAPI na TypeScript?;np. integer->number, string+enum->union/enum, format:date-time->string/Date; z paths+operationId powstają serwisy na HttpClient.
Częsty błąd z typami przy generowaniu klienta?;Brak format: date-time dla LocalDateTime lub niejednoznaczny BigDecimal (string vs number) — front dostaje zły typ.
Dlaczego warto wpiąć generowanie klienta w CI?;Bez regeneracji front używa nieaktualnego kontraktu i cicho się rozjeżdża z backendem.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/07-api-design/rest]] · [[wiedza/07-api-design/dto-mapstruct]] · [[wiedza/07-api-design/rest]]
