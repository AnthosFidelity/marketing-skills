# Video Generation: Post-Production (Subtitles, Audio, Overlays, Clipping)

## Subtitle / Caption Workflow

### Full pipeline: Video → Transcript → Subtitles → Burned Video

```python
transcript = transcribe_video(file_id="video_file_id")

subs = generate_subtitles(file_id="video_file_id", transcript=transcript, format="srt")

burn_subtitles(
    video_file_id="video_file_id",
    subtitle_file_id=subs.file_id,
    style="bold_outline",
    position="bottom",
)
```

### Subtitle Styles

| Style | Effect |
|-------|--------|
| `default` | Plain white text |
| `bold_outline` | Bold white with black outline (recommended) |
| `shadow` | White text with drop shadow |
| `box` | White text on semi-transparent black box |

## Text Overlays

Add titles, lower-thirds, CTAs, and other graphics:

```python
overlay_text(
    video_file_id="video_file_id",
    overlays=[
        {
            "text": "Episode 1: The Beginning",
            "start_time": 0.0,
            "end_time": 3.0,
            "position": "center",
            "font_size": 48,
            "color": "white",
            "background": "black@0.5",
        },
        {
            "text": "Subscribe for more!",
            "start_time": 10.0,
            "end_time": 14.0,
            "position": "bottom-right",
            "font_size": 28,
        },
    ],
)
```

### Overlay Positions

`top`, `bottom`, `center`, `top-left`, `top-right`, `bottom-left`, `bottom-right`

## Voiceover / Narration

Generate natural-sounding voiceover with TTS and add it to any video:

```python
audio = text_to_speech(
    text="Welcome to our product. Here's how it works...",
    voice="nova",
    model="tts-1",
)

add_audio_to_video(
    video_file_id="video_id",
    audio_file_id=audio.file_id,
    mode="replace",
)

add_audio_to_video(
    video_file_id="video_id",
    audio_file_id=audio.file_id,
    mode="mix",
    audio_volume=0.8,
)
```

### Available Voices

`alloy`, `ash`, `coral`, `echo`, `fable`, `nova` (recommended), `onyx`, `sage`, `shimmer`

## Highlighted Captions (TikTok / Reels Style)

Add word-by-word highlighted captions that light up as spoken:

```python
burn_highlighted_captions(
    video_file_id="video_id",
    style="tiktok",
    highlight_color="#3B82F6",
    base_color="white",
    words_per_group=3,
    position="center",
)

burn_highlighted_captions(
    video_file_id="video_id",
    style="karaoke",
    highlight_color="yellow",
    base_color="white",
    background="black@0.6",
    words_per_group=4,
    position="bottom",
)
```

## Video Clipping

Extract segments from longer videos:

```python
clip_video(
    video_file_id="long_video_id",
    start_time=45.0,
    end_time=60.0,
)
```
