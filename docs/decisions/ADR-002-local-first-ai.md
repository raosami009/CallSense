# ADR-002 - Local-first operation with optional cloud AI

**Status:** Accepted (Phase 0; pending user approval at phase gate)

## Context

CallSense processes meeting audio and transcripts - highly sensitive data.
Requirements: privacy-first, usable offline, no recurring personal
infrastructure cost, while optional cloud models may offer better speed or
quality for some users. A permanent CallSense backend is explicitly not a
project requirement.

## Decision

CallSense is **local-first**: by default audio and transcripts stay on the
device, and AI/STT run through local providers. **Cloud AI and cloud STT
are optional** modes behind the same provider abstractions (ADR-003),
enabled per user choice with credentials stored outside the renderer.
Default privacy settings: no permanent transcript storage, no permanent
audio storage, local AI on, cloud AI off.

## Consequences

- (+) Strong privacy story; data stays in memory -> processing -> discard.
- (+) Works offline; no recurring infrastructure cost; no vendor lock-in.
- (+) No backend, auth, or accounts needed (aligned with non-goals).
- (-) Local model quality depends on user hardware; hardware-aware model
tiering is deferred until benchmarking (blueprint section 26).
- (-) Two code paths (local and cloud) must be maintained behind one
abstraction, increasing Phase 6/7 scope.
- (-) Optional cloud mode adds credential handling duties (never in the
renderer, never in `VITE_*` variables).