---
temat: "SDKMAN! — zarządzanie wersjami JDK i narzędzi JVM"
faza: 0
status: nieopanowany
priorytet: 🔴
tags: [java, fundament, tooling, sdkman, jdk]
powiazane: ["[[wiedza/00-fundament/jdk-jre-jvm]]", "[[wiedza/00-fundament/wersje-javy-lts]]", "[[wiedza/01-build/maven]]", "[[wiedza/01-build/gradle]]"]
---

# SDKMAN! — zarządzanie wersjami JDK i narzędzi JVM

> **TL;DR:** **SDKMAN!** to menedżer wersji dla ekosystemu JVM. Pozwala mieć obok siebie wiele JDK
> (różne wersje i dystrybucje) oraz narzędzia (Maven, Gradle) i przełączać je jedną komendą.
> Pod spodem to czysta mechanika `PATH` + `JAVA_HOME` przez symlinki w `~/.sdkman` — żadnej magii.

## 1. Co — definicja i komendy

SDKMAN! (Software Development Kit Manager) instaluje i przełącza **kandydatów** (*candidates*) — czyli SDK i narzędzia:
Java (JDK), Maven, Gradle, Kotlin, Scala, Spring Boot CLI itd. Dla Javy dodatkowo wybierasz **dystrybucję**
(o tym niżej). Wszystko ląduje w `~/.sdkman/candidates/<kandydat>/<wersja>/`.

**Po co w ogóle wiele wersji JDK?**
- Różne projekty wymagają różnych wersji (legacy na 8/11, nowe na 21, eksperymenty na najnowszej).
- **LTS vs najnowsza**: do nauki i produkcji trzymasz się **LTS** (Long-Term Support: 8, 11, 17, **21**, 25),
  ale czasem chcesz przetestować feature z najnowszego (non-LTS) wydania. Patrz [[wiedza/00-fundament/wersje-javy-lts]].
- Bez menedżera oznacza to ręczne grzebanie w `JAVA_HOME` i `PATH` przy każdej zmianie — uciążliwe i podatne na błędy.

**Instalacja** (Bash/Zsh):
```bash
curl -s "https://get.sdkman.io" | bash      # pobiera ~/.sdkman i dopisuje init do .bashrc/.zshrc
source "$HOME/.sdkman/bin/sdkman-init.sh"    # załaduj w bieżącej sesji (lub otwórz nowy terminal)
sdk version                                  # sanity check
```

**Kluczowe komendy:**
```bash
sdk list java                 # katalog dostępnych wersji + dystrybucji (kolumna "Identifier")
sdk install java 21.0.x-tem   # instaluj konkretną wersję, "-tem" = Eclipse Temurin
sdk install java              # instaluj domyślną (rekomendowaną) wersję
sdk use java 21.0.x-tem       # przełącz TYLKO w bieżącej powłoce (do zamknięcia terminala)
sdk default java 21.0.x-tem   # ustaw jako domyślną globalnie (nowe sesje)
sdk current java              # która wersja jest aktywna teraz
sdk install maven             # narzędzia też są kandydatami
sdk install gradle 8.9
sdk upgrade                   # pokaż/zainstaluj nowsze wersje zainstalowanych kandydatów
sdk uninstall java 21.0.1-tem
sdk flush                     # czyść cache (gdy lista wersji się "zacięła")
```

## 2. Jak — co dzieje się pod spodem (zabijamy „magię")

Cała struktura żyje w `~/.sdkman/`:
```
~/.sdkman/
├── bin/sdkman-init.sh        # ten skrypt dopisuje się do ~/.bashrc i robi całą robotę
├── candidates/
│   ├── java/
│   │   ├── 21.0.5-tem/        # rzeczywiste rozpakowane JDK
│   │   ├── 25.0.1-tem/
│   │   └── current  ─────────► symlink na aktywną wersję (np. -> 21.0.5-tem)
│   └── maven/
│       ├── 3.9.9/
│       └── current ─────────► symlink
```

**Mechanika krok po kroku:**
1. Przy starcie powłoki `sdkman-init.sh` (wczytany z `.bashrc`) dopisuje do `PATH`:
   `~/.sdkman/candidates/java/current/bin`, `~/.sdkman/candidates/maven/current/bin` itd.
   oraz ustawia `JAVA_HOME=~/.sdkman/candidates/java/current`.
2. `sdk default java X` **przestawia symlink** `candidates/java/current` na katalog `X`. Nowe sesje widzą zmianę.
3. `sdk use java X` **nie rusza symlinka** — zamiast tego w bieżącej powłoce nadpisuje `PATH`/`JAVA_HOME`,
   wskazując bezpośrednio na `candidates/java/X`. Dlatego efekt znika po zamknięciu terminala.

Stąd reguła: `current` w `which java` → `~/.sdkman/...` znaczy, że SDKMAN przejął kontrolę.
Jeśli `which java` pokazuje `/usr/bin/java`, to nadpisuje to systemowe JDK (kolejność w `PATH`).

**`.sdkmanrc` — przypinanie wersji per projekt:**
```bash
cd ~/projekty/moj-projekt
sdk env init        # tworzy plik .sdkmanrc z aktualnymi wersjami
```
Plik `.sdkmanrc`:
```
java=21.0.5-tem
maven=3.9.9
```
Następnie `sdk env` w katalogu projektu ustawia te wersje (jak `sdk use`, ale z pliku).
Z opcją auto-env (`sdkman_auto_env=true` w `~/.sdkman/etc/config`) dzieje się to automatycznie przy `cd`.
To odpowiednik `.nvmrc` z Node — wersja narzędzi jest częścią repo, więc cały zespół ma to samo.

## 3. Dlaczego / kiedy — dystrybucje i alternatywy

**Pojęcie kandydata i dystrybucji.** „Java" jako kandydat to nie jedno JDK — to wybór *dystrybucji*
(builda OpenJDK od różnych dostawców). Wszystkie są zgodne ze standardem, różnią się supportem, licencją,
domyślnym GC i drobiazgami:

| Sufiks | Dystrybucja | Uwagi |
|--------|-------------|-------|
| `-tem` | **Eclipse Temurin** (Adoptium) | Domyślny rozsądny wybór, w pełni open, neutralny — **bierz to do nauki**. |
| `-amzn` | Amazon **Corretto** | Wsparcie AWS, dobre pod kątem produkcji w chmurze Amazona. |
| `-zulu` | Azul **Zulu** | Szeroki zakres wersji (też bardzo stare/egzotyczne). |
| `-graal` / `-graalce` | **GraalVM** | Native image (AOT), polyglot — gdy potrzebujesz `native-image`. Patrz [[wiedza/00-fundament/jdk-jre-jvm]]. |
| `-oracle` | Oracle JDK | Uwaga na licencję w niektórych zastosowaniach komercyjnych. |

**Alternatywy dla SDKMAN i kiedy które:**
- **Systemowe pakiety dystrybucji** (`dnf install java-21-openjdk` na Fedorze) — proste, ale jedna/dwie wersje,
  aktualizacje w rytmie OS, trudniej trzymać kilka wersji obok siebie. Dobre, gdy potrzebujesz dokładnie jednego JDK.
- **`jenv`** — sam zarządza *przełączaniem* `JAVA_HOME` między JDK już zainstalowanymi (przez pakiety/SDKMAN),
  ale **nie instaluje** JDK. Sensowny, gdy JDK dostarcza Ci system/Homebrew, a chcesz tylko per-project switching.
- **Ręczny `JAVA_HOME` + `PATH`** — pełna kontrola, zero zależności, ale ręczna robota i łatwo o pomyłkę.
  OK dla CI/skryptów albo gdy masz dokładnie jeden JDK.
- **`jabba`** — cross-platformowy (działa też natywnie na Windows, gdzie SDKMAN wymaga Git Bash/WSL),
  mniej popularny dziś. Wybór głównie historyczny/Windowsowy.

**Kiedy SDKMAN wygrywa:** lokalny dev na Linux/macOS, wiele projektów o różnych wymaganiach,
chęć szybkiego testowania nowych wydań i dystrybucji bez śmiecenia w systemie. **Kiedy NIE:** obrazy
kontenerów produkcyjnych (tam użyj oficjalnego image JDK lub `jlink`), bardzo zaślinione środowiska CI
(lepiej pinować konkretny obraz/wersję jawnie).

## Przykład w praktyce (dla Twojego setupu na Fedorze)

Masz systemowe **JDK 25 (Red Hat build)** z `dnf`. Zostaw je — to backup. Do nauki chcesz **JDK 21 (LTS)**
zarządzane przez SDKMAN i ustawione jako domyślne:

```bash
# 1. Instalacja SDKMAN (raz)
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

# 2. Zobacz dostępne buildy Temurin 21 i wybierz najświeższy patch
sdk list java | grep -i '21\..*tem'

# 3. Zainstaluj Temurin 21 LTS i ustaw jako DEFAULT (globalnie, dla nowych sesji)
sdk install java 21.0.5-tem        # <- podstaw aktualny identyfikator z listy
sdk default java 21.0.5-tem

# 4. Weryfikacja — powinno wskazywać na ~/.sdkman, nie /usr/bin
sdk current java
java -version                       # openjdk version "21.0.5" ... Temurin
which java                          # ~/.sdkman/candidates/java/current/bin/java
echo $JAVA_HOME                     # ~/.sdkman/candidates/java/current

# 5. Maven przez SDKMAN (spójnie z resztą)
sdk install maven

# 6. (opcjonalnie) eksperyment na systemowym JDK 25 tylko w jednej powłoce:
#    najpierw zainstaluj je też w SDKMAN i użyj 'sdk use', albo wskaż JAVA_HOME ręcznie.
```

Jeśli kiedyś projekt wymaga innej wersji — w jego katalogu zrób `sdk env init`, dopisz `java=...`,
i `sdk env` ustawi ją lokalnie, nie ruszając globalnego defaultu 21.

> ⚠️ Uwaga: jeśli `java -version` po `sdk default` dalej pokazuje 25, to systemowe `/usr/bin/java`
> wyprzedza SDKMAN w `PATH` w tej sesji — otwórz **nowy terminal** lub `source` init skrypt ponownie.

---

## ✅ Kryteria opanowania
- [ ] Wyjaśnię, po co mieć wiele wersji JDK i co to LTS vs najnowsza.
- [ ] Zainstaluję SDKMAN i Temurin 21, ustawię jako default, zweryfikuję `sdk current java`.
- [ ] Rozróżnię `sdk use` (sesja) od `sdk default` (globalnie) i wiem dlaczego.
- [ ] Wytłumaczę, jak SDKMAN modyfikuje `PATH`/`JAVA_HOME` (symlink `current` + init w `.bashrc`).
- [ ] Wiem, czym jest kandydat i dystrybucja (Temurin/Corretto/Zulu/GraalVM).
- [ ] Przypnę wersję per projekt przez `.sdkmanrc` + `sdk env`.
- [ ] Wybiorę między SDKMAN, jenv, pakietem systemowym i ręcznym `JAVA_HOME`.

### 🔲 Black-box check
- [ ] Co dokładnie zmienia `sdk default` vs `sdk use`? (symlink `current` vs nadpisanie PATH w sesji)
- [ ] Skąd nowa powłoka „wie", że ma używać wersji z SDKMAN? (init w `.bashrc` ustawia PATH/JAVA_HOME)
- [ ] Czemu `which java` to dobry test poprawnego setupu?
- [ ] Czym różni się Temurin od Corretto, a czym GraalVM? (dostawca/support vs AOT/native-image)
- [ ] Jak działa `.sdkmanrc` i czym przypomina `.nvmrc`?

### 🎤 Pytania rekrutacyjne
- [ ] „Jak zarządzasz wieloma wersjami JDK na maszynie?"
- [ ] „Czym jest `JAVA_HOME` i co go ustawia?"
- [ ] „Jak zapewnisz, że cały zespół buduje na tej samej wersji Javy/Maven?" (`.sdkmanrc` / pinowanie w CI)
- [ ] „Temurin vs Oracle JDK vs GraalVM — kiedy co?"

---

## 🃏 Fiszki Anki
*Pełna talia: `anki/00-fundament.csv`.*
```
Co to SDKMAN!?;Menedżer wersji dla ekosystemu JVM — instaluje i przełącza wiele JDK oraz narzędzi (Maven, Gradle) przez PATH/JAVA_HOME.
Jak zainstalować SDKMAN?;curl -s "https://get.sdkman.io" | bash, potem source "$HOME/.sdkman/bin/sdkman-init.sh".
Czym jest "candidate" w SDKMAN?;Zarządzane SDK/narzędzie — np. java, maven, gradle, kotlin, scala.
Co oznacza sufiks "-tem" przy wersji Javy?;Dystrybucję Eclipse Temurin (Adoptium) — neutralny, w pełni open build OpenJDK.
Różnica sdk use vs sdk default?;use = tylko bieżąca powłoka (do zamknięcia); default = globalnie, przestawia symlink current dla nowych sesji.
Jak SDKMAN przełącza wersję pod spodem?;Symlinkiem ~/.sdkman/candidates/java/current; init w .bashrc dopisuje go do PATH i ustawia JAVA_HOME.
Do czego służy plik .sdkmanrc?;Przypina wersje SDK per projekt (java=, maven=); sdk env ustawia je w katalogu — odpowiednik .nvmrc.
Jak sprawdzić aktywną wersję Javy w SDKMAN?;sdk current java (lub java -version / which java wskazujący na ~/.sdkman).
Czym jest dystrybucja JDK?;Build OpenJDK od konkretnego dostawcy (Temurin, Corretto, Zulu, GraalVM) — zgodny ze standardem, różny support/licencja.
Czym GraalVM różni się od zwykłego JDK?;Dodaje native image (AOT do natywnego binarki) i polyglot — sufiks -graal/-graalce.
Czym jest jenv i czym różni się od SDKMAN?;jenv tylko przełącza JAVA_HOME między już zainstalowanymi JDK — nie instaluje ich, w przeciwieństwie do SDKMAN.
Po co LTS zamiast najnowszej Javy do produkcji/nauki?;LTS (8/11/17/21/25) ma długie wsparcie i stabilność; non-LTS wygasa szybko — używaj go tylko do testów feature'ów.
```

## 🔗 Powiązane
- [[ROADMAP]] · [[wiedza/00-fundament/jdk-jre-jvm]] · [[wiedza/00-fundament/wersje-javy-lts]] · [[wiedza/01-build/maven]] · [[wiedza/01-build/gradle]]
