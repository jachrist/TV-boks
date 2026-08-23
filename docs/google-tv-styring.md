# Programmatisk styring av Google TV

Referansenotat til modell C i
[arkitektur-diskusjonsgrunnlaget](arkitektur-diskusjonsgrunnlag.md), etter at
valget falt på en Google TV-enhet som innholdsleverandør.

**Merk om sikkerhet i påstandene:** styringskanalene under er godt etablerte og
brukes blant annet av Home Assistant. Men *hvilke pakkenavn og deep links som
faktisk finnes for NRK og TV 2*, og hvordan den konkrete enheten oppfører seg,
er noe som må måles på maskinvaren — ikke noe jeg vil påstå på forhånd. §3 og
§9 er derfor skrevet som oppskrifter for å finne ut av det, ikke som fasit.

---

## 1. Kort svar

| Vi vil kunne | Går det? |
|---|---|
| Sende tastetrykk (D-pad, play/pause, volum, hjem, tilbake) | ✅ Trivielt, flere veier |
| Vekke og dvale-sette boksen | ✅ |
| Starte en bestemt app | ✅ |
| **Gå rett til en bestemt kanal i appen** | ⚠️ **Avhenger av deep links — dette er hele spørsmålet, se §3** |
| Lese hva som faktisk spiller nå | ✅ Og det er viktigere enn det høres ut, se §4 |
| Styre volum | ✅ Flere veier, men pass på responstid, se §6 |
| Slå på TV og bytte inngang | ✅ Ofte gjør boksen det selv via CEC, se §5 |

Det som *ikke* går, og som det er verdt å slå fast med en gang: vi kan ikke
skript-styre innsiden av en app på noen robust måte. Alt vi gjør må uttrykkes
som «start denne adressen» eller «trykk denne tasten» — og av de to er bare den
første pålitelig over tid.

---

## 2. De fire styringskanalene

| | Krever oppsett | Taster | Start app / deep-link | Lese tilstand | Responstid | Robusthet |
|---|---|---|---|---|---|---|
| **ADB over TCP** | Utviklermodus + engangsgodkjenning på skjerm | ✅ | ✅ | ✅ Klart best | ~100–300 ms per kommando | Middels — kan slås av ved oppdatering/reset |
| **Android TV Remote v2** | Paring med kode på skjerm | ✅ | ✅ (URI) | ⚠️ Begrenset | Lav — vedvarende forbindelse | God — det er protokollen Googles egen fjernkontroll-app bruker |
| **Google Cast** | Ingenting | ❌ | ⚠️ Cast-app-ID, ikke vilkårlig deep link | ✅ Avspilling + volum | Lav | God — stabilt, offentlig API |
| **Bluetooth HID** | Paring | ✅ | ❌ | ❌ | Svært lav | God, men helt blind |

**ADB** er `adb connect <ip>:5555` etter at utviklermodus er slått på (trykk
sju ganger på byggnummeret) og nettverksfeilsøking er aktivert. Første tilkobling
gir en godkjenningsdialog *på skjermen* — en engangsjobb ved oppsett, men den
krever at noen ser skjermen. Nøkkelen lagres og overlever omstart.

**Android TV Remote v2** er den reverse-utviklede protokollen bak Googles
fjernkontroll-app (port 6466/6467), tilgjengelig fra Python via
`androidtvremote2`. Paringen skjer med en sekssifret kode på skjermen, én gang.
Fordelen over ADB er at den **ikke krever utviklermodus** og holder en
vedvarende forbindelse — begge deler betyr noe her.

**Cast** via `pychromecast` gir volum og avspillingsstatus stabilt, men kan i
praksis ikke starte vilkårlig innhold i en DRM-tjeneste, siden autentiseringen
ligger hos sender-appen.

### Anbefaling

**Bruk ADB under utvikling, vurder å flytte driftsveien til Remote v2.**

ADB er det klart beste verktøyet for å *utforske* enheten — `dumpsys` og
`logcat` er hvordan vi i det hele tatt finner ut hvilke deep links som finnes
(§3). Men å la utviklermodus stå på i årevis på en boks hjemme hos noen er både
en liten angrepsflate og et skjørt punkt: den kan bli slått av av en oppdatering
eller et fabrikkreset, og da må noen fysisk inn og slå den på igjen.

Sluttbildet jeg vil foreslå: **Remote v2 som styringsvei, Cast for volum og
status, ADB tilgjengelig for diagnostikk** når noe skal feilsøkes. Men det er en
beslutning å ta etter at vi har målt — ikke nå.

---

## 3. Deep links er det som avgjør prosjektet

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
   (§4) og sier ifra hvis vi havnet feil. Bedre enn en ren makro, men fortsatt
   noe som vil brekke.
2. **Bare start appen**, og la den lande der den lander — typisk «fortsett å
   se» eller forsiden. Ærligere: knappen betyr da «TV 2», ikke «TV 2 direkte».
3. **Ta tjenesten ut av v1.** Fullt legitimt. Fire knapper som alltid virker er
   et bedre produkt enn seks der to av og til gjør noe rart.

Jeg vil advare mot alternativ 1 som *standardvalg*. Det er den slags løsning
som virker perfekt den dagen den bygges og som ingen oppdager er ødelagt før
noen ringer og sier at boksen har «begynt å gjøre noe rart».

---

## 4. Tilstandslesing — undervurdert, og nødvendig

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

---

## 5. Strøm og oppstart

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

## 6. Volum og responstid

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

## 7. Feilmodus som er særegne for denne modellen

Ting som vil skje, og som må håndteres i stedet for oppdages:

- **Boksen oppdaterer seg selv.** Apper også. Deep links overlever stort sett
  dette; tastesekvenser gjør det ikke.
- **Mellomskjermer.** «Fortsett å se», vilkårsendringer, kampanjeskjermer,
  «logg inn på nytt». De dukker opp uten forvarsel og bryter enhver antakelse
  om hvor vi er. Motgiften er tilstandslesing (§4) pluss en fast
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

## 8. Hva valget gjør med resten av arkitekturen

Dette er det jeg tror er den viktigste konsekvensen, og den er god:

**Hvis alle kanalene delegeres, trenger ikke Raspberry Pi-en lenger være en
HDMI-kilde i det hele tatt.**

Da faller følgende bort fra hovednotatet:

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

## 9. Spike for uke 1

Denne erstatter Widevine-spiken fra §2 i hovednotatet. Én dag, og den avgjør
om modellen holder.

1. Slå på utviklermodus, `adb connect`, bekreft at forbindelsen overlever en
   omstart av boksen.
2. Kartlegg pakkenavn for hver tjeneste (§3, teknikk 1).
3. **For hver av de fire kanalene: finn og verifiser en deep link** (§3,
   teknikk 3). Dette er dagens viktigste punkt. Noter hvilke som lyktes.
4. Mål tid fra kommando til bilde og lyd, per kanal, fra kald start.
5. Bekreft at tilstandslesing skiller kanalene fra hverandre (§4) — altså at vi
   *kan* verifisere før vi uttaler oss.
6. Test at vekking av boksen slår på TV-en og velger riktig inngang (§5).
7. Mål volumresponstid gjennom minst to kanaler (§6).

**Beslutningsregelen:** fungerer punkt 3 for alle fire kanalene, er modellen
god og resten er alminnelig arbeid. Fungerer den for to av fire, må vi snakke
om hvilke kanaler knappene faktisk skal være — det er en bedre samtale å ta nå
enn etter at boksen er bygget.

---

## 10. Åpne spørsmål

1. **Hvilken Google TV-enhet?** Chromecast med Google TV er billig og gjør
   jobben. Nvidia Shield har best ADB-støtte og mer kraft. Google TV *innebygd
   i TV-en* er en annen sak — da finnes det ingen separat HDMI-inngang å bytte
   til, og §5 må tenkes om.
2. **Beholder vi en egen NRK-vei på Pi-en?** Se §8 — jeg anbefaler nei.
3. **Utviklermodus permanent på, eller Remote v2 i drift?** Kan besvares etter
   spiken, men det er verdt å vite at spørsmålet finnes.
4. **Hva skjer når nettet er nede?** Nå som alt innhold er nettavhengig, er
   dette en tilstand som fortjener en egen talemelding og ikke bare en
   generisk feil.
