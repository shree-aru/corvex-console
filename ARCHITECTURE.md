# CORVEX — Technical Architecture

## Design Philosophy

CORVEX is built on three principles:

1. **Evidence-first** — No alert fires without machine-readable proof attached
2. **Passive-only** — CORVEX never writes to the OT network. Detection is read-only via SPAN/TAP
3. **Air-gap compatible** — Single-file deployment, zero external dependencies at runtime

---

## Detection Engine

### Rule Engine (Deterministic)

Deterministic rules fire on exact pattern matches. No model, no training data, no false-positive tuning. They either match or they don't.

| Rule | Trigger | Severity |
|---|---|---|
| `UNAUTHORIZED_WRITE` | Write (FC 5/6/15/16) from unlisted source IP | High |
| `ROGUE_DEVICE` | New MAC address on OT segment | High |
| `REPLAY_SEQUENCE` | N consecutive byte-identical transactions | Medium |
| `IT_TO_OT_RELATION` | Cross-zone session outside baseline map | Medium |

### Baseline Correlator (Statistical)

Statistical detectors compare live traffic against a clean-window baseline. They require a training period and can abstain when coverage is too low.

| Detector | Method | Severity |
|---|---|---|
| `PHYSICAL_DIVERGENCE` | Speed-to-current curve deviation beyond calibrated envelope | Critical |
| `TIMING_DRIFT` | Rolling median of polling interval vs ±2σ baseline | Low |

### Multi-Stage Correlation

The correlation engine watches for alert sequences that match known attack patterns:

```
ROGUE_DEVICE → UNAUTHORIZED_WRITE → PHYSICAL_DIVERGENCE
= "Full-chain OT compromise"
```

This is implemented as a simple pattern matcher over the last 15 open alerts, running every 10 seconds.

---

## Evidence Vault

### Hash Chain Implementation

```javascript
async function sha256(str) {
  const buf = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(str));
  return 'sha256:' + Array.from(new Uint8Array(buf))
    .map(b => b.toString(16).padStart(2, '0')).join('');
}

// Each record chains to the previous
record.hash = await sha256(
  JSON.stringify({id, type, sev, asset, t, rule, state}) + previousRecord.hash
);
```

### Chain Verification

The "Verify Chain" button re-computes every hash from the genesis block forward and confirms:
- Each record's hash matches its recomputed value
- Each record's `prev` field matches the prior record's hash

If any record was modified, deleted, or reordered, verification fails immediately.

### Limitations

- Hash chains are **tamper-evident**, not tamper-proof. An attacker who replaces the entire chain can create a valid but forged history.
- For production use, the chain head should be anchored to an external append-only system (e.g., Git commit, external database, or public timestamp service).

---

## Data Import Pipeline

### Suricata EVE JSON

CORVEX parses Suricata's EVE JSON format (one JSON object per line):

```json
{"timestamp":"2026-08-16T15:04:16.902Z","event_type":"alert","src_ip":"192.168.10.77","dest_ip":"192.168.10.30","dest_port":502,"proto":"TCP","alert":{"signature":"ET SCADA Modbus Write","severity":2}}
```

Parsed fields: `event_type`, `src_ip`, `dest_ip`, `dest_port`, `alert.signature`, `alert.severity`, `app_proto`

### Zeek Logs

CORVEX parses Zeek's tab-separated log format:

```
1723824256.123456	Cjk2jl2	192.168.10.77	52341	192.168.10.30	502	tcp	modbus
```

Fields parsed by column index: timestamp, uid, src_ip, src_port, dst_ip, dst_port, proto, service

---

## STIX 2.1 Export

Exported bundles contain:

| STIX Type | Mapping |
|---|---|
| `identity` | CORVEX system identity |
| `indicator` | One per alert — includes STIX pattern, confidence (0-100), severity label |
| `infrastructure` | One per monitored asset |
| `sighting` | Links indicator to infrastructure with first/last seen timestamps |

All objects include `spec_version: "2.1"` and proper UUIDv4 identifiers via `crypto.randomUUID()`.

---

## Operator Workflow

### Disposition Process

1. Operator clicks **Acknowledge** on an open alert
2. Inline form appears with:
   - **Classification**: Benign / Suspicious / Confirmed Malicious
   - **Notes**: Free-text field for operator context
3. Operator clicks **Sign & Seal**
4. Disposition is recorded in the alert's facts
5. Hash chain is rebuilt to include the disposition
6. State persists to localStorage

### Operator Identity

On first visit, an operator ID is auto-generated (`operator-01` through `operator-99`) and stored in localStorage. This ID is attached to every disposition.

---

## Network Topology

The topology map is rendered as pure SVG (no external libraries). Node positions are fixed to match the documented OT network layout:

```
                    HMI-01
                   /      \
              PLC-01      PLC-02
             /      \         \
        MOTOR-01    ROGUE?    ROGUE?
```

Nodes are color-coded by the highest-severity open alert on that asset:
- **Filled black** = Critical
- **Dashed border** = High
- **Gray fill** = Medium
- **Empty** = No active alert

---

## Browser Compatibility

| Browser | Status |
|---|---|
| Chrome 90+ | ✅ Full support |
| Firefox 90+ | ✅ Full support |
| Safari 15+ | ✅ Full support |
| Edge 90+ | ✅ Full support |

Required APIs: `crypto.subtle`, `crypto.randomUUID`, `localStorage`, `Notification`, `AudioContext`
