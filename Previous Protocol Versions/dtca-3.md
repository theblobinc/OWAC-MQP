# OWAC‑MQP v0.2 Implementable Specification Addendum  
Freezing PHY constants, framing/FEC, deterministic CBOR schema, replay rules, and a golden test‑vector pipeline

## Scope and design objective

This addendum is the “adulting step” that turns OWAC‑MQP from a strong concept into a spec two independent teams can implement and still interoperate—without tribal knowledge. The theme is **freeze every ambiguity point**: modulation constants, framing bytes/bits, CRC parameters, deterministic serialization rules, replay windows, state persistence, and a conformance test pack that begins with **raw PCM audio** and ends with **cryptographic verify results**.

Two guiding constraints remain unchanged:

Determinism in representation is a prerequisite for security. If two implementations can canonicalize or serialize the “same” logical message differently, cryptographic signing/hashing becomes unreliable; this is exactly why canonicalization exists for signed data formats (e.g., canonical JSON) and why deterministic encoding rules exist for CBOR. citeturn2search3turn1search0

The link is allowed to fail (availability is not guaranteed under jamming), but state‑changing actions must be **fail‑closed** and **observable**. Digital signatures are explicitly described by NIST as a mechanism to detect unauthorized modifications and authenticate signers, and OWAC‑MQP leans on that property as the acceptance gate above the noisy analog channel. citeturn2search0turn22view0turn19view0

## Frozen physical and audio interface profile

### Physical interface requirements

Balanced interconnects and transformer isolation are promoted from “nice hardening ideas” to a **normative high‑assurance option** for OWAC installations that must resist conducted interference and ground‑referenced injection.

Balanced interfaces reject **common‑mode noise** because differential receivers (active or transformer) respond to the difference between hot/cold, largely rejecting noise that appears identically on both conductors; this is a primary operational advantage cited in pro‑audio engineering guidance. citeturn12search6

Transformer isolation provides **galvanic isolation** (magnetic coupling across an electrically insulated barrier) and is commonly used to eliminate ground‑loop coupling paths; the same galvanic break also raises the bar for direct conducted injection via shared grounds. citeturn12search3turn12search0

Normative text (OWAC‑PHY‑AUD‑A):
- The recommended high‑assurance physical interface is **balanced line level** (XLR or equivalent) with **isolation transformers** at least on the receiver input; transformers at both ends are permitted and preferred when feasible for threat models including conducted injection. citeturn12search6turn12search0  
- Commodity unbalanced line‑out/line‑in is permitted only for development and low‑assurance deployments; production profiles MUST still comply with the monitoring and fail‑closed rules below.

### Sampling, levels, and file format constraints

OWAC‑MQP’s golden vectors begin with PCM audio, so we freeze capture/playback conventions:

- Capture rate: **48,000 samples/sec**, mono.  
- Sample format: **16‑bit signed PCM** for both capture and test vectors.  
- Timing reference: sample index is the authoritative clock for the test harness (no wall‑clock assumptions).  

48 kHz is not “security magic,” but freezing it eliminates a major interoperability fault line: resampling behavior differs across stacks, and test vectors must lock the full pipeline from PCM → bits.

### Pilot tone link monitoring

A pilot tone is a **continuous tamper visibility channel** that does not depend on successful frame decoding. OWAC adopts the concept of a stable pilot as used in FM stereo baseband: a 19 kHz pilot is a known reference signal whose presence and stability are meaningful and measurable. citeturn11search2turn11search1

Normative text (OWAC‑PILOT‑19K):
- A **19.000 kHz** sine pilot tone SHALL be added at **‑30 dBFS** relative to the FSK payload (RMS) in profile OWAC‑PHY‑AUD‑A.  
- The receiver SHALL continuously estimate pilot amplitude and phase stability; abrupt loss, saturation, or large excursions SHALL set `link_suspect=true` and MUST be recorded in the audit log alongside any frame accept/reject decisions.  
- The pilot is conceptually similar to FM’s 19 kHz pilot used as a reference; OWAC uses it purely for **integrity monitoring**, not multiplex decoding. citeturn11search2turn11search1

Rationale: pilot tracking gives you a fast “this cable is being messed with” alarm even when the attacker’s activity prevents clean frames from forming.

### Ultrasonic carrier as an optional profile

Ultrasonic operation (>20 kHz) remains a valid hardening direction, but it moves you away from ubiquitous commodity audio hardware. This addendum therefore **does not freeze** an ultrasonic modem profile yet; it reserves versioned IDs for it and requires that any future ultrasonic profile ship with its own golden PCM pack (because ultrasonic performance is hardware‑specific).

## Frozen modem, framing, CRC, and optional FEC

### Modulation constants

To freeze tone pairs and symbol rate, OWAC‑PHY‑AUD‑A adopts the widely documented Bell‑202 family parameters:

- Symbol rate: **1200 baud** citeturn16search39turn0search0  
- Mark frequency: **1200 Hz** (logic 1) citeturn16search39turn0search0  
- Space frequency: **2200 Hz** (logic 0) citeturn16search39turn0search0  

This mapping (mark=1, space=0) is consistent with common Bell‑202 detector specifications. citeturn16search39turn0search0

### Line coding and bit ordering

To reduce “clock drift by long unchanging runs” and align with the long history of AFSK packet practice, OWAC‑PHY‑AUD‑A uses **NRZI over HDLC‑stuffed bits**:

- NRZI rule: a data bit **0** causes a **tone state transition**; a data bit **1** causes **no transition**. citeturn16search3turn16search0  
- Initial tone state at start of frame: **Mark (1200 Hz)**.

This NRZI rule is widely taught in packet‑radio AFSK contexts and is stable enough to freeze as a normative interoperability point. citeturn16search3turn16search0

Bit order:
- OWAC transmits octets **least‑significant bit first** on the physical bitstream (after byte assembly, before stuffing). This reduces implementation ambiguity by matching common audio‑modem practice and avoids “which endianness did you mean?” bugs.

### Frame delimiters and transparency rules

OWAC uses an HDLC‑like delimiter and transparency rule set. The core constants are frozen directly from the HDLC‑style framing described in PPP’s HDLC‑like framing RFC:

- Frame flag delimiter: **0x7E**, bit sequence **01111110**. citeturn4view0turn7view0  
- Bit stuffing: insert a **0 bit after any sequence of five contiguous 1 bits** in the frame payload (including FCS bits); the receiver discards any 0 bit that directly follows five contiguous 1 bits prior to FCS computation. citeturn7view0  
- Empty frames (back‑to‑back flags) are discarded and not counted as FCS errors (helps avoid false alarms from fill). citeturn4view0  

Preamble / acquisition:
- Before each transmission burst, the sender SHALL transmit **at least 32 flag octets (0x7E)** as time‑fill/preamble to allow the receiver to lock timing and detect the delimiter cleanly. PPP explicitly recommends transmitting flags as inter‑frame fill and describes the flag as the synchronization mechanism. citeturn7view0turn4view0

### CRC polynomial and exact FCS parameters

To eliminate the most common “we both used CRC‑16, why doesn’t it match?” failure mode, OWAC freezes the CRC to PPP’s FCS‑16 parameters exactly:

- Generator polynomial: **x⁰ + x⁵ + x¹² + x¹⁶** (often represented reversed as **0x8408**). citeturn5view0turn6view0  
- Initial value: **0xFFFF** (`PPPINITFCS16`). citeturn6view0  
- Good final check value: **0xF0B8** (`PPPGOODFCS16`). citeturn6view0  
- Output procedure: compute FCS over the frame bytes (not including flags or transparency bits), ones‑complement it, append **least significant byte first**. citeturn6view0turn7view2  

This makes the receiver’s “CRC pass/fail” a deterministic gate that is identical across implementations, and it leverages a widely deployed reference algorithm with tables and code published in the RFC itself. citeturn6view0turn5view0

### Optional FEC and interleaving profiles

OWAC’s baseline philosophy is “security by strong reject gates,” and CRC already serves as the corruption detector gate. FEC is about *availability and operational robustness*, not authenticity. So OWAC treats FEC as a **versioned profile**: you either do it exactly as specified or you don’t negotiate it implicitly.

#### Convolutional code option

For environments where the audio path is noisy but mostly Gaussian‑like, OWAC defines an optional inner convolutional code based on a thoroughly standardized, widely used reference:

- Rate: **1/2**  
- Constraint length: **K=7**  
- Connection vectors: **G1=1111001 (171 octal)**, **G2=1011011 (133 octal)** citeturn10view0  

These parameters are specified in the CCSDS TM Synchronization and Channel Coding standard’s convolutional section and are extremely well‑characterized in practice. citeturn10view0

#### Reed–Solomon outer code option

For bursty disturbance (clicks, dropouts) and to reduce reliance on pure repetition, OWAC defines an optional outer Reed–Solomon code profile aligned with CCSDS’s R‑S parameters:

- Symbol size: **J=8 bits per symbol** citeturn10view0  
- Codeword length: **n = 2^J − 1 = 255 symbols** citeturn10view0  
- Error correction capability: **E=16** (corrects 16 symbol errors per codeword) citeturn10view0  
- Parity symbols: **2E** parity symbols per codeword; information symbols **k = n − 2E** citeturn10view0  
- This parameterization corresponds to a **(255,223)** code when E=16. citeturn10view0  
- Interleaving depth allowed for E=16: **I ∈ {1,2,3,4,5,8}** citeturn10view0  

OWAC‑FEC‑RS16I2 (recommended if you enable RS):
- Use E=16 and interleaving depth **I=2**.

Why this exact choice: CCSDS explicitly defines these parameters (including interleaving options) and positions the R‑S code as a **powerful burst error correcting code**. citeturn10view0

Important security note (normative): FEC decode success does **not** imply authenticity. OWAC MUST still require signature/MAC verification over the canonical transcript before execution, because error‑correcting codes can produce valid‑looking outputs from heavily corrupted inputs (a known class of risk in any ECC system). This is the reason the crypto gate remains the execute gate.

## Frozen deterministic CBOR message schema

### Deterministic encoding rules

OWAC‑MQP uses CBOR as the normative on‑wire message container but **requires deterministic encoding** to eliminate “same data model, different bytes” failures.

Core deterministic encoding requirements (normative):
- Indefinite‑length items MUST NOT be used. citeturn1search0  
- Map keys MUST be sorted in bytewise lexicographic order of their deterministic encodings. citeturn1search0  

These requirements are explicitly enumerated in RFC 8949’s deterministic encoding section. citeturn1search0

Schema notation:
- The OWAC specification SHALL publish a normative CDDL file for the message schema, because CDDL was created specifically to express CBOR/JSON structures unambiguously. citeturn1search1  

### Message envelope

OWAC defines a single top‑level CBOR map, keyed by **small unsigned integers** (0–23) so key encodings are one byte and deterministic ordering is straightforward.

Top-level message `OWAC_Message` (CBOR map):
- `0` → `v` (uint) protocol version, MUST be `1` for OWAC‑MQP v1 messages.
- `1` → `profile` (uint) identifies the PHY/L2 profile (e.g., `1 = OWAC‑PHY‑AUD‑A`, `2 = OWAC‑PHY‑AUD‑A‑FEC‑RS16I2`, reserved values for ultrasonic).
- `2` → `mid` (map) message identity and replay controls.
- `3` → `cmd` (map) typed command and parameters.
- `4` → `auth` (map) authenticity container (signature or one‑time MAC).
- `5` → `meta` (map) optional bounded metadata (never required to execute; may be logged).

Strictness:
- Unknown top‑level keys MUST cause reject (fail‑closed).
- Unknown `cmd.type` MUST cause reject.
- Out‑of‑range lengths MUST cause reject.

### Replay and identity sub-structures

`mid` is a CBOR map:
- `0` → `epoch` (uint, 64‑bit) policy/key epoch ID.
- `1` → `ctr` (uint, 64‑bit) monotonically increasing message counter for the sender key.
- `2` → `sid` (bstr, 16 bytes) sender identity (e.g., truncated public‑key hash or pinned ID).
- `3` → `exp` (uint) expiry time in seconds since Unix epoch (optional but strongly recommended for “deferred replay” reduction).

### Command structure and max lengths

`cmd` is a CBOR map:
- `0` → `type` (uint) command type enum.
- `1` → `args` (map) type‑specific params.
- `2` → `duo` (bstr) optional Duotronics symbol payload or codebook ID (bounded).

Maximum sizes (frozen defaults):
- Entire CBOR message (before L2 framing): **≤ 4096 bytes** in OWAC‑PHY‑AUD‑A. Larger payloads MUST be chunked at the application layer.  
- `cmd.args`: **≤ 1024 bytes** encoded.
- Any `tstr`: **≤ 256 bytes** UTF‑8.
- Any `bstr` (except signature): **≤ 2048 bytes**.
- Signature size must match the selected parameter set (see next section).

Rationale: size caps prevent parser‑resource DoS and make the test‑vector space tractable.

### Canonical transcript rule

OWAC signs or MACs a **canonical transcript bytestring** defined as:

`TranscriptBytes = DeterministicCBOR( OWAC_Message without key 4 (auth) )`

- That is: remove the `auth` container, deterministically encode the remaining map, hash/sign that byte sequence.

This rule ensures every field that could affect semantics (including `profile`, `mid`, and `cmd`) is cryptographically bound, and it leverages deterministic CBOR encoding requirements so both endpoints hash the same bytes. citeturn1search0

If JSON is used for human‑readable diagnostics, it MUST NOT be used as the signed form unless canonicalized with JCS, because cryptographic operations require invariant expression; JCS exists to produce such “hashable” JSON. citeturn2search3

## Frozen authenticity profiles with PQ agility

### PQ signature profiles and sizes

OWAC defines two PQ signature families as first‑class options and freezes parameter sets by referencing NIST’s finalized standards.

NIST finalized FIPS 203/204/205 as the first PQ standards set, including ML‑DSA (FIPS 204) and SLH‑DSA (FIPS 205). citeturn2search0turn2search2

#### ML‑DSA command-signing profile

Default command signing uses **ML‑DSA‑65**, unless the deployment requires a different security category.

FIPS 204 Table 2 provides the exact artifact sizes:
- ML‑DSA‑65 public key: 1952 bytes  
- ML‑DSA‑65 private key: 4032 bytes  
- ML‑DSA‑65 signature: **3309 bytes** citeturn23view0turn23view2  

This is the “practical” signing mode for frequent commands because it is significantly smaller than SLH‑DSA signatures, reducing on‑wire time.

#### SLH‑DSA critical-operation profile

For rare but extremely critical controls (epoch changes, one‑time key batch signing, root policy updates), OWAC permits SLH‑DSA because it is hash‑based and serves as a “backup method” with different math foundations than ML‑DSA. citeturn2search2turn19view0

FIPS 205 Table 2 shows that signature sizes vary widely by parameter set; for example:
- SLH‑DSA‑SHA2‑128s signature: **7856 bytes**
- SLH‑DSA‑SHA2‑128f signature: **17088 bytes**
- SLH‑DSA‑SHA2‑256f signature: **49856 bytes** citeturn21view0  

OWAC freezes one default SLH set for “critical ops”:
- **SLH‑DSA‑SHA2‑128s** (small variant) unless the deployment’s assurance case requires higher categories.

### Auth container encoding

`auth` is a CBOR map:
- `0` → `alg` (uint) algorithm enum:
  - `1 = ML‑DSA‑65`
  - `2 = SLH‑DSA‑SHA2‑128s`
  - `3 = OTMAC‑WegmanCarter‑v1` (reserved if you implement one‑time MAC pools)
- `1` → `kid` (bstr, 16 bytes) key identifier (receiver‑pinned).
- `2` → `sig` (bstr) raw signature bytes (length MUST match expectations for `alg`).
- `3` → `ctx` (bstr, max 32 bytes) domain separation/context string, default empty.

Verification rules:
- Reject if `kid` unknown.
- Reject if `sig` length does not match frozen size for that `alg` (hard gate; avoids parser confusion).

## Frozen replay, persistence, and observability rules

### Replay defense model

OWAC is one‑way. That means the receiver must enforce freshness without interactive challenges. The simplest correct rule is: accept only counters that are strictly increasing per sender key, and persist the last accepted counter.

This is conceptually aligned with widely deployed anti‑replay services such as IPsec ESP, which requires the receiver to verify that sequence numbers do not duplicate those received during the lifetime of the security association (SA), and recommends doing this check early to reject duplicates quickly. citeturn13search1

OWAC’s acceptance rule (per `sid`+`kid`):
- Maintain `last_ctr` (uint64) in durable storage.
- Accept a message only if `ctr > last_ctr`.  
- If accepted and executed, atomically write `last_ctr := ctr` before releasing “success” to any downstream actuator.

This “strictly increasing” rule is simpler than a sliding window and is appropriate because OWAC also uses execute‑once semantics and repetition of identical MIDs for reliability.

### Execute-once semantics and duplication handling

Define `MID := (epoch, sid, ctr)`.

Receiver behavior:
- If `MID` equals the most recently accepted MID and the command digest matches, treat as a **duplicate delivery** (likely repetition for reliability). Do not re‑execute; log `duplicate=true`.
- If `ctr <= last_ctr` but `MID` differs from most recent accepted MID, treat as **replay**; reject and log with reason code.

### Expiry

OWAC recommends using `mid.exp` to reduce “stored replay later” risk:
- Reject if `now > exp`.
- Expiry is not an authentication mechanism, but it limits the operational window of a captured transmission.

### Persistence and rollback resistance

If an attacker can roll back the receiver’s `last_ctr` state, they can replay previously valid messages. Therefore:

- The receiver MUST store `last_ctr` in a rollback‑resistant way: at minimum a write‑ahead log with fsync on commit; ideally hardware‑backed monotonic counters or secured storage (TPM/HSM/NVRAM depending on platform).  
- State resets MUST require a separate, explicitly logged administrative procedure that changes `epoch` and pins a new trust state.

### Heartbeats

To detect silent cable cut/jam conditions without waiting for a human to issue a command, OWAC defines an authenticated heartbeat type:

- `cmd.type = HEARTBEAT`
- Sent periodically (e.g., every 5 seconds), containing current `(epoch, ctr)` and optionally a coarse timestamp.
- Receiver logs missing heartbeats as `link_down` suspicion if pilot tone is also degraded.

This turns availability degradation into an **observable** security‑relevant event, even in idle periods.

## Golden test vector pack specification

### Why OWAC’s conformance suite must start at raw audio

Most “protocol test vectors” start at bytes. OWAC’s attack surface starts earlier: decoding ambiguity, AGC/clipping behavior, timing lock, and pilot stability. Therefore conformance must start at **PCM samples**.

The intent is similar to how standards bodies distribute example values for cryptographic algorithms to support conformance validation; NIST itself points implementers to example values resources for digital signature algorithms. citeturn19view0turn22view0

### Required artifact chain

Each golden vector MUST include:

Raw audio capture: `*.wav` (48kHz, 16‑bit PCM, mono), containing:
- pilot tone (if profile includes it),
- preamble flags,
- modulated frame burst(s),
- optional interference injections for negative tests.

Decoder expected outputs (all frozen formats):
- `demod_bits.bin`: raw demodulated bit decisions (post‑symbol timing, pre‑NRZI decode).
- `l2_frames.jsonl`: extracted frames with:
  - flag alignment indices,
  - stuffed/unstuffed bit counts,
  - payload bytes (hex),
  - FCS pass/fail (must match frozen PPP FCS parameters). citeturn6view0turn7view0

Message layer outputs:
- `message.cbor`: the CBOR bytes extracted from the frame payload.
- `message.model.json`: a canonical JSON diagnostic rendering (not signed form).
- `transcript.cbor`: deterministic CBOR of message-without-auth (the exact signed bytes). citeturn1search0

Crypto outputs:
- `transcript.sha256`: hash of transcript bytes (or the exact digest function you freeze).
- `auth.expected`: one of:
  - `verify=true` for valid signatures,
  - `verify=false` for invalid signatures,
  - `reject_reason` enumerations (e.g., `SIG_LEN_MISMATCH`, `CTR_REPLAY`, `CBOR_NONDETERMINISTIC`, `CRC_FAIL`).

### Negative tests OWAC MUST ship

To prove fail‑closed behavior and eliminate “lenient decoder” divergence, the vector pack MUST include adversarial cases:

CBOR determinism violations:
- Same data model encoded with map keys unsorted → MUST reject, because deterministic encoding requires sorted keys. citeturn1search0  
- Indefinite-length strings/maps → MUST reject. citeturn1search0

Replay cases:
- Same valid message replayed after a later `ctr` accepted → MUST reject (replay).
- Duplicate of most recent MID → MUST not re‑execute, MUST log duplicate.

Analog-layer failures:
- Pilot tone removed mid‑burst → MUST set `link_suspect` and MUST not accept any message unless all other gates pass (this is observability; acceptance still depends on crypto).
- Clipping injected → MUST log clipping statistics and CRC/signature failures as they occur.

### Packaging format

OWAC freezes the test vector pack layout:

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
```

`MANIFEST.json` MUST include:
- spec version,
- exact profile ID,
- exact decoder options required (e.g., symbol timing window length),
- hash of every file for integrity.

### A note about “filters” in the test harness

Because DSP filters can be implemented many ways, OWAC does **not** standardize the internal filter topology as long as the demodulator produces the same bitstream for the same PCM input under the same profile. The golden PCM test vectors are the enforcement mechanism: if a particular filter choice changes the bits, it fails conformance.

That said, vendors often specify noise models for FSK detection in the voice band (e.g., 300–3400 Hz band‑limited noise tolerance in Bell‑202 detector contexts), which can inform engineering margins but should not be treated as a protocol constant. citeturn16search39

---

If you want, I can turn this addendum into a fully formatted “Blue Book style” spec draft (with MUST/SHOULD language consolidated into a normative requirements section and a separate rationale appendix), but the technical freezing points above are the core moves that eliminate cross‑implementation drift while staying faithful to **fail‑closed, observable, deterministic**.