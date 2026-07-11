# Production Operations and Security

> **Repository baseline:** `tecxbro/vexa` → `main` at commit `90e5c726a75ea7ba349946648e6cf7f4e1cc845a`  
> **Source rule:** The audited findings override conflicting or inferred DeepWiki statements.  
> **Preservation note:** The original DeepWiki Q&A and audit files are retained in the external source pack and are intentionally not committed here.


## Authenticated browser identities

Authenticated mode restores a persistent Chromium profile into `/tmp/browser-data` and launches a persistent context.

Relevant profile material includes cookies, Local State, Preferences, Secure Preferences, Login Data, Network Persistent State, Web Data, and local storage.

Storage paths supported by the existing worker include:

- S3/MinIO-style profile synchronization.
- An HTTP cookie/profile service.
- The container filesystem for the active worker copy.

Stale Chromium singleton files are removed before launch.

### Required controls

- Encrypt profiles at rest.
- Use per-tenant/per-user object paths.
- Use short-lived credentials.
- Avoid logging cookies, profile archives, access keys, or signed URLs.
- Delete the worker copy after the meeting.
- Require exactly one valid profile backend in authenticated mode.
- Require `userId` when the HTTP backend needs it.

## Account and challenge handling

Google and Microsoft login/challenge flows are not solved through official meeting APIs. They are handled through restored browser state and, when blocked, an escalation path.

The worker can expose VNC and CDP so a human or trusted automation agent can clear captcha, consent, or login challenges. These are operational tools, not part of the normal product path.

## VNC and CDP

Current runtime ports:

| Port | Use |
|---|---|
| `5900` | x11vnc raw VNC. |
| `6080` | noVNC/websocket proxy. |
| `9222` | Chromium CDP target in anonymous mode when arguments are present. |
| `9223` | socat relay exposing CDP across the container network. |

Current concerns:

- x11vnc is started with `-nopw`.
- CDP accepts broad origins and grants near-total browser/profile control.
- Authenticated Chromium does not currently include CDP launch arguments, so the relay may have no target.

Production policy:

- Disable VNC/CDP by default.
- Enable only for a short-lived escalation session.
- Require gateway authorization and a per-session token.
- Apply Kubernetes NetworkPolicy/firewall rules so raw ports are not reachable.
- Tear down access immediately after resolution.

## Worker control channel

The worker uses Redis pub/sub for meeting commands. Existing command handling includes stop/leave and optional product capabilities such as reconfigure, speak, audio playback, chat, and screen content.

For LemonSlice, keep a minimal typed control contract:

- `stop` / `leave`.
- Optional provider reconnect.
- Optional diagnostics escalation.
- Optional health snapshot.

Remove Vexa-specific TTS/chat/screen commands only after confirming no cleanup or media dependency remains.

## Status callbacks

The worker sends lifecycle state changes to the meeting API/callback endpoint, including joining, awaiting admission, active, stopped/ended, and failure reasons.

Callbacks should include:

- Worker and meeting identifiers.
- Platform.
- State and reason code.
- Timestamp.
- Media/provider health summary.
- Sanitized diagnostics.

The silent admission check should run before ACTIVE is published.

## Timeouts and meeting-end behavior

Configured timeouts include:

- Waiting-room timeout.
- No-one-joined/startup-alone timeout.
- Everyone-left timeout.

Removal detection is platform-specific. Everybody-left and startup-alone behavior currently lives partly inside files named `recording.ts`. Keep those functions until monitoring is extracted.

## Browser failure and diagnostics

The worker includes:

- Playwright/browser crash handling.
- Structured JSON logs and context.
- In-memory log buffer for failure reporting.
- Resource sampling.
- Screenshots during navigation, lobby, failure, and humanized-click misses.
- VNC/CDP escalation.

The claim that there is no explicit screenshot mechanism is false; screenshots are used repeatedly.

## Cleanup sequence

Graceful cleanup must be idempotent and cover:

1. Stop accepting new commands.
2. Stop platform monitors.
3. Stop LemonSlice provider and local peer connections.
4. Stop external streamer if enabled.
5. Remove PulseAudio loopback modules and stop playback.
6. Leave the meeting when possible.
7. Close pages, context, and browser.
8. Stop Redis subscriber.
9. Flush final callback and logs.
10. Remove temporary profile/media files.
11. Exit so the orchestrator deletes the container.

Xvfb, PulseAudio, VNC, websockify, and socat are entrypoint child processes and terminate with the container. Explicit child cleanup is still preferable to avoid hangs during graceful shutdown.

## Docker/Kubernetes assumptions

- One isolated container per meeting.
- Headful Chromium under Xvfb.
- Software PulseAudio devices; no physical audio hardware is required.
- Xvfb is a virtual X server; host X11 access is not required.
- Network access to meeting platforms, Redis, callbacks, profile storage, and LemonSlice/Daily.
- A worker orchestrator creates and deletes containers.
- CPU/memory limits must account for multiple browser processes and, in compatibility mode, GStreamer/aiortc.

The Dockerfile has no explicit `USER` directive. Run as a dedicated non-root user and grant only the minimum filesystem/device permissions.

## Security threat model and corrections

### Chromium sandbox

`--no-sandbox` does not require elevated privileges. It disables a containment layer and increases the impact of a successful browser exploit. Prefer a container/runtime setup that permits Chromium sandboxing; otherwise compensate with non-root execution, seccomp/AppArmor, read-only filesystem, dropped capabilities, network policy, and one-meeting isolation.

Ordinary JavaScript cannot automatically escape to the OS. The risk is that a browser vulnerability has fewer barriers when the sandbox is disabled.

### Arbitrary bridge URL

`voiceAgentSettings.url` is currently a generic string. An arbitrary page can execute in the worker, render into the captured camera, produce audio, and access permissive local endpoints.

Require:

- HTTPS.
- An allowlisted LemonSlice origin.
- A signed, short-lived session URL or token.
- No user-controlled redirects to arbitrary origins.

### Meeting URL

Validate hostname and URL pattern against the selected platform, not only URL syntax.

### Local bridge API

The Python server currently listens on `0.0.0.0:8124`, allows broad CORS/private-network access, and has unauthenticated endpoints such as:

- `/offer`.
- `/offer_meeting_audio`.
- `/start_streaming`.
- `/keepalive`.
- `/shutdown`.

Bind to `127.0.0.1`, require a per-worker bearer token on every request, restrict CORS to the expected bridge origin, and validate offer/session identifiers.

### Secrets

`BOT_CONFIG` can contain Redis URLs, callback secrets, profile storage credentials, and provider tokens. Environment variables are convenient but observable through process/container tooling. Prefer mounted secret files or workload identity, short TTLs, redaction, and least privilege.

## Required for initial LemonSlice deployment

- Isolated worker container.
- Browser join/admission/leave adapters.
- Strict config.
- Redis stop channel or equivalent.
- Callbacks and structured logs.
- Meeting-end/removal monitoring.
- Provider and media health.
- Graceful cleanup.
- Authenticated profile support only if the initial product requires authenticated identities.

## Optional operational tooling

- VNC/CDP escalation.
- S3 profile backend when anonymous joins suffice.
- Browser-session mode.
- Built-in chat/screen/TTS commands.
- Recording and transcription.
- Native Zoom SDK.
