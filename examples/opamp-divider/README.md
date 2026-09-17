# Non-Inverting Op-Amp Amplifier

A TL071 op-amp wired as a non-inverting amplifier, biased at mid-rail by
a resistor divider — a standard single-supply analog building block.

## Circuit

| Ref | Value | Role |
| --- | --- | --- |
| U1 | TL071 | Single JFET-input op-amp |
| R1 | 10k | VCC → VIN_BIAS (divider top) |
| R2 | 10k | VIN_BIAS → GND (divider bottom) — sets the non-inverting input to VCC/2 |
| R3 | 10k | Feedback network: inverting input → GND (gain-setting) |
| R4 | 10k | Feedback network: VOUT → inverting input |
| C1 | 100nF | VCC decoupling |

Gain = 1 + R4/R3 = **2**. This is a single-supply design (V- tied to
GND), so V+/V- pin naming follows the schematic symbol, not necessarily a
split supply. U1's unused offset-null pins (1, 5) and the NC pin (8) are
explicitly flagged with `add_no_connect` so ERC doesn't complain about
them.

## Board

55×45mm, 2-layer, through-hole. Routed with Freerouting — 25 tracks, 1
via.

## Verification

- ERC: 0 errors, 0 warnings
- DRC: 0 errors, 1 warning (a minor dangling-stub routing artifact near
  a via — connectivity confirmed intact, see
  [Known Issues](../../docs/KNOWN_ISSUES.md))
- Netlist hand-verified against the intended topology above — note this
  also exercised `batch_add_and_connect`'s "power net without PWR_FLAG"
  heuristic giving a false positive on VIN_BIAS (see Known Issues)
