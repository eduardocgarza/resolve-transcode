# resolve-transcode

`resolve-transcode` converts OBS `.mkv` recordings into `.mov` files that import cleanly into DaVinci Resolve Free on Linux.

It is a small Bash wrapper around `ffmpeg`. The point is not convenience alone. It exists because OBS on Ubuntu can produce a perfectly valid recording that DaVinci Resolve Free on Linux still cannot fully read.

The default output is:

```text
Video: DNxHR HQ
Audio: PCM 16-bit
Container: MOV
```

That combination is much more Resolve-friendly than a typical OBS recording like:

```text
Video: H.264
Audio: AAC
Container: MKV
```

## The actual problem

OBS is not creating a broken file.

The problem is the specific format that OBS commonly creates on Ubuntu/NVIDIA:

```text
Container: MKV
Video:     H.264 or H.265
Audio:     AAC
```

On one real recording from this machine, `ffprobe` reported:

```text
Input:    Matroska / MKV
Video:    H.264 High, yuv420p, 1920x1080, 30 fps
Audio:    AAC LC, 48000 Hz, stereo
```

That is a normal OBS file. VLC can play it. `ffmpeg` can read it. The file itself is valid.

The problem is DaVinci Resolve Free on Linux:

1. **H.264/H.265 video is Studio-only on Linux.** In Blackmagic's DaVinci Resolve 20 supported codec list, the Linux section is listed as "Rocky Linux 8.6 (CUDA)". In that Linux table, the H.264 and H.265 rows for `mov`, `mkv`, and `mp4` say decode is **Studio Only** with GPU acceleration on NVIDIA graphics.
2. **AAC audio is not supported in the Linux audio table.** The Linux audio section lists `aac/m4a` under Advanced Audio Coding, but decode and encode are both marked `-`. For embedded audio in video containers, the Linux table lists Linear PCM, MP3, and FLAC based on container. It does not list AAC there.
3. **MKV is not the whole problem.** OBS recommends MKV because it is safer while recording. Blackmagic also lists MKV in the Linux table. The bigger problem is what is inside the MKV: H.264/H.265 video plus AAC audio.

So the failure is not:

```text
OBS made a bad file
```

The failure is:

```text
Resolve Free on Linux does not fully support the video/audio codecs inside the OBS file
```

That can show up as:

- the clip does not import,
- the clip imports but the audio is missing,
- the clip imports but playback/scrubbing is broken,
- remuxing from MKV to MP4 still does not fix it.

That is why remuxing is not enough. Changing `recording.mkv` to `recording.mp4` may change the container, but it does not change H.264/H.265 video or AAC audio.

If you use DaVinci Resolve Studio on Linux with supported NVIDIA hardware, the H.264/H.265 video problem may be different because Studio has extra decode support. This tool is mainly for the Resolve Free on Linux workflow, where converting to DNxHR video and Linear PCM audio gives Resolve a supported editing file.

`resolve-transcode` fixes the handoff by converting the file into editing-friendly media:

```text
H.264 or H.265 + AAC in MKV from OBS
        |
        v
DNxHR/PCM MOV for DaVinci Resolve
```

## Quick Setup

Clone this repo, then copy the command into `~/.local/bin`:

```bash
git clone git@github.com:eduardocgarza/resolve-transcode.git
cd resolve-transcode

mkdir -p ~/.local/bin
cp ./resolve-transcode ~/.local/bin/resolve-transcode
chmod +x ~/.local/bin/resolve-transcode
```

Confirm it is available:

```bash
command -v resolve-transcode
resolve-transcode --help
```

If `command -v` does not print `~/.local/bin/resolve-transcode`, make sure `~/.local/bin` is on your `PATH`.

For the full setup explanation, including how `PATH` works, how to fix shell lookup problems, and how to update the installed command, see [SETUP.md](./SETUP.md).

## Requirements

- Bash
- `ffmpeg`
- DaVinci Resolve on Linux
- An OBS recording, usually `.mkv`

Check for `ffmpeg`:

```bash
ffmpeg -version
```

## Usage

Convert one file and let the tool choose the output name:

```bash
resolve-transcode "recording.mkv"
```

This creates:

```text
recording_resolve.mov
```

Choose the output file:

```bash
resolve-transcode "recording.mkv" "recording_for_resolve.mov"
```

Or:

```bash
resolve-transcode -o "recording_for_resolve.mov" "recording.mkv"
```

Convert multiple files:

```bash
resolve-transcode *.mkv
```

Write the output to an external drive:

```bash
resolve-transcode --sq "recording.mkv" "/media/eduardo/your-drive/recording.mov"
```

Overwrite an existing output:

```bash
resolve-transcode -f "recording.mkv"
```

## Quality profiles

Default:

```bash
resolve-transcode --hq "recording.mkv"
```

`--hq` uses DNxHR HQ and `yuv422p`. This is the default. Use it when quality matters and file size is acceptable.

Smaller file:

```bash
resolve-transcode --sq "recording.mkv"
```

`--sq` uses DNxHR SQ and `yuv422p`. This is usually the practical choice for screen-recorded tutorials when the HQ files are too large.

10-bit / HDR source:

```bash
resolve-transcode --hqx "recording.mkv"
```

`--hqx` uses DNxHR HQX and `yuv422p10le`. Use it only when the source is actually 10-bit or HDR.

## What the command runs

The default conversion is equivalent to:

```bash
ffmpeg -hide_banner \
  -i "recording.mkv" \
  -c:v dnxhd \
  -profile:v dnxhr_hq \
  -pix_fmt yuv422p \
  -c:a pcm_s16le \
  "recording_resolve.mov"
```

The important pieces are:

- `-c:v dnxhd`: use FFmpeg's DNxHD/DNxHR encoder.
- `-profile:v dnxhr_hq`: choose DNxHR HQ by default.
- `-pix_fmt yuv422p`: write 8-bit 4:2:2 video for normal recordings.
- `-c:a pcm_s16le`: convert audio to uncompressed 16-bit PCM.
- `.mov`: use a container Resolve understands well.

## File size warning

The output files will usually be much larger than the original OBS files.

That is expected. H.264/H.265 are delivery and recording codecs. They are space efficient, but harder for editors. DNxHR is an editing codec. It takes more disk space so the editor can seek, scrub, and decode frames more reliably.

For longer tutorials, write the output directly to an external SSD:

```bash
resolve-transcode --sq "input.mkv" "/media/eduardo/external-ssd/input.mov"
```

## Troubleshooting

If the input is not found:

```text
resolve-transcode: input file not found
```

Check the path and quote filenames that contain spaces:

```bash
resolve-transcode "2026-06-09 17-22-19.mkv"
```

If the output already exists:

```text
resolve-transcode: output already exists
Use -f to overwrite it.
```

Use:

```bash
resolve-transcode -f "recording.mkv"
```

If the output is too large:

```bash
resolve-transcode --sq "recording.mkv"
```

If Resolve still has trouble:

1. Confirm the output is `.mov`, not `.mkv`.
2. Confirm the video codec is DNxHR.
3. Confirm the audio codec is PCM.

```bash
ffprobe -hide_banner "recording_resolve.mov"
```

## Research notes

The exact bottleneck that led to this tool is the Resolve Free on Linux codec support gap:

- OBS recommends MKV for safer recording and provides built-in remuxing when a different container is needed.
- OBS supports NVIDIA NVENC on Linux.
- A real OBS file from this workflow was H.264 video plus AAC audio inside MKV.
- Blackmagic's Resolve codec list treats Linux separately as "Rocky Linux 8.6 (CUDA)".
- In that Linux table, H.264/H.265 decode for `mov`, `mkv`, and `mp4` is marked Studio Only.
- In that Linux audio table, `aac/m4a` decode is marked unsupported.
- DNxHR video and Linear PCM audio are listed as supported in the Linux profile.
- Therefore the fix is to transcode the streams, not merely rename or remux the container.

Sources:

- OBS Standard Recording Output Guide: https://obsproject.com/kb/standard-recording-output-guide
- OBS Hardware Encoding: https://obsproject.com/kb/hardware-encoding
- Blackmagic DaVinci Resolve 20 Supported Formats and Codecs: https://documents.blackmagicdesign.com/SupportNotes/DaVinci_Resolve_20_Supported_Codec_List.pdf

## License

See [LICENSE](./LICENSE).
