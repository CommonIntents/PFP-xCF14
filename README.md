# PFP-xCF14 — Physical Feature Protocol

> **版本**：v1.0（Frozen Layer）
> **所属协议家族**：CI-144 Protocol Family
> **家族魔数**：`0xCF14`
> **子协议 ID**：`0x00`
> **总长度**：4 字节（32 bits）

---

## ⚠️ Authority Declaration (MUST READ)

**This repository is the PUBLICATION WINDOW, NOT the authority.**

The **Single Source of Truth (SSOT)** for this specification is maintained in:

> **[CommonIntents/BIND-19/docs/spec/pfp-xcf14.md](https://github.com/CommonIntents/BIND-19/blob/v2.0-rc.1/docs/spec/pfp-xcf14.md)**

All specification change PRs **MUST** be filed in the BIND-19 repository, updating `docs/spec/` + `src/` + `tests/` together (OpenSSL model: code and docs reviewed in the same PR).

**This repo does NOT accept specification change PRs.** Content here is synced from BIND-19.

| Role | Location |
|---|---|
| **Specification Authority (SSOT)** | [BIND-19/docs/spec/pfp-xcf14.md](https://github.com/CommonIntents/BIND-19/blob/v2.0-rc.1/docs/spec/pfp-xcf14.md) |
| **Reference Implementation** | [BIND-19/src/pfp.rs](https://github.com/CommonIntents/BIND-19/blob/v2.0-alpha/src/pfp.rs) |
| **Test Vectors** | [BIND-19/tests/test_vectors/](https://github.com/CommonIntents/BIND-19/tree/v2.0-rc.1/tests/test_vectors) |
| **Publication Window** | This repo |

**Corresponding BIND-19 version**: [`v2.0-rc.1`](https://github.com/CommonIntents/BIND-19/tree/v2.0-rc.1)

---

## Protocol Overview

PFP-xCF14 (Physical Feature Protocol) is the **frozen layer** of the CI-144 Protocol Family. It defines the physical feature metadata of a digital lifeform — posture, risk, environment — that can be read by hard real-time gates (like Tuck) without decrypting the payload.

### Design Philosophy

| Principle | Embodiment |
|---|---|
| **Extreme Energy Efficiency** | Only 4 bytes. Tuck reads only PFP for hard-real-time decisions, never decrypts payload |
| **Determinism First** | Fixed offsets, fixed lengths, fixed enums, no branching |
| **Physical Facts First** | All fields are sensor-driven. AI cannot modify PFP fields |
| **Extreme Decoupling** | Separated from security attestation (SAP). PFP does not depend on any crypto mechanism |
| **Frozen Immortality** | Once finalized, never changes. Like a stone, immortal |

---

## Byte Layout (4 bytes / 32 bits)

All fields are **plaintext, fixed-offset, fixed-length**. PFP is visible at the transport layer even when the entire frame is encrypted.

| Byte | Bits | Field | Values |
|---|---|---|---|
| 0-1 | 0-15 | `Family-Magic` | Fixed `0xCF14` (big-endian) |
| 2 | 0-1 | `Modality` | 0=COGNITIVE, 1=RENDER, 2=EXECUTIVE, 3=SENSOR_FEED |
| 2 | 2-3 | `Risk-Level` | 0=LOW, 1=MEDIUM, 2=CRITICAL, 3=CATASTROPHIC |
| 2 | 4-5 | `Body-Stance` | 0=SEATED, 1=STANDING, 2=MOVING, 3=UNKNOWN |
| 2 | 6-7 | `Proximity-Edge` | 0=SAFE, 1=WARNING, 2=DANGER, 3=CRITICAL_EDGE |
| 3 | 0 | `Output-Dest` | 0=INTERNAL, 1=EXTERNAL |
| 3 | 1 | `Override-Flag` | 0=NORMAL, 1=HARD_OVERRIDE |
| 3 | 2 | `Replay-Enable` | 0=DISABLED, 1=ENABLED |
| 3 | 3-7 | `Reserved` | Must be 0 |

### CATASTROPHIC Hard Override (Non-Negotiable Rule)

```
IF Risk-Level == CATASTROPHIC (3) AND Override-Flag == HARD_OVERRIDE (1)
THEN receiver MUST:
  1. Respond at physical layer first (event-driven, NO POLLING)
  2. In parallel, send emergency signal to human via any available channel
  3. This frame has priority over any local policy cache, user config, or AI scheduling
```

Detection is pure bitwise operations (~3 CPU cycles / ~0.3 ps):

```rust
fn is_catastrophic_override(bytes: &[u8; 4]) -> bool {
    let risk_level = (bytes[2] >> 2) & 0b11;
    let override_flag = (bytes[3] >> 1) & 0b1;
    risk_level == 3 && override_flag == 1
}
```

### Rule 6: Replay-Enable=0 Downgrade

When `Replay-Enable == 0`:
1. Effective risk level is forced to `MEDIUM` (regardless of original Risk-Level)
2. PAH-Signature verification is mandatory (compensate for missing replay protection)
3. Audit log MUST record `REPLAY_DISABLED` event
4. CATASTROPHIC hard override can never be triggered — fundamentally prevents high-risk physical attacks via replay

---

## Protocol Stack Position

```
[ 8-byte BIND-19 Header ] + [ PFP 4 bytes (optional) ] + [ SAP 28 bytes (optional) ] + [ Payload ]
```

PFP sits above the BIND-19 transport header and below the SAP security attestation layer. Hard real-time gates like Tuck read only PFP, never parsing SAP or payload.

---

## Relationship with SAP-xCF14

| Dimension | PFP-xCF14 | SAP-xCF14 |
|---|---|---|
| Layer | Frozen | Evolving |
| Length | 4 bytes | 28 bytes |
| Dependencies | None (stands alone) | Depends on PFP (cannot stand alone) |
| Tuck reads | Mandatory (hard-real-time decision) | Not required (optional verification) |
| Change frequency | Never changes (frozen) | Evolvable (v1/v2 can coexist) |
| Core value | Physical feature description | Security attestation (replay protection + signature) |

---

## Test Vectors

33 sets of test vectors for the entire CI-144 v2.0 protocol family are published at:

> [BIND-19/tests/test_vectors/ci-144-v2.0-test-vectors.json](https://github.com/CommonIntents/BIND-19/blob/v2.0-rc.1/tests/test_vectors/ci-144-v2.0-test-vectors.json)

PFP-specific vectors (5 sets in `pfp_codec` category):

| ID | Description |
|---|---|
| `pfp-all_zero` | All zeros (Cognitive/Low/Unknown/Safe/Internal/Normal/Replay-Disabled) |
| `pfp-all_one` | All ones (SensorFeed/Catastrophic/Moving/CriticalEdge/External/HardOverride/Replay-Enabled) |
| `pfp-typical_executive` | Typical executive frame (Executive/Medium/Standing/Warning/External) |
| `pfp-cognitive_low` | Cognitive low-risk frame |
| `pfp-render_critical` | Render high-risk frame |

Generate test vectors locally:
```bash
cd BIND-19
cargo run --example generate_test_vectors
```

---

## Reference Implementation

The official Rust reference implementation is maintained in BIND-19:

- [src/pfp.rs](https://github.com/CommonIntents/BIND-19/blob/v2.0-alpha/src/pfp.rs) — PFP encode/decode
- [src/frame.rs](https://github.com/CommonIntents/BIND-19/blob/v2.0-alpha/src/frame.rs) — Full frame assembly
- [src/catastrophic.rs](https://github.com/CommonIntents/BIND-19/blob/v2.0-alpha/src/catastrophic.rs) — CATASTROPHIC event bus + audit log
- [examples/tuck_integration.rs](https://github.com/CommonIntents/BIND-19/blob/v2.0-alpha/examples/tuck_integration.rs) — Tuck hard-real-time decision path example

---

## Version History

| Version | Date | Changes |
|---|---|---|
| v1.0 | 2026-08-29 | Initial frozen version. 4-byte structure, 8 fields, Family-Magic 0xCF14. |

---

## License

Apache 2.0 — see [LICENSE](LICENSE).

---

**This repository is a publication window. For the authoritative specification, go to [CommonIntents/BIND-19/docs/spec/pfp-xcf14.md](https://github.com/CommonIntents/BIND-19/blob/v2.0-rc.1/docs/spec/pfp-xcf14.md).**
