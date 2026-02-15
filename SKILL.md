---
name: ffmpeg-analyse-video
description: Analyse video content by extracting frames with ffmpeg and using AI vision
  to generate timestamped step-by-step summaries. Use when user provides a video file
  and wants to understand its visual content — screen recordings, tutorials, presentations,
  footage, or animations. Triggers on "analyse this video", "what happens in this video",
  "summarise this recording", or any request involving understanding video file contents.
---

# FFmpeg Video Analysis

Extract frames from video files with ffmpeg, view them with the Read tool, and produce a structured timestamped summary.

## 1. Prerequisites

Run before anything else:

```bash
which ffmpeg && which ffprobe
```

If either is missing, show platform-specific install instructions and STOP:
- **macOS**: `brew install ffmpeg`
- **Ubuntu/Debian**: `sudo apt install ffmpeg`
- **Windows**: `choco install ffmpeg` or `winget install ffmpeg`

## 2. Setup Temp Directory

Detect platform and create a working directory:

```bash
# macOS/Linux
TMPDIR="/tmp/video-analysis-$(date +%s)"
mkdir -p "$TMPDIR"

# Windows (PowerShell)
# $TMPDIR = "$env:TEMP\video-analysis-$(Get-Date -UFormat %s)"
# New-Item -ItemType Directory -Path $TMPDIR
```

## 3. Extract Video Metadata

```bash
ffprobe -v quiet -print_format json -show_format -show_streams "VIDEO_PATH"
```

Extract and report: duration, resolution (width x height), fps, codec, file size, whether audio is present.

If no video stream is found, report "audio-only file" and STOP.
If file size > 2GB, warn the user and suggest analysing a time range with `-ss START -to END`.

## 4. Extract Frames

Choose strategy based on duration:

| Duration | Strategy | Command |
|----------|----------|---------|
| 0-60s | 1 frame every 2s | `ffmpeg -hide_banner -y -i INPUT -vf "fps=1/2,scale='min(1280,iw)':-2" -q:v 5 DIR/frame_%04d.jpg` |
| 1-10min | Scene detection (threshold 0.3) | `ffmpeg -hide_banner -y -i INPUT -vf "select='gt(scene,0.3)',scale='min(1280,iw)':-2" -vsync vfr -q:v 5 DIR/scene_%04d.jpg` |
| 10-30min | Keyframe extraction | `ffmpeg -hide_banner -y -skip_frame nokey -i INPUT -vf "scale='min(1280,iw)':-2" -vsync vfr -q:v 5 DIR/key_%04d.jpg` |
| 30min+ | Thumbnail filter | `ffmpeg -hide_banner -y -i INPUT -vf "thumbnail=SEGMENT_FRAMES,scale='min(1280,iw)':-2" -vsync vfr -q:v 5 DIR/thumb_%04d.jpg` |

For thumbnail filter, calculate `SEGMENT_FRAMES = total_frames / 60` to cap output at ~60 frames.

**Fallbacks:**
- Scene detection yields 0 frames → retry with interval at 1 frame/5s
- More than 100 frames extracted → subsample evenly to 80
- Frame extraction fails → try the next simpler strategy (scene → interval, keyframe → interval)

**Time range analysis:** When user specifies a range, prepend `-ss START -to END` before `-i`.
**Higher detail mode:** If requested, double the fps rate and lower scene threshold to 0.2.

After extraction, list frames and calculate each frame's timestamp from its sequence number and the extraction rate.

## 5. Analyse Frames

Process frames in batches of 5-8 using the Read tool to view each image.

For each frame describe:
- Scene content (what is visible)
- Actions or state (what is happening)
- Text or UI elements (any readable text, buttons, menus, code)
- Transition from previous frame (what changed)

After the first batch, classify the content type as one of:
- **Screencast** — software demo, coding session → read text/UI elements carefully
- **Presentation** — slides → capture slide titles and key bullet points
- **Tutorial** — instructional with steps → identify each step
- **Footage** — real-world video → describe scenes and actions
- **Animation** — motion graphics → describe visual elements and transitions

Adjust analysis depth for subsequent batches based on content type.

If a frame fails to load, skip it and note the gap in the timeline.

## 6. Synthesise Output

Group analysed frames into natural segments (same scene, slide, or screen). Identify 3-7 key moments. Write a 2-5 sentence narrative summary.

Format the output as:

```markdown
# Video Analysis: [filename]

## Metadata
| Property | Value |
|----------|-------|
| Duration | M:SS |
| Resolution | WxH |
| FPS | N |
| Content Type | [detected] |
| Frames Analysed | N |

## Timeline
### [Segment Title] (M:SS - M:SS)
Description of what happens in this segment.

### [Segment Title] (M:SS - M:SS)
Description of what happens in this segment.

## Key Moments
1. **[M:SS] Title**: Description
2. **[M:SS] Title**: Description
3. **[M:SS] Title**: Description

## Summary
[2-5 sentence narrative paragraph summarising the entire video]
```

## 7. Cleanup

Remove the temp directory after output is complete:

```bash
# macOS/Linux
rm -rf "$TMPDIR"

# Windows (PowerShell)
# Remove-Item -Recurse -Force $TMPDIR
```

Skip cleanup if the user asks to keep frames.

## Advanced Options

- **Time range**: "Analyse 2:00 to 5:00 of video.mp4" → use `-ss 120 -to 300`
- **Higher detail**: "Analyse in high detail" → double frame rate, lower scene threshold to 0.2
- **Focus area**: "Focus on the code shown" → prioritise text/code extraction in analysis
- **Sprite sheet**: For a visual overview, generate a contact sheet:
  ```bash
  ffmpeg -hide_banner -y -i INPUT -vf "select='not(mod(n,EVERY_N))',scale='min(320,iw)':-2,tile=5xROWS" -frames:v 1 DIR/sprite.jpg
  ```
