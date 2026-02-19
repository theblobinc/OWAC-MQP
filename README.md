![Project Logo](docs/Data-Diode.png)
````markdown
# OWAC-MQP — One-Way Audio Control & Media Queue Protocol

OWAC-MQP is a **unidirectional** (“data-diode-like”) control protocol that lets an **air-gapped control workstation** send **authenticated, replay-resistant, fail-closed** control messages to an **internet-facing media node** over a simple **analog audio cable** (line-out → line-in).

**OWAC-MQP does *not* provide confidentiality.** Anyone who can record the audio can infer commands. The security posture is *authenticity-first*: nothing state-changing executes unless cryptographic verification and all policy gates pass.

- **Spec version:** `0.3.4-draft` (unified specification)
- **Protocol version:** `1`
- **Status:** Draft, implementable; **PHY/L2/CBOR/Auth profiles frozen**

---

## Why this exists

OWAC-MQP is designed for environments where:
- the **controller must remain air-gapped** (no inbound connectivity),
- the **receiver must be internet-facing** (and therefore higher-risk),
- you want a **physically one-way link** with **cryptographic authenticity**, **replay resistance**, and **high observability**,
- and you’re OK with **plaintext-over-audio**.

---

## System model

- **ACN (Air-Gapped Control Node):** operator workstation; no internet access; transmits only via analog line-out.
- **IMN (Internet-Facing Media Node):** receives audio via line-in; performs decode → verify → allow-list execute.
- **Trust boundary:** audio cable treated as **unidirectional**; **no acknowledgements**, no return channel.

---

## Threat model (high level)

OWAC-MQP assumes the receiver must handle:
- EMI/RF coupling, distortion, clipping, tone injection
- direct audio injection (conducted / near-field)
- replay of previously captured valid transmissions
- jamming / denial of service
- parser ambiguity exploitation

### Non-goals
- confidentiality of commands
- robust operation under continuous active jamming
- remote administration of the IMN over the same audio channel
- privacy of command metadata/intent

---

## Design goals

- **Deterministic parse:** one on-wire payload → exactly one transcript or reject
- **Authenticated execution:** state changes only after authenticity verification
- **Replay resistance:** receiver-enforced monotonic acceptance
- **Fail-closed under uncertainty:** reject ambiguous/invalid input
- **Observability:** interference/tamper becomes visible (logs/metrics/alerts)

---

## Protocol stack overview

1. **PHY**: analog audio profile (sampling, levels, pilot)
2. **Modem**: AFSK demod + NRZI bit recovery
3. **L2**: HDLC-like framing + bit stuffing + CRC/FCS
4. **L3 Message**: deterministic CBOR envelope + size limits
5. **Auth**: post-quantum signature (default) or optional MAC profile
6. **Replay/Policy gates**: strict counters, expiry, allow-lists
7. **Application**: media queue / transport / bounded “text intent” only
8. **Duotronics**: deterministic symbol layer (optional), canonically bound into the signed transcript

```mermaid
flowchart TD
A[PCM audio in] --> B[AFSK demod + NRZI]
B --> C[HDLC-like framing\n+ unstuffing + FCS]
C --> D[CBOR decode\n(deterministic rules)]
D --> E[TranscriptBytes recompute\n(message with auth present but with auth.sig omitted)]
E --> F[Verify signature/MAC]
F --> G[Replay + expiry + policy gates]
G --> H[Duotronics canonicalization (if present)]
H --> I[Allow-listed command execution]
C -->|FCS fail| X[Reject + log/metrics]
F -->|auth fail| X
G -->|replay/expiry/policy fail| X
H -->|duo fail| X
````

---

## At-a-glance: Frozen PHY/L2 profile (OWAC-PHY-AUD-A)

### Audio format

* **48,000 Hz**, **mono**, **16-bit signed PCM**
* Golden vectors MUST use exactly this format.

### Pilot tone (OWAC-PILOT-19K)

* **19.000 kHz** sine pilot mixed at **−30 dBFS RMS** (relative to payload RMS)
* Receiver tracks pilot amplitude/phase; abrupt loss/excursions must raise telemetry and set `link_suspect=true` for overlapping decisions.

### Modem (OWAC-AFSK-1200)

* **1200 baud**
* mark (1): **1200 Hz**
* space (0): **2200 Hz**
* initial tone state at burst start: mark

### Line coding

* **NRZI**

  * data bit `0` ⇒ transition (toggle mark/space)
  * data bit `1` ⇒ no transition

### Framing (HDLC-like)

* flag: `0x7E`
* preamble: **≥ 32 consecutive flags** before each burst
* bit stuffing: insert `0` after any run of five `1` bits between flags
* **PPP FCS-16**

  * poly (reflected): `0x8408`, init `0xFFFF`, “good” `0xF0B8`
  * transmit FCS: low byte then high byte

### Optional availability-oriented FEC profiles (no negotiation)

* **Profile 1:** baseline (no FEC)
* **Profile 2:** RS(255,223) + interleave depth 2 (octet-domain)
* **Profile 3:** convolutional code rate 1/2, K=7, generators 171/133 (octal)

> **Important:** Profile selection is static and out-of-band. The receiver must decode using **only the configured profile**, and then verify that the decoded message’s `profile` field matches.

---

## Message format (Deterministic CBOR)

Top-level CBOR **map** with small unsigned integer keys:

| Key | Name      | Type | Required | Meaning                                               |
| --- | --------- | ---- | -------- | ----------------------------------------------------- |
| 0   | `v`       | uint | yes      | protocol version (MUST be `1`)                        |
| 1   | `profile` | uint | yes      | PHY/L2 profile ID                                     |
| 2   | `mid`     | map  | yes      | message identity + replay controls                    |
| 3   | `cmd`     | map  | yes      | typed command                                         |
| 4   | `auth`    | map  | yes      | signature/MAC container                               |
| 5   | `meta`    | map  | no       | optional metadata (limited)                           |
| 6   | `ext`     | map  | no       | extension container (unknown keys ignored but signed) |

### Deterministic CBOR rules (strict)

* no indefinite-length items
* map keys sorted by bytewise lexicographic order of **deterministic encodings**
* minimal integer encodings only
* duplicate keys invalid
* unknown keys in protocol-defined maps (`mid`, `cmd`, `auth`, `meta`) **reject** (except `ext`)

Receivers MUST verify determinism by **decode → re-encode deterministically → byte-compare**.

### `mid` (message identity)

`mid` is a map:

* `0`: `epoch` (uint64)
* `1`: `ctr` (uint64) — monotonic counter for replay resistance
* `2`: `sid` (bstr, **16 bytes**) — sender identity
* `3`: `exp` (uint) OPTIONAL but RECOMMENDED — Unix seconds expiry

### `auth` (authenticator)

`auth` is a map:

* `0`: `alg` (uint) algorithm enum
* `1`: `kid` (bstr, **16 bytes**) receiver-pinned key id
* `2`: `sig` (bstr) signature/MAC bytes
* `3`: `ctx` (bstr, 0..32) OPTIONAL domain separation (bound into transcript)

Unknown `kid` → reject. Signature length mismatch → reject.

### TranscriptBytes (what is actually signed)

To compute `TranscriptBytes`:

1. take the full message
2. replace `auth` with the same map **but omit `auth.sig`**
3. deterministically CBOR-encode the result

Sign/verify is performed over the full `TranscriptBytes` byte string.

---

## Authentication profiles (defaults)

OWAC-MQP defines PQ signature profiles for IMN verification:

* **ML-DSA-65** (default command signing)

  * signature length: 3309 bytes
* **SLH-DSA-SHA2-128s** (critical ops / epoch changes)

  * signature length: 7856 bytes

Message size limits depend on `auth.alg`:

* alg=1 (ML-DSA-65): max message bytes 4096
* alg=2 (SLH-DSA-SHA2-128s): max message bytes 12288
* alg=3 (OTMAC profile): max message bytes 4096

---

## Replay resistance (execute-once)

The IMN enforces replay resistance using `(epoch, ctr, sid)` and persisted state.

* messages that do not advance replay state are rejected (or handled as benign duplicates under the duplicate policy)
* optional expiry `exp` is enforced with configurable clock skew allowance (default ±30s)
* optional “far future” guard MAY reject if `exp > now + MAX_LIFETIME` (recommended to prevent very-distant replay)

---

## Application commands (allow-listed base set)

`cmd.type` enum:

* `0`: `HEARTBEAT`
* `1`: `QUEUE_TRACK`
* `2`: `SUGGEST_TRACK`
* `3`: `REMOVE_FROM_QUEUE`
* `10`: `TRANSPORT_SET_STATE`  (stop/pause/play)
* `11`: `TRANSPORT_SEEK`       (bounded delta)
* `12`: `TRANSPORT_SET_RATE`   (0.5×..2.0× default)
* `13`: `TRANSPORT_NEXT`
* `14`: `TRANSPORT_PREV`
* `20`: `TEXT_INPUT` (bounded “text intent”, NOT arbitrary OS keystrokes)
* `30`: `POLICY_EPOCH_UPDATE` (typically SLH-DSA signed)
* `31`: `CATALOG_UPDATE`      (typically SLH-DSA signed)

Rules:

* unknown `cmd.type` → reject
* unknown keys in `cmd.args` → reject
* media references must be stable IDs from an allow-listed catalog
* no URLs, file paths, shell commands, or “interpretation” of arguments

---

## Observability (minimum requirements)

Implementations MUST expose at least:

* metrics (e.g., framing ok/fail, auth ok/fail, replay rejects, pilot faults, clipping windows, burst-too-long)
* structured logs for every candidate frame decision including:

  * timestamp, profile id, link health snapshot, decision, reject code (if reject)
  * if accept: `(epoch,sid,ctr)`, command type, transcript hash

---

## Reject reason codes (normative)

**L1 / Link health**

* `REJ_LINK_NOLOCK`
* `REJ_LINK_PILOT_FAULT`
* `REJ_LINK_CLIP`
* `REJ_BURST_TOO_LONG`

**L2 / Framing**

* `REJ_L2_FRAMING`
* `REJ_L2_OCTET_ALIGN`
* `REJ_L2_FCS_FAIL`

**L3 / CBOR**

* `REJ_CBOR_PARSE`
* `REJ_CBOR_NOT_DET`
* `REJ_CBOR_INDEFINITE`
* `REJ_SIZE_LIMIT`

**Auth**

* `REJ_KID_UNKNOWN`
* `REJ_SIG_LEN`
* `REJ_AUTH_FAIL`

**Replay / Time**

* `REJ_EXPIRED`
* `REJ_EXP_TOO_FAR`
* `REJ_REPLAY`

**Policy / Application**

* `REJ_UNKNOWN_CMD`
* `REJ_BAD_ARGS`
* `REJ_ALLOWLIST_DENY`
* `REJ_RATE_LIMIT`
* `REJ_CTX_MISMATCH`
* `REJ_PROFILE_MISMATCH`

**Duotronics**

* `REJ_DUO_POLICY_UNKNOWN`
* `REJ_DUO_CANON_FAIL`
* `REJ_DUO_DESCRIPTOR_MISMATCH`
* `REJ_DUO_DEGENERATE`

**Song extension**

* `REJ_SONG_MISMATCH`

---

## Conformance & golden vectors

Conformance MUST start at **raw PCM**, since the analog decode path is part of the attack surface.

Recommended vector pack layout:

```
owac-vectors/
  MANIFEST.json
  profiles/
    phy-aud-a/
      v0001_valid_heartbeat/
        audio.wav
        demod_bits.bin
        l2_frames.jsonl
        message.cbor
        transcript.cbor
        transcript.sha256
        auth.expected.json
      v0002_crc_fail/
      v0003_cbor_unsorted_keys/
      ...
    phy-aud-a-fec-rs16i2/
      v0001_valid_heartbeat/
      v0002_fec_decode_fail/
    phy-aud-a-fec-cc-1_2/
      v0001_valid_heartbeat/
      v0002_fec_decode_fail/
```

---

## Repository structure (suggested)

```
/spec/          # unified specification (PDF and/or Markdown)
/vectors/       # golden vectors pack
/reference/     # (optional) reference encoder/decoder + tools
/tools/         # (optional) analysis helpers, demod tools, vector builders
```

---

## Getting started (implementation checklist)

1. **Pick a profile** (e.g., Profile 1 baseline) and configure IMN statically.
2. Implement receiver pipeline gates in order:

   * pilot/clipping tracking → AFSK demod → NRZI decode → unstuff → FCS check
   * deterministic CBOR decode + deterministic byte-compare
   * recompute `TranscriptBytes` (omit `auth.sig`)
   * verify signature/MAC using pinned `kid`
   * replay/expiry/quorum/link-suspect policy gates
   * allow-listed command execution (bounded, idempotent)
3. Emit **reject reason codes + metrics + structured logs** for every decision.
4. Validate against **golden PCM vectors** and ensure transcript hashes match.

---

## Spec

The full unified specification lives in this repo (see `spec/`).

---

## Security notes

* This protocol is designed to **fail closed**. If anything is uncertain, **reject** and **log/alert**.
* OWAC-MQP is **plaintext over audio**; treat recordings as sensitive operational artifacts.
* Do not weaken strict parsing/determinism rules—those constraints are part of the security boundary.

---

## Contributing

Issues and PRs are welcome, especially for:

* conformance vectors & negative tests
* independent implementations for differential testing
* deployment hardening guides and real-world hardware notes

---

## License

MIT License. See `LICENSE`.

```
