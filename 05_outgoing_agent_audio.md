# Outgoing Agent Audio: LemonSlice → Meeting

> **Repository baseline:** `tecxbro/vexa` → `main` at commit `90e5c726a75ea7ba349946648e6cf7f4e1cc845a`  
> **Source rule:** The audited findings override conflicting or inferred DeepWiki statements.  
> **Preservation note:** The original DeepWiki Q&A and audit files are retained in the external source pack and are intentionally not committed here.


## Correct current external-page path

```text
LemonSlice/Daily page audio output
  → second Chromium process with PULSE_SINK=webpage_streamer_sink
  → webpage_streamer_sink.monitor
  → PulseAudio module-loopback into tts_sink
  → tts_sink.monitor
  → virtual_mic remap source
  → default Chromium getUserMedia microphone
  → Google Meet / Teams / Zoom Web outgoing audio sender
```

`VoiceAgentPageService.startAudioBridge()` creates the loopback from `webpage_streamer_sink.monitor` into `tts_sink`. The container creates `virtual_mic` from `tts_sink.monitor` and makes it the default source.

This path carries all audio produced by the external LemonSlice page. It is not limited to Vexa's built-in TTS.

## Voice-agent microphone state

For Google Meet, authenticated and anonymous join flows leave the meeting microphone enabled when:

```ts
botConfig.voiceAgentEnabled === true
```

The original DeepWiki conclusion that the external-page audio path is inherently broken because the UI microphone remains muted is superseded.

### Configuration inconsistency

The schema permits `voiceAgentSettings.url` with `voiceAgentEnabled` false or absent. In that inconsistent state, the join adapter may mute the UI microphone even though a bridge page is configured. Validation should enforce that a provider URL implies voice-agent mode and audio publishing.

## Built-in Vexa TTS path versus LemonSlice path

Vexa's built-in TTS can write audio directly into the same `tts_sink`/`virtual_mic` chain. The LemonSlice page enters earlier through `webpage_streamer_sink.monitor` and a PulseAudio loopback.

The shared downstream device graph is:

```text
tts_sink → tts_sink.monitor → virtual_mic → meeting browser microphone
```

The LemonSlice product does not need Vexa's TTS generation service, but it temporarily depends on the device-routing portion of the TTS infrastructure.

## Isolation model

One container per meeting gives each worker its own PulseAudio daemon and sink names, avoiding cross-meeting audio mixing. This isolation assumption must remain true in deployment.

## Echo and interruption behavior

- Meeting audio is delivered to the LemonSlice page over the separate incoming WebRTC path.
- Agent audio is emitted from the page into PulseAudio.
- Meeting-audio discovery attempts to exclude known bot-output streams.
- Barge-in behavior still belongs to LemonSlice. The worker should publish or stop tracks based on LemonSlice's output rather than implement its own conversational logic.

## Current weaknesses

- OS-level loopback adds latency and failure modes.
- The meeting UI microphone remains continuously open in voice-agent mode.
- PulseAudio sink/source mute state and module lifecycle can drift.
- Audio routing is coupled to a second Chromium process.
- The path relies on the meeting browser choosing the intended default source.
- Media health is harder to prove than with RTP sender stats.

## Production target

```text
LemonSlice remote audio MediaStreamTrack
  → meeting RTCRtpSender.replaceTrack()
```

Create a silent placeholder audio track before joining so the meeting platform creates an outgoing audio sender. When the LemonSlice track is available, replace the placeholder. Monitor sender stats and reapply after renegotiation.

This removes the live-agent dependency on:

- `webpage_streamer_sink`.
- `tts_sink`.
- `virtual_mic`.
- PulseAudio loopback timing.
- An always-open browser microphone.

PulseAudio can remain temporarily for built-in Vexa TTS or as a compatibility fallback while direct track forwarding is validated.

## Provider seam

```ts
getAgentAudioTrack(): Promise<MediaStreamTrack>
```

The Daily provider returns the exact subscribed LemonSlice audio track. The platform-neutral publisher owns sender discovery and replacement; platform adapters own any UI action needed to create or renegotiate that sender.
