# ffmpeg transcoding & media editing

A practical reference for common ffmpeg tasks: inspecting media, transcoding, trimming, scaling, extracting audio, concatenating, generating GIFs, and using hardware encoders. Each section shows a known-good invocation you can adapt.

## 1. Inspect a media file

Use `ffprobe` for structured metadata, or `ffmpeg -i` for a quick human-readable summary.

```bash
ffprobe -hide_banner input.mp4
```

```bash
ffprobe -v error -show_format -show_streams -of json input.mp4
```

```bash
ffmpeg -hide_banner -i input.mp4
```

## 2. Simple transcode

Re-encode video to H.264 and audio to AAC. Defaults are sensible for MP4 distribution.

```bash
ffmpeg -i input.mp4 -c:v libx264 -c:a aac -b:a 128k output.mp4
```

## 3. Quality-targeted transcode (CRF)

Constant Rate Factor is the recommended quality knob for x264/x265. Lower CRF = higher quality. 18–28 is the useful range; 23 is the default.

```bash
ffmpeg -i input.mp4 -c:v libx264 -crf 23 -preset medium -c:a aac -b:a 128k output.mp4
```

For x265 (HEVC), CRF defaults to around 28 — it produces smaller files at equivalent visual quality:

```bash
ffmpeg -i input.mp4 -c:v libx265 -crf 28 -preset medium -c:a aac -b:a 128k output.mp4
```

## 4. Trim / clip

Fast seek (`-ss` before `-i`) is quick but cuts at the nearest keyframe — combined with `-c copy` it produces a lossless clip without re-encoding.

```bash
ffmpeg -ss 00:00:10 -i input.mp4 -t 00:00:30 -c copy output.mp4
```

Slow seek (`-ss` after `-i`) is frame-accurate but re-encodes:

```bash
ffmpeg -i input.mp4 -ss 00:00:10 -t 00:00:30 -c:v libx264 -crf 23 -c:a aac output.mp4
```

## 5. Extract audio

Stream-copy the audio track to its own file — no re-encoding, original quality preserved.

```bash
ffmpeg -i input.mp4 -vn -c:a copy output.m4a
```

To force MP3 output, re-encode the audio:

```bash
ffmpeg -i input.mp4 -vn -c:a libmp3lame -b:a 192k output.mp3
```

## 6. Change container without re-encoding

If the codecs are already compatible with the target container, you can re-mux instantly with no quality loss.

```bash
ffmpeg -i input.mkv -c copy output.mp4
```

## 7. Resize / scale

Scale to a fixed width and let height be calculated. `-2` keeps the aspect ratio and rounds to an even number (required by H.264).

```bash
ffmpeg -i input.mp4 -vf scale=1280:-2 -c:v libx264 -crf 23 -preset medium -c:a copy output.mp4
```

## 8. Concatenate files

Use the concat demuxer when all inputs share the same codecs, resolution, and framerate. First write a list file:

```bash
cat > files.txt <<'EOF'
file 'part1.mp4'
file 'part2.mp4'
file 'part3.mp4'
EOF
```

Then concatenate with stream copy:

```bash
ffmpeg -f concat -safe 0 -i files.txt -c copy output.mp4
```

## 9. Generate an animated GIF (palette workflow)

Two-pass palette workflow produces dramatically better GIFs than the default 256-colour quantiser.

```bash
ffmpeg -ss 00:00:10 -t 00:00:30 -i input.mp4 -vf "fps=15,scale=1280:-2:flags=lanczos,palettegen" -y palette.png
```

```bash
ffmpeg -ss 00:00:10 -t 00:00:30 -i input.mp4 -i palette.png -filter_complex "fps=15,scale=1280:-2:flags=lanczos[x];[x][1:v]paletteuse" -y output.gif
```

## 10. Hardware encoder (macOS VideoToolbox)

On Apple Silicon / Intel Macs, `h264_videotoolbox` and `hevc_videotoolbox` offload encoding to the GPU. Quality per bit is lower than libx264 but encoding is much faster.

```bash
ffmpeg -i input.mp4 -c:v h264_videotoolbox -b:v 2M -c:a aac -b:a 128k output.mp4
```

For HEVC:

```bash
ffmpeg -i input.mp4 -c:v hevc_videotoolbox -b:v 2M -tag:v hvc1 -c:a aac -b:a 128k output.mp4
```
