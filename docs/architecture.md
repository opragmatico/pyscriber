# Architecture

Pyscriber is a Python re-implementation of [Handy](https://github.com/cjpais/Handy): press a shortcut, speak, and the transcription is pasted into whatever app has focus, fully offline.

Handy is built with Tauri (Rust backend + React UI) and runs a broad catalog of GGUF models through `transcribe.cpp`. Pyscriber keeps the same product behaviour, swaps the stack for Python + PySide6, and narrows speech recognition to two NVIDIA models run on PyTorch through Hugging Face `transformers`. Its requirements match [Goscriber](https://github.com/opragmatico/goscriber), the Go port, which runs the same two models through NeMo-Speech.cpp.

| Concern | Handy | Pyscriber |
| --- | --- | --- |
| UI | Tauri webview + React | PySide6 (Qt Widgets) |
| Tray / overlay | Tauri tray, NSPanel / webview overlay | `QSystemTrayIcon`; frameless always-on-top `QWidget`, made non-activating on macOS through pyobjc |
| Speech-to-text | `transcribe-cpp` (many families) | `transformers` on PyTorch (MPS / CUDA / CPU), two models only |
| Engine isolation | `engine_supervisor`: worker child process | Same idea: `pyscriber engine-worker` child process; the app process never imports torch |
| Model download | Catalog + downloader with mirrors | Packaged two-model catalog, `huggingface_hub` downloads at pinned revisions, SHA-256 verification |
| Voice activity detection | Silero VAD v6 (GGUF) | Silero VAD v6 (ONNX) via `onnxruntime` |
| Audio capture | `handy-recorder` (cpal) | `sounddevice` (PortAudio) + `numpy`; the engine resamples with `soxr` |
| Global shortcuts | `handy-keys` (CGEventTap) / `rdev` | `pynput` (a Quartz event tap on macOS) |
| Paste / typing | `enigo`, platform paste code | `QClipboard` + platform key events (pyobjc on macOS); `xdotool` / `wtype` / `ydotool` on Linux |
| Settings | `tauri-plugin-store` (JSON) | `pydantic` models persisted as JSON under `platformdirs` |
| History | SQLite (`rusqlite`) | SQLite (`sqlite3`) + WAV files |
| LLM post-processing | OpenAI-compatible HTTP clients | `httpx` OpenAI-compatible client; API keys in the OS keychain through `keyring` |
| Single instance / CLI | `tauri-plugin-single-instance`, `clap` | `QLocalServer` / `QLocalSocket`, `argparse` |
| Sound effects | `rodio` | `sounddevice` or `QSoundEffect` |

## Models

| | Nemotron Streaming 3.5 | Parakeet TDT 0.6B v2 |
| --- | --- | --- |
| Source | [`nvidia/nemotron-3.5-asr-streaming-0.6b`](https://huggingface.co/nvidia/nemotron-3.5-asr-streaming-0.6b) | [`nvidia/parakeet-tdt-0.6b-v2`](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v2) |
| Architecture | Cache-aware FastConformer-RNNT with language-ID prompts | FastConformer-TDT (token-and-duration transducer), full attention |
| transformers class | `Nemotron3_5AsrForRNNT` (transformers 5.13+) | `ParakeetForTDT` |
| Languages | 40+ locales, or `auto` | English, with punctuation and capitalisation |
| Decoding | Streaming with interim results (cache-aware, 80 to 1,120 ms chunks) | Offline only; up to 24 minutes in one pass, per the model card |
| Weights | NVIDIA's transformers weights (fp32 safetensors, about 2.55 GB) at a pinned revision | Converted by us from `.nemo` with transformers' converter, then hosted at a pinned revision |
| License | OpenMDW 1.1 | CC-BY-4.0 |
| Live text | Interim results in the overlay | Optional chunked decode at pauses while recording |

The `.nemo` and GGUF files NVIDIA publishes next to the transformers weights, and Handy's own GGUFs, target other runtimes; Pyscriber downloads only the transformers files.

**No model is bundled.** The app, its release artifacts and the repository contain no weights. Settings start with no selected model; the user chooses one in onboarding or on the Models page and downloads it, and the Silero VAD companion comes with that first download. Until then the shortcut and tray point to the download step instead of recording.

## Package layout

```text
src/pyscriber/
├── __main__.py            # entry point: GUI by default; subcommands `transcribe`, `models`, `engine-worker`
├── app.py                 # QApplication bootstrap and object wiring
├── cli.py                 # argparse flags, subcommands, remote-control client
├── single_instance.py     # QLocalServer / QLocalSocket
├── notices.py             # one place to report background errors to the UI
├── logs.py                # logging setup, rotating file, in-memory ring for the log viewer
├── config/
│   ├── paths.py           # settings, history, recordings, logs and model-cache directories
│   └── settings.py        # pydantic settings, JSON persistence, migrations
├── audio/
│   ├── devices.py         # input / output device enumeration
│   ├── recorder.py        # microphone capture, ring buffer, levels
│   ├── vad.py             # Silero VAD with onnxruntime
│   ├── feedback.py        # start / stop sounds, mute while recording
│   └── wav.py             # WAV read / write for history and the CLI
├── asr/
│   ├── engine.py          # Engine / Stream protocols, Result, Word, Options, typed errors
│   ├── fake.py            # scripted engine for tests and torch-free runs
│   ├── check.py           # `python -m pyscriber.asr.check`: versions and devices
│   ├── hf/                # transformers backend (the only package that imports torch)
│   └── worker/            # supervisor (parent) + worker (child) and their protocol
├── models/
│   ├── catalog.json       # the two models (+ Silero VAD), pinned revisions and checksums
│   ├── catalog.py
│   └── manager.py         # download, verify, delete, status
├── pipeline/
│   └── coordinator.py     # recording → transcription → output state machine
├── text/
│   ├── cleanup.py         # filler words, whitespace and punctuation
│   ├── vocabulary.py      # custom-word correction
│   └── hook.py            # external transcription hook
├── llm.py                 # optional post-processing
├── output/
│   ├── paste.py           # clipboard paste with restore
│   └── typing.py          # direct typing / Linux tools
├── hotkeys/
│   └── listener.py        # global shortcuts, push-to-talk and toggle
├── history/
│   └── store.py           # SQLite history + saved recordings
├── platform/              # permissions and native window tweaks (per-OS, pyobjc on macOS)
└── ui/
    ├── tray.py
    ├── overlay.py
    ├── onboarding.py
    └── settings/          # one module per settings page
scripts/
├── convert-models.sh      # reproduces the Parakeet TDT 0.6B v2 and Silero VAD artifacts
└── bench.py               # both models on MPS and CPU over the fixtures
testdata/                  # WAV fixtures and reference transcripts
```

## The engine

`pyscriber.asr.hf` wraps `transformers`:

- **Load**: `AutoProcessor` and `AutoModelForRNNT` from the model's local snapshot with `local_files_only=True`, so loading never downloads. The device follows the accelerator setting (MPS or CUDA when available, otherwise the CPU), with fp16 or bf16 on a GPU and fp32 on the CPU.
- **Offline**: the processor (with the language, for Nemotron) and `generate` on the whole clip. Word timestamps come from the TDT durations or RNNT frame indices, and the detected language from the `<xx-XX>` tag Nemotron emits with `auto`.
- **Streaming** (Nemotron only): `set_num_lookahead_tokens` picks the chunk size, which is the latency preset. `generate` runs on the inference thread and consumes a generator that yields fixed-size mel chunks as audio arrives; a `TextIteratorStreamer` delivers interim text. `finish` pads the last chunk and waits for the final result.
- **Options**: language (a locale or `auto`), word timestamps, and the custom-vocabulary phrases.

Rules the engine follows:

- Only `pyscriber.asr.hf`, and the worker process that hosts it, imports `torch` or `transformers`. Everything else imports and tests without the `asr` extra, using `asr.fake`.
- One thread owns a loaded model; PyTorch is never called from two threads at once.
- Audio crosses the engine boundary as float32 numpy arrays at any sample rate; the engine resamples to the model's 16 kHz with `soxr`.
- `generate` cannot be interrupted: cancelling discards the result, or kills the worker for a long offline run.
- Unloading drops every reference and empties the MPS or CUDA cache; only ending the worker returns all memory to the OS.

### Process isolation

An MPS assertion, a segfault in a native kernel or an out-of-memory kill takes the whole Python process with it. Following Handy's `engine_supervisor`, the model runs in a child process: the same interpreter started as `python -m pyscriber engine-worker`. The parent's `asr.worker` supervisor implements the same `Engine` protocol over a framed stdin/stdout protocol (load, unload, recognize, stream push / next / finish, cancel), restarts a crashed worker, and retries on the CPU when the GPU was at fault. A worker lives exactly as long as its loaded model, so unloading returns all memory to the OS. As a side effect the app process never imports torch, so it starts quickly and inference never competes with the UI for the GIL.

## Dictation pipeline

```text
shortcut down ─► Recorder.start()
                 │  float32 frames ─► level meter ─► overlay
                 │                └► [streaming] Stream.push ─► next ─► interim text ─► overlay
shortcut up   ─► Recorder.stop()
                 └► [streaming] Stream.finish ─► final result
                    [batch]     VAD trims silence ─► recognize(whole clip, or the last chunk when chunking)
                    └► text cleanup ─► [LLM post-processing] ─► [hook] ─► paste / type ─► history
```

The `Coordinator` owns a single state machine: `idle → recording → transcribing → post_processing → outputting → idle`. A cancel from any state returns to `idle` and discards the session. Only one session is active at a time.

## Threading model

- **Qt main thread**: all widgets, tray, overlay and clipboard access. Other threads reach the UI only through Qt signals with queued connections.
- **PortAudio callback thread**: copies frames into a ring buffer and returns; never blocks or calls into the engine.
- **Hotkey listener thread** (`pynput`): emits Qt signals through a `QObject` bridge, so handlers run on the main thread.
- **Inference thread**: one thread owns the engine client and serialises load, unload and recognition; a streaming session pumps audio from the ring buffer to `push` / `next`.
- **Background pool** (`QThreadPool`): model downloads, history I/O, LLM requests.
- **Engine worker process**: runs PyTorch and talks to the inference thread over stdin/stdout.

Nothing outside the main thread touches a widget.

## Data locations

Resolved with `platformdirs` for the app name `pyscriber`:

- **Settings**: `user_config_dir/settings.json`
- **History**: `user_data_dir/history.sqlite3` and `user_data_dir/recordings/*.wav`
- **Logs**: `user_log_dir/pyscriber.log`
- **Models**: the Hugging Face cache (`HF_HOME`, default `~/.cache/huggingface/hub`), at the catalog's pinned revisions. Files another tool already downloaded are listed as available once their checksums match, but never selected automatically.

With `PYSCRIBER_HOME` set (tests and portable use), all of these, including the model cache, live under that directory.
