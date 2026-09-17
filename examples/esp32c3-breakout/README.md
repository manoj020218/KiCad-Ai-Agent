# ESP32-C3-WROOM-02 Minimal Breakout

A minimal, functional breakout board for the ESP32-C3-WROOM-02 Wi-Fi/BLE
module — reset, boot-mode entry, and a UART header for an external
USB-serial adapter.

## Circuit

| Ref | Value | Role |
| --- | --- | --- |
| U1 | ESP32-C3-WROOM-02 | Wi-Fi/BLE SoC module (SMD) |
| R1 | 10k | EN pull-up (module runs when EN floats high) |
| SW1 | — | Reset button: EN → GND |
| SW2 | — | Boot button: IO9/BOOT → GND (hold at power-on to enter download mode) |
| C1 | 100nF | VCC decoupling |
| J1 | 1×4 header | VCC, GND, TX, RX — for an external USB-serial adapter |

All 12 unused GPIO pins are explicitly flagged no-connect. This is the
densest of the three example boards — 20 pads on the module alone,
including a 12-pad thermal/EMI ground array under the shield.

## Board

60×50mm, 2-layer, mixed SMD (module) + THT (passives, buttons, header).
Routed with Freerouting — 73 tracks, 3 vias.

## Verification

- ERC: 0 errors, 0 warnings
- DRC: 0 errors, 5 cosmetic warnings (2 dangling-stub, 1 silk overlap, 2
  silk-over-copper)
- **Note:** this board's first DRC run reported 12 `drill_out_of_range`
  **errors** — the module's own thermal-via array uses 0.2mm drills,
  finer than the board's default 0.3mm minimum. Fixed with
  `set_design_rules({minHoleDiameter: 0.2})`, not by fighting the
  footprint. See [Known Issues #12](../../docs/KNOWN_ISSUES.md) — if
  you reuse this module, check your fab supports 0.2mm holes.
