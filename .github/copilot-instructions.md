# Copilot Instructions für NPT4 - Elegoo Neptune 4 Plus Klipper Backup

## Projekt-Übersicht

Dieses Repository ist ein automatisches Backup des kompletten Klipper-Setups für einen **Elegoo Neptune 4 Plus** 3D-Drucker mit erweiterten Komponenten und Multi-Material-Unterstützung.

### Hardware & Firmware

**Drucker:**
- Elegoo Neptune 4 Plus
- OpenNept4une: v0.1.7-5-g33f60559
- Klipper: v0.13.0-694-g4c137ed3 → v0.13.0-698

**Multi-Material Unit (MMU):**
- Tradrack 14-Kanal MMU
- Happy Hare: v3.4.2-22-ga880ac0a → v3.4.2-29

**Sensoren & Steuerung:**
- Raspberry Pico #1: Pre-Gate Sensoren Management
- Raspberry Pico #2: NFC Reader Bridge für Filamentspulen-Tracking

**Filamentverwaltung:**
- Spoolman (Docker)
- Läuft auf Synology DS216+II

## Repository-Struktur

```
printer_data/config/
├── printer.cfg              # Hauptkonfiguration
├── macros/                  # Druckermakros
├── mmu/                     # Happy Hare & Tradrack Konfiguration
├── sensors/                 # Pre-Gate Sensor Konfigurationen
├── hardware/                # Pico & Bridge Konfigurationen
├── spoolman/                # Spoolman Integration
└── [weitere Konfigurationen]
```

## Coding-Standards & Konventionen

### Klipper Makros & Skripte

1. **Makro-Naming:**
   - `PRINT_*` - Drucksteuerungsmakros
   - `MMU_*` - Multi-Material Unit Operationen
   - `FILAMENT_*` - Filamentverwaltung
   - `SENSOR_*` - Sensor-bezogene Makros
   - `MAINTENANCE_*` - Wartungsmakros

2. **Best Practices:**
   - Alle Makros sollten aussagekräftige Variable und Kommentare haben
   - Fehlerverwaltung mit `RESPOND` und `PAUSE` Befehlen
   - Logging von kritischen Aktionen
   - Timeout-Werte für Pico-Operationen definieren

3. **Integration mit Happy Hare:**
   - Makros müssen mit `TRADRACK_*` Befehlen kompatibel sein
   - Pre-Gate Sensoren sollten über dedizierte Makros angesteuert werden
   - Fehlerbehandlung für MMU-Fehlzüge

### Sensor-Konfiguration

- Pre-Gate Sensoren: GPIO-basiert auf Pico #1
- NFC Reader: I2C/SPI Bridge via Pico #2
- Alle Sensor-Werte sollten in regulären Logs verfügbar sein

### Pico-Programmierung

- MicroPython für Sensor-Skripte
- CAN-Bus für Kommunikation mit Klipper/Happy Hare
- Timeout-Handling für Sensor-Ausfälle

### Spoolman Integration

- Docker-Container mit persistenten Daten
- NFC Reader trägt automatisch zur Spoolman-Datenbank bei
- API-Integration mit Klipper für Filament-Tracking

## Wichtige Dateien

- `printer_data/config/printer.cfg` - Hauptkonfiguration (NICHT direkt bearbeiten)
- `printer_data/config/macros/` - Benutzerdefinierte Makros (modular organisieren)
- `printer_data/config/mmu/` - Happy Hare & Tradrack-Konfiguration
- `printer_data/config/sensors/` - Sensor-Definitionen

## Häufige Aufgaben für Copilot

Copilot sollte bei folgenden Aufgaben helfen:

1. **Neue Makros erstellen:**
   - Konsistent mit bestehenden Naming-Konventionen
   - Mit Fehlerbehandlung
   - Mit Logging

2. **Happy Hare Integration:**
   - MMU-Makros debugging
   - Sensor-Fehlerbehandlung
   - Tradrack-Kalibrierung

3. **Sensor-Konfiguration:**
   - Pre-Gate Sensor Einrichtung
   - NFC Reader Integration
   - Fehlerdiagnose

4. **Spoolman Integration:**
   - Automatisches Filament-Tracking
   - Docker-Container Verwaltung
   - API-Requests für Spoolman

5. **Pico-Skripte:**
   - MicroPython Code für Sensoren
   - CAN-Bus Kommunikation
   - Fehlerbehandlung

## Verwandte Dokumentation

- [Klipper Dokumentation](https://www.klipper3d.org/)
- [Happy Hare Dokumentation](https://github.com/moggieuk/Happy-Hare)
- [Spoolman GitHub](https://github.com/Donkie/Spoolman)
- [OpenNept4une Repository](https://github.com/OpenNept4une)

## Version-Historie

- **Klipper:** v0.13.0-694 → v0.13.0-698
- **Happy Hare:** v3.4.2-22 → v3.4.2-29
- **OpenNept4une:** v0.1.7-5-g33f60559

Bitte Versionsupdates dokumentieren, wenn diese Datei aktualisiert wird.

## Kontakt & Fragen

Bei Fragen zu Makro-Strukturen oder Skript-Integration sollte sich Copilot an diese Dokumentation halten und konsistente Lösungen basierend auf den bestehenden Patterns im Repository erstellen.
