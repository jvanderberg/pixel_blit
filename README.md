# pixel_blit

A  32-channel addressable LED controller based on the Raspberry Pi RP2350B microcontroller.

## Overview

The pixel_blit is designed for controlling large-scale addressable LED installations. It features LVDS networking for daisy-chaining multiple controllers across long cable runs, making it ideal for large lighting installations.

## Key Specifications

- **Microcontroller**: Raspberry Pi RP2350B (QFN-80)
  - Dual Cortex-M33 cores @ 150 MHz
  - 8 PIO state machines for precise LED timing
  - USB High Speed (480 Mbps)

- **Output Channels**: 32 independent LED string outputs
  - 47Ω series resistors on all outputs
  - 3.3V to 5V level shifting
  - Screw terminal connections (3.5mm pitch)

- **Memory**:
  - 16MB QSPI Flash (W25Q128JVSIQTR)
  - MicroSD card slot for pattern storage

- **Power Management**:
  - 12V input via terminal blocks
  - TPS63070 buck-boost converter (12V → 5V, 2A)
  - ME6217C33 LDO regulator (5V → 3.3V, 300mA)

- **Networking**:
  - LVDS daisy-chaining via RJ45 connectors
  - 4-channel differential signaling (DSLVDS1047/1048)
  - 100Ω termination resistors
  - Supports long cable runs

- **User Interface**:
  - MicroSD card slot
  - 4 tactile buttons + 1 reset button
  - 4-position DIP switch
  - I2C display interface

## Board Details

- **PCB Size**: 175.5mm × 122.0mm
- **Layers**: 4-layer stackup (Signal/GND/Power/Signal)
- **Components**: 192 total (all SMT, single-sided assembly)
- **Connectors**:
  - 32× DB301V 3.5mm screw terminals (LED outputs)
  - 4× power input terminals (12V)
  - 2× RJ45 (LVDS networking)
  - 1× USB-C

## Architecture

### Signal Path
```
RP2350B GPIO (3.3V)
  → SN74LVC8T245 Level Shifter (4× 8-channel)
  → 47Ω Series Resistor
  → Screw Terminal
  → LED String
```

### Power Distribution
```
12V Input
  → TPS63070 Buck-Boost → 5V (LED outputs, level shifters)
  → ME6217C33 LDO → 3.3V (MCU, flash, LVDS)
  → RP2350B Internal → 1.1V
```

### Network Topology
```
[Controller 1] —RJ45→ Cat5/Cat6 —RJ45→ [Controller 2] —RJ45→ ...
```
LVDS differential signaling enables daisy-chaining multiple boards with deterministic latency and excellent noise immunity.

## Component Summary

| Type | Count | Examples |
|------|-------|----------|
| Resistors | 65 | 47Ω series resistors, 100Ω LVDS termination |
| Capacitors | 45 | Decoupling and power filtering |
| ICs | 50+ | MCU, flash, level shifters, LVDS transceivers |
| Terminal Blocks | 36 | 32 LED outputs + 4 power inputs |
| Level Shifters | 4 | SN74LVC8T245PWR (8-channel each) |
| Switches/Buttons | 5 | Tactile buttons + DIP switch |
| LVDS Transceivers | 2 | Driver and receiver pair |

## Features

- 32 independent addressable LED channels
- 800kHz data rate (WS2812B protocol)
- USB programming and control interface
- Offline pattern storage via MicroSD
- Long-distance networking via LVDS over Cat5/Cat6
- Complete user interface for standalone operation

## File Structure

```
pixel_blit/
├── pixel_blit.kicad_pcb      # Main PCB layout
├── pixel_blit.kicad_sch      # Top-level schematic
├── Power.kicad_sch           # Power management circuitry
├── output.kicad_sch          # LED output stage
├── communications.kicad_sch  # LVDS networking
├── peripherals.kicad_sch     # User interface components
├── Library.pretty/           # Custom footprints
└── production/               # Manufacturing outputs
```

## Manufacturing Notes

- All components on front side (single-sided assembly)
- 4-layer PCB with mixed signal/power routing
- 360 vias total
- Standard SMT assembly process

## Use Cases

- Architectural lighting installations
- Stage and entertainment lighting
- Large-scale pixel displays
- Holiday and decorative lighting
- Interactive art installations
- Any application requiring synchronized control of multiple LED strings over long distances

## License

MIT License
