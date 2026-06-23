# YouTube Tutorial Notes

Working title:

```text
Fix OBS Recordings for DaVinci Resolve on Linux with resolve-transcode
```

## Core message

This tool is not just a shortcut.

It solves a real handoff problem between OBS on Ubuntu/Linux and DaVinci Resolve Free on Linux. OBS can produce a valid H.264/AAC MKV file, especially when using NVIDIA NVENC. The problem is not that the OBS file is broken. The problem is that Resolve Free on Linux does not fully support the video and audio codecs inside that file.

The fix is to transcode the OBS recording into an editing-friendly file:

```text
OBS recording:     H.264 video + AAC audio in MKV
Resolve file:      DNxHR video + PCM audio in MOV
```

## Audience

This video is for:

- Linux users recording tutorials with OBS.
- Ubuntu users editing in DaVinci Resolve.
- People with NVIDIA GPUs using NVENC.
- People who can see their file in VLC but cannot get it working correctly in Resolve.
- People who do not want to remember a long `ffmpeg` command every time.

## What to show on screen

Prepare:

- Terminal open in a folder with an OBS `.mkv` recording.
- DaVinci Resolve installed.
- This repo open in a code editor or GitHub page.
- A short sample `.mkv` file.
- Enough disk space, ideally on an external SSD.

Useful demo commands:

```bash
command -v resolve-transcode
resolve-transcode --help
ffprobe -hide_banner "sample.mkv"
resolve-transcode --sq "sample.mkv" "sample_for_resolve.mov"
ffprobe -hide_banner "sample_for_resolve.mov"
```

If using a real existing file:

```bash
ffprobe -hide_banner "/home/eduardo/0-tutorial-videos/setup-domain.mkv"
```

That file showed:

```text
Video: h264 (High), yuv420p, 1920x1080, 30 fps
Audio: aac (LC), 48000 Hz, stereo
Container: matroska,webm
```

## Research conclusion

Say this clearly:

```text
The bottleneck is not that OBS creates a broken file. OBS creates a valid recording.

The bottleneck is that Resolve Free on Linux cannot fully decode the common OBS output: H.264 or H.265 video plus AAC audio.
```

Breakdown:

- OBS recommends MKV because it is safer for recording.
- OBS supports NVIDIA NVENC on Linux.
- A normal OBS/NVENC file is often H.264 or H.265 video.
- OBS audio is often AAC.
- Blackmagic's codec document lists Linux as Rocky Linux 8.6 with CUDA.
- In the Linux codec table, H.264/H.265 decode for `mov`, `mkv`, and `mp4` is marked Studio Only.
- In the Linux audio table, `aac/m4a` decode is marked unsupported.
- MKV itself is not the whole problem. The bigger problem is the H.264/H.265 video plus AAC audio inside the MKV.
- DNxHR video and Linear PCM audio are supported Resolve Linux targets.

This is why the tool converts:

```text
H.264/AAC/MKV
```

into:

```text
DNxHR/PCM/MOV
```

## Source links to mention or show

OBS Standard Recording Output Guide:

```text
https://obsproject.com/kb/standard-recording-output-guide
```

Key points:

- OBS says MKV is recommended because it will not corrupt the whole file if recording stops badly.
- OBS has remuxing built in.

OBS Hardware Encoding:

```text
https://obsproject.com/kb/hardware-encoding
```

Key points:

- Hardware encoders reduce CPU load.
- NVIDIA NVENC is supported on Windows and Linux.

Blackmagic DaVinci Resolve Supported Formats and Codecs:

```text
https://documents.blackmagicdesign.com/SupportNotes/DaVinci_Resolve_20_Supported_Codec_List.pdf
```

Key points:

- The Linux section is listed as Rocky Linux 8.6 (CUDA).
- DNxHR is supported.
- Linear PCM audio is supported.
- `aac/m4a` decode is not supported in the Linux audio table.
- H.264/H.265 decode for `mov`, `mkv`, and `mp4` is marked Studio Only in the Linux video table.

## Video outline

### 1. Hook

Show the problem fast.

Suggested script:

```text
If you record tutorials in OBS on Ubuntu and then try to bring those recordings into DaVinci Resolve, you can run into a weird problem.

The file plays fine.
VLC can open it.
OBS created it successfully.
But in Resolve Free on Linux, the clip may fail to import, import without audio, or play back incorrectly because the H.264/H.265 video and AAC audio are not fully supported there.

I made a small command-line tool called resolve-transcode to fix that exact handoff.
```

On screen:

```bash
ls
ffprobe -hide_banner "sample.mkv"
```

Point out:

```text
This is a normal OBS file: H.264 video, AAC audio, MKV container.
```

### 2. Explain the actual problem

Suggested script:

```text
At first this feels like OBS is doing something wrong, or Ubuntu is doing something weird, or the NVIDIA card is the issue.

After looking into it, the real bottleneck is DaVinci Resolve Free on Linux codec support.

OBS is optimized for recording and streaming.
DaVinci Resolve is optimized for editing.

Those are not always the same file format.
```

On screen:

- Show OBS recording settings if available.
- Show MKV recording format.
- Show NVIDIA NVENC encoder if available.

Say:

```text
OBS recommending MKV is reasonable. MKV is safer while recording.
NVENC is also reasonable. It records H.264 or H.265 efficiently using the NVIDIA GPU.

The problem is not simply MKV. The problem is the media streams inside the MKV.

My OBS file is H.264 video and AAC audio. Blackmagic's Linux codec table marks H.264 and H.265 decoding as Studio Only, and the Linux audio table marks AAC decode as unsupported.

So if I import this directly into Resolve Free on Linux, Resolve may fail on the video, the audio, or both.
```

### 3. Show the research

Suggested script:

```text
OBS documentation says MKV is recommended because if the recording crashes, you are less likely to lose the whole file.

OBS documentation also says NVIDIA NVENC is supported on Linux.

So this is not simply OBS making a bad file.
```

Show source:

```text
https://obsproject.com/kb/standard-recording-output-guide
https://obsproject.com/kb/hardware-encoding
```

Then:

```text
Blackmagic's supported codec document separates Linux from macOS and Windows.
For Linux, it lists Rocky Linux 8.6 with CUDA.

That already tells us something important: Ubuntu is not the exact official Linux target, even if a lot of us use Resolve on Ubuntu anyway.
```

Show source:

```text
https://documents.blackmagicdesign.com/SupportNotes/DaVinci_Resolve_20_Supported_Codec_List.pdf
```

Say:

```text
The practical issue is that the common OBS output - H.264 or H.265 video, AAC audio, inside MKV - contains codecs Resolve Free on Linux does not fully support.

So instead of fighting that every time, I convert the file into DNxHR video and PCM audio in a MOV container.
```

### 4. Introduce resolve-transcode

Suggested script:

```text
This is the tool.

It is intentionally small. It is just a Bash script that wraps the ffmpeg command I kept needing to run.
```

On screen:

```bash
command -v resolve-transcode
resolve-transcode --help
```

Say:

```text
The default is DNxHR HQ video and PCM 16-bit audio.
That gives Resolve a file that is meant for editing instead of a file that is meant for compact recording.
```

### 5. Show the script

Open `resolve-transcode`.

Point out:

```bash
profile="dnxhr_hq"
pix_fmt="yuv422p"
```

Say:

```text
By default it uses DNxHR HQ and normal 8-bit 4:2:2 video.
```

Point out:

```bash
--sq
--hq
--hqx
```

Say:

```text
I added three profiles.

HQ is the default.
SQ is smaller and usually good enough for screen-recorded tutorials.
HQX is for 10-bit or HDR sources.
```

Point out:

```bash
-c:v dnxhd
-profile:v "$profile"
-pix_fmt "$pix_fmt"
-c:a pcm_s16le
```

Say:

```text
This is the real conversion.
It tells ffmpeg to encode video as DNxHR and audio as 16-bit PCM.
```

### 6. Install the command

Suggested script:

```text
To install it, clone the repo and copy the command into your local bin folder.
```

On screen:

```bash
git clone git@github.com:eduardocgarza/resolve-transcode.git
cd resolve-transcode

mkdir -p ~/.local/bin
cp ./resolve-transcode ~/.local/bin/resolve-transcode
chmod +x ~/.local/bin/resolve-transcode
```

Verify:

```bash
command -v resolve-transcode
resolve-transcode --help
```

Say:

```text
If command -v prints a path, the command is installed.
```

### 7. Basic demo

Suggested script:

```text
Here is the simplest version.
```

On screen:

```bash
resolve-transcode "sample.mkv"
```

Say:

```text
If I do not give it an output name, it creates a new file beside the input and adds _resolve.mov to the filename.
```

Example:

```text
sample.mkv
sample_resolve.mov
```

### 8. Name the output file

Suggested script:

```text
I can also choose the output path.
This is usually what I do when I want the big converted file to go directly to an external SSD.
```

On screen:

```bash
resolve-transcode --sq "sample.mkv" "/media/eduardo/external-ssd/sample.mov"
```

Say:

```text
The SQ profile makes a smaller DNxHR file. It is still much larger than the original MKV, but it is more practical for long tutorial recordings.
```

### 9. Show before and after with ffprobe

Before:

```bash
ffprobe -hide_banner "sample.mkv"
```

Expected talking point:

```text
The input is H.264 video and AAC audio in an MKV container.
```

After:

```bash
ffprobe -hide_banner "sample_for_resolve.mov"
```

Expected talking point:

```text
The output is DNxHR video and PCM audio in a MOV container.
This is the file I import into Resolve.
```

### 10. Import into Resolve

Suggested script:

```text
Now instead of importing the OBS MKV directly, I import the MOV that resolve-transcode created.
```

On screen:

- Open DaVinci Resolve.
- Import `sample_for_resolve.mov`.
- Put it on a timeline.
- Scrub through it.

Say:

```text
This is the point of the tool.
Not smaller files.
Not final delivery.
This is an editing intermediate.
```

### 11. Explain the options

Suggested script:

```text
There are three quality options.
```

On screen:

```bash
resolve-transcode --hq "sample.mkv"
resolve-transcode --sq "sample.mkv"
resolve-transcode --hqx "sample.mkv"
```

Say:

```text
HQ is the default and gives better quality.
SQ is smaller and is usually what I use for normal tutorial screen recordings.
HQX is for 10-bit or HDR material.
```

Then:

```bash
resolve-transcode -f --sq "sample.mkv"
```

Say:

```text
The -f flag overwrites the output if it already exists.
The script avoids overwriting by default so I do not accidentally destroy a converted file.
```

### 12. Explain why remuxing is not enough

Suggested script:

```text
OBS can remux MKV to MP4.
That is useful, but remuxing is not the same thing as transcoding.

Remuxing changes the container.
Transcoding changes the codec.

If the problem is the H.264 video stream or the AAC audio stream, changing MKV to MP4 may not solve it.

This tool changes the actual media format to DNxHR and PCM.
```

On screen:

```text
Remux:      MKV -> MP4, same video/audio codecs
Transcode:  H.264/AAC -> DNxHR/PCM
```

### 13. Be honest about tradeoffs

Suggested script:

```text
The tradeoff is file size.

DNxHR files are much bigger than OBS H.264 files.
That is normal.

H.264 is efficient for recording and delivery.
DNxHR is friendlier for editing.

You are trading disk space for a smoother Resolve workflow.
```

On screen:

```bash
ls -lh sample.mkv sample_for_resolve.mov
```

### 14. Recommended workflow

Suggested script:

```text
My workflow is:

Record in OBS as MKV.
Convert with resolve-transcode.
Import the MOV into DaVinci Resolve.
Edit.
Export the final video from Resolve.
```

On screen:

```text
OBS MKV -> resolve-transcode -> Resolve MOV -> final export
```

### 15. Closing

Suggested script:

```text
So that is resolve-transcode.

It is a tiny Bash script, but it solves a specific Linux video editing problem:
getting OBS recordings from Ubuntu into DaVinci Resolve without fighting codec support every time.

If you are using OBS, Ubuntu, NVIDIA, and Resolve Free, this gives you a repeatable path:
record safely, transcode once, edit the Resolve-friendly file.
```

## Thumbnail / title ideas

Titles:

```text
Fix OBS MKV Files for DaVinci Resolve on Linux
DaVinci Resolve Won't Import OBS Files on Ubuntu? Try This
OBS to DaVinci Resolve on Linux: The Codec Problem Explained
```

Thumbnail text:

```text
OBS -> Resolve
Linux Codec Fix
H.264/AAC -> DNxHR/PCM
```

## Common mistakes to avoid while filming

- Do not say OBS creates a broken file. It creates a valid recording.
- Do not say Ubuntu is definitely the bug. Ubuntu is part of the setup, but the concrete bottleneck is Resolve Free on Linux codec support: H.264/H.265 decode is Studio-only, and AAC decode is unsupported in the Linux audio table.
- Do not promise smaller files. The converted files are usually much larger.
- Do not call this a final export tool. It creates editing intermediates.
- Do not imply everyone needs this on Windows or macOS. This is mainly for the Linux Resolve workflow.

## Short explanation for video description

```text
resolve-transcode is a small Bash/ffmpeg tool that converts OBS MKV recordings into DaVinci Resolve-friendly MOV files on Linux. It converts H.264/AAC recordings into DNxHR video with PCM audio because Resolve Free on Linux does not fully support the common OBS combination: H.264/H.265 video plus AAC audio.
```

## Commands to include in description

```bash
git clone git@github.com:eduardocgarza/resolve-transcode.git
cd resolve-transcode

mkdir -p ~/.local/bin
cp ./resolve-transcode ~/.local/bin/resolve-transcode
chmod +x ~/.local/bin/resolve-transcode

resolve-transcode --help
resolve-transcode --sq "input.mkv" "output.mov"
```

## Final takeaway

```text
The file from OBS is optimized for recording.
The file for Resolve needs to be optimized for editing.
resolve-transcode bridges that gap.
```
