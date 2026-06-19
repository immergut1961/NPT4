# 🖨️ Klipper Kontext-Starter – Elegoo Neptune 4 Plus

Bitte lies zuerst diese drei Konfigurationsdateien, bevor wir anfangen:

- https://github.com/immergut1961/NPT4/blob/main/printer_data/config/printer.cfg
- https://github.com/immergut1961/NPT4/blob/main/printer_data/config/user_settings.cfg
- https://github.com/immergut1961/NPT4/blob/main/printer_data/config/mmu/optional/mmu_eigene_macros.cfg

---

## 🖨️ Drucker & System

| | |
|---|---|
| **Drucker** | Elegoo Neptune 4 Plus |
| **OS** | OpenNept4une v0.1.7-5-g33f60559 |
| **Klipper** | v0.13.0-698 |
| **Mainsail + Fluidd** | beide aktiv (Fluidd Port 81) |
| **Slicer** | Orca Slicer 2.4.0 Beta |
| **Config-Pfad** | `printer_data/config/` (Unterordner-Struktur) |
| **Backup** | Automatisch via Klipper-Backup → GitHub (öffentlich) |

---

## 🔀 MMU – Multi-Material

| | |
|---|---|
| **MMU** | Tradrack 14-Kanal |
| **Software** | Happy Hare v3.4.2-29 |
| **Pre-Gate Sensoren** | Raspberry Pi Pico #1 (alle 14 Kanäle) |
| **NFC Reader** | Raspberry Pi Pico #2 als Serial Bridge → NFC-Tracking der Filamentspulen |
| **Filamentverwaltung** | Spoolman als Docker auf Synology DS216+II |

---

## 🎯 Laufende Projekte

- **Intelligentes Purge-Volumen** (Farbe + Material-Aufschlag beim Toolchange)
  - Status: ✅ produktiv im Einsatz, verifiziert
  - Projektdokument: https://github.com/immergut1961/NPT4/blob/main/printer_data/config/Dokumentation/Projekte/intelligentes_purge_volumen.md
  - Override von `_MMU_PURGE` in `mmu/optional/mmu_eigene_macros.cfg`
  - Persistenz über `[save_variables]` → `mmu/mmu_vars.cfg`
  - **Wichtiger Bugfix:** `gate_material`/`gate_color` immer über `printer.mmu.gate` indizieren, nie über `printer.mmu.tool` (bei 1 Hotend ist `tool` immer `0`, das Gate wechselt aber je nach Filament)
  
---


## 📁 Wichtige Konfigurationsdateien

| Datei | Zweck |
|---|---|
| `printer.cfg` | Hauptkonfiguration (Hardware, Basis-Makros) |
| `user_settings.cfg` | Persönliche Overrides – hat **Vorrang** vor printer.cfg, wird bei Updates **nicht** überschrieben |
| `mmu/base/mmu.cfg` | Happy Hare Basiskonfiguration |
| `mmu/base/mmu_hardware.cfg` | MMU Hardware-Pins |
| `mmu/base/mmu_macro_vars.cfg` | Happy Hare Makro-Variablen |
| `mmu/base/mmu_parameters.cfg` | Happy Hare Parameter |
| `mmu/optional/mmu_eigene_macros.cfg` | Eigene MMU-Makros (Tip-Forming, Material-Profile) |
| `mmu/optional/mmu_spoolman.cfg` | Spoolman-Integration für MMU |
| `KAMP_Settings.cfg` | Klipper Adaptive Meshing & Purging |
| `spoolman.cfg` | Spoolman-Verbindung |
| `shell_command.cfg` | Shell-Befehle (z.B. Tasmota) |

---

## ⚙️ Wichtige Besonderheiten & Logik

### Z-Offset Strategie (additive Logik)
| Platte | Material | Gesamt-Offset |
|---|---|---|
| Standard/High Temp | PLA | `0.00` |
| Standard/High Temp | PETG | `-0.025` |
| Textured PEI | PLA | `+0.11` |
| Textured PEI | PETG | `+0.085` |

- Platten-Offset wird in `PRINT_START` via `PLATE_TYPE=` Parameter gesetzt
- Material-Offset wird im **Filament Start G-Code** (Orca Slicer) via `SET_GCODE_OFFSET Z_ADJUST=` gesetzt

### Material-Profile (Tip-Forming)
- `_SET_MATERIAL_VARS MATERIAL="PLA"` oder `"PETG"` setzt automatisch alle Tip-Forming Parameter
- Wird aufgerufen: in `PRINT_START`, im Filament Start G-Code (pro Filamentprofil hardcoded), und im End G-Code
- Makro liegt in: `mmu/optional/mmu_eigene_macros.cfg`

### Tasmota Shutdown
- Drucker kann nach dem Druck automatisch via Tasmota (IP: 192.168.2.28) ausgeschaltet werden
- Aktivierung: `SHUTDOWN_NACH_DRUCK_VORMERKEN STATUS=1`
- Logik: Abkühlung auf 120°C → Tasmota-Timer → System-Shutdown

### Lüftersteuerung
- Mainboard-Lüfter: `controller_fan hardware_cooling` (Pin PC7) – reagiert auf Extruder, Bett und Motoren
- Bauteillüfter: Pin PB7

### Slicer Start G-Code Reihenfolge
1. `SYNC_MMU_SPOOLMAN`
2. `MMU_START_SETUP` (mit allen Parametern inkl. MATERIAL)
3. `MMU_START_CHECK`
4. `PRINT_START PLATE_TYPE=... BED_TEMP=... EXTRUDER_TEMP=... MATERIAL=...`
5. `MMU_START_LOAD_INITIAL_TOOL`

---

## 🔗 GitHub Repository
https://github.com/immergut1961/NPT4

---

## 💡 Hinweise für den Chat

- Dateien können direkt über GitHub-Links gelesen werden (normale blob-Links funktionieren)
- Änderungen gehören immer in `user_settings.cfg` (update-sicher), nicht direkt in `printer.cfg`
- Happy Hare Basis-Dateien (`mmu/base/`) **nie** direkt editieren – nur über `mmu_eigene_macros.cfg` oder `mmu_macro_vars.cfg` überschreiben
