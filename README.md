# Pyscriber

**Offline, private push-to-talk speech-to-text for your desktop — built with Python, PySide6 and Hugging Face.**

Press a shortcut, speak, release, and your words are pasted into whatever app you are using. Audio never leaves your computer.

Pyscriber is a Python re-implementation of [Handy](https://github.com/cjpais/Handy) by CJ Pais. Handy is a Tauri (Rust + React) app; Pyscriber keeps its behaviour but runs on PySide6 for the UI and Hugging Face `transformers` for speech recognition, so it is easy to hack on and to try any ASR model on the Hub.

> **Status: early development.** Nothing is runnable yet. See the [roadmap](#roadmap) and the [issue tracker](https://github.com/opragmatico/pyscriber/issues).

## How it will work

1. **Press** a global shortcut: hold to record and release to stop (push-to-talk), or tap to toggle.
2. **Speak** while a small overlay shows that Pyscriber is listening.
3. **Release**: silence is trimmed with Silero VAD and the audio is transcribed locally by a Hugging Face model (Whisper by default) on Apple Silicon (MPS), CUDA or CPU.
4. **Get** the text pasted into the focused text field.

Optional extras, following Handy: transcription history, custom vocabulary, filler-word removal, translation to English, LLM post-processing, and a user-defined transcription hook.

## Stack

| Concern | Library |
| --- | --- |
| UI, tray, overlay | [PySide6](https://doc.qt.io/qtforpython-6/) |
| Speech-to-text | [`transformers`](https://huggingface.co/docs/transformers) + PyTorch |
| Model downloads | [`huggingface_hub`](https://huggingface.co/docs/huggingface_hub) |
| Voice activity detection | Silero VAD (ONNX) + `onnxruntime` |
| Audio capture | `sounddevice` + `numpy` |
| Global shortcuts and key events | `pynput` |
| Settings | `pydantic` + `platformdirs` |

See [docs/architecture.md](docs/architecture.md) for the module layout, the dictation pipeline and the threading model.

## Roadmap

Work is split into four milestones, tracked as GitHub issues:

1. **M0 · Foundation** — project scaffolding, CI, settings, logging, app shell with tray icon.
2. **M1 · MVP dictation** — microphone capture, VAD, Hugging Face Whisper engine, model downloads, global shortcut, paste, overlay. At the end of M1 you can dictate into any app.
3. **M2 · Usable app** — settings window, onboarding, history, single-instance CLI, languages, custom words, file transcription.
4. **M3 · Advanced & distribution** — LLM post-processing, transcription hook, more ASR backends, streaming, Linux typing tools, autostart, packaged releases.

## Development

Setup instructions will land with the project scaffolding issue. The plan is [uv](https://docs.astral.sh/uv/) for environments, `ruff` for lint/format and `pytest` + `pytest-qt` for tests.

## Acknowledgements

Pyscriber's design follows [Handy](https://github.com/cjpais/Handy) (MIT, © CJ Pais). No Handy source code is copied; this is an independent implementation in Python.

## License

[MIT](LICENSE)
