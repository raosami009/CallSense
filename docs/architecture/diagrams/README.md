# CallSense - Architecture Diagrams

All diagrams are Mermaid. The canonical source of truth is
`../architecture.md`; this folder is an index of the diagrams used across
the architecture documentation.

| Diagram | Location |
|---|---|
| System architecture (full) | `../architecture.md` section 2, `../system-architecture.md` section 2 |
| Real-time pipeline | `../architecture.md` section 4, `../realtime-pipeline.md` section 1 |
| Question detector flow | `../architecture.md` section 11 |
| Capture protection flow | `../architecture.md` section 7, `../security-architecture.md` section 8 |
| Local vs cloud AI flow | `../architecture.md` section 13, `../system-architecture.md` section 10 |
| IPC round trip | `../architecture.md` section 18, `../system-architecture.md` section 6 |

## Core boundary diagram

```mermaid
flowchart TB
    Renderer["React Renderer"] --> Preload["Secure Preload"]
    Preload --> IPC["Controlled IPC"]
    IPC --> Main["Electron Main"]
    Main --> Services["Services / Platform"]
```

## Pipeline diagram

```mermaid
flowchart LR
    Audio --> VAD --> STT --> Transcript["Transcript"] --> Context["Context Engine"]
    Context --> Question["Question Detector"] --> Relevant["Relevant Context"]
    Relevant --> AI --> Stream["Streaming Answer"] --> UI
```

## Context flow (bounded by design)

```mermaid
flowchart TD
    Meeting["Meeting"] --> Rolling["Rolling Transcript"]
    Rolling --> CE["Context Engine"]
    CE --> RC["Relevant Context (bounded)"]
    RC --> Q["Question"]
    Q --> AI["AI"]
    Forbidden["Entire transcript to LLM - FORBIDDEN"] -.-> X["Blocked by design"]
```