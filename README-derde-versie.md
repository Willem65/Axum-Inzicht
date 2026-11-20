# AXUM Inzicht

Een visuele webpagina die laat zien hoe het **D&R AXUM**-mengpaneel is ingericht en wat het op dit moment doet. AXUM Inzicht vervangt de onoverzichtelijke oude webpagina (Perl/Yawf) op de engine. Het programma **leest alleen**: het verandert niets op de mixer.

- Engine: `192.168.1.222` (Postgres 8.3, axum-engine, axum-gateway)
- Pagina: **http://localhost:8082**

---

## Inhoud

1. [Starten](#starten)
2. [Tabbladen](#tabbladen)
3. [Het Surface-tabblad](#het-surface-tabblad)
4. [Waar de gegevens vandaan komen](#waar-de-gegevens-vandaan-komen)
5. [SSH naar de engine](#ssh-naar-de-engine)
6. [Windows-versie](#windows-versie)
7. [Bevindingen over de AXUM](#bevindingen-over-de-axum)
8. [Bestanden](#bestanden)
9. [Onderhoud en backups](#onderhoud-en-backups)

---

## Starten

### Linux

```bash
cd ~/claude-axumOpnieuw3
./start.sh
```

Open daarna http://localhost:8082. Stoppen doe je met Ctrl+C.

Je kunt ook een andere poort of engine opgeven:

```bash
AXUM_PORT=8090 ./start.sh
AXUM_ENGINE=192.168.1.50 ./start.sh
```

### Windows

Zie [Windows-versie](#windows-versie).

---

## Tabbladen

| Tabblad | Wat je ziet |
|---|---|
| **Rack** | De kaarten in het rack per slot, met kanaal-LED's en problemen |
| **Signaalpad** | Kaartkanaal → bron → module → bus → bestemming → kaartkanaal |
| **Surface** | Een virtuele surface met faders en knoppen, plus het signaalpad dat daardoor actief wordt (zie hieronder) |
| **Kaart-functies** | Welke knop, LED of GPO aan welke bron of module hangt, en welke import-offset klopt |
| **Modules** | De 32 modules in 2 rijen van 16: actuele bron, fader, bus aan/uit en **PPM-meters (Live)** |
| **Bussen** | Bussen en monitorbussen |
| **Problemen** | Inconsistenties die de oude webpagina verbergt |
| **Live log** | Configuratiewijzigingen uit `recent_changes` |
| **Systeem** | De vervanging van `/system`: MambaNet, templates, functies, logs, versies |

In de kopregel staan samplerate, headroom, klok en het aantal DSP-kaarten. Is `global_config` leeg, dan staat daar een **oranje waarschuwing** (zie [Bevindingen](#global_config-raakt-soms-leeg)).

---

## Het Surface-tabblad

Bovenaan kies je de surface. De belangrijkste is **Axite-UI-6FBP-CRM** (unique ID `0001:0022:0505`). Dat is een software-surface op een pc (192.168.1.2), die nu modules 7–12 bedient.

### Startstand: de echte stand van de engine

Bij het openen leest de pagina via SSH het bestand **`/var/lib/axum/.backup`** van de engine. Daaruit komen:

- **fader-niveaus** in dB;
- de **actuele bron** per module;
- per module welke **bussen aan** staan (Prog, Sub, Aux, PFL …);
- **Start** aan/uit per bron;
- de **monitorkeuze** (waar CRM en Studio naar luisteren);
- de **Phones- en Speaker-niveaus** (draaiknoppen).

Met **↺ stand van de engine** lees je de stand opnieuw in.

### Live

Met de knop **○ Live uit / ● Live aan** leest AXUM Inzicht live mee met het MambaNet-verkeer tussen de surface en de engine. De faders bewegen mee, de lampjes en displays volgen de engine en de meters lopen mee. Onderaan staan de **laatste 10 MambaNet-berichten**.

### Simulatie

Zonder Live kun je zelf op knoppen klikken en faders schuiven. Je ziet dan welk signaalpad actief wordt. **Dit verandert niets op de mixer.**

### Weergave

- **Faders zoals op de Axite.** Een zwarte gleuf met een bijna zwarte knop met wit streepje. De schaal is **+10 … 0 … -50 … -∞ dB** en volgt de officiële fader-curve van de engine (tabel `db_to_position`).
- **Draaiknoppen.** Phones en Speaker zijn ronde zwarte knopjes met een wit standstreepje. Je verdraait ze in de simulatie door te slepen, met het scrollwiel of met de pijltjestoetsen.
- **Signaalpad onderaan.** Een actief pad is dik en groen, met lopende "rupsen". Een niet-actief pad is dun en grijs gestippeld.

> **Rupsen lopen niet?** Dan staan in Windows de animaties uit, of je werkt via Extern bureaublad. Zet ze aan via
> Instellingen → Toegankelijkheid → Visuele effecten → **Animatie-effecten: Aan**.

### Meters

De engine stuurt meterwaardes **alleen voor modules die op een online surface staan**, nu dus modules 7–12. De meterbrug **Axite-PPM-Meters** (`0001:0024:005D`, adres 110) heeft meters voor alle 32 modules, maar staat **offline sinds 5 juli 2023**. Start je die software of dat apparaat weer, dan verschijnen alle 32 meters vanzelf in het tabblad Modules.

---

## Waar de gegevens vandaan komen

```
 Surface (192.168.1.2)                    Engine (192.168.1.222)
 ┌──────────────┐   UDP-pakket poort 34848  ┌─────────────────────────────┐
 │ Fader 1      │ ────────────────────────▶ │ netwerkkaart eth0           │
 │ obj 1084=529 │ ◀──────────────────────── │   ├─▶ axum-gateway → engine │
 └──────────────┘   (engine stuurt terug:   │   └─▶ Perl-script (kopie)   │
                    fader, lampjes, tekst)  └──────────────┬──────────────┘
                                                           │ SSH (tekstregels)
                                             Deze pc       ▼
                                            ┌──────────────────────────────┐
                                            │ AXUM Inzicht (mbnlive.py)    │
                                            │  ontcijfert → browser (SSE)  │
                                            └──────────────────────────────┘
```

| Wat | Bron |
|---|---|
| Configuratie: bronnen, bestemmingen, bussen, routing, welke knop welke functie heeft | **Postgres** (`axum`, poort 5432, alleen lezen) |
| De laatst bewaarde stand: faders, bussen, bron, Start, monitor, Phones/Speaker | **`/var/lib/axum/.backup`** via SSH |
| De actuele bediening, lampjes, displays, meters | **MambaNet over UDP 34848**, live meegelezen via SSH |

### Belangrijk: bediening staat niet in de database

Getest op 19-11-2026. Bij het bewegen van faders en indrukken van knoppen **verandert er niets in Postgres**: `module_config` krijgt 0 updates. De actuele stand staat alleen in het geheugen van `axum-engine`. Die schrijft hem elke **30 seconden** (als er iets veranderd is) naar `/var/lib/axum/.backup`, zodat hij na een herstart terug kan (`startup_state`).

### MambaNet meelezen

- **Hoe:** zolang **Live** aan staat, start AXUM Inzicht via SSH op de engine een klein Perl-script. Dat leest met een *raw socket* een **kopie** van de UDP-pakketten op poort 34848.
- **Niets op de engine:** er wordt niets geïnstalleerd of opgeslagen.
- **Stopt vanzelf:** het script stuurt elke 5 s een hartslag, en stopt zodra de verbinding wegvalt.
- **Berichtformaat** (uit `/usr/include/mbn.h` op de engine):
  - de data is in 7-bits bytes verpakt, met de laagste bits eerst;
  - een bericht bevat: object (2 bytes), actie, datatype, grootte en waarde;
  - actie **34** = *sensor changed* (iemand bedient iets);
  - actie **66** = *set actuator* (de engine zet een lampje, display, motorfader of meter);
  - meters zijn *floats* van 2 bytes (half precision) in dB.
- **Objectnummers** van de surface beginnen bij **1024**. Bijvoorbeeld 1084 = Fader 1, 1030 = Display 1 regel 2, 1044/1045 = Meter 5 L/R.

### Indeling van `/var/lib/axum/.backup`

Het bestand is een geheugenkopie van axum-engine (2,4 MB, little-endian, 32-bits). De indeling heb ik achterhaald door twee kopieën te vergelijken terwijl iemand de surface bediende. Ze geldt voor de axum-engine op deze engine en kan bij een andere versie anders zijn; AXUM Inzicht controleert de waardes en meldt "onbekende indeling" als iets niet klopt.

| Onderdeel | Begin | Aantal × grootte | Velden |
|---|---|---|---|
| Bronnen | 168 | 1280 × 148 bytes | label (32 bytes), **Start** = byte +133 |
| Modules | 374280 (niveau van module 1) | 128 × 4004 bytes | niveau = float dB; actuele bron = int op niveau−348; 16 bussen × 32 bytes vanaf niveau+24, **aan** = byte +8 |
| Monitorbussen | 2450600 | 16 × 108 bytes | label (32 bytes), bus-keuze = 16 bytes vanaf +56, **Phones** = float dB op +80, **Speaker** = float dB op +84 |

Een fader helemaal omlaag is in de engine **-138,1 dB**. De officiële fader-curve (dB ↔ stand 0–1023) staat in de tabel **`db_to_position`**.

---

## SSH naar de engine

De engine heeft een oude SSH-server (alleen `ssh-rsa`/`ssh-dss`). Daarom gebruiken de commando's extra opties.

### Linux

- **Sleutel:** `~/.ssh/id_rsa_axum` (RSA 4096, zonder wachtwoord).
- **Op de engine gezet** met `ssh-copy-id`. Dat moet in een **gewone terminal**, niet via `!` in Claude Code, want daar kan het wachtwoord niet ingetypt worden.

```bash
ssh -i ~/.ssh/id_rsa_axum -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa root@192.168.1.222
```

> Met deze sleutel kan iemand **als root** op de engine inloggen. Zet hem alleen op pc's die je vertrouwt.

### Windows

Dubbelklik eenmalig **`Koppel met engine.bat`** (zie hieronder). Het script maakt een eigen sleutel voor die pc. De sleutel van de Linux-pc hoeft dus niet gekopieerd te worden.

---

## Windows-versie

Het bestand **`AXUM-Inzicht-Windows.zip`** bevat alles, inclusief Python 3.12. Er hoeft niets geïnstalleerd te worden.

1. Pak de zip helemaal uit, bijvoorbeeld naar `C:\AXUM-Inzicht`.
2. **Eenmalig:** dubbelklik **`Koppel met engine.bat`**, typ het root-wachtwoord van de engine en wacht op **"GELUKT"**.
   - De engine-sleutel wordt automatisch geaccepteerd; je hoeft niet op "yes" te klikken.
   - Er komt een eigen sleutel in `C:\Users\<naam>\.ssh\id_rsa_axum`, en één regel in `authorized_keys` op de engine.
3. Dubbelklik **`Start AXUM Inzicht.bat`**. De browser opent http://localhost:8082.

Zonder koppeling werkt alles behalve de engine-stand en Live. Het Surface-tabblad is dan een simulatie.

Problemen:

- **"ssh niet gevonden":** Instellingen → Systeem → Optionele onderdelen → **OpenSSH-client**.
- **"UNPROTECTED PRIVATE KEY FILE":** de sleutel moet in je eigen `.ssh`-map staan, niet in de map van AXUM Inzicht.
- **Ander IP-adres van de engine:** pas `set AXUM_ENGINE=...` aan in `Start AXUM Inzicht.bat`.

De zip bevat **geen** SSH-sleutels en geen databasebackups.

---

## Bevindingen over de AXUM

### global_config raakt soms leeg

**Oorzaak:** de klokbron in `global_config` verwijst naar een node in `addresses`, met

```sql
FOREIGN KEY (ext_clock_addr) REFERENCES addresses(addr) ON DELETE CASCADE
```

Verdwijnt de node van de klokbron uit `addresses`, bijvoorbeeld omdat de gateway na een herstart een nieuw adres krijgt, dan **verwijdert Postgres automatisch de hele `global_config`-regel**. Op 19-11-2026 gebeurde dat toen de MambaNet Gateway van adres 26 naar adres 1 ging.

**Herstel**, op 19-11-2026 gedaan na een backup:

```sql
INSERT INTO global_config (samplerate, headroom, level_reserve, auto_momentary, startup_state, ext_clock_addr)
VALUES (48000, 20.0, 10.0, true, true, 1);   -- 1 = Axum MambaNet Gateway
```

- **Inlezen door de engine:** de engine leest dit pas bij een herstart, of als de pagina *Global configuration* in de oude webpagina één keer wordt opgeslagen. De trigger reageert alleen op een UPDATE.
- **Mogelijke blijvende oplossing (niet uitgevoerd):** de koppeling wijzigen naar `ON DELETE SET NULL`.

### Import-offset van kaart-functies

Een *predefined config* slaat **vaste posities** van bronnen en bestemmingen op, zoals ze waren op de mixer waar hij gemaakt is. Bij het importeren telt de engine de offset op en zoekt het resultaat op als **positie in de huidige lijst**. Klopt de positie niet, dan koppelt hij de functie **stil aan de eerste bron of bestemming**. De juiste offset is dus: positie van de eerste bron van de kaart in jouw lijst, min de positie in de predefined config. Het tabblad Kaart-functies rekent dit uit.

### Overige

- **Surface-adressen** veranderen bij opnieuw aanmelden (bijvoorbeeld 7 → 20); het unique ID blijft gelijk.
- Er staat nog een **oude offline surface** in de database: adres 109 / 0000006D, laatst gezien in 2023.
- **Fader-schaal op de Axite:** "0" bovenaan = **+10 dB** in de engine (`level_reserve` = 10).
- **Monitorkeuze is vergrendeld:** de laatst gekozen bus van een monitor kan niet uit. De engine zet het lampje meteen weer aan.
- De engine stuurt lampje-standen **elke seconde** opnieuw en meters ongeveer **9× per seconde per kanaal**. AXUM Inzicht tekent alleen opnieuw als er echt iets verandert.

---

## Bestanden

```
claude-axumOpnieuw3/
├── start.sh                    start op Linux (poort 8082)
├── README.md                   dit bestand
├── AXUM-Inzicht-Windows.zip    Windows-versie
├── backups/                    databasebackups (pg_dump)
├── windows/
│   ├── Start AXUM Inzicht.bat
│   ├── Koppel met engine.bat   eenmalig SSH koppelen
│   └── LEESMIJ.txt
└── axum-ui/
    ├── app.py                  FastAPI: database lezen, analyse, live-wijzigingen (LISTEN/NOTIFY)
    ├── system.py               Systeem-tabblad (vervanging van /system)
    ├── mbnlive.py              MambaNet meelezen + .backup inlezen (via SSH)
    └── static/
        ├── index.html
        ├── app.js
        └── style.css
```

### API

| Pad | Inhoud |
|---|---|
| `/api/state` | Alle configuratie uit Postgres, inclusief analyse, problemen en fader-curve |
| `/api/events` | Live configuratiewijzigingen (SSE) |
| `/api/mbn/backup` | De stand uit `/var/lib/axum/.backup` |
| `/api/mbn/events` | Live MambaNet-berichten (SSE); start het meelezen op de engine |
| `/api/system/...` | Gegevens voor het Systeem-tabblad |

---

## Onderhoud en backups

- **Regel:** verander niets op de engine (database of bestanden) zonder eerst een backup te maken.
- **Database-backup van 19-11-2026:**
  - `backups/axum_backup_20261119_2315.sql`
  - op de engine: `/root/axum_backup_20261119_2315.sql`
- **Zelf een nieuwe backup maken:**

  ```bash
  ssh -i ~/.ssh/id_rsa_axum -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa \
      root@192.168.1.222 'pg_dump -U axum axum' > backups/axum_backup_$(date +%Y%m%d_%H%M).sql
  ```

- **Nog open (eerder besproken):**
  - logrotate van de engine-logs, inclusief `postgresql.log` (nu 27 MB);
  - een schijfruimtemeter;
  - een tabblad "bekende fouten" in de oude webpagina.
