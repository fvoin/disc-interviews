# Disc Interviews — design

macOS desktop app: insert a DVD with recorded interviews, see what is on it, listen,
save the audio, transcribe it locally, and produce a Word/HTML review copy with the
suspicious phrases highlighted in yellow. Standalone project (not part of ceoapp).

## Goals

- Detect an inserted disc automatically; show its interviews with thumbnails, duration
  and stream info without any clicking.
- One title (or one media file) = one interview. Optionally split a title further at
  long pauses (opt-in checkbox; may be slow).
- Buttons, in order: Extract → Save video (MP4)… → Transcribe → Create review document →
  Open results folder. "Run all" chains the steps for every interview on the disc.
  The disc is read exactly once: the extract step writes `video.mp4` (H.264 via
  VideoToolbox, libx264 fallback, + AAC); the 16 kHz wav for recognition, the player and
  the Save button all work from that file. Audio-only sources (mp3/wav discs) get
  `audio.m4a` instead and Save offers audio formats.
- Transcription runs locally (NVIDIA Parakeet TDT 0.6B v3 via parakeet-mlx; Whisper
  fallback). The review document is produced with a cloud LLM (Gemini / OpenAI /
  Anthropic) using an API key from Settings.
- The app downloads what it needs on demand (ASR model). ffmpeg/ffprobe are bundled.

Non-goals (v1): speaker-change detection, auto-update, Windows/Linux, editing the
transcript inside the app, burning or ripping video.

## Stack and layout

Python 3.11, PyQt6, PyInstaller, bundled static ffmpeg + ffprobe, parakeet-mlx (arm64),
faster-whisper (fallback), python-docx, httpx for LLM calls (raw REST, no provider SDKs —
smaller bundle, fewer hidden imports).

```
disc_interviews/
  __init__.py
  app.py            # QApplication, MainWindow, wiring of widgets ↔ workers
  ui/
    main_window.py  # three-pane layout
    source_pane.py  # drive status + interview list
    detail_pane.py  # thumbnails, info, player, Text/Split tabs
    actions_pane.py # step buttons with progress
    settings_dialog.py
  disc/
    watcher.py      # polls /Volumes + drutil status → DiscState
    dvd.py          # VIDEO_TS parsing: titles, VOB sets, IFO chapters
    media.py        # ffprobe/ffmpeg helpers: probe, thumbnails, extract, sample
    model.py        # dataclasses: Disc, Interview, Segment, Chapter
  split/
    silence.py      # ffmpeg silencedetect → cut candidates (+ chapter boosting)
  asr/
    engine.py       # Transcriber: Parakeet (mlx) / Whisper; download with progress
    text.py         # sentences → paragraphs, txt/timed/srt writers
  review/
    llm.py          # provider-agnostic call(prompt) → str; Gemini/OpenAI/Anthropic REST
    finder.py       # chunk transcript, prompt, parse JSON, validate & merge spans
    docx_writer.py  # review.docx (highlights + comments) and review.html
  jobs/
    worker.py       # QThread wrapper with progress/finished/failed signals
    pipeline.py     # per-interview step state machine, "Run all"
  config.py         # ~/.disc-interviews/config.json, defaults, paths
  i18n.py           # EN/RU strings; UI language follows system, RU default when ru
  ffmpeg.py         # locate bundled or PATH ffmpeg/ffprobe
run.py              # entry point; sets bundled paths for PyInstaller
build.sh            # venv, deps, ffmpeg download, PyInstaller, ad-hoc sign, DMG
DiscInterviews.spec
requirements.txt
tests/              # pytest, pure logic only (no disc, no network)
docs/specs, docs/plans
```

Data locations: config `~/.disc-interviews/config.json`; ASR models
`~/.disc-interviews/models/<name>/`; results `~/Documents/Disc Interviews/<disc label>/<interview>/`
(configurable). Per interview folder: `thumbs/*.jpg`, `video.mp4` (or `audio.m4a` for audio-only sources), `audio16k.wav`,
`transcript.txt`, `transcript_timed.txt`, `transcript.srt`, `review.docx`, `review.html`,
`state.json` (step status + probe info so reopening the disc restores the list instantly).

## Disc detection (disc/watcher.py)

A QTimer (2 s) scans `/Volumes/*` and runs `drutil status` (with a 3 s timeout, in a
worker — drutil blocks while the drive spins). Result is a `DiscState`:

- `no_drive` — drutil finds no optical drive → status "No optical drive".
- `no_disc` — drive present, "No Media Inserted".
- `reading` — drutil reports media (or hangs) but no matching volume yet → "Drive is
  reading the disc; macOS has not mounted it yet".
- `mounted(volume)` — a volume that either has `VIDEO_TS/` (DVD-Video) or contains
  media files by extension (`.mp4 .m4v .mov .avi .mkv .mpg .mpeg .vob .mp3 .m4a .wav .aac .flac`).

`Open folder or ISO…` bypasses the watcher: a folder with `VIDEO_TS` or media files is
treated like a mounted volume; an `.iso` is attached read-only with `hdiutil attach
-readonly -nobrowse` and detached on close. A newly mounted volume triggers scanning
(`dvd.scan` / `media.scan`) in a worker; the source pane fills as items are probed.

## DVD structure (disc/dvd.py)

Titles: every `VTS_nn_0.IFO` with at least one `VTS_nn_k.VOB` (k ≥ 1). `VIDEO_TS.VOB`
(menu) is ignored. The VOB set is `concat:VTS_nn_1.VOB|…|VTS_nn_k.VOB`.

Chapters: parsed from `VTS_nn_0.IFO`: sector pointer to VTS_PGCI at byte 0xCC; program
map + cell playback table in the first PGC; cell playback time is BCD hh:mm:ss:ff with
the frame-rate bits in the frame byte. Program start = cumulative cell times. Failure to
parse chapters is non-fatal (log + no markers).

Per title `probe()` (media.py, ffprobe on the concat source, `-analyzeduration 20M`):
duration, video codec/size/fps, audio streams (codec, channels, language tag).
`thumbnails(n=8)`: one `ffmpeg -ss T -i … -frames:v 1` per position T at
(i+0.5)/n · duration, 320 px wide JPEGs, cached in `thumbs/`. The first thumbnail is the
list icon.

## Audio (disc/media.py)

- `extract(title, out_dir, progress)`: one ffmpeg pass on the concat source, first audio
  stream → `audio.m4a` (AAC 160 kbps, source sample rate, stereo kept). Then
  `audio16k.wav` (mono 16 kHz PCM) derived from the m4a — fast, no disc access. Progress
  from ffmpeg `-progress pipe:1` (`out_time_ms` / duration).
- `sample(title, seconds=30)`: extract the first 30 s to a temp m4a so the user can listen
  before the full extraction finishes.
- `save_as(interview, path, fmt)`: transcode the cached m4a to m4a (copy), mp3
  (libmp3lame 192k) or wav (PCM 16-bit).
- Playback: `QMediaPlayer` + `QAudioOutput` on the sample or full m4a; seek bar with
  chapter and cut markers; clicking a sentence in the Text tab seeks.

## Splitting (split/silence.py)

Level 1 always: one interview per DVD title / media file.

Level 2 opt-in (Settings → "Detect splits by long pauses", default off; pause threshold
default 8 s, min 3 s): after extraction run `ffmpeg -i audio16k.wav -af silencedetect=
noise=-35dB:d=<threshold> -f null -` and collect `silence_start/silence_end`. Each
silence midpoint is a candidate; a candidate within 3 s of a chapter start is marked
"matches chapter". Candidates appear on the Split tab: thumbnail at the cut, time,
"play around cut" (±5 s), remove; "add cut at player position". Confirm → the interview
is split into `Segment(start, end)` parts named `<title> — part N`, each with its own
folder. Audio for a part is cut from the cached m4a (`-ss/-to -c copy`); the transcript
and review doc are produced per part. Splitting can be undone (parts folder removed,
title state reset).

## Transcription (asr/)

`engine.py` ports the Transcriber pattern: model registry
`{"parakeet-v3": repo "mlx-community/parakeet-tdt-0.6b-v3", engine "parakeet"}` and
`{"whisper-medium": repo "Systran/faster-whisper-medium", engine "whisper"}`;
`download(model, progress)` via `huggingface_hub.snapshot_download` into
`~/.disc-interviews/models/<name>`; `is_downloaded` by marker file (`model.safetensors`
/ `model.bin`). Parakeet is offered only on arm64; on load failure fall back to Whisper
and tell the user. `transcribe(wav) -> list[Sentence(start, end, text)]` — Parakeet with
`chunk_duration=120, overlap_duration=15`; Whisper with `word_timestamps=False`, one
Sentence per segment. Module-level model cache. `text.py`: paragraphs break on gap > 2 s
or paragraph > 60 s; writers for txt / timed txt / srt.

## Review document (review/)

`llm.py`: `complete(system, user) -> str` for providers `gemini`, `openai`, `anthropic`
through their REST APIs with httpx (timeout 120 s, 2 retries on 5xx/429). Model names and
keys come from config; defaults `gemini-2.5-flash`, `gpt-4o-mini`, `claude-sonnet-4-5`
(editable).

`finder.py`: split paragraphs into ≤ 5 chunks of roughly equal size; run the chunks in
parallel threads (≤ 5); prompt (editable in Settings, RU default) asks for a JSON array of
`{"phrase": exact substring 1–8 words, "fix": likely wording or "?"}` for probable ASR
errors (names, terms, nonsense, broken grammar), explicitly not disfluencies. Parsing is
lenient: strip code fences, find the first `[`…`]`, ignore malformed items. Validation:
drop phrases not found verbatim in their chunk; find every occurrence per paragraph;
merge overlapping spans (concatenate distinct fixes). Result: `list[Span(paragraph_index,
start, end, fix)]`.

`docx_writer.py`: python-docx document — title, meta line (disc label, disc date from the
ISO volume descriptor when available, duration, engine), legend, then one paragraph per
transcript paragraph: grey 9 pt `[mm:ss]` run, plain runs, highlighted runs
(`WD_COLOR_INDEX.YELLOW`) each wrapped in a Word comment ("Вероятно: …" or "Проверить по
аудио") via python-docx comments API (≥ 1.2). `review.html`: same paragraphs, yellow
`<span>` + grey inline `⟨→ fix⟩` (Google Docs import keeps both). Both files are written
next to the transcript; the Actions pane offers "Open in Word / Reveal in Finder".

## Jobs and state (jobs/)

`Worker(QThread)` runs a callable with a `progress(float, str)` callback and emits
`progress`, `finished(result)`, `failed(exc_text)`. `pipeline.py` keeps per-interview step
state (`idle | running | done | failed`) for extract / transcribe / review, persists it
in `state.json`, enables buttons accordingly, and implements "Run all" as a queue that
extracts, transcribes and reviews every interview sequentially (one ffmpeg + one model
at a time; the optical drive is the bottleneck anyway). Any failure shows an inline
message with the last log lines; the app never blocks the UI on disc I/O.

## Dependencies and first run

Bundled: ffmpeg/ffprobe static builds (osxexperts.net, arch-matched), PyQt6, parakeet-mlx
+ mlx (arm64 only), faster-whisper + ctranslate2, python-docx, httpx, huggingface_hub.
Downloaded on demand: ASR model (Parakeet ≈ 2.4 GB; Whisper medium ≈ 1.5 GB) with a
progress bar on first Transcribe. Cloud keys: first "Create review document" without a
key opens Settings on the API tab. A status line at the bottom lists what is missing.

## Settings

General: output folder, UI language (auto/EN/RU). Recognition: engine (Parakeet v3 /
Whisper medium), "Detect splits by long pauses" + threshold seconds. Review: provider,
model name, API key (per provider), editable instruction text, "Reset to default".

## Build and distribution

`build.sh`: create `.venv` (python3.11), `pip install -r requirements.txt`, download
ffmpeg + ffprobe for the current arch into `bundled_ffmpeg/`, `pyinstaller
DiscInterviews.spec`, `codesign --force --deep -s -` (ad-hoc), `hdiutil create` DMG in
`dist/`. `run.py` prepends the bundled ffmpeg dir to PATH when frozen. App name
"Disc Interviews", bundle id `com.fvoin.discinterviews`.

## Testing

pytest, no disc and no network:

- `tests/test_ifo.py`: chapter parsing on a fixture copied from the Chernoplekov disc
  (`VTS_01_0.IFO`, 38 KB) — title count, chapter count, monotonic times.
- `tests/test_text.py`: paragraphing rule; srt/timed formatting.
- `tests/test_finder.py`: lenient JSON parsing; phrase validation; overlap merging.
- `tests/test_docx_writer.py`: build a doc from a small transcript + spans; unzip and
  count `w:highlight` and comments.
- `tests/test_silence.py`: parse ffmpeg silencedetect output into candidates; chapter
  boosting.
- `tests/test_watcher.py`: `DiscState` classification from fake `/Volumes` listings and
  drutil output.

Manual: real disc run end-to-end on this Mac (Chernoplekov DVD) before the first DMG.

## Editable transcript (transcript.py) — added 2026-09-07

Once recognised, an interview's text lives in `transcript.json` (sentences with word
timings from the recogniser, AI `marks` per sentence, `edited`/`original` per sentence).
Everything else — txt / timed / srt / review.docx / review.html — is regenerated from it.

- **Text tab** renders words as links: click a word → play from it (Parakeet token pieces
  merged into words by their leading space; Whisper via `word_timestamps=True`). Yellow
  spots are the AI marks; clicking one (or the ✎ after a sentence) opens the inline
  sentence editor with the model's hints, "Play sentence", "Remove mark", Save.
- **Saving an edit** rewrites the sentence text in `transcript.json`, drops its word
  timings (they no longer align) and its marks, keeps the original text, and refreshes
  the plain-text exports. The Word document is flagged stale until rebuilt.
- **Buttons**: "Find doubtful spots (AI)" runs the LLM and attaches marks (no document);
  "Create Word document" writes review.docx/html from the current text and marks — so a
  corrected transcript produces a clean document. "Run all" does extract → transcribe →
  AI check → Word.
- Older folders with only `transcript.srt` are upgraded on first open (no word timings).
