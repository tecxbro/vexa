# Agent Video and Meeting Camera Injection

> **Repository baseline:** `tecxbro/vexa` → `main` at commit `90e5c726a75ea7ba349946648e6cf7f4e1cc845a`  
> **Source rule:** The audited findings override conflicting or inferred DeepWiki statements.  
> **Preservation note:** The original DeepWiki Q&A and audit files are retained in the external source pack and are intentionally not committed here.


## Current architecture

The current path has a default canvas camera and a preferred external-page video track.

```text
External LemonSlice page
  → second Chromium process on X display :98
  → entire X display captured by GStreamer ximagesrc
  → 1920×1080 I420, 15 FPS
  → GstVideoStreamTrack / aiortc
  → local WebRTC to the meeting browser
  → preferred video track in browser globals
  → meeting RTCRtpSender.replaceTrack()
  → Meet / Teams / Zoom Web camera sender
```

## Browser init script

`getVirtualCameraInitScript()` is injected before the meeting page is created/navigated. It:

- Creates a canvas-backed default video track.
- Patches `navigator.mediaDevices.getUserMedia`.
- Tracks `RTCPeerConnection` instances.
- Hooks `addTrack`, sender `replaceTrack`, and offer creation.
- Exposes browser globals for preferred-track selection and peer-connection tracking.
- Supports replacing outgoing camera senders with an external streamer track.

`--use-fake-ui-for-media-stream` only accepts permission prompts; it does not bypass these JavaScript patches.

## External page capture

`VoiceAgentPageService` starts the Python streamer and a second browser on display `:98`. GStreamer uses `ximagesrc` to capture the whole virtual display, converts it to I420, scales it, and exposes it as an aiortc video track.

This captures rendered pixels, including:

- LemonSlice page controls.
- Loading and error states.
- Browser-page overlays.
- Any visual artifact displayed in that X session.

It is therefore a webpage screen-capture bridge, not direct avatar-track forwarding.

## Meeting-browser injection

The meeting browser receives the Python video track over local WebRTC. The media injection service selects that track over the default canvas, finds outgoing video senders, and calls `replaceTrack()`.

The service retries because:

- The camera sender may not exist in the lobby.
- Meet or Teams may recreate peer connections during admission.
- The camera UI may need to be toggled to force sender negotiation.
- The external streamer connection can become stale.

Post-admission recovery re-applies the track. Outbound RTP stats, including `framesSent`, are used to determine whether frames are actually leaving the browser.

## Platform-specific versus shared behavior

Shared:

- Preferred track selection.
- Peer-connection tracking.
- Sender replacement.
- Stats-based health.
- Streamer lifecycle.

Platform-specific:

- Camera button selectors.
- Whether/how to toggle camera for renegotiation.
- Timing of sender creation.
- Admission transition behavior.

The current shared camera service contains Meet and Teams selectors; this should be delegated to platform adapters.

## Prototype suitability

A LemonSlice page can be loaded without changing its application code, and its rendered view can appear as the meeting camera. That is useful for prototyping.

It is not production-grade because it introduces:

- A second X server and browser.
- Full-page rendering.
- Screen capture.
- Pixel conversion and scaling.
- An extra WebRTC encode/decode path.
- Visual contamination risk.
- More reconnection points.

## Production target

```text
Exact LemonSlice avatar MediaStreamTrack
  → MeetingTrackPublisher.setVideoTrack()
  → meeting RTCRtpSender.replaceTrack()
```

The Daily provider should subscribe to and return the exact avatar video track. The worker should publish that track directly. A placeholder canvas track can remain to create the sender before the LemonSlice track is ready.

## Health requirements

Video health should report:

- Provider room and participant state.
- Agent video track existence and `readyState`.
- Stream dimensions and frame activity.
- Meeting sender existence.
- Peer-connection state.
- Outbound `framesSent`, bytes, frame rate, and stalls.
- Last successful track replacement.
- Reconnection and renegotiation attempts.

## Compatibility plan

Keep the current GStreamer/Xvfb bridge behind a temporary feature flag until direct Daily track forwarding is stable across Meet, Teams, and Zoom Web. It should not remain the default final architecture.
