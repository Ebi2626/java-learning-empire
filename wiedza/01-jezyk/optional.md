---
temat: "Optional<T> — obsługa braku wartości"
faza: 1
status: nieopanowany
priorytet: 🟡
tags: [java, jezyk, optional, funkcyjne, null-safety]
powiazane: ["[[wiedza/01-jezyk/streamy-lambdy]]", "[[wiedza/01-jezyk/wyrazenia-lambda]]", "[[wiedza/01-jezyk/null-safety]]"]
---

# Optional<T> — obsługa braku wartości

> **TL;DR:** `Optional<T>` to **kontener na 0 lub 1 wartość**, który w *sygnaturze metody* jawnie mówi „tu może nie być wyniku" — zamiast cichego `null` i niespodziewanego `NullPointerException`. Używaj go jako **typu zwracanego** i operuj nim **funkcyjnie** (`map`/`flatMap`/`orElseGet`), a nie przez `isPresent()` + `get()`. **Nie** jako pole encji, parametr metody ani element kolekcji.

## 1. Co — definicja i API

`null` bywa nazywany **„the billion dollar mistake"** (Tony Hoare, twórca referencji null w ALGOL W, 1965). Problem: typ `String` *nie odróżnia* „jest String" od „nie ma Stringa". Każda dereferencja `x.length()` może rzucić `NullPointerException` (NPE) — a kompilator nie ostrzeże. `Optional<T>` (Java 8+) **przenosi tę informację do systemu typów**: `Optional<String>` jawnie znaczy „String, którego może nie być".

```java
Optional<User> findUser(String id);   // sygnatura KRZYCZY: wynik bywa pusty
// vs
User findUser(String id);             // może zwrócić null — i nie wiesz o tym
```

**Tworzenie:**
```java
Optional.of(value)          // value MUSI być != null, inaczej NPE od razu
Optional.ofNullable(value)  // null → empty(), inaczej of(value)  — most z legacy null
Optional.empty()            // pusty kontener (singleton, bez alokacji)
```

`Optional.of(null)` rzuca NPE celowo — to kontrakt: `of` dla wartości, którą *gwarantujesz*, `ofNullable` gdy może być `null`.

**Wydobywanie / wartości domyślne:**
```java
opt.orElse(other)                 // wartość albo other (other liczone ZAWSZE)
opt.orElseGet(() -> compute())    // wartość albo wynik Supplier (liczony LENIWIE)
opt.orElseThrow()                 // wartość albo NoSuchElementException
opt.orElseThrow(IllegalStateException::new)  // wartość albo własny wyjątek
opt.get()                         // wartość albo NoSuchElementException — UNIKAJ
```

**Inspekcja (używać oszczędnie):**
```java
opt.isPresent()   // true gdy jest wartość
opt.isEmpty()     // Java 11+, czytelniejsze niż !isPresent()
```

**Transformacje (rdzeń funkcyjny):**
```java
opt.map(User::getName)               // Optional<User> → Optional<String>
opt.flatMap(User::findManager)       // gdy funkcja sama zwraca Optional (bez zagnieżdżenia)
opt.filter(u -> u.isActive())        // zostawia wartość tylko gdy predykat true, inaczej empty
opt.or(() -> findInCache(id))        // Java 9+: alternatywny Optional gdy pusty
```

**Konsumpcja efektów ubocznych:**
```java
opt.ifPresent(u -> log(u));                          // robi coś TYLKO gdy jest
opt.ifPresentOrElse(u -> log(u), () -> log("brak")); // Java 9+: dwa branche
```

## 2. Jak — co dzieje się pod spodem

`Optional<T>` to zwykła, **niemutowalna** klasa-wrapper (`final class`) z jednym polem `private final T value` (gdzie `null` oznacza „empty"). `Optional.empty()` zwraca **współdzielony singleton**, więc pusty Optional nic nie alokuje. Niepusty Optional to **dodatkowa alokacja obiektu** na stercie — stąd nie używamy go w gorących pętlach ani jako pola obiektów masowo tworzonych.

`Optional` jest oznaczony `@jdk.internal.ValueBased` — JVM traktuje go jako **kandydata na typ wartościowy** (Project Valhalla). Konsekwencja praktyczna już dziś: **nie synchronizuj na Optionalu**, nie polegaj na jego tożsamości (`==`) ani identity hashcode.

**Optional jako „monada" / pipeline funkcyjny.** Optional spełnia (nieformalnie) prawa monady: `empty` to element neutralny, a `flatMap` składa operacje, które same mogą „nie dać wyniku". Dzięki temu zamiast drabiny `if (x != null) { if (y != null) {...} }` budujesz **liniowy pipeline**, który **przerywa się** na pierwszym pustym kroku (short-circuit):

```java
// drabina null-checków
String city = null;
if (user != null) {
    Address a = user.getAddress();
    if (a != null) {
        City c = a.getCity();
        if (c != null) city = c.getName();
    }
}
String result = city != null ? city : "Nieznane";

// to samo jako pipeline Optional
String result = Optional.ofNullable(user)
        .map(User::getAddress)
        .map(Address::getCity)
        .map(City::getName)
        .orElse("Nieznane");
```

**`map` vs `flatMap`.** `map` owija wynik w Optional automatycznie. Jeśli funkcja **sama zwraca `Optional`**, `map` da `Optional<Optional<X>>` — `flatMap` „spłaszcza" do `Optional<X>`:
```java
Optional<Optional<Manager>> bad  = opt.map(User::findManager);   // findManager zwraca Optional!
Optional<Manager>           good = opt.flatMap(User::findManager);
```

**`orElse` vs `orElseGet` — sedno (eager vs lazy).** `orElse(x)` to **zwykły argument metody** — Java oblicza go **zawsze**, jeszcze przed wejściem do `orElse`, **nawet gdy Optional ma wartość**. `orElseGet(supplier)` wywołuje `supplier.get()` **tylko gdy** Optional jest pusty.

```java
// fallback DROGI (zapytanie do bazy)
User u = optUser.orElse(loadDefaultFromDb());      // loadDefaultFromDb() ZAWSZE leci do bazy!
User u = optUser.orElseGet(() -> loadDefaultFromDb()); // baza tylko gdy faktycznie pusto
```
Gdy fallback jest tani i bezstanowy (np. literał `""`, `0`, `Collections.emptyList()`) — `orElse` jest OK i czytelniejszy. Gdy fallback ma **koszt** (I/O, alokacja, side-effect) lub **efekt uboczny** — **zawsze** `orElseGet`. Ta sama logika dotyczy `orElseThrow(supplier)`: wyjątek tworzony leniwie.

## 3. Dlaczego / kiedy — zasady i antywzorce

**Intencja projektantów (Brian Goetz, JEP/Stack Overflow):** Optional miał **jeden cel** — być **typem zwracanym metody**, gdy „brak wyniku" jest realnym, normalnym stanem i chcesz, by wołający *musiał* się z tym zmierzyć. To NIE jest ogólny zamiennik każdego `null`.

**Zasady użycia:**
- ✅ **Typ zwracany** metody, która może nie dać wyniku (`findById`, `Stream.findFirst`).
- ❌ **Pole encji / DTO.** Optional **nie jest `Serializable`** (świadoma decyzja — miał być przejściowym wynikiem, nie częścią stanu obiektu). Pole `Optional<X>` psuje serializację, frameworki (JPA/Jackson radzą sobie częściowo, ale to nie jego rola), i dodaje alokację na każdy rekord. Zamiast tego: pole `X` może być `null`, a **getter** zwraca `Optional<X>`.
- ❌ **Parametr metody.** Wymusza na wołającym opakowywanie (`f(Optional.of(x))`) i daje **trzy** stany do obsłużenia (`null` parametr, `empty`, wartość) zamiast dwóch. Użyj przeciążeń metody albo zwykłego argumentu.
- ❌ **W kolekcjach** (`List<Optional<T>>`, `Map<K, Optional<V>>`). Zamiast tego zwracaj **pustą kolekcję** (`Collections.emptyList()`) — „brak elementów" to naturalnie pusta lista, nie lista pustych Optionali. Dla `Map` użyj `getOrDefault` lub samej obecności klucza.

**Czemu nie `Serializable`:** dodanie `Serializable` zamroziłoby reprezentację binarną i sugerowało, że Optional ma być trwałym stanem — odwrotnie do intencji.

**Warianty prymitywne — `OptionalInt`, `OptionalLong`, `OptionalDouble`.** Unikają **autoboxingu** `Integer`/`Long`/`Double`. Zwracane przez `IntStream.max()`, `average()` itd. Mają `getAsInt()`/`orElse`, ale **nie mają `map`/`filter`** (uboższe API) — używaj ich na styku ze stream prymitywnych, nie wszędzie.

**Współpraca ze `Stream`:**
```java
Optional<T> first = stream.filter(...).findFirst();   // findFirst/findAny zwracają Optional
Optional<T> any   = stream.findAny();

// Optional.stream() (Java 9+) — 0 lub 1 element; idealne do filtrowania pustych:
List<User> users = ids.stream()
        .map(repo::findById)          // Stream<Optional<User>>
        .flatMap(Optional::stream)    // wyrzuca puste, rozpakowuje obecne → Stream<User>
        .toList();
```

### Antywzorce (i poprawne wersje)
```java
// ❌ isPresent()/get() zamiast orElse — odtwarzasz null-check, który Optional miał usunąć
String name = opt.isPresent() ? opt.get() : "domyślne";
// ✅
String name = opt.orElse("domyślne");

// ❌ get() bez sprawdzenia — NoSuchElementException to nowy NPE
return opt.get();
// ✅
return opt.orElseThrow(() -> new UserNotFoundException(id));

// ❌ pole Optional + opakowywanie wszystkiego
class Order { private Optional<Coupon> coupon; }   // nie Serializable, narzut
// ✅ pole nullable, Optional w getterze
class Order {
    private Coupon coupon;                          // może być null
    public Optional<Coupon> getCoupon() { return Optional.ofNullable(coupon); }
}

// ❌ orElse z drogim/efektownym fallbackiem
Config c = opt.orElse(loadConfigFromDisk());        // czyta dysk ZAWSZE
// ✅
Config c = opt.orElseGet(() -> loadConfigFromDisk());
```

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)

W **warstwie repozytorium** (Spring Data): `Optional<User> findById(Long id)` — sygnatura zmusza serwis do obsługi „nie ma użytkownika". Typowy serwis łączy to w pipeline:

```java
public OrderDto placeOrder(Long userId, Long couponId) {
    User user = userRepo.findById(userId)
            .filter(User::isActive)                          // nieaktywny = traktuj jak brak
            .orElseThrow(() -> new UserNotFoundException(userId));

    BigDecimal discount = couponRepo.findById(couponId)      // Optional<Coupon>
            .filter(c -> !c.isExpired())
            .map(Coupon::getDiscount)                        // Optional<BigDecimal>
            .orElse(BigDecimal.ZERO);                        // tani literał → orElse OK

    return orderService.create(user, discount);
}
```
Repozytorium zwraca **pustą listę** dla „brak zamówień" (`List<Order> findByUser(User u)`), a Optionala — tylko dla pojedynczego wyniku.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, jaki **jeden** problem rozwiązuje Optional i czym jest „billion dollar mistake".
- [ ] Bez zająknięcia podam różnicę `orElse` vs `orElseGet` (eager vs lazy) i kiedy która.
- [ ] Wiem, kiedy `map`, a kiedy `flatMap`.
- [ ] Wymienię 4 zakazane miejsca użycia (pole, parametr, kolekcja, nadużycie `get`) z uzasadnieniem.
- [ ] Wiem, czemu Optional nie jest `Serializable` i czemu nie używać go jako pola.
- [ ] Potrafię zamienić drabinę null-checków na pipeline Optional z głowy.

### 🔲 Black-box check (rzeczy do wyjaśnienia od podstaw)
- [ ] Co fizycznie siedzi w Optionalu? (jedno pole `final T value`, empty = singleton, alokacja gdy niepusty)
- [ ] Dlaczego `opt.orElse(f())` woła `f()` nawet gdy Optional ma wartość?
- [ ] Czemu `Optional.of(null)` rzuca NPE, a `ofNullable(null)` nie?
- [ ] Co robi `Optional.stream()` i po co `flatMap(Optional::stream)`?
- [ ] Czemu istnieją `OptionalInt`/`Long`/`Double` i czego im brakuje?
- [ ] W jakim sensie Optional jest „monadą"? (empty + flatMap, short-circuit)

### 🎤 Pytania rekrutacyjne (umiem odpowiedzieć)
- [ ] „Różnica `orElse` i `orElseGet`?" (ewaluacja eager vs lazy — popisowe pytanie)
- [ ] „Czy dobry jest `Optional` jako pole encji?" (nie: nie-Serializable, narzut, nie ta intencja)
- [ ] „Dlaczego `opt.isPresent() ? opt.get() : x` to zapach kodu?" (to przebrany null-check → `orElse`)
- [ ] „`map` vs `flatMap` w Optional?"
- [ ] „Po co był stworzony Optional według projektantów?" (typ zwracany metody, nie uniwersalny null)

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/`. Format: `Pytanie;Odpowiedź` (separator `;`).*

```
Czym jest Optional<T>?;Niemutowalnym kontenerem na 0 lub 1 wartość; w sygnaturze metody jawnie sygnalizuje możliwy brak wyniku zamiast null.
Czym różni się Optional.of od ofNullable?;of(null) rzuca NPE (gwarantujesz wartość), ofNullable(null) zwraca empty() (most z legacy null).
orElse vs orElseGet — różnica?;orElse(x) liczy x ZAWSZE (zwykły argument), orElseGet(supplier) wywołuje supplier tylko gdy Optional pusty (leniwie).
Kiedy używać orElseGet zamiast orElse?;Gdy fallback jest drogi lub ma efekt uboczny (I/O, alokacja); dla taniego literału orElse wystarcza.
map vs flatMap w Optional?;map owija wynik w Optional automatycznie; flatMap dla funkcji zwracającej już Optional (spłaszcza, unika Optional<Optional>).
Dlaczego isPresent() + get() to antywzorzec?;Odtwarza ręczny null-check, który Optional miał wyeliminować; zamiast tego map/orElse/orElseThrow.
Czemu unikać Optional.get()?;Rzuca NoSuchElementException bez sprawdzenia — to nowy NPE; użyj orElseThrow z sensownym wyjątkiem.
Gdzie NIE używać Optional?;Jako pole encji/DTO, parametr metody, element kolekcji; tam: nullable pole + getter Optional, przeciążenia, pusta kolekcja.
Czemu Optional nie jest Serializable?;Świadoma decyzja — miał być przejściowym wynikiem metody, nie trwałym stanem obiektu; dlatego nie nadaje się na pole.
Co zwraca metoda gdy „brak elementów" zamiast Optionala listy?;Pustą kolekcję (Collections.emptyList()) — nie List<Optional<T>> ani null.
Po co OptionalInt/Long/Double?;Unikają autoboxingu prymitywów; zwracane przez IntStream itp., ale nie mają map/filter.
Co robi Optional.stream() (Java 9+)?;Daje strumień 0 lub 1 elementu; z flatMap(Optional::stream) filtruje puste i rozpakowuje obecne.
W jakim sensie Optional to monada?;empty jest elementem neutralnym, flatMap składa operacje mogące nie dać wyniku; pipeline robi short-circuit na pierwszym empty.
Co zwracają Stream.findFirst/findAny?;Optional<T> — bo strumień może być pusty.
Jak zamienić drabinę null-checków na Optional?;Optional.ofNullable(x).map(...).map(...).orElse(default) — liniowy pipeline przerywany na pierwszym empty.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/01-jezyk/streamy-lambdy]] · [[wiedza/01-jezyk/wyrazenia-lambda]] · [[wiedza/01-jezyk/null-safety]] · [[wiedza/00-fundament/jdk-jre-jvm]]
