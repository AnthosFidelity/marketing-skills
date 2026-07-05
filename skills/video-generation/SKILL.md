---
name: video-generation
description: End-to-end AI video production through the Hyper MCP — text-to-video and image-to-video generation (Sora, Veo, Seedance), scene chaining, video analysis, transcription, subtitles, TikTok / karaoke captions, voiceover (TTS), audio mixing, clipping, stitching, and text overlays. Use when the user asks to generate a video, create UGC, scene-chain, add captions or subtitles, add narration, stitch clips, clip a podcast highlight, or do any AI video editing.
---

# Video Generation & Editing

Guide for generating, editing, analyzing, and post-processing videos using AI models and FFmpeg-backed tools exposed through the Hyper MCP.

## Requirements

This skill assumes the [Hyper MCP](https://app.hyperfx.ai/mcp) is connected to your agent so the tools below are available. The underlying providers (OpenAI Sora, Google Veo, ByteDance Seedance, OpenAI TTS, transcription, etc.) are configured under your Hyper integrations.

## Tool surface

| Group | Tools |
|-------|-------|
| Generation | `generate_video`, `sora_remix_video`, `sora_delete_video` |
| Analysis | `analyze_video`, `capture_video_frame`, `transcribe_video` |
| Subtitles & captions | `generate_subtitles`, `burn_subtitles`, `burn_highlighted_captions` |
| Audio | `text_to_speech`, `add_audio_to_video` |
| Editing | `clip_video`, `stitch_videos`, `overlay_text` |

## Out of scope

- Image generation, ad creative composition, brand extraction — use `image-generation` or `ad-creative-generation`.
- Posting finished videos to social platforms — use `tiktok`, `instagram`, or `linkedin`.
- Running paid video campaigns — use `google-ads`, `meta-ads`, `tiktok-ads`.

## Available Tools

| Tool | Purpose | Runs in Background |
|------|---------|-------------------|
| `generate_video` | Generate video from text / image prompt | Yes |
| `sora_remix_video` | Modify existing Sora video | Yes |
| `sora_delete_video` | Delete a Sora video | No |
| `capture_video_frame` | Extract frame as image | No |
| `analyze_video` | Watch and understand video content | No |
| `transcribe_video` | Extract audio transcript | No |
| `generate_subtitles` | Create SRT / VTT subtitle file | No |
| `burn_subtitles` | Burn subtitles onto video | Yes |
| `burn_highlighted_captions` | TikTok / karaoke-style word-by-word captions | Yes |
| `text_to_speech` | Generate voiceover audio from text | No |
| `add_audio_to_video` | Add / replace audio track on video | Yes |
| `clip_video` | Extract a time segment from video | Yes |
| `stitch_videos` | Concatenate multiple clips | Yes |
| `overlay_text` | Add text / titles to video | Yes |

## Video Understanding

You can **watch and analyze any video** using `analyze_video`. This sends the video to a multimodal AI that sees both visual and audio content.

### When to use `analyze_video`

- After generating a video: check if it matches your intent
- Before stitching: verify scene consistency across clips
- Quality review: check for glitches, character drift, lighting issues
- Content understanding: "what happens in this video?"

### Analysis Types

```python
analyze_video(file_id="...", analysis_type="general")
analyze_video(file_id="...", analysis_type="quality_review")
analyze_video(file_id="...", analysis_type="scene_breakdown")
analyze_video(file_id="...", question="Does this match: [original prompt]?")
```

### Self-Review Workflow

Always review generated videos before delivering to the user:

```python
result = generate_video(prompt="...", model="veo-3.1-generate-preview")
review = analyze_video(file_id="video_file_id", analysis_type="quality_review")
# If issues found, regenerate with adjustments. If quality is good, proceed to editing.
```

## Routing table

> **All reference files live in `references/`.** Read them at `references/<file>` (e.g. `references/generation.md`).

| The user wants to… | Read these files first |
|---|---|
| Generate a video (any model) | [references/generation.md](references/generation.md) — model selection, parameter matrix, prompt templates |
| Build a longer multi-scene video | [references/generation.md](references/generation.md) — script planning + scene chaining |
| Add subtitles / captions / voiceover / overlays, or clip a video | [references/post-production.md](references/post-production.md) |
| Produce UGC / TikTok content end-to-end | [references/workflows.md](references/workflows.md) → [references/generation.md](references/generation.md) |
| Turn a podcast / long video into short clips | [references/workflows.md](references/workflows.md) → [references/post-production.md](references/post-production.md) |
| Understand or QA an existing video | Use `analyze_video` (see Video Understanding above) |

## Best Practices

1. **Review before delivering:** always use `analyze_video` to check your output.
2. **Maintain visual consistency:** use the same character descriptions, lighting, and style across all scenes.
3. **Plan transitions:** design the end of each scene to flow into the next.
4. **Batch similar scenes:** generate scenes with similar settings together.
5. **Review before chaining:** check each scene before using its last frame for the next.
6. **Use single-variable iteration:** remix / regenerate by changing one variable at a time.
7. **Add captions for accessibility:** use the subtitle pipeline for all UGC content.
