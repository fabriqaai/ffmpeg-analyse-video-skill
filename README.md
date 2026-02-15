# ffmpeg-analyse-video-skill

AI agent skill that analyses video content by extracting frames with ffmpeg and using vision to generate timestamped step-by-step summaries.

Works with screen recordings, tutorials, presentations, footage, and animations.

## Install

```bash
npx skills add fabriqaai/ffmpeg-analyse-video-skill
```

## Requirements

- **ffmpeg** and **ffprobe** installed on your system
  - macOS: `brew install ffmpeg`
  - Ubuntu/Debian: `sudo apt install ffmpeg`
  - Windows: `choco install ffmpeg` or `winget install ffmpeg`

## Supported AI Agents

| Agent | Sub-Agent Support | Parallel Batches |
|-------|------------------|-----------------|
| Claude Code | Task tool (`general-purpose`) | Yes |
| Codex | Sub-task delegation | Yes |
| Cursor | Background / composer agent | Yes |
| Gemini | Tool-call sub-agents | Yes |
| Aider | `/run` / architect mode | Sequential |

Falls back to direct frame reading (capped at 20 frames) if no sub-agent mechanism is available.

## Usage

Provide a video file path and ask your agent to analyse it:

```
Analyse this video: /path/to/recording.mp4
```

```
What happens in this video? ~/Desktop/demo.mov
```

```
Summarise this recording: ./tutorial.mp4
```

### Advanced

```
Analyse 2:00 to 5:00 of meeting.mp4
```

```
Analyse this video in high detail: demo.mp4
```

```
Focus on the code shown in this video: screencast.mp4
```

## How It Works

```
Main Agent                          Sub-Agents (disposable context)
──────────                          ──────────────────────────────
1. ffprobe metadata        ───►
2. ffmpeg frame extraction ───►
3. Split frames into batches ──►   4. Read images (vision)
                                      Write text descriptions
                                      to batch_N_analysis.md
5. Read text files only    ◄───    (context discarded)
6. Synthesise final output
```

Frame images are only ever read inside disposable sub-agent contexts. The main agent receives lightweight text-only analysis files — no images enter the main conversation. This cuts context usage by ~90% compared to reading frames directly.

### Frame Extraction Strategies

| Video Duration | Strategy | Expected Frames |
|---------------|----------|-----------------|
| 0-60s | Interval (1 frame/2s) | 1-30 |
| 1-10min | Scene detection | 15-60 |
| 10-30min | Keyframe extraction | 30-80 |
| 30min+ | Thumbnail filter | Capped at 60 |

### Output Format

The skill produces a structured markdown report:

- **Metadata** — duration, resolution, fps, content type, frames analysed
- **Timeline** — chronological segments with descriptions
- **Key Moments** — 3-7 most significant timestamps
- **Summary** — 2-5 sentence narrative

## License

MIT
