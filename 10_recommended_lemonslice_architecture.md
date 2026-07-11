# Recommended LemonSlice Architecture and Transition

> **Repository baseline:** `tecxbro/vexa` → `main` at commit `90e5c726a75ea7ba349946648e6cf7f4e1cc845a`  
> **Source rule:** The audited findings override conflicting or inferred DeepWiki statements.  
> **Preservation note:** The original DeepWiki Q&A and audit files are retained in the external source pack and are intentionally not committed here.


## Central conclusion

Vexa already has the meeting-participant infrastructure LemonSlice needs: browser joining, platform adapters, admission handling, identities, lifecycle, monitoring, diagnostics, and cleanup.

Keep that infrastructure. Replace the current generic external-webpage media bridge as the production path.

## Target runtime

```text
LemonSlice Session Controller
  │ start / stop / health / provider credentials
  ▼
One isolated worker container
  ├── Shared worker lifecycle
  ├── GoogleMeetAdapter | TeamsAdapter | ZoomWebAdapter
  ├── MeetingAudioSource
  │     └── remote RTCRtpReceiver tracks → mixer
  ├── AgentMediaProvider
  │     ├── DailyProvider now
  │     └── LiveKitProvider later
  └── MeetingTrackPublisher
        ├── audio sender.replaceTrack()
        └── video sender.replaceTrack()
```

## Provider interface

```ts
interface AgentMediaProvider {
  start(config: ProviderConfig): Promise<void>;
  setMeetingAudio(track: MediaStreamTrack): Promise<void>;
  getAgentAudioTrack(): Promise<MediaStreamTrack>;
  getAgentVideoTrack(): Promise<MediaStreamTrack>;
  health(): ProviderHealth;
  stop(): Promise<void>;
}
```

## Daily provider responsibilities

The initial Daily provider should:

- Join the LemonSlice Hosted Daily session.
- Accept the mixed meeting-audio track as the provider microphone input.
- Identify the LemonSlice avatar participant/track unambiguously.
- Return the exact LemonSlice audio track.
- Return the exact LemonSlice video track.
- Report room, participant, track, and connection health.
- Reconnect or fail deterministically when tracks end or the room disconnects.

The bridge page may remain a private LemonSlice implementation detail. The worker contract must be tracks and health, not webpage pixels.

## Future LiveKit provider

LiveKit implements the same provider interface. Meeting lifecycle and platform adapters must not change when the provider changes.

## Incoming audio target

Current:

```text
HTML media elements → Web Audio → local WebRTC → patched bridge getUserMedia
```

Target:

```text
remote RTCRtpReceiver audio tracks
  → provider-neutral mixer
  → DailyProvider.setMeetingAudio(track)
```

Use HTML media discovery only when receiver access is insufficient on a platform.

## Outgoing audio target

Current:

```text
external Chromium → PulseAudio monitor → loopback → virtual_mic → meeting microphone
```

Target:

```text
Daily LemonSlice audio track → meeting audio sender.replaceTrack()
```

Create a silent placeholder before joining to establish the sender.

## Outgoing video target

Current:

```text
external page → X display → GStreamer screen capture → aiortc → meeting sender
```

Target:

```text
Daily LemonSlice avatar video track → meeting video sender.replaceTrack()
```

Keep a canvas placeholder to establish the sender and show a controlled fallback.

## What remains from Vexa

Keep:

- Container-per-meeting model.
- Playwright meeting browser.
- Meet, Teams, and Zoom Web adapters.
- Humanized Google Meet interaction.
- Authenticated profiles.
- Shared lifecycle after refactoring.
- Redis or equivalent stop/control channel.
- Status callbacks.
- Removal/everybody-left monitoring.
- Screenshots, structured logs, browser-crash diagnostics.
- Secure, disabled-by-default escalation access.

Refactor:

- `runBot` and `runMeetingFlow`.
- Browser arguments and debug policy.
- Platform recording modules into monitoring plus optional capture.
- Screen/camera service into a provider-neutral publisher.
- Voice-agent page service into a provider lifecycle.
- Profile storage behind an interface.

Replace:

- Screen-captured external-page video.
- PulseAudio live-agent output routing.
- HTML-only meeting audio as the primary source.
- Arbitrary external URL loading.
- Unauthenticated bridge endpoints.
- Vexa meeting API/control plane with LemonSlice session controller code.

Disable, observe, then remove:

- Whisper and transcript services.
- Recording/upload services.
- Vexa TTS generation.
- Dashboard, MCP, billing, admin, and generic agent services.
- Native Zoom SDK unless explicitly required.

## Recommended transition order

1. Establish a strict LemonSlice worker configuration with provider discriminated unions, platform URL validation, identity backend invariants, and secure debug policy.
2. Extract meeting monitoring from platform recording modules into `runUntilMeetingEnds()`.
3. Preserve and test all three browser adapters before media changes.
4. Introduce `AgentMediaProvider` and implement `DailyProvider` health/lifecycle.
5. Feed meeting audio into Daily through a secure local track bridge.
6. Return exact Daily LemonSlice audio/video tracks.
7. Establish placeholder meeting senders and publish both tracks with `replaceTrack()`.
8. Keep the current PulseAudio/GStreamer bridge behind a compatibility flag.
9. Add reconnection based on peer-connection state, provider state, track `ended`, audio activity, and outbound RTP stats.
10. Harden loopback endpoints, profile credentials, VNC/CDP, container user, sandbox, secrets, and network policy.
11. Run platform end-to-end tests for anonymous/authenticated joins, waiting rooms, immediate admission, denial, captcha escalation, media renegotiation, removal, everybody-left, stop, crash, and cleanup.
12. Delete compatibility and unused Vexa product areas only after evidence shows no hidden lifecycle dependency.

## Acceptance model

A worker is healthy only when all layers agree:

- Platform adapter says admitted.
- LemonSlice provider is connected.
- Meeting audio is reaching the provider.
- LemonSlice audio/video tracks are live.
- Meeting senders exist and use the expected tracks.
- Outbound RTP stats advance.
- Stop and meeting-end signals still work.
- No debug or bridge ports are publicly exposed.
