# TV-styringsboks for synshemmede/eldre

> Se [`docs/arkitektur-diskusjonsgrunnlag.md`](docs/arkitektur-diskusjonsgrunnlag.md)
> for et diskusjonsgrunnlag om arkitektur og løsningsmodell, og
> [`docs/google-tv-styring.md`](docs/google-tv-styring.md) for styring av
> Google TV-enheten.

## Prosjektmål

Bygge en fysisk boks med store, taktile knapper som lar en synshemmet eller
eldre person styre TV-titting uten å måtte navigere i en smart-TV-ens eget
grensesnitt. Boksen kobles til TV-en via én fast HDMI-inngang og håndterer
alt innhold internt (kanaler/strømmetjenester), i stedet for å fjernstyre
TV-ens egen smart-TV-app-lag.

**Designprinsipp:** Brukeren skal aldri se TV-ens hjem-skjerm, apper eller
menyer. Boksen er alltid aktiv kilde, og styrer TV-en kun for av/på og
kildevalg via HDMI-CEC.

## Målgruppe og tilgjengelighetskrav

- Bruker kan ha nedsatt syn og/eller begrenset digital erfaring
- Store, tydelig atskilte fysiske knapper (ingen berøringsskjerm som eneste
  input)
- Konsekvent, uforanderlig knappe-layout — samme knapp gjør alltid samme
  ting
- Lydtilbakemelding (kort pip/stemme) ved trykk og ved kanalbytte
- Ingen skjulte menyer, ingen flertrinns navigering for kjernefunksjoner
  (på/av, volum, kanal skal være ett trykk)

## Funksjonelle knapper (førsteversjon)

| Knapp | Funksjon |
|---|---|
| Av/på | Slår TV på/av via HDMI-CEC, starter/stopper avspilling |
| Volum + / − | Rotasjonsenkoder, styrer volum via CEC (TV-høyttalere) |
| Kanal 1–4 (eller flere) | Bytter direkte til forhåndskonfigurert kanal/strøm |

## Maskinvare

- Raspberry Pi 4 (2GB) eller Pi 5
- 32GB+ microSD, A2-klasse
- Egen mikrokontroller (Pi Pico/ESP32) for knapper, koblet som USB-tastatur
  til RPi — RPi skal ikke pollet GPIO i sanntid
- Arkade-trykknapper (24–30mm) for kanalvalg og av/på
- Rotasjonsenkoder for volum
- 5V/3A USB-C strømforsyning
- HDMI-CEC via RPi sin innebygde HDMI-port (`cec-utils`)

## Programvarearkitektur

1. **Inputlag** — lytter på USB-tastatur-events (fra mikrokontroller) eller
   GPIO, oversetter til logiske kommandoer (PÅ_AV, VOL_OPP, VOL_NED,
   KANAL_1 … KANAL_N)
2. **Innholdslag** — én kjørende applikasjon (foretrukket: Kodi med
   IPTV-tillegg, alternativt egne skript per strømmetjeneste) som bytter
   internt mellom forhåndskonfigurerte kanaler/kilder ved kommando
3. **TV-styringslag** — sender CEC-kommandoer for av/på og HDMI-kildevalg
   (`cec-client`)
4. **Oppstart** — RPi booter rett inn i kiosk-modus/innholdsappen. Ingen
   skrivebord, ingen innlogging, ingen synlig Linux-lag for sluttbrukeren

## Ikke-mål (for denne versjonen)

- Ikke styre TV-ens egne smart-TV-apper direkte (for merkeavhengig og
  skjørt, se avveining under)
- Ikke stemmestyring i første versjon
- Ikke nettverksbasert fjernstyring/app for pårørende (kan vurderes senere)

## Kjente risikoer / ting som må testes tidlig

- **CEC-støtte varierer mellom TV-merker** — test `cec-client -l` og
  faktisk av/på + kildebytte på den konkrete TV-modellen før videre
  utvikling
- Enkelte TV-er går i dyp standby og mister CEC-respons — vurder
  IR-fallback for av/på hvis CEC viser seg upålitelig
- Strømmetjenester (NRK TV, TV 2 Play m.fl.) har varierende
  Linux/nettleser-støtte — avklar hvilke kanaler/tjenester som faktisk skal
  støttes før valg av innholdslag (Kodi vs. enkeltapper)

## Foreslått teknisk stack

- Python for input- og CEC-styringslag
- Kodi (med IPTV-tillegg) eller enkel egenutviklet video-player for
  innholdslag
- systemd-tjeneste for autostart i kiosk-modus

## Første utviklingsfase (forslag til Claude Code)

1. Sett opp RPi i kiosk-modus som booter rett til en enkel testapplikasjon
2. Implementer CEC av/på og kildebytte, test mot faktisk TV-modell
3. Implementer inputlag: les knappetrykk fra mikrokontroller (USB-HID) og
   koble til logiske kommandoer
4. Koble inputlag til innholdslag: kanal-knapper bytter faktisk
   video-kilde/kanal
5. Legg til lyd-/talefeedback ved knappetrykk
6. Brukertest med reell målgruppe, juster knappelayout og responstid
   basert på tilbakemelding
