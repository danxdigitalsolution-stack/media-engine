# media_engine

A native Rust extension module for the **Macan Angkasa** suite, built with [PyO3](https://pyo3.rs)/[Maturin](https://www.maturin.rs). It wraps [FFmpeg](https://ffmpeg.org) (via `ffmpeg-next`) and a small real-time audio pipeline (via `cpal` + `ringbuf`) to handle performance-critical media work that isn't practical to do efficiently in pure Python — full playback, batch thumbnailing, waveform/BPM analysis, and audio/video transcoding.

Built as an `abi3-py310` extension, so a single compiled wheel works across Python 3.10+.

## Why this exists

Python remains the primary language across Macan Angkasa. `media_engine` (and its sibling crates `media_tools` and `macan_fft`) exist only for the specific hot paths where Python was the bottleneck: real-time decode/playback loops, frame scaling, FFT-based spectrum analysis, and sample-level audio math. Everything else stays in Python/PySide6.

## Features

- **`MediaInfo`** — Fast metadata probe of the video stream (duration, resolution, fps, codec, bitrate) without decoding frames.
- **`AudioInfo`** — Fast metadata probe of the audio stream (codec, sample rate, channels, bitrate) without decoding samples. Replaces spawning `ffprobe` for this purpose — no subprocess, no timeout to wait out.
- **`VideoDecoder`** — Single-frame seek/decode, primarily used for one-off thumbnails (e.g. seek-frame preview in a scrubber).
- **`PlayerEngine`** — Full playback engine driving synchronized video + audio decode threads:
  - Play / pause / stop / step-frame / seek / loop
  - Volume control and mute, independent of the OS mixer
  - A/V clock sync (`position`, `effective_volume`, clock freeze/reset around seeks)
  - Lock-free audio delivery to the output device via a `cpal` stream backed by an SPSC `ringbuf` (no mutex on the real-time audio callback)
  - Live FFT spectrum output (`get_spectrum`) for audio visualizers, computed with `rustfft`
- **`analyze_waveform`** — Decodes an audio track once and returns a downsampled amplitude envelope (for waveform rendering) plus a detected BPM, using `rayon` to parallelize the envelope/autocorrelation math.
- **`analyze_loudness`** — Quick peak/RMS loudness readout in dBFS (not full EBU R128 LUFS), useful for a "normalize volume" feature.
- **`generate_thumbnails`** — Extracts N evenly-spaced thumbnails from a video in a single pass, with a shared scaler sized once for the whole batch. Individual failed seeks are skipped rather than failing the whole call — useful for gallery/grid previews where a partially-populated grid beats a hard failure.
- **`convert_media`** — Native transcode/remux (replacing subprocess calls to `ffmpeg.exe`), with per-track stream-copy or full transcode, bitrate control, track dropping (e.g. extract audio only), and start/end trimming.

## Requirements

- Rust (stable) with the target you're building for
- Python 3.10+ (host, for running `maturin`)
- [Maturin](https://www.maturin.rs)
- FFmpeg 9.0 development libraries — how these are provided depends on platform (see below)

### Linux (native build/lint)

`ffmpeg-next`'s `build` feature compiles FFmpeg from source automatically. You'll need `libasound2-dev` installed if you want the `cpal` ALSA backend to build locally.

```bash
sudo apt-get install -y libasound2-dev
cargo build --release
```

### Windows (native or cross-compiled from Linux)

Audio output uses `cpal`'s WASAPI backend on Windows, which goes through `windows-sys`/COM and needs no extra C libraries — this is what makes cross-compiling from Linux practical.

FFmpeg itself is **not** compiled through `ffmpeg-sys-next`'s `build` feature on Windows targets. `ffmpeg-sys-next` 9.x passes the wrong `--target_os` string when cross-compiling (`windows` instead of `mingw32`), which breaks FFmpeg's `configure` step. Instead:

1. FFmpeg is cross-compiled manually with `--target-os=mingw32` (see `build.yml`).
2. The result is pointed to via the `FFMPEG_DIR` environment variable.
3. `ffmpeg-sys-next` links against that prebuilt FFmpeg instead of trying to build its own.

You won't need to do this by hand locally unless you're debugging the CI pipeline — see [CI](#ci) below.

## Building

```bash
# Local development build
maturin develop --release

# Produce a wheel
maturin build --release --out dist
```

## CI

`.github/workflows/build.yml` cross-compiles a Windows `abi3` wheel from an `ubuntu-latest` runner:

1. Installs the `mingw-w64` toolchain (posix threading variant — required because FFmpeg needs pthreads, and the default `-win32` variant on Ubuntu lacks it).
2. Cross-compiles `libdav1d` statically via Meson/Ninja.
3. Cross-compiles FFmpeg statically for `mingw32`, with a large set of codecs/muxers disabled (`avdevice`, `avfilter`, `postproc`, hardware acceleration) to keep build time down, since a full FFmpeg build can approach the 40-minute CI timeout on a 2-core runner. The result is cached and keyed on the FFmpeg tag plus a hash of the workflow file, so changing configure flags invalidates the cache automatically.
4. Points `ffmpeg-sys-next` at the cross-compiled FFmpeg via `FFMPEG_DIR`.
5. Runs `maturin build --release --target x86_64-pc-windows-gnu` to produce the final `abi3` wheel.
6. Uploads the wheel as a build artifact.

**Keeping `FFMPEG_TAG` and `ffmpeg-next` in sync:** `ffmpeg-next` contains hand-maintained exhaustive `match` statements over FFmpeg's C enums (pixel formats, codec IDs, etc.). If the FFmpeg version compiled in CI is newer than what `ffmpeg-next` was written against, new enum variants can appear in the generated bindings that `ffmpeg-next`'s Rust code doesn't handle, causing an `E0004` non-exhaustive-patterns compile error. When bumping `FFMPEG_TAG`, bump the `ffmpeg-next` (and transitively `ffmpeg-sys-next`) version in `Cargo.toml` too — don't change one without the other.

**Debugging a failed CI run:**
- FFmpeg `configure`/compile failure → check `ffmpeg-src/ffbuild/config.log`, dumped automatically on failure (last 200 lines).
- Rust/linker failure → check the `stderr`/`stdout` section of the `cargo`/`maturin` output in the compile step; this is usually a clearer signal than the FFmpeg log once FFmpeg itself has built successfully.

## Project structure

```
Cargo.toml              # Dependencies, per-target FFmpeg feature split
src/lib.rs               # All PyO3 bindings and media logic
.github/workflows/build.yml  # Windows cross-compile pipeline
```

## Design notes

- **No lock on the audio callback.** Audio samples move from the decode thread to the real-time `cpal` callback through an SPSC lock-free ring buffer (`ringbuf`), not a `Mutex<VecDeque<f32>>`. A blocked real-time callback is audible as glitching, so anything that could contend with it is avoided by construction.
- **`rayon` for embarrassingly parallel chunked work.** Envelope/RMS computation and BPM autocorrelation are the two spots that benefit from `par_iter()` without needing a manually managed thread pool.
- **Stream copy vs. transcode in `convert_media`.** Stream-copied tracks trim to the nearest keyframe before the requested start time (fast, but not frame-exact — the same limitation as `ffmpeg -ss` placed before `-i`). Frame-exact trimming requires transcoding that track.
- **`generate_thumbnails` builds one scaler for the whole batch**, sized to the thumbnail dimensions, rather than re-deriving it per frame like `VideoDecoder::seek_frame` does — this matters when generating many thumbnails at once (e.g. gallery grids).
