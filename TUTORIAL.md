# Tutorial Technical Spec

## Goal

Explain the Linux OBS -> DaVinci Resolve Free problem technically, then show the manual `ffmpeg` fix and the `resolve-transcode` wrapper command.

Core conversion:

```text
OBS recording:  MKV container + H.264/H.265 video + AAC audio
Resolve input:  MOV container + DNxHR video + Linear PCM audio
```

## 1. The Problem

OBS can create a valid recording that DaVinci Resolve Free on Linux still cannot fully use.

This is a real codec support problem, not just a convenience issue.

Typical OBS recording on Ubuntu/NVIDIA:

```text
Container: MKV
Video:     H.264 or H.265, often from NVIDIA NVENC
Audio:     AAC
```

Example from a real recording:

```bash
ffprobe -hide_banner "/home/eduardo/0-tutorial-videos/setup-domain.mkv"
```

Observed media:

```text
Input:  Matroska / MKV
Video:  H.264 High, yuv420p, 1920x1080, 30 fps
Audio:  AAC LC, 48000 Hz, stereo
```

That file is valid. VLC can play it and `ffmpeg` can read it. The issue is DaVinci Resolve Free on Linux.

## 2. What DaVinci Resolve Free On Linux Does Not Support

Blackmagic's DaVinci Resolve supported codec document separates Linux from macOS and Windows. The Linux section is listed as:

```text
Rocky Linux 8.6 (CUDA)
```

In that Linux table:

- H.264 decode for `mov`, `mkv`, and `mp4` is marked **Studio Only**.
- H.265 decode for `mov`, `mkv`, and `mp4` is marked **Studio Only**.
- `aac/m4a` audio decode is marked unsupported.
- Embedded audio support lists Linear PCM, MP3, and FLAC based on container; it does not list AAC.
- DNxHR video is supported.
- Linear PCM audio is supported.

So the exact problem is:

```text
Resolve Free on Linux cannot fully decode the common OBS output:
H.264/H.265 video + AAC audio
```

Symptoms can include:

- clip does not import,
- clip imports with missing audio,
- clip imports but playback/scrubbing is unreliable,
- remuxing MKV to MP4 still does not fix it.

## 3. MKV Is Not The Main Problem

OBS recommends MKV because it is safer for recording. If OBS or the system crashes, MKV is less likely to lose the whole recording.

Blackmagic's Linux table also lists MKV as a container. So the container alone is not the core issue.

The problem is inside the container:

```text
MKV container
  -> H.264/H.265 video stream
  -> AAC audio stream
```

Remuxing changes the container but keeps the same video and audio streams:

```text
Remux:     input.mkv -> input.mp4
Streams:   H.264/AAC stays H.264/AAC
```

That may not solve the Resolve Free on Linux problem.

Transcoding changes the actual codecs:

```text
Transcode: H.264/AAC -> DNxHR/PCM
Container: MKV -> MOV
```

That is what this tool does.

## 4. Manual FFmpeg Solution

The manual conversion is:

```bash
ffmpeg -hide_banner \
  -i "input.mkv" \
  -c:v dnxhd \
  -profile:v dnxhr_hq \
  -pix_fmt yuv422p \
  -c:a pcm_s16le \
  "output.mov"
```

Technical meaning:

- `-i "input.mkv"` reads the OBS recording.
- `-c:v dnxhd` uses FFmpeg's DNxHD/DNxHR encoder.
- `-profile:v dnxhr_hq` selects DNxHR HQ.
- `-pix_fmt yuv422p` writes 8-bit 4:2:2 video.
- `-c:a pcm_s16le` converts AAC audio to 16-bit little-endian Linear PCM.
- `output.mov` writes a QuickTime/MOV file for Resolve.

Result:

```text
Input:   H.264/H.265 video + AAC audio in MKV
Output:  DNxHR video + Linear PCM audio in MOV
```

## 5. Why DNxHR And PCM

DNxHR is an editing/intermediate codec. It is much larger than H.264/H.265, but it is designed for post-production workflows.

Linear PCM audio is uncompressed audio. Resolve's Linux codec table supports PCM, while AAC decode is not supported in the Linux audio table.

The tradeoff:

```text
Smaller recording file:   H.264/H.265 + AAC
Better Resolve input:     DNxHR + PCM
```

The converted file will usually be much larger. That is expected.

## 6. Standalone Command

`resolve-transcode` is the standalone wrapper around the `ffmpeg` command.

Instead of typing:

```bash
ffmpeg -hide_banner \
  -i "input.mkv" \
  -c:v dnxhd \
  -profile:v dnxhr_hq \
  -pix_fmt yuv422p \
  -c:a pcm_s16le \
  "output.mov"
```

Use:

```bash
resolve-transcode "input.mkv" "output.mov"
```

## 7. Command Reference

Create a Resolve-ready MOV beside the input:

```bash
resolve-transcode "input.mkv"
```

Output:

```text
input_resolve.mov
```

Choose the output path:

```bash
resolve-transcode "input.mkv" "output.mov"
```

Equivalent output form:

```bash
resolve-transcode -o "output.mov" "input.mkv"
```

Convert multiple files:

```bash
resolve-transcode "file1.mkv" "file2.mkv" "file3.mkv"
```

Overwrite an existing output:

```bash
resolve-transcode -f "input.mkv"
```

Write directly to an external SSD:

```bash
resolve-transcode --sq "input.mkv" "/media/eduardo/external-ssd/input.mov"
```

Show help:

```bash
resolve-transcode --help
```

## 8. Quality Profiles

Default / high quality:

```bash
resolve-transcode --hq "input.mkv"
```

Technical settings:

```text
Video profile: dnxhr_hq
Pixel format:  yuv422p
Audio:         pcm_s16le
```

Use this when quality matters and file size is acceptable.

Smaller Resolve file:

```bash
resolve-transcode --sq "input.mkv"
```

Technical settings:

```text
Video profile: dnxhr_sq
Pixel format:  yuv422p
Audio:         pcm_s16le
```

Use this for long screen-recorded tutorials when DNxHR HQ is too large.

10-bit / HDR source:

```bash
resolve-transcode --hqx "input.mkv"
```

Technical settings:

```text
Video profile: dnxhr_hqx
Pixel format:  yuv422p10le
Audio:         pcm_s16le
```

Use this only when the source is actually 10-bit or HDR.

## 9. How The Wrapper Works Internally

The script starts with these defaults:

```text
profile = dnxhr_hq
pix_fmt = yuv422p
force   = false
```

Flag mapping:

```text
--hq   -> -profile:v dnxhr_hq  -pix_fmt yuv422p
--sq   -> -profile:v dnxhr_sq  -pix_fmt yuv422p
--hqx  -> -profile:v dnxhr_hqx -pix_fmt yuv422p10le
```

Overwrite behavior:

```text
default     -> ffmpeg -n  (do not overwrite)
-f/--force  -> ffmpeg -y  (overwrite)
```

Output path behavior:

```text
resolve-transcode input.mkv
  -> input_resolve.mov

resolve-transcode input.mkv output.mov
  -> output.mov

resolve-transcode -o output.mov input.mkv
  -> output.mov
```

The script exits before running `ffmpeg` if:

- no input file is provided,
- the input file does not exist,
- `-o/--output` is used with more than one input,
- the output file already exists and `-f` was not used.

## 10. Validate The Before And After

Inspect the OBS file:

```bash
ffprobe -hide_banner "input.mkv"
```

Expected OBS-style input:

```text
Video: H.264 or H.265
Audio: AAC
Container: MKV
```

Convert:

```bash
resolve-transcode --sq "input.mkv" "input_for_resolve.mov"
```

Inspect the output:

```bash
ffprobe -hide_banner "input_for_resolve.mov"
```

Expected Resolve-friendly output:

```text
Video: DNxHR
Audio: PCM signed 16-bit little-endian
Container: MOV
```

## 11. Recommended Workflow

```text
1. Record in OBS as MKV.
2. Inspect the file with ffprobe if needed.
3. Convert with resolve-transcode.
4. Import the generated MOV into DaVinci Resolve.
5. Edit in Resolve.
6. Export the final delivery file from Resolve.
```

## 12. Sources

OBS recording format guidance:

```text
https://obsproject.com/kb/standard-recording-output-guide
```

OBS hardware encoding / NVENC:

```text
https://obsproject.com/kb/hardware-encoding
```

DaVinci Resolve supported codecs:

```text
https://documents.blackmagicdesign.com/SupportNotes/DaVinci_Resolve_20_Supported_Codec_List.pdf
```

## 13. One-Line Summary

`resolve-transcode` converts valid OBS recordings that Resolve Free on Linux cannot fully decode into Resolve-friendly DNxHR/PCM MOV files.
