# PumpTime

A local exercise tracker. You drop in a screenshot (or a phone photo) of a running
stopwatch, the app reads the set time out of the pixels, works out when the image was
taken, and compares that set against the same set number from your last workout.

---

## 1. Decisions this plan is built on

These four answers drive every structure choice below. Changing one invalidates parts
of the plan, so they are stated up front rather than buried.

| Question | Answer | What it forces |
|---|---|---|
| Where does the image come from? | **Both** camera photos and OS screenshots | Two separate date strategies: EXIF for photos, OCR/fallback for screenshots |
| How is the set time read? | **Local OCR + regex** | Python backend. No API keys, no per-image cost, nothing leaves the machine |
| What kind of app? | **Desktop app + local backend** | SQLite, local disk, no auth, no deploy step |
| What is compared day to day? | **Set-by-set** (today's set 2 vs last workout's set 2) | `set_index` becomes a required column |

---

## 2. Stack

| Layer | Choice | Reason |
|---|---|---|
| Frontend | React + TypeScript + Vite | Best ecosystem for file upload and charts |
| Styling | TailwindCSS | Fast, low-ceremony |
| Charts | Recharts | Daily bar charts and set-over-set deltas |
| Upload | React Dropzone | Drag-and-drop + paste |
| Server state | TanStack Query | Caching, refetch, optimistic updates after a confirmed set |
| Backend | Python 3.12 + FastAPI | Only mature home for Tesseract/PaddleOCR plus Pillow/OpenCV preprocessing |
| OCR | pytesseract (Tesseract) | Local, free, no network |
| Image | Pillow + OpenCV | EXIF read, decode, preprocess, thumbnail |
| DB | SQLite + SQLAlchemy 2.x + Alembic | Single user, single file, zero setup |
| Storage | Local filesystem behind an interface | Swappable to S3 later without touching callers |
| Shell (optional) | Tauri (Rust) | Only if you want a real OS window instead of a browser tab |

---

## 3. Upload → parse pipeline

```
image bytes
  │
  ├─ 1. HASH        sha256 → dedupe (re-uploading the same screenshot must not double-log)
  ├─ 2. STORE       write image + thumbnail under data/images/<yyyy-mm-dd>/
  │
  ├─ 3. DATE        precedence, first hit wins:
  │                    a. EXIF DateTimeOriginal + OffsetTimeOriginal   (camera photos)
  │                    b. OCR the status-bar clock                    (screenshots)
  │                    c. filesystem mtime                            (weak — flag it)
  │                    d. null → user enters it on the Review screen
  │
  └─ 4. SET TIME
          preprocess  grayscale → upscale 2–4× → binarize (Otsu) → invert if needed
          OCR         tesseract image_to_data (keeps per-word confidence)
          parse       patterns.py → candidate time strings
          validate    seconds part 00–59, minutes in a plausible range, decimal allowed

  → ExtractionResult { time_seconds, confidence, raw_text, source, taken_at, taken_at_source }

  5. REVIEW        user sees editable fields, each labelled with its source + confidence
  6. PERSIST       only now write set_log
```

**Two rules that are not negotiable:**

1. **Persist happens after confirmation, not after upload.** An upload that is never
   confirmed leaves an `image` row and no `set_log` row.
2. **Never trust OCR silently.** Every extracted value carries a `source` and a
   `confidence`. Low confidence or no match means the field renders blank and prompts,
   never guessed.

### Why preprocessing is the real work

Raw Tesseract on a stylised timer font is unreliable, and the failure is almost always
upstream of the OCR call — small glyph height, low contrast, anti-aliasing, an inverted
theme. Most of the tuning effort lives in `preprocess.py`. When extraction quality is
judged, judge it on the test fixtures, not on one screenshot that happened to work.

---

## 4. Folder structure

```
pumptime/
├── README.md
│
├── backend/
│   ├── pyproject.toml               # deps + ruff + pytest (replaces requirements.txt)
│   ├── alembic.ini
│   ├── app/
│   │   ├── main.py                  # FastAPI app; mounts routers, serves ../frontend/dist
│   │   ├── config.py                # data dir, db url, OCR thresholds
│   │   ├── db/
│   │   │   ├── session.py           # engine, SessionLocal, get_db dependency
│   │   │   ├── models.py            # SQLAlchemy tables
│   │   │   └── migrations/          # alembic versions
│   │   ├── api/                     # thin routers, no business logic
│   │   │   ├── images.py
│   │   │   ├── extractions.py
│   │   │   ├── sets.py
│   │   │   ├── sessions.py
│   │   │   ├── exercises.py
│   │   │   └── stats.py
│   │   ├── schemas/                 # pydantic request/response models
│   │   │   ├── image.py
│   │   │   ├── extraction.py
│   │   │   ├── set.py
│   │   │   └── session.py
│   │   ├── services/
│   │   │   ├── storage.py           # save/read/thumbnail — the only module that knows disk
│   │   │   ├── exif.py              # taken_at from metadata
│   │   │   ├── ocr/
│   │   │   │   ├── preprocess.py    # grayscale, upscale, binarize, invert
│   │   │   │   ├── engine.py        # tesseract call, per-word confidence
│   │   │   │   └── patterns.py      # time-string parsing  ← highest test value
│   │   │   ├── extraction.py        # orchestrates the pipeline, returns ExtractionResult
│   │   │   └── comparison.py        # set-by-set pairing
│   │   └── util/
│   │       └── timefmt.py           # parse/format the candidate strings
│   └── tests/
│       ├── fixtures/images/         # REAL screenshots + camera photos, known answers
│       ├── test_patterns.py
│       ├── test_exif.py
│       ├── test_extraction.py
│       └── test_comparison.py
│
├── frontend/
│   ├── package.json
│   ├── vite.config.ts               # dev proxy /api → 127.0.0.1:8000
│   └── src/
│       ├── main.tsx
│       ├── App.tsx
│       ├── api/client.ts            # typed fetch wrapper
│       ├── types.ts
│       ├── hooks/
│       │   ├── useUpload.ts
│       │   ├── useExtraction.ts
│       │   └── useSets.ts
│       ├── pages/
│       │   ├── Upload.tsx           # dropzone / paste
│       │   ├── Review.tsx           # confirm extracted values
│       │   ├── Dashboard.tsx        # daily chart, PRs
│       │   ├── History.tsx          # per-day list
│       │   └── Compare.tsx          # set-by-set deltas
│       └── components/
│           ├── ImageDropzone.tsx
│           ├── ExtractedFields.tsx  # editable, shows source + confidence
│           ├── SetComparisonTable.tsx
│           └── Sparkline.tsx
│
└── data/                            # runtime, gitignored
    ├── pumptime.db
    ├── images/<yyyy-mm-dd>/<sha256>.jpg
    └── thumbnails/<sha256>.webp
```

---

## 5. Database schema

```sql
exercise (
  id          INTEGER PK,
  name        TEXT NOT NULL,          -- "Bench Press"
  created_at  TIMESTAMPTZ
)

session (                                -- one workout
  id                    INTEGER PK,
  performed_on          DATE NOT NULL,   -- local calendar day
  started_at            TIMESTAMPTZ,     -- UTC
  local_offset_minutes  INTEGER,          -- kept separately; EXIF often has no zone
  note                  TEXT
)

image (                                  -- exists before, and independently of, any set
  id                INTEGER PK,
  sha256            TEXT UNIQUE NOT NULL,-- dedupe on re-upload
  path              TEXT NOT NULL,
  thumbnail_path    TEXT,
  width             INTEGER,
  height            INTEGER,
  byte_size         INTEGER,
  taken_at          TIMESTAMPTZ,        -- NULL if unknowable
  taken_at_source   TEXT NOT NULL,       -- exif | ocr | mtime | manual | unknown
  taken_at_raw      TEXT,                -- the literal EXIF string, for audit
  uploaded_at       TIMESTAMPTZ
)

set_log (                                -- the actual measurement
  id            INTEGER PK,
  session_id    INTEGER NOT NULL FK,
  exercise_id   INTEGER NOT NULL FK,
  image_id      INTEGER FK,             -- NULL when typed by hand
  set_index     INTEGER NOT NULL,       -- 1-based. THE unit of comparison.
  time_seconds  REAL NOT NULL,          -- REAL, not INT: stopwatches show 1:23.4
  confidence    REAL,                   -- NULL when entered manually
  raw_text      TEXT,                   -- what OCR actually read, for debugging
  source        TEXT NOT NULL,          -- ocr | manual
  verified      BOOLEAN NOT NULL DEFAULT 0,
  created_at    TIMESTAMPTZ,
  UNIQUE (session_id, exercise_id, set_index)
)
```

**Notes on the schema:**

- `set_index` is why the `UNIQUE` constraint exists — it prevents duplicate set numbers
  inside one workout.
- `image` is a separate table rather than columns on `set_log` because an image is
  parsed *before* any set exists, one screenshot can legitimately contain several sets,
  and `sha256` dedupe only works if the image has its own identity.
- `daily_summary` is deliberately **not** stored. Derive `total_sets`, `total_time`,
  `best_set`, `avg_set` per `(exercise, performed_on)` in the read layer. A stored
  summary is invalidated by every late-arriving or corrected set. Promote it to a
  materialized view only if these queries actually get slow.

---

## 6. The comparison

Input: `(exercise_id, on_date)`.

1. Resolve the current session for `on_date`.
2. Find the **most recent prior session that contains sets for this exercise.** Not
   "yesterday" — the previous workout may be four days ago.
3. Build `previous = {set_index: time_seconds}` and `current = {...}`.
4. Union the key sets and emit one row per `set_index`:

   ```
   set_index | current | previous | delta_seconds | previous_session_date
   ```

5. Unpaired indices render as `new` or `dropped`, never as a fabricated delta.

Reducing faster? A negative `delta_seconds` is the improvement. The UI must not assume
"bigger is better."

---

## 7. API

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/images` | multipart upload → `{ image_id, sha256, duplicate }` |
| GET | `/api/images/{id}` | fetch image + thumbnail |
| POST | `/api/extractions/{image_id}` | run the pipeline → `ExtractionResult` |
| PATCH | `/api/images/{id}/date` | user overrides `taken_at`, sets source `manual` |
| POST | `/api/sessions` | open a workout for a date |
| GET | `/api/sessions?from=&to=` | list workouts |
| POST | `/api/sets` | create a confirmed set |
| PATCH | `/api/sets/{id}` | correct a value, flips `verified = 1` |
| DELETE | `/api/sets/{id}` | remove a set |
| GET | `/api/exercises` | list |
| GET | `/api/exercises/{id}/comparison?on=` | set-by-set deltas vs previous session |
| GET | `/api/exercises/{id}/trend?days=90` | daily series for charting |
| GET | `/api/stats/prs?exercise=` | personal records |

---

## 8. Build order

Vertical slices, not layers. Each one leaves the app in a working state.

1. **Image ingest** — dropzone → upload → hash → write file → `image` row. No OCR.
2. **Manual logging + comparison** — session/set CRUD with a hand-entered time, plus the
   comparison query. This proves the data model is right before any OCR noise exists to
   distort it.
3. **EXIF date** — easy, verifiable, and settles the date-precedence chain for photos.
4. **OCR set time** — preprocess → engine → patterns, each with fixture tests.
5. **Comparison UI** — Recharts daily bars, set-over-set table.
6. **Streaks / PRs / badges.**

Steps 1–2 deliberately come before step 4. If OCR lands first, the schema gets bent to
fit bad reads and there is no clean baseline to test the comparison against.

---

## 9. Known risks

| Risk | Mitigation |
|---|---|
| OCR accuracy on stylised timer fonts | Mandatory Review screen; per-value `confidence` and `source`; tune `preprocess.py` against fixtures, not vibes |
| Screenshots carry no EXIF | Status-bar clock OCR as step (b), mtime as step (c), manual entry as step (d). The date field is always user-editable |
| Timezones | Store UTC + `local_offset_minutes`. `DateTimeOriginal` has no zone unless `OffsetTimeOriginal` is present — capture both raw |
| A screenshot showing several sets | `image` is 1-to-many with `set_log`. The Review screen must let you create several sets from one image |
| Naive time parsing | `patterns.py` validates the seconds field is 00–59 and applies a minutes plausibility bound, instead of regex-matching blindly |

---

## 10. Removed from the original plan

Kept deliberately, so the reasoning is not lost:

| Removed | Why |
|---|---|
| Auth, `users` table, JWT, `/auth/*`, Login page | The backend is local and single-user, and never listens on a network. Auth protects nothing here |
| PostgreSQL | The dataset is sets, not events. SQLite is one file with no setup, and `sqlalchemy` makes the later port a config change |
| Redis + Celery | Synchronous OCR is fast enough for one user, and images arrive at human speed. `extraction.py` is already a seam, so async slots in later without touching callers |
| MinIO / S3 object storage | Local disk behind `services/storage.py`. One module changes if that flips |
| `docker-compose.yml` | There are no services to compose locally |
| Deploy to Vercel / Fly.io / Supabase | Directly contradicts "desktop app + local backend". There is nothing to deploy |
| `exercise.unit` | `time_seconds` is always normalised to seconds. Unit is a display concern, not stored data |
| `daily_summary` table | Denormalised at write time and invalidated by every correction. Derive it in the read layer |
| "Compare to yesterday" widget | Wrong when days are skipped. Comparison anchors to the previous *session* for that exercise |
| `(\d{1,3}):(\d{2})` as the parser | It matches `1:23` out of `1:234`, misses `h:mm:ss` and decimal seconds, and cannot distinguish `mm:ss` from a date. Replaced by `patterns.py` with real validation |
| `requirements.txt` | `pyproject.toml` covers deps plus ruff/pytest config in one file |

---

## 11. Backlog

Re-gate each of these on the app actually outgrowing local-only. Most of them assume a
hosted backend and multi-user auth, which is exactly what was removed above.

- **Tauri shell** — wrap the built frontend in a native window
- **Auto-detect exercise** from OCR context, or per-app template matching
- **PWA** for phone capture (needs a hosted backend first)
- **Email / Telegram ingest** — forward a screenshot, auto-log it
- **Fine-tuned OCR model** — only if fixtures show Tesseract cannot reach the accuracy bar
- **Multi-user** — auth, Postgres, object storage, leaderboards
