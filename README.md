# aacmax

Encode audio to the highest-quality Apple AAC (VBR) `.m4a` your `ffmpeg` build can produce.

`aacmax` re-encodes one or more audio files to AAC in an `.m4a` container, preferring Apple's `aac_at` VBR encoder with graceful fallbacks. It preserves metadata, chapters, and cover art when possible, never overwrites existing files, writes atomically via a temp file, and verifies the result before keeping it.

## Features

- **Best available encoder** — tries Apple's `aac_at` VBR first, falls back gracefully on other ffmpeg builds.
- **Auto parallelism** — one file runs inline with live progress; multiple files run several at once, scaled to `cores - 1` so your machine stays responsive.
- **Safe by default** — atomic writes, never clobbers existing output, verifies before keeping, cleans up temp files on interrupt.
- **Preserves** metadata, chapters, and cover art where the source allows.
- **Skips already-AAC input** to avoid quality loss from re-compression (override with `AACMAX_FORCE=1`).

## Requirements

- `ffmpeg` (required) — `brew install ffmpeg`
- `ffprobe` (optional, recommended) — enables stream, chapter, cover-art, and duration checks. Ships with ffmpeg.
- `zsh`

## Install

Clone and symlink onto your `PATH`:

```zsh
git clone https://github.com/<your-username>/aacmax.git
ln -s "$PWD/aacmax/bin/aacmax" ~/bin/aacmax
ln -s "$PWD/aacmax/bin/aacmax" ~/bin/aacmax-parallel   # optional: force parallel mode
```

Make sure `~/bin` is on your `PATH`. The script dispatches on its invoked name, so the `aacmax-parallel` symlink always uses the parallel scheduler.

## Usage

```zsh
aacmax <file1> [file2 ...]
aacmax-parallel <file1> [file2 ...]      # force the parallel scheduler

aacmax *.wav                              # wildcards work
```

Tip: type `aacmax ` then drag files into Terminal and press Return.

## Environment knobs

| Variable | Effect |
| --- | --- |
| `AACMAX_FORCE=1` | Re-encode even if the input is already AAC (normally skipped). |
| `AACMAX_JOBS=N` | Use exactly N parallel workers. `N=1` forces serial mode. Unset / `auto` => `cores - 1`. |
| `AACMAX_FORCE_PARALLEL=1` | Internal: set by `aacmax-parallel` to always use the scheduler. |

## Exit status

| Code | Meaning |
| --- | --- |
| `0` | All requested files succeeded (or were harmlessly skipped). |
| `1` | One or more files failed. |
| `64` | Bad usage (no files given). |
| `127` | ffmpeg not installed. |
| `130` / `143` | Interrupted (Ctrl-C / TERM) — temp files cleaned up. |

## License

[MIT](LICENSE)
