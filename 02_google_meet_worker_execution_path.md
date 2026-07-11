# Google Meet Worker Execution Path

> **Repository baseline:** `tecxbro/vexa` → `main` at commit `90e5c726a75ea7ba349946648e6cf7f4e1cc845a`  
> **Source rule:** The audited findings override conflicting or inferred DeepWiki statements.  
> **Preservation note:** The original DeepWiki Q&A and audit files are retained in the external source pack and are intentionally not committed here.


Q2 and Q3 asked the same question. They are merged here; both original answers remain separately in the external source pack.

## Ordered execution path

```text
Docker CMD
  → /app/entrypoint.sh
  → node dist/docker.js
  → docker.ts main()
  → parse and validate BOT_CONFIG
  → runBot(botConfig)
  → create Redis control subscriber
  → restore profile when authenticated
  → launch Chromium context
  → inject media/browser scripts
  → initialize worker services
  → select Google Meet handler
  → handleGoogleMeet()
  → runMeetingFlow()
  → joinGoogleMeeting()
  → waitForGoogleMeetingAdmission()
  → silent admission verification
  → post-admission media/chat/monitoring hooks
```

## 1. Container startup

The bot image runs `/app/entrypoint.sh`. The entrypoint:

1. Adds Zoom native SDK library paths when those files exist.
2. Starts Xvfb on display `:99` at `1920x1080x24`.
3. Starts PulseAudio with retry and a system-mode fallback.
4. Creates software audio devices:
   - `zoom_sink`.
   - `tts_sink`.
   - `virtual_mic`, a remapped source backed by `tts_sink.monitor`.
5. Sets `virtual_mic` as the default PulseAudio source.
6. Mutes `tts_sink` and `virtual_mic` at startup.
7. Writes `/root/.asoundrc` so ALSA uses PulseAudio.
8. Verifies or regenerates `/app/vexa-bot/core/dist/browser-utils.global.js`.
9. Reads `mode` from `BOT_CONFIG` with an inline Node command.
10. Starts Fluxbox, x11vnc on `5900`, and noVNC/websockify on `6080`.
11. Starts a socat relay from container port `9223` to Chromium CDP port `9222`.
12. Exports `DISPLAY=:99` in meeting mode.
13. Runs `node dist/docker.js` and exits with its code.

No Python process is the main worker entrypoint. Python is launched later only when the external webpage streamer is enabled.

## 2. Configuration entrypoint and validation

`services/vexa-bot/core/src/docker.ts` is compiled to `dist/docker.js`.

Its `main()` function:

- Requires `process.env.BOT_CONFIG`.
- Parses the JSON.
- Normalizes snake_case aliases to camelCase.
- Selects `browser_session` or `meeting` mode.
- Validates meeting mode with Zod `BotConfigSchema`.
- Calls `runBot(botConfig)`.
- Reports startup validation failures and exits non-zero.

Relevant fields include:

- `platform`: `google_meet`, `teams`, or `zoom`.
- `meetingUrl`.
- `botName`.
- `redisUrl`.
- `meeting_id`.
- `meetingApiCallbackUrl` and `internalSecret`.
- `authenticated` and profile-storage settings.
- `automaticLeave.waitingRoomTimeout`, `noOneJoinedTimeout`, and `everyoneLeftTimeout`.
- `voiceAgentEnabled` and `voiceAgentSettings.url`.
- `cameraEnabled` and `defaultAvatarUrl`.

### Contract problems confirmed by the audit

- `meeting_id` is optional in the Zod schema, but `runBot()` exits with code 2 when it is absent.
- `meetingUrl` is checked as a syntactically valid URL, not validated against the selected platform's domain.
- `voiceAgentSettings.url` can be present while `voiceAgentEnabled` is false or omitted.
- Bridge URLs are not restricted to HTTPS or an allowlisted origin.
- Authenticated mode lacks strong cross-field validation for profile backend selection and required user identifiers.

These invariants belong in schema validation.

## 3. `runBot()`

Unlike the original DeepWiki uncertainty note, the full `runBot()` body was available on audited `main`. It:

- Starts or coordinates CDP relay behavior.
- Requires `meeting_id`.
- Creates the Redis subscriber and command channel.
- Restores an authenticated profile when requested.
- Launches an anonymous browser or persistent authenticated context.
- Adds browser init scripts before page creation/navigation.
- Initializes camera, chat, TTS/microphone, diagnostics, and optional external voice-agent services.
- Dispatches to Google Meet, Teams, or Zoom.

The original Q2/Q3 call graph was directionally correct, but its inferred `runBot()` sequencing should be considered superseded by this verified description.

## 4. Browser creation

### Anonymous mode

The anonymous meeting browser is headful and typically launched with:

- `--incognito`.
- `--no-sandbox` and `--disable-setuid-sandbox`.
- `--autoplay-policy=no-user-gesture-required`.
- `--use-fake-ui-for-media-stream`.
- `--use-file-for-fake-video-capture=/dev/null`.
- `--disable-blink-features=AutomationControlled`.
- CPU/GPU and site-isolation tuning.
- CDP arguments opening port `9222`.

`playwright-extra` uses `puppeteer-extra-plugin-stealth`.

### Authenticated mode

Authenticated mode uses `chromium.launchPersistentContext()` with `/tmp/browser-data`. The profile may be restored from S3 or an HTTP cookie/profile service. Stale `SingletonLock`, `SingletonCookie`, and `SingletonSocket` files are removed first.

The authenticated argument list is intentionally smaller to reduce Google automation detection.

### Authenticated CDP gap

Anonymous launch arguments include the CDP port arguments. Authenticated launch arguments do not. The socat relay may therefore wait without a browser CDP target in authenticated mode. CDP should be disabled by default and added explicitly only when secure escalation is enabled.

## 5. Pre-navigation scripts and runtime-loaded dependencies

Important runtime-loaded pieces include:

- `getVirtualCameraInitScript()` via `page.addInitScript()` before the meeting page is created or navigated. It patches media APIs and tracks peer connections.
- `browser-utils.global.js` loaded from the filesystem with `page.addScriptTag`, Trusted Types, or Blob URL fallback.
- Dynamic import of `browser-session.ts` for browser-session mode.
- Dynamic import of `startPerSpeakerAudioCapture` from `index.ts` after admission.
- AWS CLI commands for S3 profile synchronization.
- Shell tools such as `pactl`, `pulseaudio`, `x11vnc`, `websockify`, `socat`, `xdotool`, and `xclip`.
- Python/GStreamer/aiortc only when the external webpage streamer starts.

`--use-fake-ui-for-media-stream` grants media permissions automatically; it does not bypass JavaScript media API patches.

## 6. Platform selection

`runBot()` dispatches to `handleGoogleMeet`, `handleMicrosoftTeams`, or `handleZoom` from `botConfig.platform`.

`handleGoogleMeet()` creates a `PlatformStrategies` object with:

- `joinGoogleMeeting`.
- `waitForGoogleMeetingAdmission`.
- `checkForGoogleAdmissionSilent`.
- Google Meet preparation/monitoring/removal/leave functions.

It passes those strategies to the shared `runMeetingFlow()`.

## 7. Shared flow before admission

`runMeetingFlow()`:

1. Rejects a missing meeting URL.
2. Calls the platform join strategy.
3. Checks for a pre-admission stop signal.
4. Runs admission waiting and preparation in parallel.
5. Maps rejection, never-admitted, and timeout outcomes.
6. Attempts a browser-side stateless leave action when admission fails.

The shared module is not perfectly platform-neutral because it directly imports Teams captions later in the active phase.

## 8. Google Meet navigation and UI interaction

`joinGoogleMeeting()`:

- Navigates to the configured URL with `waitUntil: "domcontentloaded"`.
- Brings the page to the foreground.
- Takes a diagnostic screenshot.
- Sends a JOINING callback; failure is propagated.
- Selects humanized or synthetic UI interaction.

Google Meet defaults to humanized interaction. `HumanizedInteractor` uses OS-level XTEST through `xdotool`, with motion-capture cursor paths, because synthetic Playwright/CDP clicks can be rejected by input-authenticity checks. It falls back to synthetic interaction when X11/xdotool is unavailable.

## 9. Anonymous join

The anonymous flow:

- Waits for one of several locale-agnostic or English fallback text-input selectors.
- Fills `botName`.
- Mutes the microphone only when `voiceAgentEnabled` is false.
- Leaves the microphone enabled for the external voice-agent path.
- Turns the pre-join camera UI off; the virtual/outgoing camera track is handled separately.
- Finds a locale-agnostic join button or an English `Ask to join`/`Join now` fallback.
- Clicks through the humanized layer when available.
- Captures screenshots at important checkpoints.

## 10. Authenticated join

The authenticated flow:

- Skips name entry and uses the Google account identity.
- Waits for the SPA lobby to settle.
- Applies the same voice-agent microphone rule.
- Turns the lobby camera off.
- Races `Join now`, `Switch here`, and `Ask to join`.
- Uses `Join now` for normal authenticated entry.
- Uses `Switch here` when the same account is already in the call.
- Falls back to anonymous-style name entry when only `Ask to join` appears, indicating profile/cookie restoration did not produce the expected authenticated state.
- Screenshots and fails when no join action is found.

## 11. Waiting room, admission, rejection, and timeout

Waiting-room detection uses visible text, aria labels, and `Return to home screen`. The latter was corrected from a rejection indicator to a waiting indicator because it also appears while the request is pending.

Admission detection:

- First suppresses positives when waiting-room indicators remain visible.
- Suppresses admission when the Gemini note-taking consent gate is present.
- Moves the pointer to wake auto-hidden controls.
- Uses structural DOM presence such as participant/self markers and visible in-meeting controls.
- Avoids a media-element-only signal because the lobby has self-preview media.

A reCAPTCHA frame is treated as a recoverable blocking state, not immediate rejection.

Explicit denial text produces a host-denial outcome. With no explicit denial, a wait near Google's roughly ten-minute lobby timeout is classified as never admitted. Other waiting timeouts map to admission timeout.

## 12. Immediate post-admission behavior

After admission, the current flow:

- Waits one second so AWAITING_ADMISSION can be processed.
- Sends the ACTIVE/startup callback.
- Performs a silent admission re-check.
- Triggers external voice-agent audio readiness.
- Recovers/reinjects the camera track because lobby-to-meeting renegotiation can replace senders.
- Starts the chat observer.
- Starts optional per-speaker capture.
- Enables Teams captions inside the shared flow when the platform is Teams.
- Enters browser fullscreen.
- Starts the platform's long-running monitoring/recording function and removal monitor.

### Lifecycle correction

ACTIVE is currently sent **before** the silent admission verification. A false positive can therefore be briefly published as active. Verification should precede the ACTIVE callback.

## Important environment variables and ports

| Item | Purpose |
|---|---|
| `BOT_CONFIG` | Complete JSON worker configuration. |
| `DISPLAY=:99` | Main meeting browser X display. |
| `PULSE_SERVER`, `XDG_RUNTIME_DIR`, `PULSE_RUNTIME_PATH` | PulseAudio runtime. |
| `VEXA_CDP_PORT` | Chromium CDP port, default `9222`. |
| `ZOOM_SDK`, `ZOOM_WEB` | Select native Zoom SDK or Zoom Web. |
| `HOSTNAME` | Container identity in diagnostics/callbacks. |
| `5900` | Raw VNC. |
| `6080` | noVNC/websocket. |
| `9222` | Browser CDP target when enabled. |
| `9223` | socat-exposed CDP relay. |
| `8124` | Python external-page bridge API when started. |

## What the join path must not accidentally remove

Even when transcription and recording are disabled, the platform `recording.ts` functions may still own participant observation, startup-alone timeout, everyone-left timeout, removal cross-checking, and the promise that keeps the worker active. Extract monitoring before deleting capture code.
