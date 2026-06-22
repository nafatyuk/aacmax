# aacmax

Encode audio to the highest-quality Apple AAC `.m4a` your local `ffmpeg` build can produce.

`aacmax` is a small zsh command-line tool for batch-converting audio files to AAC in an `.m4a` container. It prefers FFmpeg's Apple AudioToolbox encoder (`aac_at`) in VBR mode when that encoder is available, then falls back through safer AAC modes when it is not.

It is designed for the boring-good parts of audio conversion: preserve what can be preserved, avoid clobbering existing files, clean up after interruption, and verify the output before keeping it.

## What it does

- Encodes audio to AAC in an `.m4a` container.
- Prefers `aac_at` VBR, usually the best option on macOS FFmpeg builds that expose Apple's AAC encoder.
- Falls back to other `aac_at` modes, then FFmpeg's native AAC encoder.
- Preserves metadata, chapters, and cover art where the input and tools allow.
- Skips already-AAC input by default to avoid lossy-to-lossy re-compression.
- Writes atomically through a temporary file and never overwrites an existing output.
- Runs one file in the foreground, or multiple files in parallel with per-job logs.

## Requirements

Required:

- macOS with `zsh`
- [`ffmpeg`](https://ffmpeg.org/)

Recommended:

- [`ffprobe`](https://ffmpeg.org/ffprobe.html), usually installed with FFmpeg

On macOS with Homebrew:

```zsh
brew install ffmpeg
```

`aacmax` does not bundle FFmpeg, ffprobe, Apple frameworks, AAC codecs, or any audio samples. It shells out to the tools already installed on your machine.

## Install

Clone the repository:

```zsh
git clone https://github.com/nafatyuk/aacmax.git
cd aacmax
```

Put the command somewhere on your `PATH`. For example, if you use `~/bin`:

```zsh
mkdir -p ~/bin
ln -s "$PWD/bin/aacmax" ~/bin/aacmax
```

Optional: add a second symlink that always forces the parallel scheduler:

```zsh
ln -s "$PWD/bin/aacmax" ~/bin/aacmax-parallel
```

Make sure `~/bin` is on your `PATH`:

```zsh
echo $PATH
```

If it is not, add this to your `~/.zshrc`:

```zsh
export PATH="$HOME/bin:$PATH"
```

Then open a new shell or run:

```zsh
source ~/.zshrc
```

Check the install:

```zsh
aacmax
```

With no files, it should print usage text and exit with code `64`.

## Usage

Convert one file:

```zsh
aacmax input.wav
```

Convert several files:

```zsh
aacmax *.wav
```

Force parallel mode:

```zsh
aacmax-parallel *.aiff
```

Type `aacmax `, drag files into Terminal, then press Return if you prefer using Finder.

Outputs are written beside the source file:

```text
Song.wav       -> Song_AAC_VBR.m4a
Song.wav       -> Song_AAC_VBR(1).m4a    # if the first output already exists
```

## Configuration

| Variable | Effect |
| --- | --- |
| `AACMAX_FORCE=1` | Re-encode even if the input is already AAC. By default, AAC input is skipped. |
| `AACMAX_JOBS=N` | Use exactly `N` parallel workers. `AACMAX_JOBS=1` forces serial mode. |
| `AACMAX_JOBS=auto` | Use automatic parallelism: CPU cores minus one, capped by the number of files. |
| `AACMAX_FORCE_PARALLEL=1` | Internal switch used by the `aacmax-parallel` symlink. |

Examples:

```zsh
AACMAX_FORCE=1 aacmax old-file.m4a
AACMAX_JOBS=2 aacmax *.wav
AACMAX_JOBS=1 aacmax *.aiff
```

## How it works

The whole application is [bin/aacmax](bin/aacmax), with three main layers:

1. `_aacmax_main` handles usage, worker-count decisions, and serial vs parallel dispatch.
2. `_aacmax_run_serial` and `_aacmax_run_parallel` decide how files are scheduled.
3. `_aacmax_one` performs the conversion for a single file.

For each input file, `_aacmax_one`:

1. Checks that `ffmpeg` exists.
2. Uses `ffprobe`, when available, to inspect streams, codec, chapters, cover art, duration, and channel count.
3. Rejects missing, unreadable, empty, non-file, or unwritable inputs.
4. Skips AAC input unless `AACMAX_FORCE=1` is set.
5. Chooses a non-clobbering output path.
6. Creates temporary output and log files.
7. Builds an FFmpeg command that maps audio, metadata, chapters, and optional cover art.
8. Tries the best encoder first, then falls back:
   - `aac_at` VBR, highest-quality flags when supported
   - `aac_at` VBR with minimal flags
   - `aac_at` constrained VBR
   - `aac_at` CBR
   - native FFmpeg `aac` CBR
9. Retries common failure cases by dropping cover art, dropping `+faststart`, or disabling chapter mapping.
10. Verifies the temporary output is AAC when `ffprobe` is available.
11. Moves the temporary file into its final output path only after success.

See [SYSTEM_DIAGRAM.md](SYSTEM_DIAGRAM.md) for an end-to-end flow diagram of files, commands, outputs, and decision points.

## Exit status

| Code | Meaning |
| --- | --- |
| `0` | All requested files succeeded or were harmlessly skipped. |
| `1` | One or more files failed. |
| `64` | Bad usage, usually no files given. |
| `127` | `ffmpeg` is not installed or not on `PATH`. |
| `130` | Interrupted with Ctrl-C. |
| `143` | Terminated. |

## Notes and limits

- This script is macOS-oriented. It uses zsh and macOS-style system commands such as `stat -f%z` and `sysctl`.
- `aac_at` availability depends on your FFmpeg build and platform. If it is not present, `aacmax` falls back to FFmpeg's native AAC encoder.
- FFmpeg builds vary. Different Homebrew, system, and custom builds may expose different encoders and licensing options.
- This project is a convenience wrapper; it is not an audio codec implementation.

## Credits and licensing

`aacmax` is copyright (c) 2026 Daniel J. Nafatyuk and is released under the [MIT License](LICENSE).

This project depends on external tools and platform components at runtime:

- [FFmpeg](https://ffmpeg.org/) and [ffprobe](https://ffmpeg.org/ffprobe.html) are developed by the FFmpeg project. FFmpeg's own license depends on build options; the FFmpeg project describes the base license as LGPLv2.1-or-later, with GPLv2-or-later applying when GPL components are enabled. See [FFmpeg License and Legal Considerations](https://ffmpeg.org/legal.html).
- `aac_at` is FFmpeg's encoder interface for Apple's [Audio Toolbox](https://developer.apple.com/documentation/audiotoolbox) AAC encoder on Apple platforms where that support is available. Apple Audio Toolbox and related platform components are not part of this repository and are not redistributed here.
- FFmpeg is a trademark of Fabrice Bellard, originator of the FFmpeg project, according to FFmpeg's legal page.
- Apple and macOS are trademarks of Apple Inc.

No FFmpeg, ffprobe, Apple source code, Apple binaries, or third-party codec libraries are included in this repository.

This README gives attribution and project-level licensing context, not legal advice. If you package or distribute `aacmax` together with an FFmpeg binary, you are responsible for complying with the license terms of that FFmpeg build and any libraries compiled into it.
