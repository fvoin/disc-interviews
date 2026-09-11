# Disc Interviews

macOS app: insert a DVD with recorded interviews → see the titles with thumbnails →
listen / save the video as MP4 (one pass over the disc) → transcribe locally (NVIDIA Parakeet TDT v3 via parakeet-mlx,
Whisper fallback) → produce a Word/HTML review copy with likely recognition errors
highlighted in yellow (found by a cloud LLM: Gemini / OpenAI / Anthropic).

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

Extraction: MP4-family sources already in H.264 / AAC are copied, not re-encoded; a DVD that yields less than its IFO playback time is re-read sector by sector with unreadable sectors zero-filled (`disc/salvage.py`). Review: ⌘] / ⌘[ walk the yellow spots, ⌘↩ puts the likely fix into the sentence editor (the yellow mark stays until Save), ⌘⌫ removes the mark; the full list is under Settings → Shortcuts. «Fair copy (Word)…» writes the text without highlights.

Player: the 1× / 1.5× / 2× box left of Play sets the speed of every playback (player, clicked word, «Play sentence»); a copy of the sound stretched with ffmpeg's `atempo` (pitch kept) is rendered once per interview and rate into `~/.disc-interviews/cache/tempo/`. Pane boundaries carry a dotted grip; drag them (side panes and the thumbnails/info block collapse, the player stays), double-click a grip to collapse or restore, and the layout is saved to the config. The window itself shrinks freely: the action column and the thumbnails/info block scroll when short, side-pane labels shrink and elide, and a pane dragged shut no longer counts towards the minimum width.

Sources: a DVD (VIDEO_TS), a disc or folder with media files, a single file ("Open file…"), an ISO. Thumbnails, probe results and the 30-second sample of every disc go to `~/.disc-interviews/cache/`; the project folder under the output dir is created only by Extract.

Without the disc: "Open folder…" on a saved project (the `<output>/<disc>` folder, or one interview folder inside it) resumes it — cached media, transcript, marks and edits — instead of scanning it as a folder of media files (`disc_interviews/disc/project.py`).

macOS 11 (Big Sur) edition: `./build_legacy.sh arm64|x86_64` → `dist-legacy-<arch>/DiscInterviews-<AppleSilicon|Intel>-macOS11.dmg` (Qt 6.7, numpy 2.2, Whisper only, Intel static ffmpeg; playback uses Qt's native AVFoundation backend because the Qt 6.7 wheel ships its ffmpeg media plugin without the libav dylibs — see `disc_interviews/media_backend.py`).
