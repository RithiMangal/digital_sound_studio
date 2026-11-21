<div align="center">

# Digital Sound Studio

**Run any song through a 16-track DTS pipeline and get a high-quality MP3 back.**

![Rust](https://img.shields.io/badge/Rust-2024_edition-orange?logo=rust)
![egui](https://img.shields.io/badge/egui-0.37-blue)
![Platform](https://img.shields.io/badge/platform-Linux%20x86__64-lightgrey?logo=linux)
![ffmpeg](https://img.shields.io/badge/ffmpeg-required-green?logo=ffmpeg)
![Version](https://img.shields.io/badge/version-0.1.0-brightgreen)

A native, lightweight Linux desktop app — Rust + egui, no Electron, no browser runtime.

</div>

• Strategic Recommendation: Frames Lubuntu as the primary native development environment for guaranteed stability.
• Technical Breakdown: Bullet points highlighting the streamlined configuration, low resource footprint, and its ability to revive older hardware with zero lag.
• Veteran Insight: Concludes with an engaging nod to the fun of managing multiple Linux environments.

---

> "yentha songs or music neenga dwnld pani vechurunthaalum antha files yellathaiyum ithu moolama DTS panikitingana antha songs enhanceda irukum"
>
> *"Whatever songs or music you have downloaded and kept — if you convert all of those files to DTS through this app, those songs will sound enhanced."*
>
> — the project author, defining the vision behind Digital Sound Studio

---

## Overview

Digital Sound Studio is a single-window desktop converter for Linux. Drop in one or more local audio files — MP3, WAV, FLAC, OGG, M4A, AAC, OPUS, or anything else `ffmpeg` can decode — and the app runs each one through a deterministic **16-track DTS pipeline**:

1. **Upmix** the source to a 16-channel **9.1.6** spatial master.
2. **Split** that master into **16 discrete DTS streams** (`track01.dts` … `track16.dts`).
3. **Re-merge** the DTS tracks into a wide, full-bodied **stereo MP3** at LAME VBR quality 0.
4. **Clean up** every intermediate artifact, leaving only the final MP3.

The "16-track DTS" stage is real DTS encoding, not a label. The MP3 you keep is decoded from those 16 DTS streams.

> **Note on scope:** a single 16-channel DTS file is not possible with stock `ffmpeg` — the `dca` encoder caps at 5.1(side). Digital Sound Studio therefore produces 16 valid mono DTS files as its intermediate format, then folds them back into a stereo MP3. In v0.1.0 the intermediate WAV and DTS folder are deleted after a successful merge; the MP3 is the deliverable.

## Why "enhanced"?

The quote above is the design brief, and it deserves an honest technical answer. The app does not use AI or a black-box remaster service. What it does is deterministic DSP:

- **Spatial re-imaging** — the stereo field is expanded through a 9.1.6 upmix matrix, then folded back to stereo using the surround and height channels (`BL`, `BR`, `SL`, `SR`, `TFL`, `TFR`, …). Instruments that share the centre in the source are spread wider across the L/R image.
- **Low-end reinforcement** — a dedicated LFE channel is derived from both input channels, adding weight without muddying the midrange.
- **Gain** — the final mix applies `volume=2.0`. On the reference test track this measured **≈ +4 dB** louder (RMS −25.5 dB → −21.7 dB, peak −20.3 dB → −16.6 dB) with no clipping.
- **High-quality encoding** — LAME VBR quality 0 targets roughly 245 kbps and typically lands between 180–260 kbps, which is the practical ceiling for MP3.

The result is a perceptually wider, fuller and louder presentation of the same recording — not a lossless remaster, and not a replacement for the original file.

## Features

- **Batch conversion** — select multiple files at once; they are processed sequentially with per-file progress.
- **Any input format** — whatever `ffmpeg`/`ffprobe` can read.
- **Real 16-track DTS path** — 9.1.6 upmix → `channelsplit` → 16 `dca` encodes → `amerge` → stereo MP3.
- **Automatic cleanup** — master WAV and `_16track_DTS/` are removed after each successful merge.
- **Auto-clear** — the selected-file list resets when the batch finishes, ready for the next run.
- **Live 4-step status bar** — `Step 1/4` → `2/4` → `3/4` → `4/4 · Done`, including output paths.
- **Encoder detection** — the UI reads `ffmpeg -encoders` and shows the real encoder name (typically `dca`), not a guess.
- **Borderless neon UI** — always opens maximized, custom min/max/close controls, red/green glowing title, animated backdrop.
- **No telemetry** — the only network action is the optional **Contact** button, which opens the author's GitHub in your browser.

## Screenshot

![Digital Sound Studio — maximized neon UI](assets/screenshot.png)

## How it works

```
┌──────────────────────┐
│  Source audio file   │  mp3 / wav / flac / ogg / m4a / …
└──────────┬───────────┘
           │  Stage 1 — pan=9.1.6
           ▼
┌──────────────────────┐
│ 16-channel master    │  <stem>_16ch_master.wav  (pcm_s16le)
└──────────┬───────────┘
           │  Stage 2 — channelsplit
           ▼
┌──────────────────────┐
│ 16 × mono DTS files  │  <stem>_16track_DTS/track01…16.dts  (dca, -strict -2)
└──────────┬───────────┘
           │  Stage 3 — amerge → aformat → pan=stereo → volume=2.0
           ▼
┌──────────────────────┐
│ Final MP3            │  <stem>_HQ.mp3  (libmp3lame, -q:a 0)
└──────────┬───────────┘
           │  Stage 4 — cleanup
           ▼
   master WAV + DTS folder removed
```

### Stage 1 — Spatial upmix to 9.1.6

The input channel count is probed with `ffprobe`. Stereo sources use a matrix that keeps left/right separation while feeding the centre, LFE, surrounds and height channels. Mono sources use a dedicated matrix driven entirely by `c0`.

```bash
ffmpeg -i song.mp3 \
  -af "pan=9.1.6|FL=0.85*c0+0.15*c1|FR=0.15*c0+0.85*c1|FC=0.45*c0+0.45*c1|..." \
  -map_metadata 0 -c:a pcm_s16le song_16ch_master.wav
```

<details>
<summary><strong>Exact matrices used by the app</strong></summary>

**Stereo input → 9.1.6**

```
FL=0.85*c0+0.15*c1 | FR=0.15*c0+0.85*c1 | FC=0.45*c0+0.45*c1 | LFE=0.35*c0+0.35*c1
BL=0.55*c0+0.25*c1 | BR=0.25*c0+0.55*c1 | FLC=0.45*c0 | FRC=0.45*c1
SL=0.35*c0+0.15*c1 | SR=0.15*c0+0.35*c1
TFL=0.25*c0+0.05*c1 | TFR=0.05*c0+0.25*c1
TBL=0.18*c0+0.12*c1 | TBR=0.12*c0+0.18*c1
TSL=0.15*c0 | TSR=0.15*c1
```

**Mono input → 9.1.6**

```
FL=0.9*c0 | FR=0.9*c0 | FC=0.85*c0 | LFE=0.7*c0
BL=0.6*c0 | BR=0.6*c0 | FLC=0.5*c0 | FRC=0.5*c0
SL=0.4*c0 | SR=0.4*c0 | TFL=0.35*c0 | TFR=0.35*c0
TBL=0.3*c0 | TBR=0.3*c0 | TSL=0.2*c0 | TSR=0.2*c0
```

</details>

### Stage 2 — Split into 16 discrete DTS tracks

One `channelsplit` pass produces 16 mono streams, each encoded with `dca`:

```bash
ffmpeg -i song_16ch_master.wav \
  -filter_complex "channelsplit=channel_layout=9.1.6[TR1][TR2]…[TR16]" \
  -map "[TR1]" -c:a dca -strict -2 song_16track_DTS/track01.dts \
  -map "[TR2]" -c:a dca -strict -2 song_16track_DTS/track02.dts \
  …
  -map "[TR16]" -c:a dca -strict -2 song_16track_DTS/track16.dts
```

Two non-obvious details matter here:

- `-strict -2` **must come after** `-c:a dca`. The DTS encoder is marked experimental by ffmpeg; placing the flag earlier makes ffmpeg exit with code 88.
- One 16-channel DTS file is impossible with stock `ffmpeg`: `dca` supports at most 5.1(side). Sixteen discrete mono streams are the only faithful workaround.

### Stage 3 — Merge and encode the HQ MP3

The 16 DTS files are decoded, merged back into a 16-channel stream, given the 9.1.6 layout, folded to stereo with a wide matrix, and encoded:

```bash
ffmpeg -i track01.dts … -i track16.dts \
  -filter_complex "[0:a]…[15:a]amerge=inputs=16,aformat=channel_layouts=9.1.6,pan=stereo|FL=0.7*c0+0.4*c4+0.5*c8+0.35*c10+0.25*c12+0.3*c6+0.08*c2|FR=0.7*c1+0.4*c5+0.5*c9+0.35*c11+0.25*c13+0.3*c7+0.08*c2,volume=2.0" \
  -c:a libmp3lame -q:a 0 song_HQ.mp3
```

The `aformat=channel_layouts=9.1.6` step is **not optional**. `amerge` concatenates channels but does not preserve a spatial layout; without re-tagging the stream, `pan` maps nearly everything to the centre and the result sounds flat and "all the same". This single filter is the difference between a proper wide mix and a mono-ish mush.

### Stage 4 — Cleanup

After a successful merge, the app deletes:

- `<stem>_16ch_master.wav`
- `<stem>_16track_DTS/` (the whole folder)

Only `<stem>_HQ.mp3` remains. The UI then clears the selected-file list automatically.

## The 9.1.6 channel order

| Index | Channel | Index | Channel |
|:---:|:---|:---:|:---|
| c0 | FL — front left | c8 | SL — side left |
| c1 | FR — front right | c9 | SR — side right |
| c2 | FC — front centre | c10 | TFL — top front left |
| c3 | LFE — low frequency | c11 | TFR — top front right |
| c4 | BL — back left | c12 | TBL — top back left |
| c5 | BR — back right | c13 | TBR — top back right |
| c6 | FLC — front left of centre | c14 | TSL — top side left |
| c7 | FRC — front right of centre | c15 | TSR — top side right |

## Requirements

**Runtime**

- Linux **x86_64** (glibc-based distributions: KDE Plasma, ParrotSec, Ubuntu, Debian, Fedora, Arch, …)
- `ffmpeg` and `ffprobe` on `PATH`, built with:
  - `libmp3lame` (MP3 encoding)
  - `dca` (DTS encoding — present in standard ffmpeg builds)
- No other runtime dependencies. The AppImage bundles the Rust binary.

**Build**

- Rust toolchain (edition 2024, stable)
- `pkg-config`, `libxkbcommon-dev`, and the usual X11/Wayland development headers
- `cargo-audit` (recommended; run before every release)

## Install & run

```bash
chmod +x digital_sound_studio-x86_64.AppImage
./digital_sound_studio-x86_64.AppImage
```

If FUSE is unavailable on your system:

```bash
./digital_sound_studio-x86_64.AppImage --appimage-extract-and-run
```

The window opens borderless and maximized on the current monitor.

## Usage

1. **Launch** the app. The Encoder line shows `dca` once background discovery finishes.
2. Click **Select Song** and pick one or more audio files (multi-select is supported).
3. *(Optional)* Click **Pick Output Folder**. If skipped, output goes next to each source file.
4. Click **Convert to MP3**.
5. Watch the status bar:
   - `Step 1/4 · File i/N — building 16-track DTS master…`
   - `Step 2/4 · File i/N — splitting into 16 DTS track files…`
   - `Step 3/4 · File i/N — merging 16 DTS → high-quality MP3…`
   - `Step 4/4 · Done — converted N MP3 file(s): <paths>`
6. The selected-file list clears automatically when the batch completes.

Each source `song.mp3` produces `song_HQ.mp3`. If a step fails, the batch stops and the status bar shows the ffmpeg error tail.

## Output files

| File | Lifetime | Purpose |
|---|---|---|
| `<stem>_HQ.mp3` | **Kept** | Final high-quality stereo MP3 (LAME VBR q0) |
| `<stem>_16ch_master.wav` | Deleted after merge | 16-channel 9.1.6 spatial master |
| `<stem>_16track_DTS/track01…16.dts` | Deleted after merge | 16 discrete mono DTS streams |

> **Keeping the DTS intermediates:** the cleanup is two lines in `src/converter.rs` (`fs::remove_file(&master)` and `fs::remove_dir_all(&dts_dir)`). Remove or gate them if you want to archive the 16 DTS tracks. They are valid DTS streams and can be played individually, but they are 16 separate mono files — not a single surround container.

## Build from source

```bash
git clone <your-repo-url> DigitalSoundStudio
cd DigitalSoundStudio

cargo build --release
cargo test
cargo audit

./target/release/digital_sound_studio
```

### Packaging the AppImage

The release artifact is built with `appimagetool` from an AppDir containing the binary, icon, `.desktop` entry and `AppRun`:

```bash
APPDIR=out/DigitalSoundStudio-AppDir
mkdir -p "$APPDIR/usr/bin" "$APPDIR/usr/share/icons/hicolor/256x256/apps"
cp target/release/digital_sound_studio "$APPDIR/usr/bin/"
cp assets/icon.png "$APPDIR/usr/share/icons/hicolor/256x256/apps/digital_sound_studio.png"
cp assets/icon.png "$APPDIR/digital_sound_studio.png"
# add digital_sound_studio.desktop + AppRun, then:
appimagetool "$APPDIR" digital_sound_studio-x86_64.AppImage
```

## Tests & security

```bash
cargo test          # unit tests (encoder detection, badge layout with real font metrics)
cargo audit         # required before every AppImage build
```

The current audit exits 0. Four warnings remain from the transitive `eframe → accesskit → atspi → zbus/zvariant` chain; they are `unmaintained` advisories, not vulnerabilities, and are allowed in the audit policy.

## Project structure

```
DigitalSoundStudio/
├── Cargo.toml               # package metadata + dependencies
├── Cargo.lock
├── assets/
│   ├── icon.png             # app icon
│   └── screenshot.png
├── src/
│   ├── main.rs              # eframe app, maximized window, status polling
│   ├── converter.rs         # ffmpeg pipeline, batch loop, cleanup, discovery
│   └── theme.rs             # neon theme, title bar, badge, status bar
├── out/                     # AppDir staging (build artifact)
└── digital_sound_studio-x86_64.AppImage
```

## Design notes

- **UI thread never blocks.** Encoder discovery and conversion each run on a background thread; status messages travel through an `Arc<Mutex<Option<String>>>` slot polled at ~80 ms while busy.
- **Sequential batches, not parallel.** DTS encoding is CPU-heavy; running files one at a time keeps memory flat and progress deterministic.
- **Deterministic DSP.** No ML models, no cloud calls, no randomness — the same input always produces the same output.
- **Maximized is enforced twice.** `ViewportBuilder::with_maximized(true)` plus a `ViewportCommand::Maximized(true)` on the first frame, because some window managers ignore the builder hint.
- **Frame-rate-friendly.** The neon pulse and status bar repaint only as often as they need to; the app idles at ~350 ms repaint intervals.

## Performance & disk

- Each file requires **three ffmpeg passes** (master, split, merge). On a modern x86_64 CPU this is typically at or faster than real time; long orchestral tracks take proportionally longer.
- The 16-channel 16-bit/44.1 kHz master is large: roughly **85 MB per minute** of audio. Cleanup runs per file, so batch disk usage stays bounded to one master at a time.
- DTS files are small relative to the master, but are also removed after the merge.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `Encoder: not found` | `ffmpeg` missing, or built without the `dca` encoder. Install a standard ffmpeg build. |
| ffmpeg exits with code 88 | `-strict -2` was placed before `-c:a dca`; the app already orders it correctly. |
| Output sounds flat / centred | The `aformat=channel_layouts=9.1.6` filter is missing from the merge chain. |
| AppImage will not start | FUSE missing → use `--appimage-extract-and-run`. |
| Window not maximized on start | Some minimal WMs ignore both hints; use your WM's maximize shortcut. |
| `xkbcommon` / `XKB-542` locale errors in console | Harmless container/desktop noise; the UI still works. |

## Limitations

- The final deliverable is **stereo MP3**; MP3 cannot carry 16 discrete channels.
- A single 16-channel DTS file is not produced — stock `dca` supports 5.1(side) at most.
- Local files only. DRM-protected streams and network URLs are out of scope.
- MP3 is lossy; keep your original files if you need archival quality.
- "Enhanced" means spatial re-imaging and gain, not restoration or AI upscaling.

## License

No `LICENSE` file is currently included in this repository. Until one is added, all rights are reserved by the author. Add a license before distributing or accepting contributions publicly.
