# Twizy Ladeberechnung

Automatisches, preisoptimiertes Laden zweier Renault Twizy an zwei
schaltbaren Steckdosen ("innen"/"außen") an einem gemeinsamen 3‑kW-
Stromkreis, mit OVMS-Standorterkennung und Tibber-Dynamiktarif.

## Inhalt

- [Architektur](#architektur)
- [Voraussetzungen in Home Assistant](#voraussetzungen-in-home-assistant)
- [Installation](#installation)
- [Verhalten im Detail](#verhalten-im-detail)
  - [Tibber-Preise](#tibber-preise)
  - [Steckdosen-Zuordnung](#steckdosen-zuordnung)
  - [Verifikation & Verbindungsprüfung](#verifikation-zu-ladebeginn--verbindungspr%C3%BCfung)
  - [Preisoptimiertes Laden & Abfahrtszeit](#preisoptimiertes-laden--abfahrtszeit)
  - [Selbstlernender Korrekturfaktor](#selbstlernender-korrekturfaktor-f%C3%BCr-die-ovms-restzeitsch%C3%A4tzung)
  - [Ladeschluss-Erkennung](#ladeschluss-erkennung)
  - [Mindest-Einschaltzeit pro Tag](#mindest-einschaltzeit-pro-tag)
  - [Manuelles Einschalten](#manuelles-einschalten)
  - [Steckdose pausieren](#steckdose-pausieren-z-b-f%C3%BCr-rasenm%C3%A4her)
  - [Gemeinsames 3-kW-Limit](#gemeinsames-3-kw-limit)
- [Bekannte Vereinfachungen](#bekannte-vereinfachungen)
- [Repository-Struktur](#repository-struktur)

## Architektur

Ein Home-Assistant-Blueprint ist eine Vorlage für *eine* Automatisierung und
kann keinen eigenen, dauerhaften Zustand halten. Diese Aufgabe braucht aber
genau das: welches Fahrzeug gerade an welcher Steckdose hängt, ob manuell
übersteuert wurde, der aktuell berechnete Ladeplan. Ein Monolith-Blueprint,
das all das per Templating in einer einzigen Automatisierung nachbildet,
wäre unlesbar und schwer zu warten. Deshalb die Trennung:

1. Ein **Helper-Paket** (`packages/twizy_charging.yaml`) legt alle
   `input_*`-Entities an, die den Zustand halten.
2. Fokussierte, wiederverwendbare **Blueprints**
   (`blueprints/automation/twizy/`), die über diese Helper miteinander
   "kommunizieren":
   - **Steckdosen-Zuordnung** – 1× angelegt, pflegt innen/außen.
   - **Lade-Scheduler** – 2× angelegt (einmal je Fahrzeug), ruft die
     Tibber-Preise per Service ab (kein eigener Cache) und berücksichtigt
     das gemeinsame Stromkreis-Limit nur bei der Planung, nicht per aktiver
     Überwachung (siehe [Gemeinsames 3-kW-Limit](#gemeinsames-3-kw-limit)).
   - **Steckdose pausieren** – optional 1× angelegt, nimmt eine Steckdose
     vorübergehend aus der Ladesteuerung heraus (siehe
     [Steckdose pausieren](#steckdose-pausieren-z-b-f%C3%BCr-rasenm%C3%A4her)).

Das ist die in der Home-Assistant-Community übliche Architektur für
Automatisierungen, die mehr als "wenn X dann Y" brauchen: Helper für
Zustand, Blueprints für Verhalten. Alternativen wie AppDaemon/pyscript oder
NodeRED wären ebenfalls möglich, erfordern aber eine zusätzliche
Laufzeitumgebung – diese Lösung kommt komplett mit Bordmitteln aus.

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
**keine** dritte, aktiv überwachende Automatisierung – jeder Lade-Scheduler
prüft nur vor dem eigenen Einschalten kurz den Leistungssensor der jeweils
anderen Steckdose.

## Voraussetzungen in Home Assistant

- Offizielle **Tibber**-Integration. Die Preise werden über deren Service
  `tibber.get_prices` abgerufen (Stunden- oder Viertelstundenauflösung, je
  nach Vertrag – wird automatisch erkannt, siehe
  [Tibber-Preise](#tibber-preise)).
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
     Sicherstellen, dass Packages aktiviert sind:
     ```yaml
     homeassistant:
       packages: !include_dir_named packages
     ```
   - `blueprints/automation/twizy/*.yaml` nach
     `<config>/blueprints/automation/twizy/` kopieren (oder das Repo direkt
     als Blueprint-Quelle importieren).
2. Home Assistant neu laden (YAML-Konfiguration neu laden reicht,
   Neustart nicht zwingend nötig).
3. Unter **Einstellungen → Automatisierungen → Blueprints** die Blueprints
   als Automatisierungen anlegen:

   | Blueprint | Wie oft anlegen | Wichtige Eingaben |
   |---|---|---|
   | Steckdosen-Zuordnung | 1× | Standort- & Moving-Sensoren beider Fahrzeuge, beide Schalter |
   | Lade-Scheduler | **2×** (einmal je Fahrzeug) | `vehicle_id` auf `twizy_1`/`twizy_2` setzen, jeweils die OVMS-Sensoren + Standort-Entität **des jeweiligen Fahrzeugs**, beide Schalter + beide Leistungssensoren |
   | Steckdose pausieren | 1× (optional, für "außen") | Schalter + Leistungssensor der Steckdose "außen" – siehe [Steckdose pausieren](#steckdose-pausieren-z-b-f%C3%BCr-rasenm%C3%A4her) |

   Alle Helper-Entities (Abfahrtszeiten, Ladedauer, Korrekturfaktor,
   Zuordnungs-/Verifikations-Flags usw.) werden automatisch aus
   `vehicle_id` abgeleitet (`input_boolean.<vehicle_id>_manuelles_laden`
   usw.) – bei der zweiten Instanz muss dafür **nichts** manuell umgestellt
   werden, nur `vehicle_id` selbst auf `twizy_2` setzen.
4. In den Helpern (`Einstellungen → Geräte & Dienste → Helfer`) die
   Abfahrtszeiten pro Wochentag (`twizy_{1,2}_abfahrtszeit_montag` …
   `twizy_{1,2}_abfahrtszeit_sonntag`, je 7 Helper pro Fahrzeug) und bei
   Bedarf das Stromkreis-Limit (`twizy_max_gesamtleistung_watt`, Default
   3000 W) sowie die Mindest-Einschaltzeit pro Tag
   (`twizy_{1,2}_mindestlaufzeit_minuten`, Default 30 min) anpassen. Am
   einfachsten geht das über das Dashboard (nächster Schritt).
5. Optional, aber empfohlen: `dashboards/twizy_dashboard.yaml` als eigenes
   Dashboard einbinden, damit alle Helper an einem Ort bedienbar sind:
   1. **Einstellungen → Dashboards → "+ Dashboard hinzufügen"** →
      "Neues Dashboard von Grund auf erstellen" (beliebiger Titel, z. B.
      "Twizy").
   2. Das neue Dashboard öffnen, oben rechts **⋮ → Dashboard bearbeiten**,
      dann nochmal **⋮ → Raw-Konfigurationseditor**.
   3. Den kompletten Inhalt von `dashboards/twizy_dashboard.yaml` einfügen
      (vorhandenen Inhalt ersetzen) und speichern.

   Das Dashboard nutzt ausschließlich eingebaute Lovelace-Karten
   (`entities`-Karten) – keine HACS-Zusatzkarten nötig. Es zeigt Ladestrom
   (nach Steckdose) und SoC (nach Fahrzeug) direkt aus den echten
   Hardware-Sensoren an (siehe [Ladeschluss-Erkennung](#ladeschluss-erkennung))
   – falls du diese Sensor-Entity-IDs oder andere Helper-Namen im Paket
   änderst, müssen die `entity:`-Zeilen in der Dashboard-Datei entsprechend
   angepasst werden. Alle `input_number`-Helfer haben außerdem `mode: box`
   statt der HA-Standardeinstellung `mode: slider` gesetzt – im Dashboard
   erscheint dadurch ein Zahlenfeld mit +/‑-Schrittweite statt eines
   Schiebereglers, unempfindlicher gegen versehentliches Verstellen (z. B.
   beim Scrollen auf dem Handy).

> **Hinweis:** Die Helper in `packages/twizy_charging.yaml` sind bewusst
> *ohne* `initial:` definiert. Ist bei einem YAML-`input_*`-Helfer
> `initial:` gesetzt, erzwingt Home Assistant bei *jedem* Neustart/Reload
> diesen Wert statt den zuletzt eingestellten wiederherzustellen
> (offizielles, aber leicht zu übersehendes HA-Verhalten) – ohne `initial:`
> bleiben eingestellte Werte dagegen über Neustarts hinweg erhalten.

## Verhalten im Detail

### Tibber-Preise
Der Lade-Scheduler ruft bei jedem Prüfzyklus den Service `tibber.get_prices`
auf (Zeitraum heute 00:00 bis übermorgen 00:00) und baut daraus die
Preisliste – kein eigener Cache, kein Zwischenspeicher-Helper.

**Preis-Zeitraster wird automatisch erkannt:** Manche Tibber-Verträge
rechnen viertelstündlich statt stündlich ab. Der Lade-Scheduler geht nicht
fest von einer der beiden Auflösungen aus, sondern ermittelt bei jedem
Prüfzyklus aus dem tatsächlichen Abstand zwischen den gelieferten
Preiseinträgen, welches Zeitraster gerade gilt (`price_resolution_seconds`:
3600 bei Stundenpreisen, 900 bei Viertelstundenpreisen) – keine Einstellung
nötig, passt sich auch an, falls sich das Zeitraster später nochmal ändert.
Anzahl benötigter Zeitfenster, Fenstergrenzen und die Anzeige "nächster
Ladestart" rechnen entsprechend mit diesem erkannten Zeitraster.

Dass der Scheduler den Service bei jedem Prüfzyklus aufruft, führt trotzdem
nicht zu häufigen echten API-Aufrufen: Die Tibber-Integration ruft die
eigentliche Tibber-API nur auf, wenn die angefragten Daten nicht schon
lokal vorliegen – ihr interner `TibberFetchPriceCoordinator` prüft das
alle 1–10 Minuten rein lokal und holt neue Daten nur, wenn die Preise für
heute komplett fehlen oder die für morgen fehlen und ein randomisierter
Zeitpunkt zwischen 14:00 und 22:00 Uhr überschritten ist – in der Praxis
also ungefähr **1× pro Tag** (Quelle:
[`homeassistant/components/tibber/coordinator.py`](https://github.com/home-assistant/core/blob/dev/homeassistant/components/tibber/coordinator.py)).

<details>
<summary>Warum <code>tibber.get_prices</code> statt Sensor-Attributen?</summary>

Ältere Community-Blueprints und -Anleitungen gehen oft davon aus, dass der
Tibber-Preis-Sensor `today`/`tomorrow`-Attribute mit den Stundenpreisen
hat. Das stimmt für die aktuelle offizielle Integration **nicht mehr** –
der Sensor liefert nur noch aggregierte Werte (`min_price`, `max_price`,
`avg_price`, `off_peak_1`, `peak`, `off_peak_2`, `intraday_price_ranking`,
s.
[`homeassistant/components/tibber/sensor.py`](https://github.com/home-assistant/core/blob/dev/homeassistant/components/tibber/sensor.py)).
Die vollständige Preisliste bekommt man nur noch über den Service
`tibber.get_prices` (Antwortformat:
`{"prices": {"<Zuhause-Name>": [{"start_time": ..., "price": ...}, ...]}}`,
s.
[`homeassistant/components/tibber/services.py`](https://github.com/home-assistant/core/blob/dev/homeassistant/components/tibber/services.py)).
Selbst ausprobieren: **Entwicklerwerkzeuge → Aktionen** → Aktion
`tibber.get_prices` auswählen, "Antwort anzeigen" aktivieren, ausführen.

</details>

### Steckdosen-Zuordnung
- Fährt ein Fahrzeug in die `home`-Zone ein, wird es "außen" zugeordnet;
  war das andere Fahrzeug da bereits zuhause, wird es auf "innen"
  zurückgestuft. Da diese Regel bei jeder Ankunft neu greift, landet immer
  das zuletzt angekommene Fahrzeug auf "außen".
- Bewegen sich beide Fahrzeuge kurz (laut OVMS-Moving-Sensor), ohne die
  `home`-Zone zu verlassen, und enden beide Fahrten innerhalb des
  konfigurierbaren Zeitfensters (Default 15 min), wird ein Platztausch
  angenommen und die Zuordnung vertauscht.
- Verlässt ein Fahrzeug die `home`-Zone, wird seine zugeordnete Steckdose
  (falls es gerade "innen" oder "außen" zugeordnet war) direkt abgeschaltet
  und die Zuordnung auf "unbekannt" zurückgesetzt – eine leere Steckdose
  soll nicht weiter als von einem Fahrzeug belegt gelten, das gar nicht da
  ist. Ist die Steckdose "außen" gerade pausiert (siehe
  [Steckdose pausieren](#steckdose-pausieren-z-b-f%C3%BCr-rasenm%C3%A4her)),
  wird sie dabei nicht angetastet – nur die Zuordnung wird zurückgesetzt.
  Das Abschalten passiert bewusst hier direkt und nicht erst im
  Lade-Scheduler: Würde zuerst nur die Zuordnung zurückgesetzt, könnte der
  Lade-Scheduler die zugehörige Steckdose beim nächsten Lauf nicht mehr
  finden und käme gar nicht mehr zum Abschalten. Dabei werden auch die
  "Zuordnung verifiziert"- und "andere Steckdose bereits getestet"-Helfer
  des abfahrenden Fahrzeugs zurückgesetzt – eine Verifikation aus der
  vergangenen Sitzung soll nicht fälschlich für die nächste weitergelten.

### Verifikation zu Ladebeginn & Verbindungsprüfung
**Nur wenn das Fahrzeug laut Standort-Entität zuhause ist:** Ist es nicht
zuhause, bleibt die Steckdose aus bzw. wird abgeschaltet, falls sie gerade
an ist – auch bei einer manuell gestarteten Ladung, denn fährt das
Fahrzeug weg, gibt es nichts mehr zu laden (siehe
[Manuelles Einschalten](#manuelles-einschalten)). Es kommt dabei auch
keine Verifikations-Benachrichtigung. Das verhindert sinnlose
Einschaltversuche und Benachrichtigungen für ein Fahrzeug, das schlicht
nicht da ist. Ausgenommen ist nur die eigenständige
[Steckdose pausieren](#steckdose-pausieren-z-b-f%C3%BCr-rasenm%C3%A4her)-Funktion,
die unabhängig vom Fahrzeugstandort funktioniert.

Da kein zuverlässiger "Kabel gesteckt"-Sensor vorausgesetzt wird, erfolgt
die Verifikation *nach* dem Einschalten, anhand von zwei unabhängigen
Signalen: dem OVMS-Lade-/Fahrzustand ODER einem messbaren Ladestrom an der
zugeordneten Steckdose (über der Ladeschluss-Schwelle
`charge_complete_power_threshold`, Default 10 W). Meldet der
OVMS-Zustandssensor des zugeordneten Fahrzeugs innerhalb der Toleranzzeit
(Default 15 min) einen "lädt aktiv"-Wert (Default nur `charging`,
konfigurierbar) **oder** wird spürbar Strom gezogen, gilt die aktuelle
Zuordnung als bestätigt
(`input_boolean.twizy_{1,2}_zuordnung_geprueft` → an – **ein eigener Helper
je Fahrzeug**, nicht geteilt, sonst würde eine erfolgreiche Verifikation
von Fahrzeug 1 fälschlich auch für Fahrzeug 2 gelten). Dieser Helfer wird
bei **jedem** Einschalten zurückgesetzt (manuell oder automatisch) – eine
frühere erfolgreiche Verifikation gilt also nie für eine neue Ladesitzung
weiter, falls inzwischen z. B. ein anderes Fahrzeug an derselben Steckdose
hängt.

Passiert das nicht (weder OVMS-Zustand noch Ladestrom bestätigen etwas):
Steht die jeweils andere Steckdose gerade frei (kein Eingriff in eine
laufende Ladung des anderen Fahrzeugs) und wurde das für diese Ankunft noch
nicht versucht, schaltet der Lade-Scheduler testweise dorthin um –
vielleicht ist die Zuordnung einfach vertauscht. Bestätigt sich dort eines
der beiden Signale, bleibt die angepasste Zuordnung bestehen; falls nicht,
bleibt es bei dieser einen Alternative (kein
Hin-und-her zwischen den Steckdosen) und es wird nur noch benachrichtigt.
Bei manuellem Einschalten wird nicht automatisch umgeschaltet – dafür gibt
es die [Steckdose pausieren](#steckdose-pausieren-z-b-f%C3%BCr-rasenm%C3%A4her)-Funktion.
In beiden Fällen (kein Alternativ-Versuch möglich, oder auch die Alternative
ohne Erfolg) wird **nur eine Benachrichtigung ausgelöst** (dedupliziert über
eine feste `notification_id`, kein Spam bei wiederholten Versuchen) – mit
Hinweis auf die möglichen Ursachen: falsche Zuordnung *oder* Fahrzeug nicht
angeschlossen. Die Steckdose wird dabei bewusst **nicht** automatisch
abgeschaltet: Ohne echten Kabel-Sensor lässt sich "nicht angeschlossen"
nicht zuverlässig von "falsche Zuordnung" unterscheiden, und ein
Abschalten+Neuversuch bei jedem Prüfzyklus würde die Steckdose fürs
restliche Ladefenster dauerhaft an-/ausklicken. Stattdessen bleibt sie
einfach an, bis die normale Preis-/Deadline-Logik sie regulär abschaltet.
`socket_assignment.yaml` setzt den jeweiligen Verifikations-Helper (und den
"andere Steckdose bereits getestet"-Helfer) bei jeder neuen Zuordnung
(Ankunft oder Platztausch) zurück, damit auch ein Steckdosenwechsel wieder
frisch geprüft und ggf. erneut ein Alternativ-Versuch erlaubt wird.

Damit "zuhause + Ladebedarf, aber nicht angeschlossen" nicht erst kurz vor
einer knappen Deadline auffällt (wo die Aufhol-Logik ohnehin einen
Ladeversuch – und damit eine Verifikation – auslösen würde), gibt es
zusätzlich eine **Verbindungsprüfung kurz nach Ankunft**: Innerhalb eines
Zeitfensters (`connectivity_probe_window`, Default 10 min) nach Ankunft
zuhause wird bei echtem Ladebedarf und noch unverifizierter Zuordnung
unabhängig vom Strompreis kurz versucht einzuschalten, rein um die
Verbindung frühzeitig zu testen. Nach diesem Zeitfenster wird dafür nicht
mehr unabhängig vom Preis eingeschaltet (nur noch preis-/deadline-getrieben)
– sonst würde ein noch unverifiziertes Fahrzeug dauerhaft unabhängig vom
Strompreis laden, bis die Zuordnung manuell korrigiert wird. Über
`twizy_{1,2}_verbindungspruefung_aktiv` lässt sich diese Prüfung pro
Fahrzeug komplett ein-/ausschalten – bei "aus" kommt die Benachrichtigung
nur noch bei einem tatsächlichen preis-/deadline-getriebenen Ladeversuch
(kann dann auch erst kurz vor der Abfahrtszeit sein).

### Preisoptimiertes Laden & Abfahrtszeit
Die Abfahrtszeit ist **pro Wochentag einzeln einstellbar** (7 Helper je
Fahrzeug, `twizy_{1,2}_abfahrtszeit_montag` … `_sonntag`). Für jedes
Fahrzeug wird pro Prüfzyklus (Default alle 10 min) berechnet:
- die Deadline: die Abfahrtszeit des heutigen Wochentags – ist die schon
  verstrichen (oder liegt kein Ladebedarf vor, s. u.), wird bis zu eine
  Woche vorausgeschaut.
- benötigte Ladedauer: die von OVMS geschätzte Restzeit bis voll, mit
  [Korrekturfaktor](#selbstlernender-korrekturfaktor-f%C3%BCr-die-ovms-restzeitsch%C3%A4tzung).
- ob das aktuelle Zeitfenster zu den günstigsten Zeitfenstern vor der
  Deadline gehört, die in Summe die Ladedauer abdecken.
- eine **Aufhol-Logik**: reicht die verbleibende Zeit bis zur Abfahrt nicht
  mehr aus, um die benötigte Dauer allein aus günstigen Zeitfenstern zu
  decken, wird unabhängig vom Preis sofort weiter geladen, damit die
  Deadline nicht gerissen wird.

**Wann ist der nächste Ladestart?** Jeder Prüfzyklus schreibt eine lesbare
Kurzfassung in `twizy_{1,2}_naechster_ladestart` (im Dashboard als "Status"
zuoberst): `Lädt jetzt`, ein Zeitpunkt wie `Di 20.08. 03:00`, oder `Kein
Ladebedarf geplant`. Das ist der aktuelle Planungsstand – ändern sich
Preise, SoC oder die Zuordnung, wird der Wert beim nächsten Zyklus neu
berechnet, ist also keine feste Zusage.

**Tage ohne Ladebedarf:** Steht die Abfahrtszeit eines Tages auf **00:00
Uhr**, gilt das als "an diesem Tag kein Ladebedarf" – ohne dafür einen
eigenen zusätzlichen Helper zu brauchen. Der Lade-Scheduler überspringt
diesen Tag dann bei der Deadline-Suche und schaut bis zu eine Woche voraus,
bis er einen Tag mit einer Zeit ungleich 00:00 findet (die Ladeplanung
kann dabei durchaus schon an den "arbeitsfreien" Tagen dazwischen
stattfinden, wenn das günstiger ist). **Sind alle 7 Tage auf 00:00
gesetzt**, gibt es keine echte Deadline und damit auch keine Aufhol-Logik –
geladen wird dann trotzdem automatisch, aber ausschließlich zu den
günstigsten Zeitfenstern innerhalb der bereits bekannten Tibber-Preisdaten
(praktisch: heute + morgen, sobald bekannt). Die
[tägliche Mindest-Einschaltzeit](#mindest-einschaltzeit-pro-tag) funktioniert
unabhängig davon immer.

<details>
<summary>Warum kein <code>schedule</code>-Helper für die Abfahrtszeiten?</summary>

Ein `schedule`-Helper wäre naheliegend, hat hier aber einen Haken: Sein
`next_event`-Attribut zeigt außerhalb des aktuellen Zeitfensters nur den
*Beginn* des nächsten Fensters (z. B. Mitternacht), nicht die eigentliche
Abfahrtszeit des nächsten Tages. Der Lade-Scheduler braucht aber jederzeit
(auch nachts) die tatsächliche nächste Abfahrtszeit, um das Preisfenster
korrekt zu berechnen – dafür sind die 7 separaten `input_datetime`-Helper
direkter nutzbar.

</details>

### Selbstlernender Korrekturfaktor für die OVMS-Restzeitschätzung
Die von OVMS geschätzte Restzeit bis voll ist die Grundlage der Ladeplanung
(siehe oben), aber je nach Fahrzeug/Firmware oft ungenau. Der Zustandstext
der hier verwendeten OVMS-Sensoren ist nicht direkt als Zahl nutzbar (z. B.
`"3h 34min"`) – der Lade-Scheduler liest deshalb das `raw_value`-Attribut
des Sensors (unformatierter Minutenwert). Der Lade-Scheduler
gleicht das mit einem Korrekturfaktor aus
(`twizy_{1,2}_ladezeit_korrekturfaktor`, Start 1,0): Die tatsächlich
verwendete Ladedauer ist immer `OVMS-Schätzung × Korrekturfaktor`.

Der Faktor lernt aus vergangenen Ladungen: Beim Einschalten wird die
aktuelle OVMS-Schätzung gemerkt (`twizy_{1,2}_sitzung_start_etr_minuten`,
intern); erreicht der SoC beim Ausschalten mindestens die
Korrektur-SoC-Schwelle (Blueprint-Eingabe `correction_factor_soc_threshold`,
Default 95 %), wird das Verhältnis tatsächliche/geschätzte Dauer dieser
Sitzung berechnet (auf 0,3–6,0 begrenzt, um Ausreißer abzufedern) und der
Korrekturfaktor per gleitendem Mittelwert angepasst (70 % alter Wert, 30 %
neues Verhältnis). Die 95 %-Schwelle liegt bewusst unter der "voll"-Schwelle
(Default 97 %), weil die letzten Prozent oft per Erhaltungsladung sehr
langsam laufen und die Messung sonst verzerren würden. Sitzungen, die diese
Schwelle nicht erreichen (z. B. vorzeitig abgebrochen), fließen nicht in die
Anpassung ein. Der aktuelle Faktor ist im Dashboard unter "Einstellungen"
als Debug-Wert sichtbar, die daraus berechnete erwartete Gesamt-Ladedauer
(`twizy_{1,2}_erwartete_ladedauer_minuten`) steht direkt in der jeweiligen
"Status"-Karte.

### Ladeschluss-Erkennung
Ist der SoC-Schwellwert erreicht, **beginnt** die Abschaltung – tatsächlich
ausgeschaltet wird aber erst, wenn zusätzlich der gemessene Ladestrom der
zugeordneten Steckdose durchgehend für eine einstellbare Zeit
(`charge_complete_power_duration`, Default 1 Minute) unter einer
einstellbaren Schwelle bleibt (`charge_complete_power_threshold`, Default
10 W). Grund: Der SoC-Wert erreicht die "voll"-Schwelle oft schon, während
die Erhaltungsladung noch mit kleinem Strom weiterläuft – ohne diese
zusätzliche Bedingung würde dieser letzte Rest abgeschnitten. Die
Bestätigung wird pro Ladesitzung intern gemerkt
(`twizy_{1,2}_ladevorgang_fertig`) und bei jedem neuen Einschalten
zurückgesetzt. Wird die Steckdose aus einem anderen Grund abgeschaltet
(z. B. weil das geplante günstige Preisfenster endet, bevor das Fahrzeug
voll ist), gilt diese zusätzliche Bedingung nicht – nur das "eigentlich
fertig, SoC-Schwelle erreicht"-Abschalten wartet auf die Strom-Bestätigung.

Der aktuelle Ladestrom wird im Dashboard in der mittleren Spalte unter
"Ladestrom" nach Steckdose sortiert angezeigt (innen/außen, da der Strom
physisch an der Steckdose hängt, nicht am wechselnd zugeordneten
Fahrzeug), direkt aus den Leistungssensoren der Steckdosen. Der SoC steht
dagegen, weil er eine Eigenschaft des Fahrzeugs ist, direkt in der
jeweiligen "Twizy X – Status"-Karte, ebenfalls direkt aus dem
OVMS-SoC-Sensor. Beide referenzieren bewusst direkt die realen Sensoren
statt eines Pakethelfers, da es dabei rein um Anzeige geht – die
Ladeschluss-Erkennung selbst nutzt unabhängig vom Dashboard weiterhin die
im jeweiligen Lade-Scheduler konfigurierten
`innen_power_sensor`/`aussen_power_sensor`-Eingaben. Änderst du diese
Sensor-Entity-IDs, müssen die `entity:`-Zeilen unter "Ladestrom" und
"Status" in der Dashboard-Datei entsprechend angepasst werden.

### Mindest-Einschaltzeit pro Tag
Damit die Steckdose auch an Tagen, an denen das Fahrzeug schon voll ist
(z. B. weil es kaum gefahren wurde), nicht komplett stromlos bleibt, gibt
es eine einstellbare Mindest-Einschaltzeit
(`twizy_{1,2}_mindestlaufzeit_minuten`, Default 30 min, 0 deaktiviert die
Funktion): Sie wird zu den günstigsten verbleibenden Zeitfenstern trotzdem
nochmal für die fehlende Zeit eingeschaltet.

Dafür führt der Lade-Scheduler intern einen Tages-Zähler
(`twizy_{1,2}_einschaltzeit_minuten_heute` + `twizy_{1,2}_einschaltzeit_tag`),
der bei jedem Ausschalten um die Dauer der gerade beendeten Einschaltzeit
erhöht wird – unabhängig davon, ob das eine reguläre Ladesitzung oder ein
gezielter "Auffüll"-Vorgang war. Reicht die bisherige Einschaltzeit des
Tages schon aus (z. B. weil vorher regulär lange geladen wurde), wird keine
zusätzliche Zeit erzwungen. Der Zähler setzt sich beim nächsten Tageswechsel
automatisch zurück (erkannt am gespeicherten Datum).

### Manuelles Einschalten
Wird eine Steckdose von Hand eingeschaltet, wird das als manueller
Ladewunsch übernommen – erkannt daran, dass die Kontext-ID der
Zustandsänderung nicht mit der Kontext-ID übereinstimmt, die dieser
Lade-Scheduler beim letzten eigenen automatischen Einschalten in einem
internen Helfer hinterlegt hat
(`twizy_{1,2}_letzter_automatischer_einschaltkontext`). Erkannt werden so
gleichermaßen ein UI-Klick, App, Sprachassistent **und ein physischer
Tastendruck direkt an der Steckdose**: Es wird sofort weitergeladen, die
Abfahrtszeit bleibt
aber als spätester "voll"-Zeitpunkt gültig – die Automatisierung schaltet
weiterhin ab, sobald der SoC-Schwellwert erreicht ist (siehe
[Ladeschluss-Erkennung](#ladeschluss-erkennung)) **oder das Fahrzeug
wegfährt** (anders als bei der Preis-/Überlast-Logik übersteuert "manuell"
den Standort-Check bewusst nicht – fährt das Fahrzeug weg, gibt es nichts
mehr zu laden, egal wie die Ladung gestartet wurde). Für die bewusste
Nutzung einer Steckdose durch ein anderes Gerät, unabhängig vom
Fahrzeugstandort, gibt es stattdessen die separate
[Steckdose pausieren](#steckdose-pausieren-z-b-f%C3%BCr-rasenm%C3%A4her)-Funktion.

### Steckdose pausieren (z. B. für Rasenmäher)
Über `input_boolean.twizy_socket_aussen_pause` lässt sich die Steckdose
"außen" komplett aus der Ladesteuerung herausnehmen, um sie z. B. für ein
anderes Gerät (Rasenmäher, Bohrmaschine, ...) zu nutzen. Wird der Schalter
eingeschaltet, schaltet der Blueprint "Twizy: Steckdose pausieren" die
Steckdose sofort mit ein. Solange die Pause aktiv ist, überspringt der
zuständige Lade-Scheduler diese Steckdose vollständig – kein automatisches
Ein-/Ausschalten, keine Verifikation, egal welches Fahrzeug ihr laut
Zuordnung gerade zugewiesen ist. Danach bleibt Ein-/Ausschalten vollständig
manuell; die Überlast-Vermeidung der jeweils anderen Steckdose bleibt
trotzdem korrekt, da sie weiterhin den echten Leistungssensor ausliest,
unabhängig davon, was tatsächlich angeschlossen ist.

Die Pause endet automatisch von selbst: Der Blueprint "Twizy: Steckdose
pausieren" (eigene, separate Automatisierung, 1× für die Steckdose "außen"
anzulegen) beobachtet den Leistungssensor der Steckdose und schaltet den
Pause-Schalter wieder aus, sobald der Stromverbrauch für eine einstellbare
Dauer (Default 30 Minuten) durchgehend unter einer einstellbaren Schwelle
(Default 10 W) bleibt – die Steckdose geht dann von selbst wieder in den
Normalbetrieb über, ohne dass man daran denken muss, die Pause manuell zu
beenden.

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
bewusst (siehe [Manuelles Einschalten](#manuelles-einschalten)).

Kommt es trotzdem zu einer Überlast (z. B. weil beide Steckdosen manuell
gleichzeitig eingeschaltet wurden oder die tatsächliche Ladeleistung höher
als geschätzt war), greift der Überlastschutz der Steckdose selbst. Sobald
sie – laut Home Assistant am Zustandswechsel `unavailable` → `off` erkennbar
– wieder verfügbar ist, bewertet der jeweilige Lade-Scheduler sofort neu, ob
wieder eingeschaltet werden soll.

## Bekannte Vereinfachungen

- Die Überlast-Vermeidung beim Einschalten arbeitet mit einem geschätzten
  Wert für die eigene typische Ladeleistung (nicht mit einer live
  gemessenen eigenen Leistung, die vor dem Einschalten ja noch bei ~0 W
  liegt). Bei knappen Fällen lieber etwas großzügiger schätzen.
- Wenn beide Steckdosen gleichzeitig manuell eingeschaltet werden, greift
  keine Vorab-Prüfung (Manuell übersteuert die Überlast-Vermeidung bewusst)
  – hier verlässt sich die Lösung vollständig auf den Überlastschutz der
  Steckdosen selbst.
- Die "manuell eingeschaltet"-Erkennung merkt sich die Kontext-ID jedes
  eigenen automatischen Einschaltens in einem internen Helfer und
  vergleicht sie beim nächsten Einschalt-Ereignis - stimmt sie nicht
  überein, gilt das Einschalten als manuell. Erkennt damit zuverlässig
  auch einen physischen Tastendruck direkt an der Steckdose. (Ein früherer
  Ansatz ohne diesen Helfer, der stattdessen prüfte, ob der Kontext zu
  irgendeiner Automatisierung passt, war bei den vielen sich teils
  überschneidenden Triggern dieses Blueprints nicht zuverlässig genug.)
- Die Verifikation greift auch bei manuell gestarteten Ladungen (bewusst
  so, damit eine falsche Zuordnung immer auffällt) – meldet nach der
  Toleranzzeit weder OVMS noch der Ladestrom etwas, kommt dadurch auch bei
  einer manuell eingeschalteten Steckdose die Verifikations-Benachrichtigung
  (die Steckdose bleibt aber an, siehe oben).
- Die Verifikation (und die Verbindungsprüfung nach Ankunft) kann ohne
  echten Kabel-Sensor nicht zuverlässig zwischen "nicht angeschlossen" und
  "falsche Zuordnung" unterscheiden. Der automatische Alternativ-Versuch an
  der jeweils anderen Steckdose (siehe
  [Verifikation & Verbindungsprüfung](#verifikation-zu-ladebeginn--verbindungspr%C3%BCfung))
  klärt eine schlicht vertauschte Zuordnung von selbst auf, hilft aber
  nicht, wenn das Fahrzeug wirklich an keiner der beiden Steckdosen hängt
  oder ein drittes, unbekanntes Gerät angeschlossen ist – die
  Benachrichtigung nennt dann weiterhin beide Ursachen als möglich, und die
  Steckdose bleibt einfach an, bis sie manuell korrigiert oder das
  Ladefenster regulär beendet wird.

## Repository-Struktur

```
packages/
  twizy_charging.yaml            # alle Helper-Entities
blueprints/automation/twizy/
  socket_assignment.yaml         # innen/außen-Zuordnung
  charge_scheduler.yaml          # Lade-Entscheidung je Fahrzeug (2x instanziieren),
                                  # inkl. Überlast-Vermeidung und Ladeschluss-Erkennung
  socket_pause.yaml              # Steckdose "außen" pausieren (optional, 1x)
dashboards/
  twizy_dashboard.yaml           # fertiges Dashboard (nur eingebaute Karten)
```
