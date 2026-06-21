# Qwen3 STS (1.7B-Base) — ComfyUI Workflow

ComfyUI workflow for **Qwen3 Text-to-Speech (TTS)** using the [Qwen3-TTS-12Hz-1.7B-Base](https://huggingface.co/Qwen/Qwen3-TTS-12Hz-1.7B-Base) model from HuggingFace.

## Overview

This workflow enables high-quality speech synthesis using the Qwen3 TTS model, supporting:

- **Voice cloning** from reference audio
- **Multi-speaker / role-based** dialogue generation
- **Advanced dialogue controls** (temperature, CFG, seed, etc.)
- **Audio post-processing** (trimming, noise reduction, sample rate)
- **MP3 export** at configurable bitrates

## Model Configuration

| Setting          | Value                              |
|------------------|------------------------------------|
| Model            | `Qwen/Qwen3-TTS-12Hz-1.7B-Base`  |
| Source           | HuggingFace                        |
| Precision        | fp16                               |
| Attention Mode   | sdpa                               |

## Node Breakdown

| #  | Node                      | Description                                              |
|----|---------------------------|----------------------------------------------------------|
| 1  | `LoadAudio`              | Loads input/reference audio file                         |
| 2  | `SaveAudioMP3`           | Saves final output as MP3 (default: `audio/ComfyUI`, 320k) |
| 4  | `Qwen3TTSLoader`         | Loads the Qwen3 TTS model object                         |
| 5  | `Qwen3TTSSenseVoiceASR`  | Transcribes input audio using SenseVoiceSmall ASR        |
| 6  | `PreviewAny`             | Preview intermediate strings                             |
| 7  | `Qwen3TTSVoiceClonePrompt` | Generates voice clone prompt from reference audio/text   |
| 8  | `Qwen3TTSPromptManager`  | Manages, edits, and saves prompts (supports `.qwen3tts` files) |
| 9  | `Qwen3TTSRoleBank`       | Defines speaker roles for multi-turn dialogue            |
| 10 | `Qwen3TTSAdvancedDialogue` | Core synthesis node — generates audio from texts, instructions, roles, pauses |
| 11 | `Qwen3TTSScriptProcessor`  | Parses script text into structured lists (texts, instructions, roles, pauses) |
| 12 | `Qwen3TTSAudioPostProcess` | Post-processes output audio (trim start/end, denoise, resample) |

## Workflow Graph

```
LoadAudio (ref audio)
    │
    ├─► Qwen3TTSSenseVoiceASR ──► text + suggested_instruct
    │            │
    │            ▼
    │   Qwen3TTSVoiceClonePrompt ◄── model_obj (from Loader)
    │            │
    │            ▼
    │   Qwen3TTSPromptManager  ──► voice_clone_prompt
    │            │
    │            ▼
    │   Qwen3TTSRoleBank       ──► role_bank
    │            │                    ▲
    │            ▼                    │
    └───────────────┐                │
                    │                │
        Qwen3TTSScriptProcessor      │
          (texts, instructs,         │
           roles, pauses)            │
                    │                │
                    ▼                │
       Qwen3TTSAdvancedDialogue ─────┘
                    │
                    ▼
       Qwen3TTSAudioPostProcess
                    │
                    ▼
               SaveAudioMP3
```

## Parameters

### Qwen3TTSAdvancedDialogue

| Parameter        | Default          | Description                          |
|------------------|------------------|--------------------------------------|
| Seed             | `165095999329882`| Random seed for reproducibility      |
| Sampler          | `randomize`      | Sampling method                      |
| Max New Tokens   | `4096`           | Maximum generation length            |
| Temperature      | `0.7`            | Creativity control                   |
| Top P            | `0.8`            | Nucleus sampling threshold           |
| Steps            | `50`             | Number of diffusion steps            |
| Scale            | `1.1`            | CFG / classifier-free guidance scale |
| Silence Threshold | `0.9`          | Silence detection threshold          |
| Num Speakers     | `1`              | Number of speakers                   |
| Seed (Dialogue)  | `50`             | Dialogue seed                        |

### Qwen3TTSAudioPostProcess

| Parameter        | Default  | Description                          |
|------------------|----------|--------------------------------------|
| Trim Start (s)   | `10`     | Seconds to trim from start           |
| Trim End (s)     | `50`     | Seconds to trim from end             |
| Sample Rate      | `44100`  | Output sample rate in Hz             |

### Qwen3TTSPromptManager

| Parameter        | Default       | Description                          |
|------------------|---------------|--------------------------------------|
| Action           | `Save`        | Prompt action (Save, Load, etc.)     |
| Filename         | `your_filename` | Base filename for prompts          |
| Extension        | `my_voice_01.qwen3tts` | File extension / preset name  |

## Custom Nodes Required

This workflow uses custom ComfyUI nodes from the **Qwen3-TTS** ecosystem:

- `Qwen3TTSLoader`
- `Qwen3TTSSenseVoiceASR`
- `Qwen3TTSVoiceClonePrompt`
- `Qwen3TTSPromptManager`
- `Qwen3TTSRoleBank`
- `Qwen3TTSAdvancedDialogue`
- `Qwen3TTSScriptProcessor`
- `Qwen3TTSAudioPostProcess`

Ensure the relevant custom node packages are installed in your ComfyUI environment.

## Additional Nodes

- `LoadAudio` — from ComfyUI's core audio loading nodes
- `PreviewAny` — from a preview/custom nodes extension
- `SaveAudioMP3` — for MP3 audio export

## Usage

1. Import this JSON file into ComfyUI via **Load** → **Import**.
2. Connect your reference audio to `LoadAudio`.
3. Edit the script in `Qwen3TTSScriptProcessor` with your desired text and roles.
4. Adjust dialogue parameters in `Qwen3TTSAdvancedDialogue` as needed.
5. Run the workflow — output MP3 will be saved to `audio/ComfyUI`.

## File Info

- **Workflow ID:** `5e5d10ea-da1b-4198-9d27-3c9513ff0022`
- **Last Node ID:** 17
- **Total Links:** 22
- **Revision:** 0

---

# macOS M-Series Max — Setup & Optimization

## Installation on Apple Silicon (M3/M4 Max, 48GB Unified Memory)

### PyTorch Setup

Use the default MPS build — no CUDA wheels needed:

```bash
pip install torch torchvision torchaudio
```

The default PyPI torch wheel for macOS includes Metal Performance Shaders (MPS) support. CUDA wheels (`cu124`, `cu128`) are unnecessary and can cause conflicts on Apple Silicon.

### Attention Mode Recommendation

| Mode | Recommendation | Reason |
|------|---------------|--------|
| **SDPA** | ✅ **Recommended** | Built-in, stable, fully MPS-compatible — matches this workflow's config |
| **FlashAttention** | ⚠️ Optional | Faster on some workloads but can be unstable with Qwen3-TTS on Mac |

### Memory Considerations (48GB Unified)

- 48GB unified memory handles the 1.7B-Base model at fp16 without issue.
- Keep `Max New Tokens` at **4096** or lower to avoid OOM on long scripts.
- Close other memory-heavy apps (browsers, Xcode) before running large TTS batches.
- If you hit memory pressure, reduce `Max New Tokens` to **2048** or process scripts in smaller chunks.

---

## Workflow Architecture

This workflow implements a three-stage pipeline for multi-speaker TTS:

1. **ScriptProcessor** parses raw text into structured lists (texts, instructions, roles, pauses)
2. **RoleBank** maps speaker names to voice references (cloned from 15s audio clips)
3. **AdvancedDialogue** generates audio sequentially — fetching the correct voice per line and inserting silence between segments

Using only the "Voice Clone" node without RoleBank loses control over timing and multi-speaker switching.

---

## Model & Parameter Tuning

### Choosing the Right Model Variant

| Variant | Best For |
|---------|---------|
| **1.7B-Base** | Voice cloning — mimics emotion from reference audio (used here) |
| **1.7B-VoiceDesign** | Creating new voices from text prompts |
| **1.7B-CustomVoice** | Internal presets (Vivian, Uncle_Fu, etc.) |

The 0.6B model struggles with emotional range — always prefer the 1.7B variant for production quality.

### ASR-Assisted Voice Cloning

Qwen3's cloning works by subtracting known words from audio to isolate tone. If `ref_text` is empty, the model guesses — resulting in mumbled output.

**Recommended flow:** Route reference audio through `Qwen3TTSSenseVoiceASR` → feed transcribed text into `Qwen3TTSVoiceClonePrompt.ref_text`. This significantly improves speaker similarity.

### Reference Audio Guidelines

| Guideline | Detail |
|-----------|--------|
| **Duration** | 10–15 seconds; longer clips waste memory and confuse the model |
| **Energy match** | The reference clip's tone should match your target output (excited, calm, etc.) |

### Recommended Generation Parameters

These values improve on the workflow defaults for more natural-sounding output:

| Parameter | Workflow Default | Recommended | Why |
|-----------|-----------------|-------------|-----|
| X-Vector Only | — | `False` | Enables context-aware cloning instead of tone guessing |
| Temperature | `0.7` | `0.8` | Reduces flat, robotic quality |
| Top_P | `0.8` | `0.9` | Wider intonation range |
| Scale | `1.1` | `1.1` | Prevents repetition on technical words |
| Max New Tokens | `4096` | `4096` | Default 1024 cuts long scripts short |

---

## Scripting & Prosody Control

Use `[pause:x]` tags in the `ScriptProcessor` for precise timing. Without this node, bracketed text is read aloud literally.

### Pause Tags

| Tag | Effect | Use Case |
|-----|--------|---------|
| **Ellipsis (`…`)** | Hesitation / trailing off | Natural speech flow |
| **Hard Pause (`[pause:1.0]`)** | Exact silence of specified duration | Dramatic beats, topic changes |

### Emotion & Sound Tags (1.7B Models)

| Tag | Description |
|-----|------------|
| **`[laugh]`** | Natural laughter |
| **`[sigh]`** | Breathy exhale |
| **`[scream]`** | Experimental — use cautiously |

> **Tip:** Remove stray brackets (e.g., `[Credit]`) to avoid unwanted noise artifacts.

### Punctuation Guide

Punctuation shapes the model's rhythm and intonation:

| Symbol | Voice Effect | When to Use |
|--------|-------------|-------------|
| `,` | Short breath / micro-pause | Breaking long sentences |
| `.` | Full stop, pitch drop | Authoritative statements |
| `?` | Pitch rise | Questions — avoid overuse |
| `!` | Increased volume / pitch | Emphasis, calls to action |
| `" "` | Tone shift between speakers | Dialogue distinction |

### Script Format Example

Format text in the `ScriptProcessor` with speaker names to switch voices defined in your `RoleBank`:

```
Host: [Neutral] Welcome to today's episode. [pause:1.0]
Guest: [Warm] Thanks for having me! [laugh]
Commentator: [Serious] The results were unexpected. [sigh]
```

---

## Audio Post-Processing

Qwen3-TTS can produce a click/pop at the start of generated audio due to autoregressive decoding. `Qwen3TTSAudioPostProcess` addresses this:

| Setting | Workflow Default | Recommended | Notes |
|---------|-----------------|-------------|-------|
| Trim Start | `10s` | 10–20ms equivalent | Removes initial click without cutting content |
| Trim End | `50s` | Adjust as needed | Depends on source length |
| Sample Rate | `44100` Hz | `48000` Hz | Better for video editor sync (Premiere, DaVinci) |

## Troubleshooting

| Issue | Likely Cause | Fix |
|-------|-------------|-----|
| **MPS / "no kernel image" error** | Outdated or mismatched PyTorch | `pip install --upgrade torch`; use default macOS build |
| **Metallic / robotic voice** | Wrong model variant, high repetition penalty, x_vector enabled | Use 1.7B-Base; lower Scale to 1.05–1.1; uncheck x_vector_only |
| **"flash_attn" module not found** | FlashAttention misconfigured | Switch attention mode to `sdpa` in Qwen3TTSLoader |
| **Mumbled / wrong transcription output** | Empty or inaccurate `ref_text` | Route reference audio through SenseVoiceASR first |
| **Out of Memory (OOM)** | Max tokens exceeded on long scripts | Reduce Max New Tokens to 2048; split scripts; close other apps |
| **Click/pop at audio start** | Autoregressive decoding artifact | Use AudioPostProcess with fade-in; adjust Trim Start |
| **Workflow crashes on import/run** | Missing dependencies or config issues | Verify all custom nodes installed; check PyTorch MPS build; confirm model is downloaded; ensure ref_text is populated |
