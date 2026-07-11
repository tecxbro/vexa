# Claim Correction Register

> **Repository baseline:** `tecxbro/vexa` → `main` at commit `90e5c726a75ea7ba349946648e6cf7f4e1cc845a`  
> **Source rule:** The audited findings override conflicting or inferred DeepWiki statements.  
> **Preservation note:** The original DeepWiki Q&A and audit files are retained in the external source pack and are intentionally not committed here.


## Question verdicts

| Question | Verdict | Knowledge document |
|---|---|---|
| Q1 — Product boundary | Mostly correct | `01_product_boundary_and_scope.md` |
| Q2 — Google Meet execution path | Mostly correct but incomplete/outdated | `02_google_meet_worker_execution_path.md` |
| Q3 — Execution path | Exact duplicate of Q2 | Merged into document 02; preserved separately in the external source pack |
| Q4 — Shared architecture | Mostly correct | `03_shared_platform_architecture.md` |
| Q5 — Incoming meeting audio | Substantially incorrect | `04_incoming_meeting_audio.md` |
| Q6 — Outgoing agent audio | Partially correct | `05_outgoing_agent_audio.md` |
| Q7 — Agent video | Partially correct; current path not production-ready | `06_agent_video_and_camera_injection.md` |
| Q8 — Operations/security | Mixed; several serious errors | `07_production_operations_and_security.md` |
| Q9 — Repository classification | Useful direction; deletion advice required correction | `08_repository_classification.md` |
| Q10 — Reduced structure | Good direction; media architecture redesigned | `09_target_repository_structure.md` |

## Source precedence

1. Audited main-branch findings at the stated commit.
2. Direct code excerpts preserved in the original Q&A when they do not conflict with the audit.
3. DeepWiki interpretations and inferences.
4. DeepWiki research monologue.

When two claims conflict, use the higher source.

## Corrected or superseded claims

### Incoming audio is not PulseAudio-based

Superseded: the external page does not intercept `getUserMedia`, or meeting audio reaches it through PulseAudio.

Correct: `webpage_streamer_payload.js` intercepts `getUserMedia`. The meeting browser sends a combined track to Python over `/offer`; Python rebroadcasts it to the page over `/offer_meeting_audio`.

### Voice-agent microphone is not always muted

Superseded: external-page audio cannot reach the meeting because the meeting UI mic stays muted.

Correct: Meet leaves the mic enabled when `voiceAgentEnabled` is true. The real defect is that the schema permits a bridge URL without enabling voice-agent mode.

### Fake media permission does not bypass JS patches

Superseded: `--use-fake-ui-for-media-stream` causes Chromium to ignore the virtual-camera `getUserMedia` override.

Correct: the flag auto-accepts permission prompts. Track failures come from sender creation, renegotiation, UI camera state, and streamer health.

### Current external video is screen capture

Clarified: the current bridge captures the entire second X display at 1920×1080 I420 and 15 FPS. It does not directly forward the avatar track.

### `runBot()` was directly available

Superseded: its sequence had to be inferred because the file was truncated.

Correct: the audit read the full function and confirmed browser/profile/control/service/platform sequencing.

### Authenticated CDP is not fully wired

Superseded: CDP relay works uniformly across browser modes.

Correct: authenticated launch arguments omit CDP port arguments.

### Configuration validation is weaker than runtime requirements

Confirmed defects:

- Optional `meeting_id` in schema but mandatory at runtime.
- No platform-host validation.
- No bridge origin/HTTPS allowlist.
- No invariant tying bridge URL to voice-agent mode.
- Weak authenticated profile backend invariants.

### Shared lifecycle is not completely neutral

Confirmed: Teams captions are started from shared `meetingFlow.ts`; ACTIVE is sent before silent verification.

### Recording files cannot be deleted wholesale

Superseded: disable recording and remove platform recording modules.

Correct: those modules also own participant observation, startup-alone/everyone-left timeouts, and active-lifetime promises. Split monitoring from capture first.

## Security claim corrections

| Incorrect/overstated claim | Correct interpretation |
|---|---|
| `--no-sandbox` requires elevated privileges | It disables Chromium sandboxing; it does not require root/SYS_ADMIN. |
| PulseAudio requires hardware devices | This worker uses software null sinks, monitors, and remapped sources. |
| Xvfb requires host X11 | Xvfb is the virtual X server. |
| Any malicious JavaScript can escape to the OS | JavaScript needs a browser exploit; disabling the sandbox reduces containment. |
| No screenshot mechanism exists | The join/admission code takes multiple explicit Playwright screenshots. |

## Risks elevated by the audit

- Python bridge on `0.0.0.0:8124`, unauthenticated endpoints, permissive CORS.
- x11vnc `-nopw` on `5900` and noVNC on `6080`.
- Raw CDP relay on `9223` with broad origin acceptance.
- Arbitrary meeting and external-agent URLs.
- Browser profile and provider credentials.
- No explicit non-root `USER` in the Dockerfile.
- Chromium sandbox disabled.

## Preservation note

No original claim was deleted from the external source pack. The raw archive is intentionally not committed to this repository. This register identifies which earlier DeepWiki statements must not be treated as current repository truth.
