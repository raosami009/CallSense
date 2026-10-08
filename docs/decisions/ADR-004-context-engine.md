# ADR-004 - The Context Engine is a first-class component

**Status:** Accepted (Phase 0; pending user approval at phase gate)

## Context

Question-only prompting is insufficient for meeting-aware answers: a
question like "How would you scale this?" is meaningless without the
surrounding facts (e.g. PostgreSQL, 500k users, read-heavy workload).
Conversely, sending the entire meeting transcript to an LLM is slow,
costly, leaky, and violates the performance budget. Retained context must
therefore be curated, bounded, and explicit.

## Decision

Introduce a dedicated **`ContextEngine`** service in Electron Main that
owns context retention and packaging. Its model is:

```text
Recent Context + Important Facts + Current Topic + Current Question
```

- **Recent Context:** rolling window of recent conversation
  (~30-120 seconds initially; benchmark-driven).
- **Important Facts:** longer-lived extracted facts (technologies, scale,
  architecture, constraints, decisions).
- **Current Topic:** what the conversation is about now.
- **Current Question:** the detected question handed to the AI request.

It feeds the Question Detector and produces the compact
"relevant context" package for every AI request. Hard rules:
`Entire meeting transcript -> LLM` and `Every transcript character -> LLM`
are architecturally forbidden.

## Consequences

- (+) Substantially better answer quality with bounded token counts,
  latency, and cost.
- (+) Context retention is a single testable unit with explicit limits.
- (+) Privacy: no unbounded transcript ever leaves the machine toward a
  cloud model.
- (-) New tuning surface: window sizes, fact extraction, topic tracking
  must be benchmark-driven rather than fixed up front.
- (-) Context curation becomes a dependency of question detection and AI
generation, so its interfaces must stay stable.