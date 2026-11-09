# YT Lecture RAG

Timestamp-level retrieval over a YouTube DSA lecture playlist. Ask a question in
Hinglish or English and get a grounded answer with citations that jump to the
relevant second in the original lecture.

```text
"memoization aur tabulation ka difference kya hai?"

Answer: a short explanation grounded in the lecture transcripts
Sources: lecture title, timestamp, and a link that starts the video at that point
```

The project is a Python CLI and FastAPI application. It can run entirely locally:
the repository already includes 126 transcript files and a prebuilt vector index, so
you can use the demo without a GPU, YouTube downloads, or a Qdrant account.

## What is in this folder

```text
ytscraper/
├── ytrag/                 # CLI and RAG pipeline
│   ├── cli.py             # `ytrag` commands
│   ├── config.py          # environment-driven settings
│   ├── playlist.py        # YouTube playlist discovery and audio download
│   ├── transcribe.py      # faster-whisper transcription and cache
│   ├── chunk.py           # timestamp-preserving chunking
│   ├── embed.py           # sentence-transformers embeddings
│   ├── index.py           # Qdrant local/Cloud index
│   ├── answer.py          # retrieval and grounded answers
│   ├── evaluate.py        # golden-set evaluation
│   └── models.py          # videos, segments, and chunks
├── api/
│   ├── main.py            # FastAPI application
│   └── static/            # single-page web UI
├── transcripts/           # bundled transcript JSON files
├── index/vectors.npz      # bundled prebuilt vector index
├── eval/golden.json       # evaluation questions
├── main.py                # convenience CLI entry point
├── pyproject.toml         # dependencies and project script
├── uv.lock                # locked dependency versions
└── .env.example           # configuration template
```

`RAGPLAN.md` is the original build specification. This README documents the
implementation that is currently in this folder; the implementation uses
`faster-whisper`, Qdrant, and optional Gemini/Groq rather than every technology
listed in that plan.

## Why this differs from `RAGPLAN.md`

`RAGPLAN.md` describes the original design and planned phases. The code in this
folder has since been adjusted for the current Windows/local-first workflow:

| Area | `RAGPLAN.md` proposed | Current implementation | Why it changed |
|---|---|---|---|
| Transcription | `mlx-whisper` on Apple Silicon, with `faster-whisper` as fallback | `faster-whisper` / CTranslate2 | The current workflow targets Windows and CUDA/CPU rather than Apple Silicon |
| Audio | `ffmpeg` conversion to WAV | `yt-dlp` downloads audio and `faster-whisper` reads it through PyAV | Avoids a separate `ffmpeg` requirement and large intermediate WAV files |
| Embeddings | `BAAI/bge-m3` locally and Gemini embeddings for deployment | `all-MiniLM-L6-v2` by default, with bge-m3 still configurable | The smaller model makes the bundled/local workflow much faster and lighter |
| Vector store | ChromaDB | Qdrant embedded locally by default, or Qdrant Cloud | Provides the current local index format and an easy hosted option |
| LLM | Gemini as the default provider | Gemini or Groq, auto-selected from configured keys | Keeps the answer provider flexible and supports the keys used by this project |
| Distribution | Build the index after cloning | 126 transcripts and `index/vectors.npz` are already bundled | Users can run the demo without a GPU, transcription, or embedding pass |
| Delivery | Deployment is a later phase | FastAPI and the web UI are already included | The current folder has progressed beyond the initial CLI-only phases |

The plan remains useful as architectural background, but commands and defaults
should be taken from this README and `pyproject.toml`, because those describe
the code that is actually present.

## Stack

| Area | Current implementation |
|---|---|
| Package/runtime | Python 3.11 or 3.12, managed with `uv` |
| Playlist and audio | `yt-dlp` Python API |
| Transcription | `faster-whisper` / CTranslate2, `large-v3` by default |
| Chunking | Custom time-window chunks that retain timestamps |
| Embeddings | `sentence-transformers`, `all-MiniLM-L6-v2` by default |
| Vector store | Qdrant embedded locally by default, or Qdrant Cloud |
| Answer generation | Gemini or Groq, selected from configured keys |
| CLI | Typer and Rich |
| Web app | FastAPI, Uvicorn, and the YouTube IFrame API |

## Requirements

- Windows, macOS, or Linux
- Python `>=3.11,<3.13`
- [`uv`](https://docs.astral.sh/uv/)
- An LLM key only if you want written answers (`GEMINI_API_KEY` or
  `GROQ_API_KEY`)
- A CUDA-capable GPU is recommended only for transcribing a new playlist

The default local workflow does not require `ffmpeg`, a Qdrant account, or a GPU.

## Quick start: use the bundled index

These commands are for the current repository and do not transcribe anything.

### 1. Install dependencies

From `C:\Users\prath\Desktop\ytscraper` (or the directory where you cloned this
project):

```powershell
uv sync
```

For CUDA support on Windows, install the optional runtime libraries:

```powershell
uv sync --extra cuda
```

### 2. Configure an answer provider

Copy the example file:

```powershell
Copy-Item .env.example .env
```

Put one free API key in `.env`:

```dotenv
# Use either one or both.
GEMINI_API_KEY=
GROQ_API_KEY=
```

Gemini is selected automatically when `GEMINI_API_KEY` is present; otherwise
Groq is selected when `GROQ_API_KEY` is present. Retrieval-only commands work
without either key. Never commit `.env`.

### 3. Load the committed index

```powershell
uv run ytrag load
```

This imports `index/vectors.npz` into the local Qdrant store. The embedding model
is downloaded on its first use; subsequent commands reuse the local model cache.
If the prebuilt index is absent, use `uv run ytrag reindex` instead.

### 4. Start the web UI

```powershell
uv run ytrag serve
```

Open <http://127.0.0.1:8000>. Ask a question, then click a citation to seek the
embedded YouTube player to the cited timestamp.

The server prints `Ready:` after loading the embedding model. Stop it with
`Ctrl+C`.

## CLI usage

All commands are available through the installed `ytrag` script:

```powershell
uv run ytrag --help
```

### Ask and search

```powershell
# Retrieval only: no LLM call and no generated answer.
uv run ytrag search "sliding window"

# Retrieve context and generate a grounded answer.
uv run ytrag ask "memoization aur tabulation ka difference kya hai?"
```

`search` is useful for checking the returned timestamps and distances without
using an LLM quota.

### Evaluate retrieval

```powershell
uv run ytrag eval
uv run ytrag eval --verbose
```

Evaluation uses `eval/golden.json`. Retrieval entries need a real
`expect_video_id` and timestamp; refusal entries check that unrelated questions
are declined.

### Inspect and maintain the local store

```powershell
uv run ytrag stats
uv run ytrag progress
uv run ytrag reindex
uv run ytrag reindex --replace
uv run ytrag clean-audio
```

`reindex` rebuilds chunks and embeddings from cached transcripts and never
re-transcribes. Use `--replace` when changing chunk settings so old chunks are
deleted first.

## Runtime data

Unless `YTRAG_ROOT` is set, generated runtime data is stored outside the
repository:

```text
~/.ytrag/
├── audio/         # temporary downloaded audio; safe to regenerate
├── transcripts/   # transcription cache; expensive to regenerate
└── qdrant/        # embedded local vector store
```

The bundled `transcripts/` folder is a shareable snapshot used automatically by
`reindex` when no personal transcript cache exists. The committed
`index/vectors.npz` is an even faster shortcut because it avoids embedding.

The local Qdrant store is single-writer. Do not run `ask`, `search`, `reindex`,
or `stats` in another process while `ytrag serve` has the local store open.
For multi-process or hosted use, set `QDRANT_URL` and `QDRANT_API_KEY`.

## Build an index from the bundled transcripts

Use this when you want to recreate the index or test a different embedding model:

```powershell
uv run ytrag reindex --transcripts .\transcripts
```

The default model is the small `all-MiniLM-L6-v2` model. It downloads much less
than `BAAI/bge-m3` and is suitable for local machines. To use bge-m3 instead:

```powershell
$env:YTRAG_EMBED_MODEL = "BAAI/bge-m3"
uv run ytrag reindex --transcripts .\transcripts
```

Embedding dimensions are included in the Qdrant collection name, so changing
models creates a separate collection instead of mixing incompatible vectors.

## Transcribe and index a new playlist

This is the GPU-consuming workflow. First choose the Whisper language using a
representative video:

```powershell
uv run ytrag langtest "https://www.youtube.com/watch?v=<VIDEO_ID>"
```

The lectures are Hinglish. Compare `hi` and `en` output and choose the value
that preserves technical terms and matches your queries. Set it in `.env` or
your shell, for example:

```dotenv
YTRAG_WHISPER_LANG=en
```

Run a small end-to-end test before starting the complete playlist:

```powershell
uv run ytrag ingest --playlist "<PLAYLIST_URL>" --limit 1
```

Then run the full ingest:

```powershell
uv run ytrag ingest --playlist "<PLAYLIST_URL>"
```

Each video is downloaded, transcribed, chunked, embedded, and upserted. The
transcript cache and deterministic chunk IDs make the command safe to rerun:
completed work is skipped and re-indexing does not create duplicate points.

Useful ingest options:

```powershell
uv run ytrag ingest --help
uv run ytrag ingest --playlist "<PLAYLIST_URL>" --skip-transcribe
uv run ytrag ingest --playlist "<PLAYLIST_URL>" --keep-audio
uv run ytrag ingest --playlist "<PLAYLIST_URL>" --force
```

Audio is deleted after successful transcription unless `--keep-audio` is used.
Transcripts are written atomically, so an interrupted write is not treated as a
valid cache. A failed video is reported while the playlist continues; repeated
failures eventually stop the run to avoid wasting time during an outage.

To share newly generated transcripts:

```powershell
uv run ytrag export-transcripts
```

To save the current local index for other users:

```powershell
uv run ytrag export-vectors-cmd
```

The exported files are the portable artifacts. Users can load the vector file
without downloading audio or running Whisper.

## Configuration

All settings are environment-driven in `ytrag/config.py`. The most useful ones
are:

| Variable | Default | Purpose |
|---|---|---|
| `YTRAG_ROOT` | `~/.ytrag` | Runtime data directory |
| `YTRAG_WHISPER_LANG` | `en` | Whisper language/translation choice |
| `YTRAG_WHISPER_MODEL` | `large-v3` | Whisper model |
| `YTRAG_WHISPER_DEVICE` | `auto` | `auto`, `cuda`, or `cpu` |
| `YTRAG_WHISPER_BATCH` | `8` | Batched transcription size; `0` disables batching |
| `YTRAG_WHISPER_BEAM` | `5` | Decoding beam size |
| `YTRAG_CHUNK_SECONDS` | `75` | Chunk time window |
| `YTRAG_CHUNK_OVERLAP` | `15` | Chunk overlap in seconds |
| `YTRAG_LINK_REWIND` | `5` | Seconds to rewind citation links |
| `YTRAG_EMBED_MODEL` | `all-MiniLM-L6-v2` | Sentence-transformers model |
| `YTRAG_COLLECTION` | `dsa_lectures` | Base Qdrant collection name |
| `YTRAG_TOP_K` | `6` | Number of retrieved chunks |
| `YTRAG_MAX_DISTANCE` | `0.6` | Retrieval cutoff |
| `YTRAG_LLM_BACKEND` | auto | `gemini`, `groq`, or `none` |
| `YTRAG_GEMINI_MODEL` | `gemini-2.0-flash` | Gemini model |
| `YTRAG_GROQ_MODEL` | `openai/gpt-oss-120b` | Groq model |
| `QDRANT_URL` | empty | Hosted Qdrant URL; empty means local |
| `QDRANT_API_KEY` | empty | Hosted Qdrant API key |

Optional YouTube settings include `YTRAG_COOKIES_FILE` and
`YTRAG_COOKIES_FROM_BROWSER`. Cookies are sensitive; keep exported cookie files
out of source control.

## HTTP API

`ytrag serve` runs `api.main:app` with Uvicorn. The available endpoints are:

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/` | Web UI |
| `GET` | `/health` | Health and model information |
| `GET` | `/meta` | Number of cached lectures and total hours |
| `GET` | `/stats` | Index statistics |
| `POST` | `/search` | Retrieval only |
| `POST` | `/ask` | Retrieval plus grounded answer |

Request body for `/search` and `/ask`:

```json
{
  "question": "bfs kaise kaam karta hai?",
  "top_k": 6
}
```

`/ask` limits requests per IP in-process and caps question length. The limiter
is not shared between multiple workers; use a shared store such as Redis before
running multiple workers publicly.

## Troubleshooting

Run the command that matches the problem:

```powershell
uv run ytrag --help
uv run ytrag stats
uv run ytrag search "test question"
uv run ytrag load
```

| Symptom | Likely fix |
|---|---|
| `GEMINI_API_KEY is not set` or `GROQ_API_KEY is not set` | Add the selected key to `.env`, or use `search` instead of `ask` |
| `No prebuilt index found` | Run from this project directory or build with `reindex` |
| `These vectors were built with...` | Use the matching `YTRAG_EMBED_MODEL`, or reindex |
| Local index is already open | Stop `uv run ytrag serve` before another local-store command |
| Whisper/CUDA load fails | Run `uv sync --extra cuda`, or force `YTRAG_WHISPER_DEVICE=cpu` |
| YouTube requests are challenged | Slow the ingest, configure cookies, and rerun; cached work is preserved |
| LLM quota errors | Use `search`, switch providers, or wait for the provider quota to reset |

The `preflight` command exercises the long ingest path:

```powershell
uv run ytrag preflight --playlist "<PLAYLIST_URL>"
```

It is intended for a fully configured ingest environment and currently checks
the hosted-Qdrant/Groq settings used by the original ingestion setup. For the
bundled local demo, `load`, `stats`, and `search` are the more appropriate
smoke tests.

## Design choices

- Chunks are built from transcript segments rather than a generic text splitter,
  so every result keeps its video ID and start/end timestamps.
- Whisper VAD settings and repetition filtering reduce hallucinated transcript
  regions caused by silence or music.
- Transcript files are written atomically because re-transcribing a large
  playlist is the most expensive operation.
- Qdrant upserts use deterministic point IDs, making retries and reruns
  idempotent.
- Results beyond `YTRAG_MAX_DISTANCE` are discarded before answer generation.
  Answers without citations are marked ungrounded rather than presented with
  misleading timestamps.
- Citation links rewind a few seconds so playback starts near the beginning of
  the explanation instead of the middle of a sentence.
