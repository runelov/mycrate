# Arkitektur — Tracklist-basert analyse i MyCrate

Fase 2 (arkitektur/teknologivalg) i produktløypa for en ikke-triviell
utvidelse av eksisterende MyCrate — ikke et nytt produkt, se
[produktlope.md](../../../docs/produktlope.md). MyCrates konsept er
uendret (Discogs-samlingsbrowser, ingen backend, engelsk-språklig, se
`mycrate-workspace/CLAUDE.md`); dette dokumentet dekker kun de nye
beslutningene som følger av å hente og lagre full tracklist for hele
samlingen, for å muliggjøre spørringer som «alle låter med et stedsnavn i
tittelen», «alle låter under 2 minutter», kombinert med eksisterende felter
(styles, labels, år/tiår, formats).

## Bakgrunn

- Samlingen: 4220 objekter, **4162 unike release-id-er** (noen utgivelser
  finnes i flere pressinger/instanser — se pkt. under om nøkkelvalg).
- `GET /releases/{id}` returnerer full tracklist og brukes allerede
  on-demand av `fetchTracklist()` (`app.js:472`), men caches kun i en
  in-memory `Map` merket «session only» (`app.js:16`) — ingen låttitler er
  persistert noe sted i dag, uansett hvor mye brukeren har bladd.
- `storeEnrichmentFromReleaseData()` (`app.js:456`) henter *samme*
  Discogs-payload allerede for å regne `totalDurationSec`, men beholder
  ikke selve tracklist-arrayen — kun et aggregert tall. Kort sagt: appen
  gjør allerede requesten den trenger, den bare kaster halve svaret.
- `mycrate-backup.json` (mycrate-db) er allerede ~14 MB (collection,
  wantlist, prices, market, artists, labels, enrich, assumedCondition) —
  ingen tracklist der i dag.
- Et forsøk denne økten på å bulk-hente alle 4162 utgivelser anonymt, som
  et frittstående shell-script utenfor selve produktet, ble avbrutt — ikke
  fordi tilnærmingen var teknisk feil, men fordi *hvor* og *hvordan* dette
  skal lagres varig er et reelt arkitekturvalg, ikke noe som bør avgjøres
  implisitt av et engangsskript. Det er derfor dette dokumentet finnes.

---

## Beslutning 1 — Datalagring: bli i klient-only IndexedDB, ikke Worker+D1

**Valg:** ny nøkkel i eksisterende IndexedDB `kv`-store (`mycrate-db`,
object store `kv` i browseren — ikke å forveksle med git-repoet
`mycrate-db`), nøyaktig samme mønster som `priceCache`/`enrichCache`:
`idbGet('mycrate:tracklists')` / `idbSet('mycrate:tracklists', …)`, et
objekt keyed på **release-id** (ikke instance-id — se begrunnelse under).

**Alternativer vurdert:**
- **Worker + D1**, slik FungiFinder og Bondøya migrerte til (se
  erfaringsbanken: delt-PAT-mønsteret måtte migreres bort fra to ganger).
  Forkastet for denne utvidelsen. De migrasjonene løste et
  *tilgangsstyrings*-problem (delt PAT i localStorage, flere brukere, ingen
  sporing) — MyCrate har ingen delt hemmelighet og ingen andre brukere: det
  er én person, én Discogs-token i egen `localStorage`, ingen roller, ingen
  innlogging. Tracklist-utvidelsen endrer ingenting ved det bildet — den
  legger bare til enda en per-release cache, samme klasse data som
  `enrichCache`/`priceCache`/`marketCache` allerede er.
- **localStorage** i stedet for IndexedDB. Forkastet av samme grunn som
  collection/wantlist allerede ble flyttet til IndexedDB (se MyCrates
  CLAUDE.md): data som skalerer med samlingsstørrelse hører ikke hjemme i
  localStorage sin faste ~5 MB-kvote.

**Er dette fortsatt en triviell klient-only-skala, eller et vippepunkt?**
Eksplisitt vurdert, ikke bare antatt: en grov overslagsberegning (ikke
verifisert mot faktisk payload-størrelse) — 4162 utgivelser × typisk 6–10
spor × ~60–100 byte JSON per spor (tittel+varighet+posisjon) — lander på
størrelsesorden 2–4 MB totalt. Det er mindre enn dagens ~14 MB
`mycrate-backup.json`, og langt innenfor IndexedDB-kvoter (typisk hundreder
av MB til GB, ikke localStorage sitt ~5 MB-tak som var *grunnen* til at
collection/wantlist ble flyttet dit i utgangspunktet). **Konklusjon: dette
er ikke vippepunktet.** Et fremtidig vippepunkt ville vært et reelt behov
for automatisk multi-enhets-synk uten manuell backup-eksport, tunge
serverside-beregninger som ikke er realistiske i klienten, eller
datamengde en størrelsesorden større enn dette — ingen av delene er
tilfellet her. Revurder når/hvis noe av det blir aktuelt, ikke før.

**Nøkkelvalg — release-id, ikke instance-id:** tracklist er en egenskap
ved *utgivelsen*, ikke ved brukerens spesifikke eksemplar. Med 4220
objekter mot 4162 unike release-id-er sparer dette reell duplisert
lagring/henting for utgivelser som finnes i flere pressinger i samlingen —
samme prinsipp `enrichCache`/`priceCache` allerede følger.

---

## Beslutning 2 — Hentestrategi: hybrid, men UTEN ny bulk-knapp (revidert under implementasjon)

**Opprinnelig valg (før implementasjon):** lat bakgrunnsfylling pluss én ny,
eksplisitt bulk-backfill-knapp som gjenbruker «value pass»-mønsteret
(`runValuePass()`, `app.js:3585`).

**Det som faktisk ble bygget, og hvorfor planen endret seg:** ved
implementasjon viste det seg at MyCrate allerede har nøyaktig den bulk-passen
som var planlagt bygget ny — «Enrich my collection»
(`runEnrichPass()`/`runEnrichLoop()`, opprinnelig rundt `app.js:3223`, og sin
wantlist-tvilling `runEnrichWantPass()`). Den er allerede: full-samling
(ikke bare gjeldende visning, i motsetning til value pass), token-krevende,
avbrytbar/gjenopptakbar, viser fremdrift, og — avgjørende — kaller
`fetchEnrichment()` → `storeEnrichmentFromReleaseData()`, som henter
*nøyaktig* samme `/releases/{id}`-payload som tracklist trengs fra. Den
kastet bare tracklist-halvparten av svaret (beholdt kun et aggregert
`totalDurationSec`).

**Faktisk implementasjon, to deler:**
1. **Lat bakgrunnsfylling:** `storeEnrichmentFromReleaseData()` persisterer nå
   `data.tracklist` til den nye IndexedDB-nøkkelen (`mycrate:tracklists`) hver
   gang den kalles — fra `fetchTracklist()` (modal åpnes) OG fra
   `fetchEnrichment()` (enrich-pass). Ingen ny nettverksrequest, kun at
   resultatet ikke lenger kastes.
2. **Bulk-backfill: ingen ny knapp — «Enrich my collection»/«Enrich my
   wantlist» dekker det.** Eneste kodeendring: «mangler enrichment»-filteret
   i `runEnrichPass`/`runEnrichWantPass`/`autoEnrichCrate`/`autoEnrichWant`
   endret fra `!enrichCache[r.id]` til `!enrichCache[r.id] ||
   !trackDataCache[r.id]` — ellers ville allerede-enrichede utgivelser
   (`enrichCache` hadde 7426 oppføringer ved implementasjonstidspunktet, langt
   mer enn de 4162 unike release-id-ene i samlingen — historiske oppføringer
   fra tidligere eide utgivelser telles også med) aldri fått tracklist
   backfilt uten en full, kostbar «Refresh all» (som re-henter ALT, ikke bare
   tracklist). Samme ene request dekker fortsatt begge halvparter — endringen
   koster ingenting ekstra når begge allerede finnes, og fyller kun
   tracklist-hullet for øvrig.

**Alternativer vurdert (fortsatt gyldige som resonnement, konklusjonen endret
seg kun på «trenger vi en ny knapp»):**
- **Kun lat fylling.** Fortsatt forkastet alene — se opprinnelig begrunnelse
  under. Løsningen er derfor fortsatt en hybrid, bare at «den eksplisitte
  bulk-delen» viste seg å allerede eksistere i stedet for å måtte bygges.
- **Kun bulk-pass.** Fortsatt forkastet alene, se Beslutning 6.

Opprinnelig begrunnelse, uendret: kun lat fylling konvergerer bare for
utgivelser brukeren faktisk åpner i UI-et — for en samling på 4162 vil store
deler aldri bli åpnet i praksis, så «kombiner med eksisterende
felter»-analysen ville aldri fått full dekning uten en eksplisitt vei til
100 %. Kun bulk-pass tvinger en lang ventetid (70–90 min med token, 2,5–3 t
uten, se Beslutning 6) *før* brukeren kan gjøre noe nytt, når brukeren
uansett allerede genererer denne dataen gratis ved normal bruk av appen over
tid. Kombinasjonen dekker begge: ingen kostnad ved normal bruk, og et
bevisst, synlig alternativ for brukeren som vil ha full dekning nå fremfor
senere — «synlig alternativ» var allerede der i form av «Enrich my
collection», bare uten at det visste om tracklist ennå.

**Lærdom for erfaringsbanken (kandidat for fase 8-retro):** en
arkitekturvurdering som antar «vi må bygge X» bør sjekke om et eksisterende
mønster i koden allerede gjør det meste av jobben, før den spesifiserer en ny
UI-komponent — her sparte det både en ny knapp, nye state-variabler, og et
nytt kritikkpass i fase 3, uten tap av funksjonalitet.

**Oppfølgingsbug funnet i produksjon (2026-09-05), rettet i commit
`736ad33`:** det utvidede «missing»-filteret i `runEnrichPass()`/
`runEnrichWantPass()` (over) plukket riktig ut poster som manglet
`trackDataCache` selv om de allerede hadde `enrichCache` — men selve
`fetchEnrichment(releaseId, force)` hadde sin EGEN, uavhengige
short-circuit (`if(!force && enrichCache[releaseId]) return
enrichCache[releaseId];`) som ikke visste noe om `trackDataCache`. Netto
effekt: for en samling der nesten alt allerede var enrichet fra før (som
brukerens — `enrichCache` hadde 7426 oppføringer), returnerte
`fetchEnrichment` umiddelbart for praktisk talt hver post uten noen gang å
faktisk hente fra Discogs — «Enrich my collection» rapporterte «Done —
checked 4220» nesten momentant, men **null** tracklists ble persistert
(bekreftet direkte mot brukerens IndexedDB: 0 nøkler i
`mycrate:tracklists` etter et fullført pass). Fikset ved at
`fetchEnrichment` nå også krever `trackDataCache[releaseId]` for å ta
short-circuit-veien — samme logikk som den ytre filtreringen, bare ett
lag dypere. Lærdom: når «missing» omdefineres til å dekke to felt, må
*alle* stedene som sjekker «er dette allerede gjort» oppdateres samtidig,
ikke bare det ytterste filteret — en indre cache-sjekk lenger nede i
kallkjeden kan stille overstyre en riktig ytre beslutning.

---

## Beslutning 3 — Backup-format: egen fil i `mycrate-db`

**Avgjort av bruker 2026-09-05:** alternativ (b) — `mycrate-db/tracklists.json`,
pushet/pullet uavhengig av hovedbackupen (`mycrate-backup.json`).

MyCrates egen CLAUDE.md er eksplisitt: `mycrate-db` er en «dumb JSON
blob»-mål, og enhver endring av den rollen krever bekreftelse fra bruker
først — dette er nå gitt, og `mycrate-db` går dermed fra «én JSON blob» til
«to filer», hver med egen push/pull.

**Alternativer som ble vurdert:**
- **(a) Slå sammen i eksisterende `mycrate-backup.json`.** Enklest, ingen
  endring av push/pull-logikken eller antall filer. Filen ville vokst fra
  ~14 MB til trolig ~17–18 MB (se størrelsesoverslag i Beslutning 1) —
  fortsatt innenfor det `githubPush()`/`githubPull()` allerede håndterer via
  Git Data API-fallback (`needsGitDataApi`). Forkastet: tracklist (i praksis
  skrives-én-gang, endrer seg aldri) hadde blitt fanget opp i samme fil og
  samme push som collection/prices/market (endrer seg ofte) — hver backup
  hadde blitt tyngre å skrive/lese enn nødvendig for den delen som faktisk
  endres. Dette er nettopp begrunnelsen for at (b) ble valgt i stedet.
- **(c) Ikke i backup i det hele tatt, kun lokal IndexedDB.** Forkastet:
  brukeren ønsket varig, gjenopprettbar lagring på tvers av enheter/
  cache-tømming, samme fordel som resten av backup-flyten gir — selv om
  tracklist i prinsippet er re-hentbar fra Discogs, er poenget med
  bulk-backfillen (Beslutning 2) nettopp å slippe å gjøre den ~70–90 min/
  2,5–3 t jobben på nytt per enhet.

**Konsekvens for implementasjon** (se også fallgruve-seksjonen under):
`githubPush()`/`githubPull()` (`app.js`) må utvides til å håndtere to
uavhengige filer i `mycrate-db`, ikke én — inkludert samme
bootstrap-på-tomt-repo-håndtering (Contents API først, deretter Git Data
API) og samme «200 OK uten brukbart `content`»-sjekk som i dag gjelder
`mycrate-backup.json`, nå for begge filer separat. To uavhengige pusher
betyr også at de to filene kan komme ut av sync med hverandre — se
eventual-consistency-punktet under, som må adresseres i implementasjonen,
ikke oppdages i drift.

---

## Beslutning 4 — Datamodell: rå felter, ikke forhåndsberegnede analyse-flagg

**Valg:** lagre kun rå tracklist-felter per spor — `title`, `duration_sec`
(konvertert via eksisterende `parseDurationToSeconds()`), `position`,
implisitt koblet til `release_id` (nøkkelen i det nye objektet) — koblet
mot eksisterende utgivelse-nivå-felter (styles, labels, genres, år/tiår,
formats) som allerede finnes i collection/wantlist-dataene. **Ingen**
forhåndsberegnede derived-felter som `hasPlaceName`/`hasProperNoun` lagres
sammen med rådataen.

**Alternativ vurdert:** forhåndsberegne og lagre slike boolske flagg ved
henting. Forkastet: brukerens uttrykte behov er *vilkårlige, fremtidige,
kombinerbare* filtre («et stedsnavn», «et egennavn», «under 2 minutter»,
og senere kombinasjoner med styles/tiår) — en liste over stedsnavn eller
egennavn er noe som forbedres/rettes over tid (falske positiver fjernes,
nye navn legges til), og et forhåndsberegnet flagg ville blitt stille
utdatert hver gang kriteriet endres, uten noen migrasjonsmekanisme til å
oppdage det. Rå tracklist-tekst er den eneste formen som er stabil nok til
å gjenbrukes for spørringer ingen har tenkt på ennå. Kostnaden er at hver
analyse regner selv (billig — tekstmatching mot noen tusen tracktitler er
millisekunder), ikke at dataen må hentes på nytt.

---

## Beslutning 5 — Hvor kjøres analysen: frittstående skript, ikke ny app-funksjon

**Valg:** stedsnavn-/egennavn-/varighetsanalyser bygges **ikke** som ny
UI-funksjonalitet i `mycrate/app.js`. I stedet: en enkel eksportvei ut av
persistert data (IndexedDB-eksport til JSON, eller lesing direkte fra
`mycrate-backup.json` når/hvis Beslutning 3 inkluderer tracklist), som mates
inn i frittstående Python/Node-skript utenfor selve app-koden.

**Begrunnelse:**
- MyCrates egen CLAUDE.md definerer produktet eksplisitt som en
  «record-collection browser», tre filer, ingen build-steg, én IIFE. Et
  åpent, voksende sett av ad hoc personlige spørringer («stedsnavn»,
  «egennavn», «under 2 minutter», og alt som kommer etter) er per
  definisjon et bevegelig mål — å bygge generisk filter-UI for «alle
  fremtidige kombinasjoner» er analyseverktøy-scope, ikke browser-scope.
- Et faktisk Python-forsøk denne økten (`geonamescache`-biblioteket for
  stedsnavn-matching) fungerte greit offline mot eksporterte data — ett
  konkret datapunkt for at «eksport + skript» er godt nok uten ny
  appfunksjonalitet, ikke bare en antakelse.
- `mycrate-db` er eksplisitt *ikke* stedet for skript (se dens egen
  CLAUDE.md-seksjon: «Ingen skript, ingen GitHub Actions... hvis bedt om å
  legge til automatisering her, bekreft først»), og `mycrate/` (appen) er
  eksplisitt tre statiske filer. Anbefalt plassering: en ny mappe på
  workspace-rot-nivå, `mycrate-workspace/analysis/` (sporet av
  `mycrate-workspace`s eget git-repo, `github.com/runelov/mycrate-workspace`
  — verken app- eller db-repoet berøres), som leser eksportert
  tracklist-JSON og kjører frittstående, uten avhengighet til noen server
  eller build-prosess.

**Alternativ vurdert:** bygge et generisk filter-/spørringsgrensesnitt i
appen (f.eks. en «avansert søk»-fane med fritekst-mønstre). Forkastet for
nå — ikke fordi det er en dårlig idé i seg selv, men fordi det er en
betydelig UX/scope-utvidelse som fortjener sin egen konsept-vurdering
(og fase 3-kritikkpass) hvis/når det blir en gjentakende, ikke
engangs-analyse. Skriv det ned i `veien-videre.md` (se Overlevering) som en
kjent fremtidig avklaring i stedet for å bygge det nå.

---

## Beslutning 6 — Rate-limit-realiteten: token kreves for bulk-knappen, ikke for lat fylling

**Valg:** den eksplisitte bulk-backfill-knappen (Beslutning 2, punkt 2)
**krever** en satt Discogs-token, akkurat som `fetchPriceSuggestions()`
allerede gjør (`app.js:517`, kaster «Add a personal access token…» uten
token) — knappen er deaktivert/viser samme melding uten token. Lat fylling
(Beslutning 2, punkt 1) krever **ikke** token — den skjer i det tempoet
brukeren uansett browser i, med samme pacing (`discogsFetch()`) som alt
annet.

**Begrunnelse:** 4162 utgivelser tar ~70–90 min autentisert (60/min) mot
~2,5–3 t anonymt (25/min). Et eksplisitt, brukerinitiert bulk-kall som tar
opptil tre timer anonymt er en dårlig opplevelse og spiser samtidig hele
den anonyme kvoten mot ethvert annet Discogs-kall brukeren måtte gjøre i
appen i mellomtiden (søk, prissjekk, etc.) — value pass-mønsteret har
allerede løst dette ved å kreve token, samme resonnement gjenbrukes
direkte i stedet for å finne opp en ny regel.

**Alternativ vurdert:** ikke bygge en eksplisitt bulk-knapp i det hele
tatt, kun stole på lat fylling siden det uansett ikke er tidspress.
Forkastet som eneste løsning av samme grunn som i Beslutning 2 — lat
fylling alene gir ingen realistisk vei til full dekning for en samling på
4162 utgivelser hvis brukeren vil kunne kjøre kombinerte analyser i nær
fremtid, ikke «når jeg tilfeldigvis har åpnet alt».

---

## Kjente fallgruver fra erfaringsbanken, eksplisitt vurdert

- **«Gjenbruk av mønster ≠ arkitekturvurdering»** (erfaringsbank,
  Arkitektur/teknologivalg): eksplisitt sjekket i Beslutning 1 — spørsmålet
  «er Worker+D1-begrunnelsen fra FungiFinder/Bondøya fortsatt relevant her»
  ble stilt og besvart nei, ikke bare antatt fra vanen.
- **Kryss-produkt-avhengigheter:** ingen finnes i denne utvidelsen — ingen
  ny delt konto, domene eller hemmelighet innføres. Nevnes eksplisitt her
  for å bekrefte at spørsmålet ble stilt, ikke hoppet over.
- **Eventual-consistency/race ved GitHub-dispatch:** ikke relevant for
  denne utvidelsen i seg selv (ingen ny GitHub Actions-dispatch innføres),
  men blir relevant *hvis* Beslutning 3 lander på (b) — en egen
  `tracklists.json`-fil i `mycrate-db` pushet uavhengig av hovedbackupen
  åpner for at de to filene kan komme ut av sync seg imellom (f.eks. en
  utgivelse fjernes fra collection i én push, men tracklist-filen fra en
  annen push fortsatt refererer til den). Skal adresseres eksplisitt i
  implementasjonen dersom (b) velges, ikke oppdages i drift.
- **«En løsning i ett produkt overføres ikke automatisk»** (kodepraksis):
  denne utvidelsen gjenbruker bevisst to eksisterende MyCrate-interne
  mønstre (value pass for avbrytbar bulk, IndexedDB `kv`-store for
  skalerende data) i stedet for å finne opp nye — ingen nytt mønster som
  trenger å spres til andre produkter oppstår her.

---

## Overlevering

**Status: implementert** (2026-09-05, på brukerens eksplisitte instruks om å
gå rett til implementasjon). Fase 3 (UX/skisse, `design:design-critique`) ble
bevisst hoppet over av brukeren — i praksis uproblematisk her siden
Beslutning 2 endte med **ingen ny UI-flate i det hele tatt** (se revidert
Beslutning 2): eksisterende «Enrich my collection»/«Enrich my wantlist»-knapp
og -fremdriftsvisning dekket bulk-behovet, så det var intet nytt å
kritikkvurdere. Hadde den opprinnelige planen (ny knapp) blitt fulgt, hadde
det vært en reell mangel å hoppe over fase 3.

Fase 5 (sikkerhet) og fase 6 (test-strategi) eies fortsatt av hhv.
sikkerhetssjekklisten/`security-review` og `testskriver` — ingen av delene
er dekket av dette dokumentet eller kjørt for denne utvidelsen. Ingen nye
hemmeligheter eller eksterne avhengigheter ble introdusert (samme
Discogs-token, samme GitHub-PAT-mønster), så risikoen er lav, men er ikke
formelt vurdert.

Kjente fremtidige avklaringer som bevisst *ikke* løses nå er samlet i
[veien-videre.md](../veien-videre.md).

## Faktisk endrede filer (implementasjon)

- `app.js`: `trackDataCache` (ny persistert state, erstatter den
  session-only `trackCache`-Map-en), `saveTrackDataCache()`,
  `extractTracklist()`, utvidet `storeEnrichmentFromReleaseData()` og
  `fetchTracklist()`, utvidede «missing»-filtre i
  `runEnrichPass`/`runEnrichWantPass`/`autoEnrichCrate`/`autoEnrichWant`,
  `clearCacheBtn`-håndtereren rydder nå også `mycrate:tracklists`,
  `githubPushFile()`/`githubPullFile()` (delte hjelpefunksjoner refaktorert
  ut av de opprinnelige `githubPush()`/`githubPull()`), nye
  `githubPushTracklists()`/`githubPullTracklists()`/`tracklistsPathFor()`,
  og Push/Pull-knappenes handlere utvidet til å håndtere `tracklists.json`
  som andre fil.
- Ingen endringer i `index.html`/`styles.css` — ingen ny UI-flate, se over.
