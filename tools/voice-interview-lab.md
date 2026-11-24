# Voice Interview Lab

Build a local voice agent that answers an AI interviewer with Justin's facts and his ElevenLabs voice clone.

## Scope

The target is an AI interviewer on a web page.
Start with a controlled test page before connecting another interview website.
The local project is `/Users/iamsegbedji/work/projects/voice-interview-lab`.

Use TypeScript for the server and browser code.
Use Bun for the runtime, package manager, and browser build.
Use Hono for server routes and HTTP request logging.
Use oxfmt for formatting and oxlint for lint checks.
Use Eleven v4 Turbo (`eleven_v4_turbo`) through the Text to Dialogue WebSocket.
Use 24 kHz, 16-bit, mono PCM audio for immediate chunk playback.
Follow the shared [code conventions](../engineering/code-conventions.md) and [engineering workflow](../engineering/workflow.md).

## Voice architecture

OpenAI Realtime receives audio and produces response text.
ElevenLabs converts that text into speech with Justin's existing voice clone.
The application controls audio playback, interruptions, and the stop control.

```mermaid
flowchart LR
  Tab[Interview tab audio] --> Capture[Browser audio capture]
  Capture --> OpenAI[OpenAI Realtime]
  Facts[Local reference files] --> OpenAI
  OpenAI --> Text[Response text]
  Text --> ElevenLabs[ElevenLabs voice clone]
  ElevenLabs --> Mic[Virtual microphone]
  Mic --> Tab
```

OpenAI does not accept an ElevenLabs voice ID as its native output voice.
Realtime supports audio input with text output through `output_modalities: ["text"]`.
The response text can then enter ElevenLabs text-to-speech.
See [Realtime conversations](https://developers.openai.com/api/docs/guides/realtime-conversations) and [ElevenLabs speech streaming](https://elevenlabs.io/docs/api-reference/text-to-speech/stream).

OpenAI also supports its own custom voices from recordings.
That feature requires eligible account access and separate consent and sample recordings.
An existing ElevenLabs clone cannot transfer through its voice ID.
See [OpenAI custom voices](https://developers.openai.com/api/docs/guides/custom-voices#create-a-voice-from-a-recording).

## Test sequence

1. Connect an AI interviewer and the candidate agent inside the local test page.
2. Pass actual audio between the agents.
3. Check the transcript against the reference files.
4. Measure the delay before candidate speech starts.
5. Check interruption and stop behavior.
6. Open the separate interviewer page.
7. Capture that tab's audio and return candidate speech through a virtual microphone.
8. Test the selected external interview website.

The direct test does not prove compatibility with another website.
It avoids virtual microphone setup during the first conversation check.
The separate-tab test checks the browser audio path.

For the return path on macOS, evaluate BlackHole as a virtual microphone.
The interview page selects that device as its microphone input.
Keep interview audio on a separate output to prevent feedback.
See [BlackHole virtual microphone setup](https://github.com/ExistentialAudio/BlackHole/wiki/Use-BlackHole-as-a-Virtual-Microphone).

## Reference material

Read reference files from their existing locations.
Do not copy the complete application folder or blog archive into the project.

| Source | Purpose |
| --- | --- |
| `~/Desktop/applications/PROFILE.md` | Career facts and source routing |
| `~/Desktop/applications/SATQUIZ-HISTORY.md` | Dated SatQuiz evidence and claim limits |
| `~/Desktop/applications/FABRIC-HISTORY.md` | Fabric work evidence |
| `~/Desktop/applications/RECENT-ENGINEERING-EVIDENCE.md` | Recent work evidence and verification limits |
| `~/work/projects/segbedji/src/lib/data/about.md` | Writing style |
| `~/work/projects/segbedji/src/lib/data/bio.md` | Writing style |

For the prototype, include this small source set in the candidate instructions.
Use dated evidence when sources disagree.
Use website text for style, rather than current factual claims.
State uncertainty when the sources do not support an answer.
Prompt instructions alone do not guarantee factual accuracy.
Check generated claims against the source evidence during live tests.

## Local operation

The project README owns the minimum start commands.
The application binds to the loopback interface.
Keep API keys on the server.
Set provider keys and `ELEVENLABS_VOICE_ID` through the project environment.
Model names are constants in `src/shared/voice-config.ts`.
OpenAI uses `gpt-realtime-2.1`; ElevenLabs uses `eleven_v4_turbo`.
The old `OPENAI_REALTIME_MODEL` and `ELEVENLABS_TTS_MODEL` environment variables have no effect.
Bun loads `.env` when the server starts.
Agents must not open that secret file.
Keys entered through the local setup form stay in server memory.
Do not save them in browser storage or source files.
The environment variable template lists the required settings.
A direct voice ID needs only Text to Speech access on the ElevenLabs key.
Listing account voices also needs Voices read access.

Starting a candidate session sends the selected reference text to OpenAI.
Candidate response text goes to ElevenLabs for speech generation.
Do not send page video to either provider.
Conversation transcripts stay in browser memory unless the user exports them.
Live tests use paid provider APIs.

## Server logs

Hono's built-in logger records HTTP requests and their duration.
Its print function adds the request ID and removes query strings.
The server prints one line for each voice activity event.
Voice activity lines include a UTC timestamp, severity, and event name.

- `requestId` links an HTTP request to its provider request and speech stream.
- `callId` links browser state events to the candidate or interviewer.
- Duration fields use milliseconds.
- Provider failures include status, request ID, and available error identifiers.

Session logs show creation, connection state, speech detection, reply generation, and stop requests.
Speech logs show generation, the first audio bytes, total bytes, completion, and cancellation.
Browser playback logs show scheduled audio and interruptions.
Startup and speech-generation logs identify the active ElevenLabs model.
Successful static-file requests and status checks do not produce routine log lines.

The browser sends selected state events to the server.
This is necessary because Realtime media travels directly between the browser and OpenAI.
Diagnostic delivery failures do not stop the conversation.

Logs exclude API keys, authorization headers, SDP, reference text, reply text, and audio contents.
Only configured-key indicators and content sizes are logged.
Provider messages are excluded because they can contain request values.

Restart the server after a server-code change.
Refresh the page after the browser build changes.

## Current state — 24 November 2026

The first Bun and TypeScript implementation is complete.
The page supports a direct conversation test, a separate interviewer, and tab audio capture.
It accepts a direct ElevenLabs voice ID from the environment or page input.
It includes transcript export, speech cancellation, and a stop control.
Formatting uses oxfmt. Lint checks use oxlint.

Justin added the provider keys and voice ID through `.env`.
The agent did not open that file or print the keys.
The OpenAI account model list includes the configured Realtime model.

The type check, lint check, and browser build passed.
The local page and compiled browser script returned HTTP 200.
Requests for `.env` and server source returned HTTP 404.
A cross-origin key update returned HTTP 403.

The live conversation attempt returned HTTP 400 during OpenAI session setup.
The request now sends SDP and session configuration as string form fields.
Justin reports that the application appears to work after the server restart.
A live Browse check completed an interviewer question, a cloned answer, and the next question.
An external interview website has not been checked.

Server activity logs now include browser state events and speech-stream timing.
Manual checks use simulated provider responses for failures and stream cancellation.
No unit test files are retained.

The silent-audio fault was reproduced in Chrome with Browse.
The browser received RTP packets but produced no decoded samples for Web Audio.
Each remote audio stream now has a muted media element that starts the decoder.
Web Audio then handles playback and audio routing.
Session stop removes these media elements.
See [Chromium issue 40094084](https://issues.chromium.org/issues/40094084).

After this change, the candidate received the interview question and generated an answer.
Cloned speech was decoded and scheduled for playback.
The return stream now includes a continuous silent source between speech buffers.
This keeps the capture clock active so turn detection can identify the end of an answer.
Session stop treats an already-closed provider call as a successful stop.

The speech default was updated to Eleven v4 at Justin's request.
A direct API check with his configured clone returned HTTP 200 and non-empty audio.

## Speech delivery

The candidate prompt uses short clauses, contractions, and a short first sentence.
A brief hesitation can fit a question that requires reflection.
It stays attached to the answer clause and occurs at most once per answer.
Simple factual answers do not require a hesitation.

The speech request adds a conversational, unhurried delivery tag for v4.
The delivery tag is separate from the displayed transcript.
The dialogue WebSocket uses voice stability 0.4.
It does not send the HTTP endpoint's similarity setting.
These settings are a starting point for listening checks.
Eleven v4 does not support the speed parameter.
See [voice settings](https://github.com/elevenlabs/skills/blob/main/text-to-speech/references/voice-settings.md) and [audio tags](https://elevenlabs.io/blog/elevenlabs-audio-tags-list).

## Delay reduction

One v4 reply started audio about seven seconds after turn detection.
Another reply took about 3.6 seconds, based on server log timestamps.
These are individual observations with different answers.
They are not a model benchmark.
Browser log delivery can add about 150 milliseconds to event receipt times.

For the second observed reply, OpenAI produced its first text after about 0.84 seconds.
ElevenLabs returned its first audio bytes about 0.80 seconds after the speech request.
The complete first-sentence audio arrived after about 2.50 seconds.
The previous browser code waited for that complete audio file before decoding and scheduling it.

| Priority | Proposed change | Expected effect and tradeoff |
| --- | --- | --- |
| 1 | Use a persistent v4 Turbo Text to Dialogue WebSocket and play PCM chunks as they arrive. | Removes whole-file waiting and repeated sentence connections. Requires new buffering and cancellation logic. |
| 2 | Stream a short first clause, then continue the answer. | Starts useful speech earlier. Chunk boundaries must preserve natural delivery. |
| 3 | Change semantic turn detection from low to medium eagerness. | Reduces waiting after a question. Check that short pauses do not cause interruptions. |
| 4 | Use a compact factual summary with detailed evidence available when needed. | May reduce model work. Check factual coverage before removing source material. |

Justin approved priorities 1 through 3; all three are implemented.
Priority 4 remains a proposal.
The implementation keeps one provider connection open between replies.
A short opening segment enters synthesis after at least eight complete words and 40 characters.
The rest of the answer follows through the same connection.
The final text fragment is flushed when OpenAI finishes.
PCM chunks are scheduled as they arrive, without MP3 decoding or whole-file waiting.
The first streaming version used medium eagerness for both agents.
The later latency pass uses high for the candidate and medium for the interviewer.
Keep the existing voice clone and compare normal speech samples by listening.
Measure the first meaningful answer, not a filler used to occupy the delay.
Hono changes routing structure. It does not remove speech-generation or playback buffering delays.

See [v4 Turbo streaming](https://elevenlabs.io/docs/eleven-api/guides/how-to/websockets/realtime-tdd).
The server creates the speech WebSocket only for an active candidate call from the local app origin.
The provider key stays in the server's first authentication frame.
A keep-alive frame runs every 10 seconds.
Stop closes both sockets and stops queued audio.
The provider protocol has no cancel command, so an interruption replaces the socket if generation is still active.
Turn IDs and connection generations discard stale audio from interrupted replies.
Each turn is limited to 6,000 text characters; a speech session is limited to six minutes.
The page stops a test after five minutes.

`src/client` owns page controls, WebRTC, and PCM playback.
`src/server` owns Hono routes, the runtime, session prompts, provider speech, and logs.
`src/shared` owns message types and model constants.
Static assets remain in `public`; Bun build scripts remain in `scripts`.

Use `flush: true` without an empty input to finish a dialogue segment.
The provider rejects an empty input with `input_text_empty`.
A flush produces `is_final_audio_for_turn` while leaving the connection open.
See the [dialogue WebSocket API](https://elevenlabs.io/docs/api-reference/text-to-dialogue/ttd-websocket).

### Streaming check — 24 November 2026

The type check, lint check, format command, and browser build passed after the source folders moved.
A live Browse check on port 4317 completed two cloned replies through the same provider connection.
First scheduled reply audio arrived 2.78 seconds and 1.86 seconds after candidate turn detection.
OpenAI first text took 2.42 seconds and 1.46 seconds in those replies.
The first provider audio chunk took 238 milliseconds after the first text segment in the first reply.
These are individual observations, not a controlled model benchmark.

The first reply scheduled 49 PCM chunks with non-zero sample values.
Its audio started before OpenAI finished the response text.
The interviewer received the audio and asked its next question.
No page error appeared during the two replies.

A controlled browser interruption injected a speech-start event during the next reply.
The candidate response was cancelled, queued playback stopped, and the pending provider stream was discarded.
The provider socket reopened in 452 milliseconds.
A new OpenAI response then produced cloned speech through the replacement connection.
No stale PCM chunk was scheduled between cancellation and the new reply's first audio.
This checks the cancellation path with live providers; it does not prove microphone interruption timing on another website.

Stop closed both WebRTC peers, the speech socket, and the audio context.
It removed both hidden audio renderers, and the server returned to zero active calls.
The rebuilt page loaded with the Turbo model and no visible error.

The local page, interviewer page, and browser bundle returned HTTP 200.
Requests for the secret file and server source returned HTTP 404.
The speech endpoint rejected a foreign origin and a missing candidate session with HTTP 403.
No unit tests were added.

### Reasoning and turn detection — 24 November 2026

The response model remains `gpt-realtime-2.1` and the clone uses `eleven_v4_turbo`.
Reasoning effort is now explicitly `minimal`.
Candidate semantic VAD uses `high`; the interviewer remains at `medium`.
All settings are constants in `src/shared/voice-config.ts`.
Startup and session-creation logs show reasoning effort and turn eagerness.
See [Realtime reasoning effort](https://developers.openai.com/api/docs/guides/voice-prompting?voice-api=realtime) and [turn detection](https://developers.openai.com/api/docs/guides/realtime-vad).

The original timing started after the speech-stop event, which excluded turn detection.
The page now estimates the full gap from the end of input audio to the first audible reply sample.
A small AudioWorklet measures input activity on the audio thread, without a background-tab polling timer.
It measures 10 ms windows and ignores levels below 0.005 full scale.
This is an audio-level estimate, not a speech recognizer or a sound-card loopback measurement.
The output measurement skips leading PCM samples below the same threshold.

`realtime.response.latency` logs these browser-clock durations in milliseconds:

- `audioGapMs`: estimated input-audio end to the scheduled audible reply sample.
- `turnDetectionMs`: estimated input-audio end to the Realtime speech-stop event.
- `modelMs`: speech-stop event to first non-empty response text.
- `speechMs`: first response text to the scheduled audible reply sample.
- `afterDetectionMs`: speech-stop event to that reply sample.

If input audio continues after turn detection, the full-gap estimate is omitted and `inputContinuedMs` is logged.
If the audio meter cannot load, the conversation continues with timing after turn detection only.
Transcript exports include timing values and the configured response settings.

A manual Browse comparison replayed the same recorded question with each setting.
A second version inserted a 1.2-second pause in the question.
The first comparison used an analyser-based prototype of the meter, before it moved to the audio thread.

| Reasoning | Candidate VAD | Normal question gap | Paused question gap | First text after detection, normal / paused |
| --- | --- | --- | --- | --- |
| low | medium | 4.09 s | 3.61 s | 1.43 s / 0.97 s |
| minimal | medium | 5.62 s | 2.32 s | 0.54 s / 0.98 s |
| minimal | high | 2.20 s | 2.10 s | 0.57 s / 0.64 s |

No setting replied before either replayed question ended.
These are six observations in one session with changing conversation history, not a controlled benchmark.
Turn-detection variation was large enough to outweigh faster text in one minimal/medium observation.
The high setting is selected for the AI interviewer use case, rather than a claim that every question will take two seconds.

The final AudioWorklet version also completed a longer Stripe retry-safety question with live providers.
Its estimated gap was 4.60 s: turn detection 2.55 s, first text 1.62 s, and speech 0.43 s.
The response matched the recorded event-ledger, operation-receipt, and uncertain-outcome recovery facts.
A follow-up asking for an exact incident-reduction percentage correctly stated that no before-and-after metric was recorded.
The prompt now explicitly forbids inferring specific test runs from a feature's release.
These checks do not establish general interview-answer accuracy.

A stop check found that an already-closed OpenAI call can return HTTP 404 without a structured error code.
The known-call hangup path now treats any HTTP 404 as already stopped.
The repeated check stopped both peers and the speech socket without a page error.
No unit tests were added.

## Two-tab live check

Open `http://localhost:4317/` and choose **Connect an interview tab**.
Open `http://localhost:4317/interviewer` in a second Chrome tab as the audio source.
Its title is **AI interviewer · Voice Interview Lab** so it is distinct in Chrome's tab picker.
The source page uses OpenAI's interviewer voice and listens to its selected microphone.
Both pages use the existing local server and server-side provider keys.
Leave both pages stopped until audio routing is selected.

BlackHole is the virtual audio cable for the return path on macOS.
Set the app's **Send your voice to** output to BlackHole, and the interviewer's microphone to BlackHole.
Keep interviewer playback on headphones.
When connecting the app, select the interviewer tab in Chrome's share picker and enable tab audio.
Chrome requires the user to choose the tab in its picker.

On this Mac, the 24 November audio-device check did not find BlackHole.
Homebrew's `blackhole-2ch` package reports that a restart is required after installation.
When ready to install it, use `brew install --cask blackhole-2ch` and follow the installer prompts.
Do not restart the Mac or its audio service during an active conversation without Justin's approval.
See the [BlackHole installation and routing guide](https://github.com/ExistentialAudio/BlackHole).
The full two-tab capture and virtual-microphone route is still unverified.

To take over manually with the current app, press **Stop**, then select the real microphone on the interview page.
A one-click takeover that routes the real microphone through BlackHole has been discussed but is not implemented.

Use Browse with the `justin-okupter` profile for further browser tests.
Use Justin's existing server on port 4317.
The separate agent-browser session and extra test server were stopped.

## Open checks

- Compare the revised speech delivery by listening to full answers.
- Collect more timing samples across different questions and check for early replies during long pauses.
- Check interruptions across both providers.
- Check what the model remembers after interrupted speech.
- Check microphone routing on the target website.
