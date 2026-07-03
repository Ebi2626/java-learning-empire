---
temat: "Ładowanie klas (class loading) w JVM"
faza: 2
status: nieopanowany
priorytet: 🟡
tags: [java, jvm]
powiazane: ["[[wiedza/00-fundament/jdk-jre-jvm]]", "[[wiedza/02-jvm/jit]]", "[[wiedza/02-jvm/model-pamieci]]"]
---

# Ładowanie klas (class loading) w JVM

> **TL;DR:** JVM ładuje typy **leniwie** przez **classloadery** w trzech fazach — **Loading** (odczyt `.class` → obiekt `Class`),
> **Linking** (Verification → Preparation → Resolution) i **Initialization** (`<clinit>`). Loadery tworzą hierarchię z **parent
> delegation** (deleguj do rodzica najpierw — bezpieczeństwo + brak duplikatów klas rdzenia). **Tożsamość klasy = nazwa + classloader**,
> więc ta sama klasa z dwóch loaderów to dwa **różne** typy.

## 1. Co — definicja i API

**Classloader** to obiekt (`java.lang.ClassLoader`) odpowiedzialny za znalezienie definicji klasy (surowe bajty `.class`)
i przekazanie ich JVM, która buduje z nich obiekt `java.lang.Class<?>` w **metaspace**. Klasy nie są ładowane z góry —
JVM ładuje typ dopiero, gdy jest po raz pierwszy **potrzebny** (lazy loading).

```java
Class<?> c = String.class;                    // literał klasy — już załadowana przez bootstrap
ClassLoader cl = MyApp.class.getClassLoader(); // application/system classloader (classpath)
System.out.println(cl.getParent());            // platform classloader
Class<?> dyn = Class.forName("com.foo.Bar");   // jawne ładowanie + inicjalizacja w runtime
```

**Hierarchia loaderów** (Java 9+, JPMS):

```
Bootstrap ClassLoader   (natywny, C++, getClassLoader() == null)  → ładuje java.base i klasy rdzenia JDK
        ▲
Platform ClassLoader    (dawny "extension"/ext)                   → reszta modułów platformy (java.sql, java.xml...)
        ▲
Application/System CL   (URLClassLoader-podobny)                  → Twój kod z classpath / module path
        ▲
[Custom ClassLoadery]   (opcjonalnie — Twoje własne)
```

## 2. Jak — fazy i delegacja pod spodem

### Trzy fazy (JVMS §5.4–5.5)

**1) Loading** — classloader lokalizuje i odczytuje surowe bajty `.class`, parsuje strukturę (constant pool, pola, metody),
sprawdza format i wersję major/minor, a JVM tworzy w metaspace obiekt `Class`. Delegacja rodzicowi dzieje się właśnie tu.

**2) Linking** — trzy podfazy:
- **Verification** — statyczna analiza bytecode pod kątem **poprawności i bezpieczeństwa**: brak przepełnień stosu operandów,
  poprawne typy w operacjach, brak skoków w środek instrukcji, spójność `final` itd. To bariera, która chroni JVM przed
  złośliwym/uszkodzonym bytecode. Najkosztowniejsza podfaza (można ograniczyć `-Xverify:none`, ale to *niebezpieczne*).
- **Preparation** — alokacja pamięci na **pola statyczne** i przypisanie im **wartości domyślnych** (`0`, `0.0`, `false`, `null`) —
  NIE wykonuje jeszcze kodu z inicjalizatorów. `static int x = 5;` po Preparation ma `x == 0`.
- **Resolution** — zamiana **referencji symbolicznych** z constant pool (nazwy klas/pól/metod jako tekst) na **referencje
  bezpośrednie** (wskaźniki/offsety). Może być leniwa (rozwiązywana przy pierwszym użyciu danej referencji).

**3) Initialization** — wykonanie metody `<clinit>` (class initializer): JVM zbiera w kolejności źródłowej **inicjalizatory
statycznych pól** i **bloki `static { ... }`** w jedną metodę i ją uruchamia. Dopiero teraz `static int x = 5;` daje `x == 5`.
Odbywa się **leniwie** i jest **thread-safe** (JVM synchronizuje `<clinit>` per klasa).

### Kiedy JVM inicjalizuje klasę (triggery wg JLS §12.4.1)
- utworzenie instancji (`new`), 
- odczyt/zapis **statycznego pola** które NIE jest `static final` **stałą kompilacji** (stałe są inline'owane, nie triggerują),
- wywołanie **statycznej metody**,
- refleksja inicjalizująca (`Class.forName("X")` domyślnie inicjalizuje; `Class.forName(name, false, cl)` nie),
- inicjalizacja **podklasy** wymusza wcześniej inicjalizację nadklasy,
- klasa startowa z metodą `main`.
Sama referencja do klasy, `X.class`, `instanceof` czy dostęp do stałej `static final` **nie** inicjalizują klasy.

### Parent delegation model
Standardowy `ClassLoader.loadClass()` działa tak:
1. sprawdź, czy klasa już załadowana przez ten loader (`findLoadedClass`),
2. jeśli nie — **deleguj do rodzica** (`parent.loadClass`), aż do bootstrap,
3. dopiero gdy rodzic zawiedzie — spróbuj sam (`findClass`).

**Po co?**
- **Bezpieczeństwo** — nie podmienisz `java.lang.String` własną wersją; próba załadowania klasy z pakietu `java.*` zawsze
  trafi najpierw do bootstrap, który ma prawdziwą. Custom loader nie „przechwyci" klas rdzenia.
- **Spójność / brak duplikatów** — klasy rdzenia ładowane raz przez jeden loader; różne loadery nie tworzą wielu
  niekompatybilnych kopii `Object`/`String`.

### Custom classloadery
Nadpisujesz zwykle `findClass()` (zachowując delegację) albo cały `loadClass()` (żeby ją zmienić). Zastosowania:
- **serwery aplikacji** (Tomcat, JEE) — izolacja WAR-ów; Tomcat *świadomie odwraca* delegację dla klas webapp,
  by aplikacja widziała własne wersje bibliotek przed serwerowymi,
- **pluginy / OSGi** — izolowane moduły z własnymi zależnościami,
- **hot reload / hot swap** — nowy loader ładuje nową wersję klas; stary loader (z klasami) idzie do GC,
- **izolacja** — dwie wersje tej samej biblioteki obok siebie (bo różne loadery = różne typy).

### Unloading klasy
Klasa może być **wyładowana** (i jej metadane w metaspace zwolnione) tylko, gdy jej `ClassLoader` staje się
**nieosiągalny** i zostaje zebrany przez GC — razem z wszystkimi swoimi klasami i ich obiektami `Class`. Klasy ładowane
przez bootstrap/application praktycznie nigdy nie znikają. Dlatego wyciek loaderów (np. przy redeploy) → wyciek metaspace.

## 3. Dlaczego / kiedy — pułapki

### Tożsamość klasy = pełna nazwa + classloader
JVM identyfikuje typ **run-time** parą `(fully-qualified name, defining classloader)`. Ta sama nazwa `com.foo.Bar`
załadowana przez dwa różne loadery to **dwa różne typy**, niekompatybilne przypisaniowo:

```java
ClassLoader a = new MyLoader(), b = new MyLoader();
Object x = a.loadClass("com.foo.Bar").getConstructor().newInstance();
com.foo.Bar y = (com.foo.Bar) x; // ClassCastException! choć nazwa ta sama — inny loader
```

To źródło **zaskakującego `ClassCastException`** typu „Bar cannot be cast to Bar" w serwerach aplikacji / pluginach.

### ClassNotFoundException vs NoClassDefFoundError — dokładna różnica
| | `ClassNotFoundException` | `NoClassDefFoundError` |
|---|---|---|
| Typ | checked **Exception** | **Error** |
| Kiedy | **jawne** ładowanie po nazwie w runtime (`Class.forName`, `loadClass`, `ClassLoader`) nie znalazło klasy | klasa **była obecna przy kompilacji** (JVM próbuje jej użyć niejawnie: `new`, dostęp), ale w runtime **brak jej w classpath**, LUB nieudana wcześniejsza **inicjalizacja** klasy (rzucony `ExceptionInInitializerError` w `<clinit>`) |
| Przyczyna | zły literał nazwy, brak zależności ładowanej dynamicznie | niespójny classpath między kompilacją a runtime; wyjątek w statycznym inicjalizatorze |

Kluczowa subtelność: po nieudanym `<clinit>` klasa jest w stanie *erroneous* — **kolejne** próby jej użycia dają
`NoClassDefFoundError` (mimo że `.class` fizycznie istnieje).

### classpath vs module path (JPMS, krótko)
- **classpath** — płaska lista JAR-ów/katalogów; wszystko na niej ładuje application classloader, brak silnej enkapsulacji
  (klasa z `unnamed module` widzi wszystko publiczne).
- **module path** — moduły z `module-info.java`; JPMS wymusza **jawne `requires`/`exports`**, silną enkapsulację i wykrywa
  konflikty (split packages) już na starcie. Loadery pozostają te same, ale reguły widoczności są ostrzejsze.

## Przykład w praktyce (gdzie spotkasz to w realnym projekcie)
Deploy dwóch WAR-ów w Tomcacie: każdy ma **własny** `WebappClassLoader`, więc mogą używać różnych wersji tej samej
biblioteki bez konfliktu. Przy `undeploy`/`redeploy` Tomcat porzuca stary loader — jeśli coś (np. wątek w tle, `ThreadLocal`,
sterownik JDBC zarejestrowany w `DriverManager`) trzyma referencję do klasy z tego loadera, loader **nie idzie do GC** →
**metaspace leak** i po kilku redeployach `OutOfMemoryError: Metaspace`. Diagnostyka: heap dump + szukanie ścieżki GC do
osieroconego `WebappClassLoader`.

---

## ✅ Kryteria opanowania
- [ ] Wymienię 3 fazy i podfazy Linking (Verification/Preparation/Resolution) i powiem, co robi każda.
- [ ] Wyjaśnię parent delegation i po co jest (bezpieczeństwo + brak duplikatów).
- [ ] Podam DOKŁADNĄ różnicę `ClassNotFoundException` vs `NoClassDefFoundError`.
- [ ] Wytłumaczę, czemu tożsamość klasy to nazwa + loader i skąd bierze się dziwny `ClassCastException`.
- [ ] Wymienię triggery inicjalizacji klasy wg JLS i wiem, co NIE inicjalizuje.

### 🔲 Black-box check
- [ ] Co robi Preparation vs Initialization dla `static int x = 5;`? (0 vs 5)
- [ ] Dlaczego dostęp do `static final int` stałej nie inicjalizuje klasy? (inline'owanie)
- [ ] Jak i kiedy klasa może zostać wyładowana? (GC loadera)
- [ ] Czym różni się bootstrap od platform od application classloadera?
- [ ] Czym różni się classpath od module path pod względem widoczności?

### 🎤 Pytania rekrutacyjne
- [ ] „Opisz fazy ładowania klasy w JVM."
- [ ] „Czym różni się `ClassNotFoundException` od `NoClassDefFoundError`?"
- [ ] „Co to parent delegation i po co?"
- [ ] „Dlaczego dostajesz `ClassCastException` mimo tej samej nazwy klasy?"
- [ ] „Kiedy JVM inicjalizuje klasę?"

---

## 🃏 Fiszki Anki
*Źródło prawdy w `anki/02-jvm.csv`. Format: `Pytanie;Odpowiedź`.*

```
Trzy główne fazy ładowania klasy w JVM?;Loading, Linking (Verification→Preparation→Resolution), Initialization.
Co robi faza Loading?;Classloader odczytuje bajty .class, JVM parsuje je i tworzy obiekt Class w metaspace.
Trzy podfazy Linking?;Verification (poprawność/bezpieczeństwo bytecode), Preparation (alokacja + domyślne wartości pól statycznych), Resolution (referencje symboliczne → bezpośrednie).
Co robi Preparation z static int x = 5?;Alokuje pole i ustawia wartość DOMYŚLNĄ (0) — kod inicjalizatora jeszcze się nie wykonuje.
Kiedy static int x = 5 dostaje 5?;W fazie Initialization, w metodzie <clinit>.
Co to <clinit>?;Metoda class initializer — statyczne inicjalizatory pól i bloki static{} scalone i uruchamiane leniwie, thread-safe.
Co to parent delegation model?;loadClass najpierw deleguje do rodzica (aż bootstrap), sam ładuje dopiero gdy rodzic zawiedzie.
Po co parent delegation?;Bezpieczeństwo (nie podmienisz java.lang.String) i brak duplikatów klas rdzenia.
Hierarchia classloaderów (Java 9+)?;Bootstrap (natywny, java.base) → Platform (moduły platformy) → Application/System (classpath/module path) → custom.
Który classloader zwraca null z getClassLoader()?;Bootstrap classloader (natywny, brak obiektu Java).
Tożsamość klasy w JVM to?;Para (pełna nazwa + defining classloader) — ta sama nazwa z dwóch loaderów = dwa różne typy.
Dlaczego ClassCastException mimo tej samej nazwy klasy?;Bo klasy załadowane przez różne classloadery to różne typy (tożsamość = nazwa + loader).
ClassNotFoundException — kiedy?;Jawne ładowanie po nazwie w runtime (Class.forName/loadClass) nie znalazło klasy; checked Exception.
NoClassDefFoundError — kiedy?;Klasa była przy kompilacji, ale brak jej w runtime, lub nieudana wcześniejsza inicjalizacja (<clinit>); to Error.
Triggery inicjalizacji klasy (JLS)?;new, dostęp do niestałego pola statycznego, wywołanie metody statycznej, refleksja (forName), inicjalizacja podklasy, klasa z main.
Co NIE inicjalizuje klasy?;X.class, instanceof, dostęp do static final stałej kompilacji (jest inline'owana).
Kiedy klasa może być wyładowana (unloaded)?;Gdy jej classloader staje się nieosiągalny i zostaje zebrany przez GC — razem z klasami.
Zastosowania custom classloaderów?;Serwery aplikacji (izolacja WAR), pluginy/OSGi, hot reload, izolacja różnych wersji bibliotek.
Classpath vs module path?;Classpath — płaska lista, słaba enkapsulacja; module path — JPMS z requires/exports, silna enkapsulacja i kontrola konfliktów.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/00-fundament/jdk-jre-jvm]] · [[wiedza/02-jvm/jit]] · [[wiedza/02-jvm/model-pamieci]]
