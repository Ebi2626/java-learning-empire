# java-learning-empire

Baza wiedzy do nauki Javy pod pracę fullstacka (Angular + Java/Spring) — **głębokie zrozumienie bez black-boxów**.
Zbudowana jako **vault Obsidiana** i publikowana jako strona z interaktywnym grafem przez **[Quartz](https://quartz.jzhao.xyz)**.

🌐 **Strona online:** https://ebi2626.github.io/java-learning-empire/

## Struktura

| Ścieżka | Co |
|---|---|
| `content/` | Vault Obsidiana = treść publikowanej strony (notatki `wiedza/`, `ROADMAP.md`, `MOC.md`, projekt). |
| `content/_templates`, `content/anki`, `content/cwiczenia` | Wykluczone z publikacji (`ignorePatterns` w `quartz.config.ts`). |
| `quartz/`, `quartz.config.ts`, `quartz.layout.ts` | Silnik i konfiguracja Quartz. |
| `.github/workflows/deploy.yml` | Automatyczny deploy na GitHub Pages przy pushu na `main`. |

## Obsidian

Otwórz folder **`content/`** jako vault (`Open folder as vault`). Wikilinki, graf i backlinki działają tak samo online.

## Lokalny podgląd (jak strona online)

```bash
npm ci                 # raz, instalacja zależności
npx quartz build --serve   # http://localhost:8080
```

## Publikacja na GitHub Pages

Deploy jest automatyczny — wystarczy **raz** włączyć Pages, potem każdy push na `main` odświeża stronę:

1. GitHub → repo **Settings → Pages → Build and deployment → Source: `GitHub Actions`**.
2. `git push origin main` — workflow zbuduje i opublikuje stronę.
3. Strona pod `https://ebi2626.github.io/java-learning-empire/`.

> Zmieniłeś nazwę repo lub użytkownika? Zaktualizuj `baseUrl` w `quartz.config.ts`.
