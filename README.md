# Disc Interviews

macOS app: insert a DVD with recorded interviews → see the titles with thumbnails →
listen / save the video as MP4 (one pass over the disc) → transcribe locally (NVIDIA Parakeet TDT v3 via parakeet-mlx,
Whisper fallback) → read and correct the text in place, like in Word (it saves itself) → write it out as a Word document.

```bash
/Library/Frameworks/Python.framework/Versions/3.11/bin/python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/python run.py          # run from source
.venv/bin/pytest                 # unit tests (no disc, no network)
./build.sh                       # .app + DiscInterviews-AppleSilicon.dmg in dist/
./release.sh                     # code snapshot → private fvoin/disc-interviews-code; DMGs → public fvoin/disc-interviews Releases (in-app updater)
```

No Mac at hand: every push to `main` runs the `release` workflow, which builds both DMGs on GitHub's macOS runners and publishes the same release whenever `APP_VERSION` has none yet (so: bump the version, push, done); it needs the `RELEASE_TOKEN` secret (PAT with write access to `fvoin/disc-interviews`).

Design: `docs/specs/2026-09-07-disc-interviews-design.md`. Plan: `docs/plans/`.

Extraction: MP4-family sources already in H.264 / AAC are copied, not re-encoded; a DVD that yields less than its IFO playback time is re-read sector by sector with unreadable sectors zero-filled (`disc/salvage.py`). Text: one editable document, a paragraph per block, the paragraph's time in the left margin (click it to play from there); edits are saved to `transcript.json` a moment after typing, and the Word document is built from that text. A tick left of the time marks a paragraph as checked (faint tint; ⌘↩ = checked and on to the next; «Checked N of M» jumps to the first unchecked), so a review can be picked up where it stopped. «Translate…» sends the paragraphs to the cloud model from Settings (Gemini / OpenAI / Anthropic) in batches with context and stores the result as a second track (`transcript.<lang>.json`), switchable above the text, editable and checkable like the original, with its own Word document (`review.<lang>.docx`). Voice-over: «Voice over the video…» synthesizes the chosen track paragraph by paragraph (`tts/engines.py`: Qwen3-TTS or Chatterbox through mlx-audio with the interviewee's own voice cloned from the recording, or the macOS voices), lays the clips on the timeline (`dub/assemble.py`: a clip that overruns its paragraph is sped up to 1.25× with the pitch kept, then shifts the rest until the next pause), ducks the original under the voice with a sidechain compressor, adds subtitles from any track as film-style cues that follow the speech (`dub/subtitles.py`; an mp4 subtitle stream, or burned in near-losslessly) and writes `video.dub.<lang>.mp4`; the player can switch to it. The cloning sample is the loudest clean paragraph, or 10 s from the player position, cleaned and loudness-normalized; every synthesized clip is levelled. Settings → Models lists and deletes downloaded models. Extraction encodes at DVD-like rates (6 Mbit/s VideoToolbox, CRF 17 x264) so nothing visible is lost. The AI review of doubtful spots is gone from the app.

Player: the 1× / 1.5× / 2× box left of Play sets the speed of every playback (player, clicked word, «Play sentence»); a copy of the sound stretched with ffmpeg's `atempo` (pitch kept) is rendered once per interview and rate into `~/.disc-interviews/cache/tempo/`. Pane boundaries carry a dotted grip; drag them (side panes and the thumbnails/info block collapse, the player stays), double-click a grip to collapse or restore, and the layout is saved to the config. The window itself shrinks freely: the action column and the thumbnails/info block scroll when short, side-pane labels shrink and elide, and a pane dragged shut no longer counts towards the minimum width.

Sources: a DVD (VIDEO_TS), a disc or folder with media files, a single file ("Open file…"), an ISO. Thumbnails, probe results and the 30-second sample of every disc go to `~/.disc-interviews/cache/`; the project folder under the output dir is created only by Extract.

Without the disc: "Open folder…" on a saved project (the `<output>/<disc>` folder, or one interview folder inside it) resumes it — cached media, transcript, marks and edits — instead of scanning it as a folder of media files (`disc_interviews/disc/project.py`).

macOS 11 (Big Sur) edition: `./build_legacy.sh arm64|x86_64` → `dist-legacy-<arch>/DiscInterviews-<AppleSilicon|Intel>-macOS11.dmg` (Qt 6.7, numpy 2.2, Whisper only, Intel static ffmpeg; playback uses Qt's native AVFoundation backend because the Qt 6.7 wheel ships its ffmpeg media plugin without the libav dylibs — see `disc_interviews/media_backend.py`).
