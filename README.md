# media_engine

A native Rust extension module for the **Macan Angkasa** suite, built with [PyO3](https://pyo3.rs)/[Maturin](https://www.maturin.rs). It wraps [FFmpeg](https://ffmpeg.org) (via `ffmpeg-next`) and a small real-time audio pipeline (via `cpal` + `ringbuf`) to handle performance-critical media work that isn't practical to do efficiently in pure Python: full audio/video playback, batch thumbnailing, waveform/BPM/spectrum analysis, loudness measurement, and audio/video transcoding.

Built as an `abi3-py310` extension, so a single compiled wheel works across Python 3.10+. Current version: **0.5.4**.

## Why this exists

Python remains the primary language across Macan Angkasa. `media_engine` exists only for the specific hot paths where Python was the bottleneck: real-time decode/playback loops, frame scaling, FFT-based spectrum analysis, and sample-level audio math. Everything else stays in Python/PySide6.

It also consolidates what used to be separate native pieces and subprocess calls into a single binary:

| Replaced | By |
|---|---|
| `macan_fft` crate | `SpectrumAnalyzer` |
| `media_tools` crate (`compute_waveform_envelope`, `detect_bpm`) | `compute_waveform_envelope`, `detect_bpm` |
| `ffprobe` subprocess for audio metadata/duration | `AudioInfo` |
| `ffmpeg -f s16le pipe:1` subprocess for PCM decode | `decode_audio` |
| `ffmpeg.exe` subprocess for conversion | `convert_media` |
| `QMediaPlayer`/`QAudioOutput` for audio-only playback | `PlayerEngine` (audio-only mode) |

## Features

### Metadata
- **`MediaInfo`**: fast probe of the video stream (duration, resolution, fps, codec, bitrate) without decoding frames.
- **`AudioInfo`**: fast probe of the audio stream (duration, codec, sample rate, channels, bitrate) without decoding samples. Duration and bitrate fall back to container-level values when the stream doesn't carry them (e.g. some CBR MP3s).

### Single-frame decoding
- **`VideoDecoder`**: single-frame `seek_frame(second)` / `read_next_frame()`, for one-off thumbnails such as seekbar hover previews. Seeking uses `AV_TIME_BASE` units for the demuxer and the stream time base for frame matching.
- **`generate_thumbnails`**: N evenly spaced thumbnails in one pass with a single scaler sized for the whole batch (optional `max_width`, aspect ratio preserved). Failed seeks are skipped instead of failing the whole call, which suits gallery/grid previews.

### Playback: `PlayerEngine`
Drives synchronized video and audio decode on **separate threads, each with its own file handle**, so heavy video work (large keyframes, post-seek catch-up) can never starve the audio ring buffer.

- Play / pause / stop / seek / loop / frame-step
- **Video is optional**: audio-only files (mp3, flac, wav, ...) play normally; `has_video` is `False` and `width`/`height`/`fps` are `0`
- **Network sources**: FFmpeg network protocols are initialized, so `http(s)` URLs (radio streams, podcasts) can be opened directly
- **Variable playback speed**: `set_playback_rate(0.25 .. 4.0)` using WSOLA time-stretching, so pitch is preserved. Changing speed needs no flush or seek, and the master clock counts *source* time, so video follows automatically
- **10-band equalizer** with preamp (`set_equalizer` / `clear_equalizer`): RBJ peaking biquads at 60, 170, 310, 600, 1k, 3k, 6k, 12k, 14k and 16k Hz, ±20 dB, Q = 1.41. Safe to change while playing
- Volume and mute, independent of the OS mixer (mute doesn't lose the volume value)
- A/V sync with the audio clock as master (wall clock for video-only content or if the audio device is unavailable), including late-frame dropping, gap padding, and drift correction from audio PTS
- Lock-free audio delivery: decode thread → SPSC `ringbuf` → `cpal` callback
- Live FFT spectrum of what is currently playing (`get_spectrum`), fed from a separate bounded visualization ring so it never interferes with playback
- Full-drain EOF detection (`is_eof`) that waits for both video and audio to finish, so callers don't cut off the audio tail

### Analysis
- **`analyze_waveform(path, target_points)`**: decodes once at 22.05 kHz mono; returns a normalized RMS envelope and an estimated BPM (`0.0` if undetectable).
- **`decode_audio(path, out_rate)`**: whole track as mono `float32` in `[-1, 1]`, resampled to `out_rate`.
- **`compute_waveform_envelope(samples, target_points)`**: operates on an array you already have; returns raw `(mins, maxs, rmss, firsts)` per column, suited to zoomable waveform rendering.
- **`detect_bpm(samples, sample_rate)`**: onset-energy + autocorrelation over 60–200 BPM. Works best on music with a clear beat; unreliable for ambient or rubato material.
- **`SpectrumAnalyzer(fft_size)`**: reusable FFT analyzer (Hann window, precomputed plan). `.compute(samples)` returns **linear** magnitudes of length `fft_size // 2 + 1`. Note that `PlayerEngine.get_spectrum()` returns **dB** instead.
- **`analyze_loudness(path)`**: peak and RMS in dBFS (floored at -120 dB). Not EBU R128 LUFS; intended for "normalize volume" style features.

### Conversion: `convert_media`
Native transcode/remux with per-track stream copy or full transcode:

- `video_codec` / `audio_codec`: encoder name (`"aac"`, `"libmp3lame"`, ...) or `None` for stream copy
- `video_bitrate` / `audio_bitrate`: bits per second (transcode only)
- `drop_video` / `drop_audio`: e.g. extract audio only
- `trim_start` / `trim_end`: seconds; output timeline restarts at 0
- `progress_callback(percent: float)`: throttled to ~0.5% steps, with a final `100.0`. Never called if the duration is unknown
- Audio is re-chunked through an internal FIFO to the encoder's exact `frame_size` (needed for e.g. AAC → MP3, where frame sizes differ)

> Encoders must be compiled into the FFmpeg build. See [Codec availability](#codec-availability).

## Python API quick reference

```python
from media_engine import (
    MediaInfo, AudioInfo, VideoDecoder, PlayerEngine, SpectrumAnalyzer,
    analyze_waveform, analyze_loudness, generate_thumbnails,
    convert_media, decode_audio, compute_waveform_envelope, detect_bpm,
)
```

| API | Signature / notes |
|---|---|
| `MediaInfo(path)` | `.duration .width .height .fps .codec .codec_id .bitrate` |
| `AudioInfo(path)` | `.duration .codec .codec_id .sample_rate .channels .bitrate` |
| `VideoDecoder(path)` | `.seek_frame(sec)`, `.read_next_frame()` → `ndarray[H,W,3] uint8`. **Unsendable** |
| `PlayerEngine(path)` | attrs: `duration width height fps has_audio has_video`. **Unsendable** |
| `.play() .pause() .stop() .seek(sec) .position()` | transport and clock |
| `.is_playing() .is_eof()` | state |
| `.set_loop(bool) .is_looping()` | auto-restart at EOF |
| `.step_frame()` | one frame forward, only while paused |
| `.set_volume(0..1) .get_volume() .set_muted(bool) .is_muted()` | volume |
| `.set_playback_rate(0.25..4.0) .get_playback_rate()` | speed (pitch preserved) |
| `.set_equalizer(gains, preamp=0.0, enabled=True) .clear_equalizer()` | 10 gains in dB |
| `.get_frame()` | `(rgb_array, pts)` or `None` if no frame is due |
| `.get_spectrum(fft_size)` | dB array, length `fft_size // 2`; `fft_size` must be a power of 2 |
| `.close()` | stop both threads; also called on drop |
| `SpectrumAnalyzer(fft_size)` | `.compute(samples)`; `len(samples)` must equal `fft_size` |
| `analyze_waveform(path, target_points)` | `(envelope, bpm)` |
| `analyze_loudness(path)` | `(peak_db, rms_db)` |
| `generate_thumbnails(path, count, max_width=None)` | `[(rgb_array, seconds), ...]` |
| `decode_audio(path, out_rate)` | mono `float32` array |
| `compute_waveform_envelope(samples, target_points)` | `(mins, maxs, rmss, firsts)`; input must be C-contiguous |
| `detect_bpm(samples, sample_rate)` | `float` |
| `convert_media(input, output, video_codec=None, audio_codec=None, video_bitrate=None, audio_bitrate=None, drop_video=False, drop_audio=False, trim_start=None, trim_end=None, progress_callback=None)` | see above |

### Usage notes

- `PlayerEngine` and `VideoDecoder` are `unsendable`: use them only on the thread that created them. If one lives in a Qt worker thread, create it *inside* that thread and release it there too (see the thumbnail worker in `player_engine_test.py`).
- Keep a reference to the array returned by `get_frame()` for as long as a `QImage` built on it is alive.
- `step_frame()` does nothing for audio-only files.
- If the default output device can't be opened, `PlayerEngine` degrades to video-only playback instead of failing.
- Audio output always uses the system default device.

## Requirements

- Rust (stable) with the target you're building for
- Python 3.10+ (host, for running `maturin`)
- [Maturin](https://www.maturin.rs)
- FFmpeg 9.0 development libraries; how these are provided depends on the platform (see below)

### Key dependencies

| Crate | Version |
|---|---|
| `pyo3` (`abi3-py310`) | 0.29.3 |
| `numpy` / `ndarray` | 0.29 / 0.17 |
| `ffmpeg-next` / `ffmpeg-sys-next` | 9.0.0 / =9.0.1 |
| `cpal` | 0.18 |
| `ringbuf` | 0.5.3 |
| `rustfft` | 6.4.1 |
| `rayon` | 1.12 |

The release profile uses `opt-level = 3`, fat LTO, `codegen-units = 1`, `strip = true` and `panic = "abort"`.

### Linux (native build/lint)

`ffmpeg-next`'s `build` feature compiles FFmpeg from source automatically. You need `libasound2-dev` for the `cpal` ALSA backend.

```bash
sudo apt-get install -y libasound2-dev
cargo build --release
```

### Windows (native or cross-compiled from Linux)

Audio output uses `cpal`'s WASAPI backend, which goes through `windows-sys`/COM and needs no extra C libraries; this is what makes cross-compiling from Linux practical.

FFmpeg itself is **not** compiled through `ffmpeg-sys-next`'s `build` feature on Windows targets (the `windows` vs `mingw32` `--target_os` issue when cross-compiling). Instead:

1. FFmpeg is cross-compiled manually with `--target-os=mingw32` (see `build.yml`).
2. The result is pointed to via the `FFMPEG_DIR` environment variable.
3. `ffmpeg-sys-next` links against that prebuilt FFmpeg.

You won't need to do this by hand locally unless you're debugging the CI pipeline.

## Building

```bash
# Local development build
maturin develop --release

# Produce a wheel
maturin build --release --out dist
```

## Codec availability

What `convert_media` (and decoding in general) can do depends on what was compiled into FFmpeg:

- **Windows CI build**: static FFmpeg with `libdav1d` (AV1 decoding), `libmp3lame` (MP3 encoding) and `libvorbis` (Vorbis encoding), plus FFmpeg's built-in codecs (e.g. native `aac`). `libx264`/`libx265` are **not** included, so video transcoding is limited to built-in encoders such as `mpeg4`.
- **Linux (`build` feature)**: only what FFmpeg's default configure provides unless you enable additional `build-lib-*` features in `Cargo.toml`.
- Requesting an encoder that isn't available returns a `ValueError` naming the encoder.
- Only `avcodec`, `avformat`, `swresample` and `swscale` are built. `avdevice`, `avfilter` and `postproc` are disabled.

## CI

`.github/workflows/build.yml` cross-compiles a Windows `abi3` wheel from an `ubuntu-latest` runner (on push to `main`/`master`, or manually via `workflow_dispatch`):

1. Installs the `mingw-w64` toolchain (posix threading variant, required because FFmpeg needs pthreads and the default `-win32` variant lacks it).
2. Cross-compiles `libdav1d` (Meson/Ninja), `libmp3lame`, `libogg` and `libvorbis` as static libraries into `~/ffmpeg-windows`.
3. Cross-compiles FFmpeg `n9.0` statically for `mingw32`, with `avdevice`, `avfilter`, programs, docs and hardware acceleration (DXVA2, D3D11VA/D3D12VA, MediaFoundation) disabled to keep build time down. The result is cached, keyed on the FFmpeg tag, a manual version marker in the cache key and a hash of the workflow file, so changing configure flags invalidates the cache automatically.
4. Writes `.cargo/config.toml` with the mingw linker and the system/codec link arguments (grouped with `--start-group`/`--end-group` to resolve static library ordering).
5. Sets `BINDGEN_EXTRA_CLANG_ARGS` to skip AMX/SIMD intrinsic headers that break `bindgen` on Clang 21 targeting mingw.
6. Points `ffmpeg-sys-next` at the cross-compiled FFmpeg via `FFMPEG_DIR` and runs `maturin build --release --target x86_64-pc-windows-gnu`.
7. Uploads the wheel as the `media-engine-windows-abi3` artifact.

**Keeping `FFMPEG_TAG` and `ffmpeg-next` in sync:** `ffmpeg-next` contains hand-maintained exhaustive `match` statements over FFmpeg's C enums (pixel formats, codec IDs, etc.). If the FFmpeg version compiled in CI is newer than what `ffmpeg-next` was written against, new enum variants can trigger an `E0004` non-exhaustive-patterns compile error. When bumping `FFMPEG_TAG`, bump `ffmpeg-next` (and transitively `ffmpeg-sys-next`) in `Cargo.toml` too. Currently `FFMPEG_TAG: n9.0` ↔ `ffmpeg-next 9.0.0` / `ffmpeg-sys-next =9.0.1`.

**Adding a codec library:** build it statically into `~/ffmpeg-windows` in its own workflow step, add `--enable-lib<name>` to FFmpeg's `configure`, and add its `-l<name>` flag to the linker group in `.cargo/config.toml`. The workflow hash in the cache key makes the FFmpeg cache rebuild automatically.

**Debugging a failed CI run:**
- FFmpeg `configure`/compile failure → check `ffmpeg-src/ffbuild/config.log`, dumped automatically on failure (last 200 lines).
- Rust/linker failure → check the `stderr`/`stdout` section of the `maturin` step (`-vv` is enabled); this is usually a clearer signal once FFmpeg itself has built.

## Testing

`player_engine_test.py` is a PySide6 testbed for `PlayerEngine` (not production player code). It covers video and audio-only playback, seek, loop, frame-step, volume/mute, and a seekbar hover thumbnail built on `VideoDecoder` in its own worker thread.

```bash
pip install PySide6 numpy
# put media_engine.pyd / .so next to the script (or on PYTHONPATH)
python player_engine_test.py
```

## Project structure

```
Cargo.toml                    # Dependencies, per-target FFmpeg feature split
src/lib.rs                    # All PyO3 bindings and media logic
player_engine_test.py         # PySide6 testbed for PlayerEngine
.github/workflows/build.yml   # Windows cross-compile pipeline
LICENSE                       # MIT
```

## Design notes

- **No lock on the audio callback.** Samples move from the decode thread to the real-time `cpal` callback through an SPSC ring buffer, not a `Mutex<VecDeque<f32>>`. EQ parameters reach the callback via a sequence counter plus `try_lock()`, so it never blocks. The time-stretcher's buffers are preallocated; its capacity is sized for the full 0.25–4.0 rate range.
- **Separate video and audio threads, each with its own file handle.** Seeks are signalled with a generation counter rather than a consumed `Option`, so both threads can independently apply and acknowledge each seek. `play()` and `seek()` wait (bounded, ~450 ms) until both sides are ready before the audio stream is started, which avoids audio running ahead of video.
- **Pause is not a discontinuity.** `pause()` freezes the clock without flushing the readahead buffer; `seek`, `step_frame` and loop-restart reset the clock *and* flush audio.
- **Audio is the master clock.** It counts source frames actually played (rate-adjusted), and keeps advancing after audio EOF so trailing video frames can finish.
- **`rayon` for embarrassingly parallel chunked work.** Envelope/RMS computation, BPM energy/autocorrelation and loudness are the spots that benefit from `par_iter()` without a manually managed thread pool.
- **Stream copy vs. transcode in `convert_media`.** Stream-copied tracks trim to the nearest keyframe before the requested start (fast, but not frame-exact; same limitation as `ffmpeg -ss` before `-i`). Frame-exact trimming requires transcoding that track.
- **Resampling avoids mid-stream `flush()`.** Output frames are allocated large enough for upsampling, and leftover filter delay carries over to the next call, which avoids per-frame clicks.
- **`generate_thumbnails` builds one scaler for the whole batch**, rather than re-deriving it per frame like `VideoDecoder::seek_frame`, which matters when generating many thumbnails at once.

## License

MIT. See [LICENSE](LICENSE). Copyright (c) 2026 Danx Exodus.
