# CallSense - Security Architecture

**Status:** Phase 0 - intended architecture only.
**Source of truth:** `docs/architecture/architecture.md`

This document covers Electron security posture, the preload boundary, IPC
discipline, API key protection, renderer restrictions, logging
restrictions, transcript/audio privacy, and capture-protection
limitations.

## 1. Electron security posture

Minimum required posture for the future application (Phase 1 onward):

```text
contextIsolation: true
nodeIntegration: false
```

- Consider enabling the Electron sandbox where compatible.
- Never use the deprecated `remote` module.
- These settings will be verified as an explicit acceptance item when the
desktop shell is created; they are not implemented in this phase.

## 2. Preload boundary

The preload script is the **only** bridge between the renderer and the
main process. It exposes a small, explicit, capability-oriented API
(`window.callSense`):

```text
window.callSense
  window.show() / window.hide() / window.toggle() / window.setAlwaysOnTop()
  audio.start() / audio.stop() / audio.pause()
  protection.enable() / protection.disable()
  ai.generateAnswer() / ai.cancel()
  events.onTranscript() / events.onQuestion()
  events.onAnswerToken() / events.onPipelineState()
```

Hard rules:

- Never expose unrestricted `require`, `fs`, `child_process`, `net`, or
  other Node APIs through preload.
- No generic "invoke any channel / run any code" passthrough.
- The preload API surface is documented and stays minimal; new methods
  require an architecture-level reason.

## 3. IPC

```text
React Renderer -> Secure Preload -> Controlled IPC -> Electron Main
Electron Main  -> Controlled IPC -> Secure Preload -> React Renderer
```

- Only documented channels exist; channels are capability-oriented
  (e.g. `audio:start`, `ai:generateAnswer`), never arbitrary.
- Commands (renderer to main) are distinct from events (main to
  renderer).
- Main validates message payloads and the sending frame before acting.
- All desktop capability requests flow through IPC; nothing bypasses it.
- Service/provider failures are normalized into structured errors
  (e.g. `MICROPHONE_UNAVAILABLE`, `STT_UNAVAILABLE`, `AI_TIMEOUT`,
  `INVALID_API_KEY`) and never crash Electron.
- Sensitive payloads (raw audio, full transcripts) are not logged at the
  IPC layer.

## 4. API key protection

- Private API keys live only in the Electron main process (or later,
  OS-protected storage such as the system keychain / encrypted-at-rest
  settings - decision deferred to the cloud phase).
- Keys are **never** embedded in React frontend code.
- Keys are **never** exposed to the renderer through the preload API.
- Frontend `VITE_*` environment variables must **never** hold private
  production secrets - Vite bakes these into the client bundle, which is
  trivially readable.
- Keys are never written to logs, error messages, or crash reports.
- Cloud AI/STT is optional; the app must run with zero cloud credentials.

## 5. Renderer restrictions

The React renderer must not directly:

```text
use Node.js APIs
access the filesystem
hold private API keys
access OS APIs
access native audio APIs
implement Windows / macOS / Linux logic
call private cloud APIs directly
```

The UI requests capabilities only; services in Electron Main perform the
work.

## 6. Logging restrictions

Logging uses categories (`[App] [Window] [Audio] [STT] [Context]
[Question] [AI] [IPC] [Platform] [Performance] [Error]`).

Never log:

- raw audio
- complete transcripts
- API keys / access tokens
- sensitive meeting content

Performance logs focus on timings only (the metrics in
`realtime-pipeline.md`). Enforcement belongs to `LoggingService`, which
must sanitize or redact before writing anywhere.

## 7. Transcript and audio privacy

Default data lifecycle:

```text
Audio      -> memory -> processing -> discard
Transcript -> rolling context -> processing -> discard
```

- No permanent audio or transcript storage is introduced during the
  initial implementation phases.
- No database, cloud sync, or telemetry containing meeting content is in
  scope (see blueprint non-goals).
- Planned future privacy settings (Phase 9, defaults shown):

```text
Save transcript: OFF
Save audio: OFF
Local AI: ON
Cloud AI: optional
Clear current context
Clear current session
```

- Sensitive transcript/audio data must never be written to logs (see
  section 6).

## 8. Capture protection - limitations

`CaptureProtectionService` applies Electron content protection and
platform-specific capabilities where supported:

```text
UI toggle -> Secure IPC -> CaptureProtectionService -> Platform adapter
             (Windows first, macOS second, Linux best effort)
```

Honesty requirements:

- UI may state `Capture Protection: ON` only reflecting the actual state.
- **Never** claim that all possible screen-recording / capture mechanisms
  are defeated.
- Linux support is best effort; behavior varies by compositor.
- Protection reduces risk; it is not a guarantee.

## 9. Explicitly out of scope for this phase

No authentication, backend server, database, user accounts, payments, or
secrets-management infrastructure is built now. This document defines
requirements those future phases must honor.