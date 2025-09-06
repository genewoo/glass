# macOS System Audio Capture with `SystemAudioDump`

Glass uses a small native helper, [`SystemAudioDump`](../src/ui/assets/SystemAudioDump), to stream macOS system audio into the Speech‑to‑Text (STT) pipeline. The helper taps the CoreAudio output device and writes raw PCM data to `stdout`, which the STT service ingests and forwards to the "Them" transcription session.

## Launch sequence

1. **Renderer request** – Starting capture in the UI triggers the IPC method `listen:startMacosSystemAudio`.
2. **Service preparation** – The [`SttService`](../src/features/listen/stt/sttService.js) kills any lingering helpers via `pkill -f SystemAudioDump` and resolves the path to the bundled binary at [`src/ui/assets/SystemAudioDump`](../src/ui/assets/SystemAudioDump) (or the unpacked equivalent in production).
3. **Spawning** – The helper is spawned with `stdio` wired so that the service can read PCM bytes from `stdout` and log diagnostics from `stderr`.

## Data flow

- The helper emits 24 kHz stereo, 16‑bit PCM. The service buffers this stream into 100 ms (`CHUNK_SIZE = 24000 * 2 bytes * 2 channels * 0.1s`) chunks.
- Each chunk is converted to mono by discarding the right channel, then base64‑encoded.
- The encoded audio is forwarded both to the renderer (`system-audio-data`) and to the active "Them" STT WebSocket session. Provider‑specific payloads are constructed as needed (e.g., Gemini expects `{ audio: { data, mimeType } }`, Deepgram receives raw buffers).

## Lifecycle management

- `stderr` output from `SystemAudioDump` is logged for troubleshooting.
- When the helper exits or errors, the service clears its reference and stops sending audio.
- Calling `stopMacOSAudioCapture` sends a `SIGTERM` to the helper, ensuring a clean shutdown.

## Integration notes

- On macOS, screen capture is obtained separately via `getDisplayMedia`; system audio never travels through the browser APIs.
- Windows relies on `getDisplayMedia({ audio: true })` for loopback capture, while Linux currently disables system audio capture.

[`SystemAudioDump`](../src/ui/assets/SystemAudioDump) enables reliable separation of microphone and system audio on macOS, letting Glass transcribe remote voices without echoing the user's own microphone input.
