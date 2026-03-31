# Component Search & Sourcing Guide

## Table of Contents
- [Local Part Search (SKiDL Built-in)](#local-part-search-skidl-built-in)
- [KiCad Default Libraries](#kicad-default-libraries)
- [Online Symbol/Footprint Downloads](#online-symbolfootprint-downloads)
- [Distributor APIs for Specs & Pricing](#distributor-apis-for-specs--pricing)
- [LCSC / JLCPCB Parts](#lcsc--jlcpcb-parts)
- [BOM Pricing from Netlists](#bom-pricing-from-netlists)
- [Companion Tools](#companion-tools)
- [Recommended Workflows](#recommended-workflows)

---

## Local Part Search (SKiDL Built-in)

SKiDL searches **local KiCad `.kicad_sym` files only** — no online integration.

### CLI

```bash
skidl-part-search opamp              # Keyword search
skidl-part-search --interactive      # Interactive mode
```

### Python

```python
from skidl import *

search('opamp')                      # Regex across names, descriptions, keywords
search('opamp dual')                 # AND logic (both terms must match)
search('lm358 | tl072')             # OR logic
search_footprints('QFP')            # Footprint search
show(Part('Amplifier_Operational', 'LM358'))  # Display part details
```

### Search Internals

SKiDL builds an SQLite cache of all parts from libraries in `lib_search_paths[KICAD]`. It indexes part names, aliases, descriptions, and keywords into a `search_text` field, then matches with regex.

### Configuring Library Paths

```python
from skidl import lib_search_paths, KICAD

# Add local directory
lib_search_paths[KICAD].append('/path/to/downloaded/libs')

# Add cloned KiCad official repo
lib_search_paths[KICAD].append('/home/user/kicad-symbols')

# Remote (experimental)
lib_search_paths[KICAD].append('https://gitlab.com/kicad/libraries/kicad-symbols/-/raw/master')
```

---

## KiCad Default Libraries

KiCad ships ~200 symbol libraries organized **by function** (per KiCad Library Conventions). Default location depends on OS:

- **Linux**: `/usr/share/kicad/symbols/`
- **macOS**: Inside KiCad.app bundle, typically `/Applications/KiCad/KiCad.app/Contents/SharedSupport/symbols/`
- **Windows**: `C:\Program Files\KiCad\<version>\share\kicad\symbols\`

### Key Library Categories

| Category | Libraries | Examples |
|----------|-----------|----------|
| **Passives** | `Device` | R, C, L, LED, D, Crystal |
| **Op-amps** | `Amplifier_Operational` | LM358, TL072, OPA2134 |
| **Regulators** | `Regulator_Linear`, `Regulator_Switching` | AMS1117, LM7805, LM2596 |
| **MCUs** | `MCU_ST_STM32*`, `MCU_Microchip_*`, `MCU_Espressif` | STM32F103, ATmega328P, ESP32 |
| **Memory** | `Memory_RAM`, `Memory_Flash`, `Memory_EEPROM` | AS6C1616, W25Q128 |
| **Connectors** | `Connector_Generic`, `Connector` | Conn_01x08, USB_B_Micro |
| **Timers** | `Timer`, `Timer_RTC`, `Timer_PLL` | NE555, DS3231 |
| **Sensors** | `Sensor_Temperature`, `Sensor_Motion`, etc. | LM75B, MPU6050 |
| **Transistors** | `Transistor_BJT`, `Transistor_FET` | 2N2222, IRLZ44N |
| **Diodes** | `Diode`, `Diode_Bridge` | 1N4148, BAT54 |
| **Logic** | `74xx`, `4xxx` | 74HC595, CD4017 |
| **Interfaces** | `Interface_USB`, `Interface_CAN_LIN`, `Interface_UART` | FT232R, MCP2515 |
| **RF** | `RF_Module`, `RF_WiFi`, `RF_Bluetooth` | ESP-12E, HC-05 |
| **Power** | `Power_Protection`, `Power_Supervisor` | TVS diodes, voltage supervisors |
| **FPGAs** | `FPGA_Xilinx_*`, `FPGA_Lattice` | XC7A35T, ICE40UP5K |
| **Drivers** | `Driver_LED`, `Driver_Motor`, `Driver_FET` | WS2812B driver, DRV8825 |

Official library repo: https://gitlab.com/kicad/libraries/kicad-symbols

---

## Online Symbol/Footprint Downloads

When the part you need isn't in KiCad's default libraries, download symbols and footprints from these sources:

| Source | Parts | Format | Notes |
|--------|-------|--------|-------|
| **SnapEDA** | 500K+ | `.kicad_sym` + `.kicad_mod` | KiCad plugin available; IPC-7351B footprints |
| **Ultra Librarian** | 16M+ | KiCad export | Includes 3D models |
| **SamacSys / Component Search Engine** | Large | KiCad via Library Loader | Auto-installs into KiCad |
| **Digi-Key KiCad Library** | Curated | Atomic parts | Pre-mapped symbol + footprint + 3D |

### Using Downloaded Libraries with SKiDL

```python
# After downloading a .kicad_sym file from SnapEDA/Ultra Librarian:
lib_search_paths[KICAD].append('/path/to/downloaded/libs')
part = Part('downloaded_lib', 'TPS63020')

# Or reference the file directly:
part = Part('/path/to/TPS63020.kicad_sym', 'TPS63020')
```

---

## Distributor APIs for Specs & Pricing

### Nexar / Octopart (recommended for parametric search)

Largest aggregated component database. GraphQL API with 1,000 free calls/month.

- **Sign up**: https://nexar.com/api
- **Auth**: OAuth2 (client ID + secret)
- **Python**: `pip install nsg-parts-search`

```python
# Conceptual example — requires OAuth2 token
import requests

query = """
query {
  supSearch(q: "LM358 op-amp", limit: 5) {
    results {
      part {
        mpn
        manufacturer { name }
        bestDatasheet { url }
        specs { attribute { name } displayValue }
        medianPrice1000 { price currency }
      }
    }
  }
}
"""
response = requests.post("https://api.nexar.com/graphql",
    headers={"Authorization": f"Bearer {token}"},
    json={"query": query})
```

Supports parametric filtering (e.g., "resistor with tolerance < 1% and package 0402").

### Digi-Key API

- **Sign up**: https://developer.digikey.com/
- **Auth**: OAuth2
- **Python**: `pip install digikey-apiv4`

```python
import digikey
from digikey.v3.productinformation import KeywordSearchRequest

search_request = KeywordSearchRequest(keywords='LM358', record_count=10)
result = digikey.keyword_search(body=search_request)
for p in result.products:
    print(p.manufacturer_part_number, p.unit_price, p.quantity_available)
```

### Mouser API

- **Sign up**: https://www.mouser.com/api-hub/
- **Python**: `pip install mouser`

```python
import mouser
mouser.api.set_api_key('YOUR_KEY')
results = mouser.api.search_keyword('LM358')
```

---

## LCSC / JLCPCB Parts

LCSC has no official API, but there are tools to bridge the gap:

### easyeda2kicad (convert LCSC parts to KiCad)

```bash
pip install easyeda2kicad

# Convert a single LCSC part to KiCad symbol + footprint + 3D model
easyeda2kicad --lcsc_id C7947 --output ./my_libs
# Creates .kicad_sym, .kicad_mod, and .wrl files
```

Then use in SKiDL:
```python
lib_search_paths[KICAD].append('./my_libs')
part = Part('my_libs', 'LM358')
```

### JLCPCB Parts CSV

JLCPCB publishes a CSV of their parts library at https://jlcpcb.com/parts — useful for filtering by "basic" vs "extended" parts for assembly cost optimization.

---

## BOM Pricing from Netlists

### KiCost

Generates pricing spreadsheets from SKiDL-generated netlists by querying Digi-Key, Mouser, Newark, Farnell, LCSC, and others.

```bash
pip install kicost

# Generate netlist from SKiDL
python my_circuit.py  # outputs circuit.xml

# Run KiCost to get pricing
kicost -i circuit.xml -o bom_pricing.xlsx
```

The output spreadsheet shows per-part pricing at various quantities across distributors.

### KiBOM

Configurable BOM generator for KiCad netlists — useful for grouping, filtering, and formatting BOMs.

```bash
pip install kibom
```

---

## Companion Tools

| Tool | Install | Purpose |
|------|---------|---------|
| **zyc** | `pip install zyc` | GUI for searching KiCad libs, generates `Part(...)` code for SKiDL |
| **eseries** | `pip install eseries` | Select standard resistor/capacitor values from E3–E192 series |
| **easyeda2kicad** | `pip install easyeda2kicad` | Convert LCSC part numbers to KiCad symbols + footprints |
| **KiCost** | `pip install kicost` | Auto-price BOMs from netlists across distributors |

---

## Recommended Workflows

### Standard design (KiCad libraries available)

1. Search local libs: `search('voltage regulator 3.3')` or use `zyc`
2. Instantiate: `Part('Regulator_Linear', 'AMS1117-3.3')`
3. Set footprint: `part.footprint = 'Package_TO_SOT_SMD:SOT-223-3_TabPin2'`
4. Generate netlist → price with KiCost

### Part not in KiCad libraries

1. Search SnapEDA or Ultra Librarian for the part
2. Download `.kicad_sym` + `.kicad_mod`
3. Add download directory to `lib_search_paths[KICAD]`
4. Use in SKiDL as normal

### JLCPCB assembly workflow

1. Design circuit in SKiDL using generic parts from `Device` library
2. Use `easyeda2kicad --lcsc_id <id>` to get KiCad-compatible symbols for specific LCSC parts
3. Set `footprint` attributes matching JLCPCB's library
4. Generate netlist and BOM for JLCPCB upload

### Finding parts by specs (parametric search)

1. Use Nexar/Octopart API to search by parameters (e.g., "op-amp, GBW > 10MHz, SOIC-8")
2. Get the manufacturer part number (MPN)
3. Search local KiCad libs for it, or download from SnapEDA/Ultra Librarian
4. Instantiate in SKiDL
