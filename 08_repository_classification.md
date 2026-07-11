# Conservative Repository Classification

> **Repository baseline:** `tecxbro/vexa` → `main` at commit `90e5c726a75ea7ba349946648e6cf7f4e1cc845a`  
> **Source rule:** The audited findings override conflicting or inferred DeepWiki statements.  
> **Preservation note:** The original DeepWiki Q&A and audit files are retained in the external source pack and are intentionally not committed here.


This classification is deliberately conservative. A component is not safe to delete merely because its filename says transcription or recording; hidden lifecycle and cleanup behavior must first be separated and covered by end-to-end tests.

## Summary categories

| Category | Meaning |
|---|---|
| Keep unchanged initially | Needed for a first working fork; postpone cleanup. |
| Keep but rename | Correct responsibility, Vexa-specific name. |
| Keep but refactor | Required, but mixed with unrelated or unsafe behavior. |
| Split | One module owns multiple product responsibilities. |
| Replace | LemonSlice needs a different implementation or control-plane contract. |
| Disable first | Turn off by configuration; observe before deletion. |
| Safe to delete after tests | No retained runtime responsibility after full coverage. |
| Unknown | Requires direct code/runtime verification. |

## Worker entrypoints

| Component | Classification | Why / runtime use | After direct Daily tracks |
|---|---|---|---|
| `entrypoint.sh` | Keep but refactor | Starts Xvfb, PulseAudio, VNC/CDP, browser utility checks, and Node worker. Removing it breaks the runtime environment. | Keep a trimmed version; remove live-agent PulseAudio and native Zoom setup when unused. |
| `src/docker.ts` | Keep but rename/refactor | Parses and validates `BOT_CONFIG`, reports startup failure, calls `runBot`. | Keep as `main.ts`; enforce strict provider/platform invariants. |
| Browser-session mode | Disable first | Separate remote-browser product; not needed for normal meetings. | Delete after escalation/VNC requirements are covered elsewhere. |
| `runBot()` in `index.ts` | Split | Creates control, browser, media, services, diagnostics, and dispatch. | Keep behavior, split into worker bootstrap modules. |

## Shared meeting lifecycle

| Component | Classification | Rationale |
|---|---|---|
| `platforms/shared/meetingFlow.ts` | Keep but refactor | Core join/admission/removal lifecycle. Remove Teams imports, verify before ACTIVE, replace `startRecording` with monitoring/capture separation. |
| `platforms/shared/escalation.ts` | Keep initially | Handles recoverable captcha/login states. Harden access paths. |
| Stop-signal and graceful-leave logic | Keep but refactor | Essential for container cleanup and deterministic callbacks. Remove transcript/recording upload branches only after extraction. |

## Browser creation and injection

| Component | Classification | Rationale |
|---|---|---|
| Browser launch args | Keep but refactor | Media permissions and stability are required, but sandbox/CDP policy and authenticated arguments need correction. |
| Stealth plugin | Keep initially | Helps browser meeting compatibility. Reassess policy and necessity later. |
| `utils/injection.ts` | Keep initially | Reliable script injection under CSP/Trusted Types. |
| `utils/browser.ts` | Split | Contains media discovery, recording/transcription browser code, and unrelated utilities. Keep media discovery fallback; remove capture pipeline later. |

## Platform adapters

| Component | Classification | Rationale |
|---|---|---|
| `platforms/googlemeet/` | Keep initially, then refactor | Join, admission, humanized input, removal, and meeting monitoring are core. |
| `platforms/msteams/` | Keep initially, then refactor | Core Teams behavior. Move caption startup into adapter and deduplicate removal/monitoring. |
| `platforms/zoom/web/` | Keep initially, then refactor | Browser Zoom support is a product requirement. |
| Native Zoom SDK | Disable first; delete after tests if unused | Separate C++/N-API path and operational burden. Do not remove until Zoom Web meets all requirements. |

## Incoming meeting audio

| Component | Classification | Rationale | After direct Daily tracks |
|---|---|---|---|
| HTML media-element discovery/mixer | Keep as fallback, refactor | Current source for combined meeting audio. | Prefer receiver tracks; retain compatibility fallback. |
| `/offer` and `/offer_meeting_audio` bridge | Keep but harden during transition | Currently delivers meeting audio to the external page. | Replace with secure provider bridge/direct provider integration. |
| Per-speaker transcription capture | Disable first | Not required by LemonSlice; may carry speaker/meeting monitoring signals. | Delete after monitoring dependencies are removed and tests pass. |

## Outgoing agent audio

| Component | Classification | Rationale | After direct Daily tracks |
|---|---|---|---|
| PulseAudio `tts_sink`/`virtual_mic` chain | Keep initially | Current meeting microphone transport. | Remove from live-agent path after sender replacement works; optionally retain for legacy TTS. |
| `VoiceAgentPageService.startAudioBridge()` | Keep but refactor | Creates streamer-monitor to meeting-mic loopback. | Replace with direct audio track publication. |
| `MicrophoneService` | Keep but split | UI mic state is necessary; selectors should be adapter-owned. | Keep platform sender/UI preparation; no OS loop required. |
| Vexa TTS service | Disable first, delete after tests | LemonSlice owns speech generation. | Not required. |
| TTS playback module | Split | Device/mute helpers may be reused while TTS generation/playback is not. | Keep only generic audio-device fallback if needed. |

## Outgoing agent video

| Component | Classification | Rationale | After direct Daily tracks |
|---|---|---|---|
| Virtual camera init script | Keep but refactor | Creates placeholder and sender hooks. | Keep sender hooks/placeholder; remove Vexa naming and screen-capture assumptions. |
| Meeting video injector | Keep but refactor | Sender discovery, replacement, retry, and stats are core. | Becomes the direct track publisher. |
| External Xvfb/GStreamer video streamer | Keep behind compatibility flag | Current prototype works but is not production target. | Delete after direct avatar track is stable. |
| Screen-share service | Disable first | Not required for agent camera. | Delete after ensuring no sender/renegotiation dependency. |

## External webpage bridge

| Component | Classification | Rationale |
|---|---|---|
| `voice-agent-page.ts` | Keep but refactor | Starts external provider surface and owns bridge lifecycle, but contains platform/media coupling. |
| Python streamer | Replace for production; retain as compatibility | Required by current screen-capture bridge. Harden immediately if retained. |
| Payload `getUserMedia` interception | Keep during transition | Current meeting-audio input seam for the page. | Replace or simplify when Daily provider receives tracks directly. |
| Arbitrary URL loading | Replace | Accept only typed LemonSlice provider sessions and allowlisted origins. |

## Authentication

| Component | Classification | Rationale |
|---|---|---|
| `s3-sync.ts` | Keep if authenticated identities are required; refactor security | Profile storage option. |
| HTTP cookie/profile client | Keep if selected; refactor validation | Alternative backend. |
| `browser-profile.ts` | Keep initially | Essential files and stale-lock cleanup. |

## Control, callbacks, diagnostics

| Component | Classification | Rationale |
|---|---|---|
| Redis subscriber/command handler | Keep but refactor | Stop/control is required; reduce to typed LemonSlice commands. |
| Unified status callback | Keep but rename/refactor | Lifecycle integration is required; adapt payload to LemonSlice controller. |
| Structured logging/resource sampler/screenshots | Keep | Operationally essential. |
| VNC/CDP | Disable by default, keep as optional | Powerful escalation tooling with high security risk. |
| Chat service | Disable first | Not required unless LemonSlice product uses meeting chat. |

## Container and deployment

| Component | Classification | Rationale |
|---|---|---|
| Bot Dockerfile | Keep but refactor | Required browser/media runtime. Add non-root user, drop unused packages, harden sandbox/network. |
| Helm/Kubernetes manifests | Keep but refactor | Useful container-per-meeting deployment patterns; remove Vexa services and apply network policy/secrets. |
| `services/meeting-api/` | Replace | LemonSlice session controller/control plane should own worker creation and callbacks. |

## Transcription and recording

| Component | Classification | Rationale |
|---|---|---|
| Whisper/transcription services | Disable first, delete after end-to-end tests | LemonSlice owns STT. Verify no health or speaker dependency remains. |
| Transcript persistence/publishing | Disable first, delete after tests | Out of scope. |
| Generic recording services | Disable first | Not required, but confirm cleanup and monitoring independence. |
| Platform `recording.ts` modules | Split, not delete | They own meeting observation and lifecycle. Extract `runUntilMeetingEnds()` before removing capture. |

## Dashboard, API, MCP, billing, generic agent runtime

Replace the worker-creation/control portions with LemonSlice-specific code. Disable and later delete dashboard, MCP, admin, billing, transcript, and generic agent services once the standalone worker passes full integration tests.

## Unknown and needs investigation

- Any implicit use of `reconnectionIntervalMs`.
- Platform behavior when an audio sender is created from a silent placeholder before admission.
- Zoom Web direct audio/video sender replacement across current UI versions.
- Whether Teams caption code is required only for transcription or also for robust participant monitoring.
- Exact dependencies of every shutdown branch in `performGracefulLeave`.
- Authenticated Google/Teams profile longevity and provider policy constraints.
