# Target Standalone Repository Structure

> **Repository baseline:** `tecxbro/vexa` → `main` at commit `90e5c726a75ea7ba349946648e6cf7f4e1cc845a`  
> **Source rule:** The audited findings override conflicting or inferred DeepWiki statements.  
> **Preservation note:** The original DeepWiki Q&A and audit files are retained in the external source pack and are intentionally not committed here.


The target keeps platform UI automation isolated from LemonSlice provider logic and makes direct media tracks the primary path. The current external webpage screen-capture bridge lives under `legacy/` during migration.

```text
lemonslice-meeting-worker/
├── container/
│   ├── Dockerfile
│   └── entrypoint.sh
├── src/
│   ├── main.ts
│   ├── worker/
│   │   ├── run-worker.ts
│   │   ├── context.ts
│   │   ├── shutdown.ts
│   │   └── signals.ts
│   ├── config/
│   │   ├── schema.ts
│   │   ├── normalize.ts
│   │   └── validate-platform-url.ts
│   ├── browser/
│   │   ├── launch.ts
│   │   ├── args.ts
│   │   ├── injection.ts
│   │   ├── profile.ts
│   │   └── permissions.ts
│   ├── lifecycle/
│   │   ├── meeting-flow.ts
│   │   ├── states.ts
│   │   ├── timeouts.ts
│   │   └── escalation.ts
│   ├── platforms/
│   │   ├── types.ts
│   │   ├── google-meet/
│   │   │   ├── adapter.ts
│   │   │   ├── join.ts
│   │   │   ├── admission.ts
│   │   │   ├── monitoring.ts
│   │   │   ├── removal.ts
│   │   │   ├── leave.ts
│   │   │   ├── media-ui.ts
│   │   │   ├── selectors.ts
│   │   │   └── humanized/
│   │   ├── teams/
│   │   └── zoom-web/
│   ├── media/
│   │   ├── meeting-audio-source.ts
│   │   ├── receiver-audio-source.ts
│   │   ├── html-media-fallback.ts
│   │   ├── audio-mixer.ts
│   │   ├── track-publisher.ts
│   │   ├── audio-publisher.ts
│   │   ├── video-publisher.ts
│   │   ├── placeholder-tracks.ts
│   │   ├── peer-connection-registry.ts
│   │   └── health.ts
│   ├── providers/
│   │   ├── types.ts
│   │   ├── daily/
│   │   │   ├── provider.ts
│   │   │   ├── bridge-client.ts
│   │   │   └── health.ts
│   │   └── livekit/
│   │       └── provider.ts
│   ├── bridge/
│   │   ├── local-webrtc.ts
│   │   ├── auth.ts
│   │   └── protocol.ts
│   ├── identities/
│   │   ├── profile-store.ts
│   │   ├── s3-profile-store.ts
│   │   ├── http-profile-store.ts
│   │   └── browser-profile.ts
│   ├── control/
│   │   ├── commands.ts
│   │   └── redis.ts
│   ├── events/
│   │   ├── lifecycle-events.ts
│   │   └── callback-client.ts
│   ├── diagnostics/
│   │   ├── log.ts
│   │   ├── screenshots.ts
│   │   ├── resources.ts
│   │   └── escalation-access.ts
│   └── legacy/
│       └── webpage-streamer/
├── tests/
│   ├── unit/
│   ├── contract/
│   ├── platform-fixtures/
│   ├── integration/
│   └── end-to-end/
└── deploy/
    ├── helm/
    └── network-policies/
```

## Mapping and treatment

### `container/`

**Responsibility:** OS processes and packages required by the worker.

**Vexa source:** bot Dockerfile and `entrypoint.sh`.

**Treatment:** Refactor. Keep Xvfb and browser runtime. Keep PulseAudio only while fallback audio exists. Remove Zoom SDK paths if native Zoom is excluded. Add a non-root user, secure defaults, and conditional VNC/CDP.

### `src/main.ts`

**Responsibility:** Read config, validate, and invoke the worker.

**Vexa source:** `src/docker.ts`.

**Treatment:** Rename and refactor. Remove browser-session mode from the normal binary and enforce cross-field invariants.

### `src/worker/`

**Responsibility:** Bootstrap, dependency ownership, stop signals, and idempotent shutdown.

**Vexa source:** `runBot()`, module globals, signal handlers, and `performGracefulLeave()` in `index.ts`.

**Treatment:** Split. The existing implementation cannot move unchanged because it mixes browser, transcription, recording, commands, callbacks, and media services.

### `src/config/`

**Responsibility:** Typed LemonSlice worker/provider/platform configuration.

**Vexa source:** `BotConfigSchema`, aliases, and types.

**Treatment:** Rewrite schema; reuse normalization style. Add platform hostname validation, provider discriminated unions, profile backend invariants, and debug policy.

### `src/browser/`

**Responsibility:** Anonymous/persistent browser creation, arguments, init scripts, permissions, and profile attachment.

**Vexa source:** browser launch section of `runBot`, `constans.ts`, `utils/injection.ts`, `browser-profile.ts`.

**Treatment:** Extract/refactor. Correct authenticated CDP behavior and make debug opt-in.

### `src/lifecycle/`

**Responsibility:** Platform-independent state machine.

**Vexa source:** `platforms/shared/meetingFlow.ts`, escalation, stop flags, timeout policy.

**Treatment:** Refactor. Remove Teams imports, verify admission before ACTIVE, and replace recording-shaped lifetime control.

### `src/platforms/`

**Responsibility:** UI selectors, join/admission/removal/leave, platform observation, and sender-creation actions.

**Vexa source:** `googlemeet/`, `msteams/`, and `zoom/web/`.

**Treatment:** Move mostly unchanged first, then extract duplicated monitoring/preparation. Rename `recording.ts` to `monitoring.ts` after separating capture.

### `src/media/`

**Responsibility:** Provider-neutral meeting audio acquisition and agent track publication.

**Vexa source:** portions of `utils/browser.ts`, `screen-content.ts`, `meeting-video-injector.ts`, microphone logic, peer-connection globals.

**Treatment:** Split and partly rewrite. Prefer `getReceivers()` for incoming audio. Retain HTML media fallback. Reuse sender-replacement and stats logic with neutral names.

### `src/providers/types.ts`

**Responsibility:** Provider contract.

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

**Vexa source:** No clean equivalent.

**Treatment:** New code.

### `src/providers/daily/`

**Responsibility:** Join LemonSlice Hosted Daily, accept meeting audio, expose exact agent tracks, and report health.

**Vexa source:** Generic external page integration provides useful lifecycle ideas but not the correct final track interface.

**Treatment:** New implementation. It may use a private LemonSlice bridge page, but should not screen-capture it.

### `src/providers/livekit/`

**Responsibility:** Later provider implementing the same interface.

**Treatment:** Stub initially; no platform code may depend on Daily-specific types.

### `src/bridge/`

**Responsibility:** Authenticated loopback-only local media/control protocol between browser contexts or provider process boundaries.

**Vexa source:** `/offer`, `/offer_meeting_audio`, payload protocol.

**Treatment:** Rewrite/harden. Bind loopback, per-worker authentication, strict origin, typed messages, clear health/reconnect behavior.

### `src/identities/`

**Responsibility:** Restore authenticated Google/Microsoft browser identities through a storage interface.

**Vexa source:** `s3-sync.ts`, HTTP cookie client, `browser-profile.ts`.

**Treatment:** Move/refactor behind `ProfileStore`; secure credentials and cleanup.

### `src/control/`

**Responsibility:** Typed worker commands.

**Vexa source:** Redis subscriber and `handleRedisMessage`.

**Treatment:** Refactor to stop, health, reconnect, and escalation commands. Keep transport replaceable.

### `src/events/`

**Responsibility:** Lifecycle and health events to the LemonSlice session controller.

**Vexa source:** unified callback service.

**Treatment:** Rename and adapt payload; keep retry and reason mapping where useful.

### `src/diagnostics/`

**Responsibility:** Structured logs, screenshots, resource metrics, and secure temporary VNC/CDP access.

**Vexa source:** log utilities, resource sampler, screenshots, escalation.

**Treatment:** Move/refactor. Do not expose raw debug ports by default.

### `src/legacy/webpage-streamer/`

**Responsibility:** Temporary compatibility path for current external-page audio/video behavior.

**Vexa source:** Python streamer, runner, payload, Xvfb/GStreamer support, `VoiceAgentPageService` bridge portions.

**Treatment:** Move only as a temporary flag-protected path. Harden immediately. Delete after direct Daily audio/video track forwarding passes all platform tests.

### `tests/`

Required test classes:

- Schema invariants and platform URL validation.
- Adapter selector fixtures and admission state tests.
- Provider contract tests.
- Peer-connection sender replacement tests.
- Audio mixer and echo-exclusion tests.
- Profile-store contract tests.
- Shutdown idempotency.
- Meet/Teams/Zoom Web end-to-end join, admission, media, removal, and everyone-left scenarios.
- Security tests for bridge token/origin, closed debug ports, and secret redaction.
