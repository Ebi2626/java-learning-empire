---
temat: "Proxy i AOP w Springu"
faza: 5
status: nieopanowany
priorytet: 🔴
tags: [java, spring, aop]
powiazane: ["[[wiedza/05-spring/ioc-di]]", "[[wiedza/06-persystencja/transakcje]]", "[[wiedza/05-spring/ioc-di]]"]
---

# Proxy i AOP w Springu

> **TL;DR:** Spring **owija** (proxy) Twój bean w wygenerowaną klasę pośredniczącą, która przechwytuje wywołania metod i dokłada **cross-cutting concerns** (transakcje, cache, async, security, logi) *wokół* Twojej logiki. Stąd `@Transactional`/`@Cacheable`/`@Async` „po prostu działają". Proxy powstaje albo z **JDK dynamic proxy** (na interfejsie), albo z **CGLIB** (podklasa klasy — domyślny w Spring Boot). Kluczowa pułapka: **self-invocation** — wywołanie takiej metody przez `this` z innej metody tej samej klasy **omija proxy** i adnotacja nie zadziała.

## 1. Co — definicja i API

**Proxy** to obiekt-pośrednik o tym samym typie co Twój bean, który przechwytuje wywołania metod, robi coś *przed/po/zamiast* i deleguje do prawdziwego obiektu (*target*). To realizacja wzorca **Proxy** + techniki **AOP (Aspect-Oriented Programming)**.

**Po co to?** Żeby wyciągnąć **cross-cutting concerns** — logikę, która przecina wiele klas i nie jest „biznesem":
- **transakcje** (`@Transactional`) — begin / commit / rollback,
- **cache** (`@Cacheable`, `@CacheEvict`) — sprawdź cache zanim wejdziesz w metodę,
- **async** (`@Async`) — wykonaj w innym wątku,
- **security** (`@PreAuthorize`, `@Secured`) — sprawdź uprawnienia,
- **logowanie / metryki / retry** (`@Retryable`, `@Observed`).

Bez proxy każda metoda repozytorium musiałaby ręcznie otwierać transakcję, łapać wyjątki i robić rollback — logika biznesowa utonęłaby w boilerplate. AOP pozwala **zadeklarować** te aspekty adnotacją, a Spring wstrzykuje je w tle.

**Słownik AOP (must-know):**
- **aspect** — moduł łączący cross-cutting concern (np. klasa `@Aspect` z logiką transakcji/logowania).
- **join point** — punkt w wykonaniu programu, w który *można* wpiąć advice. W Spring AOP join point = **wywołanie metody** (tylko to).
- **pointcut** — *predykat* wybierający, które join pointy dotyczyć (np. „wszystkie metody `@Transactional`", „wszystko w `com.app.service..`"). Definiowany wyrażeniem, np. `execution(* com.app.service.*.*(..))`.
- **advice** — kod uruchamiany w join poincie. Rodzaje:
  - `@Before` — przed metodą,
  - `@After` (*finally*) — zawsze po (jak `finally`),
  - `@AfterReturning` — po udanym zwrocie (dostaje wynik),
  - `@AfterThrowing` — gdy metoda rzuciła wyjątek,
  - `@Around` — **najpotężniejszy**: otacza wywołanie, sam decyduje czy i kiedy wywołać `proceed()`, może zmienić argumenty, wynik, złapać wyjątek. Na nim opierają się transakcje i cache.
- **weaving** — proces „wplecenia" aspektów w kod. Spring robi to **w runtime** przez proxy; AspectJ może **w czasie kompilacji** lub **ładowania klas**.

## 2. Jak — JDK vs CGLIB, przechwytywanie pod spodem

### Dwa rodzaje proxy

**JDK dynamic proxy** (`java.lang.reflect.Proxy`):
- Bazuje na **INTERFEJSIE** — target musi implementować interfejs; proxy implementuje ten sam interfejs.
- Generowany obiekt jest *osobny* od targetu i deleguje przez `InvocationHandler`.
- Ograniczenie: **działa tylko na metodach z interfejsu**. `MyServiceImpl injected as MyService` → OK, ale wstrzyknięcie po typie konkretnej klasy się nie uda (proxy nie jest jej instancją).

**CGLIB** (dziś biblioteka spakowana w `spring-core`):
- **Dziedziczy po KLASIE** — tworzy w runtime **podklasę** targetu i **nadpisuje** metody, wołając target przez `MethodInterceptor`.
- Nie wymaga interfejsu → **domyślny w Spring Boot** (`spring.aop.proxy-target-class=true` domyślnie). Dzięki temu wstrzykiwanie po typie klasy działa i mniej się kłuje z kodem bez interfejsów.
- **Ograniczenia CGLIB** (bo to dziedziczenie!):
  - nie zaproksuje klasy `final` (nie da się po niej dziedziczyć),
  - nie przechwyci metod `final` (nie da się nadpisać) ani `private`/`static` (nie są dziedziczone/wirtualne) — adnotacja na nich jest **cicho ignorowana**,
  - klasa musi mieć dostępny (choćby domyślny) konstruktor; z modern CGLIB/`Objenesis` jest to łagodzone.

**Kiedy który / jak wymusić:**
- Domyślnie w Spring Boot → **CGLIB**.
- Wymuś CGLIB globalnie: `spring.aop.proxy-target-class=true`, lub `@EnableAspectJAutoProxy(proxyTargetClass = true)`, lub `@EnableTransactionManagement(proxyTargetClass = true)`.
- Wymuś JDK proxy: ustaw `proxy-target-class=false` (i wtedy bean **musi** mieć interfejs, a wstrzykiwać go trzeba **po interfejsie**).

### Jak `@Transactional` NAPRAWDĘ działa (krok po kroku)

1. Przy starcie kontenera **`BeanPostProcessor`** (`AbstractAutoProxyCreator`, np. `InfrastructureAdvisorAutoProxyCreator`) skanuje beany. Widząc bean z metodami pasującymi do pointcutu transakcji, **podmienia** go w kontenerze na **proxy**.
2. Kiedy wołasz `userService.save(u)`, tak naprawdę wołasz **proxy**.
3. Proxy (advice `TransactionInterceptor`, typu `@Around`):
   - pyta `PlatformTransactionManager` o transakcję (nowa vs istniejąca — wg *propagation*),
   - **otwiera** transakcję (begin),
   - `proceed()` — deleguje do **prawdziwej** metody (target),
   - po powrocie: **commit**,
   - jeśli poleciał `RuntimeException`/`Error` (domyślnie): **rollback**,
   - zawsze na końcu sprząta (zwalnia/wznawia zawieszoną transakcję).
4. Twoja metoda nic o tym nie wie — dostaje gotowe połączenie z otwartą transakcją.

To samo schematycznie robi `@Cacheable` (sprawdź cache przed `proceed()`, może w ogóle nie wywołać targetu) i `@Async` (oddaj wykonanie do `Executor` zamiast wołać synchronicznie).

## 3. Dlaczego / kiedy — self-invocation i inne pułapki

### ⚠️ Self-invocation — KLUCZOWA pułapka

Advice siedzi **tylko w proxy**. Jeśli metoda beana wywoła **inną metodę tej samej klasy przez `this`**, wywołanie idzie *wewnątrz targetu*, **omijając proxy** → adnotacja (`@Transactional`, `@Cacheable`, `@Async`) **nie zadziała**.

```java
@Service
public class OrderService {

    public void process(Order o) {
        validate(o);
        save(o);          // this.save(o) → OMIJA proxy! Brak transakcji!
    }

    @Transactional
    public void save(Order o) {   // transakcja tylko gdy woła się przez proxy z ZEWNĄTRZ
        repository.save(o);
    }
}
```

Powyżej `save()` uruchomione z `process()` **nie ma transakcji** — bo `process()` biegnie w targecie, a `this.save()` nie przechodzi przez `TransactionInterceptor`. Ten sam problem dotyka `@Async` (poleci synchronicznie) i `@Cacheable` (cache pominięty).

**Obejścia:**
1. **Refaktor do osobnego beana** (najczystsze) — przenieś `save()` do innego serwisu i wstrzyknij go; wywołanie idzie przez proxy tamtego beana.
2. **Wstrzyknięcie siebie** (self-injection) — wstrzyknij własny proxy i wołaj przez niego:
   ```java
   @Autowired @Lazy private OrderService self;
   public void process(Order o) { self.save(o); }   // przez proxy → działa
   ```
3. **`ApplicationContext.getBean(OrderService.class).save(o)`** — brzydkie, ale zwraca proxy.
4. **`AopContext.currentProxy()`** — wymaga `exposeProxy = true` w `@EnableAspectJAutoProxy`.
5. **AspectJ** (load-time/compile-time weaving) — wplata advice bezpośrednio w bytecode klasy, więc self-invocation *też* jest przechwytywane (bo nie ma osobnego proxy).

### Inne ograniczenia Spring AOP

- **Tylko metody `public`** beanów (i przez CGLIB `protected` w praktyce ograniczone). `private`/`final`/`static` → nie działają.
- **Tylko beany zarządzane przez kontener** — obiekt utworzony `new` nie ma proxy.
- **Tylko wywołania przez proxy** (patrz self-invocation).
- **Kolejność aspektów** — gdy nakłada się kilka (np. `@Transactional` + własny `@Around`), steruj `@Order`.

### Spring AOP vs AspectJ

| | **Spring AOP** | **AspectJ** |
|---|---|---|
| Weaving | runtime (proxy) | compile-time / load-time (bytecode) |
| Join pointy | tylko wywołania metod (public, na beanach) | metody, konstruktory, pola, `this`, itd. |
| Self-invocation | **nie** przechwytuje | **przechwytuje** |
| Koszt | brak specjalnej kompilacji, wygodny | wymaga wplatania (weaver/agent) |
| Kiedy | 90% przypadków (transakcje, cache, security) | gdy potrzeba dowolnych join pointów lub self-invocation |

Reguła: **zacznij od Spring AOP**; sięgnij po AspectJ tylko gdy naprawdę potrzebujesz jego mocy.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Najczęściej **nie piszesz** aspektów — konsumujesz gotowe (`@Transactional` w warstwie serwisu). Ale własny aspekt bywa idealny do przekrojowego logowania/metryk. Aspekt mierzący czas wykonania metod serwisów:

```java
@Aspect
@Component
public class TimingAspect {

    private static final Logger log = LoggerFactory.getLogger(TimingAspect.class);

    @Around("execution(* com.app.service..*(..))")   // pointcut: wszystkie metody w pakiecie service
    public Object logTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed();                     // wywołaj prawdziwą metodę
        } finally {
            long ms = (System.nanoTime() - start) / 1_000_000;
            log.info("{} took {} ms", pjp.getSignature(), ms);
        }
    }
}
```

Ten sam mechanizm proxy + `BeanPostProcessor` obsłuży `@Transactional`, `@Cacheable`, `@Async` i Twój `TimingAspect` — wszystkie doklejają się „wokół" bez zaśmiecania logiki biznesowej. Gdy transakcja tajemniczo „nie działa", pierwsze pytanie brzmi: **czy to nie self-invocation?**

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, PO CO proxy (cross-cutting concerns) i jak to daje `@Transactional`/`@Cacheable`/`@Async`.
- [ ] Rozróżnię JDK dynamic proxy (interfejs) vs CGLIB (podklasa) i wiem, który jest domyślny w Spring Boot.
- [ ] Wymienię ograniczenia CGLIB (final, private, static) i jak wymusić dany typ proxy.
- [ ] Opiszę krok po kroku jak proxy realizuje `@Transactional` (begin → proceed → commit/rollback).
- [ ] Rozpoznam i naprawię problem self-invocation.
- [ ] Wskażę różnicę Spring AOP vs AspectJ i kiedy który.
- [ ] Napiszę własny `@Aspect` z `@Around` z głowy.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Czym różni się JDK dynamic proxy od CGLIB pod spodem (delegacja vs dziedziczenie)?
- [ ] Który komponent Springa tworzy proxy i kiedy? (BeanPostProcessor przy starcie kontenera)
- [ ] Co dokładnie robi `TransactionInterceptor` między wywołaniem a targetem?
- [ ] Dlaczego `this.metoda()` omija advice, a wstrzyknięty proxy — nie?
- [ ] Czemu adnotacja na metodzie `private`/`final`/`static` jest ignorowana?
- [ ] Czym jest `@Around` i po co `proceed()`?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Jak działa `@Transactional` pod spodem?"
- [ ] „Dlaczego moja transakcja/cache/async nie działa gdy wołam metodę z tej samej klasy?" (self-invocation)
- [ ] „JDK dynamic proxy vs CGLIB — różnice, który domyślny w Spring Boot?"
- [ ] „Wyjaśnij aspect / pointcut / advice / join point / weaving."
- [ ] „Spring AOP vs AspectJ — kiedy który?"
- [ ] „Napisz aspekt logujący czas wykonania metod."

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/05-spring.csv`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Po co Springowi proxy?;Do wpięcia cross-cutting concerns (transakcje, cache, async, security, logi) wokół metod bez zaśmiecania logiki biznesowej.
Na czym bazuje JDK dynamic proxy?;Na interfejsie — target musi implementować interfejs, a proxy implementuje ten sam interfejs i deleguje przez InvocationHandler.
Na czym bazuje CGLIB proxy?;Na dziedziczeniu — generuje podklasę klasy targetu i nadpisuje metody (MethodInterceptor).
Który typ proxy jest domyślny w Spring Boot?;CGLIB (proxy-target-class=true domyślnie).
Jak wymusić proxy JDK zamiast CGLIB?;proxy-target-class=false (i bean musi mieć interfejs, wstrzykiwany po interfejsie).
Jakie są ograniczenia CGLIB?;Nie proksuje klas final, nie przechwyci metod final, private ani static (bo to dziedziczenie/nadpisywanie metod wirtualnych).
Co to aspect w AOP?;Moduł łączący cross-cutting concern — np. klasa @Aspect z logiką transakcji lub logowania.
Co to join point w Spring AOP?;Punkt wykonania, w który można wpiąć advice; w Spring AOP to wyłącznie wywołanie metody.
Co to pointcut?;Predykat wybierający, które join pointy obejmuje advice (np. wyrażenie execution).
Co to advice i jakie są jego rodzaje?;Kod uruchamiany w join poincie: before, after, afterReturning, afterThrowing, around.
Który advice jest najpotężniejszy i dlaczego?;@Around — otacza wywołanie, sam decyduje czy i kiedy wywołać proceed(), może zmienić argumenty, wynik i złapać wyjątek.
Co to weaving?;Proces wplecenia aspektów w kod; Spring AOP robi to w runtime przez proxy, AspectJ w czasie kompilacji lub ładowania klas.
Jak proxy realizuje @Transactional?;Przechwytuje wywołanie, otwiera transakcję, deleguje przez proceed(), commituje po sukcesie, rollback po RuntimeException/Error.
Kto tworzy proxy w Springu i kiedy?;BeanPostProcessor (AbstractAutoProxyCreator) przy starcie kontenera podmienia bean na proxy.
Co to self-invocation i czemu jest problemem?;Wywołanie metody z adnotacją przez this z innej metody tej samej klasy omija proxy, więc advice (transakcja/cache/async) nie zadziała.
Jakie są obejścia self-invocation?;Refaktor do osobnego beana, self-injection (@Autowired @Lazy self), ApplicationContext.getBean, AopContext.currentProxy (exposeProxy), albo AspectJ.
Które metody obsługuje Spring AOP?;Tylko public metody beanów zarządzanych przez kontener, wołane przez proxy.
Spring AOP vs AspectJ — kluczowa różnica?;Spring AOP: runtime proxy, tylko wywołania metod, nie łapie self-invocation. AspectJ: compile/load-time weaving, dowolne join pointy, łapie self-invocation.
Dlaczego adnotacja na metodzie private jest ignorowana?;Bo CGLIB nadpisuje tylko dziedziczone metody wirtualne, a private/static/final nie są przez nią przechwytywane.
Po co proceed() w @Around?;Wywołuje prawdziwą metodę targetu; bez niego target się nie wykona, a advice może go pominąć (np. cache hit).
Czemu wstrzyknięty proxy siebie naprawia self-invocation?;Bo wywołanie idzie przez proxy (nie this), więc przechodzi przez interceptor z advice.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/05-spring/ioc-di]] · [[wiedza/06-persystencja/transakcje]] · [[wiedza/05-spring/ioc-di]]
