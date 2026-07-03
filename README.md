# Nauka Javy — fullstack (Angular + Java)

System nauki Javy zaprojektowany dla osoby z mocnym frontendem (Angular) i podstawami Javy
ze studiów. Cel: dojść do poziomu zawodowego fullstacka **bez black-boxów** — umieć
odpowiedzieć na dowolne pytanie z danego obszaru i rozumieć rzeczy „od pierwszych zasad".

## Co tu jest

| Plik / katalog | Po co |
|---|---|
| [[ROADMAP]] | Praktyczna roadmapa 0–12: tematy, narzędzia, frameworki, wzorce. **Zacznij tutaj.** |
| [[PLAN]] | Szczegółowy plan procesu nauki + projektu (jak realizujemy cele). |
| [[MOC]] | Map of Content — mapa nawigacyjna vaultu Obsidiana. |
| `wiedza/` | Baza wiedzy (vault Obsidiana). Notatki tematyczne z kryteriami opanowania. |
| `_templates/` | Szablony notatek (Obsidian Templater / wbudowane). |
| `anki/` | Talie fiszek (CSV) + instrukcja importu do Anki. |
| `cwiczenia/` | Małe projekty kodu (Maven) — praktyka. |
| `projekt-rekrutacja/` | Docelowy projekt: system rekrutacji dla stowarzyszeń (Flowable, multitenancy, cloud-native). |

## Jak używać (Obsidian)

1. W Obsidianie: **Open folder as vault** → wskaż ten katalog (`nauka/`).
2. Wejdź w [[MOC]] albo [[ROADMAP]].
3. Każda notatka tematyczna ma na dole sekcję **„Kryteria opanowania"** — to twój checklist:
   gdy potrafisz odpowiedzieć na wszystkie pytania *bez zaglądania*, temat jest domknięty.
4. Fiszki: importuj pliki z `anki/` (patrz [[anki/README]]).

## Zasada „brak black-boxów"

Każdy temat ma trzy poziomy:
- **Co** — definicja, API, jak użyć.
- **Jak** — co się dzieje pod spodem (np. jak działa GC, classloader, proxy Springa).
- **Dlaczego / kiedy** — kompromisy, wąskie gardła, kiedy NIE używać.

Notatka jest „kompletna" dopiero gdy masz wszystkie trzy poziomy i przeszedłeś
**Black-box check** (lista rzeczy, które musisz umieć wyjaśnić od podstaw).

## Środowisko (wykryte na tej maszynie, 2026-06-29)

- **JDK 25** (Red Hat build). Uwaga: rynek pracy to głównie **LTS: 17 i 21** — patrz [[ROADMAP]] faza 0.
- **Podman 5.8** (zamiennik Dockera, Fedora) — komendy `docker` zadziałają jako alias do `podman`.
- **Node 22 / npm 10** — dla części Angularowej.
- Brak Mavena/Gradle/SDKMAN — instalacja w fazie 0.
