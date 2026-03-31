# SKiDL Skill

A Claude Code skill for designing electronic circuits using the [SKiDL](https://github.com/devbisme/skidl) Python library.

## What This Skill Does

This skill enables Claude to assist with programmatic electronic circuit design using SKiDL — a Python module that replaces graphical schematic editors with code. It covers:

- **Circuit design** — Creating electronic circuits in Python using Parts, Nets, Buses, and Pins
- **Netlist generation** — Outputting netlists for PCB layout tools (primarily KiCad)
- **Schematic generation** — Producing KiCad schematics programmatically
- **SPICE simulation** — Running circuit simulations via PySpice/InSpice
- **Component search** — Finding parts and footprints in KiCad libraries
- **Hierarchical design** — Building reusable subcircuits with the `@SubCircuit` decorator
- **Parametric design** — Algorithmically generating circuits (e.g., R-2R ladders, filter chains)

## Skill Structure

```text
skidl-skill/
├── SKILL.md                       # Core instructions and quick reference
└── references/
    ├── api_reference.md           # Full API docs (Part, Net, Bus, Pin, Circuit, etc.)
    ├── spice_guide.md             # SPICE simulation setup, parts, and examples
    └── examples.md                # Complete working circuits (LED, op-amp, 555, buses, etc.)
```

## Installation

Copy or symlink this skill into your Claude Code skills directory, or install the packaged `.skill` file.

## Usage

Once installed, Claude will automatically activate this skill when you ask about electronic circuit design with Python, SKiDL APIs, netlist generation, SPICE simulation, or related EDA tasks.

Example prompts:

- "Design a voltage divider circuit using SKiDL"
- "Create a 555 timer astable circuit and generate a KiCad netlist"
- "Simulate an RC low-pass filter with a transient analysis"
- "Build a hierarchical power supply with 3.3V and 1.8V rails"
