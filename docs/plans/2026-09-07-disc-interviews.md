# Disc Interviews Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A standalone macOS app that detects an inserted interview DVD, shows its titles with thumbnails, extracts/saves/plays the audio, transcribes it locally with Parakeet, and produces a yellow-highlighted Word/HTML review copy via a cloud LLM.

**Architecture:** PyQt6 desktop app; all disc/ffmpeg/model work runs in `QThread` workers; pure-logic modules (IFO parsing, silence parsing, paragraphing, LLM answer validation, docx writing) have no Qt dependency and are unit-tested with pytest. Spec: `docs/specs/2026-09-07-disc-interviews-design.md`.

**Tech Stack:** Python 3.11 (`/Library/Frameworks/Python.framework/Versions/3.11/bin/python3`), PyQt6 (+QtMultimedia), parakeet-mlx (arm64), faster-whisper, python-docx ≥ 1.2, httpx, huggingface_hub, PyInstaller, bundled static ffmpeg/ffprobe.

## Global Constraints

- Project root `~/Dev/disc_interviews`, own git repo, no relation to ceoapp.
- Package name `disc_interviews`; app name "Disc Interviews"; bundle id `com.fvoin.discinterviews`.
- Config `~/.disc-interviews/config.json`; models `~/.disc-interviews/models/<name>/`; results default `~/Documents/Disc Interviews/<disc label>/<interview>/`.
- Pure-logic modules must not import PyQt6. UI strings go through `i18n.t(key)` with EN + RU.
- Every subprocess call to ffmpeg/ffprobe/drutil goes through `disc_interviews/ffmpeg.py` / has a timeout; nothing blocking runs on the Qt main thread.
- ASR chunking: `chunk_duration=120.0, overlap_duration=15.0`. Paragraph rule: new paragraph when gap > 2.0 s or paragraph length > 60 s.
- Silence detection defaults: `noise=-35dB`, threshold 8 s (min 3 s), chapter match window 3 s.
- Tests: `pytest` from project root with the venv python; no network, no disc, no Qt.
- Commit after every task with a short imperative message.

---

### Task 1: Project skeleton, config, i18n, ffmpeg locator

**Files:**
- Create: `requirements.txt`, `run.py`, `disc_interviews/__init__.py`, `disc_interviews/config.py`, `disc_interviews/i18n.py`, `disc_interviews/ffmpeg.py`, `tests/test_config.py`, `tests/test_i18n.py`, `.gitignore`, `README.md`
- Create venv: `.venv` via python3.11; `pip install -r requirements.txt`

**Interfaces (Produces):**
```python
# config.py
CONFIG_DIR = Path.home() / ".disc-interviews"
DEFAULTS = {
  "output_dir": str(Path.home() / "Documents" / "Disc Interviews"),
  "language": "auto",              # auto | en | ru
  "asr_model": "parakeet-v3",      # parakeet-v3 | whisper-medium
  "split_by_pauses": False, "pause_threshold_s": 8.0,
  "llm_provider": "gemini",        # gemini | openai | anthropic
  "llm_models": {"gemini": "gemini-2.5-flash", "openai": "gpt-4o-mini", "anthropic": "claude-sonnet-4-5"},
  "api_keys": {"gemini": "", "openai": "", "anthropic": ""},
  "review_prompt": DEFAULT_REVIEW_PROMPT,   # RU text, see Task 8
}
def load() -> dict            # DEFAULTS deep-merged with file; missing file → DEFAULTS
def save(cfg: dict) -> None   # atomic write (tmp + rename), mode 0600
def models_dir() -> Path      # CONFIG_DIR/"models", created
def output_dir(cfg) -> Path
# i18n.py
def set_language(code: str) -> None   # "auto" resolves via locale.getlocale / QLocale later
def t(key: str, **kw) -> str          # falls back to EN, then key
STRINGS: dict[str, dict[str, str]]    # key -> {"en":..., "ru":...}
# ffmpeg.py
def ffmpeg_path() -> str    # bundled (sys._MEIPASS/ffmpeg/ffmpeg) → /opt/homebrew/bin → PATH → raise FileNotFoundError
def ffprobe_path() -> str
def run(cmd: list[str], timeout: float | None = None, **popen) -> subprocess.CompletedProcess  # capture_output, text
```

- [ ] Step 1: write `tests/test_config.py` (monkeypatch `CONFIG_DIR` to tmp_path): `load()` without file returns DEFAULTS copy; `save` then `load` round-trips; unknown keys in file are kept; nested `api_keys` merge keeps defaults for missing providers.
- [ ] Step 2: write `tests/test_i18n.py`: `t("app.title")` returns EN by default, RU after `set_language("ru")`, unknown key returns the key, `t("progress.pct", pct=42)` formats.
- [ ] Step 3: run tests → fail (module missing).
- [ ] Step 4: implement the three modules; `requirements.txt`:
```
PyQt6>=6.7
parakeet-mlx; sys_platform == "darwin" and platform_machine == "arm64"
faster-whisper>=1.1
huggingface_hub
python-docx>=1.2
httpx
numpy
pyinstaller
pytest
```
`.gitignore`: `.venv/ build/ dist/ bundled_ffmpeg/ __pycache__/ *.pyc .pytest_cache/`.
- [ ] Step 5: run tests → pass. Commit `feat: project skeleton, config, i18n, ffmpeg locator`.

---

### Task 2: Data model and DVD structure parsing (IFO)

**Files:**
- Create: `disc_interviews/disc/__init__.py`, `disc_interviews/disc/model.py`, `disc_interviews/disc/dvd.py`, `tests/test_dvd.py`, `tests/fixtures/VTS_01_0.IFO` (copy from the Chernoplekov disc when mounted; if not available build a synthetic IFO in the test)

**Interfaces (Produces):**
```python
# model.py
@dataclass class Chapter: index: int; start: float                      # seconds
@dataclass class Title:
    key: str            # "VTS_01" or file stem
    label: str          # "Title 1" / file name
    sources: list[Path] # VOBs in order, or [media file]
    chapters: list[Chapter]
    duration: float = 0.0; width: int = 0; height: int = 0; fps: float = 0.0
    audio: list[str] = field(default_factory=list)   # "ac3 2ch ru"
    def concat_url(self) -> str   # "concat:a|b|c" or str(path) for single file
@dataclass class Disc: label: str; root: Path; kind: str  # "dvd" | "files"; titles: list[Title]; recorded: str | None = None
@dataclass class Segment: start: float; end: float | None; index: int   # part of a title
# dvd.py
def dvd_time_to_seconds(b: bytes) -> float       # 4-byte BCD hh mm ss ff(+rate bits); 25 or 29.97 fps
def parse_chapters(ifo: bytes) -> list[Chapter]  # from first entry PGC in VTS_PGCI; [] on any error
def scan_video_ts(root: Path) -> list[Title]     # root contains VIDEO_TS; one Title per VTS_nn with ≥1 VOB
```
IFO layout used by `parse_chapters`: sector pointer to VTS_PGCI at byte `0xCC` (big-endian u32, ×2048). VTS_PGCI: u16 count at +0; PGC search pointers start at +8, each 8 bytes: `entry_id` (bit7 = entry PGC), u8, u16 parental, u32 pgc offset (relative to VTS_PGCI start). PGC: `nr_of_programs` u8 at +2, `nr_of_cells` u8 at +3, `program_map_offset` u16 at +0xE6, `cell_playback_offset` u16 at +0xE8 (relative to PGC start). Program map: `nr_of_programs` bytes, 1-based entry cell numbers. Cell playback entries: 24 bytes each, `dvd_time` at +4. Chapter i start = sum of cell durations for cells before the program's entry cell.

- [ ] Step 1: tests: `dvd_time_to_seconds(bytes.fromhex("00193612"))` (00:19:36 + 12 frames @25 fps, rate bits 01 → frame byte 0x52) ≈ 1176.48; fixture: `parse_chapters(fixture)` returns ≥ 1 chapter, first start 0.0, starts strictly increasing, last < 70*60; `scan_video_ts(tmp VIDEO_TS with VTS_01_0.IFO + VTS_01_1.VOB + VTS_01_2.VOB, VTS_02_0.IFO w/o VOB)` → one Title with 2 sources in order and key "VTS_01".
- [ ] Step 2: run → fail. Step 3: implement. Step 4: pass. Step 5: commit `feat: DVD title and chapter parsing`.

---

### Task 3: Media helpers (ffprobe, thumbnails, extraction, sample, save-as) and silence split logic

**Files:**
- Create: `disc_interviews/disc/media.py`, `disc_interviews/split/__init__.py`, `disc_interviews/split/silence.py`, `tests/test_media_parse.py`, `tests/test_silence.py`

**Interfaces (Produces):**
```python
# media.py  (subprocess via ffmpeg.run; pure parsers are separate functions for tests)
def parse_ffprobe(json_text: str) -> dict          # {"duration": float, "width", "height", "fps", "audio": [str]}
def probe(title: Title) -> None                    # fills title fields; ffprobe -v quiet -print_format json -show_format -show_streams -analyzeduration 20M -probesize 20M
def thumbnails(title: Title, out_dir: Path, n: int = 8, width: int = 320) -> list[Path]   # skips existing
def extract_audio(title: Title, out_dir: Path, progress: Callable[[float, str], None], segment: Segment | None = None) -> tuple[Path, Path]
    # → (audio.m4a, audio16k.wav). ffmpeg -i <concat> [-ss/-to] -map 0:a:0 -vn -c:a aac -b:a 160k audio.m4a, -progress pipe:1 parsed by parse_progress
    # then ffmpeg -i audio.m4a -ac 1 -ar 16000 -c:a pcm_s16le audio16k.wav
def parse_progress(line: str, duration: float) -> float | None   # "out_time_ms=123456" → fraction
def sample_audio(title: Title, out_path: Path, seconds: int = 30) -> Path
def save_as(m4a: Path, dest: Path, fmt: str) -> None   # "m4a" copy, "mp3" libmp3lame 192k, "wav" pcm_s16le
def cut_audio(m4a: Path, dest: Path, start: float, end: float | None) -> Path   # -ss/-to -c copy
# silence.py
@dataclass class Cut: time: float; silence_len: float; matches_chapter: bool
def parse_silencedetect(stderr: str) -> list[tuple[float, float]]   # (start, end) pairs
def candidates(silences, chapters: list[Chapter], min_len: float, window: float = 3.0) -> list[Cut]  # midpoint; filter by min_len
def detect(wav: Path, threshold_s: float) -> list[tuple[float, float]]  # runs ffmpeg -af silencedetect=noise=-35dB:d=<threshold_s> -f null -
def split_segments(duration: float, cuts: list[Cut]) -> list[Segment]  # sorted, dedup within 1 s, [0..c1],[c1..c2],...,[cn..None]
```
- [ ] Step 1: tests with captured strings: `parse_ffprobe` on a sample ffprobe JSON (mpeg2video 720x576 25/1, ac3 2 ch) → duration/width/height/fps/audio `["ac3 2ch"]`; `parse_progress("out_time_ms=600000000", 1200.0) == 0.5`; `parse_silencedetect` on two `[silencedetect @ …] silence_start: 12.3` / `silence_end: 21.9 | silence_duration: 9.6` lines; `candidates` boosts when chapter at 17.5 (window 3); `split_segments(100, [Cut(40,..),Cut(40.5,..)])` → 2 segments.
- [ ] Step 2–5: fail → implement → pass → commit `feat: media helpers and pause-based split logic`.

---

### Task 4: Disc watcher (state classification) and files-disc scan

**Files:**
- Create: `disc_interviews/disc/watcher.py`, `tests/test_watcher.py`

**Interfaces (Produces):**
```python
MEDIA_EXT = {".mp4",".m4v",".mov",".avi",".mkv",".mpg",".mpeg",".vob",".mp3",".m4a",".wav",".aac",".flac"}
@dataclass class DiscState: kind: str  # no_drive | no_disc | reading | mounted ; volume: Path | None = None; label: str = ""
def classify(volumes: list[Path], drutil_out: str | None, has_video_ts: Callable[[Path], bool], has_media: Callable[[Path], bool]) -> DiscState
    # drutil_out None → timed out (treat as "reading" if no volume); "No Media Inserted" → no_disc; no "Vendor" header → no_drive
def scan_files(root: Path) -> list[Title]   # media files sorted by name → Title(key=stem, sources=[file])
def scan_volume(vol: Path) -> Disc          # dvd if VIDEO_TS exists else files; label = vol.name
def read_disc_date(volume: Path) -> str | None   # diskutil info -plist → DeviceNode → read sector 16 PVD creation date (bytes 813..829) → "YYYY-MM-DD"; None on any failure
class DiscWatcher(QObject)  # in ui layer later; here only pure functions + poll(): DiscState using the real fs/drutil with 3 s timeout
```
- [ ] Tests: classify: no volumes + "No Media Inserted" → no_disc; no volumes + drutil None → reading; volume with VIDEO_TS → mounted with label; volume without media ignored → no_disc; drutil "" → no_drive. `scan_files` orders and filters by extension.
- [ ] Commit `feat: disc state classification and file-disc scan`.

---

### Task 5: ASR engine and text writers

**Files:**
- Create: `disc_interviews/asr/__init__.py`, `disc_interviews/asr/engine.py`, `disc_interviews/asr/text.py`, `tests/test_text.py`

**Interfaces (Produces):**
```python
# text.py
@dataclass class Sentence: start: float; end: float; text: str
def paragraphs(sents: list[Sentence], gap: float = 2.0, max_len: float = 60.0) -> list[tuple[float, str]]  # (start, text)
def fmt_ts(sec: float) -> str            # "mm:ss" or "h:mm:ss"
def write_outputs(sents, out_dir: Path) -> dict[str, Path]  # transcript.txt (blank-line paragraphs), transcript_timed.txt ("[mm:ss] text"), transcript.srt
def read_sentences(srt: Path) -> list[Sentence]              # to reload without re-running ASR
# engine.py
MODELS = {"parakeet-v3": {"repo": "mlx-community/parakeet-tdt-0.6b-v3", "engine": "parakeet", "marker": "model.safetensors", "size_mb": 2400},
          "whisper-medium": {"repo": "Systran/faster-whisper-medium", "engine": "whisper", "marker": "model.bin", "size_mb": 1500}}
def available_models() -> list[str]        # parakeet only on arm64 darwin
def is_downloaded(name) -> bool
def download(name, progress: Callable[[float, str], None]) -> Path   # snapshot_download(repo, local_dir=models_dir()/name); progress via tqdm_class shim
def transcribe(wav: Path, model: str, progress) -> list[Sentence]     # loads once (module cache); parakeet → result.sentences; whisper → segments
```
- [ ] Tests (`test_text.py`): paragraph splits on gap > 2 and on > 60 s; `fmt_ts(3661) == "1:01:01"`; `write_outputs` then `read_sentences` round-trips text/start/end (ms precision).
- [ ] Engine is exercised manually (needs model); keep `transcribe` thin and guard imports inside functions.
- [ ] Commit `feat: ASR engine wrapper and transcript writers`.

---

### Task 6: LLM client

**Files:**
- Create: `disc_interviews/review/__init__.py`, `disc_interviews/review/llm.py`, `tests/test_llm.py`

**Interfaces (Produces):**
```python
class LLMError(RuntimeError): ...
def complete(provider: str, model: str, api_key: str, system: str, user: str, timeout: float = 120.0) -> str
# gemini: POST https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent?key=… {"system_instruction":{"parts":[{"text":system}]},"contents":[{"parts":[{"text":user}]}]} → candidates[0].content.parts[*].text joined
# openai: POST https://api.openai.com/v1/chat/completions, Bearer; messages system+user → choices[0].message.content
# anthropic: POST https://api.anthropic.com/v1/messages, x-api-key + anthropic-version 2023-06-01, max_tokens 8192 → content[*].text joined
# retries: 2 on 429/5xx with 2 s, 4 s sleep; missing key → LLMError("api_key_missing")
def build_request(provider, model, api_key, system, user) -> tuple[str, dict, dict]   # (url, headers, json) — pure, for tests
def parse_response(provider, data: dict) -> str                                      # pure, for tests
```
- [ ] Tests: `build_request` per provider (url, auth header/query, body shape); `parse_response` per provider on minimal dicts; missing key raises.
- [ ] Commit `feat: cloud LLM client`.

---

### Task 7: Review finder (prompting, parsing, validation, merging)

**Files:**
- Create: `disc_interviews/review/finder.py`, `tests/test_finder.py`

**Interfaces (Produces):**
```python
@dataclass class Span: para: int; start: int; end: int; fix: str
DEFAULT_REVIEW_PROMPT: str   # RU: корректор ASR-расшифровки интервью; искать вероятные ошибки распознавания (имена, институты, термины, бессмыслица, сломанная грамматика от ослышки); НЕ отмечать разговорные повторы; вернуть ТОЛЬКО JSON-массив {"phrase": точная подстрока 1–8 слов, "fix": вероятное исправление или "?"}
def chunk(paras: list[str], n: int = 5) -> list[list[int]]      # paragraph index groups, balanced by chars
def parse_answer(text: str) -> list[dict]                        # strip ``` fences, first '[' .. last ']', json.loads, keep dicts with str phrase
def locate(paras: list[str], items: list[dict], para_indices: list[int]) -> list[Span]   # every verbatim occurrence within those paragraphs
def merge(spans: list[Span]) -> list[Span]                       # per paragraph, overlapping → one span; fixes joined " / " (skip "?", dedupe)
def find_spans(paras: list[str], ask: Callable[[str], str], prompt: str, context: str = "", workers: int = 5) -> list[Span]
    # ask(user_text) -> answer; runs chunks via ThreadPoolExecutor; one failing chunk is logged and skipped
```
- [ ] Tests: `parse_answer` handles fences, prose before/after, trailing comma garbage (falls back to []); `locate` finds two occurrences of "Курчатский" in one paragraph and drops a missing phrase; `merge` merges [3,10) + [7,15) → [3,15) with fix "A / B"; `chunk` returns 5 balanced groups for 55 paragraphs and 1 group for 2 paragraphs; `find_spans` with a fake `ask` returning JSON works end-to-end and skips a chunk whose `ask` raises.
- [ ] Commit `feat: review span finder`.

---

### Task 8: DOCX and HTML review writers

**Files:**
- Create: `disc_interviews/review/docx_writer.py`, `tests/test_docx_writer.py`

**Interfaces (Produces):**
```python
@dataclass class ReviewMeta: title: str; disc_label: str; recorded: str | None; duration: float; engine: str
def write_docx(paras: list[tuple[float, str]], spans: list[Span], meta: ReviewMeta, out: Path) -> Path
    # python-docx: Title, grey 9pt meta line, legend paragraph; per paragraph: grey 9pt "[mm:ss]  " run, plain runs, highlighted runs
    # (font.highlight_color = WD_COLOR_INDEX.YELLOW) each with document.add_comment(runs=[run], text="Вероятно: <fix>" | "Проверить по аудио", author="Disc Interviews", initials="DI")
def write_html(paras, spans, meta, out: Path) -> Path
    # <span style="background-color:#ffff00">…</span><span style="color:#888;font-size:8pt"> ⟨→ fix⟩</span>; html.escape everything
```
- [ ] Tests: build from 2 paragraphs + 3 spans; unzip docx: `word/document.xml` contains `w:highlight w:val="yellow"` ×3 and `w:commentRangeStart` ×3, `word/comments.xml` has 3 comments; html has 3 yellow spans and escapes `<`.
- [ ] Commit `feat: review document writers`.

---

### Task 9: Jobs — worker thread and per-interview pipeline state

**Files:**
- Create: `disc_interviews/jobs/__init__.py`, `disc_interviews/jobs/worker.py`, `disc_interviews/jobs/pipeline.py`, `tests/test_pipeline.py`

**Interfaces (Produces):**
```python
# worker.py (PyQt6)
class Worker(QThread):
    progress = pyqtSignal(float, str); finished_ok = pyqtSignal(object); failed = pyqtSignal(str)
    def __init__(self, fn: Callable[[Callable[[float, str], None]], Any], parent=None)
# pipeline.py (pure)
STEPS = ("extract", "transcribe", "review")
@dataclass class InterviewState: title_key: str; segment: Segment | None; folder: Path; steps: dict[str, str]  # idle|running|done|failed
    def can(self, step) -> bool   # extract always; transcribe needs extract done; review needs transcribe done
def state_path(folder) -> Path; def load_state(folder) -> InterviewState | None; def save_state(st) -> None
def interview_folder(cfg, disc: Disc, title: Title, segment: Segment | None) -> Path   # <output>/<label>/<Title label>[ — part N]
def slug(s: str) -> str   # filesystem-safe, keeps Cyrillic, collapses spaces
class RunAllQueue: def __init__(self, items: list[InterviewState]); def next(self) -> tuple[InterviewState, str] | None  # next (item, step) not done, in order
```
- [ ] Tests: `can()` gating; state save/load round-trip; `slug("VTS_01 / Часть 2")`; `RunAllQueue` yields extract→transcribe→review per item then moves on, skipping done steps.
- [ ] Commit `feat: worker thread and pipeline state`.

---

### Task 10: UI — main window, panes, settings dialog, app wiring

**Files:**
- Create: `disc_interviews/ui/__init__.py`, `main_window.py`, `source_pane.py`, `detail_pane.py`, `actions_pane.py`, `settings_dialog.py`, `disc_interviews/app.py`; finalize `run.py`

**Behaviour to implement (no unit tests; manual check with the real disc):**
- `SourcePane`: status label (drive state text via i18n), `QListWidget` of interviews (icon = first thumbnail, text = label + duration + step badges ✓), "Open folder or ISO…" button (`QFileDialog`; `.iso` → `hdiutil attach -readonly -nobrowse`, remember to detach on exit).
- `DiscWatcher(QObject)`: `QTimer` 2 s → `Worker(poll)` → signal `state_changed(DiscState)`; on new `mounted` volume → `Worker(scan_volume)` → `disc_ready(Disc)`; then per title `Worker(probe + thumbnails)` → `title_ready(Title)`.
- `DetailPane`: thumbnail strip (8 `QLabel`s), info grid, player (`QMediaPlayer`, `QAudioOutput`, play/pause, `QSlider` seek with chapter ticks painted, time label), tabs: Text (`QTextBrowser`, `[mm:ss]` anchors → seek), Split (list of `Cut`s with remove/play-around, "Add cut at position", "Apply split", "Undo split").
- `ActionsPane`: 5 buttons with a `QProgressBar` + status label each; "Run all"; enabled by `InterviewState.can`; "Save audio…" → `QFileDialog.getSaveFileName` with m4a/mp3/wav filters → `media.save_as`.
- `SettingsDialog`: tabs General / Recognition / Review as in spec; save → `config.save`; API key fields `QLineEdit` with `EchoMode.Password`.
- Pipeline glue in `app.py`: extract → `media.extract_audio` (+ `silence.detect` if enabled → Split tab); transcribe → `engine.download` if needed then `engine.transcribe` → `text.write_outputs` → Text tab; review → `finder.find_spans(ask=lambda u: llm.complete(...))` → `write_docx`/`write_html` → "Open in Word / Reveal in Finder" buttons; errors → red status line with last stderr lines.
- Bottom status bar: ffmpeg found?, model downloaded?, API key set? (i18n).

- [ ] Run `python run.py` with a folder pointing at a copied `VIDEO_TS` or the disc; verify list, thumbnails, sample playback, extract, transcribe, review.
- [ ] Commit `feat: desktop UI and pipeline wiring`.

---

### Task 11: Build script, PyInstaller spec, DMG

**Files:**
- Create: `build.sh`, `DiscInterviews.spec`, `assets/icon.icns` (generate from a simple SVG/PNG with `iconutil`), `HOW TO INSTALL.txt`

- [ ] `build.sh`: `set -euo pipefail`; python3.11 venv; pip install; download ffmpeg + ffprobe zips from osxexperts.net for `uname -m` into `bundled_ffmpeg/` (skip if present, strip quarantine); `pyinstaller DiscInterviews.spec --noconfirm`; `codesign --force --deep -s - dist/Disc\ Interviews.app`; `hdiutil create -volname "Disc Interviews" -srcfolder dist/Disc\ Interviews.app -ov -format UDZO dist/DiscInterviews.dmg`.
- [ ] Spec: `collect_all` for `parakeet_mlx`, `mlx`, `faster_whisper`, `ctranslate2`, `docx`; datas `bundled_ffmpeg/*`; hiddenimports `PyQt6.QtMultimedia`; excludes `matplotlib, PIL, sympy`; `console=False`; `BUNDLE(name="Disc Interviews.app", bundle_identifier="com.fvoin.discinterviews", info_plist={"NSHighResolutionCapable": True, "LSMinimumSystemVersion": "13.0"})`.
- [ ] Build, launch `dist/Disc Interviews.app`, repeat the manual check. Commit `build: PyInstaller app and DMG`.

---

## Self-review

- Spec coverage: detection (T4, T10), DVD structure (T2), probe/thumbs/extract/sample/save (T3), splitting (T3 logic, T10 UI), ASR + downloads (T5), review LLM + finder + docx/html (T6–T8), jobs/state/Run all (T9, T10), settings (T10), build (T11), tests (T1–T9). Disc date (T4 `read_disc_date`) feeds `ReviewMeta.recorded`.
- Types: `Title`, `Chapter`, `Segment`, `Disc` from `disc/model.py`; `Sentence` from `asr/text.py`; `Span` from `review/finder.py`; `Cut` from `split/silence.py`; `InterviewState` from `jobs/pipeline.py` — used consistently above.
