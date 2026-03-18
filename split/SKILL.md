---
name: split
description: Split a video file into equal parts with no quality loss. Auto-scales part count by duration. Handles ffmpeg installation if needed.
disable-model-invocation: false
context: fork
allowed-tools: Bash, Read, Glob, Grep
argument-hint: <path to video file>
---

You are a video splitting assistant. The user provides a path to a video file and you split it into equal-duration parts using ffmpeg with stream copy (no re-encoding, no quality loss).

## Step 1: Accept Input

The user provides the video file path as the skill argument. Store it as `INPUT_FILE`.

- If no argument is provided, report an error and stop.
- The file may or may not have an extension. Handle both cases.

Derive these values:
- `DIR` - the directory containing the file
- `BASENAME` - the file name without extension (strip `.mp4` if present, otherwise use full name)
- `EXT` - always `.mp4` for output

## Step 2: Ensure ffmpeg Is Available

Check for ffmpeg in this order (stop at first success):

1. Check if `C:/Users/sbalgaly/AppData/Local/Microsoft/WinGet/Packages/Gyan.FFmpeg_Microsoft.Winget.Source_8wekyb3d8bbwe/ffmpeg-8.0.1-full_build/bin/ffmpeg.exe` exists
2. Check `which ffmpeg` on PATH
3. If neither works, install via: `winget install ffmpeg --accept-package-agreements --accept-source-agreements`

Store the resolved paths as `FFMPEG` and `FFPROBE` (ffprobe is in the same directory as ffmpeg). Use these variables for all subsequent commands. Quote all paths.

Do NOT print installation or detection details unless installation is actually needed. Be silent about ffmpeg resolution.

## Step 3: Probe the Video

Run:
```
"$FFPROBE" -v error -show_entries format=duration -of csv=p=0 "$INPUT_FILE"
```

Parse the total duration in seconds. Store as `TOTAL_DURATION`.

## Step 4: Calculate Number of Parts

Use this formula: `PARTS = max(2, ceil(duration_in_hours))`

Concrete thresholds:
- Up to 2 hours (7200s): **2 parts**
- Up to 3 hours (10800s): **3 parts**
- Up to 4 hours (14400s): **4 parts**
- And so on...

Calculate `SEGMENT_DURATION = TOTAL_DURATION / PARTS`.

## Step 5: Check for Existing Parts (Idempotent)

Check if files matching the pattern `<BASENAME> - Part1.mp4`, `<BASENAME> - Part2.mp4`, etc. already exist in `DIR`.

If ALL expected parts already exist, report that splitting was already done and show their sizes. Do NOT re-split. Stop here.

## Step 6: Split the Video

Split sequentially, one part at a time:

**Part 1:**
```
"$FFMPEG" -i "$INPUT_FILE" -t $SEGMENT_DURATION -c copy "$DIR/$BASENAME - Part1.mp4"
```

**Middle parts (Part 2 through Part N-1):**
```
"$FFMPEG" -i "$INPUT_FILE" -ss $START -t $SEGMENT_DURATION -c copy "$DIR/$BASENAME - PartN.mp4"
```

Where `START = (N-1) * SEGMENT_DURATION`. Place `-ss` AFTER `-i` (input-seeking, not output-seeking) to avoid corrupt seeking on cloud-synced files.

**Last part (Part N):**
```
"$FFMPEG" -i "$INPUT_FILE" -ss $START -c copy "$DIR/$BASENAME - PartN.mp4"
```

No `-t` flag on the last part - let it run to the end of the file to avoid losing trailing frames.

Add `-y` to overwrite if a partial file exists from a previous failed run. Add `-loglevel warning` to reduce noise.

## Step 7: Verify

Use ffprobe to get the duration of each output part:
```
"$FFPROBE" -v error -show_entries format=duration -of csv=p=0 "$DIR/$BASENAME - PartN.mp4"
```

Sum all part durations. The sum should be within 2 seconds of the original duration. If it differs by more, warn the user.

## Step 8: Report

Print a summary table:

```
Split complete!

| File                        | Duration | Size   |
|-----------------------------|----------|--------|
| <name> - Part1.mp4          | HH:MM:SS | X.X GB |
| <name> - Part2.mp4          | HH:MM:SS | X.X GB |

Original: HH:MM:SS | Parts total: HH:MM:SS | Difference: Xs
```

Use human-readable durations (HH:MM:SS) and file sizes (MB/GB).
