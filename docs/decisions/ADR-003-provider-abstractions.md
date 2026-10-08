# ADR-003 - Provider abstractions for STT and AI

**Status:** Accepted (Phase 0; pending user approval at phase gate)

## Context

Speech-to-text and LLM technology changes quickly, and users may prefer
local runtimes (privacy/offline) or cloud APIs (speed/quality). The
architecture must be provider-agnostic with no vendor lock-in, and neither
local runtimes nor cloud SDKs are installed during the architecture
phase.

## Decision

Both `TranscriptionService` and `AIService` are defined as **abstractions
with interchangeable providers**:

```text
TranscriptionService
|-- Local STT Provider      (whisper.cpp-compatible candidate)
`-- Cloud STT Provider      (optional)

AIService
|-- Local AI Provider       (Ollama / llama.cpp candidates)
`-- Cloud AI Provider       (optional)
```

Rules:

- No runtime (Ollama, llama.cpp, whisper.cpp) or cloud vendor is
  hardcoded into the architecture; they are candidates behind the
  interfaces.
- Switching providers must not require changes to the Context Engine,
  Question Detector, or UI.
- The app must run with local providers only (ADR-002).
- Streaming and cancellation semantics are normalized across providers.

## Consequences

- (+) No vendor lock-in; local/cloud can be switched at runtime/settings.
- (+) Services are testable with mocks before real providers exist.
- (+) Phased delivery: local first (Phases 3/6), cloud later (Phase 7).
- (-) An extra abstraction layer per service; adapters must normalize
  streaming, errors, and cancellation.
- (-) Lowest-common-denominator risk: provider-specific features may not
  fit the shared interface and must be rejected or feature-gated.
- (-) Each new provider costs an adapter implementation.