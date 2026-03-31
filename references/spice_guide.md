# SKiDL SPICE Simulation Guide

## Table of Contents
- [Setup](#setup)
- [Basic Workflow](#basic-workflow)
- [Units](#units)
- [Available SPICE Parts](#available-spice-parts)
- [Simulation Types](#simulation-types)
- [Extracting Results](#extracting-results)
- [Controlled Sources](#controlled-sources)
- [Behavioral Sources](#behavioral-sources)
- [XSPICE Parts](#xspice-parts)
- [Using External Subcircuits](#using-external-subcircuits)
- [Converting KiCad Parts to SPICE](#converting-kicad-parts-to-spice)
- [Complete Examples](#complete-examples)

---

## Setup

```python
from skidl.pyspice import *
```

This import auto-creates a `gnd` net, sets SPICE tool mode, and injects all SPICE part constructors (R, C, V, BJT, etc.) into the namespace.

---

## Basic Workflow

```python
from skidl.pyspice import *

# 1. Create parts and connect them
vin = V(ref='V1', dc_value=10 @ u_V)
r1 = R(ref='R1', value=1 @ u_kOhm)
r2 = R(ref='R2', value=1 @ u_kOhm)

# 2. Connect circuit
vin['p'] & r1[1] & r2[1]
r2[2] & gnd
vin['n'] & gnd

# Connect r1[2] to r2[1] is already done via series above
# Or explicitly:
# vin['p'] += r1[1]
# r1[2] += r2[1]
# r2[2] += gnd
# vin['n'] += gnd

# 3. Generate netlist and simulate
circ = generate_netlist()
sim = circ.simulator()
analysis = sim.operating_point()

# 4. Extract results
print(float(analysis.nodes['v_out']))
```

---

## Units

Use the `@` operator with unit constants:

```python
10 @ u_V          # 10 Volts
1 @ u_kOhm        # 1 kilo-Ohm
100 @ u_Ohm        # 100 Ohms
1 @ u_uF           # 1 micro-Farad
10 @ u_nF          # 10 nano-Farads
1 @ u_pF           # 1 pico-Farad
1 @ u_uH           # 1 micro-Henry
1 @ u_mH           # 1 milli-Henry
1 @ u_mA           # 1 milli-Amp
1 @ u_us           # 1 microsecond
1 @ u_ms           # 1 millisecond
1 @ u_ns           # 1 nanosecond
1 @ u_kHz          # 1 kilo-Hertz
1 @ u_MHz          # 1 mega-Hertz
```

---

## Available SPICE Parts

### Passive Components
- `R(value=...)` — Resistor
- `C(value=...)` — Capacitor
- `L(value=...)` — Inductor

### Diodes & Transistors
- `D(model=...)` — Diode
- `BJT(model=...)` — Bipolar Junction Transistor (pins: B, C, E)
- `MOSFET(model=...)` — MOSFET (pins: G, D, S)
- `JFET(model=...)` — JFET
- `MESFET(model=...)` — MESFET

### Independent Sources
- `V(dc_value=...)` — DC Voltage source (pins: p, n)
- `I(dc_value=...)` — DC Current source (pins: p, n)
- `SINEV(amplitude=..., frequency=..., offset=...)` — Sinusoidal voltage
- `SINEI(...)` — Sinusoidal current
- `PULSEV(initial_value=..., pulsed_value=..., pulse_width=..., period=...)` — Pulse voltage
- `PULSEI(...)` — Pulse current
- `RAMPV(...)` — Ramp voltage
- `RAMPI(...)` — Ramp current
- `PWLV(...)` — Piecewise linear voltage
- `PWLI(...)` — Piecewise linear current

### Controlled Sources
- `VCVS(gain=...)` — Voltage-Controlled Voltage Source
- `VCCS(gain=...)` — Voltage-Controlled Current Source
- `CCVS(gain=...)` — Current-Controlled Voltage Source
- `CCCS(gain=...)` — Current-Controlled Current Source

### Other
- `B(...)` — Behavioral source
- `XSPICE(...)` — XSPICE component
- `TLINE(...)` — Transmission line
- `SW(...)` — Voltage-controlled switch
- `CSW(...)` — Current-controlled switch

---

## Simulation Types

```python
circ = generate_netlist()
sim = circ.simulator()

# Operating point
analysis = sim.operating_point()

# DC sweep
analysis = sim.dc(V1=slice(0, 10, 0.1))  # Sweep V1 from 0 to 10V, step 0.1V

# Transient
analysis = sim.transient(
    step_time=1 @ u_us,
    end_time=1 @ u_ms
)

# AC analysis
analysis = sim.ac(
    start_frequency=1 @ u_Hz,
    stop_frequency=1 @ u_MHz,
    number_of_points=100,
    variation='dec'
)
```

---

## Extracting Results

Use the `node()` function to get net names for result extraction:

```python
# After simulation
from skidl.pyspice import node

# Get voltage at a net
v_out_net = Net('v_out')
# ... connect circuit ...
analysis = sim.operating_point()
voltage = float(analysis.nodes[node(v_out_net)])

# For transient/DC analysis, results are arrays
import matplotlib.pyplot as plt
analysis = sim.transient(step_time=1@u_us, end_time=1@u_ms)
plt.plot(analysis.time, analysis.nodes[node(some_net)])
```

---

## Controlled Sources

### VCVS (Voltage-Controlled Voltage Source)
```python
e = VCVS(ref='E1', gain=10)
e['ip', 'in'] += control_p, control_n  # Control input
e['op', 'on'] += output_p, output_n    # Output
```

### CCCS (Current-Controlled Current Source)
```python
f = CCCS(ref='F1', gain=5)
# Control via a voltage source acting as ammeter
```

---

## Behavioral Sources

```python
b = B(ref='B1')
# Voltage defined by expression
b['p', 'n'] += out_p, out_n
```

---

## XSPICE Parts

```python
from skidl.pyspice import *

# ADC bridge
adc = XSPICE(ref='A1')
# DAC bridge
dac = XSPICE(ref='A2')
# Digital buffer
buf = XSPICE(ref='A3')
```

XSPICE parts require `io` and `model` parameters specific to the XSPICE component type.

---

## Using External Subcircuits

Load `.lib` or `.subckt` files:

```python
lib = SchLib('path/to/library.lib', tool=SPICE)
part = Part(lib, 'subcircuit_name')
```

For PDK models (e.g., Skywater 130nm):

```python
lib = SchLib('path/to/sky130.lib', tool=SPICE, lib_section='tt')
nfet = Part(lib, 'nfet_01v8', params=Parameters(W=1e-6, L=150e-9))
```

---

## Converting KiCad Parts to SPICE

Use `convert_for_spice()` to map KiCad parts to SPICE equivalents:

```python
# Create KiCad part
from skidl import *
r_kicad = Part('Device', 'R', value='1K')

# Create SPICE equivalent
from skidl.pyspice import R as SpiceR
r_spice = SpiceR(value=1 @ u_kOhm)

# Map pins
r_kicad.convert_for_spice(r_spice, {1: 1, 2: 2})
```

---

## Complete Examples

### Voltage Divider with DC Sweep

```python
from skidl.pyspice import *
import matplotlib.pyplot as plt

vin = V(ref='V1', dc_value=10 @ u_V)
r1 = R(ref='R1', value=1 @ u_kOhm)
r2 = R(ref='R2', value=1 @ u_kOhm)

v_in, v_out = Net('v_in'), Net('v_out')
vin['p'] += v_in
vin['n'] += gnd
v_in += r1[1]
r1[2] += v_out
v_out += r2[1]
r2[2] += gnd

circ = generate_netlist()
sim = circ.simulator()
analysis = sim.dc(V1=slice(0, 10, 0.1))
plt.plot([float(v) for v in analysis.sweep.values],
         [float(v) for v in analysis.nodes[node(v_out)]])
plt.xlabel('Vin (V)')
plt.ylabel('Vout (V)')
plt.show()
```

### RC Transient Analysis

```python
from skidl.pyspice import *
import matplotlib.pyplot as plt

vs = PULSEV(ref='V1', initial_value=0@u_V, pulsed_value=5@u_V,
            pulse_width=500@u_us, period=1@u_ms)
r = R(ref='R1', value=1@u_kOhm)
c = C(ref='C1', value=1@u_uF)

v_in, v_out = Net('v_in'), Net('v_out')
vs['p'] += v_in
vs['n'] += gnd
v_in += r[1]
r[2] += v_out
v_out += c[1]
c[2] += gnd

circ = generate_netlist()
sim = circ.simulator()
analysis = sim.transient(step_time=1@u_us, end_time=2@u_ms)

plt.plot(analysis.time, analysis.nodes[node(v_out)])
plt.xlabel('Time (s)')
plt.ylabel('Voltage (V)')
plt.title('RC Step Response')
plt.show()
```

### BJT Common-Emitter Amplifier

```python
from skidl.pyspice import *

vin = SINEV(ref='V1', offset=0@u_V, amplitude=10@u_mV, frequency=1@u_kHz)
vcc = V(ref='VCC', dc_value=12@u_V)
q = BJT(ref='Q1', model='2N2222')
rc = R(ref='RC', value=4.7@u_kOhm)
rb = R(ref='RB', value=100@u_kOhm)
re = R(ref='RE', value=1@u_kOhm)
c_in = C(ref='CIN', value=10@u_uF)
c_out = C(ref='COUT', value=10@u_uF)

supply, v_in, v_base, v_col, v_out = (
    Net('supply'), Net('v_in'), Net('v_base'), Net('v_col'), Net('v_out')
)

vcc['p'] += supply
vcc['n'] += gnd
supply += rc[1]
rc[2] += v_col
v_col += q['C']
q['E'] += re[1]
re[2] += gnd
supply += rb[1]
rb[2] += v_base
v_base += q['B']
vin['p'] += v_in
vin['n'] += gnd
v_in += c_in[1]
c_in[2] += v_base
v_col += c_out[1]
c_out[2] += v_out

circ = generate_netlist()
sim = circ.simulator()
analysis = sim.transient(step_time=1@u_us, end_time=5@u_ms)
```
