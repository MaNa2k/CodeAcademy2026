# 🧰 Context engineering og AI harness

## 📋 Sammendrag

En AI-assistent er aldri bedre enn konteksten den får. I denne modulen lærer du å bygge en **AI harness**: et sett med filer i repoet ditt som gir GitHub Copilot det den trenger for å skrive kode som passer inn i *din* kodebase, hver gang.

Du lærer:

- hva en agent er, og hva en agent loop er
- hva en harness består av
- hvordan du skriver gode instruksjonsfiler
- hvordan du pakker arbeidsflyter inn i **skills**
- hvordan du vedlikeholder harnessen som en del av koden

**Tidsplan:** 30 min teori, deretter 60 min workshop med en pause på 5–10 min.

> 💡 Har du tatt **AI-Akademiet**? Da kjenner du igjen begrepene fra modulen *Context Engineering*. Her går vi videre og bygger en komplett harness i ditt eget repo.

---

## 🤖 Grunnlag: Hva er en agent?

En **språkmodell** (LLM) tar inn tekst og gir tekst tilbake. En **agent** er en språkmodell som kjører i en løkke og har tilgang til verktøy:

```
┌──────────────────────────────────────────────┐
│  1. Les oppgave + kontekst                   │
│  2. Tenk: hva er neste steg?                 │
│  3. Bruk et verktøy (les fil, kjør test ...) │
│  4. Observer resultatet                      │
│  5. Ferdig? Hvis nei → tilbake til 2         │
└──────────────────────────────────────────────┘
```

Dette er **agent-løkken** (agent loop). Copilot gjør dette når du bruker agentmodus i VS Code eller kjører `copilot` i terminalen. Den leser filer, kjører kommandoer, ser på feilmeldinger og prøver igjen.

Modellen vet ingenting om prosjektet ditt utover det som ligger i **kontekstvinduet**:

- systeminstruksjoner
- oppgaven din
- filene den har lest
- resultatene fra verktøyene den har brukt

Alt annet må den gjette. Når den gjetter, skriver den *generisk* kode i stedet for *din* kode.

---

## 🧰 Hva er en AI harness?

> **Harness = modell + kontekst + verktøy + regler**

Modellen velger du. Resten bygger du selv, og det bør ligge i repoet:

| Byggekloss | Hva den gjør | Fil i Copilot |
|---|---|---|
| **Instruksjoner** | Regler og fakta som *alltid* gjelder | `AGENTS.md` / `.github/copilot-instructions.md` |
| **Stibaserte instruksjoner** | Regler for bestemte deler av koden | `.github/instructions/*.instructions.md` |
| **Skills** | Kunnskap og arbeidsflyter som lastes *ved behov* | `.github/skills/<navn>/SKILL.md` |
| **Custom agents** | Spesialiserte roller med egne verktøy og egen modell | `.github/agents/<navn>.agent.md` *(modul 2)* |
| **Hooks** | Automatiske kontroller som kan stoppe handlinger | `.github/hooks/*.json` *(modul 2)* |

**Context engineering** betyr å bestemme *hva* modellen skal se, *når* den skal se det, og *hvor mye* den skal se.

### 🎯 Ansvaret ligger hos deg

Hos de fleste kunder er det ikke lagt til rette for AI på forhånd. Det finnes ingen ferdige instruksjoner, skills eller integrasjoner. Ansvaret for god kontekst ligger derfor hos hver enkelt i prosjektet. Det gode er at en harness bare er tekstfiler i repoet:

- Du trenger ingen spesiell infrastruktur fra kunden.
- Filene kan reviewes og versjoneres som all annen kode.
- Hele teamet får nytte av dem så snart de er merget.

> ⚠️ Avklar med prosjektet og kunden før du committer AI-filer til kundens repo. Det gjelder særlig hvis kunden har egne retningslinjer for bruk av AI.

---

## 📏 Instruksjonsfiler: grunnmuren

Copilot leser instruksjonsfilene automatisk i hver økt.

| Fil | Hvor | Merknad |
|---|---|---|
| `.github/copilot-instructions.md` | Rot | Leses av alle Copilot-flater. Ingen frontmatter. |
| `AGENTS.md` | Rot eller undermapper | Åpen standard som også brukes av andre verktøy. Den nærmeste filen i mappetreet gjelder. |
| `CLAUDE.md` | Rot | Støttes også, men velg ett format for teamet |

> 💡 I Copilot CLI lager `/init` et første utkast basert på kodebasen.

### Hva som hører hjemme i en instruksjonsfil

✅ **Ta med:**
- Hvordan du bygger, tester og linter, med eksakte kommandoer
- Arkitektur i noen få linjer: hvor ligger hva?
- Konvensjoner som *ikke* er åpenbare fra koden, som navngiving, feilhåndtering og logging
- Ting modellen tar feil av, for eksempel: «Bruk vår `ApiClient`, ikke `fetch` direkte»
- Hva som er forbudt, for eksempel: «Aldri endre migreringer som er kjørt»

❌ **Ikke ta med:**
- Ting modellen ser selv ved å lese koden
- Lange forklaringer og historikk
- Generelle råd som «skriv ren kode»
- Hemmeligheter, kundedata eller interne URL-er som ikke skal deles

**Tommelfingerregel:** Hold filen kort. Hver linje koster plass i kontekstvinduet i *hver* økt. Hvis du ikke kan forklare hvorfor en linje er der, fjerner du den.

### Stibaserte instruksjoner

Noen regler gjelder bare deler av koden. Legg dem i egne filer med `applyTo`:

```markdown
---
applyTo: 'src/test/**/*.java'
---
- Bruk JUnit 5 og AssertJ, ikke Hamcrest.
- Testnavn: `skal_<forventet>_når_<betingelse>`.
- Bruk `TestDataBuilder` i `src/test/support` i stedet for å lage objekter manuelt.
```

Filen lastes bare inn når agenten jobber med filer som matcher mønsteret. Slik slipper du å bruke kontekst på testregler når agenten skriver frontend-kode.

---

## 📦 Skills: kunnskap og arbeidsflyter ved behov

En **skill** er en mappe med en `SKILL.md`, og eventuelt maler, skript og eksempler. Det viktigste med skills er **progressiv lasting**:

1. Ved oppstart ser agenten bare **navn og beskrivelse** for alle skills. Det koster lite kontekst.
2. Når oppgaven passer beskrivelsen, laster agenten **innholdet** i `SKILL.md`.
3. Andre filer i mappen leses bare hvis skillen viser til dem.

Du kan derfor ha mange skills uten å fylle opp kontekstvinduet.

```
.github/skills/
└── nytt-endepunkt/
    ├── SKILL.md
    ├── mal-controller.java
    └── sjekkliste.md
```

```markdown
---
name: nytt-endepunkt
description: Lager et nytt REST-endepunkt etter teamets mønster (controller, service, DTO og tester). Bruk når brukeren ber om et nytt API-endepunkt.
---
# Nytt REST-endepunkt

1. Les `mal-controller.java` og følg strukturen.
2. Lag DTO-er i `api/dto`, aldri eksponer entiteter direkte.
3. Legg forretningslogikk i en service, ikke i controlleren.
4. Skriv en controller-test og en service-test.
5. Kjør `./mvnw test` og fiks feil før du sier at du er ferdig.
6. Gå gjennom `sjekkliste.md` til slutt.
```

**`description` er det viktigste feltet.** Det er beskrivelsen agenten bruker for å avgjøre om skillen skal brukes. Skriv derfor både *hva* skillen gjør og *når* den skal brukes.

### Skills som workflows

En workflow er en fast rekkefølge av steg, med sjekkpunkter underveis. Den kan skrives som en skill:

```markdown
---
name: feature-workflow
description: Teamets arbeidsflyt for nye features. Bruk når brukeren ber deg implementere en ny funksjon.
---
1. **Forstå:** Les relevante filer. Still spørsmål hvis kravet er uklart.
2. **Planlegg:** Skriv en kort plan og vent på godkjenning.
3. **Test først:** Skriv tester som feiler.
4. **Implementer:** Gjør testene grønne. Små steg.
5. **Verifiser:** Kjør hele testsuiten og linting.
6. **Oppsummer:** Hva ble endret, og hva bør en reviewer se på?
```

> 💡 **Prompt-filer** (`.github/prompts/*.prompt.md`) finnes fortsatt i VS Code, men de er på vei ut. Copilot CLI støtter dem ikke. Bruk skills til workflows. Da virker de både i VS Code og i CLI-en.

| Når bruker du hva? | |
|---|---|
| Gjelder det *alltid*? | → Instruksjonsfil |
| Gjelder det bestemte *filer*? | → Stibasert instruksjon |
| Gjelder det en bestemt *type oppgave*? | → Skill |
| Trenger oppgaven en egen *rolle*, egne verktøy eller en annen modell? | → Custom agent *(modul 2)* |

---

## ✅ Best practices: harnessen er kode

1. **Versjoner og review den.** Harness-filer går gjennom merge request som annen kode.
2. **Start lite.** Begynn med en kort instruksjonsfil og én skill. Legg til mer når du ser at det trengs.
3. **Hver feil er en mulighet.** Gjør AI-en samme feil to ganger, legger du inn en regel eller forbedrer en skill.
4. **Test harnessen.** Gi samme oppgave før og etter en endring, og sammenlign resultatene.
5. **Gi agenten måter å sjekke seg selv på.** Tester, linting og bygg er den beste konteksten. Agenten ser selv når noe er feil.
6. **Rydd jevnlig.** Utdaterte regler er verre enn ingen regler, fordi de forgifter konteksten.
7. **Start ny økt ofte.** En lang samtale fylles med gammel og irrelevant kontekst. En ny økt med god harness gir ofte bedre resultat.

---

## 🎓 Oppsummering

- En **agent** er en modell som bruker verktøy i en løkke. Den vet bare det som er i konteksten.
- En **harness** er instruksjoner, skills, agenter og regler som ligger i repoet.
- **Instruksjonsfiler** er korte og inneholder det som alltid gjelder.
- **Skills** lastes ved behov og passer for oppgavetyper og workflows.
- Harnessen **versjoneres, testes og forbedres** som annen kode.

**I workshopen** bygger du første versjon av harnessen i ditt eget repo.
