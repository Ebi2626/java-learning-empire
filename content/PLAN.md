# PLAN — proces nauki i projektu

Rozpisanie pięciu celów na konkretny, powtarzalny proces. Towarzyszy [[ROADMAP]] (co) — tu jest „jak".

## Cele (z briefu)
1. **Praktyczna roadmapa** zagadnień, narzędzi, frameworków, wzorców → [[ROADMAP]] ✅ gotowa.
2. **Głębokie zrozumienie** języka i zastosowania (gdzie używane, wąskie gardła, deployment) → struktura 3-poziomowa (Co/Jak/Dlaczego) w każdej notatce.
3. **Wiedza, kiedy obszar jest kompletny** (brak black-boxów) → sekcja *Kryteria opanowania* + *Black-box check* + *Pytania rekrutacyjne* w każdej notatce.
4. **Baza wiedzy do Obsidiana + fiszki Anki** → cały katalog = vault; talie CSV w `anki/`.
5. **Projekt docelowy:** system rekrutacyjny dla stowarzyszeń (Flowable, multitenancy, cloud-native) → `projekt-rekrutacja/`.

## Pętla nauki (na każdy temat)
1. **Czytaj/eksperymentuj** — `jshell`/mały kod w `cwiczenia/`.
2. **Notuj** wg szablonu `_templates/temat.md` (Co → Jak → Dlaczego).
3. **Domknij black-boxy** — uzupełnij *Black-box check*; jeśli coś jest „magią", to nie koniec.
4. **Fiszki** — dopisz pytania do talii Anki danej fazy.
5. **Zweryfikuj** — odpowiedz na *Pytania rekrutacyjne* na głos, bez notatek.
6. **Oznacz status** w nagłówku i w [[MOC]].

## Strategia (kolejność realna, nie liniowa)
- **Najpierw zatrudnialność:** fazy 0–1 → 4–5 → 6–7. Po tym potrafisz budować backend pod Angulara.
- **Projekt jako spina:** od fazy 5 zaczynamy `projekt-rekrutacja` i dociągamy 8–12 „just-in-time".
- **Głębia (2,3,9) równolegle** — gdy w projekcie natkniesz się na temat (np. N+1 → faza 6; proxy → faza 5).

## Definicja ukończenia całości
- Wszystkie notatki 🔴 ze statusem `opanowany`.
- Projekt rekrutacyjny: działa lokalnie na k8s, multitenant, proces Flowable edytowalny przez admina tenantu, z CI/CD, metrykami i testami.
- Potrafisz przeprowadzić kogoś przez architekturę i bronić każdej decyzji (ADR-y).

## Kolejne kroki (do uzgodnienia)
Patrz koniec odpowiedzi asystenta — proponowana kolejność dosypywania treści (notatki fazy 0–1, talie Anki, setup projektu).
