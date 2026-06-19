# Projekt: Intelligentes Purge-Volumen
## Elegoo Neptune 4 Plus – Happy Hare v3.4.2 – Tradrack 14-Kanal

**Status: ✅ Produktiv im Einsatz und verifiziert (Druck vom 23:33, PETG→PLA Wechsel)**

---

## Ziel

Automatische Berechnung des Purge-Volumens beim Filamentwechsel basierend auf:
- **Farbwechsel** (Helligkeit von/nach)
- **Materialwechsel** (PLA↔PETG Aufschlag)
- **Persistenz** über Stromabschaltung (Tasmota) hinaus

Zusätzlich: automatisches Wischen der Nozzle über eine montierte Silikonlippe nach jedem Purge.

---

## Ausgangssituation

| Parameter | Wert |
|---|---|
| Purge-Position | X=336 (stationär in Bucket, kein Wipetower, keine Purge Line) |
| Silikonlippe | An Z-Achse montiert, Wischen via X=336→328→336 |
| Purge-Makro | `_MMU_PURGE` (Happy Hare intern, `mmu_purge.cfg` READ ONLY) |
| `force_purge_standalone` | `1` (Happy Hare berechnet Basisvolumen intern) |
| Persistenz | `[save_variables]` bereits in `mmu_macro_vars.cfg` → `mmu_vars.cfg` |
| **Drucker-Setup** | **Nur 1 Hotend → immer `T0`**, gekoppelt an wechselnde Gates (Tradrack 14-Kanal) |

**Problem vor der Änderung:**
Happy Hare purgierte blind nur das Extruder-Fragment (ca. 18mm = ~43mm³), ohne Farbe oder Material zu berücksichtigen. Nach Stromabschaltung war keinerlei Information über das zuletzt geladene Filament vorhanden.

---

## Lösung: Übersicht

```
Toolchange ausgelöst
        ↓
Happy Hare unloaded altes Filament
        ↓
Happy Hare loaded neues Filament
        ↓
_MMU_PURGE (unser Override) wird aufgerufen
        ↓
  ┌─────────────────────────────────────────┐
  │ 1. Lese letztes Material+Farbe          │
  │    aus mmu_vars.cfg (persistent)        │
  │                                         │
  │ 2. Lese neues Material+Farbe            │
  │    aus printer.mmu.gate_material/color  │
  │    INDIZIERT ÜBER printer.mmu.gate!     │
  │                                         │
  │ 3. Berechne Aufschläge:                 │
  │    - Basis: Happy Hare intern           │
  │    - Farbe: Helligkeitsvergleich        │
  │    - Material: PLA/PETG Aufschlag       │
  │                                         │
  │ 4. Purge mit Gesamtvolumen              │
  │                                         │
  │ 5. Nozzle wischen (X336→328→336)        │
  │                                         │
  │ 6. Speichere neues Material+Farbe       │
  │    in mmu_vars.cfg (persistent)         │
  └─────────────────────────────────────────┘
```

---

## ⚠️ Bugfix-Historie (wichtig für künftige Updates!)

### Bug: `mat_to`/`col_to` immer leer

**Symptom:**
```
PURGE-CALC: PETG/FFFFFF nach / | Basis:43mm3 + Farbe:0mm3 + Material:0mm3 = GESAMT:43mm3
MMU: Nozzle-Status gespeichert - Material:  / Farbe:
```
Trotz Happy Hare's klarer Meldung `T0 (gate 10, PLA, 9a9996, 220°C)` blieben Material und Farbe leer.

**Ursache:**
`printer.mmu.gate_material[]` und `printer.mmu.gate_color[]` sind **Arrays, die über die Gate-Nummer indiziert werden** – nicht über die Tool-Nummer. Bei diesem Drucker gibt es nur **ein Hotend, also immer `T0`**, das aber an wechselnde Gates gekoppelt wird (z.B. Gate 10 für PLA grau). Der ursprüngliche Code griff fälschlich mit `printer.mmu.tool` (= 0) auf das Array zu → `gate_material[0]` ist leer, weil Gate 0 kein Filament enthält.

Ein Zwischenschritt mit `idx = gate if tool < 0 else tool` half ebenfalls nicht, da `tool` zum Purge-Zeitpunkt bereits `0` (nicht `-1`) war und damit immer der Tool-Zweig genommen wurde.

**Fix:** Ausschließlich `printer.mmu.gate` zur Indizierung verwenden, `printer.mmu.tool` komplett ignorieren:

```jinja2
{% set gate   = printer.mmu.gate | default(0) | int %}
{% set mat_to = printer.mmu.gate_material[gate] | default("") | upper %}
{% set col_to = printer.mmu.gate_color[gate]    | default("") | upper %}
```

**Verifiziert am 23:33:36 Uhr:**
```
PURGE-CALC: PETG/FFFFFF nach PLA/8F8E8E | Basis:43mm3 + Farbe:60mm3 + Material:100mm3 = GESAMT:203mm3
MMU: Nozzle-Status gespeichert - Material: PLA / Farbe: 8F8E8E
```
✓ Material- und Farbinfo werden korrekt gelesen und persistiert.

> **Merke für künftige Setups mit mehreren Gates pro Tool:** `gate_material`, `gate_color` und alle anderen Gate-bezogenen Arrays in Happy Hare sind immer über `printer.mmu.gate` zu indizieren, niemals über `printer.mmu.tool`. Das gilt unabhängig davon, ob `force_purge_standalone` aktiv ist.

---

## Purge-Volumen Logik

### Basis-Volumen
Wird von Happy Hare intern berechnet (basierend auf Farbdifferenz der Gates aus Spoolman + Extruder-Fragment-Rest). Typisch: ~43mm³ bei gleichem Material/Farbe.

### Farb-Aufschlag
Berechnung über Helligkeitsformel (Luminanz): `(R×299 + G×587 + B×114) / 1000`

| Von → Nach | Schwellwert | Aufschlag |
|---|---|---|
| Dunkel (<60) → Hell (>150) | z.B. schwarz→weiß | **+120mm³** |
| Hell (>150) → Dunkel (<60) | z.B. weiß→schwarz | **+30mm³** |
| Farbe → andere Farbe | alle anderen | **+60mm³** |
| Gleiche Farbe | — | **+0mm³** |

### Material-Aufschlag
Nur wenn sich die Materialart ändert:

| Von → Nach | Aufschlag |
|---|---|
| PETG → PLA | **+100mm³** |
| PLA → PETG | **+60mm³** |
| Andere Kombi | **+80mm³** |
| Gleiches Material | **+0mm³** |

### Beispiele aus dem Test

| Wechsel | Basis | Farbe | Material | Gesamt | Status |
|---|---|---|---|---|---|
| PLA weiß → PLA schwarz | 43mm³ | +30mm³ | +0mm³ | **73mm³** | ✓ (Vortest) |
| PETG/FFFFFF → PLA/8F8E8E | 43mm³ | +60mm³ | +100mm³ | **203mm³** | ✓ Live verifiziert |

---

## Geänderte Dateien

### 1. `mmu/optional/mmu_eigene_macros.cfg`
**Aktion:** Folgenden Block ans Ende der Datei anhängen (Stand: korrigierte Version mit Gate-Indizierung).

```ini
#####################################################################
# Persistente Nozzle-Info & Intelligente Purge-Volumen Berechnung
#####################################################################

# Speichert das zuletzt in der Nozzle geladene Material + Farbe
# persistent in mmu_vars.cfg (überlebt Stromabschaltung via Tasmota)
[gcode_macro _MMU_SAVE_NOZZLE_STATE]
description: Speichert aktuelles Nozzle-Material und -Farbe persistent
gcode:
    # WICHTIG: gate_material/gate_color sind über GATE indiziert, nicht über TOOL!
    # Bei diesem Drucker (1 Hotend, mehrere Gates) ist tool immer 0 -> NICHT verwenden.
    {% set gate = printer.mmu.gate | default(0) | int %}
    {% set mat  = printer.mmu.gate_material[gate] | default("") | upper %}
    {% set col  = printer.mmu.gate_color[gate]    | default("") | upper %}
    SAVE_VARIABLE VARIABLE=nozzle_material VALUE='"{mat}"'
    SAVE_VARIABLE VARIABLE=nozzle_color    VALUE='"{col}"'
    {action_respond_info("MMU: Nozzle-Status gespeichert - Material: %s / Farbe: %s" % (mat, col))}


# Nozzle wischen nach dem Purge
[gcode_macro _MMU_NOZZLE_WIPE]
description: Wischt die Nozzle über die Silikonlippe (X336 nach X328 nach X336)
gcode:
    {action_respond_info("MMU: Nozzle wischen...")}
    G91
    G1 X-8 F6000
    G1 X8  F6000
    G90
    {action_respond_info("MMU: Nozzle gewischt.")}


# Override von mmu_purge.cfg:
# Berechnet Purge-Volumen direkt inline (Farbe + Material + Basis von Happy Hare)
[gcode_macro _MMU_PURGE]
description: Filament purge mit intelligentem Volumen (Farbe + Material)
gcode:
    # Happy Hare retraction settings from sequence macros
    {% set sequence_vars = printer['gcode_macro _MMU_SEQUENCE_VARS'] %}
    {% set park_vars = printer['gcode_macro _MMU_PARK'] %}
    {% set retracted_length = park_vars.retracted_length %}
    {% set retract_speed = sequence_vars.retract_speed | int %}
    {% set unretract_speed = sequence_vars.unretract_speed | int %}

    # Letztes Nozzle-Material + Farbe (persistent aus mmu_vars.cfg)
    {% set svv = printer.save_variables.variables %}
    {% set mat_from = svv.nozzle_material | default("") | upper %}
    {% set col_from = svv.nozzle_color    | default("") | upper %}

    # Neues Material + Farbe – WICHTIG: immer über GATE indizieren, nie über TOOL
    # (1 Hotend = immer T0, aber das Gate wechselt je nach Filament)
    {% set gate   = printer.mmu.gate | default(0) | int %}
    {% set mat_to = printer.mmu.gate_material[gate] | default("") | upper %}
    {% set col_to = printer.mmu.gate_color[gate]    | default("") | upper %}

    # Basis: Happy Hare's berechnetes Volumen
    {% set base_vol = printer.mmu.toolchange_purge_volume | default(50) | float %}
    {% set extruder_filament_remaining = printer.mmu.extruder_filament_remaining | default(0) | float %}

    # Farb-Aufschlag berechnen (inline, kein separates Makro noetig)
    {% set color_extra = 0 %}
    {% if col_from != col_to and col_from != "" and col_to != "" %}
        {% set hex_chars = "0123456789ABCDEF" %}
        {% if col_from | length >= 6 %}
            {% set rf = hex_chars.index(col_from[0])*16 + hex_chars.index(col_from[1]) %}
            {% set gf = hex_chars.index(col_from[2])*16 + hex_chars.index(col_from[3]) %}
            {% set bf = hex_chars.index(col_from[4])*16 + hex_chars.index(col_from[5]) %}
            {% set bright_from = ((rf*299 + gf*587 + bf*114) / 1000) | int %}
        {% else %}
            {% set bright_from = 128 %}
        {% endif %}
        {% if col_to | length >= 6 %}
            {% set rt = hex_chars.index(col_to[0])*16 + hex_chars.index(col_to[1]) %}
            {% set gt = hex_chars.index(col_to[2])*16 + hex_chars.index(col_to[3]) %}
            {% set bt = hex_chars.index(col_to[4])*16 + hex_chars.index(col_to[5]) %}
            {% set bright_to = ((rt*299 + gt*587 + bt*114) / 1000) | int %}
        {% else %}
            {% set bright_to = 128 %}
        {% endif %}
        {% if bright_from < 60 and bright_to > 150 %}
            {% set color_extra = 120 %}
        {% elif bright_from > 150 and bright_to < 60 %}
            {% set color_extra = 30 %}
        {% else %}
            {% set color_extra = 60 %}
        {% endif %}
    {% endif %}

    # Material-Aufschlag berechnen
    {% set mat_extra = 0 %}
    {% if mat_from != mat_to and mat_from != "" and mat_to != "" %}
        {% if "PETG" in mat_from and "PLA" in mat_to %}
            {% set mat_extra = 100 %}
        {% elif "PLA" in mat_from and "PETG" in mat_to %}
            {% set mat_extra = 60 %}
        {% else %}
            {% set mat_extra = 80 %}
        {% endif %}
    {% endif %}

    # Gesamtvolumen
    {% set toolchange_purge_volume = (base_vol + color_extra + mat_extra) | int %}

    {action_respond_info(
        "PURGE-CALC: %s/%s nach %s/%s | Basis:%dmm3 + Farbe:%dmm3 + Material:%dmm3 = GESAMT:%dmm3" % (
        mat_from, col_from, mat_to, col_to,
        base_vol | int, color_extra, mat_extra, toolchange_purge_volume)
    )}

    # Filamentmenge berechnen
    {% set filament_diameter = printer.configfile.config.extruder.filament_diameter | float %}
    {% set filament_cross_section = (filament_diameter / 2) ** 2 * 3.1415 %}
    {% set purge_len = toolchange_purge_volume / filament_cross_section %}
    {% set segment_len = 2.0 %}

    # Retraktion rueckgaengig machen
    {% if retracted_length > 0 %}
        MMU_LOG MSG="Un-retracting {retracted_length}mm"
        M83
        G1 E{retracted_length} F{unretract_speed | abs * 60}
    {% endif %}

    {% if extruder_filament_remaining > 0 %}
        MMU_LOG MSG="Purging {purge_len | round(1)}mm inkl. {extruder_filament_remaining | round(1)}mm Fragment (Gesamt {(filament_cross_section * purge_len) | round(1)}mm3)"
    {% else %}
        MMU_LOG MSG="Purging {purge_len | round(1)}mm (Gesamt {(filament_cross_section * purge_len) | round(1)}mm3)"
    {% endif %}

    # Purge in Segmenten
    {% set num_segments = (purge_len // segment_len) | int %}
    {% for _ in range(num_segments) %}
        __MMU_PURGE_SEGMENT LENGTH={segment_len}
    {% endfor %}
    __MMU_PURGE_SEGMENT LENGTH={purge_len % segment_len}

    # Retraktion wiederherstellen
    {% if retracted_length > 0 %}
        MMU_LOG MSG="Retracting {retracted_length}mm"
        M83
        G1 E-{retracted_length} F{retract_speed | abs * 60}
    {% endif %}

    # FlowGuard Spannung neutralisieren
    {% if printer.mmu.sync_feedback_enabled %}
        MMU_SYNC_FEEDBACK ADJUST_TENSION=1
    {% endif %}

    MMU_LOG MSG="Purging complete"

    # Nozzle wischen
    _MMU_NOZZLE_WIPE

    # Nozzle-Status fuer naechsten Druck speichern
    _MMU_SAVE_NOZZLE_STATE
```

---

### 2. `mmu/base/mmu_macro_vars.cfg`
**Aktion:** Keine Änderung notwendig.
`[save_variables]` war bereits vorhanden → `~/printer_data/config/mmu/mmu_vars.cfg`

---

### 3. `mmu/mmu_vars.cfg`
**Aktion:** Wird automatisch befüllt – keine manuelle Änderung mehr nötig.
Nach jedem Toolchange werden folgende Einträge automatisch aktualisiert, z.B.:

```ini
nozzle_color = 'PETG'
nozzle_material = 'FFFFFF'
```

> **Hinweis:** Diese Werte wurden während der Fehlersuche zwischenzeitlich manuell und versehentlich vertauscht eingetragen (`nozzle_color = 'PETG'` statt `nozzle_material`). Seit dem Bugfix schreibt `_MMU_SAVE_NOZZLE_STATE` die Werte wieder korrekt automatisch – keine manuelle Pflege mehr erforderlich.

---

### 4. `mmu/base/mmu_purge.cfg`
**Aktion:** Keine Änderung – READ ONLY.
Wird durch den `[gcode_macro _MMU_PURGE]` Override in `mmu_eigene_macros.cfg` überschattet.
Da `mmu_eigene_macros.cfg` nach `mmu_purge.cfg` geladen wird, gewinnt immer die eigene Version.

> **Wichtig bei Happy Hare Updates:** Falls `mmu_purge.cfg` durch ein Update geändert wird, muss geprüft werden ob relevante Änderungen in den eigenen `_MMU_PURGE` Override übernommen werden müssen. Insbesondere prüfen, ob sich die Struktur von `printer.mmu.gate_material` / `gate_color` ändert.

---

## Anpassbare Parameter

Alle Werte befinden sich direkt in `_MMU_PURGE` in `mmu_eigene_macros.cfg`:

| Variable | Aktueller Wert | Beschreibung |
|---|---|---|
| `color_extra` Dunkel→Hell | 120mm³ | Schwellwert: from<60, to>150 |
| `color_extra` Hell→Dunkel | 30mm³ | Schwellwert: from>150, to<60 |
| `color_extra` Farbe→Farbe | 60mm³ | Alle anderen Farbwechsel |
| `mat_extra` PETG→PLA | 100mm³ | Maximaler Materialaufschlag |
| `mat_extra` PLA→PETG | 60mm³ | Mittlerer Materialaufschlag |
| `mat_extra` Andere Kombi | 80mm³ | Unbekannte Materialkombination |
| Wischgeschwindigkeit | F6000 | In `_MMU_NOZZLE_WIPE` |
| Wischdistanz | 8mm (X-8) | In `_MMU_NOZZLE_WIPE` |

---

## Hardware

| Komponente | Detail |
|---|---|
| Purge-Position | X=336, Y=variabel (kein Y-Move beim Toolchange) |
| Purge-Bucket | Stationär an X=336, rechte Seite des Druckers |
| Silikonlippe | An Z-Achse montiert bei X≈328, volle Y-Länge |
| Wischbewegung | X=336 → X=328 → X=336 (reine X-Bewegung, 8mm) |
| Hotend-Konfiguration | 1 Hotend, immer `T0`, gekoppelt an wechselnde Gates (Tradrack 14-Kanal) |

---

## Testergebnisse

| Datum/Zeit | Wechsel | Erwartet | Erhalten | Status |
|---|---|---|---|---|
| Vortest | PLA weiß → PLA schwarz | Basis+30 | 43+30=73mm³ | ✓ |
| Vortest | PLA schwarz → PETG weiß | Basis+120+60 | 43+120+60=223mm³ | ✓ |
| 23:33:36 | PETG/FFFFFF → PLA/8F8E8E | Basis+60+100 | 43+60+100=203mm³ | ✓ Live im Druck |

---

## Bekannte Einschränkungen

- Helligkeitsschwellwerte (60/150) sind Schätzwerte – nach echten Drucktests ggf. anpassen
- Aufschlagswerte (120/60/30/100mm³) sind konservative Startwerte – im Betrieb beobachten
- Bei unbekannten Materialien (nicht PLA/PETG) wird ein generischer Aufschlag von 80mm³ verwendet
- Beim allerersten Start nach einer Neuinstallation sind `mat_from`/`col_from` leer → kein Aufschlag beim ersten Toolchange (einmalig akzeptabel)
- **Bei Multi-Gate-Setups mit nur einem Hotend (`T0` fix) niemals `printer.mmu.tool` zur Indizierung von `gate_material`/`gate_color` verwenden – immer `printer.mmu.gate`** (siehe Bugfix-Historie oben)
