# CORVEX — Evidence Console

> Passive OT/ICS security monitoring with tamper-evident evidence chains.  
> **Zero dependencies. One HTML file. Real detection logic.**

[![Live Demo](https://img.shields.io/badge/demo-live-1a1a2e?style=flat-square)](https://shree-aru.github.io/corvex-console/)
[![License: MIT](https://img.shields.io/badge/license-MIT-1a1a2e?style=flat-square)](LICENSE)
[![STIX 2.1](https://img.shields.io/badge/export-STIX%202.1-1a1a2e?style=flat-square)](#stix-21-export)

---

## What is CORVEX?

CORVEX is a **passive OT (Operational Technology) security monitoring console** designed for air-gapped industrial environments. It links network events, physical sensor readings, and detector reasoning into a single, immutable evidence record — before an operator ever sees the alert.

**Core principle:** Every alert arrives with its proof attached.

### Key Features

| Feature | Description |
|---|---|
| **Evidence-First Alerts** | Each alert bundles the network event, sensor data, and detector reasoning into one record |
| **SHA-256 Hash Chain** | Real tamper-evident vault using Web Crypto API — every record chains to the previous |
| **Live Detection Engine** | 5 detector types: physical divergence, unauthorized writes, replay sequences, rogue devices, timing drift |
| **Attack Correlation** | Automatically detects multi-stage campaigns (recon → exploitation → impact) |
| **STIX 2.1 Export** | Industry-standard threat intelligence format for SOC integration |
| **Log Import** | Drag-and-drop Suricata EVE JSON and Zeek logs for real-data analysis |
| **Network Topology** | Live SVG map with severity-highlighted nodes |
| **Operator Workflow** | Classify (benign/suspicious/malicious), add notes, sign dispositions |
| **Zero Dependencies** | Single HTML file, no build step, no server, runs anywhere |

---

## Live Demo

**→ [https://shree-aru.github.io/corvex-console/](https://shree-aru.github.io/corvex-console/)**

The demo runs a live simulation engine that generates realistic OT security alerts every 20-50 seconds. All features are fully functional.

---

## Quick Start

```bash
# Clone and open — that's it
git clone https://github.com/shree-aru/corvex-console.git
cd corvex-console
open index.html
```

No build step. No `npm install`. No server. Just open in a browser.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                   CORVEX Console                     │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌───────────────────┐  │
│  │ Suricata │  │   Zeek   │  │   MQTT Sensors    │  │
│  │ EVE JSON │  │ conn.log │  │   (ESP32 edge)    │  │
│  └────┬─────┘  └────┬─────┘  └────────┬──────────┘  │
│       │              │                 │              │
│       └──────────────┼─────────────────┘              │
│                      ▼                                │
│  ┌───────────────────────────────────────────────┐   │
│  │            Detection Engine                    │   │
│  │                                                │   │
│  │  ┌─────────────┐  ┌──────────────────────┐    │   │
│  │  │ Rule Engine │  │ Baseline Correlation  │    │   │
│  │  │ (determin.) │  │ (statistical model)   │    │   │
│  │  └─────────────┘  └──────────────────────┘    │   │
│  │                                                │   │
│  │  Detection Rules:                              │   │
│  │  • UNAUTHORIZED_WRITE  • ROGUE_DEVICE          │   │
│  │  • REPLAY_SEQUENCE     • TIMING_DRIFT          │   │
│  │  • PHYSICAL_DIVERGENCE • IT_TO_OT_RELATION     │   │
│  └───────────────────────────────────────────────┘   │
│                      │                                │
│                      ▼                                │
│  ┌───────────────────────────────────────────────┐   │
│  │           Evidence Vault (SHA-256)             │   │
│  │                                                │   │
│  │  ┌────────┐    ┌────────┐    ┌────────┐       │   │
│  │  │ Rec N  │───▶│ Rec N+1│───▶│ Rec N+2│       │   │
│  │  │ hash_n │    │ hash_n+1│   │ hash_n+2│      │   │
│  │  └────────┘    └────────┘    └────────┘       │   │
│  │                                                │   │
│  │  Each record: data + prev_hash → SHA-256       │   │
│  │  Tamper-evident: break one → break all after   │   │
│  └───────────────────────────────────────────────┘   │
│                      │                                │
│              ┌───────┼───────┐                        │
│              ▼       ▼       ▼                        │
│         ┌────────┐ ┌──────┐ ┌──────────┐             │
│         │ Vault  │ │ STIX │ │ Incident │             │
│         │  JSON  │ │ 2.1  │ │  Report  │             │
│         └────────┘ └──────┘ └──────────┘             │
└─────────────────────────────────────────────────────┘
```

### Detection Pipeline

1. **Ingest** — Events arrive from Suricata (IDS), Zeek (network monitor), or MQTT (edge sensors)
2. **Detect** — Rule engine applies deterministic matches; baseline correlator checks statistical anomalies
3. **Correlate** — Multi-stage attack patterns are identified across related alerts
4. **Seal** — Evidence is hashed into an append-only chain using Web Crypto SHA-256
5. **Present** — Operator sees the alert with all proof pre-linked in the triage queue

### Monitored Protocols

| Protocol | Port | Detection Capability |
|---|---|---|
| Modbus TCP | 502 | Function code monitoring, unauthorized writes, replay detection |
| MQTT + TLS | 8883 | Physical sensor divergence, edge device authentication |
| HTTP/RDP | 80/3389 | IT-to-OT cross-zone traffic detection |

---

## Features in Detail

### SHA-256 Hash Chain

Every evidence record is hashed using the browser's native `crypto.subtle.digest('SHA-256', ...)` API. Each record includes the hash of the previous record, creating a mathematical chain:

```
Record_N.hash = SHA-256(Record_N.data + Record_N-1.hash)
```

Click **Verify Chain** to validate the entire chain in real time.

### STIX 2.1 Export

Export alerts as industry-standard [STIX 2.1](https://docs.oasis-open.org/cti/stix/v2.1/stix-v2.1.html) bundles containing:
- `identity` — CORVEX system identity
- `indicator` — One per alert with STIX patterns, confidence scores, and labels
- `infrastructure` — One per monitored asset
- `sighting` — Links indicators to infrastructure with timestamps

### Log Import

Drag and drop real IDS log files onto the console:
- **Suricata EVE JSON** — Parses `event_type: "alert"` and Modbus protocol events
- **Zeek conn.log** — Parses tab-separated connection records

Sample files are provided in the [`sample-data/`](sample-data/) directory.

### Attack Correlation

The engine automatically detects multi-stage attack campaigns:

| Pattern | Stages | Campaign Name |
|---|---|---|
| Full OT compromise | Rogue Device → Unauthorized Write → Physical Divergence | Full-chain OT compromise |
| Initial access | Rogue Device → Unauthorized Write | Reconnaissance → Exploitation |
| Evasion | Replay Sequence → Timing Drift | Evasion + Interference |

---

## Project Structure

```
corvex-console/
├── index.html              # Main console (deployed to GitHub Pages)
├── corvex-console.html     # Source mirror
├── README.md               # This file
├── LICENSE                  # MIT License
├── ARCHITECTURE.md          # Technical deep-dive
├── sample-data/
│   ├── suricata-eve.json   # Sample Suricata EVE JSON for import testing
│   └── zeek-conn.log       # Sample Zeek connection log for import testing
└── CORVEX-Implementation-Plan.pdf
```

---

## Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Rendering** | Vanilla HTML/CSS/JS | Zero-dependency, air-gap compatible |
| **Typography** | Archivo + IBM Plex Mono | Editorial monochrome design language |
| **Cryptography** | Web Crypto API | Native SHA-256 for hash chain |
| **Export** | STIX 2.1 (JSON) | Industry-standard threat intelligence |
| **Deployment** | GitHub Pages | Static hosting, no server required |
| **Storage** | localStorage | Client-side persistence for alert states |

---

## Keyboard Shortcuts

| Key | Action |
|---|---|
| `j` / `↓` | Next alert |
| `k` / `↑` | Previous alert |

---

## License

MIT — see [LICENSE](LICENSE) for details.

---

<p align="center">
  <strong>CORVEX Labs</strong> · Passive OT Monitoring · v0.1
</p>
