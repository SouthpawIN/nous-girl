---
name: hermes-voice-debugging
description: Debug TTS silence, voice mode failures, and audio playback issues in Hermes Agent CLI and gateway modes.
---

# Hermes Voice Debugging

Diagnostic procedures for when TTS is enabled but users don't hear responses, or voice recording fails silently.

## Quick Check List

1. **Verify edge-tts works**:
   ```bash
   edge-tts --voice en-US-JennyNeural --text "Hello world" | mpv --no-video -
   ```
   If this doesn't play, the issue is system-level (audio server, mpv, permissions).

2. **Verify mpv works**:
   ```bash
   mpv /path/to/tts_file.mp3 --no-video
   ```
   Try with `--volume=50` flag matching what cli.py uses.

3. **Check if TTS tool generates files**:
   ```bash
   python3 -c "from tools.tts_tool import text_to_speech_tool; print(text_to_speech_tool('test'))"
   ```

## TTS Silence (User: "I can't hear anything")

### Symptom
- `voice.auto_tts: true` is set in config
- `text_to_speech_tool()` generates audio files successfully
- `mpv` plays files correctly when called manually
- CLI responses are generated but never spoken

### Root Cause 1: _cli_speak_response_async is orphaned
The function `_cli_speak_response_async(self, text)` at **cli.py ~line 8576** generates TTS audio and plays it via mpv — but has **zero callers** in the codebase. It was written but never wired into the response handler.

**Verify**: `grep -cn '_cli_speak_response_async(' cli.py` — should return 1 (only definition).

### Root Cause 2: MANDATORY TTS block swallows errors
The inline MANDATORY TTS at **cli.py ~line 9758** calls TTS + mpv for every response, but both stdout and stderr go to `subprocess.DEVNULL`. Errors become invisible.

**Debug**: Temporarily replace `subprocess.DEVNULL` with `subprocess.PIPE` and read stderr to spot failures.

### Root Cause 3: auto_tts config flag doesn't reach response handler
The `_cli_auto_tts` flag was added to the config loader but the response handler may not check it. The flow is:
- `_cli_auto_tts` set in `HermesCLI.__init__` from config
- `_voice_speak_response_async` checks both `_voice_tts` AND `_cli_auto_tts`
- But the main response handler at ~line 9705 may not call `_voice_speak_response_async` at all

**Fix checklist**:
1. Confirm `self._cli_auto_tts = True` in `__init__` by grepping for the config load
2. Verify `_voice_speak_response_async` gets called from the response handler
3. Check if `response` variable actually has content when TTS would trigger
4. Look for early returns or conditions that skip the TTS call

## Voice Recording Not Starting

### Symptom
- User says voice message but it's sent as text
- Recording indicator flashes once then disappears
- STT logs show "no audio captured"

### Common causes

1. **SoundDevice not installed or no audio devices**:
   ```bash
   python3 -c "import sounddevice as sd; print(sd.query_devices())"
   ```
   Fix: `pip install sounddevice` or check `pavucontrol` is running.

2. **Permissions issue**:
   - User not in `audio` group: `sudo usermod -a -G audio $USER`
   - PipeWire/PulseAudio not running: `systemctl --user status pipewire pulseaudio`

3. **STT provider not configured**:
   - Check `stt.enabled: true` in config
   - Verify provider (local requires `faster-whisper`, groq needs `GROQ_API_KEY`)

## Audio Platform Differences

### CLI mode (interactive terminal)
- Two paths: MANDATORY TTS block (~9758) and `_cli_speak_response_async` (~8576)
- Both spawn mpv as background process
- Errors go to `DEVNULL` → invisible failures
- Audio saved to `/tmp/hermes_voice/` or `$HERMES_HOME/cache/audio/`

### Gateway mode (Telegram, Discord, etc.)
- Audio delivered as `MEDIA:` tags in response text
- Gateway platform adapter intercepts and sends as voice message/attachment
- No mpv playback on gateway side — audio is uploaded to platform API
- Check platform-specific voice message support (Telegram yes, Discord no native voice)

## Diagnostic Template

When a user reports voice issues, follow this order:

1. **System level**: Does `mpv test.mp3 --no-video` work? If no → audio system issue.
2. **TTS generation**: Does `text_to_speech_tool("hello")` produce a file? If no → provider/key issue.
3. **CLI code path**: Is `_cli_auto_tts` being set? Check init logs. Is it being called? Check grep.
4. **Error visibility**: Replace `DEVNULL` with `PIPE` in MANDATORY TTS block temporarily.
5. **Gateway path**: Check that `voice_compatible` flag is True in TTS response.

## Fix: Wire Up _cli_speak_response_async (if desired)

To make `_cli_speak_response_async` the canonical TTS path, add to the response handler at ~line 9757:

```python
# After MANDATORY TTS block or as its replacement:
if self._cli_auto_tts and response and not use_streaming_tts:
    self._cli_speak_response_async(response)
```

Or, use `_voice_speak_response_async` instead if voice mode infrastructure is preferred.

### TUI mode (hermes --tui)
- Uses `tui_gateway/server.py` — completely separate from cli.py code paths
- `_voice_tts_enabled()` (line ~5488) was ONLY checking `HERMES_VOICE_TTS` env var — **never read `voice.auto_tts` from config.yaml**
- This made `voice.auto_tts: true` in config.yaml have ZERO effect in TUI mode
- **FIX PATCHED**: `_voice_tts_enabled()` now checks BOTH `HERMES_VOICE_TTS` env var AND `voice.auto_tts` from config via `_voice_cfg_dict()`
- After this patch, TUI behaves the same as CLI for auto_tts
- **Important**: the patch changes import-time config loading — must restart TUI for fix to take effect

## Diagnostic Template

When a user reports voice issues, follow this order:

1. **System level**: Does `mpv test.mp3 --no-video` work? If no → audio system issue.
2. **TTS generation**: Does `text_to_speech_tool("hello")` produce a file? If no → provider/key issue.
3. **CLI code path**: Is `_cli_auto_tts` being set? Check init logs. Is it being called? Check grep.
4. **TUI code path**: Is `_voice_tts_enabled()` reading config? If not, patch it (see TUI section above).
5. **Error visibility**: Replace `DEVNULL` with `PIPE` in MANDATORY TTS block temporarily.
6. **Gateway path**: Check that `voice_compatible` flag is True in TTS response.

## Fix: Wire Up _cli_speak_response_async (if desired)

Some users configure Hermes as voice-first: the SOUL.md says "always speak your responses", config has `voice.auto_tts: true`, and TTS is the primary output channel (headless, CLI, no terminal needed to see responses).

**For voice-first users:**
- Always verify audio playback works before assuming TTS succeeded
- Add mpv error visibility (PIPE instead of DEVNULL) during debugging
- Prefer `_cli_speak_response_async` as it's cleaner than inline MANDATORY TTS
- If TTS is broken, prioritize fixing it over other work — voice-first users may not even see text output
- Patch SOUL.md to document the auto-TTS fix when it's relevant  
- If the user says "fix your voice" or "make it speak", treat it as CRITICAL — not optional
