# One-Way Audio Control and Media Queue Protocol (OWAC‑MQP)  
## Unified Specification: Architecture, Duotronics Symbol Layer, and v0.2 Implementable Details

---

# Part I: Architecture and Design Principles

## Abstract and Design Intent

This document specifies **OWAC‑MQP** (One‑Way Audio Control & Media Queue Protocol), a unidirectional control protocol that allows an air‑gapped operator workstation (ACN) to issue commands to an internet‑facing media server (IMN) over a direct analog audio cable. The primary requirement is **fail‑closed tamper/interference detectability**: any manipulation of the analog signal, whether by EMI, injection, replay, or parser ambiguity, must cause the receiver to **reject** the command and produce **observable evidence** (logs, alerts, metrics). The design explicitly does **not** require confidentiality; it prioritizes **authenticity, replay resistance, deterministic parsing, and explicit gates**.

The core architectural choice is to treat the audio cable as a **physical unidirectional “diode‑like” path**, paired with a protocol stack that makes “success” rare unless every deterministic validation gate passes. (In security guidance, “diodes” are used to enforce one-way flows in unidirectional gateways, reducing the ability to use the same path for both intrusion and exfiltration.) citeturn7search3

This unified specification integrates three previously separate documents: the initial protocol design, the **Duotronics representation layer** (a formal symbol system based on polygonal cell configurations), and the **v0.2 implementable addendum** which freezes all physical, link, and cryptographic parameters for cross‑implementation interoperability. The document is organized in two parts: Part I describes the architectural layers, threat model, and design principles; Part II provides the normative frozen constants, message schemas, and test vector requirements necessary for building a conforming implementation.

## 1. System Context and Threat Model

### 1.1 Nodes and Trust Boundaries

- **Air‑Gapped Control Node (ACN)**: The CNC/operator workstation. It has no network path to the internet‑facing server and can only transmit via its analog line‑out.
- **Internet‑Facing Media Node (IMN)**: The web/media server. It hosts public services and receives audio from its line‑in. It runs a strict decoder + verifier + allow‑listed actuator.

The only interconnect is a **one‑way analog audio link** (ACN → IMN). There is no protocol‑level acknowledgement or response channel.

### 1.2 Threat Model Summary

The system must assume:

- **EMI/RF coupling** into the cable or audio front‑end, causing amplitude distortion, clipping, and injected tones.
- **Audio injection** (near‑field acoustic, inductive, or direct electrical coupling).
- **Replay** of previously captured valid transmissions.
- **Jamming** – availability is not required under jamming, but ambiguous input must be rejected and degradation made observable.
- **Parser ambiguity** as a first‑class attack surface: “unknown” must not silently coerce into “valid.”

### 1.3 Design Invariants

1. **Deterministic Parse**: A given on‑wire payload must canonicalize to **exactly one** internal transcript, or it is rejected.
2. **Cryptographic/authentic acceptance**: No state‑changing action may execute unless it passes an authenticity check (digital signature or information‑theoretic MAC) and replay controls. (Digital signatures are explicitly described by NIST as mechanisms to detect unauthorized modifications and authenticate the signatory.) citeturn4search0
3. **Reject‑by‑default under uncertainty**: Any failure in framing, error detection, authenticity verification, canonicalization, parameter validation, or policy checks yields **rejection + logging**.

## 2. Physical and Link Layer Design (Conceptual)

This section outlines the principles of the analog audio link. Exact constants (sampling rate, modulation, framing, CRC) are frozen in Part II, §6.

### 2.1 Physical Layer Profile

The analog audio layer is intentionally simple, but “aerospace‑grade” thinking demands explicit constraints and testability. The physical profile SHALL specify:

- A fixed sampling rate and bit depth for receiver capture (see Part II for values).
- A fixed operating amplitude window with headroom to avoid clipping; clipping SHALL be logged as an interference indicator.
- Short, shielded cabling; ferrites and isolation are recommended where practical to reduce susceptibility.
- A receiver‑side “link health monitor” that continuously records amplitude statistics, clipping events, and lock/unlock behavior.

### 2.2 Link Layer: Robust Framing

A key reliability and safety goal is that random noise should almost never become a valid frame. Mature HDLC‑style framing is a strong baseline because it combines:

- A unique frame delimiter (“flag” pattern),
- Bit stuffing to prevent accidental delimiter appearance,
- A frame check sequence (FCS/CRC) to detect corruption.

AX.25, a well‑known HDLC‑derived link‑layer protocol, documents these mechanisms precisely: a flag is `01111110` (0x7E), bit stuffing inserts a `0` after five consecutive `1` bits, and the receiver discards stuffed zeros; invalid frames include those not bounded by opening/closing flags or not octet‑aligned. citeturn10view1turn10view0

OWAC‑MQP therefore uses an **HDLC‑like envelope** with:

- Preamble and acquisition sequence (see Part II for length).
- Frame delimiter: a strong sync marker (HDLC flag).
- Error detection: CRC‑16 or CRC‑32 at the link layer, used only as a **corruption detector**, not as an authenticity mechanism.

### 2.3 Modulation Choice

For “aerospace‑grade” assurance, simplicity and testability usually beat raw spectral efficiency. Audio‑frequency shift keying (AFSK) is attractive because it is robust, easy to characterize, and historically proven over “voice‑grade” links. Packet‑radio Bell‑202 style AFSK is widely used and well documented. citeturn6search6

OWAC‑MQP defines one baseline modem profile (AFSK‑1200) and optional higher‑rate profiles, all frozen in Part II.

### 2.4 Link Health Metrics as First‑Class Security Signals

Because the channel is analog and one‑way, the receiver must treat measurements as evidence. The IMN SHALL track and expose at least:

- Lock/unlock events.
- CRC failure rate.
- Symbol error indicators.
- Clipping/AGC anomalies.

These metrics serve two purposes: they help operators diagnose interference, and they provide an observable audit trail consistent with “tampering becomes invalid and visible.”

## 3. Duotronics: Representation and Symbol Layer

This section specifies **Duotronics** as the **representation/symbol layer** inside OWAC‑MQP. The design goal is **aerospace‑grade determinism**: the receiver must be able to canonicalize, hash, and verify a sender’s intended command transcript **without ambiguity**, so that any interference, manipulation, or “parser creativity” becomes a **hard reject** at the receiver. This aligns with OWAC‑MQP’s “tamper ⇒ invalid” philosophy.

Duotronics contributes three high‑leverage properties: determinism under symmetry, strong canonical identity beyond a single label integer, and gate‑based rejection rather than interpretation. Deterministic serialization is anchored to standards such as **RFC 8785 (JSON Canonicalization Scheme, JCS)** for “hashable” JSON, and **RFC 8949**’s deterministic encoding for CBOR. citeturn2search4turn2search1turn6search0

### 3.1 Role of Duotronics in the Protocol Stack

Duotronics can be placed either as an internal sublayer of the **message/auth** layer (where Duotronics outputs become the signed “transcript”) or as an intermediate layer between link framing and message/auth. In either case, its purpose is to enforce deterministic meaning under declared symmetry and policies.

### 3.2 Formal Definition of Duotronics

#### 3.2.1 Core Objects and Open Parameters

Duotronics requires fixing the following open parameters (each must be declared and versioned in‑protocol):

- Polygon size(s) `n` (e.g., `n=6` for hex).
- Vertex weight vector `w ∈ ℤ^n` and center weight `w_c ∈ ℤ` (typically `w_c=1`).
- Dot alphabet(s): baseline binary dots `d_k ∈ {0,1}`, center dot `c ∈ {0,1}`; optional extensions allow multi‑valued, signed, or attached features.
- Symmetry policy: rotations only (`C_n`) vs rotations + reflections (`D_n`).
- Canonicalization policy: ordering rules and tie‑breakers.
- Label mapping policy: modular base rules, offsets, and display/semantic separation.

#### 3.2.2 Polygon Cell Configurations

Let `n ≥ 3`. A **polygon cell configuration** (baseline dialect) is:

- Center dot indicator `c ∈ {0,1}`
- Vertex dot vector `d = (d_0, …, d_{n−1})` with `d_k ∈ {0,1}`
- Optional attachments `a` in a declared attachment space `𝒜`.

Configuration space:  
\[
\mathcal{X}_{n} = \{0,1\}\times\{0,1\}^{n}\times\mathcal{A}.
\]

#### 3.2.3 Symmetry Group Actions

Let `G` be the chosen symmetry group:

- Rotation‑only: `G = C_n`, elements `r^t` for `t ∈ {0,…,n−1}`
- Rotation + reflection: `G = D_n` with `2n` elements.

For rotation by `t`:  
\[
(r^{t}\cdot d)_k = d_{(k-t)\bmod n}.
\]  
For a reflection `s` (convention: `k ↦ (-k) mod n`):  
\[
(s\cdot d)_k = d_{(-k)\bmod n}.
\]

Two configurations are equivalent if `∃ g∈G: g·x = y`.

#### 3.2.4 Orbit Size

The orbit of `x` is `Orb(x) = {g·x : g∈G}`. Orbit size relates to stabilizer:  
\[
|\mathrm{Orb}(x)| = \frac{|G|}{|\mathrm{Stab}(x)|}.
\]  
Orbit size is a valuable descriptor field for detecting symmetries and distinguishing canonical forms.

#### 3.2.5 Raw Sum and Label Policies

Define integer weights `w_c` and `w = (w_0,…,w_{n−1})`. Raw sum:  
\[
S(d,c; w, w_c) = c·w_c + \sum_{k=0}^{n-1} d_k w_k.
\]  
Label mapping policy `Π_label` maps `S` to a primary label, e.g., `Z_primary = S + o` or `(S+o) mod M`.

#### 3.2.6 Descriptor Tuple

A minimal descriptor tuple includes:

| Field        | Type         | Meaning                                      |
|--------------|--------------|----------------------------------------------|
| `Z_primary`  | int          | Primary label from `Π_label`                 |
| `family_id`  | string       | Identifies polygon family & weight policy    |
| `canon_hash` | bytes/hex    | Canonical transcript hash                    |
| `parity_bit` | bit          | e.g., `S mod 2`                              |
| `dot_count`  | int          | `c + Σ d_k`                                  |
| `orbit_size` | int          | Size of symmetry orbit under chosen `G`      |
| `optional_moments` | object | Additional invariants                         |

### 3.3 Canonicalization and Hashing

#### 3.3.1 Canonicalization Operator

A deterministic operator `Can: 𝒳_n → 𝒳_n` such that:

- Idempotence: `Can(Can(x)) = Can(x)`
- Invariance: for all `g ∈ G`, `Can(g·x) = Can(x)`
- Total and deterministic.

Typical rule: enumerate all symmetry transforms, encode each in a deterministic comparison form, select the **minimum** under a declared ordering (e.g., lexicographic on `d_bits`). This is analogous to canonical labeling in graph isomorphism. citeturn3search1turn3search0

#### 3.3.2 Canonical Hash Construction

Construct a canonical **transcript object** containing:

- schema version
- `family_id`
- symmetry policy and canonicalization algorithm IDs
- label policy IDs
- canonicalized configuration fields
- derived metrics (`S`, `Z_primary`, `orbit_size`, etc.)

Deterministically serialize (using JCS or deterministic CBOR) and hash (e.g., SHA‑256). If JSON is used, constrain to I‑JSON (RFC 7493) to avoid divergence. citeturn7search1

### 3.4 Deterministic Algorithms and Gate Checks

Pseudocode for canonicalization under symmetry (see Appendix for full listing). The following are **hard gates**:

- Canonicalization invariance failure.
- Descriptor mismatch.
- Policy mismatch (unknown `family_id`, etc.).
- Degeneracy ambiguity (received descriptor insufficient to uniquely identify a catalog entry).

### 3.5 Security Binding to OWAC‑MQP Authenticity

The canonical transcript becomes the payload for cryptographic authentication. Signatures or MACs are computed over:

```
AuthPayload = Encode(MID) ‖ H(CanonicalTranscript) ‖ Context
```

where `MID` contains `epoch`, `ctr`, `sender_key_id`, and optional `exp`. The receiver verifies the signature, checks replay, and only then releases the command to the application allow‑list.

### 3.6 Storage/Catalog Design

A catalog maps Duotronics descriptors to semantic symbols and allowed actions. See Part II for schema details.

## 4. Canonical Message Layer and Authenticity

This section describes how OWAC‑MQP ensures that **only explicitly authorized** actions execute, even if an attacker can inject audio. The exact message format and cryptographic profiles are frozen in Part II.

### 4.1 Deterministic Canonicalization as a Prerequisite

Non‑canonical parsing can undermine signatures. OWAC‑MQP uses **deterministic CBOR** (RFC 8949) as the normative on‑wire container, with indefinite‑length items forbidden and map keys sorted. citeturn1search0

### 4.2 Authenticity Mechanisms

OWAC‑MQP supports two families:

- **Post‑quantum signatures**: ML‑DSA (FIPS 204) for frequent commands, SLH‑DSA (FIPS 205) for critical operations. citeturn4search1turn5search0
- **One‑time MACs** (Wegman‑Carter) for information‑theoretic authentication when key material is pre‑shared and consumed. citeturn12search5

A hybrid approach (PQ‑signed epoch of one‑time keys) is also supported.

### 4.3 Replay Protection Without Acknowledgements

Every authenticated message includes:

- Monotonic counter `ctr` (64‑bit)
- Epoch identifier `epoch`
- Expiry time `exp` (Unix seconds)

Receiver enforces `ctr > last_ctr` per sender key, and `now ≤ exp`.

### 4.4 Ackless Reliability

Reliability is achieved by **redundancy**, not retransmission:

- Sender transmits each message N times with randomized spacing.
- Receiver executes only after at least K identical, independently verified instances of the same `MID` within a time window W (a “K‑of‑N quorum”). Execute‑once semantics prevent duplicate execution.

## 5. Application Layer Workflows

### 5.1 Application Safety Envelope

The IMN strictly separates “audio decode” from “system control.” Commands are allow‑listed, typed, and validated. Every accept/reject decision is logged.

### 5.2 Media Queue Actions

Examples:

- `QueueTrack`: `{track_id, priority_hint, earliest_play_time, latest_play_time}`
- `SuggestTrack`: `{track_id, weight_delta}`
- `RemoveFromQueue` (constrained)

### 5.3 Real‑Time Transport Controls

Narrow command set:

- `TransportSetState` (play/pause/stop)
- `TransportSeek` (bounded delta)
- `TransportSetRate` (bounded range)
- `TransportNext` / `Previous`

### 5.4 Keyboard‑Equivalent Input as Bounded Text Intent

Define `TextInput` with target (e.g., SEARCH_BOX) and bounded UTF‑8 text.

### 5.5 Song‑Driven Control Using Unmodified Full Songs

Songs alone are not authentication. The safe pattern:

1. Send authenticated control frame `QueueTrack(track_id=X, …)`.
2. Immediately play the full unmodified song audio.
3. Receiver executes only if both the authenticated frame verifies **and** the subsequent audio fingerprint matches track X within a confidence threshold, and link health is normal.

### 5.6 Optional Bulk Transfer

Bulk transfer (e.g., small images, config snapshots) is supported with chunk hashes, size limits, and receiver‑side quarantine.

## 6. Assurance Case and Test Strategy

“Aerospace grade” requires explicit requirements, traceability, verification coverage, and environmental qualification.

- **EMI robustness**: Define susceptibility tests (inspired by DO‑160) citeturn11search0; interference must cause rejection + telemetry.
- **Software assurance**: Requirements traceability, deterministic behavior, configuration management, and security process evidence (inspired by DO‑178C/DO‑326A). citeturn11search3turn11search4
- **Observability**: Structured audit stream of decode events, authenticity failures, policy denials, and accepted actions.
- **Key management**: Public keys pinned, key rotation gated; one‑time keys must be consumed and logged.

---

# Part II: v0.2 Implementable Specification Addendum

This part freezes all physical, link, framing, error detection, FEC, message schema, cryptographic profiles, replay rules, and test vector requirements. Two independent teams can implement this specification and interoperate without ambiguity.

## 7. Frozen Physical and Audio Interface Profile

### 7.1 Physical Interface Requirements

- **High‑assurance option**: Balanced line level (XLR) with isolation transformers at least on the receiver input. citeturn12search6turn12search0  
- **Commodity option**: Unbalanced line‑out/line‑in permitted for development/low‑assurance, but must still comply with monitoring rules.

### 7.2 Sampling, Levels, and File Format Constraints

- Capture rate: **48,000 samples/sec**, mono.
- Sample format: **16‑bit signed PCM**.
- Timing reference: sample index is authoritative.

### 7.3 Pilot Tone Link Monitoring (OWAC‑PILOT‑19K)

- A **19.000 kHz** sine pilot tone added at **‑30 dBFS** relative to the FSK payload (RMS).
- Receiver continuously estimates pilot amplitude and phase stability; abrupt loss or large excursions set `link_suspect=true` and are logged. citeturn11search2turn11search1

### 7.4 Ultrasonic Carrier as Optional Profile

Reserved versioned IDs; future ultrasonic profiles must ship with golden PCM packs.

## 8. Frozen Modem, Framing, CRC, and Optional FEC

### 8.1 Modulation Constants (OWAC‑PHY‑AUD‑A)

- Symbol rate: **1200 baud** citeturn16search39turn0search0
- Mark frequency (logic 1): **1200 Hz**
- Space frequency (logic 0): **2200 Hz**

### 8.2 Line Coding and Bit Order

- NRZI rule: data bit **0** causes tone transition; data bit **1** causes no transition. Initial tone state: Mark. citeturn16search3turn16search0
- Bit order: octets transmitted **least‑significant bit first** (after byte assembly, before stuffing).

### 8.3 Frame Delimiters and Transparency

- Flag delimiter: **0x7E**, bit sequence `01111110`. citeturn4view0turn7view0
- Bit stuffing: insert a `0` after five contiguous `1`s in payload; receiver discards such zeros.
- Preamble: at least **32 flag octets** before each burst. citeturn7view0turn4view0

### 8.4 CRC Polynomial and FCS Parameters (PPP FCS‑16)

- Generator polynomial: **x⁰ + x⁵ + x¹² + x¹⁶** (reversed **0x8408**). citeturn5view0turn6view0
- Initial value: **0xFFFF**.
- Good final check value: **0xF0B8**.
- FCS appended **least significant byte first**. citeturn6view0turn7view2

### 8.5 Optional FEC Profiles

#### 8.5.1 Convolutional Code (OWAC‑FEC‑CC‑1/2)

- Rate 1/2, constraint length K=7, connection vectors G1=171 octal, G2=133 octal (CCSDS). citeturn10view0

#### 8.5.2 Reed–Solomon Outer Code (OWAC‑FEC‑RS16I2)

- Symbol size 8 bits, codeword length 255, error correction capability E=16 → (255,223) code.
- Interleaving depth I=2. citeturn10view0

**Security note**: FEC success does **not** imply authenticity; signature/MAC verification is still required.

## 9. Frozen Deterministic CBOR Message Schema

### 9.1 Deterministic Encoding Rules (RFC 8949)

- Indefinite‑length items **MUST NOT** be used. citeturn1search0
- Map keys **MUST** be sorted in bytewise lexicographic order of their deterministic encodings. citeturn1search0

### 9.2 Message Envelope (CBOR Map)

| Key | Field    | Type         | Description                                |
|-----|----------|--------------|--------------------------------------------|
| 0   | `v`      | uint         | Protocol version (MUST be 1)               |
| 1   | `profile`| uint         | PHY/L2 profile ID (1 = OWAC‑PHY‑AUD‑A, 2 = with FEC‑RS16I2, etc.) |
| 2   | `mid`    | map          | Message identity and replay controls       |
| 3   | `cmd`    | map          | Typed command and parameters                |
| 4   | `auth`   | map          | Authenticity container (signature/MAC)     |
| 5   | `meta`   | map          | Optional bounded metadata                   |

Unknown top‑level keys → reject.

### 9.3 Replay and Identity Sub‑structures

`mid` map:
- `0`: `epoch` (uint, 64‑bit)
- `1`: `ctr` (uint, 64‑bit)
- `2`: `sid` (bstr, 16 bytes) – sender identity (e.g., truncated public‑key hash)
- `3`: `exp` (uint) – expiry time (Unix seconds, optional)

### 9.4 Command Structure

`cmd` map:
- `0`: `type` (uint) – command type enum
- `1`: `args` (map) – type‑specific parameters
- `2`: `duo` (bstr) – optional Duotronics symbol payload or codebook ID

Maximum sizes:
- Whole message ≤ 4096 bytes (before L2 framing)
- `cmd.args` ≤ 1024 bytes
- Any `tstr` ≤ 256 bytes UTF‑8
- Any `bstr` (except signature) ≤ 2048 bytes

### 9.5 Canonical Transcript Rule

The signed/MACed payload is:

```
TranscriptBytes = DeterministicCBOR( OWAC_Message without key 4 (auth) )
```

## 10. Frozen Authenticity Profiles with PQ Agility

### 10.1 ML‑DSA Command‑Signing Profile

- Default: **ML‑DSA‑65** (FIPS 204). citeturn23view0
- Public key: 1952 bytes
- Signature: **3309 bytes**

### 10.2 SLH‑DSA Critical‑Operation Profile

- Default: **SLH‑DSA‑SHA2‑128s** (FIPS 205). citeturn21view0
- Signature: **7856 bytes**

### 10.3 Auth Container Encoding

`auth` map:
- `0`: `alg` (uint) – 1 = ML‑DSA‑65, 2 = SLH‑DSA‑SHA2‑128s, 3 = OTMAC‑WegmanCarter‑v1 (reserved)
- `1`: `kid` (bstr, 16 bytes) – key identifier
- `2`: `sig` (bstr) – raw signature bytes (length MUST match expected)
- `3`: `ctx` (bstr, max 32 bytes) – domain separation context (default empty)

Reject if `kid` unknown or `sig` length mismatch.

## 11. Frozen Replay, Persistence, and Observability Rules

### 11.1 Replay Defense Model

Per `(sid, kid)`:

- Maintain `last_ctr` in durable storage.
- Accept only if `ctr > last_ctr`.
- After acceptance, atomically write `last_ctr := ctr`.

### 11.2 Execute‑Once Semantics and Duplication Handling

Define `MID = (epoch, sid, ctr)`.

- If `MID` equals most recent accepted MID and command digest matches → log duplicate, do not re‑execute.
- If `ctr ≤ last_ctr` but `MID` differs → reject as replay.

### 11.3 Expiry

Reject if `now > exp`.

### 11.4 Persistence and Rollback Resistance

- Store `last_ctr` in a rollback‑resistant manner (e.g., write‑ahead log with fsync, hardware monotonic counter).
- State resets require explicit `epoch` change.

### 11.5 Heartbeats

- `cmd.type = HEARTBEAT` sent periodically (e.g., every 5 seconds).
- Receiver logs missing heartbeats as link suspicion.

## 12. Golden Test Vector Pack Specification

### 12.1 Required Artifacts per Vector

Each golden vector directory contains:

- `audio.wav` – 48 kHz, 16‑bit PCM mono (with pilot, preamble, modulated burst)
- `demod_bits.bin` – raw demodulated bit decisions (pre‑NRZI decode)
- `l2_frames.jsonl` – extracted frames with flag indices, payload bytes (hex), FCS pass/fail
- `message.cbor` – CBOR bytes from frame payload
- `transcript.cbor` – deterministic CBOR of message‑without‑auth (the exact signed bytes)
- `transcript.sha256` – hash of transcript bytes
- `auth.expected.json` – expected verification result (verify=true/false, reject_reason)

### 12.2 Negative Test Cases

Must include:

- CBOR determinism violations (unsorted keys, indefinite‑length) → MUST reject.
- Replay cases (same message after later `ctr`) → MUST reject.
- Pilot tone removal → MUST set `link_suspect` and reject.
- Clipping injection → MUST log clipping and not accept.

### 12.3 Packaging Format

```
owac-vectors/
  MANIFEST.json (spec version, profile ID, file hashes)
  profiles/
    phy-aud-a/
      v0001_valid_heartbeat/
        (all files)
      v0002_crc_fail/
      v0003_cbor_unsorted_keys/
      ...
```

---

## Appendix: Deterministic Canonicalization Pseudocode

(Provided in the Duotronics section; available upon request.)

---

*This concludes the unified specification of OWAC‑MQP. Part I provided the architectural and design rationale; Part II freezes all parameters necessary for interoperable implementation. The protocol is now ready for aerospace‑grade development.*

## Appendix: Deterministic Canonicalization Pseudocode

The following pseudocode provides a reference implementation for the Duotronics canonicalization procedure described in §3. It is written for clarity and auditability; production implementations may optimize using linear‑time algorithms for circular string canonization (e.g., Booth’s algorithm, Duval’s Lyndon factorization) provided they produce identical results under the frozen ordering rules.

```pseudo
# Inputs:
#   c : {0,1}                     – center dot
#   d : array[0..n-1] of {0,1}    – vertex dots
#   a : attachment object          – optional; may be empty
#   n : integer                    – polygon size (≥3)
#   G_policy : string              – "C_n" or "D_n"
#   canon_order_policy : string    – e.g., "lexicographic_bits_then_attachments"
#   attachment_transform(policy)   – function to transform attachments under symmetry

function apply_rotation(d, t):
    # rotate vertex vector left by t positions
    new_d = array of size n
    for k in 0..n-1:
        new_d[k] = d[(k - t) mod n]
    return new_d

function apply_reflection(d):
    # reflect vertex vector: index mapping depends on declared convention
    # here we use k -> (-k) mod n (vertical axis through vertex 0)
    new_d = array of size n
    for k in 0..n-1:
        new_d[k] = d[(-k) mod n]
    return new_d

function render_for_compare(c, d, a):
    # produce a deterministic byte string or tuple for total ordering
    # must be consistent across implementations
    # example: concatenate:
    #   - c as a single byte
    #   - d bits packed into bytes (most significant bit first per byte, but any fixed rule works)
    #   - deterministic encoding of attachments (e.g., CBOR deterministic)
    #   then use lexicographic order on the resulting byte string
    return deterministic_bytes(c, d, a)

function canonicalize(c, d, a, n, G_policy):
    candidates = empty list

    # generate all rotations
    for t in 0..n-1:
        d_rot = apply_rotation(d, t)
        a_rot = transform_attachments(a, rotation=t)   # deterministic
        candidates.append( (c, d_rot, a_rot) )

    if G_policy == "D_n":
        # generate reflections and their rotations
        d_ref = apply_reflection(d)
        a_ref = transform_attachments(a, reflection=true)
        for t in 0..n-1:
            d_rot_ref = apply_rotation(d_ref, t)
            a_rot_ref = transform_attachments(a_ref, rotation=t)
            candidates.append( (c, d_rot_ref, a_rot_ref) )

    # compute canonical representative as the minimum under render_for_compare
    best = candidates[0]
    best_render = render_for_compare(best.c, best.d, best.a)
    for candidate in candidates[1:]:
        cr = render_for_compare(candidate.c, candidate.d, candidate.a)
        if cr < best_render:
            best = candidate
            best_render = cr

    # compute orbit size: number of distinct renders among candidates
    renders = set()
    for candidate in candidates:
        renders.add(render_for_compare(candidate.c, candidate.d, candidate.a))
    orbit_size = len(renders)

    return best.c, best.d, best.a, orbit_size
```

**Implementation notes:**

- The `transform_attachments` function must be defined for each attachment type in a versioned, deterministic manner. For baseline (no attachments), it is a no‑op.
- The `render_for_compare` function must produce a total order that is stable across platforms. Using a byte string produced by deterministic CBOR (RFC 8949) is recommended; the lexicographic order of the resulting CBOR bytes is well‑defined.
- For high‑performance environments, replace the enumeration with a linear‑time minimal rotation algorithm (e.g., Booth’s algorithm for circular strings) that returns the starting index of the lexicographically smallest rotation. The orbit size can still be computed by checking equality of transformed renders, but this optimization is not required for conformance.

---

## References and Further Reading

1. NIST FIPS 204: Module‑Lattice‑Based Digital Signature Standard (ML‑DSA).  
2. NIST FIPS 205: Stateless Hash‑Based Digital Signature Standard (SLH‑DSA).  
3. RFC 8949: Concise Binary Object Representation (CBOR).  
4. RFC 8785: JSON Canonicalization Scheme (JCS).  
5. RFC 7493: The I‑JSON Message Format.  
6. RTCA DO‑160: Environmental Conditions and Test Procedures for Airborne Equipment.  
7. RTCA DO‑178C: Software Considerations in Airborne Systems and Equipment Certification.  
8. RTCA DO‑326A: Airworthiness Security Process Specification.  
9. CCSDS 131.0‑B‑3: TM Synchronization and Channel Coding.  
10. Lamdan, Y., & Wolfson, H. J. (1988). Geometric hashing: A general and efficient model‑based recognition scheme.  
11. McKay, B. D., & Piperno, A. (2014). Practical graph isomorphism, II.  
12. Shiloach, Y. (1981). Fast canonization of circular strings.  
13. Duval, J. P. (1983). Factorizing words over an ordered alphabet.

---

*This completes the unified specification of OWAC‑MQP. All normative constants, message formats, and test requirements are now frozen. Implementers should refer to the golden test vector pack for conformance validation.*