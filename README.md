# Mitsubishi Ecodan PV-Boost — Home Assistant Integration

Automatische PV-Überschuss-Steuerung für Mitsubishi Ecodan Wärmepumpen mit Home Assistant, Modbus (Procon A1M), Shelly Pro 3EM und smartem Raumthermostat-Ersatz.

> **Ziel:** Wenn genug Solarstrom vorhanden ist und die Batterie ausreichend geladen ist, wird die Warmwasser-Solltemperatur automatisch von 50°C auf 60°C angehoben. Ein Shelly an IN1 der FTC6 ersetzt den Raumthermostat und steuert den Heizbedarf intelligent basierend auf allen Raumtemperaturen — inklusive Nachtabsenkung.

---

## Inhaltsverzeichnis

- [Systemübersicht](#systemübersicht)
- [Hardware](#hardware)
- [Voraussetzungen](#voraussetzungen)
- [Installation](#installation)
  - [1. Modbus-Verbindung](#1-modbus-verbindung)
  - [2. Shelly Pro 3EM — WP-Verbrauchsmessung](#2-shelly-pro-3em--wp-verbrauchsmessung)
  - [3. Shelly 1 Mini Gen4 — Thermostat-Ersatz IN1](#3-shelly-1-mini-gen4--thermostat-ersatz-in1)
  - [4. Home Assistant Konfiguration](#4-home-assistant-konfiguration)
  - [5. Helfer anlegen](#5-helfer-anlegen)
  - [6. Automationen einrichten](#6-automationen-einrichten)
  - [7. Dashboard einrichten](#7-dashboard-einrichten)
- [DIP-Switch Einstellungen](#dip-switch-einstellungen)
- [Modbus Register-Map](#modbus-register-map)
- [PV-Boost Logik](#pv-boost-logik)
- [Shelly Thermostat-Logik](#shelly-thermostat-logik)
- [Nachtabsenkung](#nachtabsenkung)
- [Taktungsbewertung](#taktungsbewertung)
- [COP-Berechnung](#cop-berechnung)
- [Fehlerbehebung](#fehlerbehebung)
- [Roadmap — Geplante Erweiterungen](#roadmap--geplante-erweiterungen)
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
│ EHSD-YM9D       │     │  PV-Boost        │     │ CT2: Heizstab(×3)│
│ FTC6 Controller  │     │  Automationen    │     │ CT3: Innen (×1) │
└──┬─────┬─────────┘     └──────┬───────────┘     └─────────────────┘
   │     │                      │
   │     │ Modbus RTU           │
   │     │ RS-485               │
   │     ▼                      │
   │  ┌─────────────────┐     ┌┴──────────────────┐
   │  │ Procon A1M      │     │ Waveshare         │
   │  │ MelcoBEMS MINI  │◀───▶│ USB to RS-485     │
   │  │ Modbus Adapter  │     │ /dev/ttyUSB0      │
   │  └─────────────────┘     └───────────────────┘
   │
   │ IN1 (potentialfrei)
   ▼
┌─────────────────┐     ┌──────────────────────────┐
│ Shelly 1 Mini   │     │ Raumthermostaten         │
│ Gen4             │◀───│ LinkedGo ST1820-W (×8)   │
│ WP Thermostat   │     │ → Heizbedarf-Logik in HA │
└─────────────────┘     └──────────────────────────┘
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
| **Thermostat-Ersatz** | Shelly 1 Mini Gen4 | Potentialfreier Kontakt an IN1 der FTC6 |
| **Raumthermostaten** | LinkedGo ST1820-W (×8) | Smart-Thermostate mit Stellantrieben |
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

---

## Voraussetzungen

- Home Assistant (2024.x oder neuer)
- HACS installiert (für Mushroom Cards)
- [Mushroom Cards](https://github.com/piitaya/lovelace-mushroom) installiert
- Procon A1M korrekt verkabelt (RS-485 → Waveshare → USB → Raspberry Pi)
- Shelly Pro 3EM und Shelly 1 Mini Gen4 in HA integriert
- FoxESS Integration in HA (für PV-Daten und Batterie-SOC)
- Raumthermostaten (z.B. LinkedGo ST1820-W) als `climate.*` Entities in HA
- Folgende HA-Sensoren müssen vorhanden sein:
  - `sensor.pv_power` — aktuelle PV-Leistung in **kW**
  - `sensor.battery_soc_1` — Batterie-Ladezustand in **%**
  - `sensor.solar_energy_today` — PV-Erzeugung heute in **kWh**
  - `sensor.feed_in_energy_today` — Einspeisung heute in **kWh**

---

## Installation

### 1. Modbus-Verbindung

#### Verkabelung

```
Procon A1M (CN105) ──── Waveshare USB-to-RS485 ──── Raspberry Pi USB
    A+ ──────────────────── A+
    B- ──────────────────── B-
    GND ─────────────────── GND (optional)
```

Der Waveshare Adapter erscheint als `/dev/ttyUSB0` auf dem Raspberry Pi.

#### Modbus-Package

Dieses Projekt basiert auf dem [helgeklein Mitsubishi A1M Package](https://github.com/helgeklein/mitsubishi-heat-pump-modbus-home-assistant), angepasst für die **serielle Verbindung** (A1M statt A1M+).

Kopiere die Datei `packages/mitsubishi_a1m.yaml` in deinen HA-Ordner `config/packages/`.

> **Wichtig:** Die Datei muss die Endung `.yaml` haben und darf keine Sonderzeichen im Namen enthalten. Home Assistant lädt über `!include_dir_named` nur `.yaml`-Dateien.

> **Hinweis Entity-IDs (Umlaute):** Home Assistant wandelt Umlaute in Entity-IDs je nach Version unterschiedlich um — z.B. wird "Wärmepumpe" zu `warmepumpe` (ohne e) oder `waermepumpe` (mit ae). Die Dateien in diesem Repo verwenden die Variante **ohne e** (`warmepumpe`, `rucklauftemperatur`). Falls dein HA die Variante mit ae/ue erzeugt, musst du die Entity-IDs in den Automationen, Templates und im Dashboard entsprechend anpassen.

Falls der `packages`-Ordner noch nicht existiert, füge in der `configuration.yaml` hinzu:

```yaml
homeassistant:
  packages: !include_dir_named packages/

automation: !include automations.yaml
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

### 2. Shelly Pro 3EM — WP-Verbrauchsmessung

Der Shelly Pro 3EM misst den elektrischen Verbrauch der WP über drei CT-Klemmen im Sicherungskasten:

- **CT1 (Phase A)** → Außeneinheit (1 Phase von 3)
- **CT2 (Phase B)** → Heizstab (1 Phase von 3)
- **CT3 (Phase C)** → Innenmodul (1-phasig)

In HA werden daraus per Template-Sensor die Gesamtwerte berechnet (siehe `configuration_templates.yaml`).

> **Wichtig:** Die Entity-IDs der Shelly-Sensoren enthalten die MAC-Adresse des Geräts. Diese muss in den Template-Sensoren angepasst werden (Platzhalter `XXXXXXXXXXXX` ersetzen).

---

### 3. Shelly 1 Mini Gen4 — Thermostat-Ersatz IN1

Der Shelly 1 Mini Gen4 ersetzt den originalen Mitsubishi Raumthermostat und wird direkt im WP-Innengerät verbaut.

#### Funktionsweise

- **Relais ON** (Kontakt geschlossen) = Heizbedarf → FTC gibt Heizung frei
- **Relais OFF** (Kontakt offen) = kein Heizbedarf → FTC stoppt Heizung Zone 1

> **Wichtig:** Der Shelly gibt der FTC nur die **Erlaubnis** zu heizen — er zwingt sie nicht. Die FTC entscheidet weiterhin selbstständig wann und was sie macht (Heizen, Warmwasser oder Stopp).

#### Verkabelung

```
Shelly 1 Mini Gen4          WP Innengerät
────────────────────────────────────────────
  L  ──────────────────── Klemme L (230V)
  N  ──────────────────── Klemme N
  O  ──────────────────── IN1 Pin 1 (TBI.1)
  I  ──────────────────── IN1 Pin 2 (TBI.1)
  SW ──────────────────── frei (nicht anschließen)
```

> **ACHTUNG:** Die Klemme **I** wird **NICHT** mit **L** verbunden! I und O gehen beide direkt auf IN1 — dadurch ist der Kontakt potentialfrei. 230V auf IN1 würde die FTC6-Platine zerstören!

#### Shelly-Einstellungen

| Einstellung | Wert | Begründung |
|---|---|---|
| Eingang (SW) | Deaktiviert | Keine physische Bedienung |
| Ausgang | Schalter | Dauerhaft ON oder OFF |
| Startzustand nach Stromausfall | **AN** | WP darf sofort heizen nach Stromausfall |

#### FTC6 DIP-Switch

**SW2-1 muss auf ON** stehen — aktiviert den externen Thermostat-Eingang IN1 für Zone 1.

> **Warnung:** SW2-1 nur auf ON setzen wenn der Shelly angeschlossen und in HA eingebunden ist! Ohne angeschlossenen Kontakt interpretiert die FTC "offener Kontakt" als "kein Heizbedarf" und die Heizung für Zone 1 stoppt.

---

### 4. Home Assistant Konfiguration

Füge in der `configuration.yaml` den Inhalt von `configuration_templates.yaml` aus diesem Repo ein.

Die Template-Sensoren umfassen:

| Sensor | Beschreibung |
|---|---|
| `sensor.wp_aussen_gesamt` | Außeneinheit Leistung (Phase A × 3) |
| `sensor.wp_heizstab_gesamt` | Heizstab Leistung (Phase B × 3) |
| `sensor.wp_innen_gesamt` | Innenmodul Leistung (Phase C × 1) |
| `sensor.wp_gesamtverbrauch` | Gesamter elektrischer WP-Verbrauch in W |
| `sensor.wp_waermeleistung` | Erzeugte Wärmeleistung (Durchfluss × Temperaturdifferenz) |
| `sensor.wp_cop` | COP Momentanwert (behält letzten Wert bei WP-Stopp) |
| `sensor.wp_cop_heute` | COP Tageswert |
| `sensor.wp_laufzeit_pro_zyklus` | Ø Laufzeit pro Zyklus in Minuten |
| `sensor.wp_taktungsbewertung` | Gesamtbewertung (Optimal/Gut/Normal/Beobachten/Kritisch) |
| `sensor.wp_heizbedarf` | True wenn mindestens ein Raum unter Soll |
| `sensor.wp_nachtabsenkung` | Anzeige ob Nachtabsenkung aktiv |
| `binary_sensor.wp_laeuft` | WP läuft (Frequenz > 5 Hz) |
| `sensor.wp_laufzeit_heute` | Gesamte Laufzeit heute in Stunden |

---

### 5. Helfer anlegen

**Einstellungen → Helfer → + Helfer erstellen**

#### Schalter (Toggle)

| Name | Entity-ID |
|---|---|
| WP Boost aktiv | `input_boolean.wp_boost_aktiv` |

#### Nummer (Number)

| Name | Min | Max | Schritt | Einheit |
|---|---|---|---|---|
| WP Soll Temperatur | 40 | 60 | 1 | °C |

#### Zähler (Counter)

| Name | Anfangswert | Minimum | Schritt |
|---|---|---|---|
| WP Starts Heute | 0 | 0 | 1 |

> **Wichtig:** Beim Counter darauf achten, dass **Maximum** nicht auf 0 gesetzt wird!

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

### 6. Automationen einrichten

Kopiere den Inhalt von `automations.yaml` aus diesem Repo in `config/automations.yaml`.

#### Übersicht (14 Automationen)

| Nr | Name | Beschreibung |
|---|---|---|
| 1 | WP Taktung - Start zählen | Zählt WP-Starts (Frequenz > 5 Hz für 1 Min) |
| 2 | WP Taktung - Reset Mitternacht | Setzt Counter täglich auf 0 |
| 3 | **WP Boost aktivieren** | Soll auf 60°C bei PV > 1 kW, Batterie > 70%, TWW < 48°C |
| 4 | **WP Boost beenden** | Zurück auf 50°C bei PV < 0.8 kW, Batterie < 65% oder 21:00 |
| 5 | **WP Boost Ziel erreicht** | Zurück auf 50°C bei TWW > 59°C |
| 6 | **WP Boost WP gestoppt** | Zurück auf 50°C wenn WP 5 Min stoppt |
| 7 | WP Soll manuell setzen | Slider im Dashboard schreibt Soll per Modbus |
| 8 | **WP Thermostat Heizbedarf AN** | Shelly ON wenn ein Raum unter Soll |
| 9 | **WP Thermostat Heizbedarf AUS** | Shelly OFF wenn alle Räume über Soll (10 Min Verzögerung) |
| 10 | **WP Nachtabsenkung Start** | 21:00 — alle Thermostaten Soll -3°C |
| 11 | **WP Nachtabsenkung Ende** | 03:00 — alle Thermostaten Soll +3°C zurück |

> Dazu kommen die Weihnachtsbeleuchtung-Automationen (projektunabhängig) und ggf. weitere eigene Automationen.

---

### 7. Dashboard einrichten

Kopiere den Inhalt von `dashboards/wp_dashboard.yaml` als manuelle YAML-Karte in dein Dashboard.

Das Dashboard zeigt:
- WP-Status (Läuft/Aus) mit Verdichterfrequenz
- Speicher-, Soll- und Außentemperatur
- Batterie-SOC, PV-Erzeugung, Einspeisung, aktuelle PV-Leistung
- Verdichter Hz, Vorlauf, Betriebsart, WP-Leistung
- Soll-Temperatur TWW mit +/− Tasten
- COP Momentan und COP Heute
- WP-Verbrauch Heute/Monat/Jahr
- Taktung (Starts + Ø Laufzeit), Bewertung, PV-Boost Status

---

## DIP-Switch Einstellungen

### FTC6 Controller

#### SW1

| Schalter | Stellung | Funktion |
|---|---|---|
| SW1-1 | **ON** | Monoblock-Wärmepumpe |
| SW1-3 | **ON** | Warmwasserbetrieb aktiv |
| SW1-6 | **ON** | Modbus-Kommunikation (A1M) |
| SW1-7 | **ON** | Modbus-Kommunikation (A1M) |
| SW1-8 | OFF | Keine Mitsubishi Funk-Fernbedienung |
| Rest | OFF | — |

#### SW2

| Schalter | Stellung | Funktion |
|---|---|---|
| SW2-1 | **ON** | Externer Thermostat an IN1 (Shelly) |
| SW2-8 | **ON** | Strömungswächter aktiv |
| Rest | OFF | — |

#### Procon A1M

| DIP | Stellung |
|---|---|
| DIP 1 | **ON** |
| DIP 6 | **ON** |
| DIP 7 | **ON** |

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

| Register | Beschreibung | Skalierung Lesen | Skalierung Schreiben | Beispiel |
|---|---|---|---|---|
| **30** | **TWW-Solltemperatur** | × 0.01 | **× 100** | 5000 = 50°C, 6000 = 60°C |

> **ACHTUNG:** Das helgeklein-Package verwendet Register 53 für die TWW-Solltemperatur. Bei der seriellen A1M-Variante funktioniert **nur Register 30** zum Schreiben mit Skalierung **× 100**. Register 53 reagiert nicht auf Schreibbefehle!

> **Verzögerung:** Nach einem Schreibbefehl dauert es ca. **1-2 Minuten**, bis der neue Wert von der FTC übernommen wird.

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
│              TWW-Soll: 60°C (Register 30 = 6000)       │
└───────┬──────────┬──────────┬──────────┬────────────────┘
        │          │          │          │
        ▼          ▼          ▼          ▼
   TWW ≥ 59°C  PV<0.8kW   Bat<65%    21:00 Uhr
                (5 Min)    (2 Min)
        │          │          │          │
        ▼          ▼          ▼          ▼
   WP stoppt                              
   (5 Min)                                 
        │                                  
        ▼                                  
┌───────────────────────────────────────────────────────┐
│                 BOOST BEENDET                         │
│           TWW-Soll: zurück auf 50°C (Reg 30 = 5000)  │
│   Nächster WP-Start erst bei ~43-45°C                 │
└───────────────────────────────────────────────────────┘
```

### Schwellenwerte

| Parameter | Wert | Beschreibung |
|---|---|---|
| PV-Leistung AN | > 1 kW | Minimum PV für Boost |
| PV-Leistung AUS | < 0.8 kW | PV zu niedrig |
| Batterie AN | > 70% | Minimum SOC |
| Batterie AUS | < 65% | SOC zu niedrig |
| TWW Start | < 48°C | Boost nur wenn Speicher kühl |
| TWW Ziel | > 59°C | Zieltemperatur erreicht |
| Boost-Temp | 60°C | Erhöhte Solltemperatur |
| Normal-Temp | 50°C | Standard-Solltemperatur |
| Zeitfenster | 07:00 – 21:00 | Boost nur tagsüber |

---

## Shelly Thermostat-Logik

Der Shelly 1 Mini Gen4 an IN1 steuert den Heizbedarf für Zone 1 basierend auf den Raumtemperaturen aller Thermostaten:

| Bedingung | Shelly | FTC |
|---|---|---|
| Mindestens ein Raum **unter** Soll | ON | Heizung freigegeben |
| Alle Räume **über** Soll (10 Min) | OFF | Heizung gesperrt |

### Vorteile gegenüber originalem Thermostat

- **Alle Räume** werden berücksichtigt, nicht nur ein Referenzraum
- **Nachtabsenkung** automatisch über HA
- **Keine kurzen Heiz-Takte** nachts wenn alle Räume warm sind
- **Stellantriebe** schließen bei Nachtabsenkung → kein Wasser fließt → WP bleibt aus

---

## Nachtabsenkung

| Uhrzeit | Aktion |
|---|---|
| **21:00** | Alle Thermostaten Soll **-3°C** → Stellantriebe schließen → WP stoppt |
| **03:00** | Alle Thermostaten Soll **+3°C** zurück → Stellantriebe öffnen → WP darf heizen |

Die Nachtabsenkung setzt die **echten Thermostat-Solltemperaturen** herunter (nicht nur die Logik im Heizbedarf-Sensor). Dadurch:

1. **Stellantriebe schließen** physisch → kein Heizwasser fließt
2. **Heizbedarf-Sensor** erkennt "alle warm genug" → Shelly OFF
3. **WP** bekommt kein Signal → keine unnötigen Starts

Ab 03:00 wird zurückgesetzt, damit die Fußbodenheizung 4-6 Stunden hat um die Räume bis morgens auf Normaltemperatur zu bringen.

---

## Taktungsbewertung

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

---

## COP-Berechnung

### Zwei COP-Sensoren

Dieses Projekt erzeugt zwei unabhängige COP-Werte:

1. **`sensor.mitsubishi_wp_cop`** — Wird im Modbus-Package aus WP-internen Leistungsdaten berechnet.
2. **`sensor.wp_cop`** — Wird in `configuration_templates.yaml` über den externen Shelly Pro 3EM und die Temperaturdifferenz (Vorlauf minus Rücklauf) berechnet. Dieser Wert ist in der Regel genauer, da der Shelly den tatsächlichen Stromverbrauch misst.

Das Dashboard zeigt `sensor.wp_cop` (den externen Wert). Der WP-interne Wert kann zum Vergleich herangezogen werden.

### Vorlauf-Sensoren

Für die COP-Berechnung wird `sensor.mitsubishi_wp_vorlauftemperatur` verwendet (direkt am WP-Ausgang). Im Dashboard wird `sensor.mitsubishi_wp_vorlauftemperatur_heizkreis_1` angezeigt (nach dem Mischer/Heizkreis). Der COP wird am WP-Ausgang berechnet, da dies den tatsächlichen Wirkungsgrad der Wärmepumpe widerspiegelt.

### COP Momentan

```
Wärmeleistung (W) = Durchfluss (L/min) × (Vorlauf - Rücklauf) (°C) × 4186 / 60
COP = Wärmeleistung (W) / Elektrische Leistung (W)
```

### COP Heute (Tageswert)

```
COP Heute = Wärme Heute (kWh) / Strom Heute (kWh)
```

---

## Fehlerbehebung

### Modbus-Verbindung funktioniert nicht

- Prüfe `/dev/ttyUSB0`: `ls -la /dev/ttyUSB*`
- Prüfe Procon A1M DIP-Switches (1, 6, 7 auf ON)
- Prüfe Baudrate (9600) und Slave-Adresse (1)

### Solltemperatur wird nicht geändert

- **Register 30** verwenden, nicht Register 53!
- Skalierung: **× 100** (5000 = 50°C, 6000 = 60°C)
- Hub-Name muss exakt `mitsubishi-a1m` sein
- Nach dem Schreiben **1-2 Minuten warten**

### WP blockiert (Betriebsart "Stopp")

- Prüfe `switch.wp_thermostat_in_1` → muss ON sein
- Prüfe SW2-1 → muss ON sein wenn Shelly angeschlossen
- "Stopp" zwischen Zyklen ist **normal** — FTC wechselt intern

### Shelly 1 Mini Gen4 — 230V auf IN1 vermeiden

- **NIEMALS** den Shelly 2PM Gen4 oder andere Shellys mit Netzspannungs-Ausgängen verwenden!
- Nur Shellys mit **potentialfreiem Relais** (Shelly 1 Mini Gen4, Shelly Plus 1 Mini)
- Oder ein externes 230V-Koppelrelais dazwischenschalten

### Nachtabsenkung funktioniert nicht

- Prüfe ob alle `climate.*` Entity-IDs korrekt sind
- Prüfe ob die Automationen aktiviert sind
- Thermostaten die physisch geschlossen sind (z.B. Eltern) auf niedrige Soll-Temperatur setzen

### Taktungszähler zählt nicht

- Counter darf **kein Maximum von 0** haben
- Automation muss aktiviert sein
- Trigger feuert nur beim **Übergang** von ≤ 5 Hz auf > 5 Hz

---
## ## Roadmap — Geplante Erweiterungen

### 🔜 Smart Meter / TRuDi Integration
- Digitaler Stromzähler mit TRuDi-Anbindung für offizielle Verbrauchsdaten
- Exakte Messung von Netzbezug, Einspeisung und Eigenverbrauch
- Ablösung / Ergänzung der Shelly Pro 3EM Verbrauchsmessung

### 🔜 Flexibler Stromtarif (dynamisch)
- Anbindung an dynamische Stromtarife (z.B. Tibber, aWATTar, Octopus Energy)
- WP-Betrieb bevorzugt in günstigen Stunden
- TWW-Aufheizung in Niedrigpreis-Fenster verschieben
- Kombination mit PV-Boost: PV-Überschuss hat Vorrang, danach günstigster Tarif

### 🔜 Batterie-Management Optimierung (FoxESS)
- Intelligentes Laden/Entladen basierend auf Strompreis
- Batterie laden wenn Strom günstig, entladen wenn teuer
- Koordination mit WP-Betrieb: WP aus Batterie versorgen wenn kein PV und Strom teuer
- Eigenverbrauchsoptimierung über den gesamten Tag

### 🔜 Feintuning Heizkurve & Taktungsoptimierung
- Auswertung der Monitoring-Daten (Kurzzyklen-Erkennung läuft bereits)
- Automatische Anpassung der Heizfreigabe basierend auf Außentemperatur
- Mindest-Laufzeiten und Mindest-Pausenzeiten für den Kompressor
- Saisonale Profile (Übergangszeit vs. Winter)

> **Status:** Diese Features sind in Planung und werden nach und nach ergänzt. Beiträge und Ideen sind willkommen — einfach ein [Issue](https://github.com/Powerjoerg/ecodan-pv-boost/issues) erstellen!

---
## Danksagung

- **[helgeklein](https://github.com/helgeklein/mitsubishi-heat-pump-modbus-home-assistant)** — Für das hervorragende Modbus-Package für Mitsubishi Wärmepumpen
- **Anthropic Claude** — KI-gestützte Entwicklung und Konfiguration

---

## Lizenz

Dieses Projekt steht unter der [MIT-Lizenz](LICENSE).
