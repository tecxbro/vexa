# Shared Platform Architecture

> **Repository baseline:** `tecxbro/vexa` → `main` at commit `90e5c726a75ea7ba349946648e6cf7f4e1cc845a`  
> **Source rule:** The audited findings override conflicting or inferred DeepWiki statements.  
> **Preservation note:** The original DeepWiki Q&A and audit files are retained in the external source pack and are intentionally not committed here.


## Core strategy pattern

The browser-platform architecture is organized around a shared lifecycle and a platform strategy object. The current strategy surface is conceptually:

```ts
interface PlatformStrategies {{
  join(...): Promise<void>;
  waitForAdmission(...): Promise<boolean | AdmissionDecision>;
  checkAdmissionSilent(...): Promise<boolean>;
  prepare(...): Promise<void>;
  startRecording(...): Promise<void>;       // actually capture + monitoring + lifetime
  startRemovalMonitor(...): StopFunction;
  leave(...): Promise<void>;
}}
```

Google Meet, Teams, Zoom Web, and native Zoom provide implementations. `Page | null` exists because native Zoom does not use a Playwright page.

## Responsibilities that are genuinely shared

`runMeetingFlow()` currently coordinates:

- Meeting URL guard.
- Platform join.
- Stop-signal guard.
- Admission wait and preparation.
- Rejection/timeout mapping.
- Lifecycle callbacks.
- Post-admission services.
- Removal race.
- Graceful leave.

Browser creation is also shared for Meet, Teams, and Zoom Web. They use the same Chromium runtime, profile modes, permissions, injection layer, diagnostics, and control channel.

## Platform-specific responsibilities

| Responsibility | Google Meet | Teams | Zoom Web | Native Zoom SDK |
|---|---|---|---|---|
| Browser creation | Shared browser | Shared browser | Shared browser | None |
| Join UI | Meet adapter | Teams adapter | Zoom Web adapter | SDK strategy |
| Admission | Meet DOM/state | Teams DOM/modals | Zoom Web DOM | SDK events |
| Mic/camera UI | Meet selectors | Teams selectors | Zoom selectors | SDK calls |
| Removal | Meet adapter | Teams adapter | Zoom Web adapter | SDK events |
| Participant/end monitoring | Hidden in Meet monitoring/recording | Hidden in Teams monitoring/recording | Zoom Web monitoring | SDK strategy |
| Meeting audio discovery | Meeting media/WebRTC | Meeting media/WebRTC/captions | Browser/PulseAudio-dependent paths | Raw SDK callbacks |
| Leave | Meet-specific | Teams-specific | Zoom Web-specific | SDK-specific |

## Zoom Web versus native Zoom

Zoom Web is the default browser architecture and belongs with Meet and Teams. Native Zoom is an opt-in path selected by environment variables and uses a C++ N-API addon instead of Chromium.

For the standalone LemonSlice worker, preserve Zoom Web. Keep native Zoom disabled until a product requirement proves it is needed; delete it only after browser-based end-to-end coverage exists and no operational dependency remains.

## Current architectural contamination

### Teams code in the shared lifecycle

`meetingFlow.ts` imports and calls Teams live-caption logic directly. That belongs in a Teams post-admission hook.

### Preparation duplicated between Meet and Teams

Both adapters have near-identical `prepareForRecording` behavior that exposes browser globals and installs a leave action, differing mainly by selector lists. Extract common preparation and keep selectors inside adapters.

### Meeting monitoring duplicated

Meet and Teams implement similar:

- Participant/speaker presence.
- Startup-alone timeout.
- Everyone-left timeout.
- Long-running meeting lifetime.

The common state machine should be shared, while each adapter supplies platform observations.

### Teams removal duplicated

Teams removal is checked both by a dedicated removal monitor and inside the long-running recording/monitoring code. Consolidate it to one adapter-owned signal.

### Camera renegotiation mixes selectors

The shared screen/camera service contains both Meet and Teams selectors. The shared service should request an adapter action such as `ensureCameraSender()` or `toggleCameraForRenegotiation()`.

### Microphone service contains platform branches

A shared microphone controller is reasonable, but DOM selectors and UI toggling should be delegated to adapters.

### ACTIVE before verification

The shared flow emits ACTIVE before silent verification. Reverse that order.

## Hidden responsibility inside `startRecording`

The strategy name is misleading. For browser platforms, the platform recording module may own:

- Browser utility injection.
- Participant observation.
- Speaker/activity signals.
- Startup-alone and everyone-left timeouts.
- Removal cross-validation.
- The unresolved promise that represents the active meeting lifetime.

The target interface should separate:

```ts
startOptionalCapture(): Promise<StopFunction>
runUntilMeetingEnds(): Promise<MeetingEndReason>
```

Only optional capture should disappear from the LemonSlice worker.

## Recommended shared interfaces

```ts
interface MeetingPlatformAdapter {
  validateUrl(url: URL): void;
  join(ctx: WorkerContext): Promise<void>;
  waitForAdmission(ctx: WorkerContext): Promise<AdmissionResult>;
  verifyAdmission(ctx: WorkerContext): Promise<boolean>;
  prepareMediaSenders(ctx: WorkerContext): Promise<void>;
  observeMeeting(ctx: WorkerContext): AsyncIterable<MeetingObservation>;
  leave(ctx: WorkerContext): Promise<void>;
}

interface MeetingAudioSource {
  start(): Promise<MediaStreamTrack>;
  health(): AudioSourceHealth;
  stop(): Promise<void>;
}

interface MeetingTrackPublisher {
  setAudioTrack(track: MediaStreamTrack): Promise<void>;
  setVideoTrack(track: MediaStreamTrack): Promise<void>;
  health(): PublisherHealth;
  stop(): Promise<void>;
}
```

The provider-neutral media bridge must not import platform selectors. Platform adapters must not import Daily or LiveKit SDK logic.
