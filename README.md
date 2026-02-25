# RAG Glass Box Demo

An interactive browser-based simulation that visualizes the **Retrieval-Augmented Generation (RAG)** pipeline in real time. Step through every stage — from document chunking and embedding through vector similarity search, re-ranking, and grounded LLM generation with citations.

## Quick Start

Open `index.html` in any modern browser. That's it — no install, no build step, no server required.

> The file loads React, Babel, and Tailwind from CDN, so you need an internet connection on first load.

## What You'll See

The demo walks through **4 acts**:

| Act | Title | What Happens |
|-----|-------|-------------|
| 0 | The Problem — Why RAG? | Ask a plain LLM about Disney's PTO policy — it searches, finds nothing, and asks *you* to provide the document. Motivates the entire pipeline. |
| 1 | Ingestion Pipeline | 3 documents loaded, chunked (512-char windows with overlap), embedded into vectors, and indexed in a vector DB — 12 searchable chunks |
| 2 | Simple Retrieval | "What is our PTO policy?" — embed the query, find nearest chunks via cosine similarity, assemble context, generate a cited answer |
| 3 | Multi-Doc Retrieval | "What's the meal reimbursement limit?" — initial search gets close, then a cross-encoder re-ranker reorders results for better accuracy |

Each step shows:
- **Architecture Diagram** — animated SVG pipeline with highlighted components and data flow packets
- **3D Vector Space** — isometric scatter plot of embeddings with drag-to-orbit, nearest-neighbor lines, and similarity scores
- **Chat Panel** — conversation UI with typing indicators, inline citations, and source footnotes
- **Chunk Log** — timeline of chunk creation, retrieval, and re-ranking with scores
- **Payload Inspector** — chunk text, embedding vectors, similarity results, full LLM prompts, and responses
- **Step Narrator** — explanation of what's happening and why at each step

## Controls

| Input | Action |
|-------|--------|
| **Next / Back** buttons | Step forward or backward |
| **Play** button | Auto-advance through steps |
| Arrow keys | Navigate steps |
| `0` `1` `2` `3` | Jump to act |
| `P` | Toggle play/pause |
| `Esc` | Return to landing page |

## Simulated Data

The demo uses **Disney Corp** as a fictional company with 3 internal documents:

| Document | Chunks | Content |
|----------|--------|---------|
| `pto-policy.md` | 4 | Vacation allowances by tenure, carryover limits, holidays, sick leave |
| `expense-policy.md` | 5 | Travel & lodging, meal reimbursement, professional dev, approval thresholds |
| `employee-directory.md` | 3 | Engineering, Product, and Executive departments |

All embeddings, similarity scores, and LLM responses are pre-computed — no API calls, no keys needed.

## Project Structure

```
index.html                                  # Self-contained demo (open directly in browser)
prompts/RAG-Architecture-Deep-Dive.md       # Educational guide to RAG concepts
prompts/RAG-Demo-Build-Spec-agents.md       # Build spec and maintenance reference
README.md                                   # This file
```

## Tech Stack

- **React 18** — UI rendering (CDN)
- **Babel Standalone** — in-browser JSX transpilation (CDN)
- **Tailwind CSS** — utility styles (CDN)
- **Zero dependencies** — no npm, no node_modules, no bundler

## License

MIT
