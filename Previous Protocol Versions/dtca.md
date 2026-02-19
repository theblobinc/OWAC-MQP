# One-Way Audio Control and Media Queue Protocol over an Air-Gapped Analog Audio Link

## Abstract and design intent

This paper specifies a unidirectional (“one-way only”) control and media-queue protocol that allows an air-gapped CNC/operator workstation to control an internet-facing web/media server using a direct **analog audio cable** (workstation line-out → server line-in). The primary requirement is **fail-closed tamper/interference detectability**: if the analog audio signal is interfered with, manipulated, injected, or ambiguously decoded, the receiver **rejects** the command and produces **observable evidence** (logs/alerts/metrics). The design explicitly does **not** require confidentiality; it prioritizes **authenticity, replay resistance, deterministic parsing, and explicit gates**.

The core architectural choice is to treat the audio cable as a **physical unidirectional “diode-like” path**, paired with a protocol stack that makes “success” rare unless every deterministic validation gate passes. (In security guidance, “diodes” are used to enforce one-way flows in unidirectional gateways, reducing the ability to use the same path for both intrusion and exfiltration.) citeturn7search3

The document is organized as a layered stack aligned with your OSI-like framing: physical → link/signal → canonical message/authenticity → application allow-list. It also integrates your “Duotronics / polygon dialect” concept as an **explicit symbol/representation layer** whose outputs are **canonicalized and signed** (so the receiver never executes “interpretations,” only verified canonical transcripts).

## System context and threat-driven requirements

### Nodes and trust boundaries

The system has two nodes:

The **Air-Gapped Control Node (ACN)** is the CNC/operator workstation. It has no network path to the internet-facing server and acts as a “fancy keyboard/controller” that can only transmit via audio line-out.

The **Internet-Facing Media Node (IMN)** is the web/media server. It hosts the public portal and live stream services. It receives audio from line-in and runs a strict decoder + verifier + allow-listed actuator.

The only intentional interconnect is a **one-way analog audio link** (ACN → IMN). There is no protocol-level acknowledgement or response channel.

### Threat model summary

The system must assume, at minimum:

An adversary (or environment) can cause **EMI/RF coupling** into the analog cable or the audio front-end, including amplitude distortion, clipping, and injected tones.

An adversary can attempt **audio injection** (e.g., near-field acoustic coupling, inductive coupling, or direct electrical coupling) to cause the receiver to decode actions.

An adversary can **replay** previously captured valid transmissions.

An adversary can **jam** the channel; the system is not required to remain available under jamming, but must reject ambiguous input and **make degradation observable**.

The system must treat **parser ambiguity** as a first-class attack surface. “Unknown” must not silently coerce into “valid.”

### Design invariants and acceptance policy

This paper defines three non-negotiable invariants:

Deterministic parse: A given on-wire payload must canonicalize to **exactly one** internal transcript, or it is rejected.

Cryptographic/authentic acceptance: No state-changing action may execute unless it is bound to a verified authenticity check (signature or information-theoretic MAC), and passes replay controls. (Digital signatures are explicitly described by NIST as mechanisms to detect unauthorized modifications and authenticate the signatory.) citeturn4search0

Reject-by-default under uncertainty: Any failure in framing, error detection, authenticity verification, canonicalization, parameter validation, or policy checks yields **rejection + logging**.

## Physical and link layer design over analog audio

### Physical layer profile

The analog audio layer is intentionally simple, but “aerospace-grade” thinking demands explicit constraints and testability.

The physical profile SHALL specify:

A fixed sampling rate (e.g., 48 kHz) and bit depth (e.g., 16-bit PCM) for receiver capture, with explicit handling for resampling if the hardware differs.

A fixed operating amplitude window with **headroom** to avoid clipping; clipping SHALL be logged as an interference indicator.

Short, shielded cabling; ferrites and isolation are recommended where practical to reduce susceptibility. (These are implementation measures, but the protocol’s assurance story depends on measuring when they fail.)

A receiver-side “link health monitor” that continuously records amplitude statistics, clipping events, and lock/unlock behavior (defined below).

### Link layer: robust framing that “rarely parses by accident”

A key reliability and safety goal is that random noise should almost never become a valid frame. Mature HDLC-style framing is a strong baseline because it combines:

A unique frame delimiter (“flag” pattern), bit stuffing to prevent accidental delimiter appearance, and a frame check sequence (FCS/CRC) to detect corruption.

AX.25, a well-known HDLC-derived link-layer protocol, documents these mechanisms precisely: a flag is `01111110` (0x7E), bit stuffing inserts a `0` after five consecutive `1` bits, and the receiver discards stuffed zeros; invalid frames include those not bounded by opening/closing flags or not octet-aligned. citeturn10view1turn10view0

The OWAC-MQP link layer SHOULD therefore use an **HDLC-like envelope** (conceptually similar to AX.25/PPP framing) with:

Preamble and acquisition sequence: A modem-specific training/preamble that lets the receiver lock symbol timing and estimate channel quality.

Frame delimiter: A strong sync marker (HDLC flag or equivalent), with stuffing/escaping rules so delimiters are not ambiguous. (PPP similarly describes inserting a `0` after five `1` bits for transparency.) citeturn13search0

Error detection: At minimum, CRC-16 or CRC-32 at the link layer, used only as a **corruption detector**, not as an authenticity mechanism.

A conservative modulation choice

For “aerospace-grade” assurance, simplicity and testability usually beat raw spectral efficiency. Audio-frequency shift keying (AFSK) is attractive because it is robust, easy to characterize, and historically proven over “voice-grade” links.

For example, packet-radio Bell-202 style AFSK is widely used at 1200 bit/s and uses two tones to represent bits (commonly 1200 Hz and 2200 Hz), explicitly documented in technical amateur radio literature. citeturn6search6

OWAC-MQP SHOULD define one or more modem profiles, for example:

OWAC-AFSK-1200: a conservative, low-rate profile intended for high robustness and easier certification test coverage.

OWAC-AFSK-2400/4800: optional higher-rate profiles with stricter receiver quality gating.

### Link health metrics as first-class security signals

Because the channel is analog and one-way, the receiver must treat measurements as evidence. The IMN SHALL track and expose at least:

Lock/unlock events: transitions into/out of “valid frame acquisition” state.

CRC failure rate: ratio of frames failing link CRC to total candidate frames.

Symbol error indicators: demodulator confidence metrics (implementation-defined) and abnormal jitter/drift.

Clipping/AGC anomalies: sustained saturation, near-silence, or suspiciously “flat” distributions.

These metrics serve two purposes: they help operators diagnose interference, and they provide an observable audit trail consistent with “tampering becomes invalid and visible.”

## Canonical message layer with tamper detection and replay resistance

This section defines how OWAC-MQP ensures that **only explicitly authorized** actions execute, even if an attacker can inject audio.

### Deterministic canonicalization as a prerequisite to security

A central lesson from real-world signed-protocol failures is that **non-canonical parsing** can undermine signatures: if two endpoints can serialize/parse “the same” data differently, signatures can verify unexpectedly or fail unpredictably.

Two strong, standards-based options exist:

Canonical JSON (JCS): RFC 8785 defines a JSON canonicalization scheme designed so hashing/signing operations produce repeatable results, using deterministic property sorting and constrained JSON forms. citeturn5search0

Deterministic CBOR: RFC 8949 defines deterministic encoding requirements (e.g., forbidding indefinite-length items and requiring sorted map keys) to make byte-level canonical forms stable. citeturn14search0

Recommendation

For “aerospace-grade” assurance, OWAC-MQP SHOULD use deterministic CBOR for on-wire payloads (smaller, binary, fewer Unicode pitfalls), while optionally allowing canonical JSON for human-visible logs/debug traces. If JSON is used, it SHOULD be constrained to interoperable profiles (e.g., I-JSON) to reduce ambiguous representations. citeturn14search2turn5search0

### Authenticity mechanisms

OWAC-MQP supports two authenticity families, matching your requirements.

Post-quantum standardized signatures

NIST finalized three post-quantum cryptography FIPS standards in August 2024, including FIPS 205 (SLH-DSA, derived from SPHINCS+), and described SLH-DSA as a backup signature method with different mathematical foundations than ML-DSA. citeturn4search1turn4search0

SLH-DSA (FIPS 205) is attractive for your philosophy because it is hash-based and designed to remain secure even against quantum adversaries, while explicitly serving the purpose of detecting unauthorized modifications and authenticating the signatory. citeturn4search0turn4search7

However, hash-based signatures generally have **larger signatures** and higher compute cost than lattice signatures. OWAC-MQP SHOULD therefore define explicit operational profiles:

OWAC-SIG-SLH: SLH-DSA for highest-assurance, low-frequency control messages and key-rotation events.

OWAC-SIG-ML: ML-DSA (FIPS 204) as an optional alternative when latency and transmission time dominate, with the trade clearly documented. citeturn4search1

Pre-shared, one-time authenticity material

If you want “tamper detection that does not rely on future computational hardness,” information-theoretic message authentication is the classical approach: Wegman and Carter introduced universal-hash-based authentication techniques in which an enemy “even one with infinite computer resources” cannot forge or modify messages without detection (assuming proper one-time key use). citeturn12search5

OWAC-MQP can support an OWAC-OTMAC mode:

Each message consumes a one-time MAC key (or one-time pad of MAC key material), making reuse explicitly forbidden.

Key exhaustion and re-provisioning become operational concerns and must be handled with explicit procedures.

A pragmatic hybrid that often fits “operator workflows”

Use a PQ signature (SLH-DSA) rarely to authenticate a new batch/epoch of one-time MAC material, then use one-time MAC tags for fast, frequent controls (play/pause/seek) with short on-wire overhead. This keeps “forgery = infeasible” even under quantum threat while reducing per-command audio airtime.

If you choose conventional symmetric MACs instead of one-time MACs, HMAC is a widely standardized mechanism: NIST FIPS 198 describes HMAC and notes its strength depends on the underlying hash function. citeturn13search1

### Replay protection without acknowledgements

Because OWAC-MQP is one-way, replay resistance must be receiver-enforced and must not depend on interactive freshness negotiation.

Every authenticated message SHALL include:

A monotonic counter (CTR): a 64-bit unsigned integer, strictly increasing per sender emission.

An epoch identifier (EPOCH): a receiver-stored identifier used to distinguish resets/rekeys.

An expiry time (EXP): a timestamp after which the receiver rejects the message.

Why include expiry?

Without expiry, an attacker who records a valid message that was never successfully received (e.g., jammed) could replay it later and potentially have it accepted—because the receiver’s last-seen counter did not advance. An expiry bound limits how long “deferred replay” remains viable, at the cost of requiring reasonable clock discipline at the air-gapped sender.

### Ackless reliability: repetition with deterministic execute-once semantics

Because there is no return channel, OWAC-MQP reliability should be achieved by **redundancy**, not retransmission protocols.

Define:

A message identifier MID = (EPOCH, CTR)

A receiver-side execute-once rule: if MID has already executed, duplicates are discarded (but logged as duplicates, not as attacks).

A sender rule: transmit each message N times with randomized inter-frame spacing (within a bounded window) to increase delivery probability under burst noise.

A receiver rule: execute only after at least K identical, independently verified instances of the same MID are received within a time window W (a “K-of-N quorum”). This reduces the probability that random decode errors produce a valid command, while still requiring authenticity verification.

### Duotronics / polygon dialect integration as a symbol layer

Your polygon/Duotronics concept fits best as a **representation layer** that feeds the canonical transcript and signature—not as a bespoke cryptographic primitive.

The integration pattern is:

Define a versioned Duotronics schema: symmetry group policy, canonicalization algorithm ID, offset policy ID, and the “full identity tuple” you described (not just a single label).

Canonicalize polygon tokens deterministically, producing a transcript object (e.g., a deterministic CBOR map) whose byte encoding is the object of cryptographic authentication.

Sign/MAC the canonical transcript bytes; the receiver re-derives the same canonical bytes and verifies authenticity.

This achieves the property you want: interference that changes even one semantic token changes the canonical transcript bytes, causing signature/MAC verification to fail, so the command is rejected.

The “aerospace-grade” discipline here is: the receiver never “interprets” multiple possible meanings. It accepts only if canonicalization + gates yield one admissible transcript and authenticity passes.

## Application layer workflows and strict allow-listed actions

### Application safety envelope

The IMN must strictly separate “audio decode” from “system control.” OWAC-MQP requires an allow-listed, typed command set whose semantics are:

Explicit (each message has one declared type)

Validated (type-specific parameter constraints)

Bounded (length limits, rate limits, and safe defaults)

Audited (every accept/reject decision logged with reason)

This aligns with your “unknown ≠ zero / fail-closed gates” philosophy.

### Media queue actions

OWAC-MQP SHALL support queue operations that reference media by stable identifier, not by arbitrary URLs or filenames. Example message types:

QueueTrack: `{track_id, priority_hint, earliest_play_time, latest_play_time}` with strict bounds.

SuggestTrack: `{track_id, weight_delta}` with per-track and per-sender rate limits.

RemoveFromQueue: optionally allowed, but generally riskier; if supported, constrain to removing only items queued by the same authenticated identity.

### Real-time transport controls

Transport controls are time-sensitive and should remain narrow:

TransportSetState: play/pause/stop

TransportSeek: delta in milliseconds with maximum absolute magnitude per command

TransportSetRate: bounded playback rate range

TransportNext/Previous: strict allow-list behavior

To avoid “command surface creep,” OWAC-MQP SHOULD forbid arbitrary “remote keypress injection” into the OS. “Keyboard-equivalent” should mean: bounded text and shortcut inputs into specific application targets.

### Keyboard-equivalent input as bounded “text intent,” not OS keystrokes

To provide broad operator control while remaining safe, define a TextInput message:

Target: an enumerated application field (e.g., SEARCH_BOX, TITLE_FIELD, CHAT_MOD_PANEL)

Text: UTF-8 with max length (e.g., 128 or 512), allowed character policy, normalization policy

Submit behavior: explicit (e.g., “set field value” vs “append” vs “submit/enter”)

This gives you “full keyboard layout” capability at the UX level without becoming an arbitrary remote control channel.

### Song-driven control using unmodified full songs

This is a compelling UX idea, but it is the most subtle security point in your proposal.

Acoustic fingerprinting is real and robust; the classic Shazam/ISMIR approach (“An Industrial Strength Audio Search Algorithm”) is designed for identifying audio from noisy excerpts. citeturn5search1  
Chromaprint (AcoustID) is an open-source fingerprinting component explicitly intended for audio identification and stream monitoring use cases. citeturn6search0

Security issue: songs alone are not authentication

If the command is “play song X” and the only evidence is that the receiver recognized song X, then any attacker capable of injecting that song (or a loud replay) can cause the effect—no cryptography required.

A safer architecture that preserves “unaltered songs”

Use the song as a second factor, not the sole authority:

Step one: Send a short authenticated control frame that says “QueueTrack(track_id=X) with MID=(EPOCH, CTR) and expiry EXP.”

Step two: Immediately play the full unmodified song audio.

Step three: The receiver only executes QueueTrack if:
it verifies the authenticated frame, and
it independently recognizes that the subsequent audio matches track X within a confidence threshold, and
it sees no link-health anomalies beyond policy.

This keeps your “unaltered full song” requirement intact (the song waveform is not modified), while ensuring that song recognition cannot be used as a forgery path.

### Optional cassette-style bulk transfer over audio

Bulk transfer is feasible but economically constrained. The link layer can carry arbitrary payload chunks (file blocks) with FEC + hashing, but the throughput of robust audio modems is generally in the kbps range under conservative profiles. As a result, OWAC-MQP should treat bulk transfer as:

A niche ingestion path for small artifacts (metadata bundles, small images, configuration snapshots)

Or a deliberately slow, high-assurance transfer path for cases where “air-gap breakaway” is more important than speed

If bulk transfer is enabled, it SHALL have:

Chunk hashes and a final manifest hash (content-addressed integrity)

Strict size limits per transfer session

Operator-visible progress at the receiver (since there is no sender-side acknowledgement)

A receiver-side quarantine: content is staged and not made public until manually reviewed or scanned (because the server is internet-exposed)

## Assurance case, test strategy, and “aerospace-grade” posture

“Aerospace grade” has less to do with fashionable primitives and more to do with **explicit requirements, traceability, verification coverage, and environmental qualification.**

### Environmental and EMI robustness

RTCA DO-160 provides standardized environmental and EMI test methods intended to ensure airborne equipment functions appropriately under aircraft environmental conditions; it is explicitly positioned as a reference standard and is updated through RTCA. citeturn11search0

While your system is not necessarily airborne avionics, borrowing DO-160-style thinking means:

Define susceptibility tests for conducted and radiated interference on the audio link.

Demonstrate that interference causes **rejection + telemetry**, not accidental execution.

Demonstrate safe behavior under clipping, saturation, dropouts, and burst noise.

### Software assurance and security process alignment

For airborne software assurance, DO-178C is the core RTCA document for defining design assurance and product assurance for airborne software (and is referenced by FAA advisory circulars as an acceptable means of compliance). citeturn11search3turn11search5  
For aviation cybersecurity process guidance, RTCA’s airworthiness security document set includes DO-326A (published in 2014) within its security portfolio. citeturn11search4

Translating these mindsets into OWAC-MQP yields concrete engineering actions:

Requirements traceability: Every command type, gate, and reject condition must be traceable to a requirement and a test.

Deterministic behavior: Canonicalization and parsing outcomes must be identical across builds, platforms, and versions (or explicitly versioned).

Configuration management: Keys, allow-lists, and policy thresholds are controlled configuration items, not ad hoc runtime state.

Security process evidence: Threat scenarios (injection, replay, jamming, parser ambiguity) map to mitigations and verification tests.

### Mandatory observability outputs

To satisfy your “tamper/interference must be observable” requirement, the IMN SHALL expose a structured audit stream that includes:

Decode events: lock/unlock, frame counts, CRC failures

Authenticity events: signature/MAC verify failures, replay rejections, expiry rejections

Policy events: parameter validation failures, rate-limit hits, allow-list denials

Action audit: accepted actions with MID, timestamp, authenticated identity, and parameters

These records should be emitted both to durable storage and to an operator-visible dashboard/alerting system.

### Key management and lifecycle gates

For PQ signatures (SLH-DSA / ML-DSA), the receiver must pin trusted public keys and treat key rotation as a gated event. NIST’s framing of PQ standards emphasizes they are designed to resist future quantum attacks, but operational security still depends on correct implementation and key management. citeturn4search1turn4search3

For one-time MAC material (information-theoretic mode), the system must treat key reuse as catastrophic and must provide:

A consumable key ledger (sender + receiver)

A re-provisioning procedure (offline, authenticated, auditable)

A hard fail-close when keys are exhausted

The “aerospace-grade” stance is: key lifecycle is explicit, tested, and logged—not handled informally.

### Key issues resolved and remaining open questions

Resolved in this paper:

Songs become safe controls only when paired with authenticated command frames; recognition alone is not authorization.

Ackless reliability is achieved with execute-once semantics plus repetition/quorum rules, not interactive retransmission.

Canonicalization is elevated to a prerequisite of authentication, using standards-based canonical JSON (JCS) or deterministic CBOR, both designed to make hashing/signing stable. citeturn5search0turn14search0

Remaining design decisions you must finalize to complete an implementable spec:

Modem profile selection: AFSK baseline is easiest to certify, but you must choose specific tone pairs, symbol rates, filtering, and acquisition thresholds, and then freeze them into a versioned PHY profile.

Signature profile sizing: SLH-DSA is attractive philosophically, but you must quantify acceptable command latency (signature transmission time) and decide where one-time MACs are required.

Clock discipline for expiry: Decide whether expiry is enforced (recommended) and how you manage clock drift on the air-gapped sender.

Human factors: Decide what the operator sees when commands are rejected (e.g., a local server dashboard), since there is no return channel to the CNC node.

This paper’s central claim is that an “air-gapped control over audio” protocol can be approached with aerospace-grade rigor if—and only if—the system treats **determinism, canonicalization, authenticity, replay resistance, and observability** as first-class, testable requirements rather than implementation details.