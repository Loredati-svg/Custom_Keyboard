# Custom Keyboard – RP2040 Macropad

A custom 9-key macropad with two rotary encoders and per-key RGB lighting, designed from scratch in **KiCad 10** around a bare **RP2040** microcontroller.

<p align="center">
  <img src="https://img.shields.io/badge/Status-Completed%20·%20Untested-yellowgreen" alt="Status">  <img src="https://img.shields.io/badge/KiCad-10.0-blue?logo=kicad&logoColor=white" alt="KiCad 10.0">
  <img src="https://img.shields.io/badge/Hardware%20Rev-v1.0-brightgreen" alt="Hardware Rev">
  <img src="https://img.shields.io/badge/License-CERN--OHL--P-orange" alt="License">
</p>


<p align="center">
  <img width="350" alt="Top view" src="https://github.com/user-attachments/assets/ecb11acc-f282-4864-b464-dba538917e3a" />
  <img width="350" alt="Bottom view" src="https://github.com/user-attachments/assets/d3d79ca4-ac9d-4038-b3dc-ebd776abf5a5" />

<img width="350" alt="Bottom layout" src="https://github.com/user-attachments/assets/6ef2a868-96b0-48b3-ba7e-4b8e4044740a" />

<img width="350" alt="Top layout" src="https://github.com/user-attachments/assets/27cf5249-b942-4c55-91e0-c8a8d699fcf1" />

<img width="350" alt="Power plane" src="https://github.com/user-attachments/assets/abc91846-feee-4452-97ef-5678b328156d" />

<img width="350" alt="GND plane" src="https://github.com/user-attachments/assets/1dbdf4f1-f3c4-40fe-93fc-21222cafa8e2" />
</p>


---
## Features

- **9 mechanical keys** in a 3×3 grid (Cherry MX compatible, PCB-mount, standard 19.05 mm pitch)
- **2 rotary encoders** with push button (Bourns PEC12R, 12 detents) – e.g. volume, scrolling, media control
- **Per-key RGB lighting** with 9 addressable SK6812MINI-E LEDs (reverse-mount, shining through the switch)
- **USB-C** connector (USB 2.0 Full Speed)
- **Bare RP2040** design (no module): QSPI flash, crystal and power supply on board
- **4-layer PCB**, all SMD components on the bottom side, only switches and encoders on top
- Designed for **QMK / VIA** firmware

---

## Block Diagram

```
USB-C ──► Polyfuse ──► VBUS (5 V) ──┬──► AP2112K-3.3 ──► +3V3 ──► RP2040, Flash, pull-ups
  │                                 │
  │                                 └──► SN74LV1T34 (3.3 V → 5 V) ──► SK6812MINI-E ×9 (daisy chain)
  │
  └── D+/D− ──► USBLC6-2SC6 (ESD) ──► 27 Ω ──► RP2040 USB
                                               │
                         W25Q32 QSPI Flash ◄───┤
                         12 MHz Crystal    ◄───┤
                         9 Keys            ◄───┤ GPIO0–8
                         2 Encoders        ◄───┤ GPIO9–14
                         LED data          ◄───┘ GPIO15
```

---

## Hardware

| Block | Part | Notes |
|---|---|---|
| MCU | **RP2040** (QFN-56) | Dual Cortex-M0+, internal 1.1 V core regulator |
| Flash | **W25Q32JV** (4 MB, QSPI) | Stores the firmware |
| Clock | **ABM8-272-T3** 12 MHz | 15 pF load caps, 1 kΩ series resistor on XOUT (per RP2040 hardware design guide) |
| LDO | **AP2112K-3.3** | 600 mA, VBUS → 3.3 V |
| USB | USB-C 16-pin receptacle | 5.1 kΩ pull-downs on CC1/CC2 (device/UFP) |
| ESD | **USBLC6-2SC6** | Flow-through layout on D+/D− |
| Protection | Resettable polyfuse | On VBUS input |
| Keys | Cherry MX compatible switches | Direct GPIO connection, no matrix / no diodes |
| Encoders | **Bourns PEC12R-4120F-S0012** | 12 pulses / 12 detents, push switch |
| LEDs | **SK6812MINI-E** ×9 | Reverse-mount, powered from VBUS |
| Level shifter | **SN74LV1T34** | Translates the 3.3 V LED data line to 5 V |

---

## Design Notes

**Direct-wired keys.** With only 9 keys and 2 encoders the RP2040 has plenty of GPIOs, so every key is wired directly to its own GPIO with the internal pull-up enabled. No key matrix and no diodes are needed, and there is no ghosting by design.

**Encoder filtering.** Each encoder channel (A/B) uses a 10 kΩ external pull-up plus an RC low-pass filter (10 kΩ / 10 nF, τ ≈ 0.1 ms) to suppress contact bounce in hardware, while still allowing fast rotation.

**LED data level shifting.** The SK6812 LEDs are supplied from 5 V, so their minimum input-high threshold is ≈ 0.7 × VDD = 3.5 V, above the RP2040's 3.3 V output. A single-gate SN74LV1T34 buffer powered from VBUS shifts the data line to 5 V. A 33 Ω series resistor dampens ringing, and a 10 kΩ pull-down keeps the data line low while the MCU is booting, so the LEDs do not flash randomly at power-up.

**LED power.** Each LED has its own 100 nF decoupling capacitor, plus a 10 µF bulk capacitor on VBUS. LED brightness should be capped in firmware to stay within the 500 mA USB 2.0 budget.

**USB.** The RP2040 runs USB 2.0 Full Speed (12 Mbps). D+/D− are routed as a differential pair over an unbroken ground plane, with the ESD protection in flow-through configuration and the 27 Ω series resistors placed close to the MCU.

**Boot mode.** A BOOT push-button pulls QSPI_SS low through a 1 kΩ resistor. Holding it while plugging in USB starts the RP2040 USB mass-storage bootloader (`RPI-RP2`). No reset button is needed: unplugging and replugging the cable does the same job. SWD (SWCLK/SWDIO) is exposed on test points for debugging.

---

## PCB

- **4 layers**, 1.6 mm
  - `F.Cu` – key / encoder signals
  - `POWER` – power plane (+3V3 / VBUS), some low-speed routing
  - `GND` – solid ground plane, reference for USB and bottom-side signals
  - `B.Cu` – all SMD components, USB pair
- **Single-sided SMD assembly** (bottom) to reduce assembly cost. Switches and encoders are through-hole and hand-soldered on top.
- Board cut-outs under each switch for the reverse-mount LEDs
- 4× **M3 mounting holes** for standoffs / case

---

## Pinout

| Function | GPIO |
|---|---|
| Keys SW2–SW10 | GP0 – GP8 |
| Encoder VR2 – A / B / Push | GP9 / GP10 / GP11 |
| Encoder VR1 – A / B / Push | GP12 / GP13 / GP14 |
| RGB LED data (9 LEDs) | GP15 |
| SWD | SWCLK / SWDIO (test points) |

---

## Flashing

1. Hold the **BOOT** button.
2. Plug in the USB-C cable.
3. Release the button. A drive called `RPI-RP2` appears.
4. Drag and drop the `.uf2` firmware file onto it. The board reboots automatically.

After the first flash, the bootloader can also be entered from a key mapped to `QK_BOOT` in QMK.

---

## Firmware

Planned: **QMK** with **VIA/Vial** support
- Encoders mapped with `ENCODER_MAP` (e.g. volume up/down + mute, scroll, media keys)
- `RGB_MATRIX` with the WS2812 PIO driver on GP15

---

## Repository Structure

```
Custom-Keyboard/
├── Kicad_Project/   # KiCad schematic, PCB and project files
├── Lib/             # Custom symbols, footprints and 3D models
└── 3D/              # 3D models / renders
```

---

## Project Status

- [x] Schematic
- [x] PCB layout (4-layer)
- [x] DRC / ERC clean
- [ ] Manufacturing
- [ ] Bring-up and testing
- [ ] QMK firmware

---

## Author

**Lorenzo Dati** – Analog IC Designer

<!-- Choose a license, e.g. CERN-OHL-S-2.0 for hardware -->
