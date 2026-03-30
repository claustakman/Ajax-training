# CLAUDE.md – Ajax U11 Træningsplanlægger

## Projektbeskrivelse
Single-page webapp til planlægning af håndboldtræninger for Ajax U11 piger.
Bygget som én enkelt HTML-fil med vanilla JS. Ingen build-step, ingen frameworks.

**Live URL:** https://claustakman.github.io/Ajax-training/
**Seneste fil:** `ajax-u11-traening-v6.html`

---

## Arkitektur

### Én fil, ingen dependencies (undtagen CDN)
- Al kode er i én `.html`-fil – HTML + CSS + JS
- XLSX-bibliotek indlæses fra CDN ved startup
- Ingen npm, ingen build, ingen bundler

### State
Al applikationsstate ligger i ét globalt objekt `S`:
```js
S = {
  tab,           // aktiv tab
  draft,         // træning under redigering (null = listevisning)
  trainings,     // alle træninger
  halCatalog,    // haltrænings-øvelser
  fysCatalog,    // fysisk trænings-øvelser
  quarters,      // årshjul med temaer
  sectionTypes,  // sektionstyper med tag-mapping (KV-gemt)
  templates,     // træningsskabeloner
  users,         // brugere med PIN og rolle
  modal,         // aktiv modal { type, ...data } eller null
  currentUser,   // { id, name, role } eller null
}
```

### Render-model
`render()` → `innerHTML` på `#app` → `bind()` (sætter event listeners).
Undtagelse: `renderSecs()` opdaterer kun sektionsdelen (bevarer scroll).

### Persistens
Data gemmes i **Cloudflare KV** via en Worker. Lokalt bruges `localStorage` som fallback.
`scheduleSave(type)` debouncer gemning med 1500ms.

**KV-nøgler:** `trainings`, `quarters`, `hal`, `fys`, `templates`, `users`, `sectionTypes`

---

## Cloudflare Workers

| Worker | URL | Formål |
|--------|-----|--------|
| traening-worker | https://traening-worker.claus-takman.workers.dev | KV-datapersistens |
| holdsport-worker | https://holdsport-worker.claus-takman.workers.dev | Proxy til Holdsport API |
| ajax-ai-proxy | https://ajax-ai-proxy.claus-takman.workers.dev | Proxy til Anthropic API |

---

## AI-forslag (vigtig logik)

### Hele træningen (`runAISuggest`)
1. Temaer hentes **kun** fra `S.draft.themes` (aktivt valgte på træningen – ikke alle årshjulstemaer)
2. Sektionstyper dedupliceres inden kald (max én af hver type)
3. Øvelseslister bygges med **strict tag-filter**: øvelse skal matche sektionstype-tags fra `getSectionTypes()`
4. Sektioner nummereres `SEKTION 1, SEKTION 2…` i promptet
5. AI's `type`-felt i svaret ignoreres — type tildeles fra vores eget `secsMins` array **ved position**
6. `validateAISections` filtrerer ukendte øvelses-ID'er fra

### Per sektion (`#aisec-go` handler)
- Sender `[]` som temaer til `runAISuggest` — funktionen henter selv fra `S.draft.themes`
- Returnerer ét element fra AI-svaret og sætter det på sektionen

### Hvorfor position og ikke type?
AI returnerer konsekvent `"type": "fysisk"` uanset instruktion. Løsningen er at nummerere sektionerne og matche på position i stedet.

---

## Sektionstyper
Defineret i `DEFAULT_SECTION_TYPES` og overskrevet af KV-gemte `S.sectionTypes`.
Hver type har:
- `id`, `label`, `color`, `cls`
- `tags[]` – bruges til at filtrere relevante øvelser til AI
- `themes[]` – reserveret til fremtidig brug
- `required` – hvis true, altid med i AI-forslag

`getSectionTypes()` returnerer KV-version hvis tilgængelig, ellers defaults.

---

## Kendte mønstre og gotchas

### Mins følger med ved sektions-swap
`movesec`-handleren flusher alle `[data-secmins]` inputs til `S.draft` **før** swap.
Ellers arver sektioner hinanden mins pga. position-bundne DOM-inputs.

### Nested template literals crasher
Python `str.replace()` korrupterer nested backticks i JS template literals.
**Brug altid `str_replace`-tool direkte, eller byg strenge med `+` concatenation.**
Syntakstjek med Acorn efter ændringer: `node -e "require('/tmp/node_modules/acorn').parse(...)"`

### `render()` vs `renderSecs()`
- `render()` re-renderer hele appen + kalder `bind()` – bruger til tab-skift, modal-åbning mv.
- `renderSecs()` opdaterer kun `#secs-wrap` + kalder `updateTotalMins()` – bruges efter sektionsændringer

### updateTotalMins
Opdaterer `#total-mins-display` (i sektionskortet, altid synlig) og `#total-mins-display-info` (i Oplysninger-sektionen).
Kaldes automatisk fra `renderSecs()` og ved input på `[data-secmins]`.

### Roller
- `guest` – kan se
- `trainer` – kan redigere træninger og katalog
- `admin` – kan alt inkl. årshjul og brugerstyring

`requireUnlock(action)` kræver minimum `trainer`.

---

## Vigtige funktioner

| Funktion | Beskrivelse |
|----------|-------------|
| `render()` | Re-renderer hele appen |
| `renderSecs()` | Opdaterer kun sektionslisten |
| `bind()` | Sætter alle event listeners efter render |
| `runAISuggest(secsMins, [], vary)` | Kalder AI og returnerer sektioner med øvelser |
| `validateAISections(sections)` | Filtrerer ugyldige øvelses-ID'er fra AI-svar |
| `getSectionTypes()` | Returnerer aktive sektionstyper (KV eller defaults) |
| `scheduleSave(type)` | Debounced gem til KV |
| `updateTotalMins()` | Opdaterer minutter-display live |
| `flushSecMins()` | Læser alle mins-inputs ind i `S.draft` |
| `durMin(start, end)` | Beregner minutter mellem to tidspunkter |
| `totalMins(training)` | Summerer sektionsminutter (håndterer parallelle grupper) |

---

## Deployment
Filen deployes manuelt til GitHub Pages:
```
https://claustakman.github.io/Ajax-training/
```
Fil hedder `index.html` i repo – upload `ajax-u11-traening-vX.html` omdøbt til `index.html`.

---

## Hvad der virker (v6)
- KV-baseret datapersistens
- Træning opret/rediger/arkiver/slet
- Øvelseskatalog med import/export (CSV + Excel)
- `defaultMins` på øvelser (bruges af AI til at estimere antal øvelser)
- AI træningsforslag – hele træning + per sektion
  - Korrekte sektionstyper i output (position-baseret matching)
  - Temaer fra træningen (kun aktivt valgte)
  - Strict tag-filter per sektionstype
  - Minutter-summering i modal med live-opdatering
- Sektionstyper med tag-mapping i Indstillinger (KV-gemt)
- Billede/illustration på øvelser (base64, upload + paste)
- Holdsport import
- PIN-baseret login (Gæst/Træner/Admin)
- Mobil-optimeret (bottom sheet modaler)
- Live minutter-display i sektionskort (altid synlig, opdateres instant)
- Sektions-swap bevarer korrekte minutter

## Kendte udestående
- Holdstyr-app (index-2.html) ikke migreret til KV
- AI kvalitet afhænger af at øvelser har `defaultMins` + korrekte tags
