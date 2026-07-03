---
temat: "IoC i DI w Springu"
faza: 5
status: nieopanowany
priorytet: 🔴
tags: [java, spring]
powiazane: ["[[wiedza/05-spring/proxy-aop]]", "[[wiedza/05-spring/autokonfiguracja]]", "[[wiedza/05-spring/konfiguracja-profile]]"]
---

# IoC i DI w Springu

> **TL;DR:** **IoC (Inversion of Control)** to *zasada* — to framework, nie twój kod, tworzy obiekty i wiąże je ze
> sobą. **DI (Dependency Injection)** to *technika* realizująca IoC — zależności są *wstrzykiwane* z zewnątrz
> zamiast tworzone przez `new` w środku klasy. Rdzeniem jest **kontener** (`ApplicationContext`), który zarządza
> **beanami** (obiektami pod nadzorem Springa) — ich definicją, tworzeniem, wiązaniem i cyklem życia.
> **Preferuj constructor injection**: daje niemutowalność, jawne wymagane zależności i wykrycie cykli na starcie.

## 1. Co — IoC, DI, kontener i beany

**Inversion of Control** — odwrócenie sterowania. Bez frameworka *twój* kod steruje: `new PaymentService(new StripeClient())`.
Z IoC to **framework przejmuje kontrolę** nad tworzeniem i łączeniem obiektów; ty tylko *deklarujesz* zależności,
a kontener je dostarcza („Hollywood principle": *don't call us, we'll call you*). IoC to szerokie pojęcie
(event loop, template method, callbacki też są IoC); **DI to jego konkretna forma** dotycząca dostarczania zależności.

```
        BEZ IoC                             Z IoC / DI (Spring)
   ┌──────────────┐                    ┌──────────────────────────┐
   │  MyService   │                    │   ApplicationContext     │
   │  new Repo()  │  ← ja tworzę       │  tworzy Repo, tworzy     │
   │  new Client()│    zależności      │  Service, WSTRZYKUJE     │
   └──────────────┘                    │  Repo do konstruktora    │
                                       └──────────────────────────┘
```

**Bean** = obiekt tworzony, składany i zarządzany przez kontener IoC Springa. **Kontener** wie, jakie beany
istnieją (z *definicji beanów*), rozwiązuje zależności między nimi i buduje graf obiektów.

**Dwie abstrakcje kontenera:**
- **`BeanFactory`** — bazowy, „goły" kontener: leniwe (lazy) tworzenie beanów na żądanie, minimalny narzut.
- **`ApplicationContext`** — nadzbiór `BeanFactory`, **domyślny wybór**. Dodaje: **eager** tworzenie singletonów na
  starcie (błędy wychodzą od razu, nie w runtime), publikowanie zdarzeń (`ApplicationEvent`),
  internacjonalizację (`MessageSource`), integrację z AOP, wsparcie środowiska (`Environment`, profile, `@Value`).
  W praktyce **zawsze używasz `ApplicationContext`**; `BeanFactory` to szczegół implementacyjny pod spodem.

**Definiowanie beanów — dwa style:**

1. **Stereotypy + component scanning** (dla *swoich* klas):
   - `@Component` — ogólny bean.
   - `@Service` — warstwa logiki biznesowej (semantyka, nie techniczna różnica).
   - `@Repository` — warstwa dostępu do danych; dodatkowo **tłumaczy wyjątki** JDBC/JPA na `DataAccessException`.
   - `@Controller` / `@RestController` — warstwa web (MVC).
   - `@ComponentScan` (włączony przez `@SpringBootApplication`) skanuje pakiety i rejestruje adnotowane klasy jako beany.

2. **`@Configuration` + `@Bean`** (dla *cudzych* klas i konfiguracji imperatywnej):
   ```java
   @Configuration
   class AppConfig {
       @Bean
       DataSource dataSource() { return new HikariDataSource(/* ... */); }
   }
   ```
   Używasz, gdy **nie możesz dodać adnotacji** (klasa z biblioteki, np. `RestClient`, `ObjectMapper`), albo gdy
   konstrukcja beana wymaga logiki. Metoda `@Bean` może przyjmować inne beany jako parametry — Spring je wstrzyknie.

**Reguła:** stereotypy dla *własnego* kodu (mniej boilerplate'u), `@Bean` dla *zewnętrznych* typów i konfiguracji.

## 2. Jak — kontener i cykl życia beana pod spodem

### Bootstrap kontenera
`SpringApplication.run(App.class, args)` w skrócie:
1. Tworzy `ApplicationContext` (dla web: `AnnotationConfigServletWebServerApplicationContext`).
2. **Skanuje** komponenty i czyta klasy `@Configuration` → buduje **`BeanDefinition`** dla każdego beana
   (metadane: klasa, scope, zależności, metody init/destroy). To jeszcze *nie* są obiekty — to „przepisy".
3. Uruchamia **`BeanFactoryPostProcessor`**-y (mogą modyfikować definicje; tu działa m.in. autokonfiguracja).
4. Tworzy instancje **singletonów** (eager) — instancjonuje, wstrzykuje zależności, przetwarza przez `BeanPostProcessor`-y.
5. Publikuje zdarzenia (`ContextRefreshedEvent`), a w Boot startuje serwer i wywołuje `ApplicationRunner`/`CommandLineRunner`.

### Rozwiązywanie `@Autowired` — kolejność
Przy wstrzykiwaniu Spring dobiera kandydata tak:
1. **Po typie** (najczęstszy przypadek) — szuka beana pasującego typem.
2. Jeśli **wielu kandydatów tego samego typu** → próbuje po **nazwie** (nazwa pola/parametru = nazwa beana).
3. `@Primary` na jednym z beanów rozstrzyga „domyślnego zwycięzcę".
4. `@Qualifier("nazwa")` na punkcie wstrzyknięcia wskazuje konkretny bean jawnie.
5. Jeśli nadal niejednoznacznie → **`NoUniqueBeanDefinitionException`** (a gdy zero kandydatów → `NoSuchBeanDefinitionException`).

```java
@Service class SmsSender implements Notifier {}
@Service @Primary class EmailSender implements Notifier {}

@Service
class AlertService {
    private final Notifier notifier;
    AlertService(@Qualifier("smsSender") Notifier notifier) { this.notifier = notifier; }
}
```

**Wstrzykiwanie kolekcji:** żądając `List<Notifier>` (albo `Map<String, Notifier>` — klucz = nazwa beana) dostajesz
**wszystkie** beany danego typu. Idealne do wzorca strategii/pluginów; kolejność sterowana `@Order`/`Ordered`.

### Trzy typy wstrzykiwania

| Typ | Jak | Uwagi |
|-----|-----|-------|
| **Konstruktorowe** | zależności jako parametry konstruktora | **PREFEROWANE** |
| **Setterowe** | `@Autowired` na setterze | dla zależności *opcjonalnych*/zmiennych |
| **Polowe** | `@Autowired` bezpośrednio na polu | **ODRADZANE** |

- **Constructor injection (preferowane):** pola mogą być `final` → **niemutowalność**; zależności są **wymagane**
  i jawne (nie da się utworzyć obiektu w niepełnym stanie); **łatwa testowalność** (w teście po prostu `new Service(mock)`
  bez Springa i refleksji); wymusza **wykrycie cykli** na starcie. Od Springa 4.3 przy **jednym** konstruktorze
  `@Autowired` jest zbędne.
- **Setter injection:** dla zależności **opcjonalnych** lub rekonfigurowalnych; obiekt może istnieć bez nich.
- **Field injection (odradzane):** krótkie, ale: **nie da się `final`** (mutowalne), **ukrywa zależności**
  (nie widać ich w API klasy — klasa może „po cichu" urosnąć do 15 zależności), **testowalność** wymaga refleksji
  lub kontenera, i zachęca do cykli. Reguła: **field injection tylko w testach**, nie w kodzie produkcyjnym.

### Cykl życia beana (pod spodem)
```
BeanDefinition
   │  instancjonowanie (konstruktor)
   ▼
wstrzyknięcie zależności (setter/pole)
   │
   ▼
callbacki *Aware  (BeanNameAware, ApplicationContextAware...)
   │
   ▼
BeanPostProcessor.postProcessBeforeInitialization
   │
   ▼
@PostConstruct  →  InitializingBean.afterPropertiesSet  →  @Bean(initMethod)
   │
   ▼
BeanPostProcessor.postProcessAfterInitialization   ←── TU powstają PROXY (AOP, @Transactional)
   │
   ▼
★ BEAN GOTOWY (używany) ★
   │  (przy zamykaniu kontenera, tylko singletony)
   ▼
@PreDestroy  →  DisposableBean.destroy  →  @Bean(destroyMethod)
```
Kluczowe: **proxy** (transakcje, `@Async`, security) powstają w `postProcessAfterInitialization` — dlatego
wstrzyknięty bean bywa **proxy**, a nie „gołym" obiektem (→ [[wiedza/05-spring/proxy-aop]]).

### Scope beana
- **`singleton`** (domyślny) — **jedna instancja na kontener**. Uwaga: to **NIE** wzorzec projektowy Singleton
  (nie ma prywatnego konstruktora ani statycznej instancji na JVM); to „jeden na `ApplicationContext`". Singletony
  muszą być **bezstanowe** (współdzielone między wątkami — stan mutowalny = bug współbieżnościowy).
- **`prototype`** — **nowa instancja przy każdym pobraniu**. Uwaga: Spring **nie zarządza pełnym cyklem** prototype'ów
  (nie woła `@PreDestroy`). Wstrzyknięcie prototype do singletona daje *jedną* instancję — po świeże trzeba
  `ObjectProvider`/`@Lookup`/proxy.
- **`request`, `session`, `application`, `websocket`** — tylko w kontekście web; instancja na żądanie/sesję HTTP.

### Cykliczne zależności
- **Konstruktor↔konstruktor:** A wymaga B w konstruktorze, B wymaga A → Spring **nie może** utworzyć żadnego
  (żaden nie może być pierwszy) → **`BeanCurrentlyInCreationException`** na starcie. To *dobra* wiadomość — błąd projektu wychodzi wcześnie.
- **Setter/pole:** Spring **radzi sobie** dzięki *wczesnej referencji* — tworzy A (jeszcze niedokończone),
  wstawia jego wczesną (goła instancja) referencję do B, potem domyka A. Cykl „działa", ale to zapach architektury.
- **Naprawa:** najlepiej **refaktoryzacja** (wydziel trzecią klasę, przenieś odpowiedzialność); doraźnie
  **`@Lazy`** na jednym punkcie wstrzyknięcia (Spring wstrzyknie *proxy*, realny bean powstanie przy pierwszym użyciu).
  Uwaga: Boot 2.6+ domyślnie **zabrania** cykli (`spring.main.allow-circular-references=false`).

### Lazy vs eager
Singletony domyślnie **eager** (na starcie) → błędy wychodzą natychmiast, pierwszy request jest szybki.
`@Lazy` odracza tworzenie do pierwszego użycia — przyspiesza start, ale odsuwa błędy w czas działania.

### `@Value` i properties
`@Value("${app.timeout:30}")` wstrzykuje wartość z `application.yml`/`env` (z domyślną `30`).
Dla wielu powiązanych propertiesów lepszy jest typowany `@ConfigurationProperties` (→ [[wiedza/05-spring/konfiguracja-profile]]).

## 3. Dlaczego / kiedy — dobór wstrzykiwania, cykle, pułapki

- **Zawsze constructor injection** dla wymaganych zależności. Kolekcja parametrów rosnąca do 5+ to nie wada
  wstrzykiwania, tylko **sygnał złamania SRP** — field injection ten smród tylko *ukrywa*.
- **`@Repository` nie jest kosmetyką** — włącza translację wyjątków persystencji na spójną hierarchię `DataAccessException`.
- **Singleton + stan mutowalny = pułapka współbieżności.** Trzymaj stan w argumentach metod / obiektach request-scope,
  nie w polach singletona.
- **Prototype w singletonie** prawie zawsze robi coś innego niż programista myśli (jedna instancja na całe życie
  singletona). Użyj `ObjectProvider<T>` gdy potrzebujesz świeżej instancji na wywołanie.
- **`NoUniqueBeanDefinitionException`** przy wielu implementacjach interfejsu — rozwiąż `@Primary` (globalny domyślny)
  albo `@Qualifier` (lokalny, jawny wybór). Nie „dobieraj po nazwie" przypadkiem — to kruche.
- **Wstrzyknięty bean to często proxy** — dlatego wywołanie metody `@Transactional` z *tej samej klasy* (self-invocation)
  omija proxy i transakcja nie działa. Klasyczna pułapka (→ [[wiedza/05-spring/proxy-aop]]).
- **Nie mieszaj `new` z beanami** — obiekt utworzony ręcznie *nie* przechodzi przez kontener (brak DI, brak proxy, brak AOP).

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

Typowa warstwa serwisowa: serwis zależy od repozytorium, wstrzykiwanego przez **konstruktor**.

```java
@Repository
interface OrderRepository extends JpaRepository<Order, Long> {}

@Service
class OrderService {

    private final OrderRepository orders;      // final → niemutowalne, wymagane
    private final PaymentGateway payments;

    // jeden konstruktor → @Autowired zbędne (Spring 4.3+)
    OrderService(OrderRepository orders, PaymentGateway payments) {
        this.orders = orders;
        this.payments = payments;
    }

    @Transactional
    public Order place(NewOrder cmd) {
        var order = orders.save(Order.from(cmd));
        payments.charge(order);
        return order;
    }
}
```

Test **bez Springa** — czysta zaleta constructor injection:
```java
var service = new OrderService(mock(OrderRepository.class), mock(PaymentGateway.class));
```
Gdyby `orders`/`payments` były wstrzyknięte przez pole (`@Autowired` na polu), tego `new` **nie dałoby się napisać** —
trzeba by refleksji lub całego kontekstu Springa w teście jednostkowym.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię różnicę **IoC (zasada)** vs **DI (technika)** jednym zdaniem.
- [ ] Uzasadnię, czemu **constructor injection** bije field injection (niemutowalność, testy, wykrycie cykli).
- [ ] Znam różnicę `BeanFactory` vs `ApplicationContext` i wiem, czemu używam tego drugiego.
- [ ] Wiem, kiedy stereotyp, a kiedy `@Configuration`+`@Bean`.
- [ ] Rozrysuję **cykl życia beana** i wskażę, gdzie powstają proxy.
- [ ] Rozwiążę `NoUniqueBeanDefinitionException` (`@Primary` vs `@Qualifier`).
- [ ] Wyjaśnię, czemu singleton to NIE wzorzec Singleton i czemu musi być bezstanowy.
- [ ] Wiem, czemu cykl konstruktorowy wybucha, a setterowy „działa" — i jak naprawić.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co to `BeanDefinition` i kiedy staje się realnym obiektem?
- [ ] Jak kontener bootstrapuje (od `run()` do gotowych singletonów)?
- [ ] W jakiej kolejności `@Autowired` dobiera kandydata (typ → nazwa → @Primary/@Qualifier)?
- [ ] Co dzieje się w `postProcessAfterInitialization` i skąd bierze się proxy?
- [ ] Czemu wstrzyknięcie `List<Interfejs>` daje wszystkie beany danego typu?
- [ ] Jak Spring rozwiązuje cykl setterowy (wczesna referencja)?

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Czym różni się IoC od DI?"
- [ ] „Czemu constructor injection jest preferowane nad field injection?"
- [ ] „`ApplicationContext` vs `BeanFactory` — po co ten pierwszy?"
- [ ] „Czym się różni scope singleton od wzorca Singleton?"
- [ ] „Co się stanie przy cyklicznej zależności i jak to naprawić?"
- [ ] „Masz dwie implementacje interfejsu — jak Spring wybierze, którą wstrzyknąć?"
- [ ] „Kiedy `@Bean` w `@Configuration`, a kiedy `@Component`?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/05-spring.csv`.*

```
Co to IoC (Inversion of Control)?;Zasada: framework (nie twój kod) tworzy i wiąże obiekty; ty tylko deklarujesz zależności.
Co to DI (Dependency Injection)?;Technika realizująca IoC — zależności są wstrzykiwane z zewnątrz zamiast tworzone przez new wewnątrz klasy.
IoC vs DI — relacja?;DI to konkretna forma IoC dotycząca dostarczania zależności; IoC jest pojęciem szerszym.
Co to bean?;Obiekt tworzony, składany i zarządzany przez kontener IoC Springa.
BeanFactory vs ApplicationContext?;ApplicationContext to nadzbiór BeanFactory — dodaje eager init singletonów, zdarzenia, i18n, AOP, Environment. Używasz go domyślnie.
Czym różni się @Repository od @Component?;Semantyką warstwy danych + włącza translację wyjątków JDBC/JPA na DataAccessException.
Kiedy @Bean w @Configuration zamiast stereotypu?;Gdy nie możesz dodać adnotacji (klasa z biblioteki) lub konstrukcja beana wymaga logiki.
Który typ wstrzykiwania preferować i czemu?;Konstruktorowe — pola final (niemutowalność), zależności wymagane i jawne, łatwe testy, wykrycie cykli na starcie.
Czemu field injection jest odradzane?;Brak final (mutowalne), ukrywa zależności, testy wymagają refleksji/kontenera, zachęca do cykli.
Kiedy setter injection?;Dla zależności opcjonalnych lub rekonfigurowalnych — obiekt może istnieć bez nich.
Kolejność rozwiązywania @Autowired?;Najpierw po typie, potem po nazwie; rozstrzyga @Primary (globalny) lub @Qualifier (jawny).
Jaki wyjątek przy wielu kandydatach tego samego typu?;NoUniqueBeanDefinitionException.
@Primary vs @Qualifier?;@Primary = globalny domyślny zwycięzca; @Qualifier = jawny wybór w konkretnym punkcie wstrzyknięcia.
Co dostaniesz wstrzykując List<Interfejs>?;Wszystkie beany danego typu (Map<String,T> — kluczem nazwa beana). Przydatne do wzorca strategii.
Domyślny scope beana?;singleton — jedna instancja na kontener (ApplicationContext).
Singleton scope to wzorzec Singleton?;Nie — to "jeden na kontener", bez prywatnego konstruktora/statycznej instancji na JVM. Musi być bezstanowy.
Scope prototype?;Nowa instancja przy każdym pobraniu; Spring nie woła jego @PreDestroy.
Kolejność cyklu życia beana?;Instancjonowanie → wstrzyknięcie zależności → *Aware → BPP before → @PostConstruct/InitializingBean → gotowy → @PreDestroy.
Gdzie powstają proxy (AOP, @Transactional)?;W BeanPostProcessor.postProcessAfterInitialization — dlatego wstrzyknięty bean bywa proxy.
Cykliczna zależność konstruktorowa?;BeanCurrentlyInCreationException na starcie — żaden bean nie może być pierwszy.
Cykl setterowy/polowy — czemu działa?;Spring używa wczesnej referencji (goła instancja) i domyka bean później. Napraw: refaktor lub @Lazy.
Co robi @Value("${app.timeout:30}")?;Wstrzykuje wartość property app.timeout (z domyślną 30) z konfiguracji/env.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/05-spring/proxy-aop]] · [[wiedza/05-spring/autokonfiguracja]] · [[wiedza/05-spring/konfiguracja-profile]]
