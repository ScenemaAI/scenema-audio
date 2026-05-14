# Scenema Audio

**Zero-shot expressive voice cloning and speech generation.**

**[Visit scenema.ai/audio to hear all demos and try it out.](https://scenema.ai/audio)**

[![Demo Video](https://cdn.scenema.ai/public/content/scenema-audio-waveform-thumbnail.png)](https://youtu.be/VnEQ_ImOaAc)

Every existing text-to-speech system converts words into sound, but none of them perform. Speech that merely pronounces words correctly is functionally useless for filmmaking, audiobooks, or any context where the emotional delivery carries as much meaning as the words themselves. Scenema Audio generates speech with intention, pacing, breath control, and emotional arcs that shift within a single generation, all from a text prompt that describes not just what to say but how to say it.

Built on an audio diffusion transformer extracted from [LTX 2.3](https://github.com/Lightricks/LTX-2)'s 22B parameter audiovisual model, it learned how people actually sound in real scenes: angry, laughing, whispering, crying, exhausted, terrified.

## Quick Start

### Docker (Recommended)

```bash
git clone https://github.com/ScenemaAI/scenema-audio.git
cd scenema-audio

# Set your HuggingFace token (Gemma 3 access required)
export HF_TOKEN=your_huggingface_token

# Build and run (models are downloaded on first start)
docker compose up
```

First startup downloads ~38 GB of model checkpoints and caches them in a Docker volume. Subsequent starts are fast.

### Generate Audio

```bash
# Using the included script
python generate.py output.wav

# Or with curl
curl -s -X POST http://localhost:8000/generate \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "<speak voice=\"A warm, clear male voice with a slight British accent. Measured, thoughtful pacing.\" gender=\"male\">The old lighthouse had stood on the cliff for over a century, its beam cutting through the fog like a blade of light.</speak>",
    "seed": 42
  }' | grep -oP '"audio":"\K[^"]+' | base64 -d > output.wav
```

### Voice Design (Preview a Voice)

```bash
curl -s -X POST http://localhost:8000/generate \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "<speak voice=\"A young woman with a smoky, low register voice. Intimate, confessional tone.\" gender=\"female\">The city never really sleeps. It just closes its eyes and pretends for a while.</speak>",
    "mode": "voice_design"
  }' | grep -oP '"audio":"\K[^"]+' | base64 -d > voice_preview.wav
```

### Zero-Shot Voice Cloning

Provide 10-20 seconds of reference audio with some emotional variability. The model generates expressive speech from the prompt, then transfers the reference voice's identity onto the performance. References that contain a range of pitch and intonation produce significantly better identity transfer than flat, monotone clips.

```bash
curl -s -X POST http://localhost:8000/generate \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "<speak voice=\"Gravelly male voice, fast talking, rough.\" gender=\"male\"><action>He completely loses it, shouting</action>What are you waiting for?!</speak>",
    "reference_voice_url": "https://example.com/calm-reference.wav",
    "seed": 42
  }' | grep -oP '"audio":"\K[^"]+' | base64 -d > cloned_angry.wav
```

Any voice can perform any emotion, even if that voice has never been recorded in that emotional state. The reference provides identity. The performance comes from the prompt.

## Prompt Format

```xml
<speak voice="VOICE_DESCRIPTION" gender="male|female"
       scene="OPTIONAL_SCENE" language="OPTIONAL_LANG_CODE"
       shot="closeup|wide|scene">
  <action>Performance direction.</action>
  Speech text here.
  <sound>Environmental audio event.</sound>
  More speech.
</speak>
```

### Attributes

| Attribute | Required | Default | Description |
|-----------|----------|---------|-------------|
| `voice` | Yes | | Detailed voice description. Drives vocal quality, emotion, accent, age, timbre, delivery style. The more specific and theatrical, the better. |
| `gender` | Yes | | `"male"` or `"female"`. Controls pronoun assignment in the compiled prompt sent to the diffusion model. |
| `scene` | No | | Environmental context. Conditions the ambient audio environment around the speech (rain, office hum, crowd noise). |
| `language` | No | `"en"` | Language code. The model supports major world languages with native-sounding output. |
| `shot` | No | `"closeup"` | Controls SFX prominence. `"closeup"`: speech-focused, SFX minimal. `"wide"`: environment + speech. `"scene"`: maximum environmental audio, SFX reinforced. |

### Child Elements

| Element | Description |
|---------|-------------|
| Text nodes | The actual speech content. Write natural prose. |
| `<action>` | Performance directions that shape HOW the speech is delivered. Not spoken aloud. Stage directions for the diffusion model: emotional shifts, physical delivery, pacing cues, breath control. |
| `<sound>` | Environmental audio events generated alongside the speech. Thunder cracks, doors slamming, rain starting. Only effective in `wide` or `scene` shot modes. |

### Voice Description

The `voice` attribute is the primary control for the entire output. Be specific and theatrical:

```xml
<!-- Weak -->
<speak voice="A man speaking" gender="male">...</speak>

<!-- Strong -->
<speak voice="Male, mid 60s. Deep baritone with gravel. Slight Southern American inflection.
Worn but warm. Nostalgic, firelight cadence. The voice of someone who has seen too much
and chosen kindness anyway." gender="male" scene="Fireside, night, crickets">...</speak>
```

### Action Tags

Action tags are the primary tool for controlling emotional performance. Place them between speech segments to direct delivery shifts:

```xml
<speak voice="Middle-aged man, warm but weathered." gender="male">
  <action>Calm, almost casual. Staring at his hands.</action>
  I used to think I had all the time in the world.
  <action>Voice tightens. Swallows. Fighting to stay composed.</action>
  Then one Tuesday morning, the doctor said three words that changed everything.
  <action>Long pause. Deep breath. When he speaks again, his voice is raw but steady.</action>
  And I realized... I hadn't called my son in six months.
  <action>Voice breaks on the last word. Clears throat. Forces a half-laugh.</action>
  Funny how that works, isn't it?
</speak>
```

Describe what the speaker is DOING and FEELING, not what the audio should sound like. Combine physical and emotional cues for richer performance.

## API Reference

### POST /generate

#### Request Body

| Field | Type | Default | Description                                                                                                                                                                                                                                                                                                                                                                             |
|-------|------|---------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `prompt` | string | **required** | `<speak>` XML string. See Prompt Format above.                                                                                                                                                                                                                                                                                                                                          |
| `mode` | string | `"generate"` | `"generate"` for full pipeline with chunking. `"voice_design"` for a single 15-second voice sample (no chunking, useful for previewing a voice description).                                                                                                                                                                                                                            |
| `reference_voice_url` | string | `null` | URL to reference audio (WAV or MP3) for zero-shot voice cloning. 10-20 seconds of clean speech with some emotional variability is ideal. The reference provides identity; emotional performance comes from the prompt.                                                                                                                                                                                             |
| `background_sfx` | bool | `false` | Keep generated environmental sound effects in the output. When `false`, non-vocal audio is removed. Set to `true` when using `shot="scene"` or `shot="wide"` with `<sound>` tags.                                                                                                                                                                                                       |
| `validate` | bool | `true` | Enable Whisper speech validation. Each generated chunk is transcribed by faster-whisper and compared against expected text. If word match ratio falls below the threshold, the chunk is regenerated with extended duration and a new seed (up to 3 retries), keeping the best result. Adds <1s per chunk on GPU. Disable for faster generation when prompt reliability is not critical. |
| `seed` | int | `-1` | Generation seed. `-1` for random. Fixed seeds produce deterministic output for the same prompt and configuration.                                                                                                                                                                                                                                                                       |
| `pace` | float | `1.5` | Duration allocation multiplier. Higher values give the model more time, resulting in slower, more deliberate speech. Lower values produce faster speech. The default 1.5x accounts for LTX's naturally slower speaking pace compared to real-time speech.                                                                                                                               |
| `min_match_ratio` | float | `0.90` | Whisper validation threshold. Minimum word match ratio (0.0 to 1.0) between generated audio transcription and expected text. Only used when `validate` is `true`. Lower values accept more pronunciation variance. Lower threshold recommended for languages with accents.                                                                                                              |
| `skip_vc` | bool | `false` | Skip voice conversion (SeedVC) post-processing entirely. When `true`, no voice identity transfer or cross-chunk voice consistency normalization is applied. Useful for single-chunk generations where the voice description alone is sufficient.                                                                                                                                        |
| `vc_steps` | int | `25` | SeedVC diffusion steps. More steps produce higher-quality voice identity transfer at the cost of processing time. Range: 10-50.                                                                                                                                                                                                                                                         |
| `vc_cfg_rate` | float | `0.5` | SeedVC classifier-free guidance rate. Controls how strongly the target voice identity is applied. Higher values produce stronger identity transfer but may reduce naturalness. Range: 0.0-1.0.                                                                                                                                                                                          |

#### Response

Returns JSON with base64-encoded WAV audio:

```json
{
  "status": "succeeded",
  "audio": "<base64-encoded WAV>",
  "content_type": "audio/wav",
  "metadata": {
    "duration_s": 12.4,
    "sample_rate": 48000,
    "processing_ms": 8200,
    "seed": 42,
    "mode": "generate",
    "has_reference_voice": false
  }
}
```

On error:

```json
{
  "status": "failed",
  "error": "Description of what went wrong"
}
```

## Capabilities

### Emotional Acting

Emotional state shifts within a single generation. Action tags function as stage directions at specific points in the script.

```xml
<speak voice="A man on the edge. Explosive rage. Italian-American inflection."
       gender="male" scene="A dimly lit office, late at night">
  <action>He stands up slowly, voice dangerously low</action>
  You come into my house, you eat my food, and then you got the nerve
  to tell me how to run my business.
  <action>Voice rising, finger pointing</action>
  I built this thing from nothing while you were sitting on your ass.
</speak>
```

### Child Voices

```xml
<speak voice="A six-year-old girl, bright and excited, speaking fast
with breathless enthusiasm. Slight lisp on S sounds."
gender="female">
  Mommy look! There is a rainbow and it goes all the way across the whole sky!
</speak>
```

### Scene-Aware Audio (Voice + Environment)

Set `shot="scene"` and `background_sfx: true` to generate speech with environmental audio in the same diffusion pass.

```xml
<speak voice="Male, mid 40s. Weathered. Urgent, projecting over wind."
       gender="male" scene="Open dock in a thunderstorm, heavy rain"
       shot="scene">
  <sound>Heavy rain and wind howling</sound>
  <action>He shouts over the storm</action>
  Get the lines! She is pulling loose!
  <sound>Thunder cracks overhead</sound>
  Move! I said move!
</speak>
```

### Multilingual

The model supports major world languages with native fluency. Set the `language` attribute and write the voice description to match.

```xml
<speak voice="Female, mid 70s. Soft alto. Native French speaker, Parisian accent.
Warm like wool blankets. Unhurried." gender="female"
scene="Cozy bedroom, lamplight" language="fr">
  <action>Elle s'assied au bord du lit</action>
  Alors, mon petit. Tu veux que je te raconte l'histoire du renard
  qui a trompé la lune?
</speak>
```

### Long-Form Narration

Text is automatically split at sentence boundaries using [Kokoro](https://github.com/hexgrad/kokoro) phoneme-level duration estimation. Voice identity is maintained across chunks via A2V latent conditioning.

```xml
<speak voice="An elderly storyteller with a weathered knowing voice.
Deep baritone, slow deliberate pacing."
gender="male">
  Many years later, as he faced the firing squad, Colonel Aureliano Buendia
  was to remember that distant afternoon when his father took him to discover ice.
  At that time Macondo was a village of twenty adobe houses, built on the bank
  of a river of clear water that ran along a bed of polished stones, which were
  white and enormous, like prehistoric eggs.
</speak>
```

## Hardware Requirements

### Minimum: 16 GB VRAM (RTX 4060 Ti 16GB, RTX A4000)

INT8 audio transformer on GPU, Gemma streams from CPU RAM (requires 32 GB system RAM). Slower text encoding (~7s per chunk) but fully functional.

### Standard: 24 GB VRAM (RTX 4090, RTX A5000)

INT8 audio transformer + NF4 Gemma, all on GPU. Default configuration via `docker compose up`.

### Recommended: 48 GB VRAM (A6000 Ada, A40, L40S)

Full bf16 precision, all models resident on GPU. Best quality, fastest generation. Set environment variables:

```
AUDIO_CKPT=/app/models/scenema-audio-transformer.safetensors
GEMMA_QUANTIZE=
```

### VRAM Configurations

| VRAM | Audio Model | Gemma | Encode Speed | Notes |
|------|------------|-------|-------------|-------|
| 16 GB | INT8 (4.9 GB) | CPU streaming | ~7s/chunk | Needs 32 GB system RAM |
| 24 GB | INT8 (4.9 GB) | NF4 on GPU (~8 GB) | ~0.2s/chunk | Default config |
| 48 GB | bf16 (9.8 GB) | bf16 on GPU (24 GB) | ~0.2s/chunk | Best quality |

VRAM strategy is auto-detected. The service automatically selects the best configuration for your GPU.

## Performance

Benchmarked on NVIDIA RTX 4090, 100-word passage (~55 seconds of audio, 4 chunks):

| Configuration | Total Time | Real-Time Factor |
|--------------|-----------|-----------------|
| bf16 + bf16 (CPU streaming) | 83s | 0.66x |
| INT8 + bf16 (CPU streaming) | 66s | 0.83x |
| INT8 + NF4 (all GPU) | 35s | 1.57x |
| INT8 + NF4 + SageAttention 2 | 35s | 1.57x |

## Pipeline Architecture

```
XML prompt (voice description + scene + stage directions + text)
  |
  v
[Text Splitting] -----------> Sentence boundaries via Kokoro, ~15s max per segment
  |
  v
[Gemma 3 12B Encode] -------> Text conditioning (per segment)
  |
  v
[8-Step Diffusion] ---------> Audio latent generation
  |                            Voice continuity via A2V latent conditioning between segments
  v
[Audio Decode] --------------> Waveform
  |
  v
[MelBandRoFormer] ----------> Vocal separation (strips SFX unless background_sfx=true)
  |
  v
[SeedVC] -------------------> Voice identity transfer (when reference_voice_url provided
  |                            or multi-chunk for cross-chunk consistency)
  v
Output WAV (48kHz stereo)
```

### Key Design Decisions

**Kokoro for duration estimation.** Kokoro TTS (82M params, CPU) provides phoneme-level duration estimates. The chunker splits text at sentence boundaries when accumulated Kokoro estimates exceed 15 seconds (with a configurable `pace` multiplier for LTX's naturally slower speaking pace). No word counting.

**15-second chunk cap.** The model was trained on 20-second clips, but quality degrades (repetition, pronunciation failure) beyond ~15 seconds. The 15s cap ensures consistent quality.

**Voice continuity across segments.** The tail of each segment's audio is encoded and used as a voice reference for the next segment. This maintains consistent voice identity across arbitrarily long outputs without requiring a separate voice embedding model.

**Zero-shot voice cloning.** A2V latent conditioning gets about 60% of the way to matching a reference voice. SeedVC post-processing brings it to full identity transfer. No training, no enrollment, no voice database.

**Emotion and identity are independent controls.** The voice description drives the emotional performance. The reference audio drives the voice identity. For maximum emotional range with a cloned voice, use a strong character archetype in the voice description and let the reference audio handle identity.

**INT8 quantization.** Per-channel INT8 reduces the transformer from 9.8 GB to 4.9 GB with no measurable quality difference, enabling generation on consumer GPUs.

## Model Checkpoints

Hosted on HuggingFace: [ScenemaAI/scenema-audio](https://huggingface.co/ScenemaAI/scenema-audio)

| File | Size | Description |
|------|------|-------------|
| `scenema-audio-transformer.safetensors` | 9.8 GB | Audio diffusion transformer (bf16) |
| `scenema-audio-transformer-int8.safetensors` | 4.9 GB | Audio diffusion transformer (INT8, identical quality) |
| `scenema-audio-pipeline.safetensors` | 6.7 GB | Audio VAE decoder + vocoder + text projection |
| `scenema-audio-vae-encoder.safetensors` | 42.7 MB | Audio VAE encoder for reference voice encoding |

## Building from Source

```bash
git clone https://github.com/ScenemaAI/scenema-audio.git
cd scenema-audio

export HF_TOKEN=your_huggingface_token
docker compose build
docker compose up
```

### Environment Variables

Set in `docker-compose.yml` or pass via `docker run -e`:

| Variable | Default | Description |
|----------|---------|-------------|
| `HF_TOKEN` | **required** | HuggingFace token with Gemma 3 access |
| `AUDIO_CKPT` | `/app/models/scenema-audio-transformer-int8.safetensors` | Path to audio transformer checkpoint |
| `PIPELINE_CKPT` | `/app/models/scenema-audio-pipeline.safetensors` | Path to pipeline checkpoint |
| `GEMMA_ROOT` | `/app/models/gemma-3-12b-it` | Path to Gemma 3 12B model directory |
| `GEMMA_QUANTIZE` | `nf4` | Gemma quantization. `nf4` for 24 GB cards, empty for bf16 on 48 GB+ |
| `PORT` | `8000` | HTTP service port |
| `MODEL_DIR` | `/app/models` | Base directory for model downloads and cache |

## Limitations

- **Pronunciation**: The model occasionally garbles complex multi-syllable words and proper nouns. Spelling out difficult words phonetically can help.
- **15-second generation window**: Each audio segment is limited to ~15 seconds. Longer text is automatically split, but very long single sentences may be divided at suboptimal points.
- **Emotional range with voice cloning**: Voice cloning optimizes for identity accuracy, which can reduce the extremes of emotional delivery. For maximum expressiveness, use a strong emotional archetype in the voice description and provide a reference clip with natural emotional variability (10-20 seconds, not monotone).
- **Multilingual pronunciation**: When a character switches languages mid-speech, the model may apply the primary language's phonetics to the foreign words. Use separate requests per language.
- **Generation speed**: Each 15-second segment takes 3-8 seconds depending on hardware. Audio is returned as a complete file, not streamed.
- **Gemma 3 12B is gated**: Requires accepting Google's terms of use and a HuggingFace token with access.
- **Reference audio quality sensitivity**: Low-quality references (compressed MP3, background noise) significantly degrade output. Use clean reference audio or rely on the voice description alone with SeedVC as a post-processing step.

## Acknowledgments

- [LTX-2](https://github.com/Lightricks/LTX-2) by Lightricks for the base audiovisual model
- [Gemma 3](https://ai.google.dev/gemma) by Google for the text encoder
- [SeedVC](https://github.com/Plachtaa/seed-vc) by Plachta for voice refinement
- [Kokoro](https://github.com/hexgrad/kokoro) by hexgrad for duration estimation
- [SageAttention](https://github.com/thu-ml/SageAttention) for attention acceleration

## License

**Model weights:** [LTX-2 Community License Agreement](https://github.com/Lightricks/LTX-2/blob/main/LICENSE). The audio diffusion transformer is derived from LTX 2.3's audiovisual model, and its weights are subject to the same license terms.

**Code:** [MIT License](LICENSE). The inference server, chunking pipeline, and all supporting code are MIT licensed.

[Gemma 3 12B](https://ai.google.dev/gemma/terms) (text encoder) is a gated model requiring acceptance of Google's terms of use.
