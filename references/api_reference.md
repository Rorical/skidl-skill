# SKiDL API Reference

## Table of Contents
- [Part Class](#part-class)
- [PartUnit Class](#partunit-class)
- [Net Class](#net-class)
- [NCNet Class](#ncnet-class)
- [Pin Class](#pin-class)
- [Bus Class](#bus-class)
- [Circuit Class](#circuit-class)
- [Enumerations](#enumerations)
- [Top-Level Functions](#top-level-functions)
- [Key Exports](#key-exports)

---

## Part Class

`skidl.part.Part` — Stores a schematic part (electronic component).

### Constructor

```python
Part(lib=None, name=None, dest=NETLIST, tool=None, connections=None,
     part_defn=None, circuit=None, ref_prefix='', ref=None, tag=None,
     pin_splitters=None, **kwargs)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `lib` | str/SchLib | Library file name or SchLib object |
| `name` | str | Part name to find in library |
| `dest` | const | Destination: `NETLIST`, `TEMPLATE`, `LIBRARY` |
| `tool` | const | Library format: `KICAD`, `SKIDL`, `SPICE` |
| `connections` | dict | Pin-to-net mapping, e.g. `{'IN-': net_a, '1': net_b}` |
| `circuit` | Circuit | Owning circuit (default if None) |
| `ref_prefix` | str | Reference prefix (e.g., 'U', 'R') |
| `ref` | str | Specific reference designator |
| `tag` | str | Tag tying part to PCB footprint |
| `pin_splitters` | str | Characters splitting pin names into aliases |
| `**kwargs` | | Additional attributes (e.g., `value='1K'`, `footprint='...'`) |

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `ref` | str | Reference designator (auto-uniquified on set) |
| `value` | str | Part value, or name if unset |
| `foot` | str | Part footprint |
| `name` | str | Part name |
| `description` | str | Part description |
| `pins` | list | List of Pin objects |
| `unit` | dict | Dictionary of PartUnit subunits |
| `circuit` | Circuit | Owning circuit |
| `hiername` | str | Hierarchical name including prefix and tag |
| `partclasses` | set | All part classes from hierarchy + direct |
| `match_pin_regex` | bool | Enable regex matching for pin selection |
| `ordered_pins` | list | Pins sorted by number/name |

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `copy` | `(num_copies=None, dest=NETLIST, circuit=None, **attribs)` | Copy part. Also via `part()` or `N * part` |
| `get_pins` | `(*pin_ids, **criteria)` | Select pins by number/name/regex/attributes |
| `make_unit` | `(label, *pin_ids, **criteria)` | Create PartUnit from pin subset |
| `rmv_unit` | `(label)` | Remove a PartUnit |
| `is_connected` | `()` | True if part has connected pins |
| `attached_to` | `(nets=None)` | True if any pin connects to listed nets |
| `convert_for_spice` | `(spice_part, pin_map)` | Convert for SPICE simulation |
| `export` | `(addtl_part_attrs=None)` | Return string to recreate the Part |
| `add_pins` | `(*pins)` | Add pins to the part |
| `create_pins` | `(base_name, pin_count=None, connections=None)` | Create pins systematically |
| `disconnect` | `()` | Disconnect all pins from nets |
| `rename_pin` | `(pin_id, new_pin_name)` | Rename a pin |
| `renumber_pin` | `(pin_id, new_pin_num)` | Change a pin's number |
| `rmv_pins` | `(*pin_ids)` | Remove pins by name or number |
| `swap_pins` | `(pin_id1, pin_id2)` | Swap names and numbers of two pins |
| `get` | `(text, circuit=None)` | Classmethod: find part by ref/name/alias |
| `similarity` | `(part, **options)` | Measure similarity to another part |
| `validate` | `()` | Assert pins/units reference this part |

### Pin Access Patterns

```python
part[3]              # By number
part[3, 1, 6]        # Multiple numbers
part[2:4]            # Slice
part.p2              # Attribute (by number)
part['VDD']          # By name
part['GP0 GP1 GP2']  # Multiple names (space-separated)
part['DQ[0:2]']      # Name with index range
part['.*/AN1/.*']    # Regex (if match_pin_regex=True)
part.p['A1, A2']     # Pin-number accessor
part.n['A1, A2']     # Pin-name accessor
```

---

## PartUnit Class

`skidl.part.PartUnit` — A subset of pins from a parent Part.

### Constructor
```python
PartUnit(parent, label, *pin_ids, **criteria)
```

| Method | Description |
|--------|-------------|
| `add_pins_from_parent(*pin_ids, **criteria)` | Add more pins from parent |
| `grab_pins()` | Set pin references to this unit |
| `release_pins()` | Return pins to parent Part |

---

## Net Class

`skidl.net.Net` — Electrical connection between component pins.

### Constructor
```python
Net(name=None, circuit=None, *pins_nets_buses, **attribs)
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | str | Net name (auto-generated if None) |
| `drive` | int | Drive strength (max of connected pins) |
| `netclasses` | set | Net classes for PCB routing rules |
| `nets` | list | All connected net segments |
| `pins` | list | All attached pins |
| `stub` | bool | Stub status for schematic generation |
| `valid` | bool | Whether net is still valid |
| `width` | int | Always 1 for a Net |

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `connect` | `(*pins_nets_buses)` | Connect pins/nets/buses. Also via `+=` |
| `copy` | `(num_copies=None, circuit=None, **attribs)` | Copy net(s) |
| `disconnect` | `(pin)` | Remove a pin from net |
| `get` | `(name, circuit=None)` | Classmethod: find net by name/alias |
| `fetch` | `(name, *args, **attribs)` | Classmethod: get-or-create by name |
| `get_nets` | `()` | All connected net segments |
| `get_pins` | `()` | All connected pins |
| `is_attached` | `(pin_net_bus)` | Check if object is connected |
| `is_implicit` | `()` | True if name is auto-generated |
| `merge_names` | `()` | Choose best name for all segments |

---

## NCNet Class

`skidl.net.NCNet` — No-connect net. `drive` always returns `NOCONNECT`. Excluded from netlists.

---

## Pin Class

`skidl.pin.Pin` — Stores data about component pins.

### Constructor
```python
Pin(**attribs)
```

### Key Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `name` | str | Pin name (e.g., "GND") |
| `num` | str | Pin number (e.g., "1", "A5") |
| `func` | int | Pin function from `pin_types` enum |
| `nets` | list | Connected Net objects |
| `part` | Part | Parent Part |
| `do_erc` | bool | ERC checking flag |
| `aliases` | list | Alternative names |

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `drive` | int | Drive strength from `pin_drives` |
| `net` | Net | First connected Net (or None) |
| `ref` | str | Reference of parent part |
| `width` | int | Always 1 |

### Methods

| Method | Description |
|--------|-------------|
| `connect(*pins_nets_buses)` | Connect to nets/pins. Also via `+=` |
| `copy(num_copies=None, **attribs)` | Copy pin(s) |
| `disconnect()` | Disconnect from all nets |
| `is_connected()` | True if connected to normal net |
| `is_attached(pin_net_bus)` | True if attached to object |
| `move(net)` | Move pin to another net |

### Pin Functions (`Pin.funcs` / `pin_types`)

`INPUT`, `OUTPUT`, `BIDIR`, `TRISTATE`, `PASSIVE`, `UNSPEC`, `PWRIN`, `PWROUT`, `OPENCOLL`, `OPENEMIT`, `PULLUP`, `PULLDN`, `NOCONNECT`, `FREE`

---

## Bus Class

`skidl.bus.Bus` — A collection of related nets.

### Constructor
```python
Bus(name, width)           # Named bus with width
Bus(name, net1, net2, ...) # Bus from existing nets
Bus(name, 8, net, bus)     # Composite bus
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | str | Bus name |
| `width` | int | Number of nets |
| `netclasses` | set | Net classes |

### Methods

| Method | Description |
|--------|-------------|
| `connect(*pins_nets_buses)` | Connect. Also via `+=` |
| `copy(num_copies=None, **attribs)` | Copy bus(es) |
| `extend(*objects)` | Add nets to end (MSB) |
| `insert(index, *objects)` | Insert nets at index |
| `get(name, circuit=None)` | Classmethod: find by name |
| `fetch(name, *args, **attribs)` | Classmethod: get-or-create |
| `get_nets()` | List of nets in bus |

### Bus Indexing

```python
b[0]          # Single net
b[2, 4]       # Multiple nets
b[3:0]        # Slice (reversed)
b[-1]         # Last net
b['BUS_B.*']  # Regex match
```

---

## Circuit Class

`skidl.circuit.Circuit` — Central container for all circuit elements.

### Constructor
```python
Circuit(**attrs)
```

### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `parts` | list | All parts |
| `nets` | list | All nets |
| `buses` | list | All buses |
| `NC` | Net | No-connect net |

### Key Methods

| Method | Description |
|--------|-------------|
| `ERC(*args, **kwargs)` | Run Electrical Rule Checking |
| `generate_netlist(**kwargs)` | Generate netlist (`file_`, `tool`, `do_backup`) |
| `generate_pcb(**kwargs)` | Generate PCB file |
| `generate_schematic(**kwargs)` | Generate schematic |
| `generate_svg(file_=None, tool=None)` | SVG visualization |
| `generate_xml(file_=None, tool=None)` | XML representation |
| `generate_dot(file_=None, engine='neato', ...)` | Graphviz DOT graph |
| `add_parts(*parts)` | Add parts |
| `add_nets(*nets)` | Add nets |
| `add_buses(*buses)` | Add buses |
| `add_stuff(*stuff)` | Add parts/nets/buses/interfaces |
| `rmv_parts(*parts)` | Remove parts |
| `rmv_nets(*nets)` | Remove nets |
| `rmv_buses(*buses)` | Remove buses |
| `reset(init=False)` | Full reset |
| `backup_parts(file_=None)` | Save parts as SKiDL library |

### Context Manager Usage

```python
my_circuit = Circuit()
with my_circuit:
    p = Part('Device', 'R')
    n = Net('GND')
my_circuit.ERC()
my_circuit.generate_netlist()
```

---

## Enumerations

### `pin_types` (IntEnum) — Pin Functions
`INPUT`, `OUTPUT`, `BIDIR`, `TRISTATE`, `PASSIVE`, `UNSPEC`, `PWRIN`, `PWROUT`, `OPENCOLL`, `OPENEMIT`, `PULLUP`, `PULLDN`, `NOCONNECT`, `FREE`

### `pin_drives` (IntEnum) — Drive Strengths (weakest to strongest)
`NOCONNECT` < `NONE` < `PASSIVE` < `PULLUPDN` < `ONESIDE` < `TRISTATE` < `PUSHPULL` < `POWER`

---

## Top-Level Functions

These operate on the default circuit:

| Function | Description |
|----------|-------------|
| `ERC()` | Run Electrical Rule Checking |
| `generate_netlist(**kwargs)` | Generate netlist |
| `generate_pcb(**kwargs)` | Generate PCB file |
| `generate_schematic(**kwargs)` | Generate schematic |
| `generate_svg()` | SVG visualization |
| `generate_xml()` | XML output |
| `generate_dot()` | Graphviz DOT graph |
| `set_default_tool(tool)` | Set default ECAD tool |
| `get_default_tool()` | Get current default tool |
| `reset()` | Clear all circuitry and caches |
| `erc_assert(fail_msg, severity)` | Add ERC assertion |
| `search(term)` / `search_parts(term)` | Search for parts |
| `search_footprints(term)` | Search for footprints |
| `show(part)` / `show_part(part)` | Display part info |
| `show_footprint(fp)` | Display footprint info |
| `no_files()` | Suppress file output |

---

## Key Exports

| Export | Description |
|--------|-------------|
| `Part`, `SkidlPart`, `PartTmplt` | Component classes |
| `Pin` | Pin class |
| `Net` | Net class |
| `Bus` | Bus class |
| `Circuit` | Circuit container |
| `SchLib` | Library class |
| `SubCircuit`, `subcircuit`, `Group` | Hierarchy tools |
| `Interface` | Subcircuit interface |
| `Network`, `tee` | Network operations |
| `NetClass`, `PartClass` | Classification |
| `Alias` | Name aliasing |
| `Rgx` | Regex utilities |
| `NC` | No-connect constant |
| `NETLIST`, `TEMPLATE`, `LIBRARY` | Destination constants |
| `KICAD`, `SKIDL`, `SPICE` | Tool constants |
| `POWER` | Power drive constant |
| `lib_search_paths` | Library search paths dict |
| `footprint_search_paths` | Footprint search paths dict |
