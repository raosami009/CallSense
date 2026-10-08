# ADR-001 - Use Electron for the desktop application

**Status:** Accepted (Phase 0; pending user approval at phase gate)

## Context

CallSense is a Windows-first cross-platform desktop meeting assistant that
needs a floating always-on-top frameless window, global shortcuts, native
audio and OS integration, and a fast-moving React UI. The chosen stack is
Electron, React, Vite, JavaScript, Node.js, Tailwind CSS. Alternatives
considered: fully native toolkits (WPF/Qt/Swift), lighter webview shells
(e.g. Tauri).

## Decision

Use **Electron** as the desktop runtime. React (rendered by Vite) provides
the UI; Node.js capabilities stay in the Electron main process; Tailwind
CSS styles the UI. Electron provides cross-platform windowing, shortcuts,
and content-protection APIs while allowing one codebase for Windows,
macOS, and Linux.

## Consequences

- (+) Mature cross-platform desktop APIs; single web-based UI codebase;
  rapid iteration; large ecosystem.
- (+) Clear security model available: `contextIsolation: true`,
  `nodeIntegration: false` (see ADR-005).
- (-) Larger binary and higher memory usage than native or webview-based
  shells.
- (-) Security posture is opt-in and must be enforced (preload/IPC
discipline; see `security-architecture.md`).
- (-) Heavy CPU work (future STT/LLM/VAD) must not block the main
  process; it will later move to worker/utility processes.