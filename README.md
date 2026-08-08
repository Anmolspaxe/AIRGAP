# Airgap — a browser LLM that never phones home

**A single HTML file that runs a real language model entirely inside your browser tab — no server, no API key, no cloud AI dependency of any kind.**

Built for [OSDHack 2026](#) — theme: **On-Device AI**.

![status](https://img.shields.io/badge/status-hackathon_build-E8A33D)
![license](https://img.shields.io/badge/license-MIT-8C877D)
![runs on](https://img.shields.io/badge/runs%20on-Chrome%2FEdge%20113%2B-8C877D)

---

## What it does

Open `airgap.html`, pick a model, click **Load locally**. The model downloads once (cached by the browser afterward) and then runs entirely on your device:

- **Chat** — full streaming conversation with a local LLM, powered by [WebLLM](https://github.com/mlc-ai/web-llm) running on your GPU via WebGPU
- **Local RAG** — upload your own `.txt`/`.md` files, they get chunked and embedded on-device, and relevant chunks get retrieved and injected into answers — no vector database, no cloud embedding API
- **CPU fallback** — if your browser/OS doesn't support WebGPU (older Windows, unsupported GPU, etc.), it automatically falls back to a WebAssembly runtime ([transformers.js](https://github.com/xenova/transformers.js)) — slower, but still 100% on-device
- **Live proof, not just a claim** — a telemetry panel hooks `window.fetch` the instant the model is ready. If a single network request fires while you're chatting, the counter turns red and tells you what fired. Turn on airplane mode and keep talking to it.

No backend. No API key. No account. It runs from a `file://` URL or any static file host.

---

## Why this fits "On-Device AI"

The hackathon rules allow cloud for *support* (hosting, auth, storage) but require the **core AI feature** to run locally. Airgap has no backend to allow cloud into in the first place — there is nothing running anywhere except:
1. A one-time model weight download (equivalent to downloading a font or an image — not an AI service call)
2. Your own GPU/CPU executing the model afterward

Every chat turn, every embedding, every retrieval — computed on-device. The network watchdog exists specifically so this isn't just an assertion you have to trust.

---

## Quick start

**Option A — just open it**
Download `airgap.html` and open it directly in Chrome or Edge (113+). If module imports are blocked on `file://` in your browser, use Option B.

**Option B — serve it locally**
```bash
git clone https://github.com/<you>/airgap.git
cd airgap
python3 -m http.server 8000
# Windows: use `python` instead of `python3` if that's what's installed
```
Then open `http://localhost:8000/airgap.html`.

**Option C — live demo**
Hosted via GitHub Pages: `https://<you>.github.io/airgap/` *(no download needed, works straight from the link)*

---

## Requirements

| Mode | Requirement | Speed |
|---|---|---|
| GPU (default) | **Windows 10/11**, macOS, or Linux + Chrome/Edge 113+, WebGPU-capable GPU + drivers | Real-time streaming, tens of tokens/sec |
| CPU fallback (automatic) | Any modern browser with WebAssembly — including Windows 8/8.1 and other unsupported OSes | Much slower — single-digit tokens/sec, expect a real wait |

**Why Windows 10+ specifically:** Chromium's WebGPU implementation on Windows requires Direct3D 12, which Microsoft never shipped on Windows 8/8.1 or earlier. This is an OS-level limitation — no browser update, flag, or driver update can unlock WebGPU below Windows 10. Airgap detects this automatically and switches to the CPU/WASM runtime instead of failing, so it still runs — just slower.

**Known unsupported for GPU mode:** iOS Safari and iOS Chrome — Apple hasn't shipped WebGPU in WebKit yet. Android Chrome/Edge and any desktop Chrome/Edge on Windows 10+/macOS/Linux work.

First model load needs an internet connection to download weights (300MB–1GB depending on model). Every conversation after that works fully offline.

---

## Architecture

```
┌─────────────────────────────────────────────┐
│  Browser tab (airgap.html — one file)        │
│                                               │
│  ┌────────────┐   ┌────────────────────────┐ │
│  │  Chat UI   │──▶│  WebLLM (WebGPU)        │ │
│  │            │   │  or                     │ │
│  │            │◀──│  transformers.js (WASM) │ │
│  └────────────┘   └────────────────────────┘ │
│        │                                     │
│        ▼                                     │
│  ┌────────────────────────────────────────┐  │
│  │  Local RAG: chunk → embed → cosine sim  │  │
│  │  (in-memory, per-session)               │  │
│  └────────────────────────────────────────┘  │
│        │                                     │
│        ▼                                     │
│  ┌────────────────────────────────────────┐  │
│  │  Network watchdog (fetch() hook)        │  │
│  │  proves zero calls post-load             │  │
│  └────────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
         no server, no API, no database
```

## Tech stack

- [WebLLM (MLC)](https://github.com/mlc-ai/web-llm) — LLM inference via WebGPU
- [transformers.js](https://github.com/xenova/transformers.js) — CPU/WASM fallback inference + embeddings
- Vanilla HTML/CSS/JS — no build step, no framework, one file

## Known limitations (being upfront)

- Knowledge base (uploaded files) is in-memory only — it clears on page refresh. Persisting to IndexedDB is a planned improvement.
- CPU fallback mode doesn't have true token-by-token streaming (the underlying library's streaming API isn't stable across versions), so it generates the full reply first, then reveals it progressively.
- Larger models on lower-end GPUs/phones may be slow or fail to load — the model picker is filtered by device type to keep options realistic.

## License

MIT — see [LICENSE](./LICENSE).