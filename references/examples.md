# SKiDL Examples

## Table of Contents
- [Voltage Divider](#voltage-divider)
- [LED with Current-Limiting Resistor](#led-with-current-limiting-resistor)
- [Op-Amp Inverting Amplifier](#op-amp-inverting-amplifier)
- [555 Timer Astable](#555-timer-astable)
- [Arduino Shield](#arduino-shield)
- [Parametric Resistor Ladder](#parametric-resistor-ladder)
- [Hierarchical Power Supply](#hierarchical-power-supply)
- [Bus-Based Memory Interface](#bus-based-memory-interface)
- [Multiple Circuit Objects](#multiple-circuit-objects)
- [Ad-Hoc Part Creation](#ad-hoc-part-creation)

---

## Voltage Divider

```python
from skidl import *

vin, vout, gnd_net = Net('VI'), Net('VO'), Net('GND')

r1, r2 = 2 * Part("Device", 'R', dest=TEMPLATE)
r1.value = '1K'
r2.value = '500'

vin & r1 & vout & r2 & gnd_net

ERC()
generate_netlist(tool=KICAD8)
```

---

## LED with Current-Limiting Resistor

```python
from skidl import *

vcc, gnd_net = Net('VCC'), Net('GND')

r = Part('Device', 'R', value='330')
led = Part('Device', 'LED', footprint='LED_SMD:LED_0805_2012Metric')

vcc & r & led['A,K'] & gnd_net

ERC()
generate_netlist()
```

---

## Op-Amp Inverting Amplifier

```python
from skidl import *

opamp = Part('Amplifier_Operational', 'LM358', unit=1)
r_in = Part('Device', 'R', value='10K')
r_fb = Part('Device', 'R', value='100K')

vin, vout, gnd_net = Net('VIN'), Net('VOUT'), Net('GND')
vcc, vee = Net('VCC'), Net('VEE')

# Input through R_in to inverting input
vin & r_in & opamp['-']

# Feedback resistor
opamp['~'] & r_fb & opamp['-']

# Non-inverting input to ground
opamp['+'] += gnd_net

# Power
opamp['V+'] += vcc
opamp['V-'] += vee

# Output
opamp['~'] += vout

ERC()
generate_netlist()
```

---

## 555 Timer Astable

```python
from skidl import *

vcc, gnd_net, out = Net('VCC'), Net('GND'), Net('OUT')

timer = Part('Timer', 'NE555')
r1 = Part('Device', 'R', value='1K')
r2 = Part('Device', 'R', value='10K')
c1 = Part('Device', 'C', value='100nF')
c2 = Part('Device', 'C', value='10nF')

# Power
timer['VCC'] += vcc
timer['GND'] += gnd_net
timer['RESET'] += vcc

# Timing network
vcc & r1 & timer['DISCH'] & r2 & timer['TRIG,THR'] & c1 & gnd_net

# Control voltage bypass
timer['CONT'] += c2[1]
c2[2] += gnd_net

# Output
timer['OUT'] += out

ERC()
generate_netlist()
```

---

## Arduino Shield

```python
from skidl import *

# Define Arduino Uno header
arduino = Part('Connector_Generic', 'Conn_01x20', dest=TEMPLATE)
header = arduino(ref='J1')

# I2C sensor
sensor = Part('Sensor_Temperature', 'LM75B', footprint='Package_SO:SOIC-8_3.9x4.9mm_P1.27mm')

# Pull-ups for I2C
sda_pullup = Part('Device', 'R', value='4.7K')
scl_pullup = Part('Device', 'R', value='4.7K')

vcc, gnd_net = Net('VCC'), Net('GND')
sda, scl = Net('SDA'), Net('SCL')

# Connections
sensor['SDA'] += sda
sensor['SCL'] += scl
sensor['VCC'] += vcc
sensor['GND'] += gnd_net
sensor['A0, A1, A2'] += gnd_net  # Address = 0x48

# Pull-ups
vcc & sda_pullup & sda
vcc & scl_pullup & scl

ERC()
generate_netlist()
```

---

## Parametric Resistor Ladder

Algorithmic circuit generation:

```python
from skidl import *

def resistor_ladder(num_taps, r_value='1K'):
    """Create an R-2R DAC ladder."""
    inp = Net('DAC_IN')
    outp = Net('DAC_OUT')
    gnd_net = Net('GND')

    r_series = num_taps * Part('Device', 'R', value=r_value, dest=TEMPLATE)
    r_shunt = num_taps * Part('Device', 'R', value=f'{int(r_value[:-1])*2}{r_value[-1]}', dest=TEMPLATE)

    prev_net = inp
    for i in range(num_taps):
        tap = Net(f'TAP_{i}')
        prev_net & r_series[i] & tap
        tap & r_shunt[i] & gnd_net
        prev_net = tap

    prev_net += outp
    return inp, outp

dac_in, dac_out = resistor_ladder(8)
ERC()
generate_netlist()
```

---

## Hierarchical Power Supply

Using `@SubCircuit` decorator:

```python
from skidl import *

@SubCircuit
def voltage_regulator(v_out_val):
    """3-terminal linear regulator subcircuit."""
    reg = Part('Regulator_Linear', 'AMS1117-3.3')
    c_in = Part('Device', 'C', value='100nF')
    c_out = Part('Device', 'C', value='10uF')

    reg['VI'] & c_in[1]
    c_in[2] & reg['GND']
    reg['VO'] & c_out[1]
    c_out[2] & reg['GND']

    return Interface(
        vin=reg['VI'],
        vout=reg['VO'],
        gnd=reg['GND']
    )

@SubCircuit
def power_supply():
    """Full power supply with multiple rails."""
    reg_3v3 = voltage_regulator('3.3V')
    reg_1v8 = voltage_regulator('1.8V')

    vin = Net('VIN_12V')
    gnd_net = Net('GND')

    vin += reg_3v3.vin
    reg_3v3.vout += reg_1v8.vin
    reg_3v3.gnd += gnd_net
    reg_1v8.gnd += gnd_net

    return Interface(
        vin=vin,
        v3v3=reg_3v3.vout,
        v1v8=reg_1v8.vout,
        gnd=gnd_net
    )

# Top-level
psu = power_supply()
vin_connector = Net('VIN')
vin_connector += psu.vin

ERC()
generate_netlist()
```

---

## Bus-Based Memory Interface

```python
from skidl import *

# SRAM chip
sram = Part('Memory_RAM', 'AS6C1616')

# MCU (simplified)
mcu = Part('MCU_ST', 'STM32F103C8Tx')

# Address and data buses
addr_bus = Bus('ADDR', 16)
data_bus = Bus('DATA', 16)

# Control signals
cs_n, oe_n, we_n = Net('CS_N'), Net('OE_N'), Net('WE_N')

# Connect buses
sram['A[0:15]'] += addr_bus[0:15]
sram['DQ[0:15]'] += data_bus[0:15]

# Control
sram['CE'] += cs_n
sram['OE'] += oe_n
sram['WE'] += we_n

# Power
vcc, gnd_net = Net('VCC'), Net('GND')
sram['VCC'] += vcc
sram['GND'] += gnd_net

ERC()
generate_netlist()
```

---

## Multiple Circuit Objects

```python
from skidl import *

# Create isolated circuits
analog_ckt = Circuit(name='analog')
digital_ckt = Circuit(name='digital')

# Add parts to specific circuits
with analog_ckt:
    opamp = Part('Amplifier_Operational', 'LM358')
    r1 = Part('Device', 'R', value='10K')
    r2 = Part('Device', 'R', value='100K')
    # ... connect ...

with digital_ckt:
    mcu = Part('MCU_ST', 'STM32F103C8Tx')
    cap = Part('Device', 'C', value='100nF')
    # ... connect ...

# Generate separate netlists
analog_ckt.generate_netlist(file_='analog.net')
digital_ckt.generate_netlist(file_='digital.net')
```

---

## Ad-Hoc Part Creation

Create parts without a library:

```python
from skidl import *

# Create a custom connector
conn = Part(name='MY_CONN', tool=SKIDL, dest=TEMPLATE,
            ref_prefix='J', description='Custom 4-pin connector')
conn += Pin(num=1, name='VCC', func=Pin.funcs.PWRIN)
conn += Pin(num=2, name='SDA', func=Pin.funcs.BIDIR)
conn += Pin(num=3, name='SCL', func=Pin.funcs.BIDIR)
conn += Pin(num=4, name='GND', func=Pin.funcs.PWRIN)

# Instantiate
j1 = conn()
j1['VCC'] += Net('VCC')
j1['GND'] += Net('GND')
j1['SDA'] += Net('SDA')
j1['SCL'] += Net('SCL')

ERC()
generate_netlist()
```

---

## Network Operators Quick Reference

```python
# Series: use &
vcc & r1 & r2 & gnd

# Parallel: use |
vcc & (r1 | r2) & gnd

# Explicit pin order
vcc & r1 & d1['A,K'] & gnd

# Tee junction
from skidl import tee
inp & tee(c_shunt & gnd) & l_series & tee(c_load & gnd) & outp

# Multiple copies
r1, r2, r3, r4 = Part('Device', 'R', dest=TEMPLATE) * 4
r1, r2, r3 = Part('Device', 'R', dest=TEMPLATE)(3, value='1K')
```
