# Notfall-Anleitung: Ecodan Rollback auf Originalzustand

> **Stand:** April 2026 — basierend auf WP-Steuerung v2.5 "The Rock"
> **System:** Mitsubishi Ecodan PUD-SHWM140YAA / EHSD-YM9D / FTC6
> **Steuerung:** Raspberry Pi 5 + Home Assistant + Shelly 1 Mini Gen4 + Procon A1M Modbus

---

## Übersicht: Was wurde am Original verändert?

### Am Raspberry Pi / Home Assistant
- `configuration.yaml` — Template-Sensoren (Ladewächter, COP, Leistung)
- `automations.yaml` v2.5 — Boost, Thermostat, Watchdog, Nachtabsenkung
- `packages/wp_automationen_v2.yaml` — Modbus-Verbindung (mitsubishi-a1m, /dev/ttyUSB0)

### Am Shelly 1 Mini Gen4 (IP: 192.168.178.106)
- Eingebaut im WP-Innengerät, Relaisausgang an TBI.1 (IN1)
- Auto-Off Timer: 30 Minuten konfiguriert
- Entity: `switch.wp_thermostat_in_1`

### An der FTC6 (per SD-Karte geändert)
- **Item 138** (Ext. Thermostat AT Grenze unten): von -5°C auf **-15°C** geändert
- **Item 139** (Ext. Thermostat AT Grenze oben): von +5°C auf **+35°C** geändert
- **DIP-Switch SW2-1:** auf **ON** gesetzt (Thermostat-Eingang IN1 aktiv)

### Am Procon MelcoBEMS MINI A1M
- Angeschlossen an FTC6 über CN105
- RS-485 Verbindung über Waveshare USB-Adapter an Raspberry Pi (/dev/ttyUSB0)
- Slave-Adresse: 1, Baudrate: 9600, Parität: N, Stoppbits: 1

---

## Stufe 1: Software-Deaktivierung (Soft-Rollback)

> **Wann nutzen?** Wenn du temporär Home Assistant ignorieren und die Anlage am Display übersteuern willst, ohne Kabel abzuklemmen.

### Schritt 1: Automationen stoppen
Deaktiviere in Home Assistant unter **Einstellungen → Automationen** alle Automationen die mit "WP" beginnen. Besonders wichtig:
- `WP Thermostat & Modbus - Watchdog Heartbeat` (sendet alle 10 Min Modbus-Werte)
- `WP Thermostat - Ladung AN`
- `WP Thermostat - Ladung AUS`
- `WP Boost bei PV-Überschuss aktivieren`
- `WP Boost beenden`

### Schritt 2: Modbus-Timeout abwarten (30 Minuten)
Die FTC6 hat einen integrierten Server-Timeout (Item 128 = 30 Minuten). Wenn der Watchdog keine Befehle mehr sendet, vergisst die FTC6 nach 30 Minuten alle Modbus-Sollwerte (40°C, 50°C, 60°C) und verwendet wieder die Werte vom Display.

### Schritt 3: Shelly dauerhaft auf ON stellen
In der Shelly-App oder in Home Assistant den Schalter `switch.wp_thermostat_in_1` dauerhaft auf **ON** stellen. Damit ist der Thermostat-Kontakt dauerhaft geschlossen und die WP hat bedingungslose Heizfreigabe.

> **Wichtig:** Den Shelly Auto-Off Timer in der Shelly-App vorher auf **0** (deaktiviert) stellen! Sonst schaltet der Shelly nach 30 Minuten selbstständig ab und die WP geht in den Stopp.

### Ergebnis Stufe 1
Die WP arbeitet wieder rein nach FTC6-Display-Einstellungen. Die geänderten AT-Grenzen (-15/+35°C) bleiben aktiv — das schadet nicht, im Gegenteil.

---

## Stufe 2: Hardware-Rückbau (Hard-Rollback)

> **Wann nutzen?** Wenn der Raspberry Pi tot ist oder der Shelly defekt ist und die Wärmepumpe im Winter sofort wieder normal arbeiten muss.

### Schritt 1: Heizfreigabe sicherstellen (Shelly von IN1 trennen)

Der Shelly fungiert als "Türsteher" — wenn er offline ist und auf OFF steht, bekommt die WP kein Heizsignal!

**Lösung A — DIP-Switch (empfohlen, kein Werkzeug nötig):**
Am FTC6 Controller den **DIP-Switch SW2-1 auf OFF** stellen. Damit ignoriert die FTC6 den Thermostat-Eingang IN1 komplett. Die WP heizt dann wettergeführt nach Heizkurve.

**Lösung B — Kabelbrücke:**
Die Drähte vom Shelly-Relais (O/I Klemmen) am TBI.1 Stecker der FTC6 abklemmen und stattdessen eine **Drahtbrücke** zwischen den beiden IN1-Klemmen einsetzen. Damit hat die WP dauerhaft Heizfreigabe.

**Lösung C — FTC6-Menü:**
Service-Menü → Einstellungen ext. Eingänge → Außenthermostat → Deaktivieren.

### Schritt 2: Modbus-Verbindung trennen

Das Procon A1M Gateway ist harmlos wenn es keine Befehle bekommt, aber um es komplett stillzulegen:

- **Option 1:** USB-Stecker (Waveshare) vom Raspberry Pi abziehen
- **Option 2:** Am Procon A1M die Kabel **A+** und **B-** aus dem grünen RS-485 Stecker lösen
- **Option 3:** DIP-Schalter am Procon A1M auf OFF stellen

> Der A1M bleibt weiterhin über CN105 an der FTC6 — er liest nur passiv mit und stört nicht. Man kann ihn auch drin lassen.

### Schritt 3: FTC6-Werte auf Original zurücksetzen

Falls gewünscht, die per SD-Karte geänderten Werte zurücksetzen:

| Parameter | Geändert auf | Original-Wert |
|-----------|-------------|---------------|
| Item 138 (Ext. Thermostat AT unten) | -15°C | **-5°C** |
| Item 139 (Ext. Thermostat AT oben) | +35°C | **+5°C** |

Dies ist **nur nötig, wenn SW2-1 auf ON bleibt**. Wenn SW2-1 auf OFF gestellt wird, sind die AT-Grenzen irrelevant.

Weitere FTC6-Werte die wir **nicht** geändert haben und die auf Original stehen:
- Item 2 (TWW Soll): 50°C
- Item 3 (TWW Hysterese): 10°C
- Item 4 (TWW Max): 60°C
- Item 85 (TWW Vorrang): Ein
- Item 86 (TWW Vorrang Dauer): 60 Min
- Item 90 (TWW Ladungen/Tag): 4
- Item 128 (Server Timeout): 30 Min

### Schritt 4: Display-Werte kontrollieren

Prüfe am FTC6-Display:
- **Warmwasser Solltemperatur:** 50°C (Normalmenü, nicht Service)
- **Betriebsmodus:** Heizen (nicht Stopp)
- **Heizkurve:** Zone 1 aktiv, wettergeführt
- **Legionellenschutz:** Aktiv, 65°C, Donnerstag 03:00

---

## Stufe 3: Komplett-Reset (Werkseinstellung)

> **Nur im absoluten Notfall!** Setzt ALLE Einstellungen zurück — auch Heizkurve, Timer, Pumpeneinstellungen.

Am FTC6: Service-Menü (Passwort 0000) → Seite 4/4 → **Werkseinstellung** → Bestätigen.

Danach muss ein Installateur alle Werte neu konfigurieren (Heizkurve, Zonen, Pumpe, etc.).

---

## Schnellreferenz: Notfall im Winter

Wenn es kalt ist und die Heizung nicht läuft:

1. **Sofort:** Shelly-App öffnen → Shelly 1 Mini Gen4 → auf **ON** schalten
2. **Wenn Shelly nicht erreichbar:** Am FTC6 → DIP-Switch **SW2-1 auf OFF**
3. **Prüfen:** Am Display steht Heizbetrieb mit Play-Symbol?
4. **30 Minuten warten:** Modbus-Timeout läuft ab, FTC6 übernimmt
5. **Am nächsten Tag in Ruhe:** Ursache suchen (Raspi, WLAN, Shelly)

> Die Ecodan hat einen eigenen Frostschutz. Selbst wenn alles ausfällt, friert nichts ein — die WP schützt sich selbst bei Temperaturen unter 3°C.
