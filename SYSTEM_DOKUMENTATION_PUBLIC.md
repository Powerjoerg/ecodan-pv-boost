# Mitsubishi Ecodan PV-Boost — Home Assistant Integration (PUBLIC DOKUMENT)

✅ **EXTERNE VERSION: BEREINIGT UM SENSIBLE NETZWERK- & GERÄTEDATEN - HÄCKERFEST (ZUM TEILEN)!** ✅

Automatische PV-Überschuss-Steuerung für Mitsubishi Ecodan Wärmepumpen mit Home Assistant, Modbus (Procon A1M) und Shelly Pro 3EM. Dies ist die **Version 2.5 "The Rock"**.

> **Ziel:** Wenn genug Solarstrom vorhanden ist und die Batterie ausreichend geladen ist, wird die Warmwasser-Solltemperatur automatisch von 50°C auf 60°C angehoben. Dadurch wird überschüssiger PV-Strom als Wärme gespeichert. Ein Ladewächter sorgt für den "Zwangsschlaf" durch Modbus-Temperatur-Absenkung (auf 40°C), während ein Hardware-Shelly In1 als Ausfallsicherung dient.

---

## 1. Systemübersicht & Hydraulik

```text
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   PV-Anlage     │────▶│ Hybrid-          │────▶│   Batterie      │
│   [LEISTUNG]    │     │ Wechselrichter   │     │   [KAPAZITÄT]   │
└─────────────────┘     └──────┬───────────┘     └─────────────────┘
                               │
                               │ PV-Daten via HA
                               ▼
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│ Mitsubishi       │     │  Home Assistant  │     │ Smart-Meter     │
│ Ecodan WP        │◀──▶│  Zentrale        │◀───│ WP-Verbrauch    │
│ PUD-SHWM      │     │                  │     │ 3-Phasen Mess.  │
│ FTC6 Controller  │     │  WP-Steuerung v2 │     │                 │
└────────┬─────────┘     └───────┬──┬───────┘     └─────────────────┘
   │     │                       │  │                      
  Hydr.  │ Modbus RTU            │  │ WLAN (Isoliert)
   │     │ RS-485                │  │                      
   ▼     ▼                       │  ▼                      
┌─────────────────┐     ┌────────┴─────────┐     ┌─────────────────┐
│ Modbus Adapter  │     │ USB to RS-485    │     │ Shelly Relais   │
│ MelcoBEMS MINI  │◀───▶│ /dev/ttyUSB0     │     │ Potentialfrei  │
│                 │     │                  │     │ an IN1 FTC6     │
└─────────────────┘     └──────────────────┘     └─────────────────┘
   │
   ▼ (Hydraulik)
┌─────────────────┐     ┌──────────────────┐
│ Trenn-/Puffer-  │────▶│ Frischwasser-    │ (Speist TWW aus Puffer)
│ speicher        │     │ station (FriWa)  │
└──────┬──────────┘     └──────────────────┘
       │
       ▼
┌─────────────────┐
│ externe Pumpen- │
│ gruppe & FBH    │
└─────────────────┘
```

**Hydraulische Arbeitsweise:** 
Die Ecodan speist in einen Puffer. An diesem Puffer hängt eine Frischwasserstation, welche ihre Wärme für Trinkwarmwasser (TWW) direkt aus dem Puffer bezieht. Eine externe Pumpengruppe mischt die FBH-Kreise. Dadurch kann die WP über den reinen TWW-Modus (Register 30 Manipulation via Modbus) in einen Zwangsschlaf versetzt werden, ohne dass die Hausheizung abbricht, sofern der Puffer Restwärme besitzt und die Raumthermostate bedient, da die Heizkreisgruppen extern am Puffer bedient werden.

---

## 2. Hardware-Architektur

Es werden ausschließlich allgemeine Entitäts-Beschreibungen genutzt. **Befülle diese in deiner eigenen Anlage mit deinen IDs!**

| Funktion | Setup & Integration | Parameter-Hinweis |
|-------|-----|----------------------|
| **WP Freigabe-Relais** | Smartes Relais am WP-Mainboard (IN1). | `switch.[NAME_DEINES_SHELLYS]` |
| **WP Verbrauchsmessung** | 3-Phasen Smart Meter an den WP-Sicherungen. | `sensor.[DEIN_STROMZAHLER]_phase_a_leistung` |
| **EV Charger** | Smarte Wallbox für PV-Überschuss. | `sensor.[WALLBOX_ID]_plug_status` |
| **Smart Thermostate** | Zigbee/WLAN Thermostate in den Räumen. | `climate.[RAUMNAME]` |

DIP-Switches (Müssen zwingend so stehen!):
- **FTC6 SW2-1**: ON (Externe Thermostatsteuerung IN1 aktiv. **Zwingend für Relais-Steuerung**)
- **Modbus Adapter**: DIP 1, 6, 7 auf ON (Spezifisch für Mitsubishi MelcoBEMS)

---

## 3. Die v2.5 "The Rock" Logik (Software)

### 3.1 Zwangsschlaf & Ladewächter-Hysterese
Die Kernlogik eliminiert das Takten der WP. Die Automation in Home Assistant erzeugt eine künstliche Hysterese:
- Puffer unter **40°C** → Modbus Set-Temp schreibt **50°C** ans Register. Relais ON. WP startet.
- Puffer über **50°C** → Modbus Set Temp schreibt **40°C** ans Register. Relais OFF. WP fällt in den Zwangsschlaf!.

### 3.2 PV-Boost
- Bedingungen: PV-Überschuss greift und Hausspeicher ist ausreichend voll.
- **Aktion**: Ignoriert die reguläre 50°C Grenze des Ladewächters und überschreibt das Modbus Register auf 60°C. 
- Fällt der PV-Überschuss weg, wird der Boost umgehend beendet. Modbus schreibt den Wert auf 50°C zurück und zwingt die WP dadurch bei Erreichen der neuen Grenze wieder in den Regel-Zwangsschlaf.

### 3.3 Modbus-Heartbeat Watchdog (Sicherheitsschicht!)
- **Problem**: Wärmepumpen-Controller wie der FTC6 haben ein hartes Time-Out (Mitsubishi ltem 128 = 30 Minuten). Bricht das Modbus-Gateway ab, wird der geschriebene Registerwert gelöscht und auf Default gesetzt.
- **Lösung**: Ein Watchdog sendet alle 10 Minuten einen "Heartbeat" (den aktuellen Soll-Wert per Modbus) als Write-Command an das Register. Dadurch reißt das Timeout des Herstellers niemals ab.

### 3.4 Fallback-Strategie / Notfall-Sicherungen (Failsafe)
Bei allen DIY-Integrationen in kritische Infrastruktur wie Hausheizungen, muss der Systemausfall (Raspi Tot, WLAN Tot) hardwareseitig verbeugt werden:
- **Relais-Fallbacks (Power-On-Default)**: Dem Schaltrelais muss physikalisch oder via Firmware mitgegeben werden, wie es bei Stromrückkehr reagieren soll. Im tiefsten Winter sollte es fest auf "ON" oder "Restore Last" booten, um einer frierenden Anlage eine ungesicherte Dauerfreigabe zu erteilen. 
- **Modbus Trennung (Hard-Rollback)**: Bei Komplettversagen reicht am Mainboard der WPF das Umlegen des betroffenen DIP-Schalters (SW2-1 auf OFF). Dann ignoriert das System den "toten" Türsteher-Shelly und regelt wieder autonom temperaturgesteuert nach Werkswerte.
