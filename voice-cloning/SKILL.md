---
name: voice-cloning
description: >-
  Write correct Sarvam Voice Cloning code — create a cloned voice from a
  reference clip, then generate cross-lingual speech with it. Use this skill
  when building a feature that speaks in a specific person's voice (not a
  stock Bulbul speaker) in Python or JS/TS. For stock-speaker TTS, use
  text-to-speech instead. For live cloned-voice speech in chat via MCP, use
  sarvam-mcp instead.
license: Apache-2.0
metadata:
  author: sarvam-ai
  version: "1.0"
---

# Voice Cloning — Sarvam AI

> Stock speaker TTS (`shubh`, `priya`, ...)? → [text-to-speech](../text-to-speech/SKILL.md) — Bulbul, not this API.

> [!IMPORTANT]
> Auth: `api-subscription-key` header — NOT `Authorization: Bearer`. Base URL: `https://api.sarvam.ai`.
> Voice cloning requires consent — only clone a voice you have the right to use.

## What this API does

Two calls: create a saved voice from a short reference clip, then synthesize speech with it. Cross-lingual — the reference clip's language does not need to match the language you synthesize in. 12 Indian languages on the synthesis side.

## Quick Start (Python)

```python
from sarvamai import SarvamAI

client = SarvamAI(api_subscription_key="YOUR_SARVAM_API_KEY")

# 1. Create the voice from a reference clip (10-30s recommended, single speaker, <50MB)
created = client.voice_cloning.create_voice(
    file=open("reference.wav", "rb"),
    name="my-cloned-voice",
    language="en-IN",
)
voice_id = created.voice_id   # "svc-<uuid>" — save this, no need to re-upload the clip again

# 2. Generate speech with it, in any supported language
response = client.voice_cloning.text_to_speech(
    voice_id=voice_id,
    text="नमस्ते, मैं आपकी कैसे मदद कर सकता हूँ?",
    language_code="hi-IN",
)

import base64
with open("output.wav", "wb") as f:
    f.write(base64.b64decode(response.audio))
```

## Quick Start (JavaScript/TypeScript)

```typescript
import fs from "fs";
import { SarvamAIClient } from "sarvamai";

const client = new SarvamAIClient({ apiSubscriptionKey: "YOUR_SARVAM_API_KEY" });

// 1. Create the voice
const created = await client.voiceCloning.createVoice({
  file: fs.createReadStream("reference.wav"),
  name: "my-cloned-voice",
  language: "en-IN",
});
const voiceId = created.voiceId;

// 2. Generate speech — note voiceId is camelCase but language_code stays snake_case
const response = await client.voiceCloning.textToSpeech({
  voiceId,
  text: "नमस्ते, मैं आपकी कैसे मदद कर सकता हूँ?",
  language_code: "hi-IN",
});

fs.writeFileSync("output.wav", Buffer.from(response.audio, "base64"));
```

## Managing saved voices

| Action | Endpoint | Notes |
|--------|----------|-------|
| List | `GET /voices` | Filter by name/gender/accent, paginated. Shared with Content Studio's Voice Library. |
| Get details | `GET /voices/{voice_id}` | Includes transcript + time-limited signed URLs for reference/preview audio. |
| Delete | `DELETE /voices/delete/{voice_id}` | Frees a slot — voice count is capped by subscription tier. |

## Gotchas

| Gotcha | Detail |
|--------|--------|
| **Two separate calls, one voice_id** | `POST /voices/create` returns `voice_id` + an auto-generated transcript (`reference_text`) — the voice is ready immediately, no polling. `POST /voices/clone` (the "generate speech" call, SDK method `text_to_speech`/`textToSpeech`) does **not** accept a reference audio file — only `voice_id` + `text` + `language_code`. |
| **Response is unwrapped — no `.data`** | Unlike `dubbing`/`document_translation` (which return `response.data.job_id`), `voice_cloning` calls return fields directly: `created.voice_id` (Python) / `created.voiceId` (JS), not `created.data.voice_id`. Don't copy the `.data.` pattern from other async-job skills into this one. |
| **Mixed casing in JS** | `client.voiceCloning.textToSpeech({...})` takes `voiceId` (camelCase) but `language_code` (snake_case) in the same call — the SDK does not fully normalize this endpoint's field names. |
| **1,000-char cap on synthesis** | `text` over 1,000 characters returns `400`. Split and synthesize in parts for longer content. |
| **Studio-only fields reject on create** | Sending `consent`, `passage_id`, `noise_reduction`, `accent`, or `ref_text` to `create_voice`/`createVoice` returns `400` — those are Content Studio–only fields, not part of the public API surface. |
| **`pace` and `max_audio_duration` are mutually exclusive** | Sending both on `text_to_speech` returns `400`. Use `pace` (0.5–2.0) for a speed multiplier, or `max_audio_duration` to time-stretch to a fixed length — not both. |
| **`enable_qc` is off by default** | Quality control (re-transcribe + score, retry on failure) only runs when `enable_qc=True` — adds latency (~0.5s+ for single-script text, more for code-mixed). |
| **12-language output, not 23** | Synthesis (`language_code`) supports the same 12-language set as Bulbul-adjacent products (`as-IN`, `bn-IN`, `en-IN`, `gu-IN`, `hi-IN`, `kn-IN`, `ml-IN`, `mr-IN`, `od-IN`, `pa-IN`, `ta-IN`, `te-IN`) — not the 23-language Saaras/Vision set. |
| **Reference audio limits** | Max 50MB, 10–30s recommended, single speaker, minimal background noise — poor references fail with `422` (too noisy/short/multi-speaker). |

## Full Docs

Fetch cross-lingual details, quality-control tuning, and audio-format options from:

- **https://docs.sarvam.ai/llms.txt** — comprehensive docs index
- [Voice Cloning Overview](https://docs.sarvam.ai/api/api-guides-tutorials/voice-cloning/overview)
- [API Reference](https://docs.sarvam.ai/api-reference/voice-cloning)
- [Rate Limits](https://docs.sarvam.ai/api/ratelimits)
