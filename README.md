# 200W Programmable DC Electronic Load

![Electronics Load](https://github.com/e135193/Electronics-Load/blob/main/ElectronicsLoad.png)

MCU-controlled programmable DC electronic load designed for bench testing power supplies, batteries, and chargers up to **200 W**.


## Overview

The load sinks current from an external DC source (via a 4 mm banana jack input) and dissipates it across four independent MOSFET current-sink channels, each closed-loop controlled from a PWM reference. Total bus voltage/current is monitored centrally, and per-channel targets are set and displayed through an onboard TFT UI.

## Key Specifications

| Parameter | Value |
|---|---|
| Input voltage (Vin max) | 30 V |
| Input current (Iin max) | 10 A |
| Load input connector | 4 mm banana jack |
| Aux power input | USB Type-C |
| Channels | 4x independent current-sink channels |
| Target max current per channel | 5 A |

## Control & UI

- **MCU:** RP2354A
- **Power monitor:** INA226 (I2C), shared across all channels via a common shunt on +Vin, with a hardware ALERT line to the MCU
- **Display:** ST7789 TFT LCD (SPI)
- **Input:** Rotary encoder
- **Status indicator:** 1x RGBW LED

## Load Channel Architecture

Each of the 4 channels is fully independent and consists of:

- 1x **IRFP260NPBF** MOSFET as the current-sink element
- 1x **OPA2197IDR** dual op-amp per channel:
  - Half A: error amplifier, comparing the PWM-derived reference to shunt feedback
  - Half B: differential shunt-sense amplifier
- Dedicated per-channel shunt resistor (10 mΩ, 5 W)
- Gate resistor (10 Ω) and a 5 V gate-protection zener + pulldown, to prevent the MOSFET from fully turning on (near-short) if the control loop becomes unstable

**Reference & feedback signal chain (per channel):**

```
PWM (0-1.67V) --> gain-of-3 non-inverting stage (10k/5k) --> 0-5V reference
                                                                  |
                                                                  v
                                                   error amp (OPA2197, half A)
                                                                  ^
                                                                  |
Shunt (10mΩ, 5W) --> diff-amp gain x100 (99k/1k, OPA2197 half B)
```

- Loop compensation: 10 kΩ + 100 nF low-pass filter in the feedback path
- Op-amp supply: 5 V (reduced from initial design to limit MOSFET gate drive under fault conditions)

## Cooling

- 1x 12 VDC fan with RPM (tachometer) output
- 2x NTC thermistors: 1x internal (ambient/enclosure), 1x mounted on the heat sink

### Heat Sink (mechanical)

- **Copper base plate:** 81 mm (L) x 58.5 mm (W) x 5 mm (H), with two mounting ears (9 x 25 mm, Ø4 mm hole each) and a 54 x 58.5 x 2.2 mm contact pad on the underside
- **Aluminum fin stack:** 40x fins, 0.5 mm thick each, 67.5 mm wide, spread over a 67.5 mm span along the base length, 43.5 mm tall
- **Fan:** 12 VDC, attached directly to the fin stack

CAD sources (parametric, dimensions adjustable):
- `heatsink_cadquery.py` — CadQuery script, exports colored `heatsink.step` / `heatsink.stl`
- `heatsink_freecad_macro.py` — FreeCAD macro (native `Part` API), exports STEP

## Known Issues / Open Items

- Per-channel shunt value has a documentation discrepancy between the schematic (5 mΩ) and the BOM table (1 mΩ, 2 W, 2512 package) — needs resolving before layout.

## Repository Structure (suggested)

```
.
├── hardware/           # Schematics, BOM, PCB layout
├── mechanical/
│   ├── heatsink_cadquery.py
│   ├── heatsink_freecad_macro.py
│   ├── heatsink.step
│   └── heatsink.stl
├── firmware/            # RP2354A firmware
└── README.md
```

## Status

Under active development — driver architecture and channel feedback design finalized; shunt value discrepancy and PCB layout still in progress.
