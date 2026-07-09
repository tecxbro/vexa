# LemonSlice Meeting Worker — Vexa Audit Map

This branch is a safe workspace for reducing Vexa into a multi-platform LemonSlice meeting worker. `main` remains untouched.

## Decision

Use Vexa as the cloud meeting participant and browser automation layer.

Use LemonSlice Hosted Daily as the first media/agent backend because the existing LemonSlice hosted pipeline already produces a Daily room. Do not add LiveKit or Pipecat unless the LemonSlice agent itself moves to those frameworks.

Vexa does not need a provider-specific integration. Its existing boundary is `voiceAgentSettings.url`: it opens an arbitrary voice-agent webpage in a second Chromium process and bridges that webpage with Google Meet, Microsoft Teams, or Zoom.

## Target architecture

```text
LemonSlice session controller
  ├─ creates LemonSlice Hosted Daily session
  ├─ creates short-lived bridge-page URL
  └─ starts one meeting-worker container with BOT_CONFIG

Meeting-worker container
  ├─ Chromium A: Google Meet / Teams / Zoom
  ├─ Playwright: join, admission, controls, leave, recovery
  ├─ Chromium B: LemonSlice Daily bridge page
  ├─ Meet/Teams/Zoom audio -> Chromium B microphone
  ├─ LemonSlice avatar audio -> meeting virtual microphone
  └─ LemonSlice avatar video -> meeting WebRTC camera sender
```

## Current media flow in Vexa

### Meeting audio to LemonSlice

1. The meeting page scans `<audio>` and `<video>` elements for live audio tracks.
2. Those tracks are combined into one meeting-audio `MediaStream`.
3. A local WebRTC peer connection sends the meeting-audio track to the Python webpage streamer.
4. The Python streamer exposes that track back to Chromium B.
5. `webpage_streamer_payload.js` overrides `getUserMedia({audio: ...})` so the LemonSlice/Daily page receives meeting audio as its microphone.

### LemonSlice audio to the meeting

1. Chromium B plays the LemonSlice avatar audio.
2. Chromium B is configured to output to `webpage_streamer_sink`.
3. PulseAudio loops `webpage_streamer_sink.monitor` into `tts_sink`.
4. `tts_sink.monitor` is exposed as `virtual_mic`.
5. Chromium A uses `virtual_mic` as the meeting microphone.

### LemonSlice video to the meeting

Current implementation:

1. Chromium B renders the LemonSlice page on a dedicated Xvfb display.
2. GStreamer captures that display at 15 FPS.
3. Python `aiortc` publishes the captured frames over a local WebRTC connection.
4. Chromium A receives the video track.
5. The meeting-page init script swaps the Meet/Teams/Zoom outgoing camera sender to that track.

Production optimization:

Replace the Xvfb/GStreamer screen-capture step with direct forwarding of LemonSlice's remote Daily `MediaStreamTrack`. Keep the current capture path as a fallback and debugging mode.

## Chrome camera equivalent of the FaceTime camera extension

FaceTime needs a macOS CoreMediaIO camera extension because FaceTime obtains cameras from the operating system.

Browser meeting apps obtain media through browser WebRTC APIs. Vexa injects JavaScript before the meeting page loads and patches:

- `navigator.mediaDevices.getUserMedia`
- `RTCPeerConnection.prototype.addTrack`
- `RTCRtpSender.prototype.replaceTrack`
- `RTCPeerConnection.prototype.createOffer`

The injected code supplies a canvas or LemonSlice video track instead of a physical camera track. No macOS camera extension is involved.

## Required core files — audit first

### Worker startup and orchestration

- `services/vexa-bot/core/src/docker.ts`
  - Parses `BOT_CONFIG` and launches the worker.
- `services/vexa-bot/core/src/types.ts`
  - Defines the worker contract and platform/media flags.
- `services/vexa-bot/core/src/index.ts`
  - Browser launch, service initialization, platform routing, cleanup.
- `services/vexa-bot/core/src/constans.ts`
  - Chromium arguments, fake media devices, user agent and CDP settings.
- `services/vexa-bot/core/src/platforms/shared/meetingFlow.ts`
  - Shared join -> admission -> active -> monitor -> leave lifecycle.
- `services/vexa-bot/core/src/services/unified-callback.ts`
  - Worker state callbacks. Replace or adapt to the LemonSlice controller.
- `services/vexa-bot/core/src/utils.ts`
  - Shared callbacks/log helpers used by the platform adapters.

### Media bridge

- `services/vexa-bot/core/src/services/voice-agent-page.ts`
  - Starts Chromium B, Xvfb, the local streamer and audio routing.
- `services/vexa-bot/core/src/services/screen-content.ts`
  - Meeting-page media hooks, `getUserMedia` patch, WebRTC track swapping, canvas fallback.
- `services/vexa-bot/core/src/services/meeting-video-injector.ts`
  - Post-admission video recovery and outbound-frame verification.
- `services/vexa-bot/core/src/services/microphone.ts`
  - Meeting UI mute/unmute controls across all platforms.
- `services/vexa-bot/core/webpage_streamer/run_webpage_streamer.py`
  - Python streamer process entrypoint.
- `services/vexa-bot/core/webpage_streamer/webpage_streamer.py`
  - Chromium B, GStreamer capture and local `aiortc` bridge.
- `services/vexa-bot/core/webpage_streamer/webpage_streamer_payload.js`
  - Supplies meeting audio as Chromium B's virtual microphone.
- `services/vexa-bot/core/webpage_streamer/requirements.txt`
  - `aiohttp`, `aiortc`, `av`, `numpy`, Playwright.

### Browser-side audio utilities

These are required only if retaining Vexa recording, speaker attribution or browser-side audio diagnostics:

- `services/vexa-bot/core/src/utils/browser.ts`
- `services/vexa-bot/core/src/utils/injection.ts`
- `services/vexa-bot/core/build-browser-utils.js`

The LemonSlice media bridge itself should eventually avoid depending on Vexa's transcription pipeline.

### Runtime image

- `services/vexa-bot/core/Dockerfile`
  - Chromium, Edge, Xvfb, PulseAudio, GStreamer, Python and Playwright.
- `services/vexa-bot/core/entrypoint.sh`
  - Starts Xvfb, PulseAudio, `tts_sink`, `virtual_mic`, VNC and the worker.
- `services/vexa-bot/core/package.json`
- `services/vexa-bot/core/tsconfig.json`

## Google Meet adapter files

Audit these in this order:

1. `services/vexa-bot/core/src/platforms/googlemeet/index.ts`
2. `services/vexa-bot/core/src/platforms/googlemeet/join.ts`
3. `services/vexa-bot/core/src/platforms/googlemeet/humanized.ts`
4. `services/vexa-bot/core/src/platforms/googlemeet/selectors.ts`
5. `services/vexa-bot/core/src/platforms/googlemeet/admission.ts`
6. `services/vexa-bot/core/src/platforms/googlemeet/leave.ts`
7. `services/vexa-bot/core/src/platforms/googlemeet/removal.ts`
8. `services/vexa-bot/core/src/platforms/googlemeet/recording.ts`

`recording.ts` contains meeting monitoring and participant/speaker DOM logic in addition to recording. Split those responsibilities before deleting recording support.

## Microsoft Teams adapter files

- `services/vexa-bot/core/src/platforms/msteams/index.ts`
- `services/vexa-bot/core/src/platforms/msteams/join.ts`
- `services/vexa-bot/core/src/platforms/msteams/admission.ts`
- `services/vexa-bot/core/src/platforms/msteams/leave.ts`
- `services/vexa-bot/core/src/platforms/msteams/removal.ts`
- `services/vexa-bot/core/src/platforms/msteams/recording.ts`
- `services/vexa-bot/core/src/platforms/msteams/selectors.ts`
- `services/vexa-bot/core/src/platforms/msteams/captions.ts`

## Zoom Web adapter files

Prefer Zoom Web for the initial LemonSlice worker. Do not carry the native SDK path unless there is a measured reason.

- `services/vexa-bot/core/src/platforms/zoom/index.ts`
- `services/vexa-bot/core/src/platforms/zoom/web/index.ts`
- `services/vexa-bot/core/src/platforms/zoom/web/join.ts`
- `services/vexa-bot/core/src/platforms/zoom/web/admission.ts`
- `services/vexa-bot/core/src/platforms/zoom/web/prepare.ts`
- `services/vexa-bot/core/src/platforms/zoom/web/recording.ts`
- `services/vexa-bot/core/src/platforms/zoom/web/removal.ts`
- `services/vexa-bot/core/src/platforms/zoom/web/leave.ts`

Candidate for removal:

- `services/vexa-bot/core/src/platforms/zoom/native/`
- `services/vexa-bot/core/src/platforms/zoom/strategies/`

Remove those only after confirming no desired deployment needs Zoom's native Meeting SDK.

## Optional interaction files

Keep only when required by the LemonSlice product:

- `services/vexa-bot/core/src/services/chat.ts`
- `services/vexa-bot/core/src/services/screen-share.ts`
- `services/vexa-bot/core/src/services/tts-playback.ts`

The LemonSlice avatar already produces speech, so Vexa TTS is not required for the normal agent path.

## Disable first; delete later

Set these defaults in the LemonSlice worker:

```json
{
  "transcribeEnabled": false,
  "recordingEnabled": false,
  "voiceAgentEnabled": true,
  "cameraEnabled": true,
  "videoReceiveEnabled": false
}
```

Disable or remove after the meeting bridge passes end-to-end tests:

- `services/transcription-service/`
- `services/tts-service/`
- `services/mcp/`
- `services/dashboard/`
- `services/agent-api/`
- Vexa transcript persistence and Whisper/VAD files inside the bot
- recording upload/finalization services
- remote browser/workspace features not needed by meeting identities

Do not immediately remove:

- `services/runtime-api/` until the LemonSlice session controller can launch/stop isolated workers itself.
- authenticated browser profile storage until Google/Microsoft login identity handling is replaced.
- Redis command handling until an equivalent worker-control channel exists.

## Daily vs LiveKit vs Pipecat

### Choose Daily now

LemonSlice Hosted already returns a Daily room URL and token. Build a private bridge webpage that joins that Daily room and renders only the LemonSlice avatar.

Daily supports a custom `MediaStreamTrack` for its microphone. The production bridge should explicitly install the Meet/Teams/Zoom audio track as the Daily local audio input instead of depending forever on a global `getUserMedia` override.

### Use LiveKit only when

- LemonSlice is running its self-managed LiveKit pipeline, or
- the product needs direct backend track APIs unavailable in the Hosted Daily contract.

Do not add a Daily-to-LiveKit translation layer.

### Use Pipecat only when

Pipecat owns the agent's STT/LLM/TTS pipeline. It is not necessary merely to connect Vexa to LemonSlice.

## New LemonSlice files to build

Suggested additions:

```text
services/lemonslice-bridge-page/
  src/session.ts
  src/daily-client.ts
  src/avatar-tracks.ts
  src/meeting-audio-input.ts
  src/health.ts

services/lemonslice-meeting-controller/
  src/create-session.ts
  src/worker-config.ts
  src/worker-launcher.ts
  src/worker-events.ts
  src/identity-pool.ts
```

### Bridge-page responsibilities

- Receive a short-lived bridge token, never a long-lived LemonSlice API key.
- Fetch Daily `room_url` and `token` from the LemonSlice controller.
- Join Daily with local video disabled.
- Send meeting audio as the Daily local microphone track.
- Subscribe only to the LemonSlice avatar participant.
- Expose the avatar audio/video tracks to the local Vexa bridge.
- Report `daily_joined`, `bot_ready`, `audio_live`, `video_live`, `ended`, and errors.

## Recommended refactor sequence

1. Run the current fork with `transcribeEnabled=false` and `recordingEnabled=false`.
2. Build the private LemonSlice Daily bridge page.
3. Pass its URL through `voiceAgentSettings.url`.
4. Prove Google Meet audio in both directions and avatar video output.
5. Add Teams and Zoom Web regression tests using the same bridge page.
6. Extract the giant `src/index.ts` into:
   - `worker.ts`
   - `browser/create-context.ts`
   - `media/media-bridge.ts`
   - `lifecycle/shutdown.ts`
   - `control/commands.ts`
7. Replace screen capture with direct Daily track forwarding.
8. Replace Vexa meeting-api/runtime-api callbacks with the LemonSlice controller contract.
9. Remove transcription, recording and product UI services.
10. Add platform canaries, outbound RTP health checks, browser-version pinning and rollback.

## Production acceptance checks

For every platform, a session is healthy only when all are true:

- Browser launched and the expected meeting domain loaded.
- Join action completed.
- Bot was admitted.
- LemonSlice Daily room joined.
- LemonSlice emitted `bot_ready`.
- Meeting audio track is live at the Daily input.
- LemonSlice avatar audio is reaching the meeting microphone.
- LemonSlice avatar video track is live.
- Meeting `outbound-rtp` audio packets increase.
- Meeting `outbound-rtp` video `framesSent` increases.
- Worker leaves and destroys all browser/audio processes after the meeting.

## First code edits after this audit

Do not delete the repository wholesale. The first code PR should:

1. Add a provider-neutral `AgentMediaBridge` interface.
2. Rename `VoiceAgentPageService` to a provider-neutral bridge implementation.
3. Add a LemonSlice Daily bridge-page implementation.
4. Make transcription and recording imports lazy or optional.
5. Split meeting monitoring out of each platform's `recording.ts`.
6. Keep Google Meet, Teams and Zoom Web adapters behind the shared `meetingFlow.ts` interface.
