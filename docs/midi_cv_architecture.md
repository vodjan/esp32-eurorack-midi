<!--
esp32-eurorack-midi - USB MIDI to CV converter for Eurorack modular synthesizers
Copyright (C) 2026  Jan Vodstrčil

This work is licensed under the Creative Commons Attribution-ShareAlike 4.0
International License. To view a copy of this license, visit
https://creativecommons.org/licenses/by-sa/4.0/ or see docs/LICENSE.

SPDX-License-Identifier: CC-BY-SA-4.0
-->

# Eurorack USB MIDI-to-CV Module — Architecture Document

## Project Overview

A Eurorack module that converts MIDI messages to control voltages and gates,
intended to bridge Bitwig Studio (and any USB or TRS MIDI source) with an
analog modular synthesizer. The module accepts MIDI over USB (primary) and
TRS (secondary), and outputs four 1V/octave pitch CVs, four gates, and four
auxiliary CVs with runtime-configurable CC mapping.

**Estimated panel width:** 14–16HP  
**Target use case:** Bitwig Studio on computer → USB → module → Eurorack rack  
**Portfolio rationale:** Original open-source hardware/firmware project;
demonstrates analog design (op-amp output stage, precision voltage reference),
digital design (SPI DAC, level translation), and embedded systems (ESP-IDF,
FreeRTOS, USB MIDI stack)

---

## Hardware Architecture

### Microcontroller — ESP32-S3-WROOM-1-N8

- Dual-core Xtensa LX7 at 240 MHz; used for FreeRTOS task pinning
- Native USB OTG on GPIO19 (D−) / GPIO20 (D+): enumerates as a
  class-compliant USB MIDI device — no drivers required on any OS
- UART available for TRS MIDI receive
- SPI2 peripheral for DAC communication
- I2C for OLED display
- NVS (Non-Volatile Storage) partition for persistent configuration
- 8 MB flash — sufficient for firmware + NVS partition

**USB power isolation:**  
The module is powered exclusively from the Eurorack PSU. The USB-C connector's
VBUS pin must NOT connect to the module's 5V rail, as this would create two
competing 5V sources. VBUS should connect only to the ESP32-S3's VBUS
detection input (for USB enumeration awareness) through a 5.1kΩ / 10kΩ
voltage divider to keep it within 3.3V logic levels, or left to the ESP32-S3's
internal detection circuitry per Espressif's reference design.

---

### DAC Subsystem — 4× MCP4922

Each MCP4922 is a dual-channel, 12-bit, SPI DAC operating from a 5V supply
with an external voltage reference input per channel.

| IC   | Channel A    | Channel B    |
|------|--------------|--------------|
| DAC1 | Pitch CV 1   | Pitch CV 2   |
| DAC2 | Pitch CV 3   | Pitch CV 4   |
| DAC3 | Aux CV 1     | Aux CV 2     |
| DAC4 | Aux CV 3     | Aux CV 4     |

All four ICs share the SPI bus (MOSI, SCK) with individual chip-select lines
(CS1–CS4). MISO is not needed — the MCP4922 is write-only.

**12-bit resolution:** Provides 4096 steps over the full output range.
At 1V/octave over 8 octaves (96 semitones), each semitone spans
4096 / 96 ≈ 42.7 DAC steps. This gives a maximum quantization error of
~1.2 cents (half a step), which is below the ~5–10 cent threshold of
human pitch discrimination and consistent with commercial 12-bit
MIDI-to-CV converters. It does not resolve individual cents (which would
require ≥100 steps per semitone, i.e. a 14-bit DAC or higher), but is
musically acceptable for v1. Per-channel calibration trimmers compensate
for op-amp gain tolerance.

---

### Voltage Reference — 4× LM4040BIZ-5.0

- One reference per MCP4922 Vref pin (one IC drives both channels of one DAC)
- ±0.2% initial accuracy; 20 ppm/°C typical temperature coefficient
- TO-92 package; acts as a shunt regulator with a series resistor from +5V
- Do NOT use the supply rail directly as Vref — noise on the supply
  directly modulates pitch

---

### Output Stage — Pitch CV (4 channels)

The MCP4922 outputs 0–5V (with 5V Vref). Eurorack 1V/octave pitch CV
requires a larger range (typically 0–8V to cover 8 octaves).

**Per channel:**
- TL072 op-amp in non-inverting configuration
- Gain = 1.6× (0–5V → 0–8V)
- Resistor-set gain with a small trimmer (e.g., 100Ω) in series for
  per-channel calibration
- Powered from +12V / −12V rails, allowing clean swing near 0V

Two TL072 ICs (dual op-amp) cover all four pitch CV channels.

---

### Output Stage — Aux CV (4 channels)

Aux CV carries velocity, MIDI CC values, or other controller data. The
full 0–5V DAC output range is appropriate (0 = CC value 0, 5V = CC value 127).
No scaling needed — destination modules apply their own attenuation.

**Per channel:**
- TL072 in unity-gain buffer configuration
- No trimmer needed (not pitch-critical)

Two additional TL072 ICs cover all four aux CV channels.

**Total op-amps:** 4× TL072

---

### Gate Outputs — 74AHCT125

ESP32-S3 GPIO operates at 3.3V. Eurorack gates are conventionally 5V.

- 74AHCT125: quad 3-state buffer, accepts 3.3V logic input, outputs 5V
  logic from a 5V supply
- One IC handles all four gate outputs
- 3-state outputs: enable pins tied permanently low (outputs always active)

---

### MIDI Inputs — Overview

The module has two physical MIDI inputs and three logical MIDI streams.
All three are merged by the firmware router before voice allocation.

```
Physical inputs                Logical streams
───────────────                ───────────────
USB-C port        ────────────→ Virtual Port 1 (notes, keyboard data)
                  ────────────→ Virtual Port 2 (automation, CC from arranger)
TRS jack          ────────────→ Single MIDI stream
```

**Physical inputs:**
- 1× USB-C port (primary; two virtual MIDI ports inside one USB connection)
- 1× TRS jack (secondary; single standard MIDI stream)

There is no per-voice MIDI jack. MIDI is a multiplexed protocol — all
voices travel over a single stream and the firmware demultiplexes them
into the four independent CV/gate output pairs.

---

### MIDI Input — USB (Two Virtual Ports)

The USB MIDI 1.0 specification supports multiple virtual "cables" per
device. The module exposes two, which appear in Bitwig (and any OS) as
two separately named MIDI ports from one USB connection:

| Port   | Name (example)               | Intended use                                      |
|--------|------------------------------|---------------------------------------------------|
| Port 1 | ESP32 MIDI-CV — Notes        | Instrument tracks, keyboard input, note data      |
| Port 2 | ESP32 MIDI-CV — Automation   | CC automation lanes from Bitwig arranger          |

This separation is structural, not just conventional. CC messages on
Port 2 cannot collide with note data on Port 1 regardless of channel or
CC number assignments, making the Bitwig routing setup unambiguous.

**Typical usage scenarios:**

*Bitwig only (piano roll / arranger):*  
Instrument tracks → Port 1 for pitch/gate. Separate automation lanes
→ Port 2 for aux CV. Both run simultaneously with no conflicts.

*Hardware keyboard + Bitwig running:*  
Keyboard → TRS for live note playing. Port 2 from Bitwig → aux CV
automation in the background. Live performance and DAW automation
coexist cleanly.

*Hardware keyboard only (standalone, no computer):*  
USB disconnected. TRS carries all MIDI. Aux CVs follow the last saved
routing table (velocity by default).

---

### MIDI Input — TRS

**Circuit:**
- 3.5mm TRS jack (panel-mounted)
- DPDT panel switch for Type A / Type B wiring selection
  (physically swaps tip and ring before reaching the optocoupler)
- 1N4148 signal diode in series with the optocoupler LED anode
  (reverse polarity protection — wrong switch position = silence, no damage)
- H11L1 optocoupler (3.3V-native, Schmitt-trigger output) with 220Ω
  current-limiting resistor on the MIDI input side and a pull-up resistor
  on the output side to 3.3V
- Output of H11L1 connected directly to ESP32-S3 UART RX pin (no level
  translation needed; both run from 3.3V)
- Wrong switch position: 1N4148 blocks current, MIDI is silent, no damage

The H11L1 is specifically designed for 3.3V operation with MIDI-compatible
input current, and its built-in Schmitt trigger produces clean digital
edges without the pull-up tuning and base-bypass resistor required by
the legacy 6N138 in 3.3V applications.

---

### Power Supply

Eurorack 16-pin IDC connector provides +12V, −12V, and +5V (unreliable).
The module derives all rails from +12V and −12V.

| Rail   | Source            | Method                          | Consumers                         |
|--------|-------------------|---------------------------------|-----------------------------------|
| +5V    | +12V              | Linear regulator (e.g., L7805) | MCP4922s, LM4040s, 74AHCT125, TRS circuit |
| +3.3V  | +5V               | LDO (e.g., AMS1117-3.3)        | ESP32-S3-WROOM-1                  |
| −12V   | Eurorack directly | —                               | TL072 negative supply rail        |

**Reverse polarity protection:** P-channel MOSFET or Schottky diode on the
+12V input. A reversed header kills nothing; the protection device blocks
current before it reaches anything sensitive.

**Bypass capacitors:** 100nF ceramic + 10µF electrolytic at each IC power
pin; additional bulk capacitance (100µF+) at the power connector.

---

### User Interface

| Component        | Interface | Notes                                                  |
|------------------|-----------|--------------------------------------------------------|
| 1.3" 128×64 OLED | SPI       | SSD1309 controller (SH1106 as fallback if vendor varies) |
| Rotary encoder   | GPIO      | With pushbutton; panel-mounted                         |

The OLED shares the SPI bus with the four MCP4922 DACs, with its own
chip-select and DC (data/command) pins. The DAC task runs at higher
FreeRTOS priority than the display task; combined with short DAC
transactions (3 bytes each), this keeps pitch CV write latency in
microseconds even during a full display refresh.

The 1.3" panel is preferred over the smaller 0.96" variant for in-rack
readability. The SSD1309 controller is command-compatible with the
SSD1306, and ESP-IDF's panel driver handles both with the same code
path. If the received unit turns out to be SH1106 (controller varies
between vendors for this size), the firmware switches to U8g2's SH1106
init with a one-line change.

The menu system (driven by encoder + display) provides:
- MIDI channel assignment per voice
- Aux CV source selection per channel (velocity / named CC)
- Monophonic mode toggle and note priority (last / lowest / highest)
- Global CC-to-aux-CV mapping (for use without a keyboard)
- Calibration adjustment per pitch CV channel

---

### Panel Layout (preliminary, 14–16HP)

```
┌──────────────────┐
│  [OLED display]  │
│   [encoder]      │
│                  │
│  USB-C           │
│  TRS MIDI [A|B]  │  ← DPDT switch
│                  │
│  CV1  CV2  CV3  CV4   │  ← Pitch CV outs
│  G1   G2   G3   G4   │  ← Gate outs
│  A1   A2   A3   A4   │  ← Aux CV outs
└──────────────────┘
```

---

## Firmware Architecture

**Framework:** ESP-IDF with FreeRTOS  
**Language:** C  
**Build system:** pioarduino (PlatformIO fork) with `framework = espidf`

---

### FreeRTOS Task Structure

| Task              | Core | Priority | Description                                              |
|-------------------|------|----------|----------------------------------------------------------|
| `usb_midi_task`   | 0    | High     | TinyUSB MIDI stack; receives on both virtual cables (Port 1 and Port 2); tags each message with its source port before posting to `midi_queue` |
| `uart_midi_task`  | 1    | High     | UART receive ISR feed; posts messages to `midi_queue`    |
| `midi_router_task`| 1    | Medium   | Reads `midi_queue`; applies routing table (Port 1 → voice allocation; Port 2 → aux CV CC map); posts to `dac_queue` and sets gate GPIOs |
| `dac_task`        | 1    | Medium   | Reads `dac_queue`; writes to MCP4922s over SPI           |
| `ui_task`         | 1    | Low      | Polls encoder; updates OLED; handles menu; triggers NVS saves |

**Core assignment rationale:**  
The TinyUSB stack (USB MIDI) is pinned to Core 0, consistent with
Espressif's recommendation for protocol stacks. All application logic runs
on Core 1, keeping the two cores cleanly separated.

---

### Inter-task Communication

```
usb_midi_task (Port 1) ──┐
usb_midi_task (Port 2) ──┼──→ midi_queue (tagged MIDI messages) ──→ midi_router_task ──→ dac_queue ──→ dac_task
uart_midi_task (TRS)   ──┘                                                │
                                                                           └──→ GPIO (gate outputs, direct write)

ui_task ←──────────────── reads routing_table (mutex-protected shared struct)
midi_router_task ────────→ writes routing_table on config change → triggers NVS save
```

Each message on `midi_queue` carries a source tag (`PORT_1`, `PORT_2`,
or `TRS`). The router uses this to decide dispatch: Port 1 and TRS
messages feed the voice allocator; Port 2 messages feed the aux CV
CC map directly.

**Queue depths:**
- `midi_queue`: 32 messages (handles burst note events)
- `dac_queue`: 16 commands (DAC writes are fast; smaller buffer acceptable)

---

### Routing Table

The routing table is the central runtime-configurable data structure.
It is protected by a FreeRTOS mutex and persisted to NVS on change.

```c
// Source tag carried by every message on midi_queue
typedef enum {
    MIDI_SOURCE_USB_PORT1,  // USB virtual cable 0 — notes / keyboard
    MIDI_SOURCE_USB_PORT2,  // USB virtual cable 1 — automation / CC
    MIDI_SOURCE_TRS,        // TRS UART input
} midi_source_t;

typedef struct {
    midi_source_t source;
    uint8_t       status;
    uint8_t       data1;
    uint8_t       data2;
} midi_message_t;

typedef enum {
    AUX_SOURCE_VELOCITY,
    AUX_SOURCE_CC,
    AUX_SOURCE_AFTERTOUCH,
    AUX_SOURCE_PITCHBEND,
} aux_source_t;

typedef struct {
    uint8_t  midi_channel;       // 0–15 (MIDI channels 1–16)
    aux_source_t aux_source;     // What drives this voice's aux CV
    uint8_t  aux_cc_number;      // CC number if aux_source == AUX_SOURCE_CC
    // Per-note pitch bend offset — reserved for MPE (v2); zero in v1
    int16_t  pitch_bend_offset;  // DAC steps to add on top of semitone value
} voice_config_t;

typedef struct {
    voice_config_t voices[4];
    bool           mono_mode;
    uint8_t        mono_note_priority;  // 0=last, 1=lowest, 2=highest
    // Global CC-to-aux-CV map (Port 2 automation, or mono/idle mode)
    uint8_t        global_cc_map[4];    // CC number → aux CV channel 0–3
} routing_table_t;
```

The `pitch_bend_offset` field in `voice_config_t` is intentionally
included in v1 even though it is unused. Adding MPE in v2 requires
populating this field in the voice allocator — no struct refactor needed.

---

### Voice Allocation

**Polyphonic mode:**
- Round-robin assignment for incoming Note On messages
- On Note Off: the corresponding voice is released
- Stolen voices (all 4 active, new note arrives): oldest active voice
  is reassigned

**Monophonic mode:**
- Single voice used; note priority configurable (last / lowest / highest)
- Remaining aux CV channels remapped via `global_cc_map`

---

### DAC Output Mapping

For each pitch CV output, the MIDI note number is converted to a DAC value:

```
dac_value = round((note_number / 12.0) * (4096.0 / 8.0)) + pitch_bend_offset
```

Derivation: the op-amp gain of 1.6× maps the DAC's 0–5V output to 0–8V,
covering 8 octaves. One octave = 5V / 8 = 0.625V at the DAC output =
4096 / 8 = 512 DAC steps. One semitone = 512 / 12 ≈ 42.67 steps.
Maximum quantization error is half a step ≈ 1.2 cents.

`pitch_bend_offset` is always zero in v1. In v2 (MPE), it carries the
per-note pitch bend value scaled to DAC steps (±2 semitones = ±85 steps
for standard pitch bend range).

A per-channel calibration offset stored in NVS is added at the same
point to compensate for op-amp gain and Vref tolerance.

For aux CV outputs, the 7-bit MIDI value (0–127) is linearly mapped to
DAC full-scale: `dac_value = round((cc_value / 127.0) * 4095)`.

---

### NVS Persistence

ESP-IDF's NVS partition provides key-value storage in flash.

```c
// On boot
nvs_get_blob(handle, "routing_table", &routing_table, &size);

// On any config change (called from ui_task after menu interaction)
nvs_set_blob(handle, "routing_table", &routing_table, sizeof(routing_table));
nvs_commit(handle);
```

NVS writes are infrequent (user-initiated only) so wear is not a concern.

---

## Bill of Materials

The BOM is split into two phases. Phase 1 is for breadboard prototyping
on a devkit; Phase 2 is the final PCB build. Phase 1 uses through-hole
packages throughout for ease of hand-wiring; Phase 2 uses SMD where
practical for a more compact, professional-looking module.

### Phase 1 — Prototyping (breadboard)

| Component             | Part Number              | Qty | Notes                                  |
|-----------------------|--------------------------|-----|----------------------------------------|
| ESP32-S3 devkit       | ESP32-S3 N16R8 (Dratek)  | 1   | Dual USB-C; pin headers supplied loose |
| DAC                   | MCP4922-E/P              | 4   | DIP-14                                 |
| Voltage reference     | LM4040BIZ-5.0/NOPB       | 4   | TO-92                                  |
| Op-amp                | TL072BCP                 | 4   | DIP-8                                  |
| Level translator      | SN74AHCT125N             | 1   | DIP-14                                 |
| Optocoupler           | H11L1-ISO (Isocom)       | 1   | DIP-6                                  |
| Protection diode      | 1N4148 (Diotec, 1N4148-DIO) | 5 | DO-35; order extras            |
| OLED display          | 1.3" 128×64 SPI (white)  | 1   | Dratek; SSD1309 (or SH1106) controller |
| Rotary encoder        | Bourns PEC11R-4220F-S0024 | 2 | With pushbutton; 24 pulses/24 detents; 11mm body; 20mm D-shaft; order 2 |
| Toggle switch         | JS202011CQN (C&K)        | 1   | DPDT slide, THT, 2.54mm pitch; TRS Type A/B selection |
| 3.5mm jack (stereo)   | JC-128 (Vanguard)        | 2   | TRS, THT, 2.5mm pitch; pigtail with flying leads to avoid breadboard stress |
| Breadboard            | 830-point                | 2   | One isn't enough for this              |
| Jumper wires          | Dupont M-M and M-F       | —   | Mixed lengths                          |
| Resistor assortment   | E12 series, 1/4W         | 1   | 220Ω, 1kΩ, 10kΩ used heavily          |
| Trimmer               | Bourns 3296W-1-101 (100Ω) | 6 | 25-turn, top adjust; pitch CV calibration; order spares |
| Capacitor assortment  | 100nF ceramic, 10µF elec | —   | Power supply decoupling                |

**Power for prototyping:** A bench supply providing +12V/−12V or a
Eurorack PSU brick. The DPS150 alone provides only +5V single-rail, so
parts of the analog output stage cannot be tested without an additional
±12V source. See "Power Supply" section.

### Phase 2 — Final PCB

| Component             | Part Number              | Qty | Notes                                  |
|-----------------------|--------------------------|-----|----------------------------------------|
| Microcontroller       | ESP32-S3-WROOM-1-N8      | 1   | 8 MB flash; soldered directly to PCB   |
| DAC                   | MCP4922-E/SL             | 4   | SO-14                                  |
| Voltage reference     | LM4040BIZ-5.0/NOPB       | 4   | TO-92                                  |
| Op-amp                | TL072CDR                 | 4   | SOIC-8                                 |
| Level translator      | SN74AHCT125D             | 1   | SO-14                                  |
| Optocoupler           | H11L1SM-ISO (Isocom)     | 1   | Gull-wing SMD-6                        |
| Protection diode      | 1N4148                   | 1   | TRS MIDI polarity protection           |
| +5V regulator         | L7805 or SMD equivalent  | 1   | From +12V Eurorack rail                |
| +3.3V LDO             | AMS1117-3.3              | 1   | SOT-223; from +5V                      |
| Reverse polarity prot.| P-MOSFET (e.g., SI2301)  | 1   | On +12V input                          |
| USB-C connector       | PCB-mount, USB 2.0       | 1   | Connects to ESP32-S3 native USB pins   |
| Eurorack power header | 2×8 IDC, 2.54mm pitch    | 1   | Standard Eurorack PSU connection       |
| 3.5mm jacks           | Mono PCB-mount (Thonkiconn) | 13 | 4 pitch + 4 gate + 4 aux + 1 TRS MIDI |
| OLED display          | 1.3" 128×64 SPI (white)  | 1   | Module mounted via headers; SSD1309/SH1106 |
| Rotary encoder        | Bourns PEC11R-4220F-S0024 | 1 | PCB-mount; 24 pulses/24 detents; 11mm body; 20mm D-shaft |
| Toggle switch         | DPDT panel mount         | 1   | TRS Type A/B selection                 |
| Pitch CV trimmers     | 100Ω multiturn           | 4   | SMD or through-hole                    |

**Passives (resistors, capacitors, ferrite beads):** Finalize during
schematic capture. Rough estimate: 30–50 resistors, 20–30 capacitors.
Use 1% metal film resistors for the pitch CV gain network — tolerance
directly affects pitch accuracy.

**Eurorack panel:** Aluminum, FR4, or PCB-as-panel approach (cheaper,
doable from the same fab). Design comes after the main PCB is finalized.

### Sourcing notes

- **TME** (Poland, ships to Czechia) covers all the ICs, passives, and
  most connectors. Stock for the LM4040BIZ-5.0 was patchy when checked
  — verify before ordering.
- **Dratek** (Czech) covers the devkit and prototyping consumables
  (breadboards, jumpers, headers).
- **Thonk** (UK) or **Modular Addict** (US) for Eurorack-specific parts:
  Thonkiconn jacks, IDC headers, blank panels. Both ship to Czechia.
- **AliExpress** for the encoder if Dratek doesn't stock the preferred
  variant — fine quality at a fraction of the price, with longer shipping
  times. The OLED display is sourced from Dratek instead (local stock,
  faster delivery, can be added to the same order as the devkit).

---

## Development Plan

### Phase 1 — Firmware prototype (ESP32-S3 DevKitC-1 + breadboard)
1. Verify USB MIDI enumeration and message receive in ESP-IDF
2. Verify SPI communication with MCP4922; confirm DAC output with multimeter
3. Implement basic MIDI note → pitch CV + gate output (single voice, no RTOS)
4. Add FreeRTOS task structure; verify no timing issues under load
5. Add TRS MIDI receive via UART
6. Add 4-voice polyphonic allocation
7. Add routing table, aux CV output, CC mapping
8. Add OLED + encoder menu system
9. Add NVS persistence
10. Calibration routine

### Phase 2 — PCB design (KiCAD)
1. Schematic capture (all blocks above)
2. Power supply design and decoupling
3. PCB layout (Eurorack form factor, signal integrity for analog outputs)
4. Design rule check and review
5. Gerber generation and PCB order

### Phase 3 — Bring-up and validation
1. Power supply verification before populating signal ICs
2. DAC output calibration (1V/oct accuracy across temperature)
3. MIDI latency measurement
4. Full functional test with Bitwig and hardware keyboard
5. Panel design and fabrication

---

## Open Questions and Future Work

- **Panel width:** Finalize HP count during schematic/layout phase once
  connector and display placement is known

- **Calibration UX:** Decide whether calibration is menu-driven or
  requires external equipment (multimeter minimum; CV meter ideal)

- **v2 — MPE support:** MIDI Polyphonic Expression maps naturally onto
  the 4-voice architecture. Each voice would occupy its own MIDI channel
  (channels 2–5 per the MPE spec; channel 1 as the global channel), and
  per-note pitch bend would populate `pitch_bend_offset` in `voice_config_t`.
  The firmware data structures are already designed to support this without
  refactoring. Requires an MPE-capable input device or Bitwig's per-note
  expression editor.

- **v2 — USB host mode:** Allows a USB MIDI keyboard to connect directly
  to the module's USB-C port without a computer. The ESP32-S3 OTG
  controller supports host mode, but requires VBUS switching circuitry
  (to provide 5V to the connected device) and a mode-select mechanism.
  TRS input already covers standalone keyboard use for v1.

- **Considered and rejected — OSC:** Open Sound Control over WiFi was
  considered as a high-resolution alternative to MIDI CC for automation.
  Rejected because WiFi introduces variable latency incompatible with
  real-time CV control. The two-port USB MIDI design achieves the same
  clean automation separation with a wired, deterministic connection.

- **Open-source release:** MIT license, KiCAD source files + firmware
  on GitHub; full documentation as portfolio artifact
