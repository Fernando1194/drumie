# Third-party notices

Drumie is free to use and closed source (see [LICENSE](LICENSE)). It is built on the following
software and models, each under its own license. Their licenses apply to them, not Drumie's.

## Models

| Component | Used for | License |
|---|---|---|
| ADTOF drum transcription model (Mickaël Zehren, Marco Alunno, Paolo Bientinesi), PyTorch port by Xavier Riley | Finding every drum hit | Weights: **CC BY-NC-SA 4.0** (non-commercial, share-alike). Installed separately; the weights stay under their own license. |
| Demucs (Alexandre Défossez, Meta AI), models htdemucs_6s and htdemucs_ft | Separating the drums from the song | MIT |
| Basic Pitch (Spotify) | Note transcription for other instruments (older page) | Apache 2.0 |

## Libraries and tools

| Component | License |
|---|---|
| PyTorch, torchaudio | BSD-style |
| NumPy, SciPy, soundfile | BSD |
| librosa | ISC |
| pretty_midi | MIT |
| ONNX Runtime | MIT |
| yt-dlp | Unlicense (public domain) |
| FastAPI | MIT |
| uvicorn | BSD 3-Clause |
| python-multipart | Apache 2.0 |
| Gradio (older all-instruments page) | Apache 2.0 |
| FFmpeg (installed on your system, called as a separate program) | LGPL 2.1+ or GPL, depending on the build |
| Barlow, Barlow Condensed, JetBrains Mono fonts (loaded from Google Fonts) | SIL Open Font License 1.1 |

The full license texts come with each package (in your Python environment, `pip show -f <name>`)
and are on each project's page. If you think a notice here is missing or wrong, please open an
issue on the public Drumie page.
