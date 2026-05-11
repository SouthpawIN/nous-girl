# TTS Silence — Session-Specific Diagnostics (2026-05-06)

## Environment
- Profile: `nous-girl` at `/home/sovthpaw/.hermes/profiles/nous-girl/`
- OS: Linux (user `sovthpaw`)
- Audio: pipewire (`ao: [pipewire] 24000Hz mono 1ch floatp`)
- mpv: `/usr/bin/mpv` — works with `--no-video` flag
- edge-tts: installed, generates audio successfully
- TTS provider: `edge` (voice: `en-US-JennyNeural`)

## Key Findings

### cli.py TTS Architecture (at time of investigation)

**3 separate TTS code paths exist in cli.py:**

1. **`_voice_speak_response_async`** (~line 8513) — voice-mode TTS, checks `_voice_tts` AND `_cli_auto_tts`. Has callers in the voice mode path (~line 8727).

2. **`_cli_speak_response_async`** (~line 8576) — standalone TTS, generates audio + pipes to mpv. Has **ZERO callers** in the codebase. It's dead/orphan code.

3. **MANDATORY TTS block** (~line 9758-9774) — inline TTS + mpv spawn for every response. Has `subprocess.DEVNULL` on both stdout/stderr. Silent failures on mpv.

### _cli_auto_tts Configuration

The `_cli_auto_tts` flag is read at init time in `HermesCLI.__init__`:
```python
_voice_cfg = (CLI_CONFIG.get("voice") or {}) if isinstance(CLI_CONFIG.get("voice"), dict) else {}
self._cli_auto_tts = bool(_voice_cfg.get("auto_tts", False))
```

However, the response handler at ~line 9758 does NOT check `_cli_auto_tts` — it runs TTS unconditionally if `response and not use_streaming_tts`. This means the MANDATORY TTS block runs for every response regardless of config, BUT it may be failing silently.

### TTS Response Object Structure

When `text_to_speech_tool()` succeeds, it returns:
```json
{
  "success": true,
  "file_path": "/path/to/tts_file.ogg",
  "provider": "edge",
  "voice_compatible": true
}
```

The MANDATORY TTS block parses this as JSON and extracts `file_path`. If the file doesn't exist at the returned path, mpv is never called.

### MANDATORY TTS Block Issues

The inline code at ~line 9758:
```python
if response and not use_streaming_tts:
    # ... generate audio ...
    try:
        tr = text_to_speech_tool(text=tts_text, output_path=audio_path)
        rj = json.loads(tr) if isinstance(tr, str) else tr
        ap = rj.get('file_path', audio_path)
        if os.path.isfile(ap):
            subprocess.Popen(['mpv', '--no-video', '--volume=50', ap],
                           stdout=subprocess.DEVNULL,
                           stderr=subprocess.DEVNULL)
    except Exception as e:
        logger.warning("Mandatory TTS failed: %s", e)
```

Potential silent failures:
1. TTS throws an exception → caught, logged to `agent.log` only, no terminal output
2. `json.loads(tr)` fails on non-JSON response → caught by `except JSONDecodeError`, file_path falls back to `audio_path` which may not exist
3. `os.path.isfile(ap)` is False → mpv never called, no warning shown
4. mpv throws `FileNotFoundError` or `OSError` → caught by except, logged to `agent.log` only

### Testing Results

- `edge-tts --voice en-US-JennyNeural --text "test" --write-media /tmp/test.mp3` → SUCCESS (26784 bytes)
- `mpv /tmp/test.mp3 --no-video` → SUCCESS via pipewire, exit 0
- `mpv /tmp/test.mp3 --no-video --volume=50` → SUCCESS, exit 0
- `mpv /tmp/test.mp3 --no-video --ao=pulse,alsa` → FAILS with `ao` option parsing error
- `text_to_speech_tool("test text")` → Returns `{success: true, file_path: ..., voice_compatible: true}`

## Recommended Fix

1. **Add visibility**: In the MANDATORY TTS block, add a terminal print before mpv spawn:
   ```python
   if os.path.isfile(ap):
       print(f"🔊 [TTS] Playing: {ap}")  # visible confirmation
       subprocess.Popen(['mpv', '--no-video', '--volume=50', ap],
                       stdout=subprocess.DEVNULL,
                       stderr=subprocess.DEVNULL)
   else:
       print(f"⚠️ [TTS] Audio file not found: {ap}")  # diagnose missing files
   ```

2. **Wire up `_cli_speak_response_async`**: Since it has the full TTS+mpv logic but zero callers, either:
   - Call it from the response handler instead of the MANDATORY TTS block
   - Delete it as dead code if the goal is to keep the inline MANDATORY TTS block

3. **Alternative**: Add a separate flag like `display.tts_always_playback` to explicitly enable CLI TTS playback, making the behavior opt-in rather than silent.

## Related Files

- `cli.py` — lines 8513-8575 (voice modes), 8576-8650 (_cli_speak_response_async), 9758-9774 (MANDATORY TTS)
- `tools/tts_tool.py` — TTS generation, output path resolution
- `config.yaml` — `voice.auto_tts: true`, `tts.provider: edge`
- `SOUL.md` — personality says "Always speak your responses"

## Session Date
2026-05-06