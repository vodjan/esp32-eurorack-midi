# esp32-eurorack-midi

A USB MIDI to CV converter for Eurorack modular synthesizers, built around the
ESP32-S3. Accepts MIDI over both USB (as a class-compliant device with two
virtual ports) and TRS, and outputs four pitch CVs, four gates, and four
runtime-configurable auxiliary CVs.

Designed to bridge Bitwig Studio (or any USB or TRS MIDI source) with an
analog modular rack, with a specific focus on clean, wired, low-latency
integration between DAW automation and CV control.

## Status

**In development.** Firmware and hardware are being developed in parallel.
See the roadmap below for the current state of each subsystem.

## Features

- **4 voices** of 1V/octave pitch CV over 8 octaves, ±1.2 cent max quantization
  error via 12-bit DAC + trimmer calibration
- **4 gate outputs** (5V logic), one per voice
- **4 auxiliary CVs** with runtime-configurable routing from any MIDI CC,
  velocity, aftertouch, or pitch bend
- **USB MIDI** with two virtual ports (notes and automation) — appears as a
  standard two-port MIDI interface in any DAW, no drivers needed
- **TRS MIDI input** with DPDT Type A/B switch for compatibility with any
  hardware controller
- **On-device UI** via 128×64 OLED and rotary encoder for configuration
- **Configuration persisted** to ESP32-S3's NVS flash across power cycles

## Hardware Overview

- **Microcontroller:** ESP32-S3-WROOM-1 (dual-core Xtensa LX7 @ 240 MHz,
  native USB OTG)
- **DACs:** 4× MCP4922 (12-bit dual-channel SPI DACs) for 8 total CV channels
- **Voltage reference:** LM4040BIZ-5.0 (±0.2%) per DAC for pitch stability
- **Output stage:** TL072 op-amps with per-channel calibration trimmers,
  ±12V-powered for clean 0V–8V pitch swing
- **Level translation:** 74AHCT125 for 3.3V→5V gate outputs
- **TRS MIDI:** H11L1 optocoupler (3.3V-native with built-in Schmitt trigger)

Full component details in [docs/architecture.md](docs/architecture.md).

## Firmware Overview

- **Framework:** ESP-IDF with FreeRTOS
- **USB MIDI stack:** TinyUSB (built into ESP-IDF)
- **Task structure:** dedicated tasks for USB MIDI, UART MIDI, routing, DAC
  output, and UI, with priorities tuned to keep pitch CV update latency in
  the microsecond range
- **Configuration:** persistent routing table in NVS, editable via the
  on-device UI

Design details, task priorities, and inter-task communication are documented
in [docs/architecture.md](docs/architecture.md).

## Repository Structure

```
esp32-eurorack-midi/
├── LICENSE           Multi-license pointer (see below)
├── README.md
├── firmware/         ESP-IDF project, C source, build config
│   └── LICENSE       GPL v3
├── hardware/         KiCAD project, schematics, PCB, panel design
│   └── LICENSE       CERN-OHL-S v2
└── docs/             Architecture, build guides, calibration procedure
    └── LICENSE       CC BY-SA 4.0
```

## Licensing

This project uses separate licenses for hardware, firmware, and documentation
to reflect the different nature of each:

- **Firmware** (under `firmware/`) is licensed under the
  [GNU General Public License v3.0 or later](firmware/LICENSE).
- **Hardware** (under `hardware/`) is
  licensed under the
  [CERN Open Hardware Licence v2 – Strongly Reciprocal](hardware/LICENSE).
- **Documentation** (under `docs/`) is licensed under
  [Creative Commons Attribution-ShareAlike 4.0 International](docs/LICENSE).

Copyright © 2026  Jan Vodstrčil.

## Roadmap

### Phase 1 — Firmware prototype
- [ ] ESP-IDF project scaffold and FreeRTOS task skeleton
- [ ] USB MIDI device enumeration (two virtual ports)
- [ ] SPI communication with MCP4922 DACs verified
- [ ] Single-voice MIDI note → pitch CV + gate output
- [ ] TRS MIDI receive via UART
- [ ] 4-voice polyphonic voice allocation
- [ ] Routing table, aux CV output, CC mapping
- [ ] OLED + encoder menu system
- [ ] NVS persistence
- [ ] Calibration routine

### Phase 2 — PCB design
- [ ] Schematic capture
- [ ] PCB layout for Eurorack form factor
- [ ] Design rule check
- [ ] Gerber generation and PCB order
- [ ] Bring-up and validation

### v2 (future)
- [ ] MPE support (per-note pitch bend to pitch CV)
- [ ] USB host mode (direct USB keyboard input without a computer)

## Acknowledgements

Built as a portfolio project during the final year of a Medical Electronics
and Bioinformatics degree at the Faculty of Electrical Engineering, Czech
Technical University in Prague.
