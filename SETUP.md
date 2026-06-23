# Setup

This guide explains how to install `resolve-transcode` so you can call it from any terminal folder like this:

```bash
resolve-transcode "recording.mkv"
```

Instead of this:

```bash
./resolve-transcode "recording.mkv"
```

## How command lookup works

When you type a command like:

```bash
resolve-transcode
```

your shell searches each folder listed in the `PATH` environment variable.

You can see those folders with:

```bash
echo "$PATH"
```

On most Linux systems, `~/.local/bin` is the right place for personal command-line tools. It is inside your home folder, does not require `sudo`, and is commonly included in `PATH`.

The full path is:

```text
/home/YOUR_USER/.local/bin
```

For this machine, that is:

```text
/home/eduardo/.local/bin
```

## Quick install

From inside the cloned repo:

```bash
mkdir -p ~/.local/bin
cp ./resolve-transcode ~/.local/bin/resolve-transcode
chmod +x ~/.local/bin/resolve-transcode
```

Then verify:

```bash
command -v resolve-transcode
resolve-transcode --help
```

If setup is correct, `command -v` should print something like:

```text
/home/eduardo/.local/bin/resolve-transcode
```

## Clone and install from scratch

```bash
cd ~/Desktop/code/tutorials
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

## If `command -v` shows nothing

That means the shell cannot find `resolve-transcode`.

First confirm the file exists:

```bash
ls -l ~/.local/bin/resolve-transcode
```

Then confirm `~/.local/bin` is in `PATH`:

```bash
echo "$PATH"
```

If `~/.local/bin` is missing, add it to `~/.bashrc`:

```bash
printf '\nexport PATH="$HOME/.local/bin:$PATH"\n' >> ~/.bashrc
source ~/.bashrc
```

Check again:

```bash
command -v resolve-transcode
```

## If the file is not executable

If you see a permission error, make it executable:

```bash
chmod +x ~/.local/bin/resolve-transcode
```

Check permissions:

```bash
ls -l ~/.local/bin/resolve-transcode
```

You want to see an `x` in the permissions, like:

```text
-rwxr-xr-x
```

## Updating the installed command

If you change the script in the repo, copy it into `~/.local/bin` again:

```bash
cd ~/Desktop/code/tutorials/resolve-transcode
cp ./resolve-transcode ~/.local/bin/resolve-transcode
chmod +x ~/.local/bin/resolve-transcode
```

Then clear Bash's command cache:

```bash
hash -r
```

Verify:

```bash
resolve-transcode --help
```

## Optional: install with a symlink

Instead of copying the file, you can link the repo file into `~/.local/bin`:

```bash
mkdir -p ~/.local/bin
ln -sf "$PWD/resolve-transcode" ~/.local/bin/resolve-transcode
chmod +x ./resolve-transcode
```

With a symlink, edits to the repo copy are immediately reflected when you run:

```bash
resolve-transcode
```

For a tutorial or public install instructions, the copy method is simpler. For local development, the symlink method is convenient.

## Required dependency

`resolve-transcode` uses `ffmpeg`.

Check whether it is installed:

```bash
ffmpeg -version
```

On Ubuntu, install it with:

```bash
sudo apt update
sudo apt install ffmpeg
```

## Test conversion

From a folder containing an OBS recording:

```bash
resolve-transcode --sq "recording.mkv"
```

Expected output:

```text
recording_resolve.mov
```

Check the output:

```bash
ffprobe -hide_banner "recording_resolve.mov"
```

The file should be a `.mov` with DNxHR video and PCM audio.
