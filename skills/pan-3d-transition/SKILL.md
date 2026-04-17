---
name: pan-3d-transition
description: Create 3D pan/swivel transition effects for videos using Remotion. Use when user asks to add 3D transitions, create swivel effects, or add video transitions.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# 3D Pan Transition

## Goal
Create 3D rotating "swivel" transition effects for videos using Remotion rendering. The effect inserts a fast-forward teaser of later video content at a specified point, with a 3D rotation animation, while preserving the original audio continuously throughout.

## Scripts
- `./scripts/insert_3d_transition.py` - Insert transition into video

## Usage

```bash
python3 ./scripts/insert_3d_transition.py input.mp4 output.mp4 \
  --insert-at 3 \
  --duration 5 \
  --teaser-start 60 \
  --bg-image .tmp/bg.png
```

## Parameters
| Argument | Default | Description |
|----------|---------|-------------|
| `--insert-at` | 3 | Where to insert the teaser (seconds from start) |
| `--duration` | 5 | How long the swivel teaser plays (seconds) |
| `--teaser-start` | 60 | Where in the video to sample teaser content from (seconds) |
| `--bg-color` | `#2d3436` | Background fill color (hex) when no bg-image is provided |
| `--bg-image` | none | Path to background image (overrides bg-color) |

## How It Works
1. Reads video dimensions and duration using `ffprobe`
2. Calculates playback speed: content from `--teaser-start` to end is compressed into `--duration` seconds (capped at 100x)
3. Extracts frames from the teaser region at the calculated interval
4. Copies a background image (or generates a solid color one) into the Remotion `public/frames/` directory
5. Renders a 3D rotating animation at 60fps using Remotion (`Pan3D` composition in `src/dynamic-index.ts`)
6. Splits the original video into three segments: before insert point, transition, after insert point
7. Concatenates the three video segments (no audio in any segment)
8. Re-attaches the original audio track using `-shortest` so audio and video durations match

**Timeline result:**
```
Video: [0 → insert_at] [swivel teaser, duration seconds] [insert_at+duration → end]
Audio: [original audio plays continuously throughout, unmodified]
```

**Encoding:** Uses Apple `hevc_videotoolbox` hardware encoder at 17Mbps if available, falls back to `libx265` software encoder at CRF 18. Output is always H.265/HEVC at 30fps.

## Expected Output
The output file is an H.265 MP4 at 30fps. The video is longer than the input by exactly `--duration` seconds. Audio sync is preserved because the original audio is extracted separately and merged last with `-shortest`.

Example with defaults (3s insert, 5s teaser):
- 10-minute input → 10:05 output
- Swivel teaser plays from 0:03 to 0:08, sourcing content from 1:00 onward at ~119x speed

## Dependencies
```bash
cd video_effects && npm install
```

Also requires:
- `ffmpeg` and `ffprobe` on PATH
- Node.js with `npx` available

Verify ffmpeg is available:
```bash
ffmpeg -version
ffprobe -version
```

## Previewing Before Final Render

To check what the teaser will look like without rendering the full output, render just the Remotion composition directly:
```bash
cd video_effects
npx remotion render src/dynamic-index.ts Pan3D preview_transition.mp4 \
  --props '{"frameCount": 300, "playbackRate": 10}'
```

This skips the ffmpeg split/concat steps and just renders the 3D animation with whatever frames are already in `public/frames/`.

## Workflow Context

Typical use case: a long-form tutorial or explainer video where you want to hook viewers early by showing a preview of what comes later. Insert the teaser in the first 3-10 seconds of the video, sourcing from the most compelling part (often the 1-minute mark or beyond).

The script is self-contained and can be run on any video regardless of resolution or frame rate. Input resolution is detected automatically.

## Troubleshooting

**"Teaser start exceeds video duration" error:** `--teaser-start` must be less than the total video length. Use a smaller value.

**"Insert point + duration exceeds video duration" error:** The insert point plus the teaser duration must fit within the original video length. Reduce `--insert-at` or `--duration`.

**Remotion render fails:** Make sure `cd video_effects && npm install` was run. Check that Node.js is installed (`node --version`). If the `public/frames/` directory is empty, the frame extraction step failed — look for ffmpeg errors above the Remotion step in the output.

**Hardware encoding not available:** The script falls back to software encoding automatically. Software encoding is slower but produces identical quality output.

**Audio out of sync:** Verify the output duration matches input + `--duration`. If ffmpeg's `-shortest` flag is trimming the audio early, check that the concat_video.mp4 intermediate file has the expected duration.

**Black background instead of custom image:** Confirm the path passed to `--bg-image` exists before running. The script copies the file to `video_effects/public/frames/bg_image.png` — if the source path is wrong it silently falls back to a solid color.
