# Known Issues & Workarounds

**Last Updated:** 2026-03-21
**Version:** 2.2.3

This document tracks known issues and provides workarounds where available.

---

## Current Issues

### 1. `get_board_info` KiCAD 9.0 API Issue

**Status:** KNOWN - Non-critical

**Symptoms:**

```
AttributeError: 'BOARD' object has no attribute 'LT_USER'
```

**Root Cause:** KiCAD 9.0 changed layer enumeration constants

**Workaround:** Use `get_project_info` instead for basic project details

**Impact:** Low - informational command only

---

### 2. Zone Filling via SWIG Causes Segfault

**Status:** KNOWN - Workaround available

**Symptoms:**

- Copper pours created but not filled automatically when using SWIG backend
- Calling `ZONE_FILLER` via SWIG causes segfault

**Workaround Options:**

1. Use IPC backend (zones fill correctly via IPC)
2. Open the board in KiCAD UI -- zones fill automatically when opened
3. Use `refill_zones` tool (may still segfault in some configurations)

**Impact:** Medium - affects copper pour visualization until opened in KiCAD

---

### 3. UI Manual Reload Required (SWIG Backend)

**Status:** BY DESIGN

**Symptoms:**

- MCP makes changes via SWIG backend
- KiCAD does not show changes until file is reloaded

**Why:** SWIG-based backend modifies files directly and cannot push changes to a running UI

**Fix:** Use IPC backend for real-time updates (requires KiCAD running with IPC enabled)

**Workaround:** Click the reload prompt in KiCAD or use File > Revert

---

### 4. IPC Backend Limitations

**Status:** EXPERIMENTAL

**Known Limitations:**

- KiCAD must be running with IPC enabled (Preferences > Plugins > Enable IPC API Server)
- Some commands fall back to SWIG (e.g., delete_trace)
- Footprint loading uses hybrid approach (SWIG for library, IPC for placement)

**Workaround:** The server automatically falls back to SWIG backend when IPC is unavailable

---

### 5. package.json Version Mismatch

**Status:** KNOWN - Non-critical

**Symptoms:** package.json shows version 2.1.0-alpha while CHANGELOG documents version 2.2.3

**Impact:** Cosmetic only. CHANGELOG.md is the authoritative version reference.

---

### 6. `.kicad_pro` net_settings Must Be Edited via JSON Merge

**Status:** BY DESIGN (guard added)

Backend board saves are wrapped in `preserve_project_settings()`
(`python/utils/project_settings_guard.py`): pcbnew serializes a possibly
stale in-memory project model over `.kicad_pro` on every
`SaveBoard`/`BOARD.Save`, so the guard restores the on-disk
`net_settings` (and any dropped top-level keys) after each save.

**Implication:** commands must persist net class / netclass_patterns
changes via direct JSON read-modify-write of the `.kicad_pro` (the
`persist_netclass_to_project` pattern in `python/commands/routing.py`),
never through the pcbnew project model — model-side changes to
`net_settings` are intentionally reverted by the guard.

### 7. Flat Vendor Symbols Break kicad-skip-Based Tools (now diagnosed)

**Status:** MITIGATED — loud structured errors

SnapEDA/SamacSys `.kicad_sym` captures put pins/graphics directly under the
top-level `(symbol "NAME" ...)` with no `_1_1` sub-unit. KiCad and kicad-cli
tolerate this, but kicad-skip's parser crashes on it, taking down every
skip-based tool for the whole sheet (including sheets that merely embed a
snapshot of such a symbol in their own `lib_symbols`).

Schematic load failures now raise `SchematicLoadError` and all schematic
tools return a structured error naming the offending symbols:

```json
{
  "success": false,
  "error": "schematic_load_failed",
  "flatSymbols": ["LIB:PART"],
  "message": "Schematic load failed for ...: embedded flat lib symbols [...]"
}
```

**Workaround:** run the `repair_flat_symbols` tool (if available) or wrap
each flat symbol's pins/graphics in a `(symbol "NAME_1_1" ...)` sub-unit.
Note that tools which previously returned partial/empty results on
unparseable schematics (e.g. `find_orphaned_wires`, hierarchical net
traversal, `sync_schematic_to_board`) now return errors instead.

---

### 8. `kicad-cli`-backed tool calls can exceed a 30s client timeout

**Status:** KNOWN — environment-dependent, workaround is "just retry"

**Symptoms:** `run_erc`, `export_*`, and `get_schematic_view` occasionally
report `Command timeout after 30s` even though the operation is not stuck.

**Root cause:** these tools shell out to `kicad-cli.exe`, which is a
heavyweight process — on a machine with limited free RAM and/or active
antivirus file scanning, cold-starting it alone measured **~30–31s real
time** (verified 2026-09-17, `time kicad-cli sch erc ...`), independent of
schematic size (a 6-component and an 18-net schematic both took the same
~30s). This lands right at or just over a 30s client-side tool-call
timeout, so the *first* call after a file change often times out while the
underlying `kicad-cli` process completes successfully moments later.

**Workaround:** on timeout, simply retry the same call once — it typically
returns immediately on retry. Don't treat a single timeout as a sign the
schematic or the server is broken; cross-check with the file on disk
(e.g. `list_schematic_components`) if in doubt.

---

### 9. `get_schematic_view`/`get_board_2d_view`'s PNG conversion can silently fall back to raw SVG text

**Status:** KNOWN — Windows-specific dependency gap

**Symptoms:** `get_schematic_view` and `get_board_2d_view` (default
`format: "png"`) return a message `No PNG converter available —
returning SVG. Install pymupdf, inkscape, or imagemagick.` followed by the
full SVG as text — for a modest schematic/board this can be 150k+
characters, unusable for visual inspection and expensive in an LLM
context window. Both tools share the same PNG conversion path, so this
hits either one.

**Root cause:** the server's PNG path needs `cairosvg`, which itself needs
the native `libcairo-2.dll`. On Windows, `pip install cairosvg` installs
the Python wheel but does **not** bundle that native DLL (no system GTK3
runtime provides it by default), so `import cairosvg` raises `OSError: no
library called "cairo-2" was found` even though the package shows up in
`pip list`. Confirmed reproducible in both the project `.venv` and KiCad's
bundled Python (2026-09-17).

**Workaround used during Phase 2 validation:** `pip install pymupdf` into
the project `.venv` (pure-Python wheel, no native DLL gap) and rasterize
the `export_schematic_svg` output locally:

```python
import fitz  # pymupdf
page = fitz.open("schematic.svg")[0]
page.get_pixmap(matrix=fitz.Matrix(4, 4)).save("schematic.png")
```

**Real fix (not yet done):** either bundle/vendor a working cairo runtime
for Windows, or switch `get_schematic_view`'s own PNG path to use
`pymupdf` instead of `cairosvg` — pymupdf does not have this native-DLL
dependency problem.

---

### 10. `sync_schematic_to_board` can partially persist on a save-guard conflict

**Status:** KNOWN — workaround is "just call it again"

**Symptoms:** `sync_schematic_to_board` reports success with footprints
and most nets added, but also returns a warning: `Auto-save refused: the
on-disk PCB file's contents changed externally since this MCP session
loaded it ... the in-memory mutation has NOT been written to disk.` A
follow-up call against the same (reloaded) board then reports far fewer
additions than expected — e.g. `0 footprints added, 1 nets added` — even
though nothing about the schematic changed.

**Root cause (observed, not fully diagnosed):** the main footprint/pad
sync appears to complete and save, but a secondary write (net info,
likely the same `net_settings` JSON-merge path described in issue #6)
gets caught by the external-change guard separately and is dropped
silently rather than erroring the whole call. Net effect: the board can
end up momentarily missing one net (observed: a `PWR_FLAG` net) after an
otherwise-successful-looking sync.

**Workaround:** if `sync_schematic_to_board` returns an
`autoSave.saved: false` / "changed externally" warning, don't treat the
reported counts as final — reload the board (`open_board`) and call
`sync_schematic_to_board` again with the same arguments. It's idempotent;
the second call reports only what was actually still missing (in the
observed case, exactly the one dropped net) and completes cleanly with no
further warning.

**More generally (field-tested across a whole PCB build session):** once
any `kicad-cli`-backed tool (DRC, gerber/PDF export, 2D view, ...) reads
the board file mid-session, the *next* board-mutating call is likely to
hit this same "changed externally" guard, even though nothing outside
the session actually edited the file — reading alone seems to be enough
to update the tracked mtime in some cases. In-memory mutations still
apply correctly across repeated guard trips (verified: 7 sequential
`delete_trace` calls each reported the warning but all 7 nets' traces
were genuinely gone by the time of the next `autoroute`), so it's safe to
keep issuing mutating calls in-memory and do a single `save_board
{force: true}` at the end of a sequence, rather than fighting the guard
after every call.

---

### 11. `suggest_placement` can propose a position that hangs a footprint off the board edge, and `check_courtyard_overlaps`'s boundary check can miss it

**Status:** KNOWN — validate placements by hand, don't trust the boundary
check alone

**Symptoms:** after `suggest_placement(apply: true)`, `run_drc` reports a
`copper_edge_clearance` **error** (board edge clearance violated) plus
`silk_edge_clearance` warnings — even though `check_courtyard_overlaps`
was called on the exact same proposed positions beforehand and reported
`boundary_violations: []` (zero).

**Root cause (observed 2026-09-17):** on a 50×40mm board, `suggest_placement`
proposed a `Resistor_THT:R_Axial_DIN0207_L6.3mm...` at `(20.5, 3.5)`
rotation 90°. `get_component_list` on the applied board then showed that
footprint's real bounding box as `min_y: -7.735mm` — i.e. ~7.7mm of the
vertical resistor body extends **above the board's y=0 top edge**. The
pre-apply `check_courtyard_overlaps` validation call against that same
position did not flag it.

Separately, `get_component_list`'s reported `boundingBox` for that same
footprint+rotation did not match what `check_courtyard_overlaps` computed
for a different position with the same rotation (width `6.4365mm` vs.
`3.09mm` — height matched closely). One of the two is reading the wrong
geometry (courtyard polygon vs. some other extent); which one wasn't
pinned down this session.

**Workaround:** after `suggest_placement(apply: true)`, don't stop at a
clean `check_courtyard_overlaps` result — pull `get_component_list` (no
bounding-box filter) and manually confirm every footprint's `boundingBox`
falls inside the board outline with margin, *before* routing. If not,
`move_component` it clear of the edge (verify the new spot with
`check_courtyard_overlaps` first), delete any traces already routed to
it (`delete_trace {net: ...}`), and re-run `autoroute`. Always finish
with `run_drc` as the real ground truth — it caught what the placement
tools' own pre-checks missed.

**Second reproduction, sharper (2026-09-17, different board):** the same
exact `positions` override (`{"R1": [10, 8, 90]}`) returned
`boundary_violations: []` when passed together with 5 other refs in one
batched call, then returned a real violation for the *identical* ref and
coordinates moments later in an isolated single-ref call with nothing
else on the board changed in between. The check is not just occasionally
blind to real violations — it can give two different answers for the
same declared input depending on call context. Also note:
`get_pads`/`get_component_list`'s `position` field for a 2-pin THT part
(resistor, capacitor) is the **pad-1 anchor**, not the body center — a
90°-rotated resistor's second pad lands `position ± pitch` (e.g.
10.16mm), not `position ± half-height`. Computing a "safe" position by
assuming center-anchoring is exactly how this session first put a pad
off-board. **The only check that proved reliable both times:** pull
`get_pads` (or `get_component_list`) for the real, currently-applied
board state and confirm every pad/bbox coordinate by eye — never trust a
"clean" `check_courtyard_overlaps` result on its own, applied or virtual.

---

### 12. A footprint's own via/hole geometry can violate the board's default design rules

**Status:** BY DESIGN — not a bug, but easy to mistake for one

**Symptoms:** `run_drc` reports many `drill_out_of_range` **errors**
(e.g. 12 of them) all clustered at the same footprint's location,
immediately after `autoroute` on a freshly-synced board that otherwise
looked fine.

**Root cause:** some library footprints embed their own via/hole geometry
with tighter tolerances than a generic board's default design rules — for
example `RF_Module:ESP32-C3-WROOM-02`'s exposed-pad thermal/EMI ground
array uses a grid of 0.2mm-drill through-hole vias (`pad 19`, repeated),
while a freshly-created board's default `minHoleDiameter` is 0.3mm (a
common budget-fab-safe default). This isn't a placement or routing
mistake — it's the module's manufacturer-specified footprint genuinely
needing finer tolerances than the board's default rule set allows.

**Fix:** when a chosen part's datasheet/footprint calls for finer
tolerances, relax the board's design rules to match — don't fight the
footprint. `set_design_rules({minHoleDiameter: 0.2})` (matched to
whatever the offending footprint actually needs; check the DRC message's
"actual" value) resolved this cleanly with no other side effects
(verified: 12 errors → 0 after the change, same board, same routing).
Before finalizing a design that needs sub-0.3mm holes, confirm your
target fab house actually supports that tolerance — it's a real
manufacturing constraint, not just a KiCad setting.

---

## Recently Fixed (v2.2.0 - v2.2.3)

### B.Cu Footprint Routing (Fixed v2.2.3)

- `route_pad_to_pad` now correctly detects B.Cu footprints and inserts vias
- KiCAD 9 SWIG `pad.GetLayerName()` always returned F.Cu for flipped footprints -- fixed using `footprint.GetLayer()`

### B.Cu Placement Hang (Fixed v2.2.3)

- Placing footprints on B.Cu no longer causes ~30s freeze
- Fix: call `board.Add()` before `Flip()`

### Board Outline Rounded Corners (Fixed v2.2.3)

- `add_board_outline` now correctly applies cornerRadius for rounded_rectangle shape

### Project-Local Library Resolution (Fixed v2.2.2)

- `add_schematic_component` and `place_component` now search project-local sym-lib-table and fp-lib-table
- Previously only global KiCAD library paths were searched

### Template File Corruption (Fixed v2.2.2)

- Removed invalid `;;` comment lines from template schematics
- Restored KiCAD 9 format version (20250114) in templates

### copy_routing_pattern Empty Results (Fixed v2.2.2)

- Added geometric fallback when pads have no net assignments

### Schematic Component Corruption (Fixed v2.2.1)

- `add_schematic_component` no longer corrupts .kicad_sch files
- Rewritten to use text manipulation instead of sexpdata formatting

### SWIG/UUID Comparison Bugs (Fixed v2.2.0)

- Fixed SwigPyObject UUID comparison
- Fixed SWIG iterator invalidation after board.Remove()
- Added board.SetModified() to prevent dangling pointer crashes

---

## Reporting New Issues

If you encounter an issue not listed here:

1. **Check MCP logs:** `~/.kicad-mcp/logs/kicad_interface.log`
2. **Enable developer mode:** Set `KICAD_MCP_DEV=1` to capture session logs
3. **Check KiCAD version:** `python3 -c "import pcbnew; print(pcbnew.GetBuildVersion())"` (must be 9.0+)
4. **Try the operation in KiCAD directly** -- is it a KiCAD issue?
5. **Open a GitHub issue** with:
   - Error message and log excerpt
   - Steps to reproduce
   - KiCAD version and OS
   - MCP session log (from `logs/` folder if dev mode is enabled)

---

## General Workarounds

### Server Will Not Start

```bash
# Check Python can import pcbnew
python3 -c "import pcbnew; print(pcbnew.GetBuildVersion())"

# Check paths
python3 python/utils/platform_helper.py
```

### Commands Fail After Server Restart

```
# Board reference is lost on restart
# Always run open_project after server restart
```

### KiCAD UI Does Not Show Changes (SWIG Mode)

```
# File > Revert (or click reload prompt)
# Or: Close and reopen file in KiCAD
# Or: Use IPC backend for automatic updates
```

### IPC Not Connecting

```
# Ensure KiCAD is running
# Enable IPC: Preferences > Plugins > Enable IPC API Server
# Have a board open in PCB editor
# Check socket exists: ls /tmp/kicad/api.sock
```

---

**Need Help?**

- Check [IPC_BACKEND_STATUS.md](IPC_BACKEND_STATUS.md) for IPC details
- Check logs: `~/.kicad-mcp/logs/kicad_interface.log`
- Open an issue on GitHub
