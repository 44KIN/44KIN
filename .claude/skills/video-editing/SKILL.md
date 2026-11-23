---
name: video-editing
description: Edit videos from the command line. Use when the user wants to cut, trim, merge, resize or export video, add subtitles/captions, download a reference video, or build motion graphics or programmatic video. Uses FFmpeg, Whisper, yt-dlp, Remotion and HyperFrames.
---

# Video editing toolkit

| Tool | Job | Check |
|------|-----|-------|
| FFmpeg | cuts, merges, resizing, audio, exports | `ffmpeg -version` |
| Whisper | transcription and subtitles (SRT/VTT) | `whisper --help` |
| yt-dlp | download reference videos or audio | `yt-dlp --version` |
| Remotion | programmatic video in React (`npx remotion`) | `npx remotion versions` |
| HyperFrames | graphics: write HTML, render MP4 (`npx hyperframes`) | `npx hyperframes --help` |

## Setup (run if a tool is missing)

```bash
apt-get install -y ffmpeg            # or: brew install ffmpeg
pip install yt-dlp openai-whisper
npx create-video@latest              # new Remotion project
npm i -g hyperframes                 # optional; npx works too
```

## Workflow

1. Work on copies. Keep the original file untouched and write outputs to a new folder, such as `out/`.
2. Get the source: `yt-dlp -f "bv*+ba/b" --merge-output-format mp4 -o "ref/%(title)s.%(ext)s" URL`. Only download content the user has the right to use.
3. Transcribe: `whisper input.mp4 --model small --output_format srt --output_dir out/`. Use `--model base` for speed and `--model medium` for accuracy.
4. Cut, trim or merge with FFmpeg (recipes below).
5. For animated titles, lower thirds or overlays, build them in Remotion or HyperFrames, render to MP4, then composite with FFmpeg.
6. Verify the result with `ffprobe` (duration, resolution, streams). Extract a frame to check it visually: `ffmpeg -ss 5 -i out.mp4 -frames:v 1 frame.png`.

## FFmpeg recipes

```bash
# Trim (re-encode, frame-accurate)
ffmpeg -ss 00:00:10 -to 00:00:25 -i in.mp4 -c:v libx264 -c:a aac out/clip.mp4
# Trim without re-encoding (fast, cuts on keyframes)
ffmpeg -ss 00:00:10 -to 00:00:25 -i in.mp4 -c copy out/clip.mp4
# Concatenate clips (list.txt lines: file 'a.mp4')
ffmpeg -f concat -safe 0 -i list.txt -c copy out/merged.mp4
# Burn in subtitles
ffmpeg -i in.mp4 -vf subtitles=out/in.srt out/subbed.mp4
# Vertical 9:16 crop for Shorts/Reels/TikTok
ffmpeg -i in.mp4 -vf "crop=ih*9/16:ih,scale=1080:1920" out/vertical.mp4
# Extract audio / speed up / compress
ffmpeg -i in.mp4 -vn -c:a libmp3lame out/audio.mp3
ffmpeg -i in.mp4 -filter_complex "[0:v]setpts=PTS/1.25[v];[0:a]atempo=1.25[a]" -map "[v]" -map "[a]" out/fast.mp4
ffmpeg -i in.mp4 -c:v libx264 -crf 23 -preset medium -c:a aac out/small.mp4
```

## Notes

- Large files do not belong in git. Add `out/` and `ref/` to `.gitignore`.
- Whisper downloads its model on first run, so it needs network access to the model host.
- Remotion renders: `npx remotion render <CompositionId> out/video.mp4`. It needs Chromium (`npx remotion browser ensure`).
