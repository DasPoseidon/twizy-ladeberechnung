# Twizy Ladeberechnung

Automatisches, preisoptimiertes Laden zweier Renault Twizy an zwei
schaltbaren Steckdosen ("innen"/"außen") an einem gemeinsamen 3‑kW-
Stromkreis, mit OVMS-Standorterkennung und Tibber-Dynamiktarif.

## Kurz beantwortet: Reicht ein einzelnes Blueprint?

**Nein – und zwar aus einem strukturellen Grund, nicht aus Bequemlichkeit.**
Ein Home-Assistant-Blueprint ist eine Vorlage für *eine* Automatisierung
(oder ein Skript). Es kann keinen eigenen, dauerhaften Zustand halten. Diese
Aufgabe braucht aber genau das:

- welches Fahrzeug hängt gerade an welcher Steckdose (ändert sich bei jeder
  Ankunft/Abfahrt),
- die 1×/Tag gecachten Tibber-Preise,
- ob gerade manuell übersteuert wurde,
- ob eine Steckdose wegen des 3‑kW-Limits pausiert ist,
- der aktuell berechnete Ladeplan.

Ein Monolith-Blueprint, das all das in einer einzigen riesigen Automatisierung
mit Templating nachbildet, wäre unlesbar, schwer zu debuggen und würde bei
jeder Kleinigkeit alle anderen Teile mit anfassen.

**Der saubere Weg – und der hier gewählte:**

1. Ein kleines **Helper-Paket** (`packages/twizy_charging.yaml`) legt alle
   `input_*`-Entities an, die den Zustand halten.
2. Vier fokussierte, wiederverwendbare **Blueprints**
   (`blueprints/automation/twizy/`), die jeweils eine klar abgegrenzte
   Verantwortung haben und über diese Helper miteinander "kommunizieren":
   - Tibber-Preis-Cache (1×/Tag)
   - Steckdosen-Zuordnung (Ankunft & Platztausch)
   - Lade-Scheduler (je Fahrzeug einmal instanziiert)
   - Gemeinsames Stromkreis-Limit

Das ist die in der Home-Assistant-Community übliche Architektur für
Automatisierungen, die mehr als "wenn X dann Y" brauchen: Helper für
Zustand, Blueprints/Automatisierungen für Verhalten. Alternativen wie
AppDaemon/pyscript oder NodeRED wären ebenfalls möglich (mehr
Programmierkomfort, z. B. für die Preisoptimierung), erfordern aber eine
zusätzliche Laufzeitumgebung. Die hier gewählte Lösung kommt komplett mit
Bordmitteln (Blueprints + Helpers + Templates) aus.

## Architektur-Übersicht

```
┌─────────────────────────┐   1×/Tag    ┌──────────────────────────┐
│ Tibber-Preis-Sensor      │ ──────────▶ │ tibber_price_cache.yaml  │
│ (offizielle Integration) │             │ (Blueprint)              │
└─────────────────────────┘             └──────────┬───────────────┘
                                                     │ schreibt
                                                     ▼
                                     input_text.tibber_guenstige_stunden_*
                                                     │ liest
                                                     ▼
┌─────────────────────────┐             ┌──────────────────────────┐
│ OVMS Standort/Trip       │ ──────────▶ │ socket_assignment.yaml   │
│ Sensoren                 │             │ (Blueprint)              │
└─────────────────────────┘             └──────────┬───────────────┘
                                                     │ schreibt
                                                     ▼
                                 input_select.twizy_socket_{innen,aussen}_belegt_durch
                                                     │ liest
                        ┌────────────────────────────┴───────────────────────────┐
                        ▼                                                        ▼
        ┌──────────────────────────┐                              ┌──────────────────────────┐
        │ charge_scheduler.yaml    │                              │ charge_scheduler.yaml    │
        │ Instanz "twizy_1"        │                              │ Instanz "twizy_2"        │
        └──────────┬───────────────┘                              └──────────┬───────────────┘
                   │ switch.turn_on/off                                       │
                   ▼                                                          ▼
             Steckdose innen/außen (je nach Zuordnung)         Steckdose innen/außen (je nach Zuordnung)
                   ▲                                                          ▲
                   │                        Leistungssensoren                │
                   └──────────────┬───────────────────────────────────────────┘
                                  ▼
                    ┌──────────────────────────────┐
                    │ shared_circuit_guard.yaml     │
                    │ (Blueprint, überwacht Summe    │
                    │ beider Leistungssensoren)      │
                    └──────────────┬─────────────────┘
                                   │ setzt
                                   ▼
                 input_boolean.twizy_{1,2}_leistungsbedingt_pausiert
                                   │ wird respektiert von
                                   ▼
                        charge_scheduler.yaml (beide Instanzen)
```

## Voraussetzungen in Home Assistant

- Offizielle **Tibber**-Integration (liefert einen Sensor mit den
  Attributen `today`/`tomorrow`).
- **OVMS**-Integration für beide Twizys mit (mindestens):
  - Standort/Zone-Entität (device_tracker oder Zone-Sensor, Zustand
    `home`/`not_home`)
  - "Fährt gerade"-Sensor (Moving/Trip binary_sensor)
  - SoC-Sensor (%)
  - geschätzte Restzeit bis voll (Minuten)
  - Ladekabel-gesteckt-Sensor (binary_sensor)
- Zwei schaltbare Steckdosen mit **Leistungsmessung** (z. B. Shelly, Sonoff
  POW, Tasmota-Plug) – ein `switch`- und ein `sensor`(W)-Entity je Steckdose.

## Installation

1. Repo-Inhalt in die Home-Assistant-Konfiguration übernehmen:
   - `packages/twizy_charging.yaml` nach `<config>/packages/` kopieren.
   - Sicherstellen, dass Packages aktiviert sind:
     ```yaml
     homeassistant:
       packages: !include_dir_named packages
     ```
   - `blueprints/automation/twizy/*.yaml` nach
     `<config>/blueprints/automation/twizy/` kopieren (oder das Repo direkt
     als Blueprint-Quelle importieren).
2. Home Assistant neu laden (YAML-Konfiguration neu laden reicht,
   Neustart nicht zwingend nötig).
3. Unter **Einstellungen → Automatisierungen → Blueprints** die vier
   Blueprints als Automatisierungen anlegen:

   | Blueprint | Wie oft anlegen | Wichtige Eingaben |
   |---|---|---|
   | Tibber-Preise cachen | 1× | Tibber-Sensor |
   | Steckdosen-Zuordnung | 1× | Standort- & Moving-Sensoren beider Fahrzeuge |
   | Lade-Scheduler | **2×** (einmal je Fahrzeug) | `vehicle_id` auf `twizy_1`/`twizy_2` setzen, jeweils die OVMS-Sensoren **des jeweiligen Fahrzeugs**, sowie bei der zweiten Instanz die `twizy_2_*`-Helper statt der `twizy_1_*`-Defaults auswählen |
   | Gemeinsames Stromkreis-Limit | 1× | beide Schalter + Leistungssensoren |

4. In den Helpern (`Einstellungen → Geräte & Dienste → Helfer`) die
   Abfahrtszeiten (`twizy_1_abfahrtszeit`, `twizy_2_abfahrtszeit`) und bei
   Bedarf das Stromkreis-Limit (`twizy_max_gesamtleistung_watt`, Default
   3000 W) anpassen.

## Verhalten im Detail

### Tibber-Preise cachen (1×/Tag)
Liest kurz nachdem Tibber die Preise des Folgetags veröffentlicht (Default
13:05 Uhr) die Preise aus dem Sensor-Attribut, ermittelt die günstigsten N
Stunden im Cache-Fenster (heute+morgen) und speichert sie kompakt (als
Unix-Epoch-Stunden, semikolon-getrennt) in `input_text`-Helpern. **Alle**
anderen Automatisierungen lesen ausschließlich diese Helper – der
Tibber-Sensor/-Service wird zwischen den täglichen Läufen nicht mehr
angefasst.

### Steckdosen-Zuordnung
- Fährt ein Fahrzeug in die `home`-Zone ein, wird es "außen" zugeordnet;
  war das andere Fahrzeug da bereits zuhause, wird es auf "innen"
  zurückgestuft. Da diese Regel bei jeder Ankunft neu greift, landet immer
  das zuletzt angekommene Fahrzeug auf "außen".
- Bewegen sich beide Fahrzeuge kurz (laut OVMS-Moving-Sensor), ohne die
  `home`-Zone zu verlassen, und enden beide Fahrten innerhalb des
  konfigurierbaren Zeitfensters (Default 15 min), wird ein Platztausch
  angenommen und die Zuordnung vertauscht.

### Verifikation zu Ladebeginn
Bevor der Lade-Scheduler eine Steckdose tatsächlich einschaltet, prüft er,
ob am dafür laut Zuordnung vorgesehenen Fahrzeug laut OVMS überhaupt ein
Ladekabel gesteckt ist. Falls nicht, wird **nicht** eingeschaltet, sondern
eine Benachrichtigung ausgelöst und `input_boolean.twizy_socket_zuordnung_geprueft`
auf "aus" gesetzt – ein Hinweis, dass die Zuordnung von Hand korrigiert
werden sollte.

### Preisoptimiertes Laden & Abfahrtszeit
Für jedes Fahrzeug wird pro Prüfzyklus (Default alle 10 min) berechnet:
- benötigte Ladedauer: standardmäßig die von OVMS geschätzte Restzeit bis
  voll; per Schalter-Helper (`twizy_{1,2}_eigene_ladedauer_verwenden`) auf
  einen selbst gepflegten Minutenwert (`twizy_{1,2}_eigene_ladedauer_minuten`)
  umschaltbar – vorbereitet für eine spätere eigene Berechnung.
- ob die aktuelle Stunde zu den günstigsten Stunden vor der Abfahrtszeit
  gehört, die in Summe die Ladedauer abdecken.
- eine **Aufhol-Logik**: reicht die verbleibende Zeit bis zur Abfahrt nicht
  mehr aus, um die benötigte Dauer allein aus günstigen Stunden zu decken,
  wird unabhängig vom Preis sofort weiter geladen, damit die Deadline nicht
  gerissen wird.

### Manuelles Einschalten
Wird eine Steckdose von Hand eingeschaltet (erkannt daran, dass die
Zustandsänderung keinen Automatisierungs-Kontext hat – eine gängige, nicht
hundertprozentig fälschungssichere Heuristik), wird das als manueller
Ladewunsch übernommen: Es wird sofort weitergeladen, die Abfahrtszeit bleibt
aber als spätester "voll"-Zeitpunkt gültig – die Automatisierung schaltet
weiterhin ab, sobald der SoC-Schwellwert erreicht ist.

### Gemeinsames 3-kW-Limit
Überschreitet die Summe beider Leistungssensoren für die konfigurierte
Entprellzeit (Default 30 s) das Limit, wird eine der beiden Steckdosen
abgeschaltet (Heuristik: die zuletzt gestartete, siehe Blueprint-Beschreibung)
und als "leistungsbedingt pausiert" markiert. Nach ausreichend langer
Erholphase (Default 5 min unter Limit − Sicherheitsabstand) wird die Pause
aufgehoben; das Wiedereinschalten übernimmt dann regulär der jeweilige
Lade-Scheduler.

## Bekannte Vereinfachungen

- Die Abfahrtszeit ist ein einzelner täglicher Zeitpunkt (kein
  Wochentags-Zeitplan). Für unterschiedliche Zeiten je Wochentag könnte man
  die generierte Automatisierung um eine `weekday`-Bedingung erweitern oder
  mehrere Instanzen mit Zeitfenster-Bedingungen anlegen.
- Die Priorisierung beim Stromkreis-Limit berücksichtigt nicht, welches
  Fahrzeug die knappere Abfahrtszeit hat, sondern pausiert die zuletzt
  gestartete Ladung.
- Die "manuell eingeschaltet"-Erkennung basiert auf der Home-Assistant-
  Heuristik "Zustandsänderung ohne automation-Kontext" und ist nicht zu
  100 % robust (z. B. wenn ein Skript ohne eigenen Kontext schaltet).
- Die eigene Ladedauer-Berechnung (Alternative zur OVMS-Schätzung) ist als
  Schalter + Helper vorbereitet, die eigentliche Berechnung müsste noch mit
  fahrzeugspezifischen Werten (Akkukapazität, Ladeleistung, Temperatur-
  Einfluss o. Ä.) befüllt werden – aktuell trägt man dort einen Minutenwert
  von Hand ein.

## Repository-Struktur

```
packages/
  twizy_charging.yaml            # alle Helper-Entities
blueprints/automation/twizy/
  tibber_price_cache.yaml        # Preise 1x/Tag cachen
  socket_assignment.yaml         # innen/außen-Zuordnung
  charge_scheduler.yaml          # Lade-Entscheidung je Fahrzeug (2x instanziieren)
  shared_circuit_guard.yaml      # 3kW-Stromkreis-Schutz
```
