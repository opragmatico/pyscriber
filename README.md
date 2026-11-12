# Pyscriber

**Offline, private push-to-talk speech-to-text for your desktop — built with Python, PySide6 and Hugging Face transformers.**

Press a shortcut, speak, release, and your words are pasted into whatever app you are using. Audio never leaves your computer.

Pyscriber is a Python re-implementation of [Handy](https://github.com/cjpais/Handy) by CJ Pais. Handy is a Tauri (Rust + React) app that runs many model families through `transcribe.cpp`. Pyscriber keeps Handy's dictation behaviour but deliberately narrows the speech stack to **two NVIDIA models running locally on PyTorch through Hugging Face [`transformers`](https://huggingface.co/docs/transformers)**, with a [PySide6](https://doc.qt.io/qtforpython-6/) UI.

Pyscriber has the same requirements as [Goscriber](https://github.com/opragmatico/goscriber), its Go sibling, and the two issue trackers line up: issue #N covers the same feature in both. Only the stack differs.

> **Status: early development.** Nothing is runnable yet. See the [roadmap](#roadmap) and the [issue tracker](https://github.com/opragmatico/pyscriber/issues).

## Supported models

| Model | Hugging Face | Languages | Mode | Notes |
| --- | --- | --- | --- | --- |
| **Nemotron Streaming 3.5** (0.6B) | [`nvidia/nemotron-3.5-asr-streaming-0.6b`](https://huggingface.co/nvidia/nemotron-3.5-asr-streaming-0.6b) | 40+ locales, `auto` detection | Cache-aware streaming RNNT | Default model. NVIDIA publishes transformers weights (`Nemotron3_5AsrForRNNT`). |
| **Parakeet TDT 0.6B v2** | [`nvidia/parakeet-tdt-0.6b-v2`](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v2) | English | Offline TDT (batch) | Fast, accurate English with punctuation and timestamps. Only a `.nemo` checkpoint is published; Pyscriber converts it for `ParakeetForTDT` and hosts the result (transformers officially documents its v3 sibling). |

Pyscriber ships **without any model**. On first run you choose one and download it (Nemotron Streaming 3.5 is about 2.5 GB, because NVIDIA publishes its transformers weights in fp32 only); nothing is bundled or downloaded automatically.

No other models are planned. Whisper, Canary, Moonshine and the rest of Handy's catalog are out of scope, and so is translation to English.

## How it will work

1. **Press** a global shortcut: hold to record and release to stop (push-to-talk), or tap to toggle.
2. **Speak**. A small overlay shows that Pyscriber is listening and, with Nemotron Streaming 3.5, the words as they are recognised.
3. **Release**. The model finishes the utterance locally on Apple Silicon (MPS), CUDA or the CPU.
4. **Get** the text pasted into the focused text field.

Following Handy, Pyscriber also plans transcription history, custom vocabulary, filler-word removal, an external transcription hook and optional LLM post-processing.

## Stack

| Concern | Library |
| --- | --- |
| UI, tray, overlay | [PySide6](https://doc.qt.io/qtforpython-6/) |
| Speech-to-text | [`transformers`](https://huggingface.co/docs/transformers) + PyTorch (MPS, CUDA or CPU) |
| Model downloads | [`huggingface_hub`](https://huggingface.co/docs/huggingface_hub) |
| Voice activity detection | Silero VAD (ONNX) + `onnxruntime` |
| Audio capture | `sounddevice` + `numpy`, resampled with `soxr` |
| Global shortcuts | `pynput` |
| Clipboard, paste and macOS APIs | `QClipboard`, platform key events, `pyobjc` on macOS |
| Settings | `pydantic` + `platformdirs` |
| History | SQLite (stdlib `sqlite3`) |
| Logging | stdlib `logging` |

See [docs/architecture.md](docs/architecture.md) for the package layout, the engine design, the dictation pipeline and the threading model.

## Roadmap

Work is split into five milestones, tracked as GitHub issues:

1. **M0 · Foundation**: Python project, CI, settings, logging, and a PySide6 app shell with a tray icon.
2. **M1 · Speech engine**: the pinned PyTorch + transformers stack, the engine (offline and streaming), the Parakeet TDT 0.6B v2 conversion, the two-model catalog and downloader. At the end of M1, `pyscriber transcribe file.wav` works with both models.
3. **M2 · MVP dictation**: microphone, VAD, global shortcut, paste, coordinator, overlay and macOS permissions. At the end of M2 you can dictate into any app.
4. **M3 · Live streaming & usable app**: live transcription in the overlay, language selection, custom vocabulary, engine process isolation, settings window, onboarding, history and the CLI.
5. **M4 · Distribution & platforms**: a signed macOS app and releases, Linux and Windows, launch at login, transcription hook, LLM post-processing and file transcription from the UI.

macOS on Apple Silicon is the first target. Linux and Windows follow in M4.

## Development

Setup instructions will land with the scaffolding issue ([#2](https://github.com/opragmatico/pyscriber/issues/2)). The plan is [uv](https://docs.astral.sh/uv/) for environments, `ruff` for lint and format, `pyright` for types and `pytest` + `pytest-qt` for tests.

`torch` and `transformers` will sit behind an `asr` extra, and only the engine package imports them, so most of the code and its tests run without PyTorch installed.

## Acknowledgements

- Pyscriber's design follows [Handy](https://github.com/cjpais/Handy) (MIT, © CJ Pais). No Handy source code is copied; this is an independent implementation in Python.
- Speech recognition runs on [Hugging Face transformers](https://github.com/huggingface/transformers) (Apache-2.0) and [PyTorch](https://pytorch.org) (BSD-3-Clause).
- The models are released by NVIDIA: Nemotron Streaming 3.5 under [OpenMDW 1.1](https://openmdw.ai/license/1-1/), Parakeet TDT 0.6B v2 under CC-BY-4.0. Silero VAD is MIT. See each model card for terms.

## License

[MIT](LICENSE)
