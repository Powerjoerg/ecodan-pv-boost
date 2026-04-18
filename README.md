# Mitsubishi Ecodan PV-Boost — Home Assistant Integration

Automatische PV-Überschuss-Steuerung für Mitsubishi Ecodan Wärmepumpen mit Home Assistant, Modbus (Procon A1M) und Shelly Pro 3EM.

> **Ziel:** Wenn genug Solarstrom vorhanden ist und die Batterie ausreichend geladen ist, wird die Warmwasser-Solltemperatur automatisch von 50°C auf 60°C angehoben. Dadurch wird überschüssiger PV-Strom als Wärme gespeichert, die Taktung reduziert und die Eigenverbrauchsquote maximiert.

---

## Inhaltsverzeichnis

- [Systemübersicht](#systemübersicht)
- [Hardware](#hardware)
- [Voraussetzungen](#voraussetzungen)
- [Installation](#installation)
  - [1. Modbus-Verbindung](#1-modbus-verbindung)
  - [2. Home Assistant Konfiguration](#2-home-assistant-konfiguration)
  - [3. Helfer anlegen](#3-helfer-anlegen)
  - [4. Automationen einrichten](#4-automationen-einrichten)
  - [5. Dashboard einrichten](#5-dashboard-einrichten)
- [DIP-Switch Einstellungen](#dip-switch-einstellungen)
- [Modbus Register-Map](#modbus-register-map)
- [PV-Boost Logik](#pv-boost-logik)
- [Taktungsbewertung](#taktungsbewertung)
- [COP-Berechnung](#cop-berechnung)
- [Fehlerbehebung](#fehlerbehebung)
- [Danksagung](#danksagung)
- [Lizenz](#lizenz)

---

## Systemübersicht

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   PV-Anlage     │────▶│ FoxESS H3-Pro    │────▶│   Batterie      │
│   14,5 kWp      │     │ Wechselrichter   │     │ FoxESS ESC 2900 │
└─────────────────┘     └──────┬───────────┘     │ 17,28 kWh       │
                               │                  └─────────────────┘
                               │ PV-Daten via HA
                               ▼
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│ Mitsubishi       │     │  Home Assistant   │     │ Shelly Pro 3EM  │
│ Ecodan WP        │◀──▶│  Raspberry Pi 5   │◀───│ WP-Verbrauch    │
│ PUD-SHWM140YAA  │     │                  │     │ CT1: Außen (×3) │
│ EHSD-YM9D       │     │  PV-Boost        │     │ CT2: Heizstab(×3│)
│ FTC6 Controller  │     │  Automationen    │     │ CT3: Innen (×1) │
└────────┬─────────┘     └───────┬──┬───────┘     └─────────────────┘
         │                       │  │
         │ Modbus RTU            │  │ WLAN
         │ RS-485                │  │
         ▼                       │  ▼
┌─────────────────┐     ┌────────┴─────────┐     ┌─────────────────┐
│ Procon A1M      │     │ Waveshare        │     │ Shelly 1 Mini   │
│ MelcoBEMS MINI  │◀───▶│ USB to RS-485    │     │ Thermostat IN1  │
│ Modbus Adapter  │     │ /dev/ttyUSB0     │     │ (Potentialfrei) │
└─────────────────┘     └──────────────────┘     └─────────────────┘
```

---

## Hardware

| Komponente | Modell | Beschreibung |
|---|---|---|
| **Außengerät** | Mitsubishi PUD-SHWM140YAA | 14 kW Luft/Wasser-Wärmepumpe |
| **Innengerät** | Mitsubishi EHSD-YM9D | Hydromodul mit FTC6 Controller |
| **Modbus-Adapter** | Procon MelcoBEMS MINI (A1M) | Modbus RTU über RS-485 (Serial) |
| **USB-Adapter** | Waveshare USB to RS-485 | Anschluss an Home Assistant |
| **WP-Verbrauchsmessung** | Shelly Pro 3EM (SPEM-003CEBEU) | 3× CT-Klemmen im Sicherungskasten |
| **Thermostat-Steuerung** | Shelly 1 Mini Gen3 | Potentialfreier Kontakt an IN1 (Zone 1) zur Heizungssteuerung |
| **Wechselrichter** | FoxESS H3-Pro-15.0 | Hybrid-Wechselrichter |
| **Batterie** | FoxESS ESC 2900-2 | 17,28 kWh Speicherkapazität |
| **PV-Anlage** | — | 14,5 kWp |
| **Home Assistant** | Raspberry Pi 5 | rpi5-64, HA OS |

### Shelly Pro 3EM — CT-Klemmen Belegung

Der Shelly Pro 3EM sitzt im Sicherungskasten und misst den WP-Verbrauch über drei CT-Klemmen:

| Kanal | Messung | Berechnung |
|---|---|---|
| Phase A (CT1) | Außeneinheit (1 von 3 Phasen) | × 3 = Gesamtleistung Außen |
| Phase B (CT2) | Heizstab (1 von 3 Phasen) | × 3 = Gesamtleistung Heizstab |
| Phase C (CT3) | Innenmodul (1-phasig) | × 1 = direkt |

> **Hinweis:** Die Außeneinheit und der Heizstab sind 3-phasig angeschlossen. Da nur eine Phase gemessen wird, wird der Messwert ×3 genommen. Der Fehler beträgt ca. 10-15%, was für die Automation ausreichend genau ist.

### Shelly 1 Mini Gen3 — Thermostat-Kontakt (IN1)

Der Shelly 1 Mini Gen3 wird an der Wärmepumpen-Platine (FTC6) als Raumthermostat angeschlossen, um die Heizkreisfreigabe zu steuern. Er agiert als "Schalter", der den Heizbetrieb basierend auf der Logik in Home Assistant zulässt oder blockiert.

**1. Anschlüsse am Shelly:**
- **L / N:** 230V Spannungsversorgung (am besten vom gleichen Sicherungsautomat wie die WP-Steuerung).
- **O / I:** Potentialfreier Schließer-Kontakt. 

**2. Anschluss an der FTC6 (Steckerbrett TBI.1):**
- Die Klemmen **O** und **I** des Shellys werden mit den Klemmen **1** und **2** am Anschluss **IN1** (auf dem Klemmblock TBI.1) der Wärmepumpe verbunden.
- **Funktion:** Wenn der Shelly einschaltet (Relais geschlossen), erkennt die FTC6 einen geschlossenen Kontakt. Dadurch wird der Raumheiz-Betrieb freigegeben.

> **Extrem wichtig:** Hier darf **nur** ein potentialfreier Kontakt (wie der Shelly 1 Mini Gen3 *ohne* PM) verwendet werden. Schließe **keine 230V** an den IN1-Anschluss an, sonst wird das Mainboard der Wärmepumpe zerstört!

---

## Voraussetzungen

- Home Assistant (2024.x oder neuer)
- HACS installiert (für Mushroom Cards)
- [Mushroom Cards](https://github.com/piitaya/lovelace-mushroom) installiert
- Procon A1M korrekt verkabelt (RS-485 → Waveshare → USB → Raspberry Pi)
- Shelly Pro 3EM in HA integriert (für Verbrauchsmessung)
- Shelly 1 Mini Gen3 in HA integriert (für Heizkreisfreigabe)
- FoxESS Integration in HA (für PV-Daten und Batterie-SOC)
- Folgende HA-Sensoren müssen vorhanden sein:
  - `sensor.pv_power` — aktuelle PV-Leistung in **kW**
  - `sensor.battery_soc_1` — Batterie-Ladezustand in **%**
  - `sensor.solar_energy_today` — PV-Erzeugung heute in **kWh**
  - `sensor.feed_in_energy_today` — Einspeisung heute in **kWh**
- Folgende HA-Zähler (Helfer) müssen unter Settings → Geräte & Dienste → Helfer angelegt werden:
  - `counter.wp_starts_heute`
  - `counter.wp_kurzzyklen_heute`
  - `counter.wp_abtauzyklen_heute` (Neu!)
- Folgende HA-Entität (Schalter) muss zwingend übereinstimmen:
  - `switch.wp_thermostat_in_1` — Dies muss die genaue Entitäts-ID des **Shelly 1 Mini Gen3** sein. Bitte in HA entsprechend umbenennen!

---

## Installation

### 1. Modbus-Verbindung

#### Procon A1M — DIP-Switch Einstellungen

Am Procon A1M folgende DIP-Switches setzen:
- **DIP 1:** ON
- **DIP 6:** ON
- **DIP 7:** ON

#### Verkabelung

```
Procon A1M (RS-485) ──── Waveshare USB-to-RS485 ──── Raspberry Pi USB
    A+ ──────────────────── A+
    B- ──────────────────── B-
    GND ─────────────────── GND (optional)
```

Der Waveshare Adapter erscheint als `/dev/ttyUSB0` auf dem Raspberry Pi.

#### Modbus-Package

Dieses Projekt basiert auf dem [helgeklein Mitsubishi A1M Package](https://github.com/helgeklein/mitsubishi-heat-pump-modbus-home-assistant), angepasst für die **serielle Verbindung** (A1M statt A1M+).

Kopiere die Datei `packages/mitsubishi_a1m.yaml` in deinen HA-Ordner `config/packages/`.

Falls der `packages`-Ordner noch nicht existiert, füge in der `configuration.yaml` hinzu:

```yaml
homeassistant:
  packages: !include_dir_named packages/
```

**Wichtig:** In der `mitsubishi_a1m.yaml` die Modbus-Verbindung anpassen:

```yaml
modbus:
  - name: mitsubishi-a1m
    type: serial
    port: /dev/ttyUSB0
    baudrate: 9600
    bytesize: 8
    parity: N
    stopbits: 1
    method: rtu
    timeout: 2
```

---

### 2. Home Assistant Konfiguration

Füge in der `configuration.yaml` folgende Blöcke hinzu:

#### Automationen einbinden

```yaml
automation: !include automations.yaml
```

#### Template-Sensoren

Kopiere den Inhalt von `configuration_templates.yaml` aus diesem Repo in deine `configuration.yaml`.

Die Template-Sensoren umfassen:

| Sensor | Beschreibung |
|---|---|
| `sensor.wp_aussen_gesamt` | Außeneinheit Leistung (Phase A × 3) |
| `sensor.wp_heizstab_gesamt` | Heizstab Leistung (Phase B × 3) |
| `sensor.wp_innen_gesamt` | Innenmodul Leistung (Phase C × 1) |
| `sensor.wp_gesamtverbrauch` | Gesamter elektrischer WP-Verbrauch in W |
| `sensor.wp_waermeleistung` | Erzeugte Wärmeleistung aus Durchfluss und Temperaturdifferenz |
| `sensor.wp_cop` | COP Momentanwert (behält letzten Wert bei WP-Stopp) |
| `sensor.wp_cop_heute` | COP Tageswert |
| `sensor.wp_laufzeit_pro_zyklus` | Durchschnittliche Laufzeit pro Zyklus in Minuten |
| `sensor.wp_taktungsbewertung` | Gesamtbewertung der Taktung (Optimal/Normal/Beobachten/Kritisch) |
| `binary_sensor.wp_laeuft` | WP läuft (Frequenz > 5 Hz) |
| `sensor.wp_laufzeit_heute` | Gesamte Laufzeit heute in Stunden (history_stats) |

> **Wichtig:** Die Entity-IDs der Shelly-Sensoren müssen an dein Gerät angepasst werden! Suche in den Templates nach `shellypro3em_xxxxxxxxxxxx` und ersetze die MAC-Adresse durch deine eigene.

---

### 3. Helfer anlegen

Folgende Helfer müssen in Home Assistant über die UI angelegt werden:

**Einstellungen → Helfer → + Helfer erstellen**

#### Schalter (Toggle)

| Name | Entity-ID | Beschreibung |
|---|---|---|
| WP Boost aktiv | `input_boolean.wp_boost_aktiv` | Zeigt an ob der PV-Boost aktiv ist |

#### Nummer (Number)

| Name | Entity-ID | Min | Max | Schritt | Einheit |
|---|---|---|---|---|---|
| WP Soll Temperatur | `input_number.wp_soll_temperatur` | 40 | 60 | 1 | °C |

#### Zähler (Counter)

| Name | Entity-ID | Anfangswert | Minimum | Schritt |
|---|---|---|---|---|
| WP Starts Heute | `counter.wp_starts_heute` | 0 | 0 | 1 |

> **Wichtig:** Beim Counter darauf achten, dass **Maximum** nicht auf 0 gesetzt wird, sonst kann der Counter nie hochzählen!

#### Integration — Riemann-Summenintegral

| Name | Eingangssensor | Methode | Genauigkeit | Einheit | Zeiteinheit |
|---|---|---|---|---|---|
| WP Verbrauch kWh | `sensor.wp_gesamtverbrauch` | Links | 2 | k (Kilo) | Stunden |
| WP Waerme kWh | `sensor.wp_waermeleistung` | Links | 2 | k (Kilo) | Stunden |

#### Verbrauchszähler (Utility Meter)

| Name | Quelle | Zyklus |
|---|---|---|
| WP Verbrauch Heute | `sensor.wp_verbrauch_kwh` | Täglich |
| WP Verbrauch Monat | `sensor.wp_verbrauch_kwh` | Monatlich |
| WP Verbrauch Jahr | `sensor.wp_verbrauch_kwh` | Jährlich |
| WP Waerme Heute | `sensor.wp_waerme_kwh` | Täglich |

---

### 4. Automationen einrichten

Kopiere den Inhalt von `automations.yaml` aus diesem Repo in deine HA-Datei `config/automations.yaml`.

#### Übersicht der Automationen

| Nr | Name | Beschreibung |
|---|---|---|
| 1 | WP Taktung - Start zählen | Zählt WP-Starts (Frequenz > 5 Hz für 1 Min) |
| 2 | WP Taktung - Reset Mitternacht | Setzt Start- und Abtau-Counter täglich auf 0 |
| 3 | WP Taktung - Abtauzyklus zählen & Filtern | Erkennt Abtauung, filtert den Takt heraus und erhöht Abtau-Zähler |
| 4 | **WP Boost aktivieren** | Soll auf 60°C wenn PV > 1 kW, Batterie > 70%, TWW < 48°C |
| 5 | **WP Boost beenden** | Zurück auf 50°C wenn PV < 0.8 kW, Batterie < 65% oder 21:00 Uhr |
| 5 | **WP Boost Ziel erreicht** | Zurück auf 50°C wenn Speicher 59°C erreicht |
| 6 | WP Soll manuell setzen | Schreibt Soll-Temperatur wenn Slider im Dashboard geändert wird |

---

### 5. Dashboard einrichten

Kopiere den Inhalt von `dashboards/wp_dashboard.yaml` als manuelle YAML-Karte in dein Dashboard.

**Dashboard → Bearbeiten → Karte hinzufügen → Manuell (YAML)**

Das Dashboard zeigt:
- WP-Status (Läuft/Aus) mit Verdichterfrequenz
- Speicher-, Soll- und Außentemperatur
- Batterie-SOC, PV-Erzeugung, Einspeisung, PV-Leistung
- Verdichter Hz, Vorlauf, Betriebsart, WP-Leistung
- Soll-Temperatur Slider (+/− Tasten)
- COP Momentan und COP Heute
- WP-Verbrauch Heute/Monat/Jahr
- Taktung (Starts + Ø Laufzeit), Bewertung, PV-Boost Status

---

## Wärmepumpen-Konfiguration (FTC6 Touch-Display)

Um das gefürchtete Takten (kurze Start-Stopp-Zyklen) der Wärmepumpe zu minimieren, ist es entscheidend, nicht nur die Home Assistant Automationen anzupassen, sondern auch die **physikalischen Einstellungen im Fachhandwerkermenü** des FTC6-Controllers zu optimieren. 

Folgende Parameter haben wir als optimal ermittelt:

### 1. TWW-Vorrang (Warmwasser-Priorität / Ladezeit)
Stelle sicher, dass die WP nicht ständig zwischen Heizen und Warmwasser hin- und herspringt.
* **Max. TWW-Ladezeit:** z. B. `60 Min`
* **TWW-Sperrzeit (Dauer bis zum nächsten möglichen TWW-Start):** z. B. `30 Min`
* *Hintergrund:* So wird garantiert, dass das Haus nach einer Warmwasser-Ladung (wie unserem PV-Boost) auch wieder Zeit hat, Wärme für den Heizkreis zu produzieren, bevor direkt wieder TWW nachgeladen wird.

### 2. Hysterese-Einstellungen (Einschaltdifferenz)
Die Hysterese bestimmt, wie weit die Temperatur fallen darf, bevor die WP wieder anspringt.
* **TWW Hysterese (Einschaltdifferenz):** Auf `-3 K` bis `-5 K` erhöhen (im Vergleich zum oft engen Werksstandard).
* *Hintergrund:* Nach dem PV-Boost auf 60°C kühlt der Speicher so in Ruhe ab, bevor die WP wegen "Warmwasserbedarf" wieder anspringt. Das bringt enorm lange Betriebs- und Pausenzeiten.

### 3. Sperrzeiten (Kompressor-Verzögerung)
* **Verdichter-Stillstandszeit (Mindestpause):** Auf z. B. `10 Min` bis `20 Min` erhöhen.
* *Hintergrund:* Ein sofortiges Wiederanspringen nach dem Abschalten (z. B. durch kleine Temperatur-Schwankungen) wird physikalisch von der FTC6-Steuerung blockiert. Das schont den Verdichter enorm.

### 4. Einsatzgrenzen (AT-Bivalenzpunkt / Sommerabschaltung)
* **Umschaltung Sommerbetrieb:** Zwingend auf `Außentemperatur` (o.ä.) stellen, **nicht** auf `Keine`! Steht der Wert auf `Keine`, ist die Sommerabschaltung komplett deaktiviert und die wp-interne Heizkurve versucht das ganze Jahr stur zu heizen.
* **Außentemperatur Heizen EIN:** z. B. `11°C` bis `14°C`.
* **Außentemperatur Heizen AUS:** z. B. `13°C` bis `15°C`.
* **Dämpfungszeit Außentemp.:** auf z.B. `6h` (Standard) belassen. Verhindert, dass die WP bei kurzen sonnigen Abschnitten sofort springt, sondern immer den gleitenden Durchschnitt nutzt.
* *Hintergrund:* Bei milden Außentemperaturen im Frühling/Herbst (z.B. 19°C) und plötzlicher Sonneneinstrahlung schließen intelligente Einzelraumregler (ERR) oft die Kreise. Ist die Sommerabschaltung an der FTC6 deaktiviert (`Keine`), pumpt die WP autonom voll gegen diese geschlossenen Kreise. Die Folge: Spreizung 0.0°C, 2-Minuten-Kurzzyklus (Hydraulischer Kurzschluss) und extremer Verschleiß! Die korrekte Anpassung der Sommerabschaltung ist der wichtigste Hebel gegen das Takten im milden Frühjahr.

> **Achtung:** Die ganz exakten Werte können je nach Gebäudedämmung und Flächenheizung leicht variieren. Diese Optionen (Vorrang, Hysterese, Sperrzeiten, Sommerabschaltung) haben sich aber als die entscheidenden Hebel im FTC6-Menü erwiesen, um die Start-Zyklen der Ecodan signifikant in den Griff zu bekommen!

---

## DIP-Switch Einstellungen

### FTC6 Controller

#### SW1

| Schalter | Stellung | Funktion |
|---|---|---|
| SW1-1 | **ON** | Monoblock-Wärmepumpe |
| SW1-2 | OFF | — |
| SW1-3 | **ON** | Warmwasserbetrieb aktiv |
| SW1-4 | OFF | Elektroheizstab nicht über FTC gesteuert |
| SW1-5 | OFF | — |
| SW1-6 | **ON** | Modbus-Kommunikation (A1M) |
| SW1-7 | **ON** | Modbus-Kommunikation (A1M) |
| SW1-8 | OFF | Keine Mitsubishi Funk-Fernbedienung |

#### SW2

| Schalter | Stellung | Funktion |
|---|---|---|
| SW2-1 | **ON** | Externer Thermostat an IN1 (Zone 1) — **Zwingend erforderlich für Shelly Steuerung!** |
| SW2-2 bis SW2-7 | OFF | — |
| SW2-8 | **ON** | Strömungswächter aktiv |

> **Wichtig:** Da in diesem Konstrukt der **Shelly 1 Mini Gen3** zwingend erforderlich ist, MUSS `SW2-1` auf **ON** stehen! (Hintergrund: Ohne den angeschlossenen Shelly würde die FTC den "offenen Kontakt" an IN1 als "kein Heizbedarf" interpretieren und die Heizung würde dauerhaft stoppen).

#### Procon A1M

| DIP | Stellung | Funktion |
|---|---|---|
| DIP 1 | **ON** | — |
| DIP 6 | **ON** | — |
| DIP 7 | **ON** | — |

---

## Modbus Register-Map

### Verbindungsparameter

| Parameter | Wert |
|---|---|
| Schnittstelle | Serial (RS-485) |
| Port | `/dev/ttyUSB0` |
| Baudrate | 9600 |
| Bytesize | 8 |
| Parity | N (None) |
| Stopbits | 1 |
| Methode | RTU |
| Slave-Adresse | 1 |

### Wichtige Register (Holding Register)

| Register | Beschreibung | Skalierung | Lesen | Schreiben | Beispiel |
|---|---|---|---|---|---|
| **30** | **TWW-Solltemperatur** | **× 0.01** (lesen) / **× 100** (schreiben) | ✅ | ✅ | 5000 = 50°C, 6000 = 60°C |
| 31 | TWW-Isttemperatur | × 0.1 | ✅ | ❌ | 500 = 50.0°C |

> **Achtung:** Das helgeklein-Package verwendet Register 53 für die TWW-Solltemperatur. Bei der seriellen A1M-Variante funktioniert **Register 30** mit Skalierung **× 100** beim Schreiben. Register 53 reagiert nicht auf Schreibbefehle.

> **Verzögerung:** Nach einem Schreibbefehl dauert es ca. **1-2 Minuten**, bis der neue Wert von der FTC übernommen und im Modbus-Sensor sichtbar wird.

### Alle Sensoren aus dem helgeklein-Package

Eine vollständige Liste aller verfügbaren Modbus-Sensoren findest du in der Datei `packages/mitsubishi_a1m.yaml`. Die wichtigsten:

| Sensor | Beschreibung |
|---|---|
| `sensor.mitsubishi_wp_warmepumpe_frequenz` | Verdichterfrequenz in Hz |
| `sensor.mitsubishi_wp_isttemperatur_wasserspeicher_tww` | Speichertemperatur TWW |
| `sensor.mitsubishi_wp_solltemperatur_wasserspeicher_tww` | Solltemperatur TWW |
| `sensor.mitsubishi_wp_aussentemperatur` | Außentemperatur |
| `sensor.mitsubishi_wp_vorlauftemperatur_heizkreis_1` | Vorlauftemperatur |
| `sensor.mitsubishi_wp_rucklauftemperatur` | Rücklauftemperatur |
| `sensor.mitsubishi_wp_durchflussmenge` | Durchflussmenge in L/min |
| `sensor.mitsubishi_wp_betriebsart` | Betriebsart (Stopp/Warmwasser/Heizen) |

---

## PV-Boost Logik

### Ablauf

```
┌─────────────────────────────────────────────────────────┐
│                    NORMALZUSTAND                        │
│              TWW-Soll: 50°C, Boost: AUS                │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼ WP springt an (Frequenz > 5 Hz, 2 Min)
                         │ UND PV > 1 kW
                         │ UND Batterie > 70%
                         │ UND TWW < 48°C
                         │ UND zwischen 07:00 – 21:00
                         │
┌────────────────────────▼────────────────────────────────┐
│                    BOOST AKTIV                          │
│              TWW-Soll: 60°C, Boost: AN                 │
│         Register 30 = 6000 via Modbus                  │
└───────┬──────────────────┬─────────────────┬────────────┘
        │                  │                 │
        ▼                  ▼                 ▼
   TWW ≥ 59°C      PV < 0.8 kW (5 Min)   21:00 Uhr
                   ODER Bat < 65% (2 Min)
        │                  │                 │
        ▼                  ▼                 ▼
┌───────────────────────────────────────────────────────┐
│                 BOOST BEENDET                         │
│           TWW-Soll: zurück auf 50°C                   │
│         Register 30 = 5000 via Modbus                 │
│                                                       │
│   WP stoppt → Speicher kühlt langsam ab               │
│   Von 60°C auf 43°C = lange Stillstandszeit           │
│   Nächster Start erst bei ~43-45°C                    │
└───────────────────────────────────────────────────────┘
```

### Schwellenwerte anpassen

Die Schwellenwerte können in der `automations.yaml` angepasst werden:

| Parameter | Standard | Beschreibung |
|---|---|---|
| PV-Leistung AN | > 1 kW | Minimum PV für Boost-Aktivierung |
| PV-Leistung AUS | < 0.8 kW | PV zu niedrig, Boost beenden |
| Batterie AN | > 70% | Minimum SOC für Boost |
| Batterie AUS | < 65% | SOC zu niedrig, Boost beenden |
| TWW Start | < 48°C | Boost nur wenn Speicher noch kühl |
| TWW Ziel | > 59°C | Boost beenden, Zieltemperatur erreicht |
| Boost-Temp | 60°C | Erhöhte Solltemperatur |
| Normal-Temp | 50°C | Standard-Solltemperatur |
| Zeitfenster | 07:00 – 21:00 | Boost nur tagsüber |

---

## Taktungsbewertung

Die Taktungsbewertung analysiert die WP-Laufmuster und gibt eine Gesamtnote:

| Starts/Tag | Ø Laufzeit | Bewertung |
|---|---|---|
| 0 (WP läuft) | — | 🟢 Dauerlauf |
| 0 (WP aus) | — | Letzte Bewertung bleibt |
| 1–2 | egal | 🟢 Optimal |
| 3–4 | ≥ 30 min | 🟢 Gut |
| 3–4 | < 30 min | 🟡 Normal |
| 5–8 | ≥ 15 min | 🟡 Normal |
| 5–8 | < 15 min | 🟠 Beobachten |
| 9+ | egal | 🔴 Kritisch |

> **Hinweis:** Da ein Abtaubetrieb die Wärmepumpe physikalisch stoppt und neu anfährt, würde dies normalerweise als "Taktung" gezählt werden. Unsere Automation *WP Taktung - Abtauzyklus zählen & Filtern* greift hier jedoch smart ein: Sie erhöht den neuen `Abtauzyklen`-Zähler und zieht exakt in dem Moment `-1` vom `Starts`-Zähler ab. Wenn die WP nach dem Abtauen also wieder anläuft (+1 Start), hebt sich das physikalisch auf. Damit hast du eine zu 100 % bereinigte "Starts/Tag" Statistik, die nur echte Takte zählt!

---

## COP-Berechnung

### COP Momentan

Berechnet aus den Modbus-Werten:

```
Wärmeleistung (W) = Durchfluss (L/min) × (Vorlauf - Rücklauf) (°C) × 4186 / 60
COP = Wärmeleistung (W) / Elektrische Leistung (W)
```

- Nur berechnet wenn WP läuft (> 200 W elektrisch)
- Behält den letzten Wert bei WP-Stopp
- Farbampel: ≥ 4 grün, ≥ 3 orange, < 3 rot

### COP Heute (Tageswert)

```
COP Heute = Wärme Heute (kWh) / Strom Heute (kWh)
```

Aussagekräftiger als der Momentanwert, da er Anlauf- und Abtauphasen mittelt.

---

## Fehlerbehebung

### Modbus-Verbindung funktioniert nicht

- Prüfe ob `/dev/ttyUSB0` existiert: `ls -la /dev/ttyUSB*`
- Prüfe Procon A1M DIP-Switches (1, 6, 7 auf ON)
- Prüfe Baudrate (9600) und Slave-Adresse (1)
- HA-Logs prüfen: Einstellungen → System → Protokolle → nach `modbus` suchen

### Solltemperatur wird nicht geändert

- **Register 30** verwenden, nicht Register 53!
- Skalierung: **× 100** (5000 = 50°C, 6000 = 60°C)
- Hub-Name muss exakt `mitsubishi-a1m` sein
- Nach dem Schreiben **1-2 Minuten warten** — es gibt eine Verzögerung

### WP Verbrauch zeigt 0

- Prüfe ob Shelly Pro 3EM Sensoren Werte liefern
- MAC-Adresse in den Template-Sensoren prüfen
- Shelly-Integration in HA neu laden

### Taktungszähler zählt nicht

- Counter `counter.wp_starts_heute` darf **kein Maximum von 0** haben
- Automation `WP Taktung - Start zählen` muss aktiviert sein
- Trigger feuert nur beim **Übergang** von ≤ 5 Hz auf > 5 Hz

### PV Boost löst nicht aus

- Alle Bedingungen prüfen (PV, Batterie, TWW, Zeit, Boost-Status)
- `input_boolean.wp_boost_aktiv` muss als Helfer angelegt sein
- `sensor.pv_power` ist in **kW** (nicht W!) — Schwelle ist `above: 1`

---

## Danksagung

- **[helgeklein](https://github.com/helgeklein/mitsubishi-heat-pump-modbus-home-assistant)** — Für das hervorragende Modbus-Package für Mitsubishi Wärmepumpen. Dieses Projekt baut darauf auf und erweitert es um PV-Boost Funktionalität.
- **Anthropic Claude** & **Google Antigravity Gemini 3.1 Pro** — Intensive KI-gestützte Fehleranalyse, Code-Optimierung und Refactoring in v2

---

## Lizenz

Dieses Projekt steht unter der [MIT-Lizenz](LICENSE).
