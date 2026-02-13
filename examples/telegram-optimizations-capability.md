---
capability_id: telegram-optimizations
version: 1.0
triggers:
  keywords: [telegram, inline, buttons, voice, audio, keyboard, chat, message, tts]
  session_types: [main]
  confidence_threshold: 0.5
priority: 8
token_budget: 400
---

# Telegram Optimizations

**Primary channel:** Telegram (via openclaw-telegram plugin)

## Available Features

**Inline buttons:**
- Set `webchat.capabilities.inlineButtons` to enable
- Format: Standard Telegram inline keyboard JSON
- Not enabled by default for webchat

**Voice TTS:**
- Enabled automatically
- Use `[[tts:...]]` tags for natural speech
- Auto-summary if message >1500 chars
- Only use when user sends voice/audio input

**Voice input:**
- Automatic transcription via Whisper (Groq)
- Handles voice messages and audio files
- No configuration needed

**Reply context:**
- Use `[[reply_to_current]]` tag to reply to triggering message
- Preserves conversation threading
- Works on all Telegram chat types

**Silent replies:**
- Use `NO_REPLY` (alone, nothing else) to skip response
- Useful for heartbeat acknowledgments
- Must be the ONLY content in your response

## Platform Limitations

**Formatting:**
- No markdown tables (use bullet lists instead)
- Headers work but keep them short
- Native link support (no wrapping needed)

**Messages:**
- Max length: ~4096 characters
- Long messages auto-split
- Preserve formatting across splits

**TTS config:**
- Keep spoken text concise (<1500 chars)
- Use expressive tags: `[[tts:excited]]`, `[[tts:serious]]`
- Separate text and voice: `[[tts:text]]..[[/tts:text]]`
