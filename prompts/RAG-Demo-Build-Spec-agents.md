# RAG "Glass Box" Demo — Build Specification

---

## Part 1: What We're Building

### Vision

A **"Glass Box" RAG Demo** — a fully interactive, browser-based simulation that makes every invisible layer of Retrieval-Augmented Generation visible. The user experiences a working chat interface asking questions about company documents, while simultaneously watching the entire RAG pipeline operate in real-time: documents being chunked, text being embedded into vectors, similarity search finding nearest neighbors in a 3D vector space visualization, re-ranking refining results, and the LLM generating grounded responses with citations.

This is not a slideshow. It's a **living RAG architecture** that you can interact with.

### Why a Simulation (Not a Live System)

A browser-based simulation is the right choice because:

- **Controllable pacing** — You can pause, step through, and replay any phase. Impossible with a real pipeline.
- **Zero infrastructure** — No vector database, no embedding API keys, no LLM calls. Opens in a browser.
- **Perfect for any audience** — Engineers, stakeholders, and mixed groups can all follow along.
- **Observable by design** — Every embedding, similarity score, and prompt construction can be inspected because we control the entire pipeline.
- **Portable** — Share as a single file. Demo anywhere.

---

## Part 2: Technical Implementation

### Single-File Architecture

The entire application is a single `index.html` file containing:
- CDN links: React 18, ReactDOM, Babel (inline JSX), Tailwind CSS
- Inline `<style>` block for animations and base styles
- Single `<script type="text/babel">` block with all application code

Code is organized in numbered sections:

| # | Section | Contents |
|---|---------|----------|
| 1 | CONSTANTS | Colors, component IDs, document color map |
| 2 | DOCUMENTS | 3 source documents with metadata |
| 3 | CHUNKS | 12 chunks with text, embeddings, 3D positions |
| 4 | QUERIES | 2 queries with similarity scores, results, LLM prompts/responses |
| 5 | ACT_METADATA | Title, subtitle, description, watchFor for each act |
| 6 | STEPS | 33 steps across 4 acts with narration and state mutations |
| 7 | useSimulation | Core hook: state machine, step navigation, playback |
| 8 | ArchitectureDiagram | SVG pipeline diagram with animated data flow |
| 9 | VectorSpaceView | 3D isometric scatter plot with drag-to-orbit |
| 10 | DiagramPanel | Toggle container: Pipeline View / Vector Space |
| 11 | ChunkLog | Scrolling timeline of chunk operations |
| 12 | PayloadInspector | Detail view for chunks, queries, results, prompts, responses |
| 13 | DataInspectorPanel | Container: ChunkLog (40%) + PayloadInspector (60%) |
| 14 | ChatPanel | Chat messages with citations and thinking indicator |
| 15 | StepNarrator | Narration text, phase badge, navigation controls, act buttons |
| 16 | ActTransition | Full-screen interstitial between acts |
| 17 | LandingScreen | Title, description, Start Demo button, skip-to links |
| 18 | HeaderBar | Act buttons, progress bar |
| 19 | App | Root component, act transition detection, keyboard shortcuts |

### Color Scheme

| Component | Color | Hex | Usage |
|-----------|-------|-----|-------|
| Application/Host | Amber | `#f59e0b` | Orchestrator, prompt builder, Act 0 intro |
| Documents/Chunks | Green | `#10b981` | Source data, chunk entries, Act 1 |
| Vector DB | Blue | `#3b82f6` | Storage, indexing, search, Act 2 |
| Embedding Model | Teal | `#06b6d4` | Text → vector transformation |
| LLM | Purple | `#a855f7` | Generation, response |
| Re-ranker | Pink | `#ec4899` | Score refinement, Act 3 |

### Phase Colors

Each pipeline phase has a color used for the phase badge in the narrator:

```
intro: #f59e0b (amber)
loading: #10b981, chunking: #10b981
embedding: #06b6d4
indexing: #3b82f6
query_start: #f59e0b, query_embedding: #06b6d4, searching: #3b82f6
reranking: #ec4899
assembling: #f59e0b, generating: #a855f7, responding: #f59e0b
```

---

## Part 3: Demo Structure — Four Acts (0–3)

### Act 0 — The Problem: Why RAG? (6 steps)

**Purpose:** Motivate the entire demo by showing a plain LLM failing to answer a company-specific question. The user asks about Disney's PTO policy and the LLM admits it has no access to internal documents.

**Chat flow:**

| Step | id | Chat Message | Role |
|------|----|-------------|------|
| 0 | a0s1 | "What is PTO?" | user |
| 1 | a0s2 | "PTO stands for Paid Time Off. It is a workplace policy that combines vacation days, sick leave, and personal days into a single bank of time..." | assistant |
| 2 | a0s3 | "I work at Disney, what is our PTO policy?" | user |
| 3 | a0s4 | "Searching for Disney PTO policy..." | thinking |
| 4 | a0s5 | "I searched for Disney's PTO policy but wasn't able to find any specific results. If you could share your company's PTO policy document, I'd be happy to provide a summary for you." | assistant |
| 5 | a0s6 | "IF I KNEW THE POLICY, I WOULDN'T BE ASKING YOU!" | user |

**Step details:**
- All steps: `activeComponents:[]`, `dataFlow:null`, `vectorSpaceUpdate:null`, `chunkLogUpdate:null`, `inspectorContent:null`
- All use `phase:'intro'` and `act:0`
- Step 0 sets `stateMutations:{systemPhase:'intro'}`, rest are `{}`
- The thinking→assistant replacement (a0s4→a0s5) uses the existing `computeChatAtStep` logic: when an assistant message arrives, any prior thinking message is removed

**Key narrative arc:**
- General knowledge question → LLM answers fine from training data
- Company-specific question → LLM searches but finds nothing
- User frustration → motivates the RAG pipeline that follows

---

### Act 1 — Ingestion Pipeline: The Setup (10 steps)

**Purpose:** Show how raw documents get transformed into a searchable vector index. This is the "offline" phase that happens before any questions are asked.

**Simulated Documents (3 total, all public):**

| Document | File | Chunks |
|----------|------|--------|
| PTO Policy | `pto-policy.md` | 4 |
| Expense Policy | `expense-policy.md` | 5 |
| Employee Directory | `employee-directory.md` | 3 |
| **Total** | | **12** |

**Chunk Breakdown:**

Chunks use fixed-size windowing (~512 characters) with 50-character overlap. Chunk boundaries are arbitrary — they can fall mid-sentence. Overlapping text appears at the end of chunk N and the start of chunk N+1.

```
pto-policy.md (4 chunks):
  Chunk 0 [0:512]:    Overview + PTO allowances by tenure
  Chunk 1 [462:974]:  Carryover limits + requesting time off
  Chunk 2 [924:1436]: Blackout dates + holidays & sick leave
  Chunk 3 [1386:1720]: Sick leave details + documentation requirements

expense-policy.md (5 chunks):
  Chunk 4 [0:512]:    Travel & lodging + meal limits intro
  Chunk 5 [462:974]:  Meal limits + client entertainment + professional dev
  Chunk 6 [924:1436]: Professional dev + home office + submission process
  Chunk 7 [1386:1898]: International expenses + approval thresholds
  Chunk 8 [1848:2160]: Capital expenditures + recurring expense review

employee-directory.md (3 chunks):
  Chunk 9 [0:512]:    Engineering department (Mickey, Minnie, Donald, Goofy)
  Chunk 10 [462:870]: Product department (Snow White, Cinderella, Elsa) + Exec intro
  Chunk 11 [820:1100]: Executive leadership (Walt Disney, CEO)
```

**Step sequence:**

| Step | id | Phase | Label | Key Action |
|------|----|-------|-------|------------|
| 6 | a1s1 | loading | Document Corpus Overview | Show 3 documents, set systemPhase:'ingesting' |
| 7 | a1s2 | loading | Load PTO Policy | Load pto-policy.md, show in inspector |
| 8 | a1s3 | chunking | Chunk PTO Policy → 4 Chunks | Create chunks 0-3, show in chunk log |
| 9 | a1s4 | embedding | Embed PTO Chunks | Add chunks 0-3 to vector space |
| 10 | a1s5 | indexing | Index PTO Chunks in Vector DB | 4/12 indexed |
| 11 | a1s6 | chunking | Load & Chunk Expense Policy → 5 Chunks | Create chunks 4-8 |
| 12 | a1s7 | indexing | Embed & Index Expense Chunks | Add chunks 4-8 to vector space, 9/12 indexed |
| 13 | a1s8 | embedding | Load & Process Employee Directory → 3 Chunks | Create chunks 9-11, add to vector space |
| 14 | a1s9 | indexing | Index Employee Chunks | 12/12 indexed |
| 15 | a1s10 | indexing | Indexing Complete — 12 Chunks Ready | ingestionComplete:true |

**Payload Inspector behavior:**
- During loading/chunking phases (before embedding), the inspector shows chunk text and source but displays "Embedding not yet computed — awaiting embedding step" instead of embedding vectors and 3D positions
- Once chunks are embedded (added to vector space), the inspector shows the full embedding array (first 8 dims) and 3D position
- The `embeddedChunks` set (derived from `vectorSpace.visibleChunks`) controls this conditional rendering

---

### Act 2 — Simple Retrieval: The Basic Flow (7 steps)

**Purpose:** Show the basic query-time flow: embed question → search → retrieve → generate. The simplest RAG scenario where relevant chunks come from a single document.

**Query:** "What is our PTO policy?"

**Step sequence:**

| Step | id | Phase | Label | Key Action |
|------|----|-------|-------|------------|
| 16 | a2s1 | query_start | User Asks: 'What is our PTO policy?' | User message in chat |
| 17 | a2s2 | query_embedding | Embed the Query | Query dot appears in vector space |
| 18 | a2s3 | searching | Similarity Search | Top-3 highlighted, lines drawn to nearest chunks |
| 19 | a2s4 | searching | Top-K Results | Results shown in inspector |
| 20 | a2s5 | assembling | Assemble Context & Prompt | Full LLM prompt visible |
| 21 | a2s6 | generating | LLM Generates Response | Thinking indicator in chat |
| 22 | a2s7 | responding | Response with Citations | Final answer with [1][2][3] citations |

**Retrieved Chunks (top-3 by cosine similarity):**

```
[1] Chunk 0 — pto-policy.md: 1/4 [0:512]      (0.92)
[2] Chunk 1 — pto-policy.md: 2/4 [462:974]     (0.89)
[3] Chunk 3 — pto-policy.md: 4/4 [1386:1720]   (0.84)
```

All from the same document — the query lands squarely in the PTO cluster in vector space.

**LLM Response:**
> "According to the PTO policy, Disney Corp provides PTO based on years of service: employees with 0-2 years receive 15 days, 3-5 years get 20 days, 6-9 years get 25 days, and 10+ years receive 30 days per year. PTO accrues bi-weekly with a maximum carryover of 5 days per calendar year [1][2]. The company also observes 11 paid holidays and provides 10 sick days annually [3]."

---

### Act 3 — Multi-Doc Retrieval: Re-ranking Saves the Day (10 steps)

**Purpose:** Show a more complex query that benefits from re-ranking. Demonstrates that initial vector retrieval ordering isn't always optimal — the cross-encoder re-ranker improves results.

**Query:** "What's the meal reimbursement limit for business travel?"

**Step sequence:**

| Step | id | Phase | Label | Key Action |
|------|----|-------|-------|------------|
| 23 | a3s1 | query_start | User Asks About Meal Reimbursement | User message, reset query state |
| 24 | a3s2 | query_embedding | Embed the Query | Query dot near expense cluster |
| 25 | a3s3 | searching | Initial Similarity Search | Top-5 retrieved, all expense-policy chunks |
| 26 | a3s4 | searching | Initial Ranking (Imperfect) | Show ordering issues in inspector |
| 27 | a3s5 | reranking | Enter Re-ranking Phase | Cross-encoder explanation |
| 28 | a3s6 | reranking | Re-ranker Scores Each Pair | New scores, order shifts |
| 29 | a3s7 | reranking | Re-ranked Results | Final ranking shown |
| 30 | a3s8 | assembling | Assemble Re-ranked Context | Top-3 re-ranked chunks in prompt |
| 31 | a3s9 | generating | LLM Generates Response | Thinking indicator |
| 32 | a3s10 | responding | Multi-Source Citations | Final answer with citations |

**Initial Retrieval (top-5 by cosine similarity — all expense-policy):**

```
Chunk 5 — expense-policy: 2/5 [462:974]   (0.88) — Meal limits & entertainment
Chunk 4 — expense-policy: 1/5 [0:512]     (0.85) — Travel & lodging
Chunk 7 — expense-policy: 4/5 [1386:1898] (0.72) — International & approvals
Chunk 6 — expense-policy: 3/5 [924:1436]  (0.68) — Office & professional dev
Chunk 8 — expense-policy: 5/5 [1848:2160] (0.55) — Capital expenditures & submission
```

All 5 results are from the expense-policy cluster — the query lands nearby in vector space. No stray chunks from other document clusters.

**Re-ranked Results (cross-encoder scores):**

```
Chunk 5 — Meals & Entertainment     (0.88 → 0.95) ↑
Chunk 4 — Travel & Lodging          (0.85 → 0.91) ↑
Chunk 8 — Submission/Approval Rules  (0.55 → 0.78) ↑↑ promoted from 5th to 3rd
Chunk 7 — International/Approvals   (0.72 → 0.52) ↓  demoted
Chunk 6 — Office & Prof Dev         (0.68 → 0.48) ↓  demoted
```

**Key teaching point:** The re-ranker's value here is reordering **within** the correct cluster. Chunk 8 (receipt requirements, approval thresholds) was ranked last by embedding similarity but promoted to 3rd by the cross-encoder, which recognized its direct relevance to reimbursement questions.

**LLM Response:**
> "For business travel, meal reimbursement limits are: Breakfast up to $20, Lunch up to $30, and Dinner up to $50 per person [1][2]. Client entertainment meals can be up to $150 per person but require pre-approval. Alcohol is reimbursable only during client entertainment, capped at $50 per person [1]. All meal expenses over $25 require itemized receipts [3]."

---

## Part 4: Data Model

### Embeddings & 3D Positions

Each chunk has:
- `embedding`: 8-dimensional array (abbreviated; real embeddings would be 768–1536 dims)
- `position3D`: `{x, y, z}` coordinates for the 3D isometric scatter plot (values 0–1)

Positions are hand-placed to create meaningful clusters:
- **PTO chunks (0-3):** upper-left region `x:0.12–0.22, y:0.12–0.24, z:0.22–0.30`
- **Expense chunks (4-8):** center-right region `x:0.56–0.68, y:0.28–0.42, z:0.52–0.62`
- **Employee chunks (9-11):** lower-left region `x:0.14–0.26, y:0.58–0.65, z:0.78–0.85`

### Query Similarity Scores

Each query has a `similarities` object mapping chunk IDs to cosine similarity scores. These are hand-tuned to:
1. Be highest for chunks in the same topical cluster as the query
2. Match the visual proximity in the 3D scatter plot
3. Be internally consistent — the 3D visualization computes `nearestIds` by sorting `query.similarities` and taking the top K, so similarity rankings **must** match the intended visual

**Important:** The `topK` array in query data and the `similarities` object must agree. The vector space view computes nearest chunks independently from `similarities`, not from `topK`. If they disagree, the visual will show different chunks than the data inspector.

### State Shape

```javascript
INITIAL_STATE = {
  systemPhase: 'landing',    // 'landing' | 'intro' | 'ingesting' | 'querying'
  ingestionPhase: null,       // 'loading' | 'chunking' | 'embedding' | 'indexing'
  currentDocIndex: -1,        // 0, 1, or 2
  chunksIndexed: 0,           // 0 through 12
  ingestionComplete: false,
  activeQueryIndex: -1,       // 0 or 1
  queryComplete: false,
}
```

Each step can mutate state via `stateMutations` (shallow merge).

### Step Data Shape

```javascript
{
  id: 'a1s3',                    // act + step identifier
  act: 1,                        // act number (0–3)
  phase: 'chunking',             // pipeline phase (determines phase badge color)
  label: 'Chunk PTO Policy',     // short label shown in UI
  activeComponents: [C.CHUNKER], // which architecture boxes highlight
  dataFlow: {from, to, label},   // animated arrow between components (or null)
  narration: "...",               // educational text for this step
  stateMutations: {},             // state changes applied at this step
  chatUpdate: {role, content},    // chat message to add (or null)
  chunkLogUpdate: [{type, chunkId, score}],  // chunk log entries (or null)
  inspectorContent: 'chunk:0',   // what to show in payload inspector (or null)
  vectorSpaceUpdate: {            // vector space changes (or null)
    addChunks: [0,1,2,3],        // chunk IDs to make visible
    showQuery: 0,                 // query index to show (or null to hide)
    highlightNearest: {queryId, topK},  // nearest-neighbor highlighting
  },
}
```

### Act Offsets & Lengths

```javascript
ACT_OFFSETS = {0:0, 1:6, 2:16, 3:23};
ACT_LENGTHS = {0:6, 1:10, 2:7, 3:10};
// Total: 33 steps
```

---

## Part 5: UI Components

### Layout

```
┌──────────────────────────────────────────────────────────────────────┐
│  RAG Glass Box                              [Act 0][1][2][3]  ━━ 12/33│
├────────────────────────┬─────────────────────────────────────────────┤
│                        │                                             │
│   CHAT PANEL (38%)     │   DIAGRAM PANEL (flex 5)                   │
│                        │   ┌─ Architecture Diagram ────────────────┐│
│   "RAG Assistant"      │   │  DocStore → Chunker → Embedder → VDB ││
│   Questions & AI       │   │  AppHost ← LLM ← Reranker ← QEngine ││
│   responses with       │   │                                        ││
│   inline citations     │   │  Toggle: [Pipeline View] [Vector Space]││
│                        │   └────────────────────────────────────────┘│
│   Typing dots for      │                                             │
│   thinking state       │   DATA INSPECTOR (flex 3)                  │
│                        │   ┌─ Chunk Log (40%) ─┐┌─ Payload (60%) ──┐│
│                        │   │ Created/Retrieved  ││ Chunk text        ││
│                        │   │ with scores        ││ Embedding (if     ││
│                        │   │                    ││   embedded)        ││
│                        │   └────────────────────┘└──────────────────┘│
│                        │                                             │
│                        │   STEP NARRATOR (flex 3)                   │
│                        │   ┌────────────────────────────────────────┐│
│                        │   │ [INTRO] Step 3 of 6 • Act 0    4/33  ││
│                        │   │                                        ││
│                        │   │ "Now the user wants something specific ││
│                        │   │  — their company's actual policy..."   ││
│                        │   │                                        ││
│                        │   │ [Act 0][1][2][3]  [◀][▶ Play][▶] 1x  ││
│                        │   └────────────────────────────────────────┘│
└────────────────────────┴─────────────────────────────────────────────┘
```

### Keyboard Shortcuts

| Key | Action |
|-----|--------|
| Right Arrow / Space | Next step |
| Left Arrow | Previous step |
| P | Toggle play/pause |
| 0–3 | Jump to Act |
| Escape | Return to landing page + reset |

### 3D Vector Space Visualization

An interactive isometric projection of high-dimensional embedding space:

- **Rendering:** SVG-based with `project3D()` function (isometric projection with configurable Y-rotation)
- **Interaction:** Mouse drag to orbit (rotates around Y axis)
- **Grid:** Three-plane grid (floor, back wall, side wall) with 5 subdivisions each
- **Chunk dots:** Color-coded by source document, sized by depth (parallax), with tooltips
- **Query dot:** Pulsing white ring with "Query" label
- **Nearest-neighbor lines:** Solid lines from query to top-K chunks with similarity scores
- **Legend:** Document color key at bottom of view
- **Disclaimer:** "3D projection of high-dimensional vectors — distances are approximate. Drag to orbit."

During ingestion, dots appear as chunks are embedded. During queries, the query dot and nearest-neighbor lines appear.

### Chat Panel

- **Header:** "RAG Assistant" / "Ask questions about company documents"
- **User messages:** Right-aligned, blue background
- **Assistant messages:** Left-aligned, dark background, with inline `[1][2][3]` citations in teal
- **Thinking state:** Typing dots animation + message text (e.g., "Searching for Disney PTO policy...")
- **Citation footnotes:** Below assistant messages, listing source file and chunk section
- **Thinking→assistant replacement:** When an assistant message follows a thinking message, the thinking message is removed (handled by `computeChatAtStep`)

### Act Transition Screen

Full-screen overlay shown when transitioning between acts:
- Act number + color (amber/green/blue/pink for Acts 0/1/2/3)
- Large title + italic subtitle
- Description paragraph
- "Watch for:" callout box
- Auto-dismisses after 3.5 seconds, or click to dismiss

### Landing Screen

- Title: "RAG Glass Box Demo"
- Subtitle: "See every step of Retrieval-Augmented Generation"
- Description of what the demo shows
- "Start Demo" button (begins at Act 0, step 0)
- Skip-to links: Act 0 (Why RAG?), Act 1 (Ingestion), Act 2 (Simple Query), Act 3 (Re-ranking)

---

## Part 6: Key Design Decisions & Invariants

### Data Consistency Rules

1. **Similarity scores must match visual positions.** The `nearestIds` computation in `VectorSpaceView` sorts `query.similarities` to find the top-K — this must produce the same set as the `topK` array in query data. If they disagree, the 3D visualization will draw lines to different chunks than what the data inspector and chunk log show.

2. **Chunks within a query's top-K should be visually close to the query in 3D space.** A retrieval line stretching far across the scatter plot to a different cluster looks wrong and confuses the spatial metaphor.

3. **Re-ranked results can only contain chunks from the initial top-K.** The re-ranker reorders candidates it received — it doesn't discover new ones. Every `chunkId` in `query.reranked` must also appear in `query.topK`.

4. **Embeddings are only visible after the embedding step.** The PayloadInspector conditionally shows embedding vectors and 3D positions based on `embeddedChunks` (derived from `vectorSpace.visibleChunks`). During loading/chunking phases, it shows "Embedding not yet computed."

5. **Chat message replacement.** The `computeChatAtStep` function replaces thinking messages with assistant messages. A `role:'thinking'` message is removed when the next `role:'assistant'` message arrives. This is used in Act 0 (search→failure) and Acts 2-3 (generating→response).

### Adding New Acts

To add a new act (e.g., Act 4: Guardrails):

1. **Add steps** to the end of the `STEPS` array with `act:4` and appropriate phases
2. **Add metadata** to `ACT_METADATA` array (index 4)
3. **Update `ACT_OFFSETS`** — add `4: <first step index>`
4. **Update `ACT_LENGTHS`** — add `4: <number of steps>`
5. **Update UI ranges** — change `[0,1,2,3]` to `[0,1,2,3,4]` in:
   - `HeaderBar` act buttons
   - `StepNarrator` act buttons
   - `LandingScreen` skip-to links
   - Keyboard shortcut range (`e.key >= '0' && e.key <= '4'`)
6. **Add act color** to `actColors` array in `ActTransition`
7. **Add phase colors** to `PHASE_COLORS` if new phases are introduced
8. **Add any new query data** to `QUERIES` array if the act involves a new question

### Adding New Documents

To add a new document:

1. **Add to `DOCUMENTS`** array with id, filename, title, chunkCount, preview
2. **Add chunks** to `CHUNKS` array with sequential IDs, text, `position3D`, `embedding`
3. **Add color** to `DOC_COLORS` for the new source name
4. **Update all `query.similarities`** objects to include scores for the new chunk IDs
5. **Place 3D positions** to create a visually distinct cluster
6. **Update ingestion steps** in Act 1 to load, chunk, embed, and index the new document
7. **Update `ACT_LENGTHS`** and downstream `ACT_OFFSETS` if step count changes

### Critical Lessons Learned

- **All React hooks must be called before any conditional returns** (Rules of Hooks). In a single-file app with many components, this is easy to violate.
- **Use `useRef` for keyboard event handlers** to always access current simulation state (avoids stale closures).
- **The vector space visual and data model must agree.** The 3D view computes nearest-IDs from `query.similarities` independently — it does not read `topK`. Any mismatch creates confusing visuals.
- **Act transitions trigger on `sim.currentAct !== prevActRef.current`.** The `prevActRef` must be initialized to match the starting act (currently 0).
- **`computeChatAtStep` replays all steps from 0 to current index** to build chat state. This means chat messages are deterministic regardless of navigation direction.

---

## Part 7: Company & Document Context

The demo uses **Disney Corp** as the fictional company. All documents reference Disney Corp employees (Mickey Mouse, Minnie Mouse, Donald Duck, etc.) and Disney-specific details.

Act 0 establishes this context when the user says "I work at Disney, what is our PTO policy?" — connecting the chat scenario to the documents that will be ingested in Act 1.

The three documents cover:
- **PTO Policy:** Vacation allowances by tenure, carryover limits, request procedures, holidays, sick leave
- **Expense Policy:** Travel & lodging, meal reimbursement, professional development, submission process, approval thresholds
- **Employee Directory:** Engineering, Product, and Executive departments with names, roles, offices, employee IDs
