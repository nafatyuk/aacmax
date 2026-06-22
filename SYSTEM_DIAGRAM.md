# aacmax System Diagram

```mermaid
flowchart TD
  User["User"]
  Cmd1["Command: aacmax <file...>"]
  Cmd2["Command: aacmax-parallel <file...>"]
  Script["File: bin/aacmax"]
  Dispatch{"Invoked as aacmax-parallel?"}
  Main["_aacmax_main <file...>"]
  Usage{"Any input files?"}
  UsageOut["Output: usage text\nExit: 64"]
  Workers["_aacmax_worker_count\nreads AACMAX_JOBS\nreads CPU count"]
  ForceParallel{"Forced parallel?\nor workers > 1?"}
  Serial["_aacmax_run_serial"]
  Parallel["_aacmax_run_parallel"]
  LogDir["Temp log dir:\n$TMPDIR/aacmax-logs.*"]
  Jobs["Background jobs:\n_aacmax_one per file"]
  One["_aacmax_one <file>"]

  User --> Cmd1 --> Script
  User --> Cmd2 --> Script
  Script --> Dispatch
  Dispatch -- "yes" --> Main
  Dispatch -- "no" --> Main
  Main --> Usage
  Usage -- "no" --> UsageOut
  Usage -- "yes" --> Workers --> ForceParallel
  ForceParallel -- "no: one file or AACMAX_JOBS=1" --> Serial --> One
  ForceParallel -- "yes: multiple workers or aacmax-parallel" --> Parallel
  Parallel --> LogDir --> Jobs --> One

  One --> Deps{"ffmpeg found?"}
  Deps -- "no" --> NoFF["Output: ERROR ffmpeg not found\nExit: 127"]
  Deps -- "yes" --> ProbeAvail{"ffprobe found?"}
  ProbeAvail -- "yes" --> ProbeChecks["Commands: ffprobe\ncheck audio stream\ncheck input codec\ncheck cover art\ncheck chapters\ncheck channels"]
  ProbeAvail -- "no" --> LimitedChecks["Continue with limited checks"]

  ProbeChecks --> InputChecks
  LimitedChecks --> InputChecks

  InputChecks{"Input usable?\nexists, file, readable,\nnon-empty, writable dir,\nenough disk"}
  InputChecks -- "no" --> InputErr["Output: ERROR reason\nExit: 1"]
  InputChecks -- "yes" --> AlreadyAAC{"Input codec is AAC\nand AACMAX_FORCE unset?"}
  AlreadyAAC -- "yes" --> SkipAAC["Output: SKIP already AAC\nExit: 0"]
  AlreadyAAC -- "no or unknown" --> OutputPath["Output path:\n<stem>_AAC_VBR.m4a\nor <stem>_AAC_VBR(n).m4a"]

  OutputPath --> TempFiles["Temp files:\n/tmp/aacmax.*.m4a\n/tmp/aacmax.log.*"]
  TempFiles --> Maps["Build ffmpeg args:\n-map_metadata 0\n-map_chapters 0 or -1\n-map 0:a:0\noptional cover-art video map\n-movflags +faststart+use_metadata_tags"]
  Maps --> Encoder{"Which AAC encoder works?"}

  Encoder -- "aac_at VBR q0" --> FFmpeg1["Command: ffmpeg ... -c:a aac_at -q:a 0 ..."]
  Encoder -- "aac_at VBR minimal" --> FFmpeg2["Command: ffmpeg ... -c:a aac_at -q:a 0 ..."]
  Encoder -- "aac_at CVBR" --> FFmpeg3["Command: ffmpeg ... -c:a aac_at -aac_at_mode cvbr -b:a 128k/256k ..."]
  Encoder -- "aac_at CBR" --> FFmpeg4["Command: ffmpeg ... -c:a aac_at -aac_at_mode cbr -b:a 128k/256k ..."]
  Encoder -- "native AAC fallback" --> FFmpeg5["Command: ffmpeg ... -c:a aac -b:a 128k/256k ..."]

  FFmpeg1 --> Remedies
  FFmpeg2 --> Remedies
  FFmpeg3 --> Remedies
  FFmpeg4 --> Remedies
  FFmpeg5 --> Remedies

  Remedies{"ffmpeg succeeded?"}
  Remedies -- "cover-art parse error" --> DropCover["Drop cover-art map\nretry same encoder once"] --> Encoder
  Remedies -- "invalid data with faststart" --> DropFaststart["Drop +faststart\nretry once"] --> Encoder
  Remedies -- "chapter failure" --> DropChapters["Set -map_chapters -1\nretry once"] --> Encoder
  Remedies -- "too many bits per frame" --> NextEncoder["Try next encoder mode"] --> Encoder
  Remedies -- "no, no fallbacks left" --> EncodeFail["Output: encode failed\nkeep log path\nremove temp m4a\nExit: 1"]
  Remedies -- "yes" --> Verify{"ffprobe available?"}

  Verify -- "yes" --> VerifyCodec["Command: ffprobe temp output\nverify codec is AAC\ncompare duration"]
  Verify -- "no" --> MoveOutput
  VerifyCodec --> CodecOK{"Codec is AAC?"}
  CodecOK -- "no" --> VerifyFail["Output: verification failed\nremove temp m4a\nExit: 1"]
  CodecOK -- "yes" --> DurationWarn{"Duration drift > 1s?"}
  DurationWarn -- "yes" --> Warn["Output: WARN duration differs"] --> MoveOutput
  DurationWarn -- "no" --> MoveOutput["Command: mv temp.m4a final.m4a"]

  MoveOutput --> MoveOK{"Move succeeded?"}
  MoveOK -- "no" --> MoveFail["Output: could not move output\nremove temp m4a\nExit: 1"]
  MoveOK -- "yes" --> Success["Output: Wrote final .m4a\nincludes codec/bitrate when available\nExit: 0"]

  Parallel --> ParallelDone{"Any job failed?"}
  ParallelDone -- "yes" --> ParallelFail["Output: Some jobs failed\nkeep log dir\nExit: 1"]
  ParallelDone -- "no" --> ParallelSuccess["Output: all jobs completed\nremove log dir\nExit: 0"]
```

## File and Runtime Inventory

| Item | Role |
| --- | --- |
| `bin/aacmax` | The whole application: CLI dispatch, serial/parallel scheduling, per-file conversion, retries, verification, and output move. |
| `README.md` | User-facing installation, usage, environment knobs, and exit-code documentation. |
| `.gitignore` | Ignores macOS/editor temp files and generated `.m4a` outputs. |
| `LICENSE` | MIT license. |
| `/tmp/aacmax.*.m4a` | Atomic temporary output before final move. |
| `/tmp/aacmax.log.*` | Per-file ffmpeg stderr log during conversion. |
| `$TMPDIR/aacmax-logs.*` | Parallel-mode job log directory; removed only when all jobs succeed. |
| `<input_dir>/<stem>_AAC_VBR.m4a` | Final output file. |
| `<input_dir>/<stem>_AAC_VBR(n).m4a` | Non-clobbering alternate output name when a prior output exists. |

## Command Surface

| Command or Variable | Effect |
| --- | --- |
| `aacmax <file...>` | Convert one or more files; one file runs serially, multiple files run in parallel. |
| `aacmax-parallel <file...>` | Force the parallel scheduler, even when worker count would otherwise be one. |
| `AACMAX_FORCE=1` | Re-encode inputs that are already AAC. |
| `AACMAX_JOBS=N` | Set exact parallel worker count; `1` forces serial mode. |
| `AACMAX_JOBS=auto` | Use automatic worker count, effectively CPU cores minus one, capped by file count. |
