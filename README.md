# KiCad-Ai-Agent

Drive [KiCAD](https://www.kicad.org/) end-to-end from an AI assistant —
schematic, PCB layout, routing, electrical/design-rule checks, and
manufacturing files — using natural language, via the [Model Context
Protocol](https://modelcontextprotocol.io/).

This is a fork of [mixelpixx/KiCAD-MCP-Server](https://github.com/mixelpixx/KiCAD-MCP-Server)
(MIT-licensed), rebranded and validated end-to-end. See
[NOTICE.md](NOTICE.md) for full upstream credit. If you want a
from-scratch Rust rewrite instead, upstream also maintains
[Konnect](https://github.com/mixelpixx/Konnect) (AGPL-3.0, with commercial
licensing for businesses) — this fork deliberately stays on the original
Python/TypeScript MIT base instead, chosen for permissive licensing over
Konnect's newer feature set.

[![CI/CD Pipeline](https://github.com/manoj020218/KiCad-Ai-Agent/actions/workflows/ci.yml/badge.svg)](https://github.com/manoj020218/KiCad-Ai-Agent/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## What it actually does

You describe a circuit in plain English; the agent calls precise MCP
tools — the same operations you'd otherwise do by hand in KiCad's UI — to
place components, wire nets, run electrical checks, lay out and route a
board, run design-rule checks, and export the files a fab house needs.

**Key capabilities:**

- 146 tools across 13 categories with JSON Schema validation
- Complete schematic workflow: 27 tools, dynamic symbol loading (~10,000
  standard KiCad symbols), hierarchical sheets, ERC
- Complete PCB workflow: placement (including a force-directed heuristic
  optimizer), manual and pad-to-pad routing, a real Freerouting autorouter
  integration, DRC, copper pours, differential pairs
- Manufacturing output: Gerber, drill, PDF, SVG, 3D, BOM, position files
- JLCPCB parts integration (2.5M+ component catalog) and datasheet
  enrichment via LCSC
- Cross-platform: Linux, Windows, macOS

**What it honestly is not:** an autonomous PCB-layout AI. The placement
optimizer is a real, useful heuristic — not a router, not ML-based, and
its own pre-checks aren't always reliable (see
[Known Issues](docs/KNOWN_ISSUES.md)). Treat every AI-driven step the way
you'd treat a junior engineer's work: verify with ERC/DRC, don't just
trust a tool's own "success" message. See
[docs/PCB_DESIGN_WORKFLOW.md](docs/PCB_DESIGN_WORKFLOW.md#full-build-verification-checklist)
for the verification discipline this project actually follows.

## See it in action

[`examples/`](examples/) has 3 complete, validated projects — built
end-to-end through this server, each with 0 schematic ERC errors, 0 PCB
DRC errors, and exported Gerbers ready to send to a fab:

| Project | What it is |
| --- | --- |
| [`555-astable/`](examples/555-astable/) | Classic 555 timer astable oscillator driving an LED |
| [`opamp-divider/`](examples/opamp-divider/) | Non-inverting op-amp amplifier (gain 2) biased off a voltage divider |
| [`esp32c3-breakout/`](examples/esp32c3-breakout/) | ESP32-C3-WROOM-02 minimal breakout with reset/boot buttons and a UART header |

Each folder has the schematic, the routed PCB, rendered previews, and a
`gerbers/` directory with the manufacturing files.

## Which Claude client should I use, and what does it cost?

This is a **local** server — it runs on your own machine (KiCad + Node.js
+ this repo) and talks to whatever MCP-capable client you point at it.
Pick based on what you already have, not because one is "the" way to run
this:

| Client | Cost | Notes |
| --- | --- | --- |
| [Claude Desktop](https://claude.ai/download) | Free tier usable | Easiest way to try this out; the free tier has usage limits like any Claude product |
| [Claude Code](https://docs.claude.com/claude-code) | Requires a Pro/Max subscription or an Anthropic API key | No standalone free tier; best if you're already coding in a terminal |
| Your own agent on the [Anthropic API](https://docs.claude.com/en/api) | Pay-per-token | For custom integrations; you own the loop |
| [Cline](https://github.com/cline/cline) / [OpenCode](https://opencode.ai/) | Varies | Other MCP-capable editors/agents also work |

Whichever you choose, **this project doesn't run anywhere "for free at
scale"** — every design step is real inference against a real model, so
normal usage limits/costs for that client apply. Don't expect it to be
free if you're running it constantly across a team.

---

## Prerequisites

**KiCAD 9.0 or higher** — [kicad.org/download](https://www.kicad.org/download/), must include the Python module (`pcbnew`)

**Node.js 18+** — [nodejs.org](https://nodejs.org/)

**Python 3.9+** — comes bundled with KiCAD; required packages
(kicad-skip, cairosvg, Pillow, pydantic, etc.) are auto-installed from
`requirements.txt`

**An MCP client** — see the table above

Supported platforms: **Linux** (Ubuntu 22.04+, Fedora, Arch — primary,
fully tested), **Windows 10/11** (fully supported, automated setup
script), **macOS** (supported, setup script provided).

## Installation

### Linux (Ubuntu/Debian)

```bash
# Install KiCAD 9.0 or higher
sudo add-apt-repository --yes ppa:kicad/kicad-9.0-releases
sudo apt-get update
sudo apt-get install -y kicad kicad-libraries

# Install Node.js
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Clone and build
git clone https://github.com/manoj020218/KiCad-Ai-Agent.git
cd KiCad-Ai-Agent
npm install
pip3 install -r requirements.txt
npm run build

# Verify
python3 -c "import pcbnew; print(pcbnew.GetBuildVersion())"
```

### Windows 10/11

```powershell
git clone https://github.com/manoj020218/KiCad-Ai-Agent.git
cd KiCad-Ai-Agent
.\setup-windows.ps1
```

The script detects your KiCAD installation (machine-wide or per-user),
verifies prerequisites, installs dependencies, builds the project,
generates your MCP client configuration, and runs diagnostics. See
[Windows Troubleshooting](docs/WINDOWS_TROUBLESHOOTING.md) and
[Platform Guide](docs/PLATFORM_GUIDE.md) if anything doesn't detect
correctly.

### macOS

```bash
# Install KiCAD 9.0 from kicad.org/download/macos, and Node.js:
brew install node@20

git clone https://github.com/manoj020218/KiCad-Ai-Agent.git
cd KiCad-Ai-Agent

# Use KiCAD's bundled Python so pcbnew resolves correctly
/Applications/KiCad/KiCad.app/Contents/Frameworks/Python.framework/Versions/Current/bin/python3 -m venv venv --system-site-packages
source venv/bin/activate

npm install
pip install -r requirements.txt
npm run build
```

Or run `./setup-macos.sh` after the manual steps above to auto-generate
your Claude Desktop configuration (`chmod +x setup-macos.sh` first, or
run it via `bash setup-macos.sh`).

## Configuration

Add the server to your MCP client's config. For Claude Desktop:

- **Linux:** `~/.config/Claude/claude_desktop_config.json`
- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "kicad": {
      "command": "node",
      "args": ["/path/to/KiCad-Ai-Agent/dist/index.js"],
      "env": {
        "PYTHONPATH": "/path/to/kicad/python",
        "LOG_LEVEL": "info"
      }
    }
  }
}
```

The Windows/macOS setup scripts generate this for you automatically. Full
per-platform `PYTHONPATH` values, Claude Code / Cline / OpenCode config,
and troubleshooting live in
[docs/CLIENT_CONFIGURATION.md](docs/CLIENT_CONFIGURATION.md).

## Documentation

[**docs/INDEX.md**](docs/INDEX.md) is the full documentation index —
tool references, the PCB design workflow, the schematic authoring
playbook, JLCPCB/Freerouting integration guides, and troubleshooting.
Two worth calling out directly:

- [docs/HEADLESS_AUTHORING.md](docs/HEADLESS_AUTHORING.md) — field-tested
  practice for driving schematic authoring reliably (ERC triage, power
  flags, the connection-grid pitfall, symbol gotchas)
- [docs/KNOWN_ISSUES.md](docs/KNOWN_ISSUES.md) — real bugs and
  environment gaps found while validating this fork, with workarounds

## Available Tools

146 tools across project management, schematic design (27), PCB layout
and routing (13), design rules/DRC (8), footprint/symbol creation and
libraries, JLCPCB integration, the Freerouting autorouter (4), and
manufacturing export (8). Full per-tool parameter reference:
[docs/TOOL_INVENTORY.md](docs/TOOL_INVENTORY.md).

## Development

```bash
npm install
npm run build
npm run dev          # watch mode
npm run test:ts      # TypeScript tests
pytest                # Python tests
npm run lint          # lint TS + Python
```

Architecture details are in [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Troubleshooting

Start with [docs/KNOWN_ISSUES.md](docs/KNOWN_ISSUES.md) and
[docs/WINDOWS_TROUBLESHOOTING.md](docs/WINDOWS_TROUBLESHOOTING.md). Common
first checks:

```bash
node --version   # 18+
python3 -c "import pcbnew; print(pcbnew.GetBuildVersion())"
```

If a `kicad-cli`-backed tool call (`run_erc`, `export_*`) times out on a
slow machine, just retry it once before assuming something is broken —
see Known Issues for why.

## Contributing

Bug reports, feature requests, and PRs are welcome. See
[CONTRIBUTING.md](CONTRIBUTING.md) for guidelines, and
[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for how the server is put
together before adding a new tool.

## License

MIT — see [LICENSE](LICENSE). This project carries a second copyright
line for changes made in this fork, with the original upstream notice
intact; see [NOTICE.md](NOTICE.md) for the full attribution.

## Acknowledgments

- Built on the [Model Context Protocol](https://modelcontextprotocol.io/) by Anthropic
- Powered by [KiCAD](https://www.kicad.org/) open-source PCB design software
- Forked from [mixelpixx/KiCAD-MCP-Server](https://github.com/mixelpixx/KiCAD-MCP-Server) — see [NOTICE.md](NOTICE.md) for full credit, including the community contributors who built the tools this fork is validated on
- Uses [kicad-skip](https://github.com/kicad-skip) for schematic manipulation
- [JLCSearch API](https://jlcsearch.tscircuit.com/) by [@tscircuit](https://github.com/tscircuit/jlcsearch) and the [JLCParts Database](https://github.com/yaqwsx/jlcparts) by [@yaqwsx](https://github.com/yaqwsx) for JLCPCB parts data

## Citation

```bibtex
@software{kicad_ai_agent,
  title = {KiCad-Ai-Agent: AI-Assisted PCB Design (fork of KiCAD MCP Server)},
  author = {manoj020218},
  year = {2026},
  url = {https://github.com/manoj020218/KiCad-Ai-Agent},
  note = {Forked from mixelpixx/KiCAD-MCP-Server (MIT)}
}
```
