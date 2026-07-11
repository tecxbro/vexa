# Incoming Meeting Audio: Meeting → LemonSlice

> **Repository baseline:** `tecxbro/vexa` → `main` at commit `90e5c726a75ea7ba349946648e6cf7f4e1cc845a`  
> **Source rule:** The audited findings override conflicting or inferred DeepWiki statements.  
> **Preservation note:** The original DeepWiki Q&A and audit files are retained in the external source pack and are intentionally not committed here.


## Correct current path

The audited repository does **not** send meeting audio to the external LemonSlice page through PulseAudio. The actual path is browser media discovery plus two local WebRTC connections.

```text
Meeting participants
  → platform remote WebRTC tracks
  → meeting page <audio>/<video>.srcObject streams
  → screen-content.ts ensureMeetingAudioStream()
  → Web Audio mix when multiple streams exist
  → meeting browser creates local WebRTC offer
  → POST /offer to Python webpage streamer
  → Python stores received track as _upstream_audio_track
  → external page creates second local WebRTC offer
  → POST /offer_meeting_audio
  → Python rebroadcasts _upstream_audio_track
  → webpage_streamer_payload.js receives meeting track
  → patched navigator.mediaDevices.getUserMedia returns it
  → LemonSlice/Daily consumes it as microphone input
```

The original DeepWiki statement that no JavaScript `getUserMedia` interception occurs in the external page is superseded.

## 1. Discovering meeting audio

The meeting browser inspects active HTML media elements whose `srcObject` contains live audio tracks. The current logic filters out:

- Paused elements.
- Muted elements.
- Zero-volume elements.
- Ended or inactive tracks.
- Duplicate tracks.
- Known bot-output streams to reduce feedback.

When multiple eligible streams are present, Web Audio nodes feed a `MediaStreamDestination` to create one combined meeting-audio track.

This is separate from Vexa's per-speaker transcription pipeline. LemonSlice only requires the provider-neutral combined meeting track unless it later needs attributed tracks.

## 2. First local WebRTC connection

The meeting browser creates an `RTCPeerConnection`, adds the combined meeting-audio track, requests the external page's video/audio media, and sends its offer to the local Python service at `/offer`.

Python receives and stores the upstream meeting-audio track while returning the screen-captured external page media to the meeting browser.

## 3. Second local WebRTC connection

The payload injected into the external bridge page replaces `navigator.mediaDevices.getUserMedia`.

When the bridge page asks for audio, the payload:

- Creates another local `RTCPeerConnection`.
- Posts an offer to `/offer_meeting_audio`.
- Receives the stored `_upstream_audio_track` from Python.
- Resolves `getUserMedia()` with that track as the page microphone stream.

This is the point at which a LemonSlice Daily page can receive meeting audio without a physical microphone.

## 4. PulseAudio's actual role

PulseAudio is used in the reverse direction: external LemonSlice page output toward the meeting microphone. It is not the meeting-to-LemonSlice transport.

## Current dependencies

- Main meeting Chromium on X display `:99`.
- `screen-content.ts` meeting-audio discovery/mixing.
- Browser init and payload scripts.
- Python `aiohttp`/`aiortc` local service on port `8124`.
- Two local WebRTC peer-connection pairs.
- `webpage_streamer_payload.js` in the second browser page.
- The external bridge page's normal `getUserMedia` request.

## Current limitations

### HTML element discovery

The current source depends on meeting media being attached to discoverable HTML elements. Platform rendering changes, offscreen elements, or alternate media pipelines can break discovery.

### Echo filtering is heuristic

Known bot-output tracks are excluded, but HTML-element identity is a weaker boundary than WebRTC sender/receiver direction.

### Connection lifecycle

The meeting-to-Python and Python-to-external-page peer connections can become disconnected or stale. Health must include peer-connection state, track `readyState`, audio activity, and reconnection.

### Local bridge security

The Python service currently listens on `0.0.0.0:8124`, has permissive CORS, and exposes unauthenticated endpoints. An arbitrary external page can reach it. Production must bind to loopback, require a per-worker token, and restrict the expected bridge origin.

## Production target

The preferred source is the meeting browser's remote `RTCRtpReceiver` audio tracks:

```text
meeting RTCPeerConnection.getReceivers()
  → live remote audio receiver tracks
  → provider-neutral mixer
  → secure local WebRTC bridge
  → DailyProvider.setMeetingAudio(track)
```

Receiver tracks provide a cleaner separation: remote participant audio is a receiver, while the bot microphone is a sender. Keep HTML media-element discovery only as a compatibility fallback.

## LemonSlice seam

The provider contract should expose:

```ts
setMeetingAudio(track: MediaStreamTrack): Promise<void>
```

For Daily, the bridge page or a direct Daily integration publishes that track as the agent session's microphone input. The meeting-worker core should not contain Daily-specific room logic.
