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

### 9. `get_schematic_view`'s PNG conversion can silently fall back to raw SVG text

**Status:** KNOWN — Windows-specific dependency gap

**Symptoms:** `get_schematic_view` (default `format: "png"`) returns a
message `No PNG converter available — returning SVG. Install pymupdf,
inkscape, or imagemagick.` followed by the full SVG as text — for a modest
schematic this can be 150k+ characters, unusable for visual inspection and
expensive in an LLM context window.

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
