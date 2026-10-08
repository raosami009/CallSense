# ADR-005 - Strict separation of Electron Main, Preload, and React Renderer

**Status:** Accepted (Phase 0; pending user approval at phase gate)

## Context

Electron renders web content in the renderer while desktop powers
(windows, shortcuts, audio, OS integration) live in the main process.
Without an enforced boundary, renderers gain Node.js access, secrets leak
into frontend bundles, and platform logic scatters across UI code - a
common source of Electron vulnerabilities and unmaintainable codebases.

## Decision

Enforce a three-layer separation:

```text
React Renderer
    down
Secure Preload
    down
Controlled IPC
    down
Electron Main
    down
Services / Platform
```

- **React Renderer:** UI only. No Node.js, no filesystem, no private API
  keys, no OS APIs, no native audio APIs, no platform checks.
- **Secure Preload:** exposes only the small, documented
  `window.callSense` capability API; `contextIsolation: true`,
  `nodeIntegration: false`; never exposes `require`/`fs`/`child_process`.
- **Controlled IPC:** documented capability channels only; commands and
events are distinct; payloads and senders validated in main.
- **Electron Main:** owns application lifecycle, windows, shortcuts,
  audio orchestration, service lifecycle, AI orchestration, platform
  integration, capture protection, logging, and error boundaries.

## Consequences

- (+) Security: compromise of web content cannot reach Node.js or disk.
- (+) Secrets remain main-process only (see `security-architecture.md`).
- (+) Clear ownership: UI requests capabilities; services do the work;
  the platform layer absorbs OS differences - improving testability and
  maintainability.
- (-) IPC design and event marshalling add overhead and boilerplate.
- (-) More files/modules than a monolithic design; the IPC surface must
be kept deliberately small and documented to avoid drift.
- (-) Cross-boundary debugging is harder; structured logging categories
([IPC], [App], ...) are required.