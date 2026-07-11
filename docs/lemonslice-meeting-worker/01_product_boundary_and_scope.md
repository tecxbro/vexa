# Product Boundary and System Scope

> **Repository baseline:** `tecxbro/vexa` → `main` at commit `90e5c726a75ea7ba349946648e6cf7f4e1cc845a`  
> **Source rule:** The audited findings override conflicting or inferred DeepWiki statements.  
> **Preservation note:** The original DeepWiki Q&A and audit files are retained in the external source pack and are intentionally not committed here.


## Product objective

The product is a production-grade, cloud-hosted meeting worker that lets an existing LemonSlice real-time video agent participate in Google Meet, Microsoft Teams, and Zoom Web as a normal attendee. Nothing runs on the end user's computer. Each active meeting runs in one isolated worker container.

The worker is not a new conversational-agent stack. It is the meeting-participant and media-transport layer around the LemonSlice agent.

## LemonSlice owns

LemonSlice remains the source of truth for:

- The conversational agent and conversation state.
- Speech-to-text and listening behavior.
- LLM reasoning.
- Tool execution.
- Text-to-speech and generated speech.
- Real-time talking-avatar video.
- Turn-taking, interruption, and agent behavior.
- LemonSlice session creation and termination.
- The LemonSlice-hosted media session. The initial provider is Daily; a later provider may be LiveKit.
- A private bridge surface or provider integration that can accept meeting audio and expose the exact LemonSlice audio and video tracks.

The worker must treat the agent as a provider-backed media source, not rebuild any of these systems.

## The meeting worker owns

The meeting worker owns everything needed to represent that agent inside a third-party meeting:

- Start one isolated cloud container per meeting.
- Launch and manage the browser runtime.
- Select Google Meet, Teams, or Zoom Web from configuration.
- Join with the required display name or authenticated identity.
- Handle pre-join UI, microphone/camera state, waiting rooms, admission, rejection, removal, and leave behavior.
- Receive participant audio from the meeting and forward it to the LemonSlice provider.
- Receive LemonSlice audio and video and publish them into the meeting.
- Track connection and media health.
- Handle stop commands, meeting-end conditions, browser failure, timeouts, and cleanup.
- Emit lifecycle callbacks, structured logs, screenshots, and diagnostics.
- Restore and protect authenticated browser profiles when authenticated joining is requested.

## Shared across platforms

The shared meeting-worker core should own:

- Configuration validation and normalization.
- Container and process lifecycle.
- Browser creation, permissions, stealth configuration, and profile restoration.
- Worker state transitions: starting, joining, awaiting admission, active, stopping, ended, failed.
- Stop signals and graceful cleanup.
- Provider-neutral meeting-audio source and agent-track publisher interfaces.
- LemonSlice provider lifecycle.
- Health reporting, callbacks, resource metrics, screenshots, and logs.
- Common escalation interfaces for captcha or login challenges.
- Common timeout policy.

The current Vexa strategy-based `runMeetingFlow` is the right conceptual base, but it is not perfectly platform-neutral today: it imports Teams caption behavior directly, and it publishes ACTIVE before completing the silent admission re-check.

## Platform-specific adapters

Each platform adapter should own only behavior that depends on the meeting product:

- URL shape and platform validation.
- Pre-join UI and DOM selectors.
- Anonymous name entry.
- Authenticated join variants.
- Microphone and camera UI state.
- Waiting-room, admission, rejection, removal, and meeting-ended detection.
- Platform-specific methods of forcing sender creation or camera renegotiation.
- Leave actions.
- Compatibility fallbacks for platform-specific media behavior.

Keep Google Meet, Teams, and Zoom Web UI selectors isolated from LemonSlice/Daily/LiveKit provider code.

## Intended architecture

```text
LemonSlice Session Controller
        │
        ▼
One isolated worker container per meeting
        │
        ├── Shared worker lifecycle
        ├── GoogleMeetAdapter | TeamsAdapter | ZoomWebAdapter
        ├── MeetingAudioSource
        ├── MeetingTrackPublisher
        └── AgentMediaProvider
                ├── DailyProvider — initial
                └── LiveKitProvider — later
```

## Vexa capabilities to reuse

Retain or adapt:

- Chromium and Playwright meeting automation.
- Google Meet, Teams, and Zoom Web joining.
- Humanized Google Meet interaction.
- Waiting-room and admission handling.
- Authenticated browser profiles.
- Shared lifecycle orchestration.
- Removal and meeting-end detection.
- Browser WebRTC interception and track replacement.
- Container isolation and cleanup.
- Redis control channel, callbacks, logs, screenshots, and resource diagnostics.
- VNC/CDP only as tightly controlled escalation tooling.

## Product areas that are not part of the LemonSlice worker

These are not core product responsibilities:

- Vexa's hosted SaaS API and control plane.
- Whisper transcription and transcript persistence.
- Vexa's built-in TTS service.
- Recording uploads and storage.
- Dashboard.
- MCP server.
- Generic agent runtime.
- Billing, admin, and SaaS product logic.
- Native Zoom SDK unless deliberately chosen later.

These areas should be disabled before deletion. In particular, platform files named `recording.ts` also contain participant monitoring, alone-time timeouts, meeting-end detection, and long-running lifecycle behavior; they cannot be deleted until those responsibilities are extracted.

## Important boundary correction

Vexa's current generic external-webpage bridge is a useful proof of concept, not the final production media boundary. It screen-captures a second browser for video and uses an OS audio loop for outgoing agent audio. The production boundary should expose exact LemonSlice `MediaStreamTrack` objects behind a provider interface.
