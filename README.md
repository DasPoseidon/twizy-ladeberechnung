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
- ob gerade manuell übersteuert wurde,
- der aktuell berechnete Ladeplan.

Ein Monolith-Blueprint, das all das in einer einzigen riesigen Automatisierung
mit Templating nachbildet, wäre unlesbar, schwer zu debuggen und würde bei
jeder Kleinigkeit alle anderen Teile mit anfassen.

**Der saubere Weg – und der hier gewählte:**

1. Ein kleines **Helper-Paket** (`packages/twizy_charging.yaml`) legt alle
   `input_*`-Entities an, die den Zustand halten.
2. Zwei fokussierte, wiederverwendbare **Blueprints**
   (`blueprints/automation/twizy/`), die jeweils eine klar abgegrenzte
   Verantwortung haben und über diese Helper miteinander "kommunizieren":
   - Steckdosen-Zuordnung (Ankunft & Platztausch)
   - Lade-Scheduler (je Fahrzeug einmal instanziiert; ruft die Tibber-Preise
     per Service ab, kein eigener Cache, und berücksichtigt das gemeinsame
     Stromkreis-Limit nur bei der Planung, nicht per aktiver Überwachung –
     siehe unten)

Das ist die in der Home-Assistant-Community übliche Architektur für
Automatisierungen, die mehr als "wenn X dann Y" brauchen: Helper für
Zustand, Blueprints/Automatisierungen für Verhalten. Alternativen wie
AppDaemon/pyscript oder NodeRED wären ebenfalls möglich (mehr
Programmierkomfort, z. B. für die Preisoptimierung), erfordern aber eine
zusätzliche Laufzeitumgebung. Die hier gewählte Lösung kommt komplett mit
Bordmitteln (Blueprints + Helpers + Templates) aus.

## Architektur-Übersicht

```
┌─────────────────────────┐
│ OVMS Standort/Trip       │ ──────────▶ ┌──────────────────────────┐
│ Sensoren                 │             │ socket_assignment.yaml   │
└─────────────────────────┘             │ (Blueprint)              │
                                         └──────────┬───────────────┘
                                                     │ schreibt
                                                     ▼
                                 input_select.twizy_socket_{innen,aussen}_belegt_durch
                                                     │ liest
┌─────────────────────────┐                         │
│ Tibber-Preis-Sensor      │ ─────────liest──────────┤
│ (offizielle Integration) │                         │
└─────────────────────────┘                         │
                        ┌────────────────────────────┴───────────────────────────┐
                        ▼                                                        ▼
        ┌──────────────────────────┐                              ┌──────────────────────────┐
        │ charge_scheduler.yaml    │                              │ charge_scheduler.yaml    │
        │ Instanz "twizy_1"        │                              │ Instanz "twizy_2"        │
        └──────────┬───────────────┘                              └──────────┬───────────────┘
                   │ switch.turn_on/off                                       │
                   ▼                                                          ▼
             Steckdose innen/außen (je nach Zuordnung)         Steckdose innen/außen (je nach Zuordnung)
                   ▲            │ Leistungssensor (vor dem Einschalten          ▲            │
                   │            │ gegengeprüft, siehe "Gemeinsames Limit")      │            │
                   └────────────┴────────────────────────────────────────────────────────────┘
```

Die beiden Steckdosen haben eigenen Überlastschutz (werden bei zu hoher
Last selbstständig `unavailable` und später wieder `off`). Es gibt daher
**keine** dritte, aktiv überwachende Automatisierung mehr – jeder
Lade-Scheduler prüft nur vor dem eigenen Einschalten kurz den
Leistungssensor der jeweils anderen Steckdose.

## Voraussetzungen in Home Assistant

- Offizielle **Tibber**-Integration. Die Stundenpreise werden über deren
  Service `tibber.get_prices` abgerufen (die neueren Versionen der
  Integration stellen `today`/`tomorrow` **nicht** mehr als Sensor-Attribute
  bereit – nur noch aggregierte Werte wie `min_price`/`max_price`/`peak`).
- **OVMS**-Integration für beide Twizys mit (mindestens):
  - Standort/Zone-Entität (device_tracker oder Zone-Sensor, Zustand
    `home`/`not_home`)
  - "Fährt gerade"-Sensor (Moving/Trip binary_sensor)
  - SoC-Sensor (%)
  - geschätzte Restzeit bis voll (Minuten)
  - Lade-/Fahrzustand als Text-Sensor (z. B. "charging"/"topoff"/"done"/
    "stopped" – ein eigener "Kabel gesteckt"-Sensor wird nicht
    vorausgesetzt, da viele OVMS-Setups keinen solchen bereitstellen)
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
3. Unter **Einstellungen → Automatisierungen → Blueprints** die beiden
   Blueprints als Automatisierungen anlegen:

   | Blueprint | Wie oft anlegen | Wichtige Eingaben |
   |---|---|---|
   | Steckdosen-Zuordnung | 1× | Standort- & Moving-Sensoren beider Fahrzeuge |
   | Lade-Scheduler | **2×** (einmal je Fahrzeug) | `vehicle_id` auf `twizy_1`/`twizy_2` setzen, jeweils die OVMS-Sensoren **des jeweiligen Fahrzeugs**, beide Schalter + beide Leistungssensoren, sowie bei der zweiten Instanz die `twizy_2_*`-Helper statt der `twizy_1_*`-Defaults auswählen |

4. In den Helpern (`Einstellungen → Geräte & Dienste → Helfer`) die
   Abfahrtszeiten (`twizy_1_abfahrtszeit`, `twizy_2_abfahrtszeit`) und bei
   Bedarf das Stromkreis-Limit (`twizy_max_gesamtleistung_watt`, Default
   3000 W) sowie die Mindest-Einschaltzeit pro Tag
   (`twizy_{1,2}_mindestlaufzeit_minuten`, Default 30 min) anpassen.

## Verhalten im Detail

### Tibber-Preise
Der Lade-Scheduler ruft bei jedem Prüfzyklus den Service `tibber.get_prices`
auf (Zeitraum heute 00:00 bis übermorgen 00:00) und baut daraus die Liste
der Stundenpreise – kein eigener Cache, kein Zwischenspeicher-Helper.

**Wichtig, falls du das nachvollziehen willst:** Ältere Community-Blueprints
und -Anleitungen gehen oft davon aus, dass der Tibber-Preis-Sensor
`today`/`tomorrow`-Attribute mit den Stundenpreisen hat. Das stimmt für die
aktuelle offizielle Integration **nicht mehr** – der Sensor liefert nur noch
aggregierte Werte (`min_price`, `max_price`, `avg_price`, `off_peak_1`,
`peak`, `off_peak_2`, `intraday_price_ranking`, s.
[`homeassistant/components/tibber/sensor.py`](https://github.com/home-assistant/core/blob/dev/homeassistant/components/tibber/sensor.py)).
Die vollständige Stundenliste bekommt man nur noch über den Service
`tibber.get_prices` (Antwortformat: `{"prices": {"<Zuhause-Name>": [{"start_time": ..., "price": ...}, ...]}}`,
s.
[`homeassistant/components/tibber/services.py`](https://github.com/home-assistant/core/blob/dev/homeassistant/components/tibber/services.py)).
Das kannst du in **Entwicklerwerkzeuge → Aktionen** selbst ausprobieren:
Aktion `tibber.get_prices` auswählen, "Antwort anzeigen" aktivieren, und
ausführen.

Dass der Scheduler diesen Service bei jedem Prüfzyklus aufruft, führt trotzdem
nicht zu häufigen echten API-Aufrufen: Die Tibber-Integration ruft die
eigentliche Tibber-API dabei nur auf, wenn die angefragten Daten nicht schon
lokal vorliegen – ihr interner `TibberFetchPriceCoordinator` prüft das
ohnehin alle 1–10 Minuten rein lokal und holt neue Daten nur, wenn die
Preise für heute komplett fehlen oder die für morgen fehlen und ein
randomisierter Zeitpunkt zwischen 14:00 und 22:00 Uhr überschritten ist –
in der Praxis also ungefähr **1× pro Tag** (Quelle:
[`homeassistant/components/tibber/coordinator.py`](https://github.com/home-assistant/core/blob/dev/homeassistant/components/tibber/coordinator.py)).

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
Da kein zuverlässiger "Kabel gesteckt"-Sensor vorausgesetzt wird, erfolgt
die Verifikation *nach* dem Einschalten, anhand des OVMS-Lade-/Fahrzustands:
Meldet der Sensor des zugeordneten Fahrzeugs innerhalb der Toleranzzeit
(Default 15 min) einen "lädt aktiv"-Wert (Default nur `charging`,
konfigurierbar), gilt die Zuordnung als bestätigt
(`input_boolean.twizy_socket_zuordnung_geprueft` → an). Passiert das nicht,
wird davon ausgegangen, dass die Zuordnung falsch ist: Die Steckdose wird
wieder ausgeschaltet und eine Benachrichtigung ausgelöst (dedupliziert über
eine feste `notification_id`, kein Spam bei wiederholten Versuchen). Der
nächste Prüfzyklus versucht es automatisch erneut, bis entweder die
Zuordnung korrigiert wurde oder tatsächlich geladen wird.

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

### Laden abgeschlossen & Mindest-Einschaltzeit pro Tag
Ist der SoC-Schwellwert erreicht, wird die Steckdose abgeschaltet – so weit
die Grundregel. Zusätzlich gibt es eine einstellbare **Mindest-Einschaltzeit
pro Tag** (`twizy_{1,2}_mindestlaufzeit_minuten`, Default 30 min, 0
deaktiviert die Funktion): Damit die Steckdose auch an Tagen, an denen das
Fahrzeug schon voll ist (z. B. weil es kaum gefahren wurde), nicht komplett
stromlos bleibt, wird sie zur günstigsten verbleibenden Stunde trotzdem
nochmal eingeschaltet, bis die Mindestzeit erreicht ist.

Dafür führt der Lade-Scheduler intern einen Tages-Zähler
(`twizy_{1,2}_einschaltzeit_minuten_heute` + `twizy_{1,2}_einschaltzeit_tag`),
der bei jedem Ausschalten um die Dauer der gerade beendeten Einschaltzeit
erhöht wird – unabhängig davon, ob das eine reguläre Ladesitzung oder ein
gezielter "Auffüll"-Vorgang war. Reicht die bisherige Einschaltzeit des
Tages schon aus (z. B. weil vorher regulär lange geladen wurde), wird keine
zusätzliche Zeit erzwungen. Der Zähler setzt sich beim nächsten Tageswechsel
automatisch zurück (erkannt am gespeicherten Datum).

### Manuelles Einschalten
Wird eine Steckdose von Hand eingeschaltet (erkannt daran, dass die
Zustandsänderung keinen Automatisierungs-Kontext hat – eine gängige, nicht
hundertprozentig fälschungssichere Heuristik), wird das als manueller
Ladewunsch übernommen: Es wird sofort weitergeladen, die Abfahrtszeit bleibt
aber als spätester "voll"-Zeitpunkt gültig – die Automatisierung schaltet
weiterhin ab, sobald der SoC-Schwellwert erreicht ist.

### Gemeinsames 3-kW-Limit
Es gibt **keine aktive Überwachung/Abschaltung** durch die Automatisierung –
die Steckdosen haben eigenen Überlastschutz und werden bei Überlast von
selbst `unavailable`, bis sie sich erholt haben und wieder im Zustand "aus"
verfügbar sind. Das Limit wird stattdessen nur bei der **Planung**
berücksichtigt: Bevor der Lade-Scheduler eine Steckdose einschaltet, prüft
er, ob die jeweils andere Steckdose gerade eingeschaltet ist, und falls ja,
ob deren gemessene Leistung plus die eigene, typische Ladeleistung
(einstellbarer Schätzwert, Default 2300 W) das Limit überschreiten würde.
Wenn ja, wird nicht eingeschaltet – der nächste Prüfzyklus (Default alle
10 min) versucht es erneut, z. B. sobald das andere Fahrzeug fertig geladen
hat. Eine manuell eingeschaltete Steckdose übersteuert diese Vorsicht
bewusst (siehe "Manuelles Einschalten").

Kommt es trotzdem zu einer Überlast (z. B. weil beide Steckdosen manuell
gleichzeitig eingeschaltet wurden oder die tatsächliche Ladeleistung höher
als geschätzt war), greift der Überlastschutz der Steckdose selbst. Sobald
sie – laut Home Assistant am Zustandswechsel `unavailable` → `off` erkennbar
– wieder verfügbar ist, bewertet der jeweilige Lade-Scheduler sofort neu, ob
wieder eingeschaltet werden soll.

## Bekannte Vereinfachungen

- Die Abfahrtszeit ist ein einzelner täglicher Zeitpunkt (kein
  Wochentags-Zeitplan). Für unterschiedliche Zeiten je Wochentag könnte man
  die generierte Automatisierung um eine `weekday`-Bedingung erweitern oder
  mehrere Instanzen mit Zeitfenster-Bedingungen anlegen.
- Die Überlast-Vermeidung beim Einschalten arbeitet mit einem geschätzten
  Wert für die eigene typische Ladeleistung (nicht mit einer live
  gemessenen eigenen Leistung, die vor dem Einschalten ja noch bei ~0 W
  liegt). Bei knappen Fällen lieber etwas großzügiger schätzen.
- Wenn beide Steckdosen gleichzeitig manuell eingeschaltet werden, greift
  keine Vorab-Prüfung (Manuell übersteuert die Überlast-Vermeidung bewusst)
  – hier verlässt sich die Lösung vollständig auf den Überlastschutz der
  Steckdosen selbst.
- Die "manuell eingeschaltet"-Erkennung basiert auf der Home-Assistant-
  Heuristik "Zustandsänderung ohne automation-Kontext" und ist nicht zu
  100 % robust (z. B. wenn ein Skript ohne eigenen Kontext schaltet).
- Die Verifikation über den Lade-/Fahrzustand greift auch bei manuell
  gestarteten Ladungen (bewusst so, damit eine falsche Zuordnung immer
  auffällt) – dadurch schaltet sich eine manuell eingeschaltete Steckdose
  nach der Toleranzzeit wieder ab, wenn OVMS kein "lädt aktiv" meldet,
  selbst wenn das gewünscht gewesen wäre. Bei Bedarf `verification_grace_period`
  großzügiger einstellen oder die Bedingung im Blueprint anpassen.
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
  socket_assignment.yaml         # innen/außen-Zuordnung
  charge_scheduler.yaml          # Lade-Entscheidung je Fahrzeug (2x instanziieren),
                                  # inkl. Überlast-Vermeidung bei der Planung
```
