# Programmatisk styring av Google TV

Referansenotat til modell C i
[arkitektur-diskusjonsgrunnlaget](arkitektur-diskusjonsgrunnlag.md), etter at
valget falt på en Google TV-enhet som innholdsleverandør.

**Merk om sikkerhet i påstandene:** styringskanalene under er godt etablerte og
brukes blant annet av Home Assistant. Men *hvilke pakkenavn og deep links som
faktisk finnes for NRK og TV 2*, og hvordan den konkrete enheten oppfører seg,
er noe som må måles på maskinvaren — ikke noe jeg vil påstå på forhånd. §4 og
§10 er derfor skrevet som oppskrifter for å finne ut av det, ikke som fasit.

---

## 1. Kort svar

| Vi vil kunne | Går det? |
|---|---|
| Sende tastetrykk (D-pad, play/pause, volum, hjem, tilbake) | ✅ Trivielt, flere veier |
| Vekke og dvale-sette boksen | ✅ |
| Starte en bestemt app | ✅ |
| **Gå rett til en bestemt kanal i appen** | ⚠️ **Avhenger av deep links — dette er hele spørsmålet, se §4** |
| Lese hva som faktisk spiller nå | ✅ Og det er viktigere enn det høres ut, se §5 |
| Styre volum | ✅ Flere veier, men pass på responstid, se §7 |
| Slå på TV og bytte inngang | ✅ Ofte gjør boksen det selv via CEC, se §6 |

Det som *ikke* går, og som det er verdt å slå fast med en gang: vi kan ikke
skript-styre innsiden av en app på noen robust måte. Alt vi gjør må uttrykkes
som «start denne adressen» eller «trykk denne tasten» — og av de to er bare den
første pålitelig over tid.

---

## 2. Enhetsvalg og oppsett av boksen

**Chromecast med Google TV er utgått.** Google avviklet både 4K- og HD-varianten
— de ble fjernet fra Google Store i februar 2025 — og erstattet dem med **Google
TV Streamer (4K)**, en set-top-boks i stedet for en dongle, til omtrent dobbel
pris. Eksisterende Chromecast-enheter får fortsatt oppdateringer, men ikke
nødvendigvis nye funksjoner.

| | Chromecast med Google TV (4K) | Google TV Streamer (4K) |
|---|---|---|
| Lagring | 8 GB | 32 GB |
| Minne | 2 GB | 4 GB |
| Nettverk | Kun Wi-Fi (ethernet krevde ekstra adapter) | **Gigabit ethernet** + Wi-Fi |
| Status | Utgått | Gjeldende |

For dette prosjektet mener jeg valget er enkelt, og det handler ikke om ytelse:

**Lagringen.** 8 GB er den kjente svakheten ved Chromecast med Google TV. Med
noen apper installert og et par års oppdateringer fylles den, og resultatet er
at enheten blir treg og oppfører seg uforutsigbart. Det er nøyaktig den
feilmodusen vi ikke har råd til: den kommer gradvis, den gir ingen tydelig
feilmelding, og brukeren kan ikke beskrive den for noen.

**Ethernet-porten.** Nå som *alt* innhold ligger bak nettverket, er en
Wi-Fi-dropout det samme som at TV-en slutter å virke. Kabel til både boksen og
Pi-en fjerner hele den klassen av feil, og er antakelig tiltaket med best effekt
per krone i hele prosjektet.

Å kjøpe en brukt Chromecast fordi den er billigere, betyr å sette en utgått
enhet med for lite lagring til å stå og virke uten tilsyn i mange år. Det vil
jeg fraråde.

### Oppsettsjekkliste

Hvert punkt her fjerner en mulig felle for blind automatikk (§8). Verdt å gjøre
én gang, grundig, og skrive ned mens det gjøres:

- **Egen Google-konto for boksen.** Ikke en pårørendes personlige. Ingen
  private data på en enhet i et annet hjem, og en ren startskjerm uten
  anbefalinger vi ikke styrer.
- **Kablet nettverk**, og fast IP-reservasjon på ruteren for både boks og Pi.
  Merk rekkefølgen: ADB-utforskingen i §4 må gjøres over **Wi-Fi**, fordi
  trådløs feilsøking ikke lar seg slå på over ethernet (§3). Kjør derfor spiken
  først, og legg kabelen etterpå.
- **Slå av skjermsparer og ambient-modus**, eller sett dem så langt ut som
  mulig.
- **Slå av automatisk avspilling av forhåndsvisninger** på startskjermen.
- **Avinstaller apper vi ikke bruker.** Hver app er en potensiell
  oppdateringsdialog i veien.
- **Sett strømsparing slik at nettverket holdes i live i dvale** — vi må kunne
  vekke boksen over nettet.
- **Bekreft at HDMI-CEC er på** (§6).
- **Logg inn i alle tjenestene**, og noter hvor tokenene kan tilbakekalles.
- **Par fjernkontrollen og legg den i en skuff.** Den trengs ved vedlikehold —
  men brukeren skal aldri trenge den.

### Konsekvens for Pi-en

Nå som Pi-en verken har HDMI, nettleser, avspilling eller DRM å forholde seg
til, er en Pi 4 eller 5 kraftig overdimensjonert. En Pi 3 eller Pi Zero 2 W
holder fint — merk bare at Zero 2 W har få USB-porter, og at både
mikrokontrolleren og lydkortet skal ha plass. Har du allerede en Pi liggende,
bruk den; ikke kjøp en Pi 5 til denne jobben.

Det logiske neste spørsmålet er om Pi-en trengs i det hele tatt — en ESP32 har
både nettverk og nok kraft til å sende kommandoer og spille av lydfiler.
Teknisk er svaret ja. Jeg vil likevel beholde Pi-en, og grunnen er §10 i
hovednotatet: fjernvedlikehold, logging og oppdatering. På en boks som skal stå
hos noen i årevis er det verdt mer enn de kronene og wattene en ESP32 sparer.

---

## 3. De fire styringskanalene

| | Krever oppsett | Taster | Start app / deep-link | Lese tilstand | Responstid | Robusthet |
|---|---|---|---|---|---|---|
| **ADB over TCP** | Utviklermodus + paring. **Krever Wi-Fi, ikke ethernet. Overlever ikke omstart** | ✅ | ✅ | ✅ Klart best | ~100–300 ms per kommando | **Dårlig i drift** — se under |
| **Android TV Remote v2** | Paring med kode på skjerm | ✅ | ✅ (URI) | ⚠️ Begrenset | Lav — vedvarende forbindelse | God — det er protokollen Googles egen fjernkontroll-app bruker |
| **Google Cast** | Ingenting | ❌ | ⚠️ Cast-app-ID, ikke vilkårlig deep link | ✅ Avspilling + volum | Lav | God — stabilt, offentlig API |
| **Bluetooth HID** | Paring | ✅ | ❌ | ❌ | Svært lav | God, men helt blind |

### ADB: svakere enn jeg først antok

Jeg skrev tidligere at ADB-nøkkelen lagres og overlever omstart. Det stemmer
ikke på Android 13/14, som er det Google TV Streamer kjører, og forskjellen er
stor nok til at den snur anbefalingen.

Slik ser det faktisk ut:

- **Trådløs feilsøking er skilt ut fra USB-feilsøking** fra Android 13, med en
  egen paringsflyt: en sekssifret kode på skjermen og en egen paringsport.
- **Porten randomiseres, og oppsettet overlever ikke omstart.** Android 14
  strammet inn her. Boksen oppdaterer og starter seg selv om natten — så dette
  er ikke en teoretisk ulempe, det er noe som vil skje jevnlig.
- **Trådløs feilsøking lar seg ikke slå på over ethernet.** Bryteren hopper
  tilbake til av. Den dokumenterte omveien er å koble fra nettverkskabelen,
  la boksen komme opp på Wi-Fi, og pare derfra.

Det siste punktet kolliderer direkte med ethernet-anbefalingen i §2. Den
kollisjonen er reell, og den må løses ved å velge — ikke ved å håpe.

Det finnes tredjeparts-apper som slår på trådløs ADB igjen ved hver oppstart
uten root. Jeg vil ikke bygge driften på en slik app: det er en ekstra
avhengighet, uten garanti for at den overlever neste Android-versjon, på en
enhet ingen har tilsyn med.

### Android TV Remote v2: den som faktisk egner seg til drift

Protokollen bak Googles egen fjernkontroll-app (port 6466/6467), tilgjengelig
fra Python via `androidtvremote2` — det samme biblioteket Home Assistants
`androidtv_remote`-integrasjon bruker, altså noe som er i bruk hos mange og
blir vedlikeholdt.

Egenskapene som betyr noe her, punkt for punkt mot ADBs svakheter:

- **Krever ikke utviklermodus i det hele tatt.** Den snakker med Android TV
  Remote Service, som er forhåndsinstallert.
- **Paringen består.** Sertifikatene utveksles én gang; klienten kobler til
  igjen av seg selv etter omstart, og faller tilbake til paringsflyten hvis
  boksen skulle trekke tilbake tilliten.
- **Fungerer over ethernet.**
- **Vedvarende forbindelse**, altså lav responstid — det §7 trenger.
- **Kan sende deep links**, ikke bare tastetrykk. Det er den ene evnen hele
  modellen står og faller på (§4), og den er i behold.

**Cast** via `pychromecast` gir volum og avspillingsstatus stabilt, men kan i
praksis ikke starte vilkårlig innhold i en DRM-tjeneste, siden autentiseringen
ligger hos sender-appen.

### Anbefaling

Dette er ikke lenger en åpen avveiing, slik jeg framstilte den tidligere:

**Remote v2 er driftsveien. ADB er et oppsett- og feilsøkingsverktøy.**

ADB er fortsatt uunnværlig — men til å *utforske* boksen, ikke til å styre den.
`logcat` og `dumpsys` er den eneste måten å finne ut hvilke deep links som
finnes (§4), og den jobben gjøres én gang, på Wi-Fi, i verkstedfasen. Deretter
kobles boksen på kabel og driftes over Remote v2, og utviklermodus kan slås av.

At utviklermodus da *ikke* står på permanent er en tilleggsgevinst: ADB åpent på
nettverket er en kjent angrepsflate på Android TV-enheter, og en boks hjemme hos
en person som ikke kan overvåke den er ikke stedet å la den stå åpen.

Skal du feilsøke senere: bytt boksen midlertidig til Wi-Fi, par ADB på nytt,
gjør det som skal gjøres, sett den tilbake på kabel. Tungvint, men det er en
sjelden operasjon — og prisen for at den vanlige driften er robust.

### Hva med Homey?

Athom har en **offisiell Android TV-app for Homey** (`com.android.tv`, med
kildekode på GitHub). Den støtter både tastetrykk og å starte apper via deep
link, altså det vi trenger. To forbehold: den **krever Homey Pro**, siden den
må ha lokal forbindelse til boksen — en Homey Bridge holder ikke — og Athom
skriver selv at enkelte funksjoner kan være begrenset eller ikke virke, på
grunn av begrensninger i selve Android TV-protokollen.

**Bør vi styre boksen gjennom Homey? Nei — ikke styringsveien.**

Homey snakker den samme Remote v2-protokollen som vi ville snakket direkte.
Å gå via Homey legger derfor ikke til noen evne; det legger til et ledd:

- **Ett ekstra ledd i responstiden.** Knapp → Pi → Homey → boks, i stedet for
  knapp → Pi → boks. For volumknappen, der målet er under 100 ms (§7), er det
  et dårlig bytte.
- **Ett ekstra feilpunkt.** Er Homey nede eller midt i en oppdatering, virker
  ikke TV-en. Vi har allerede akkurat nok ting som kan ryke.
- **Mindre kontroll over feilhåndtering.** Vi trenger å vite *hvorfor* noe
  feilet for å kunne si det riktige til brukeren (§5). Gjennom en
  Flow-abstraksjon blir det tynnere.

Vi bruker `androidtvremote2` direkte fra Python i stedet. Det er også
vedlikeholdt, og det er én avhengighet i stedet for en enhet til i kjeden.

**Men Homey har en god rolle — bare en annen enn styring.**

Har du en Homey Pro fra før, er den et nesten ferdig svar på varslingskravet
fra §2 i hovednotatet: at pårørende skal få beskjed *før* brukeren treffer et
problem. Pi-en kaller en webhook, Homey sender pushvarselet. Fordelen er at
denne veien er **ufarlig å la være avhengig av Homey**: ryker den, mister vi et
varsel, men brukeren merker ingenting. Det er motsatt av styringsveien, der et
ekstra ledd rammer henne direkte.

En bonus på kjøpet: Homey-appen på telefonen blir en fjernkontroll de pårørende
kan bruke når de hjelper til over telefon, uten at det gir brukeren noe nytt å
forholde seg til.

**Uansett verdt å låne fra Homey-miljøet:** både Homeys dokumentasjon og Home
Assistant-miljøet peker til en felles, dugnadsbasert oversikt over deep links
for Android TV-apper. Den er verdt å slå opp i før spiken (§4) — kanskje noen
allerede har funnet adressene vi trenger. Merk også teknikken
`market://launch?id=<pakkenavn>`, som starter en app på pakkenavn alene, og som
er et brukbart minimum for tjenester uten en ordentlig innholds-deep-link.

---

## 4. Deep links er det som avgjør prosjektet

Alt annet i dette notatet er enkelt. Dette er ikke.

Løftet til brukeren er «ett trykk = NRK1». Det løftet holder bare hvis vi kan
uttrykke «NRK1» som **én adresse vi kan sende**, i stedet for en sekvens av
piltastetrykk gjennom en meny vi ikke kan se. Forskjellen er ikke
programmeringsestetikk:

- En deep link er en slags kontrakt. Den overlever at appen redesignes.
- En tastesekvens er en antakelse om hvor markøren står og hvordan menyen ser
  ut i dag. Den brekker ved neste appoppdatering — stille, hjemme hos brukeren,
  uten at noen får vite det før hun trykker på knappen.

### Slik finner vi ut om de finnes

Tre teknikker, i økende nytte:

**1. Finn pakkenavnene**

```bash
adb shell pm list packages | grep -iE 'nrk|tv2|tv 2'
```

**2. Se hvilke adresser appen har meldt seg på**

```bash
adb shell dumpsys package <pakkenavn> | grep -B2 -A10 'android.intent.action.VIEW'
```

Dette lister intent-filtrene appen registrerer — skjemaer og verter den sier
den kan håndtere. Ser du appens egne nettadresser der (`tv.nrk.no` og
liknende), er sjansen god for at innholdsadresser fungerer direkte.

**3. Den som faktisk gir svaret: se hva appen selv gjør**

```bash
adb logcat -c
# naviger nå manuelt til kanalen med fjernkontrollen
adb logcat | grep -i 'START u0'
```

Her ser du det eksakte intentet appen starter når *den selv* går til den
kanalen. Det er den mest pålitelige måten å finne den riktige adressen på,
fordi det er appens egen interne kontrakt du leser av — ikke noe du gjetter deg
til utenfra.

Test så at den virker fra kald start:

```bash
adb shell am start -a android.intent.action.VIEW -d "<adressen>"
```

### Hvis en tjeneste ikke har brukbar deep link

Da står valget mellom tre dårlige alternativer, og det er verdt å ta stilling
til det bevisst i stedet for å skli inn i det første:

1. **Tastesekvens med verifisering.** Ikke blind — vi leser tilstanden etterpå
   (§5) og sier ifra hvis vi havnet feil. Bedre enn en ren makro, men fortsatt
   noe som vil brekke.
2. **Bare start appen**, og la den lande der den lander — typisk «fortsett å
   se» eller forsiden. Ærligere: knappen betyr da «TV 2», ikke «TV 2 direkte».
3. **Ta tjenesten ut av v1.** Fullt legitimt. Fire knapper som alltid virker er
   et bedre produkt enn seks der to av og til gjør noe rart.

Jeg vil advare mot alternativ 1 som *standardvalg*. Det er den slags løsning
som virker perfekt den dagen den bygges og som ingen oppdager er ødelagt før
noen ringer og sier at boksen har «begynt å gjøre noe rart».

---

## 5. Tilstandslesing — undervurdert, og nødvendig

Vi kan spørre boksen hva som foregår:

```bash
adb shell dumpsys media_session      # hva spiller, i hvilken app, spiller/pauset
adb shell dumpsys activity activities | grep mResumedActivity   # hva er i forgrunnen
adb shell dumpsys power | grep 'mWakefulness'                   # våken eller ikke
adb exec-out screencap -p > skjerm.png                          # for fjernfeilsøking
```

Grunnen til at dette betyr mer her enn i et vanlig prosjekt: **lydmodellen vår
(§6 i hovednotatet) lover brukeren at boksen sier hva som skjer.** Uten
tilstandslesing er «NRK1» bare noe vi *håper* er sant fordi vi sendte en
kommando. Med tilstandslesing er det noe vi vet.

For en bruker som ikke kan se skjermen er den forskjellen hele forskjellen. Hvis
boksen sier «NRK1» og det er TV 2 som spiller, har vi ikke gitt henne feil
informasjon én gang — vi har lært henne at boksen ikke er til å stole på.

Så: **si aldri kanalnavnet før tilstanden er bekreftet.** Sekvensen blir klikk
(«hørte deg») → lastetone → bekreftet kanalnavn, eller feilmelding. Ikke
kanalnavn med én gang og håp.

`screencap` er også verdt å merke seg for §10 i hovednotatet: en pårørende kan
se hva som faktisk står på skjermen, uten å være i huset.

### Men: alt dette krever ADB, som vi nettopp flyttet ut av driften

Her er en konsekvens jeg ikke så da jeg skrev anbefalingen over. `dumpsys` er
et ADB-verktøy. Er driftsveien Remote v2 (§3), har vi det ikke tilgjengelig til
daglig.

Remote v2 gir oss noe — blant annet hvilken app som er i forgrunnen, og
volumnivå — men såvidt jeg forstår ikke *hva som spilles inne i appen*. Det er
en svakere form for verifisering enn `dumpsys media_session`, og forskjellen
treffer nøyaktig det løftet lydmodellen gir brukeren.

Konkret: vi kan trolig bekrefte «vi er i NRK-appen», men ikke nødvendigvis «det
er NRK1 som spiller». Tre måter å leve med det på:

1. **Si bare det vi vet.** Les opp appnavnet der vi bare kan bekrefte appen —
   «NRK» — og kanalnavnet der vi faktisk kan bekrefte kanalen. Ærlig, men gir
   ujevn oppførsel mellom knappene, og det er i seg selv uheldig for en bruker
   som bygger muskelminne.
2. **Behold ADB i drift likevel**, og godta Wi-Fi framfor kabel. Det bytter én
   svakhet mot en annen, og jeg tror nettverksstabilitet veier tyngst.
3. **Verifiser at deep-linken traff riktig app, og stol på deep-linken for
   resten.** En adresse som peker på NRK1, og som vi har bekreftet startet
   NRK-appen, har ikke mange måter å ende opp på feil kanal på.

Jeg heller mot **3, med 1 som sikkerhetsnett** ved feil. Men det bør avgjøres
på målte data, ikke på antakelser om hva protokollen gir — derfor er det lagt
inn som eget punkt i spiken (§10).

---

## 6. Strøm og oppstart

Her er en hyggelig overraskelse: **Google TV-boksen gjør mesteparten av
CEC-jobben selv.** Når den vekkes, slår den typisk på TV-en og bytter til sin
egen HDMI-inngang. Det betyr at «på»-knappen kan bli:

```
vekk boksen  →  TV-en følger etter av seg selv
```

CEC-risikoen fra §7 i hovednotatet blir dermed mindre, fordi vi flytter
CEC-ansvaret fra vår egen implementasjon over på en enhet der det er
produsenttestet. Men merk at det bare gjelder når en delegert kilde er aktiv —
og at det fortsatt må verifiseres på den faktiske TV-en. Det er den samme
CEC-testen, bare med et mer sannsynlig utfall.

Av-veien er verdt en beslutning: skal «av» dvale-sette boksen, eller også slå av
TV-en? Jeg foreslår **begge deler** — for brukeren betyr «av» at det blir stille
og mørkt, ikke at én av to bokser går i dvale.

---

## 7. Volum og responstid

Dette er stedet hvor en naiv implementasjon kommer til å føles dårlig.

Rotasjonsenkoderen produserer mange hendelser raskt. Hvis hvert hakk blir en
`adb shell`-kommando, betyr det en ny prosess og en ny rundtur på 100–300 ms
per hakk. Brukeren vrir, og lyden henger etter og kommer haltende i etterkant.
For en som styrer på hørsel alene er det ikke en skjønnhetsfeil — det er tap av
kontroll.

To ting løser det, og begge bør inn fra starten:

1. **Vedvarende forbindelse.** Remote v2 eller Cast holder forbindelsen åpen.
   Ingen prosessoppstart per hakk.
2. **Slå sammen hakk.** Mikrokontrolleren sender allerede `ENC:+3` og ikke tre
   separate meldinger (§5 i hovednotatet). Det gir oss én kommando i stedet
   for tre.

Sett et konkret mål og mål mot det: **under 100 ms fra vri til hørbar endring.**
Er vi over, er valget av styringskanal feil, ikke koden.

---

## 8. Feilmodus som er særegne for denne modellen

Ting som vil skje, og som må håndteres i stedet for oppdages:

- **Boksen oppdaterer seg selv.** Apper også. Deep links overlever stort sett
  dette; tastesekvenser gjør det ikke.
- **Mellomskjermer.** «Fortsett å se», vilkårsendringer, kampanjeskjermer,
  «logg inn på nytt». De dukker opp uten forvarsel og bryter enhver antakelse
  om hvor vi er. Motgiften er tilstandslesing (§5) pluss en fast
  gjenopprettingsrutine: `KEYCODE_HOME`, vent, send deep link på nytt.
- **Skjermsparer og dvale.** Send alltid `KEYCODE_WAKEUP` først og verifiser
  `mWakefulness=Awake` før noe annet.
- **ADB blir slått av.** Ved fabrikkreset eller enkelte oppdateringer. Hvis
  driftsveien er Remote v2, er dette en diagnostikkulempe og ikke et utfall.
- **Boksen bytter IP.** Fast DHCP-reservasjon på ruteren, eller oppslag på
  mDNS. Ikke hardkod IP-adressen.
- **Nettverket faller.** Da er *alt* borte, siden hele innholdslaget nå ligger
  bak en nettverksforbindelse. Boksen må si det med ord — «nettet er nede» — og
  ikke bare bli stille.

---

## 9. Hva valget gjør med resten av arkitekturen

Dette er det jeg tror er den viktigste konsekvensen, og den er god:

**Hvis alle kanalene delegeres, trenger ikke Raspberry Pi-en lenger være en
HDMI-kilde i det hele tatt.**

Da faller følgende bort fra hovednotatet (numrene under viser til det, ikke
til dette notatet):

- Kiosk-modus og oppstart rett i en app (§13, punkt om kiosk)
- mpv kontra Kodi (§4) — det er ikke lenger noe å velge mellom
- HDMI-CEC fra Pi-en (§7) — boksen gjør det
- Widevine-spiken (§2) — spørsmålet forsvinner, det er boksens problem nå
- Hele token-håndteringen (§2) — appene eier den, og fornyer stille selv

Pi-en blir en **hodeløs nettverkskontroller**: knapper inn, lyd ut på egen
høyttaler, kommandoer ut på nettet. Ingen skjerm, ingen X, ingen nettleser,
ingen DRM, ingen tokens. Det er dramatisk mindre å bygge og vedlikeholde enn
det hovednotatet la opp til.

Og da er det verdt å stille spørsmålet: **skal vi da beholde en egen
spillervei for NRK i det hele tatt?**

Argumentet for å droppe den er ensartethet — én kodesti, én feilmodus, én ting
som kan gå i stykker, og lik oppførsel og responstid på alle fire knapper. For
en bruker som bygger opp muskelminne er det siste ikke en detalj.

Argumentet for å beholde den er uavhengighet: hvis Google TV-boksen får en
dårlig oppdatering, er det fint at minst én kanal fortsatt virker.

**Jeg anbefaler å droppe den.** Halvparten så mange feilmodus er mer verdt her
enn en reservevei som uansett bare dekker én av fire knapper — og som selv vil
råtne stille i bakgrunnen fordi den nesten aldri brukes. Men dette er en reell
avveiing, og motargumentet er ikke dumt.

---

## 10. Spike for uke 1

Denne erstatter Widevine-spiken fra §2 i hovednotatet. Én dag, og den avgjør
om modellen holder.

0. Kjør oppsettsjekklisten i §2 først. Flere av punktene der påvirker hva
   spiken måler.
1. Slå på utviklermodus og par ADB **over Wi-Fi** (ikke ethernet — §3).
   Forvent at paringen må gjøres på nytt etter hver omstart; det er normalt på
   Android 14, og ikke et tegn på at noe er galt.
2. Kartlegg pakkenavn for hver tjeneste (§4, teknikk 1).
3. **For hver av de fire kanalene: finn og verifiser en deep link** (§4,
   teknikk 3). Dette er dagens viktigste punkt. Noter hvilke som lyktes.
4. Mål tid fra kommando til bilde og lyd, per kanal, fra kald start.
5. **Par Remote v2, og kartlegg hvor mye tilstand den faktisk gir** (§5): ser
   vi hvilken app som kjører, og ser vi noe om innholdet i den? Bekreft
   samtidig at Remote v2 kan sende de samme deep-linkene som ADB gjorde i
   punkt 3 — det er den kritiske evnen.
6. Bekreft at Remote v2-paringen overlever omstart av både boks og Pi.
7. Test at vekking av boksen slår på TV-en og velger riktig inngang (§6).
8. Mål volumresponstid gjennom minst to kanaler (§7).

**Beslutningsregelen:** fungerer punkt 3 og 5 for alle fire kanalene — deep
link funnet, *og* sendbar over Remote v2 — er modellen god og resten er
alminnelig arbeid. Fungerer den for to av fire, må vi snakke
om hvilke kanaler knappene faktisk skal være — det er en bedre samtale å ta nå
enn etter at boksen er bygget.

---

## 11. Åpne spørsmål

1. ~~Hvilken Google TV-enhet?~~ **Avklart:** separat boks, ikke innebygd i
   TV-en. Se §2 — anbefalingen er Google TV Streamer framfor en utgått
   Chromecast, i hovedsak på grunn av lagring og ethernet.
2. **Beholder vi en egen NRK-vei på Pi-en?** Se §9 — jeg anbefaler nei.
3. ~~Utviklermodus permanent på, eller Remote v2 i drift?~~ **Avklart:**
   Remote v2 i drift, ADB kun til oppsett og feilsøking (§3).
4. **Hvor mye tilstand gir Remote v2?** Spørsmålet som erstatter det forrige,
   se §5. Det avgjør hva boksen kan si til brukeren uten å risikere å ta feil.
5. **Har du Homey Pro, eller Bridge?** Android TV-appen krever Pro. Svaret
   avgjør ikke styringsveien (se §3 — den går utenom Homey uansett), men det
   avgjør om Homey kan brukes til varsling av pårørende.
6. **Hva skjer når nettet er nede?** Nå som alt innhold er nettavhengig, er
   dette en tilstand som fortjener en egen talemelding og ikke bare en
   generisk feil.
