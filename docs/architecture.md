# Architecture

Pyscriber is a Python re-implementation of [Handy](https://github.com/cjpais/Handy): press a shortcut, speak, and the transcription is pasted into whatever app has focus — fully offline.

Handy is built with Tauri (Rust backend + React UI) and runs GGML/GGUF models through `transcribe-cpp`. Pyscriber keeps the same product behaviour but swaps the stack:

| Concern | Handy | Pyscriber |
| --- | --- | --- |
| UI | Tauri webview + React | PySide6 (Qt Widgets) |
| Tray / overlay | Tauri tray, custom overlay window | `QSystemTrayIcon`, frameless always-on-top `QWidget` |
| Speech-to-text | `transcribe-cpp` (GGUF) | Hugging Face `transformers` ASR pipeline on PyTorch (MPS / CUDA / CPU) |
| Model download | Custom downloader + catalog | `huggingface_hub` (shared `~/.cache/huggingface/hub`) |
| Voice activity detection | Silero VAD v6 (GGUF) | Silero VAD (ONNX, fetched from the HF Hub) via `onnxruntime` |
| Audio capture | `handy-recorder` (cpal) | `sounddevice` (PortAudio) + `numpy`, resampled to 16 kHz mono |
| Global shortcuts | `rdev` / `handy-keys` | `pynput` |
| Paste / typing | `enigo`, platform paste code | `QClipboard` + `pynput` key events; `xdotool` / `wtype` / `ydotool` on Linux |
| Settings | `tauri-plugin-store` (JSON) | `pydantic` models persisted as JSON under `platformdirs` |
| History | SQLite (`rusqlite`) | SQLite (`sqlite3`) + WAV files |
| LLM post-processing | OpenAI-compatible HTTP clients | `httpx` OpenAI-compatible clients, plus an optional local HF text model |
| Single instance / CLI | `tauri-plugin-single-instance`, `clap` | `QLocalServer` / `QLocalSocket`, `argparse` |
| Sound effects | `rodio` | `QSoundEffect` (QtMultimedia) |

## Package layout

```text
src/pyscriber/
├── __main__.py            # `python -m pyscriber` / `pyscriber` entry point
├── app.py                 # QApplication bootstrap and object wiring
├── cli.py                 # argparse flags (startup + remote control)
├── single_instance.py     # QLocalServer / QLocalSocket
├── notices.py             # one place to report background errors
├── config/
│   ├── paths.py           # data, cache, log and model directories
│   └── settings.py        # pydantic settings, JSON persistence, migrations
├── audio/
│   ├── devices.py         # input / output device enumeration
│   ├── recorder.py        # microphone capture, ring buffer, levels
│   ├── vad.py             # Silero VAD
│   └── feedback.py        # start / stop sounds
├── asr/
│   ├── catalog.py         # curated model list (HF repo ids, sizes, languages)
│   ├── engine.py          # Engine protocol
│   └── hf_engine.py       # transformers pipeline backend
├── models/
│   └── manager.py         # download, verify, delete, list cached models
├── pipeline/
│   └── coordinator.py     # recording → transcription → output state machine
├── text/
│   ├── cleanup.py         # custom words, filler words, spacing
│   ├── hook.py            # external transcription hook
│   └── llm.py             # LLM post-processing
├── output/
│   ├── paste.py           # clipboard paste with restore
│   └── typing.py          # direct typing / Linux tools
├── hotkeys/
│   └── listener.py        # global shortcuts, push-to-talk and toggle
├── history/
│   └── store.py           # SQLite history + saved recordings
└── ui/
    ├── tray.py
    ├── overlay.py
    ├── onboarding.py
    └── settings/          # one module per settings page
```

## Dictation pipeline

```text
shortcut down ─► Recorder.start()
                 │  16 kHz mono frames ─► level meter ─► overlay
shortcut up   ─► Recorder.stop() ─► VAD trims silence
                 └► ASR engine (worker thread) ─► text cleanup
                    └► [LLM post-processing] ─► [hook] ─► paste / type ─► history
```

The `Coordinator` owns a single state machine: `idle → recording → transcribing → post_processing → outputting → idle`. A cancel from any state returns to `idle` and discards the session. Only one session is active at a time; a shortcut press while transcribing is ignored or queued according to settings.

## Threading model

- **Qt main thread** — all widgets, tray, overlay, clipboard access.
- **PortAudio callback thread** — copies frames into a lock-free ring buffer; never blocks.
- **Hotkey listener thread** (`pynput`) — emits Qt signals through a `QObject` bridge, so handlers always run on the main thread.
- **Inference worker** — one dedicated `QThread` owns the loaded model; jobs are queued to it so PyTorch is never called from two threads at once.
- **Background pool** (`QThreadPool`) — model downloads, history I/O, LLM requests.

Cross-thread communication uses Qt signals with queued connections. Nothing outside the main thread touches a widget.

## Data locations

Resolved with `platformdirs` for the app name `pyscriber`:

- **Settings** — `user_config_dir/settings.json`
- **History** — `user_data_dir/history.sqlite3` and `user_data_dir/recordings/*.wav`
- **Logs** — `user_log_dir/pyscriber.log`
- **Models** — the shared Hugging Face cache (`HF_HOME`, default `~/.cache/huggingface/hub`), so models downloaded by other tools are reused.
