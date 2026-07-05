# Video Generation: End-to-End Production Workflows

## UGC / TikTok Production Workflow

Complete workflow for producing UGC-style content:

1. **Script:** plan scenes, dialogue, and visual style
2. **Generate:** create each scene with `generate_video`
3. **Review:** use `analyze_video` to check each scene for quality
4. **Chain:** extract last frames with `capture_video_frame`, generate next scenes
5. **Stitch:** combine all scenes with `stitch_videos`
6. **Narrate:** generate voiceover with `text_to_speech` + `add_audio_to_video`
7. **Caption:** add TikTok-style captions with `burn_highlighted_captions`
8. **Overlay:** add titles / CTAs with `overlay_text`
9. **Final review:** use `analyze_video` on the final video for quality check

### Example: Narrated UGC Video

```python
generate_video(prompt="...", model="veo-3.1-generate-preview")

audio = text_to_speech(text="Your narration script here...", voice="nova")

add_audio_to_video(video_file_id="generated_video_id", audio_file_id=audio.file_id)

burn_highlighted_captions(video_file_id="narrated_video_id", style="tiktok")
```

### Example: Podcast to Short-Form Clips

```python
transcript = transcribe_video(file_id="podcast_video_id")

analysis = analyze_video(
    file_id="podcast_video_id",
    question="Identify the 3 most memorable / quotable moments with timestamps",
    analysis_type="scene_breakdown",
)

clip_video(video_file_id="podcast_video_id", start_time=120.0, end_time=150.0)
clip_video(video_file_id="podcast_video_id", start_time=340.0, end_time=365.0)

stitch_videos(video_file_ids=["clip1_id", "clip2_id"])

burn_highlighted_captions(video_file_id="stitched_id", style="tiktok")
```
