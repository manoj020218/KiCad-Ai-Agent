# 555 Astable Oscillator

A classic NE555-based astable (free-running) oscillator driving an LED,
built entirely through MCP tool calls.

## Circuit

| Ref | Value | Role |
| --- | --- | --- |
| U1 | NE555P | Timer IC, astable configuration |
| R1 | 10k | VCC → DISCH (charge path) |
| R2 | 10k | DISCH → THR/TRIG (charge/discharge path) |
| C1 | 100nF | Timing capacitor (THR/TRIG → GND) |
| C2 | 10nF | CONT pin decoupling |
| R3 | 330Ω | LED current-limit resistor |
| D1 | LED | Output indicator |

Output frequency ≈ 1.44 / ((R1 + 2·R2) · C1) ≈ **480 Hz**, roughly 50%
duty cycle skewed slightly high (standard 555-astable asymmetry, since
charge goes through R1+R2 but discharge only through R2).

RESET tied to VCC (always enabled). Two `power:PWR_FLAG` components
satisfy ERC's "every power net needs a driver" rule for VCC and GND,
since nothing else on this simple board drives them.

## Board

50×40mm, 2-layer, through-hole parts throughout. Routed with the
Freerouting autorouter — 30 tracks, 0 vias.

## Verification

- ERC: 0 errors (1 benign `lib_symbol_mismatch` warning — see
  [Known Issues](../../docs/KNOWN_ISSUES.md), §4 of
  [Headless Authoring](../../docs/HEADLESS_AUTHORING.md))
- DRC: 0 errors, 6 cosmetic silkscreen warnings
- Netlist hand-verified against the intended topology above
