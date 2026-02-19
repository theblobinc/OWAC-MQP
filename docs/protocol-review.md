# OWAC-MQP Protocol Review: What It Does for Us

This note summarizes the current unified protocol spec in practical terms.

## Bottom line

OWAC-MQP gives us a one-way, air-gap-friendly command channel from a trusted control workstation to an internet-facing media node over plain analog audio. The protocol is designed so that commands execute only when they are authentic, fresh, and policy-compliant. If anything is ambiguous or malformed, the receiver rejects the message and records telemetry. 

## What value we get

- **Physical one-way control path**: We can issue commands without opening an inbound network path back into the air-gapped side.
- **Strong authenticity gating**: Every state-changing command is cryptographically verified before execution.
- **Replay resistance without ACKs**: The receiver enforces monotonic `(epoch, ctr, sid)` acceptance to block captured-message reuse.
- **Deterministic parsing**: Canonical CBOR and strict reject rules reduce parser confusion and implementation mismatch risk.
- **Fail-closed behavior**: Decode/auth/policy failures do not degrade into “best effort”; they reject.
- **Operational observability**: PHY and policy anomalies are exposed via logs/metrics so tamper/jamming attempts are detectable.

## How the protocol works end-to-end

1. ACN emits a framed, AFSK-modulated audio burst (with pilot tone) into the one-way cable.
2. IMN demodulates NRZI bits, un-stuffs HDLC-like frames, and verifies FCS.
3. IMN decodes deterministic CBOR and re-encodes to confirm canonical form.
4. IMN reconstructs `TranscriptBytes` and verifies signature/MAC against pinned keying policy.
5. IMN enforces replay/expiry checks and command allow-list policy.
6. IMN executes allowed command exactly once (per replay state policy), then persists replay state.

## Security posture and limits

### What it protects

- Unauthorized state changes (without valid authenticator)
- Replays of previously valid commands
- Parser ambiguity and malformed-message edge cases

### What it does **not** protect

- **Confidentiality**: audio is plaintext; observers can infer commands.
- **Availability under active jamming**: sustained interference can deny service.
- **Return-channel administration**: there is no bidirectional control loop in-band.

## Practical implications for deployment

- We must treat key provisioning and key rotation as critical operations.
- We should enforce strict command allow-lists and bounded argument schemas.
- We should monitor pilot/fault/reject metrics as security signals, not only reliability signals.
- We should keep profile selection static and explicit (out-of-band), with conformance tests based on golden vectors.

## Why this is a good fit for us

If our threat model prioritizes keeping the control environment isolated while still allowing outbound operational control to a higher-risk internet node, OWAC-MQP is purpose-built for that exact tradeoff: authenticity and replay safety over a physically constrained one-way medium.
