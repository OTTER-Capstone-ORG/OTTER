# OTTER Proof of Concept
***Open Text Transcription Editing Resource***

<img src="assets/icon.png" width="35%" alt="OTTER icon"> <img src="assets/Screenshot1.png" width="50%" align="right" alt="OTTER screenshot">
## Overview

This repository contains a Proof of Concept (PoC) for OTTER, the **O**pen **T**ext **T**ranscription **E**diting **R**esource. It is for use with CSUMB Computer Science Capstone Program.

OTTER uses a local automatic speech recognition (ASR) model so users can edit audio by editing text rather than only by cutting a waveform. The desktop app is Electron (TypeScript UI) plus a Python transcription pipeline. Audio and text never leave the machine.

It exists to:

+ Ground discussions in a working system
+ Demonstrate feasibility of transcript-driven editing
+ Demonstrate concrete mechanisms that can be used
+ Highlight real technical tradeoffs

It is:

+ **Able** to stage, remove, restore, reorder, and export transcript-backed edits (source audio files are never overwritten)
+ **Not** thoroughly tested or documented
+ **Not** production-ready code

## Scope

### What This Prototype Demonstrates

+ Transcription
	+ Runs locally; no cloud ASR is required.
	+ Produces word-level timestamps from a Whisper-family model (`faster_whisper` or `whisperx_vad`).
	+ Progress is streamed from the Python process (`PROGRESS:NN` on stderr) to the UI.
	+ Pause, resume, and cancel are cooperative JSON lines on the child process stdin (cancel also force-kills after a timeout).
	+ Results are cached on disk (keyed by audio content and pipeline spec); override with `--no-cache` or clear via `clear-cache` (see CLI below).
	+ For long audio (~20+ minutes), `whisperx_vad` can use a parallel chunked path; other transcribers stay single-process.
+ Transcript-driven editing
	+ Imported files and microphone recordings are transcribed into **staged** panes, then appended into a **main transcript**.
	+ Edits use a piece table: words can be removed (kept as strikethrough), restored, and reordered. Undo/redo is snapshot-based.
	+ Source audio is not rewritten. Playback after edits is a disposable ffmpeg preview; **Save Edits** writes a new WAV.
	+ **Save EDL / Load EDL** persist piece-table project state as JSON (current format version 4; older versions are upgraded on load).
+ UI concepts
	+ Clicking a word seeks audio; playback highlights the current word independently of selection.
	+ Main waveform plus a detail waveform for fine-tuning a selected range.
	+ Select vs Move modes for highlighting vs drag-and-drop reorder.
+ Architecture
	+ Electron split: main process (files, ffmpeg, process spawn), renderer (UI), preload (`window.otter` IPC).
	+ Local Whisper-based ASR behind a JSON pipeline spec (presets under `otter_py/sample_specs/`).

### What This Prototype Intentionally Does Not Do

+ Handle multiple speakers (diarization)
+ Package into a self-contained installer with bundled Python/models
+ Provide a polished end-user experience
+ Edit source audio files in place
+ Keep the Python ASR process alive between jobs (`otter_py/worker.py` is a persistent-worker sketch; Electron still spawns `python -m otter_py.transcribe run` each time)

These gaps are appropriate capstone topics. More implementation detail lives under [`doc/`](doc/) (for example [`UnderstandingElectron.md`](doc/UnderstandingElectron.md), [`Feature_Implementation_Guide.md`](doc/Feature_Implementation_Guide.md), and [`User_Stories.md`](doc/User_Stories.md)). A separate Vite site lives in [`docs-site/`](docs-site/).

### Languages

The app is TypeScript in Electron; the transcription pipeline is Python.

+ TypeScript sources live in `src/` and compile to `dist/` during `npm start`.
+ Main and preload compile to CommonJS (Node context).
+ The renderer compiles to ES modules (browser context) to avoid `exports`/`require` issues.
+ Python runs as a **separate process** spawned by the main process. The renderer talks only to main through IPC (`window.otter`).

### Supported Audio Formats

The file picker accepts **WAV, MP3, and M4A**. Microphone recordings are converted to PCM WAV with ffmpeg. Waveform windows, previews, and exports are also PCM WAV so seeking stays predictable.

ffmpeg must be on `PATH`. Formats the picker does not list can still be converted first:

```
ffmpeg -y -i input.aifc -c:a pcm_s16le -ac 1 output.wav
```

Broader codec support, loudness normalization, and a single import pipeline are reasonable capstone work.

## Installing, Launching, and Using the PoC

### Requirements

+ Node.js (v18+ recommended)
+ Python 3.10+
+ [Electron](https://www.electronjs.org) (see `package.json`; currently `^41.2.0`)
+ [faster-whisper](https://pypi.org/project/faster-whisper/) for text transcription
+ [whisperx](https://pypi.org/project/whisperx/) for text transcription (often brings in PyTorch and related deps)
+ [pydash](https://pypi.org/project/pydash/)
+ [soundfile](https://pypi.org/project/soundfile/) (audio duration and I/O in the pipeline runner)
+ [NumPy](https://numpy.org/) (used by the parallel WhisperX path)
+ [FFmpeg](https://ffmpeg.org):
	+ Used for inspection, format conversion, waveform slices, edited preview, and export
	+ Must be available on the system PATH

### Installation & Running

1. Clone the repository (use your fork or upstream URL as appropriate)

	```
	git clone https://github.com/Austin-Metke/OTTER.git
	cd OTTER
	```

2. Install Node dependencies

	```
	npm install
	```

3. Set up Python environment

	```
	python3 -m venv .venv
	```

	Activate the venv: on macOS/Linux use `source .venv/bin/activate`; on Windows use `.venv\Scripts\activate`.

	```
	pip install pydash faster-whisper whisperx soundfile numpy
	```

	(`whisperx` and `faster-whisper` pull in their own stacks; install errors usually mean a missing system library or incompatible Python—see those projects’ docs.)

4. Install `ffmpeg`

   ```
   # This is system dependent. For example, on the mac you can use homebrew:
   brew install ffmpeg
   ```
   
5. Run the app

    **NOTE**: This PoC is not a cleanly packaged app. Run it from a shell where the Python virtual environment is already active, or ensure `python`/`python3` on your PATH can import `otter_py` (the Electron main process prefers `.venv` when present: `Scripts\python.exe` on Windows, `bin/python3` on Unix).

	```
	npm start
	```

    The `npm start` command compiles the TypeScript sources into `dist/` before launching Electron.


### Using the PoC

**Import and transcribe**

1. Click **Choose Audio…** and select a WAV, MP3, or M4A file.
1. Optionally pick a pipeline preset from **Quality** (every `*.json` in `otter_py/sample_specs` is listed). If none is chosen, `default_spec.json` is used.
1. Click **Transcribe**. The result lands in **Imported Clips** (staged), not immediately in the main transcript.
1. Click **Add to Transcript** to append the staged clip into **Main Transcript**.

**Record**

1. Choose a microphone, click **Record**, then **Stop Rec**.
1. Click **Transcribe Rec**. The result lands in **Recorded Clips**.
1. **Add to Transcript** appends it to main, same as an import.

**Navigate and play**

1. Click a word to seek; Shift-click (or drag in Select mode) to select a range.
1. The detail waveform shows audio around the selection; you can nudge region bounds there.
1. **Play** on the main waveform plays the *edited* preview (active words only). A separate playhead highlight follows the current word and does not change the selection.
1. Use **Pause** / **Stop** on the transcription controls while a job is running.

**Edit**

1. **Remove** marks selected words deleted (they remain visible for restore). **Restore** brings them back. Delete and `R` are keyboard shortcuts for those actions.
1. Switch to **Move Words** and drag a selection to reorder.
1. **Undo** / **Redo** revert piece-table snapshots.

**Save**

1. **Save EDL** writes project JSON (piece tables + audio paths). **Load EDL** restores it if those audio files still exist.
1. **Save Edits** exports a new WAV of the main transcript’s active words. Original media is only read.

**Developer controls**

+ Open **Developer Controls** for the pipeline log and optional custom spec JSON.
+ Check Developer Tools in Electron for renderer errors.

### Transcription CLI (Python)

The Electron app invokes the pipeline as a module from the repo root:

`python -m otter_py.transcribe <command>`

- `list` — print registered transcribers and post-processors (JSON on stdout).
- `run --audio <path> (--spec-file <path> | --spec-json '<json>')` — run a pipeline; progress lines go to stderr as `PROGRESS:NN`.
- `run ... --no-cache` — bypass the filesystem cache.
- `run ... --emit-meta` — emit full `{ words, meta }` instead of trimming to words plus `language`.
- `clear-cache` — delete cached pipeline results (cache lives under the system temp dir, or set `OTTER_CACHE_DIR`).

By default, `run` prints JSON on stdout with `words` and a detected `language` string (metadata is otherwise omitted for a smaller payload). The app parses this the same way.

`otter_py/worker.py` is a long-lived JSON-line worker meant to keep loaded models in RAM across jobs. It is **not** what Electron launches today.

## Architectural Notes

The system is two primary components: the Electron app and the transcription pipeline. See [Understanding Electron](doc/UnderstandingElectron.md) for process roles, and [Feature Implementation Guide](doc/Feature_Implementation_Guide.md) for editing, EDL, and recording code paths.

TypeScript under `src/` compiles to `dist/` on `npm start`. Main and preload are CommonJS; the renderer is ES modules.

Typical request path:

`renderer → preload (window.otter) → main → ffmpeg and/or Python → IPC events/promises → renderer`

The renderer should not import Node modules. Privileged work (dialogs, ffmpeg, Python) is centralized in `src/main.ts`. Preload currently also reads files as ArrayBuffers for WaveSurfer (`sandbox` is off); tightening that boundary is a known improvement.

Python is spawned per transcription (`python -m otter_py.transcribe run`). STDOUT is one JSON object; STDERR is logs plus `PROGRESS:` and `CONTROL:` lines. That split matters: library chatter must not land on stdout.

The Quality dropdown and Developer Controls let you swap pipeline JSON without changing app code, so people can work on new transcribers or post-processors in parallel.

Transcript timings are approximate. The detail waveform exists because ASR word bounds are not sample-perfect; nudging them is part of learning the technology.

### Transcription Pipeline

A run is one transcriber plus zero or more post-processors. Specs are JSON. The post list may use `"postprocessors"` (sample specs) or `"post"` (canonical alias).

Components in this PoC:

+ Transcription
	+ **faster_whisper**: `faster-whisper` package. Options such as model size, device, and compute type are set in the spec.
	+ **whisperx_vad**: `whisperx` plus Silero VAD and forced alignment. Heavier, typically better word timings.
+ Post-processing
	+ **clean_word_timings**: Tightens adjacent word boundaries (small overlaps and tiny gaps).
	+ **adjust_short_words**: Expands very short words leftward without overlapping the previous word.
	+ **filter_fillers**: Drops fillers such as "uh" / "um", optionally gated on confidence.
	+ **filter_low_confidence_words**: Removes or replaces low-confidence words.

Example spec (`faster_whisper` plus two post-processors):

```
{
  "transcriber": {
    "id": "faster_whisper",
    "opts": {
      "model": "small",
      "device": "cpu",
      "compute_type": "int8"
    }
  },
  "postprocessors": [
    {
      "id": "adjust_short_words",
      "opts": {
        "max_len": 0.30,
        "min_extend": 0.10
      }
    },
    {
      "id": "clean_word_timings",
      "opts": {
        "tiny_gap_ms": 300.0
      }
    }
  ]
}
```

New transcribers and post-processors register themselves in `otter_py/pipelines/` (import the module from the package `__init__.py`). Example directions: waveform-based boundary cleanup, or Whisper plus [MFA](https://montreal-forced-aligner.readthedocs.io/en/latest/index.html). Tuned spec files become the production defaults.

## TypeScript Context

TypeScript is a superset of JavaScript. Every valid JavaScript program is valid TypeScript; TypeScript adds static types. It compiles to JavaScript, so changes in `src/` must be rebuilt. `npm start` does that before launching Electron. Output lives in `dist/`.

Why we use TypeScript here:

+ Easier to understand and refactor as the project grows.
+ Catches common errors (wrong property names, wrong argument types) at compile time.
+ Better editor tooling (autocomplete, go-to-definition, inline documentation).

Special considerations for Electron in this project:

+ Safety:
	+ `strict: true` so TypeScript is a correctness tool, not just a hint system.
	+ ESLint for style and common mistakes. Run `npm run lint`.
+ Runtime contexts:
	+ Main/preload run in Node; the renderer runs in the browser.
	+ Main/preload compile to CommonJS; the renderer compiles to ES modules.
	+ The renderer talks to main through `window.otter`. Types for that API are declared in `src/renderer.ts`.

## License

This project is licensed under the [MIT License](LICENSE).

You are free to use, modify, and distribute this project under the terms of the MIT license. See the [LICENSE](LICENSE) file for more details.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
