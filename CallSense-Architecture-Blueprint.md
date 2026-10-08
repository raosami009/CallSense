# CallSense --- Architecture Blueprint

**Status:** Pre-implementation architecture\
**Project:** CallSense --- Real-Time Meeting Intelligence\
**Primary platform:** Windows\
**Targets:** Windows, macOS, Linux where practical\
**Stack:** Electron, React, Vite, JavaScript, Node.js, Tailwind CSS

## 1. Architecture Goals

CallSense is a privacy-first desktop meeting assistant that captures
meeting audio, transcribes speech, retains relevant context, detects
meaningful questions, and streams concise AI answers into a compact
floating window.

Core principles:

-   Local-first and usable without recurring cloud infrastructure.
-   Optional cloud STT/LLM for speed or quality.
-   Provider-agnostic services; no vendor lock-in.
-   React is UI only; Electron Main owns desktop/native capabilities.
-   OS-specific behavior is hidden behind platform abstractions.
-   Audio/transcript are not permanently stored by default.
-   Never send the entire meeting or every partial transcript update to
    an LLM.
-   Keep the Electron main process responsive.
-   Build and validate one phase at a time.

## 2. System Architecture

``` mermaid
flowchart TB
    User["User / Meeting"]

    subgraph App["CallSense Desktop"]
        Renderer["React Renderer<br/>UI + UI State"]
        Preload["Secure Preload<br/>Controlled API"]
        Main["Electron Main<br/>Orchestrator"]

        Window["Window Manager"]
        Shortcut["Shortcut Manager"]
        Platform["Platform Service"]
        Protection["Capture Protection"]
        Audio["Audio Service"]
        STT["Transcription Service"]
        Context["Context Engine"]
        Question["Question Detector"]
        AI["AI Service"]
        Events["Application Events"]
        Errors["Error Normalizer"]
    end

    subgraph Local["Local Runtime"]
        Whisper["Local STT<br/>whisper.cpp / compatible"]
        LocalLLM["Local LLM<br/>Ollama / llama.cpp"]
    end

    subgraph Cloud["Optional Cloud"]
        CloudSTT["Cloud STT"]
        CloudLLM["Cloud LLM"]
    end

    User --> Renderer
    Renderer <--> Preload
    Preload <--> Main

    Main --> Window
    Main --> Shortcut
    Main --> Platform
    Main --> Protection
    Main --> Audio
    Main --> STT
    Main --> Context
    Main --> Question
    Main --> AI

    Audio --> STT
    STT --> Context
    Context --> Question
    Question --> AI

    STT --> Whisper
    STT --> CloudSTT
    AI --> LocalLLM
    AI --> CloudLLM

    Events --> Renderer
    Errors --> Renderer
```

## 3. Process Boundaries

### Electron Main

Owns:

-   application lifecycle
-   windows and always-on-top behavior
-   global shortcuts
-   audio orchestration
-   platform integration
-   capture protection
-   service lifecycle
-   AI orchestration
-   IPC
-   structured logging
-   error boundaries

Main must not become one giant file. Keep services and platform code
separate.

### React Renderer

Owns:

-   assistant window UI
-   transcript display
-   question display
-   answer display
-   pipeline state
-   controls
-   settings UI
-   user-facing errors

Renderer must not:

-   use Node APIs directly
-   access filesystem directly
-   contain private API keys
-   implement Windows/macOS/Linux logic
-   call private cloud APIs directly

### Preload

Expose only a small, explicit API such as:

``` text
window.callSense
  window.show()
  window.hide()
  window.toggle()
  window.setAlwaysOnTop()

  audio.start()
  audio.stop()
  audio.pause()

  protection.enable()
  protection.disable()

  ai.generateAnswer()
  ai.cancel()

  events.onTranscript()
  events.onQuestion()
  events.onAnswerToken()
  events.onPipelineState()
```

Use `contextIsolation: true` and `nodeIntegration: false`.

## 4. Real-Time Pipeline

``` mermaid
flowchart LR
    Mic["Microphone"] --> Capture["Audio Capture"]
    Capture --> VAD["Voice Activity Detection"]
    VAD --> STT["Speech-to-Text"]
    STT --> Buffer["Transcript Buffer"]
    Buffer --> Context["Context Engine"]
    Context --> Detector["Question Detector"]
    Detector --> Package["Question + Relevant Context"]
    Package --> AI["AI Service"]
    AI --> Stream["Streaming Response"]
    Stream --> UI["Answer UI"]
```

### Pipeline rules

1.  Capture audio.
2.  Use VAD to ignore silence where practical.
3.  Produce partial and final transcript events.
4.  Maintain rolling context.
5.  Extract important facts/current topic.
6.  Detect meaningful questions.
7.  Prepare a small relevant context package.
8.  Ask the selected AI provider.
9.  Stream tokens to the UI.
10. Cancel stale requests if the question changes.

## 5. Audio Architecture

``` text
AudioService
├── Microphone
├── SystemAudio
└── Combined
```

Initial implementation should prioritize microphone input. System audio
is a separate milestone because capture APIs and permissions differ by
operating system.

Conceptual interface:

``` text
startListening()
stopListening()
pauseListening()
onTranscriptUpdate()
onListeningStarted()
onListeningStopped()
onError()
```

Platform-specific audio behavior must stay behind
`AudioService`/platform adapters.

## 6. Platform Architecture

``` text
PlatformService
├── WindowsPlatformService
├── MacOSPlatformService
└── LinuxPlatformService
```

Possible responsibilities:

-   OS-specific audio capture
-   permissions
-   global shortcuts
-   window behavior
-   capture protection
-   hardware information

Do not scatter `process.platform` checks throughout the application.
Centralize platform differences.

## 7. Capture Protection

``` mermaid
flowchart LR
    UI["UI Toggle"] --> IPC["Secure IPC"]
    IPC --> Service["CaptureProtectionService"]
    Service --> Platform["Platform Adapter"]
    Platform --> Windows["Windows"]
    Platform --> Mac["macOS"]
    Platform --> Linux["Linux / Best Effort"]
```

Use Electron content protection and platform-specific capabilities where
supported.

UI state should say something like:

``` text
Capture Protection: ON
```

Never claim that all possible screen-recording/capture mechanisms are
defeated.

## 8. Speech-to-Text

``` text
TranscriptionService
├── LocalSTTProvider
└── CloudSTTProvider
```

Conceptual operations:

``` text
start()
stop()
processAudio()
onPartialTranscript()
onFinalTranscript()
onError()
```

Initial local option: whisper.cpp or compatible local runtime.

The provider must be replaceable without changing the Context Engine or
UI.

## 9. Transcript Model

Distinguish:

-   **Partial transcript:** still changing.
-   **Final transcript:** stable recognized speech.
-   **Context transcript:** processed information retained for
    reasoning.

Do not trigger an AI request for every partial transcript update.

Use stability/debounce before question detection.

## 10. Context Engine

Context is a core CallSense feature.

Example:

``` text
PostgreSQL
Node.js
500k users
Read-heavy workload
Database bottleneck
```

Then:

``` text
How would you scale this?
```

The AI request should receive the relevant facts plus the question,
rather than only the question.

Maintain two conceptual levels:

### Immediate context

Recent conversation, approximately 30--120 seconds initially. Exact size
must be benchmark-driven.

### Important context

Longer-lived facts such as:

``` text
Technologies
Scale
Architecture
Constraints
Current problem
Decisions
```

Never retain an unlimited transcript in the AI prompt.

## 11. Question Detector

Use a lightweight deterministic layer first.

Signals:

-   question mark
-   interrogative words
-   sentence structure
-   direct requests
-   conversational context
-   transcript stability

Examples:

``` text
How would you scale this?
What database would you use?
Why did you choose PostgreSQL?
Can you explain the architecture?
```

Flow:

``` mermaid
flowchart TD
    Transcript["Transcript"] --> Stability["Stability / Debounce"]
    Stability --> Signals["Question Signals"]
    Context["Relevant Context"] --> Detector["Question Detector"]
    Signals --> Detector
    Detector --> Decision{"Meaningful Question?"}
    Decision -->|Yes| Request["Create AI Request"]
    Decision -->|No| Continue["Continue Listening"]
```

The detector should avoid unnecessary LLM classification calls.

## 12. AI Architecture

``` text
AIService
├── LocalAIProvider
└── CloudAIProvider
```

Conceptual operations:

``` text
generateAnswer()
summarizeContext()
explainQuestion()
cancelRequest()
```

Initial local runtime candidates:

-   Ollama
-   llama.cpp

Do not permanently hardcode either runtime into the application
architecture.

Optional cloud providers can be added later behind the same abstraction.

## 13. Local vs Cloud

``` mermaid
flowchart TD
    Request["AI Request"] --> Mode{"AI Mode"}
    Mode -->|Local| Local["Local Provider"]
    Mode -->|Cloud| Cloud["Cloud Provider"]
    Local --> LocalRuntime["Local LLM Runtime"]
    Cloud --> CloudAPI["Provider API"]
    LocalRuntime --> Stream["Streaming Tokens"]
    CloudAPI --> Stream
    Stream --> UI["Answer UI"]
```

Cloud mode is optional. There is no requirement for a permanent
CallSense backend.

API keys must never be hardcoded in React or exposed through frontend
`VITE_*` secrets.

## 14. Context Packaging

Before an AI request, create a compact package:

``` text
Relevant context:
- technology
- scale
- constraints
- current topic
- important decisions
- recent relevant statements

Question:
<detected question>

Answer requirements:
- concise
- direct
- technically useful
- use only relevant context
```

This reduces token use, latency, cost, and irrelevant model input.

## 15. Streaming and Cancellation

Preferred flow:

``` text
Question detected
    ↓
AI request starts
    ↓
First token
    ↓
Render immediately
    ↓
More tokens
    ↓
Complete
```

If the question changes:

``` text
Question A
  ↓
Request A
  ↓
Question changes
  ↓
Abort Request A
  ↓
Question B
  ↓
Request B
```

Use `AbortController` or equivalent provider cancellation.

## 16. State Architecture

Use centralized application state without adding Redux unless complexity
later justifies it.

Conceptual state:

``` text
AppState
├── session
│   ├── status
│   └── sessionId
├── audio
│   ├── source
│   ├── listening
│   └── error
├── transcription
│   ├── partial
│   └── final
├── processing
│   └── currentTopic
├── question
│   ├── text
│   ├── confidence
│   └── context
├── ai
│   ├── provider
│   ├── status
│   ├── answer
│   └── error
├── settings
└── privacy
```

## 17. Event Architecture

Important internal events:

``` text
audio.started
audio.stopped
audio.speech.started
audio.speech.stopped

transcript.partial
transcript.final
transcript.error

context.updated

question.detected
question.updated

ai.request.started
ai.token
ai.completed
ai.cancelled
ai.error

captureProtection.changed
```

Events must remain controlled and documented.

## 18. IPC Architecture

``` mermaid
flowchart LR
    Renderer["React Renderer"] --> Preload["Preload API"]
    Preload --> IPC["Controlled IPC"]
    IPC --> Main["Electron Main"]
    Main --> IPC
    IPC --> Preload
    Preload --> Renderer
```

Expose only capabilities required by the UI.

Never expose unrestricted `require`, `fs`, `child_process`, or other
Node APIs to React.

## 19. Window Architecture

The main assistant window should be:

-   frameless
-   compact
-   dark/minimal
-   always-on-top
-   draggable
-   resizable where practical
-   toggled by `Ctrl + Shift + M`

Example UI:

``` text
┌─────────────────────────────────┐
│ CallSense                  ×    │
│                                 │
│ ● Listening                     │
│                                 │
│ Question                        │
│ How would you scale this?       │
│                                 │
│ Answer                          │
│ Start with read replicas...     │
│                                 │
│ Capture Protection: ON          │
└─────────────────────────────────┘
```

## 20. Pipeline UI

States:

``` text
Idle
Listening
Transcribing
Processing
Generating
Complete
Error
```

Visual pipeline:

``` text
Listen → Transcribe → Understand → Answer
```

The UI must reflect real service state rather than fake progress.

## 21. Error Handling

Normalize service failures into structured application errors.

Examples:

``` text
MICROPHONE_UNAVAILABLE
MICROPHONE_PERMISSION_DENIED
STT_UNAVAILABLE
LOCAL_MODEL_UNAVAILABLE
NETWORK_UNAVAILABLE
INVALID_API_KEY
RATE_LIMITED
AI_TIMEOUT
AUDIO_DEVICE_DISCONNECTED
UNSUPPORTED_PLATFORM
```

A provider/service error must not crash Electron.

## 22. Logging

Use categories:

``` text
[App]
[Window]
[Audio]
[STT]
[Context]
[Question]
[AI]
[IPC]
[Platform]
[Performance]
[Error]
```

Never log:

-   raw audio
-   complete transcript
-   API keys
-   access tokens
-   sensitive meeting content

Performance logs should focus on timings.

## 23. Performance Architecture

Measure:

``` text
questionDetectionLatency
aiRequestStartLatency
timeToFirstToken
totalGenerationTime
endToEndLatency
```

Rules:

-   do not send the entire meeting to AI
-   do not call AI for every transcript character
-   use VAD
-   use debounce/stability
-   use rolling context
-   cancel stale requests
-   stream answers
-   batch UI token updates if necessary
-   keep Electron Main responsive
-   move expensive CPU work to a worker/utility process when justified

## 24. Privacy

Default behavior:

``` text
Audio
→ memory
→ processing
→ discard

Transcript
→ rolling context
→ processing
→ discard
```

Future settings:

``` text
Save transcript: OFF
Save audio: OFF
Local AI: ON
Cloud AI: optional
Clear current context
Clear current session
```

## 25. Security

Minimum Electron posture:

``` text
contextIsolation: true
nodeIntegration: false
```

Consider sandboxing where compatible.

Secrets must stay outside the React renderer.

Do not use frontend environment variables for private production
credentials.

## 26. Hardware Awareness

Eventually detect:

``` text
OS
CPU
RAM
GPU
GPU memory where available
```

Use this information to select model tiers:

``` text
Low hardware → smaller STT + smaller LLM
Medium       → medium STT + medium LLM
High         → larger STT + larger LLM
```

Do not lock final model sizes before benchmarking on the target machine.

## 27. Target Repository Structure

``` text
CallSense/
├── docs/
│   ├── architecture/
│   │   ├── architecture.md
│   │   ├── system-architecture.md
│   │   ├── realtime-pipeline.md
│   │   ├── security-architecture.md
│   │   └── diagrams/
│   └── decisions/
│       ├── ADR-001-electron.md
│       ├── ADR-002-local-first-ai.md
│       └── ADR-003-provider-abstractions.md
│
├── electron/
│   ├── main/
│   │   ├── index.js
│   │   ├── windowManager.js
│   │   ├── shortcuts.js
│   │   ├── ipc/
│   │   ├── platform/
│   │   │   ├── index.js
│   │   │   ├── windows/
│   │   │   ├── macos/
│   │   │   └── linux/
│   │   └── services/
│   │       ├── audio/
│   │       ├── transcription/
│   │       ├── context/
│   │       ├── question/
│   │       ├── ai/
│   │       ├── captureProtection/
│   │       └── logging/
│   └── preload/
│       └── preload.js
│
├── src/
│   ├── components/
│   │   ├── AssistantWindow/
│   │   ├── Transcript/
│   │   ├── QuestionPanel/
│   │   ├── AnswerPanel/
│   │   ├── PipelineStatus/
│   │   └── Controls/
│   ├── state/
│   │   └── appState.js
│   ├── hooks/
│   ├── utils/
│   ├── services/
│   │   └── ipc/
│   ├── App.jsx
│   └── main.jsx
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── package.json
├── vite.config.js
├── tailwind.config.js
├── .gitignore
└── README.md
```

This is a target structure. Do not create every directory before its
phase needs it.

## 28. Dependency Strategy

Core first:

``` text
Electron
React
Vite
Tailwind CSS
Node.js
```

Introduce later only when needed:

``` text
Vitest
E2E tooling
VAD runtime
whisper.cpp
Ollama or llama.cpp
packaging tools
```

Do not install native audio/local-AI dependencies during the
desktop-shell phase.

## 29. Testing Strategy

### Unit

Test:

-   question detection
-   context selection
-   debounce/stability
-   state transitions
-   cancellation
-   provider selection
-   error normalization

### Integration

Test:

``` text
Audio → STT
STT → Context
Context → Question
Question → AI
```

Use mocks where real providers are unnecessary.

### E2E

Eventually test:

-   app launch
-   floating window
-   shortcut
-   show/hide
-   pipeline state
-   answer rendering
-   error rendering

### Manual platform tests

Required for:

-   microphone permissions
-   system audio
-   global shortcuts
-   capture protection
-   always-on-top
-   packaging
-   OS-specific behavior

## 30. Development Phases

### Phase 0 --- Architecture

Deliver architecture documents, diagrams, ADRs, boundaries, decisions.
No production feature code.

### Phase 1 --- Desktop Shell

Electron + React + Vite + Tailwind; floating window; frameless;
always-on-top; drag; shortcut; capture-protection abstraction; pipeline
UI; mock question/answer. No real audio/STT/AI.

### Phase 2 --- Audio

AudioService, microphone capture, lifecycle, VAD boundary, platform
abstraction, errors.

### Phase 3 --- STT

TranscriptionService, local STT, partial/final transcript.

### Phase 4 --- Context

Rolling transcript, relevant context, important facts, current topic,
context limits.

### Phase 5 --- Question Detection

Deterministic detection, stability/debounce, confidence, question
events.

### Phase 6 --- Local AI

AIService, local runtime provider, prompt/context packaging, streaming,
cancellation.

### Phase 7 --- Cloud AI

Optional cloud STT/LLM, secure credentials, provider settings.

### Phase 8 --- Real-Time Integration

Connect audio → VAD → STT → context → question → AI → streaming UI.

### Phase 9 --- Production

Settings, privacy controls, logging, performance instrumentation,
hardware detection, model selection, packaging, installers,
cross-platform validation.

## 31. Phase Boundary Rule

Cline must stop after each phase.

Cline must not:

-   implement future phases early
-   install future dependencies without a reason
-   create fake infrastructure for future features
-   refactor unrelated areas
-   replace approved architecture without approval

After each phase:

``` text
STOP
REPORT
WAIT FOR APPROVAL
```

## 32. Initial ADRs

### ADR-001 --- Electron

Chosen for cross-platform desktop capabilities plus a React UI.

### ADR-002 --- Local-first AI

Chosen for privacy, offline capability, and zero recurring personal
infrastructure cost.

### ADR-003 --- Provider abstractions

Chosen to avoid vendor lock-in and allow local/cloud switching.

### ADR-004 --- Context Engine

Chosen because question-only prompting is insufficient for meeting-aware
answers.

### ADR-005 --- Main/Renderer separation

Chosen for Electron security and maintainability.

## 33. Explicit Non-Goals

Do not build initially:

``` text
SaaS backend
User accounts
Cloud database
Authentication server
Payment system
Web dashboard
Mobile app
Browser extension
Multi-agent system
Vector database
RAG infrastructure
Kubernetes
Docker infrastructure
Permanent transcript storage
Permanent audio storage
```

## 34. Architecture Acceptance Checklist

Before implementation, the architecture must answer:

-   Where does audio enter?
-   Where is VAD performed?
-   Where does STT run?
-   Where is transcript state held?
-   How is context retained?
-   How are important facts retained?
-   How is a question detected?
-   What context is sent to AI?
-   Where does local AI run?
-   Where does cloud AI run?
-   How does streaming reach the UI?
-   How are stale requests cancelled?
-   How does Renderer communicate with Main?
-   How are API keys protected?
-   How do OS implementations differ?
-   How is capture protection abstracted?
-   What happens when services fail?
-   How is sensitive data kept out of logs?
-   How is latency measured?
-   How is each layer tested?

If these answers are not clear, architecture is not finished.

## 35. Final Design Rule

> The UI must not know how audio, transcription, context processing,
> question detection, or AI providers work internally.

The UI requests capabilities.

Services perform the work.

The platform layer handles OS differences.

Provider adapters handle local/cloud implementations.

The Context Engine decides what information is relevant.

The AI Service answers using that prepared context.

This separation is the foundation for CallSense's speed, privacy,
testability, and extensibility.
