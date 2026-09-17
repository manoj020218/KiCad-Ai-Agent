# Example Projects

Three complete circuits built end-to-end through this MCP server —
schematic, routed PCB, and manufacturing files — used to validate the
schematic and PCB pipelines documented in
[docs/HEADLESS_AUTHORING.md](../docs/HEADLESS_AUTHORING.md) and
[docs/PCB_DESIGN_WORKFLOW.md](../docs/PCB_DESIGN_WORKFLOW.md). Each one
passes ERC and DRC with 0 errors.

| Project | Circuit | Highlights |
| --- | --- | --- |
| [`555-astable/`](555-astable/) | 555 timer astable oscillator driving an LED | Power-flag rules, multi-pin-same-net labels |
| [`opamp-divider/`](opamp-divider/) | Non-inverting op-amp amplifier (gain 2) biased off a resistor divider | Analog feedback network, NC pin handling |
| [`esp32c3-breakout/`](esp32c3-breakout/) | ESP32-C3-WROOM-02 module + reset/boot buttons + UART header | SMD module, thermal-via design rules, denser routing (73 tracks, 3 vias) |

Each folder contains:

- `*.kicad_sch` / `*.kicad_pcb` / `*.kicad_pro` — the KiCad project
- `*.svg`, `*-view.png` — rendered previews of the schematic
- `*-board.pdf`, `*-board-view.png` (555 only) — rendered PCB views
- `*_drc_violations.json` — the final DRC report (warnings only, 0 errors)
- `gerbers/` — Gerber, drill, and job files ready to send to a fab house

## How these were built

Each project followed the same recipe, laid out in full in the two docs
linked above:

1. `create_project` → place & wire components (`batch_add_and_connect`) →
   `batch_add_no_connects` for unused pins
2. `lint_offgrid` → `run_erc` (0 errors) → cosmetic cleanup
   (`lint_schematic_cosmetic`, `autoplace_schematic_fields`) →
   `run_erc` again to confirm the cleanup didn't change connectivity
3. `sync_schematic_to_board` → `suggest_placement` (verified against real
   pad positions, not just the tool's own pre-checks — see
   [Known Issues #11](../docs/KNOWN_ISSUES.md)) → `autoroute`
   (Freerouting) → `run_drc` (0 errors) → `export_gerbers` /
   `export_drill` / `export_pcb_pdf`

Real problems found and fixed along the way (a resistor placed off the
board edge, an SMD module's thermal vias needing a relaxed design rule,
a dead Freerouting download URL) are documented in
[docs/KNOWN_ISSUES.md](../docs/KNOWN_ISSUES.md) rather than hidden —
they're representative of what to expect, not edge cases to ignore.
