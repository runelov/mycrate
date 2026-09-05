# Veien videre

Kjente, bevisst utsatte avklaringer for MyCrate — ikke glemte, men ikke
løst nå. Se [docs/arkitektur.md](docs/arkitektur.md) for konteksten disse
kommer fra.

## Fra tracklist-arkitekturen (2026-09-05)

- **Eventual-consistency mellom `mycrate-backup.json` og `tracklists.json`.**
  Beslutning 3 i arkitektur.md valgte to uavhengige filer i `mycrate-db`,
  pushet/pullet hver for seg. Det åpner for at de to kan komme ut av sync —
  f.eks. en utgivelse fjernes fra collection i én push, men
  `tracklists.json` fra en tidligere/senere push fortsatt refererer til
  den, eller omvendt. Må adresseres eksplisitt når `githubPush()`/
  `githubPull()` utvides til å håndtere begge filene (f.eks.: `tracklists.json`
  er strengt tatt et cache-lag som alltid kan avvike i pluss — ekstra
  utgivelser der er harmløst og ryddes ved neste bulk-pass — men mangler i
  `tracklists.json` for noe som *er* i collection skal aldri tolkes som
  «utgivelsen har ingen låter», bare som «ikke hentet ennå»).
- **Generisk filter-/spørrings-UI i selve appen.** Beslutning 5 valgte
  frittstående skript (`mycrate-workspace/analysis/`) fremfor ny
  UI-funksjonalitet i `mycrate/app.js`, fordi det per nå er ad hoc,
  engangs-analyser. Hvis stedsnavn/egennavn/varighet-filtre (eller lignende)
  blir noe du vil kjøre jevnlig i selve appen fremfor via skript, er det en
  egen, betydelig UX/scope-utvidelse («avansert søk»-fane) som fortjener
  sin egen konsept-vurdering og et fase 3-kritikkpass (`design:design-critique`)
  — ikke noe å bygge stille inn som en biprodukt av tracklist-lagringen.
