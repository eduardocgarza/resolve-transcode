# resolve-transcode

`resolve-transcode` converts OBS `.mkv` recordings into `.mov` files that import cleanly into DaVinci Resolve on Linux.

It is a small Bash wrapper around `ffmpeg`. The point is not convenience alone. It exists because OBS on Ubuntu can produce perfectly valid recordings that are still a bad fit for DaVinci Resolve Free on Linux.

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

## Why this exists

OBS is good at recording. On Linux, OBS can use NVIDIA NVENC to record H.264 or H.265 with low CPU usage. OBS also recommends MKV for safer recording because MKV is less likely to corrupt the whole recording if OBS or the system crashes.

The bottleneck is what happens next.

DaVinci Resolve on Linux does not have the same codec support as Resolve on macOS or Windows. Blackmagic's supported codec list separates Linux under "Rocky Linux 8.6 (CUDA)", and the Linux table marks H.264/H.265 support differently than macOS and Windows. In practical terms, the free Linux version is not a safe target for OBS-style H.264/AAC recordings, especially when those recordings are coming from NVIDIA/NVENC workflows.

On one real recording from this machine, `ffprobe` reported:

```text
Input:    Matroska / MKV
Video:    H.264 High, yuv420p, 1920x1080, 30 fps
Audio:    AAC LC, 48000 Hz, stereo
```

That is a normal OBS file. It is not a "bad" file. The mismatch is between the recording format and Resolve Free on Linux.

`resolve-transcode` fixes the handoff by converting the file into editing-friendly media:

```text
H.264/AAC MKV from OBS
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

The bottleneck that led to this tool is the Linux Resolve handoff:

- OBS recommends MKV for safer recording and provides built-in remuxing when a different container is needed.
- OBS supports NVIDIA NVENC on Linux.
- Blackmagic's Resolve codec list treats Linux separately as "Rocky Linux 8.6 (CUDA)".
- On Linux, H.264/H.265 support is not equivalent to macOS/Windows and is tied to Studio/NVIDIA support in the codec list.
- AAC audio is not a good target for Resolve Linux; PCM is safer.
- DNxHR video and PCM audio are listed as supported formats for Resolve's Linux profile.

Sources:

- OBS Standard Recording Output Guide: https://obsproject.com/kb/standard-recording-output-guide
- OBS Hardware Encoding: https://obsproject.com/kb/hardware-encoding
- Blackmagic DaVinci Resolve 20 Supported Formats and Codecs: https://documents.blackmagicdesign.com/SupportNotes/DaVinci_Resolve_20_Supported_Codec_List.pdf

## License

See [LICENSE](./LICENSE).
