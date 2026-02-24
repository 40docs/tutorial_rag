# RAG "Glass Box" Demo — Build Specification &amp; Agent Team Prompt

---

## Part 1: What We're Building

### Vision

A **"Glass Box" RAG Demo** — a fully interactive, browser-based simulation that makes every invisible layer of Retrieval-Augmented Generation visible. The user experiences a working chat interface asking questions about company documents, while simultaneously watching the entire RAG pipeline operate in real-time: documents being chunked, text being embedded into vectors, similarity search finding nearest neighbors in a 2D vector space visualization, re-ranking refining results, and the LLM generating grounded responses with citations.

This is not a slideshow. It's a **living RAG architecture** that you can interact with.

### Why a Simulation (Not a Live System)

A browser-based simulation is the right choice because:

- **Controllable pacing** — You can pause, step through, and replay any phase. Impossible with a real pipeline.
- **Zero infrastructure** — No vector database, no embedding API keys, no LLM calls. Opens in a browser.
- **Perfect for any audience** — Engineers, stakeholders, and mixed groups can all follow along.
- **Observable by design** — Every embedding, similarity score, and prompt construction can be inspected because we control the entire pipeline.
- **Portable** — Share as a single file. Demo anywhere.

---

### The Demo Structure: Four Acts

The demo follows a deliberate narrative arc. Each act builds on the previous one, layering understanding.

---

#### Act 1 — Ingestion Pipeline (The Setup)

**Purpose:** Show how raw documents get transformed into a searchable vector index. This is the "offline" phase that happens before any questions are asked.

**What the audience sees:**

1. Five raw documents displayed with their filenames and classifications
2. A "Start Ingestion" button that triggers the pipeline
3. **Document Loading:**
   - Each document loads sequentially with a brief preview
   - Metadata extracted: filename, classification (public vs confidential), section headers
4. **Chunking:**
   - Documents split into segments with visual boundaries
   - Chunk count per document shown (17 chunks total across 5 docs)
   - Chunk text visible in the Chunk Log panel
   - Overlap regions highlighted
5. **Embedding:**
   - Each chunk animated through the embedding model
   - Vector representation shown as a list of numbers (abbreviated)
   - 2D position assigned for the vector space scatter plot
   - Dots appear one by one on the 2D vector space visualization
6. **Indexing:**
   - Vectors stored in the vector DB with metadata
   - Vector DB "fills up" as chunks are indexed
   - Final state: 17 chunks indexed, ready for queries

**Simulated Documents (from project_bedrock):**

| Document | File | Classification | Chunks |
|----------|------|---------------|--------|
| PTO Policy | `pto-policy.md` | public | 4 |
| Expense Policy | `expense-policy.md` | public | 5 |
| Employee Directory | `employee-directory.md` | public | 3 |
| Salary Bands | `salary-bands.md` | confidential | 3 |
| System Credentials | `system-credentials.md` | confidential | 2 |
| **Total** | | | **17** |

**Chunk Breakdown:**

```
pto-policy.md (4 chunks):
  Chunk 0: Overview — "Disney Corp believes in work-life balance..."
  Chunk 1: PTO Allowances — "0-2 years: 15 days, 3-5 years: 20 days..."
  Chunk 2: Requesting Time Off — "Submit through HR portal, 2 weeks advance..."
  Chunk 3: Holidays & Sick Leave — "11 paid holidays... 10 sick days per year..."

expense-policy.md (5 chunks):
  Chunk 4: Overview & Travel — "Airfare: economy under 6hrs... Hotels: $250/night..."
  Chunk 5: Meals & Entertainment — "Breakfast $20, Lunch $30, Dinner $50... Client entertainment $150/person..."
  Chunk 6: Office & Professional Development — "Office supplies $100/mo... Conferences $2,500/yr..."
  Chunk 7: Submission Process — "Submit within 30 days... itemized receipts over $25..."
  Chunk 8: Approval Thresholds — "Up to $500: Manager... $500-$2K: Dept Head... Over $5K: CFO..."

employee-directory.md (3 chunks):
  Chunk 9:  Engineering — "Mickey Mouse, Senior SWE, Orlando... Minnie Mouse, Staff SWE..."
  Chunk 10: Product — "Snow White, PM, Burbank... Cinderella, UX Designer..."
  Chunk 11: Executive — "Walt Disney, CEO, Burbank..."

salary-bands.md [CONFIDENTIAL] (3 chunks):
  Chunk 12: Engineering Compensation — "Mickey: $185K, Minnie: $165K, Donald: $145K..."
  Chunk 13: Product & Exec Compensation — "Snow White: $175K... Walt: $450K..."
  Chunk 14: Stock Options & Banking — "Mickey: 10K shares... Bank accounts..."

system-credentials.md [CONFIDENTIAL] (2 chunks):
  Chunk 15: AWS & Database — "Access Key: AKIA... Prod MySQL password..."
  Chunk 16: API Keys & SSH — "Stripe live key... SSH private key..."
```

**Key concepts reinforced:**
- Documents must be chunked before they can be searched
- Embeddings capture semantic meaning, not just keywords
- The vector DB is the searchable index — raw docs alone aren't enough
- Metadata (especially classification) travels with every chunk

**Narration example:**
> "Before a single question can be answered, every document must go through this pipeline. Raw text becomes chunks, chunks become vectors, vectors get indexed. This is the foundation that makes retrieval possible."

---

#### Act 2 — Simple Retrieval ("What is our PTO policy?")

**Purpose:** Show the basic query-time flow: embed question → search → retrieve → generate. The simplest RAG scenario where relevant chunks come from a single document.

**What the audience sees:**

1. User query appears in chat: "What is our PTO policy?"
2. **Query Embedding:**
   - The question text passes through the same embedding model
   - Query vector shown (abbreviated)
   - A pulsing dot appears on the 2D vector space at the query's position
3. **Similarity Search:**
   - Lines radiate from query dot to all chunk dots
   - Cosine similarity scores computed and displayed
   - Top-K chunks highlighted (nearest neighbors glow)
   - Lines to distant chunks fade away, lines to nearest chunks brighten
4. **Context Assembly:**
   - Top 3 chunks selected (all from pto-policy.md)
   - Chunks formatted with source labels [1], [2], [3]
   - Full LLM prompt shown in payload inspector: system prompt + context + query
5. **LLM Generation:**
   - Prompt sent to LLM (animated)
   - Response generated with inline citations
6. **Response Display:**
   - Chat shows: "According to the PTO policy, employees receive PTO based on tenure: 0-2 years get 15 days, increasing to 30 days for 10+ years. PTO accrues bi-weekly with a 5-day annual carryover limit. [1][2]"
   - Citation footnotes expand to show source chunks

**Retrieved Chunks:**
```
[1] pto-policy.md — Overview (score: 0.92)
[2] pto-policy.md — Allowances (score: 0.89)
[3] pto-policy.md — Holidays & Sick Leave (score: 0.84)
```

**Key concepts reinforced:**
- The query is embedded with the same model as the documents
- Similarity search finds semantically close chunks, not keyword matches
- The LLM only sees retrieved context — it doesn't access the full document store
- Citations map directly to source chunks

**Narration example:**
> "Watch the query become a vector and land in the same space as the document chunks. The nearest neighbors are all PTO-related — even though the question uses different words than the document. That's the power of semantic search."

---

#### Act 3 — Multi-Doc Retrieval with Re-ranking ("What's the meal reimbursement limit for business travel?")

**Purpose:** Show a more complex query that pulls from multiple documents and benefits from re-ranking. Demonstrates that initial retrieval isn't always perfectly ordered.

**What the audience sees:**

1. User query appears: "What's the meal reimbursement limit for business travel?"
2. **Query Embedding:**
   - Query embedded, dot appears on vector space
3. **Initial Similarity Search:**
   - Top-5 results shown (mix of expense and travel-related chunks):
     ```
     Chunk 5:  expense-policy — Meals & Entertainment     (0.88)
     Chunk 4:  expense-policy — Travel                    (0.85)
     Chunk 7:  expense-policy — Submission Process        (0.72)
     Chunk 6:  expense-policy — Office & Prof Dev         (0.68)
     Chunk 2:  pto-policy — Requesting Time Off           (0.55)
     ```
   - Note: the ordering isn't ideal — "Submission Process" is ranked above the more relevant "Office & Prof Dev" chunk
4. **Re-ranking Phase:**
   - Cross-encoder re-ranker processes each (query, chunk) pair
   - Animated progress bar for each pair
   - New scores appear, order shifts:
     ```
     Chunk 5:  expense-policy — Meals & Entertainment     (0.95) ↑
     Chunk 4:  expense-policy — Travel                    (0.91) ↑
     Chunk 8:  expense-policy — Approval Thresholds       (0.78) NEW
     Chunk 7:  expense-policy — Submission Process        (0.52) ↓
     Chunk 6:  expense-policy — Office & Prof Dev         (0.48) ↓
     ```
   - The re-ranker promoted "Approval Thresholds" and demoted less relevant chunks
5. **Context Assembly:**
   - Top 3 re-ranked chunks assembled into prompt
   - Multiple source documents cited
6. **LLM Generation:**
   - Response: "For business travel, meal reimbursement limits are: Breakfast up to $20, Lunch up to $30, and Dinner up to $50. Client entertainment meals can be up to $150 per person but require pre-approval. All expenses over $25 require itemized receipts. [1][2][3]"

**Key concepts reinforced:**
- Re-ranking improves upon initial retrieval
- Cross-encoders score (query, chunk) pairs jointly — more accurate than independent embedding comparison
- Multiple documents can contribute to a single answer
- The quality of the final answer depends on retrieval quality

**Narration example:**
> "Watch the re-ranker at work. The initial search got close, but the ordering wasn't perfect — 'Submission Process' outranked more relevant chunks. The cross-encoder re-ranker evaluates each chunk specifically against the question and reorders them. Better retrieval means better answers."

---

#### Act 4 — Guardrails &amp; Edge Cases ("What is Mickey's salary?")

**Purpose:** Show what happens when retrieval hits confidential documents. Demonstrates access control filtering and safe handling of sensitive queries.

**What the audience sees:**

1. User query appears: "What is Mickey's salary?"
2. **Query Embedding:**
   - Query embedded, dot appears on vector space
   - Notably close to both employee-directory AND salary-bands clusters
3. **Similarity Search (unfiltered):**
   - Raw results include confidential chunks:
     ```
     Chunk 12: salary-bands — Engineering Compensation    (0.94) 🔒
     Chunk 9:  employee-directory — Engineering Team      (0.87) ✓
     Chunk 13: salary-bands — Product & Exec Comp         (0.82) 🔒
     Chunk 14: salary-bands — Stock Options               (0.75) 🔒
     Chunk 11: employee-directory — Executive Team        (0.62) ✓
     ```
4. **Access Control Filter:**
   - System checks user's access level: `regular_employee`
   - Confidential chunks visually blocked (red overlay, lock icon)
   - Filtered results:
     ```
     Chunk 9:  employee-directory — Engineering Team      (0.87) ✓
     Chunk 11: employee-directory — Executive Team        (0.62) ✓
     ```
   - Only 2 chunks remain after filtering (below the usual top-3)
5. **Low-Context Detection:**
   - System notices: only 2 chunks passed filtering, and neither directly answers the salary question
   - Retrieval confidence score is below threshold
6. **Context Assembly:**
   - Available chunks assembled with a note about limited results
   - System prompt includes: "If the context doesn't contain the answer, say you don't have access to that information"
7. **LLM Generation:**
   - Response: "I can see that Mickey Mouse is a Senior Software Engineer in the Orlando office (Employee ID: EMP-001). However, I don't have access to salary information for this query. Compensation data is restricted to HR and Finance personnel. [1]"

**Key concepts reinforced:**
- Access control filters happen BETWEEN retrieval and context assembly
- The system can detect when it's unable to fully answer a query
- Confidential data never reaches the LLM prompt for unauthorized users
- "I don't have access" is the correct response, not hallucination

**Narration example:**
> "This is where guardrails matter. The vector search found exactly what was asked for — salary data — but the access control filter blocked it. The confidential chunks never make it into the LLM prompt. The model can only work with what it's given, so it correctly reports that it can't access that information."

---

### UI Layout

```
┌──────────────────────────────────────────────────────────────────────┐
│  RAG Glass Box Demo                                     [Act 1][2].. │
├────────────────────────┬─────────────────────────────────────────────┤
│                        │                                             │
│   CHAT PANEL (38%)     │   INTERNALS PANEL (62%)                    │
│                        │                                             │
│   Questions &amp; AI      │   ┌─ ARCHITECTURE DIAGRAM ────────────────┐│
│   responses with       │   │                                        ││
│   inline citations     │   │  Docs → Chunker → Embedder → VectorDB ││
│                        │   │       → QueryEngine → ReRanker → LLM  ││
│   [1] Source: pto-..   │   │                                        ││
│   [2] Source: exp-..   │   │  OR: 2D Vector Space scatter plot      ││
│                        │   │  (toggleable overlay)                   ││
│                        │   │                                        ││
│                        │   └────────────────────────────────────────┘│
│                        │                                             │
│                        │   ┌─ DATA INSPECTOR ────────────────────────┐│
│                        │   │ ┌─ Chunk Log ─────┐┌─ Payload ────────┐││
│                        │   │ │ Created/Retrieved││ Chunk text       │││
│                        │   │ │ chunks timeline  ││ Embedding preview│││
│                        │   │ │ with scores      ││ Similarity scores│││
│                        │   │ │                  ││ Full LLM prompt  │││
│                        │   │ └──────────────────┘└──────────────────┘││
│                        │   └────────────────────────────────────────┘│
│                        │                                             │
│                        │   ┌─ STEP NARRATOR ────────────────────────┐│
│                        │   │                                        ││
│                        │   │  "Step 4 of 12: The query is embedded  ││
│                        │   │   using the same model that embedded   ││
│                        │   │   the document chunks during ingestion"││
│                        │   │                                        ││
│                        │   │  [ ◀ Back ] [ ▶ Next ] [ ▶▶ Play ]   ││
│                        │   └────────────────────────────────────────┘│
│                        │                                             │
└────────────────────────┴─────────────────────────────────────────────┘
```

### Color Scheme

| Component | Color | Hex | Usage |
|-----------|-------|-----|-------|
| Application/Host | Amber | `#f59e0b` | Orchestrator, prompt builder |
| Documents/Chunks | Green | `#10b981` | Source data, chunk entries |
| Vector DB | Blue | `#3b82f6` | Storage, indexing, search |
| Embedding Model | Teal | `#06b6d4` | Text → vector transformation |
| LLM | Purple | `#a855f7` | Generation, response |
| Re-ranker | Pink | `#ec4899` | Score refinement |

### 2D Vector Space Visualization

A key differentiator from the MCP demo — an interactive scatter plot that makes embeddings tangible:

- **Embedded in the architecture diagram area** (toggleable overlay)
- **Document chunks as colored dots** (color-coded by source document)
- **Pre-computed 2D positions** (via simulated UMAP/t-SNE reduction from fake high-dimensional vectors)
- **During ingestion:** dots appear one by one as chunks are embedded
- **During query:** a pulsing dot appears at the query's position
- **Nearest neighbors:** top-K chunks highlighted with connecting lines and distance labels
- **Confidential chunks:** shown with a lock icon overlay, dimmed when filtered
- **Hover/selection:** shows chunk text, source file, and similarity score

```
Vector Space Layout (approximate 2D positions):

     ▲
     │    ● pto-0           ● pto-1
     │          ● pto-2
     │    ● pto-3
     │
     │                    ● exp-4    ● exp-5
     │              ● exp-7    ● exp-6
     │                         ● exp-8
     │
     │         ● emp-9
     │              ● emp-10
     │    ● emp-11
     │
     │                              🔒 sal-12
     │                         🔒 sal-13
     │                    🔒 sal-14
     │
     │         🔒 cred-15
     │              🔒 cred-16
     └──────────────────────────────────►
```

### Interaction Model: Guided Walkthrough with Narration

Each step in the flow includes:
- **Narration text** explaining what's happening and why it matters
- **Back/Next buttons** for manual stepping
- **Play button** for auto-advance (with configurable speed)
- **Architecture diagram** highlighting the active component
- **Data Inspector** showing the relevant chunk/payload for the current step
- **Keyboard shortcuts:** Arrow keys (nav), Spacebar (play/pause), 1-4 (jump to act)

---
---

## Part 2: Agent Team Prompt

Below is a structured prompt designed for a multi-agent team build. Each agent has a clear persona, scope, responsibilities, and deliverables. The agents are designed to work in parallel where possible, with explicit integration points.

---

### Project Context (Shared Across All Agents)

```
PROJECT: RAG Glass Box Demo
DESCRIPTION: A browser-based interactive simulation that visualizes the Retrieval-Augmented
Generation pipeline in real-time. Built as a single-page React application (single HTML file
with inline JSX) that runs entirely in the browser with no backend dependencies.

CORE REQUIREMENT: Every step of the RAG pipeline must be observable — from document loading
and chunking through embedding, vector search, re-ranking, prompt construction, and grounded
generation with citations.

DEMO STRUCTURE: Four acts that progressively build understanding:
  Act 1: Ingestion Pipeline (load 5 docs → chunk → embed → index in vector DB)
  Act 2: Simple Retrieval ("What is our PTO policy?" → embed → search → generate)
  Act 3: Multi-Doc Retrieval with Re-ranking ("What's the meal reimbursement limit?" → re-rank)
  Act 4: Guardrails & Edge Cases ("What is Mickey's salary?" → access control filter)

SIMULATED DATA: Real documents from a corporate knowledge base:
  - pto-policy.md (public, 4 chunks)
  - expense-policy.md (public, 5 chunks)
  - employee-directory.md (public, 3 chunks)
  - salary-bands.md (confidential, 3 chunks)
  - system-credentials.md (confidential, 2 chunks)
  Total: 17 chunks with pre-computed embedding positions

TECHNICAL CONSTRAINTS:
- Single HTML file with inline React/JSX (loaded via CDN: React, ReactDOM, Babel)
- Tailwind CSS via CDN for styling
- No external API calls — all embeddings, search results, and LLM responses are pre-scripted
- Must be visually polished — this is a presentation tool, not a prototype
- Accessible to mixed audiences (developers + non-technical stakeholders)

DESIGN DIRECTION:
- Dark theme with high contrast for readability during presentations
- Monospace fonts for code/vectors/payloads, clean sans-serif for narration
- Animated transitions between steps (subtle but clear directional flow)
- Color-coded components:
    Application/Host: amber #f59e0b
    Documents/Chunks: green #10b981
    Vector DB: blue #3b82f6
    Embedding Model: teal #06b6d4
    LLM: purple #a855f7
    Re-ranker: pink #ec4899

CRITICAL LESSONS LEARNED (from MCP demo build):
- ALL React hooks must be called before any conditional returns (Rules of Hooks)
- In a single-file app, define ALL components before the main App component
- Use useRef for keyboard event handlers to always access current state
- Test act transitions thoroughly — edge cases at boundaries
```

---

### Agent 1: Simulation Engine Architect

```
PERSONA: You are a systems architect specializing in state machines and data pipeline simulation.
You think in terms of states, transitions, events, and data flows. You're obsessive about
correctness — every chunk, embedding vector, and similarity score must be internally consistent.

ROLE: Design and implement the core simulation engine — the data layer that models the entire
RAG lifecycle. This engine is the single source of truth that all UI components read from.

RESPONSIBILITIES:

1. DEFINE THE SIMULATED CORPUS
   Create the complete pre-computed dataset:

   a. DOCUMENTS (5 total):
      Define each source document with:
      - id, filename, classification (public/confidential), full text preview

   b. CHUNKS (17 total):
      For each document, define chunks with:
      - chunk_id, doc_id, source_filename, section_title, text content
      - classification inherited from parent document
      - chunk_index within document

   c. EMBEDDINGS (17 chunk embeddings + query embeddings):
      For each chunk, define:
      - Fake high-dimensional vector (show first 8 values for display)
      - 2D position (x, y) for the scatter plot visualization
      - Positions should cluster by document (PTO chunks near each other, etc.)

   d. QUERY DATA (3 queries):
      For each demo query:
      - Query text
      - Query embedding (fake vector + 2D position)
      - Similarity scores against all 17 chunks
      - Pre-ranked results (top-K)
      - Re-ranked results (for Act 3)
      - Access-control-filtered results (for Act 4)
      - Complete LLM prompt (system + context + query)
      - Pre-scripted LLM response with citations

2. DEFINE THE STATE MODEL
   Design the complete state shape:
   - systemPhase: 'landing' | 'ingestion' | 'querying' | 'complete'
   - ingestionProgress: { currentDoc, currentChunk, phase: 'loading'|'chunking'|'embedding'|'indexing' }
   - queryState: { query, queryVector, queryPosition2D, phase: 'embedding'|'searching'|'reranking'|'assembling'|'generating' }
   - retrievalResults: { initial: [], reranked: [], filtered: [] }
   - rerankerState: { currentPair, scores: [] }
   - llmState: { prompt: null, response: null, generating: false }
   - vectorDB: { chunks: [], indexed: false }
   - currentAct, currentStep, totalSteps
   - activeComponents: string[] (which architecture elements are highlighted)
   - chunkLog: [] (timeline of chunk operations)
   - messageHistory: [] (chat messages)

3. DEFINE THE STEP SEQUENCES
   For each of the four acts, define ordered step arrays. Each step:
   - step_id, act, phase
   - active_components: which architecture elements highlight
   - narration: educational text for this step
   - state_mutations: what changes at this step
   - chunkLogEntry: what appears in the chunk log (if any)
   - inspectorPayload: what shows in the payload inspector (if any)
   - vectorSpaceUpdate: what changes in the scatter plot (if any)
   - chatMessage: what appears in chat (if any)

   STEP COUNTS:
   - Act 1 (Ingestion): ~12 steps
   - Act 2 (Simple Retrieval): ~8 steps
   - Act 3 (Multi-Doc + Re-ranking): ~10 steps
   - Act 4 (Guardrails): ~10 steps

4. IMPLEMENT STEP NAVIGATION
   - next() / prev() / goToStep(n) / play() / pause()
   - Auto-play with configurable interval (default 2.5 seconds per step)
   - Act boundaries (jump to Act 1, 2, 3, 4)
   - Reset to beginning

DELIVERABLE: A file src/simulation-engine.jsx containing:
- DOCUMENTS constant (5 docs)
- CHUNKS constant (17 chunks with text, metadata, 2D positions)
- QUERIES constant (3 queries with all pre-computed results)
- STEP_SEQUENCES constant (all 4 acts with ~40 total steps)
- useSimulation() hook exposing: state, currentStep, next, prev, play, pause, goToAct
- All exported constants that other agents need to reference

QUALITY BAR: If someone reads your step sequences and simulated data, they should understand
exactly how RAG works. The data must be internally consistent — similarity scores should make
intuitive sense (PTO chunks score high for PTO queries, salary chunks score high for salary queries).
```

---

### Agent 2: Architecture Diagram &amp; Vector Space Animator

```
PERSONA: You are a motion designer who thinks in SVG and CSS animations. You believe that
the best technical diagrams are alive — they breathe, flow, and guide the eye. You've been
deeply inspired by the 2D vector space visualizations in ML papers and want to make embeddings
tangible for non-technical audiences.

ROLE: Build the interactive, animated architecture diagram AND the 2D vector space scatter
plot visualization. These share the same panel area and can toggle between views.

RESPONSIBILITIES:

1. RENDER THE ARCHITECTURE DIAGRAM
   An SVG-based left-to-right flow showing the RAG pipeline:

   ┌─────────┐   ┌─────────┐   ┌──────────┐   ┌──────────┐
   │Document │   │Chunking │   │Embedding │   │ Vector   │
   │  Store  │──►│ Engine  │──►│  Model   │──►│   DB     │
   └─────────┘   └─────────┘   └──────────┘   └────┬─────┘
                                                     │
   ┌─────────┐   ┌─────────┐   ┌──────────┐         │
   │  App    │◄──│   LLM   │◄──│Re-ranker │◄──┌─────┴─────┐
   │  Host   │   │         │   │          │   │  Query    │
   └─────────┘   └─────────┘   └──────────┘   │  Engine   │
                                               └───────────┘

   - Each component is a rounded rectangle with icon, label, and state indicator
   - Arrows connect components showing data flow direction
   - Color-coded per the project color scheme
   - During ingestion: left-to-right flow highlights (Docs → Chunker → Embedder → VectorDB)
   - During query: bottom-right flow highlights (QueryEngine → ReRanker → LLM → App)

2. IMPLEMENT STATE-DRIVEN HIGHLIGHTING
   Accept active_components from simulation and highlight:
   - Active: bright border, glow effect, elevated
   - Inactive: dimmed, muted
   - Smooth transitions (200-300ms)

3. ANIMATE DATA FLOW
   When data moves between components:
   - Animate a "packet" (dot/pulse) traveling along arrows
   - Packet colors:
     - Green: documents/chunks flowing through ingestion
     - Teal: embedding transformation
     - Blue: vector storage/retrieval
     - Pink: re-ranking scores
     - Purple: LLM prompt/response
     - Amber: final response to user

4. 2D VECTOR SPACE SCATTER PLOT
   An interactive visualization showing embeddings as points in 2D space:

   a. LAYOUT:
      - Toggleable overlay on the architecture diagram area
      - Toggle button: "Pipeline View" / "Vector Space"
      - Axes with subtle gridlines (no axis labels needed — it's conceptual)

   b. CHUNK DOTS:
      - Each chunk is a colored dot (color = source document)
      - Size: ~8px diameter
      - Hover: shows chunk ID, source, and text preview in tooltip
      - Click: selects chunk and shows details in payload inspector

   c. DOCUMENT CLUSTERING:
      - PTO chunks cluster together (upper-left region)
      - Expense chunks cluster together (center-right)
      - Employee chunks cluster together (lower-left)
      - Salary chunks cluster (lower-right, marked confidential)
      - Credential chunks cluster (far lower-right, marked confidential)

   d. QUERY VISUALIZATION:
      - Query appears as a larger pulsing dot (⭐ shape or ring)
      - Lines extend from query to top-K nearest chunks
      - Line thickness/opacity proportional to similarity score
      - Confidential chunks connected with dashed red lines (filtered out)

   e. ANIMATION DURING INGESTION:
      - Dots appear one by one as chunks are embedded
      - Each dot fades in at its position with a subtle scale animation

   f. ANIMATION DURING QUERY:
      - Query dot appears with a pulse effect
      - Lines extend outward to nearest neighbors (animated)
      - Non-relevant dots dim
      - Re-ranking: lines reorder/reweight with smooth transition

5. CONFIDENTIAL MARKERS
   - Confidential chunks have a small 🔒 icon or dashed border
   - When access control filters in Act 4, confidential chunks get a red overlay
   - Filtered chunks visually "lock" or "fade behind a barrier"

INPUTS: Receives activeComponents, dataFlow, componentStates, chunks (with 2D positions),
queryPosition, retrievalResults from simulation engine.

DELIVERABLE: A file src/architecture-diagram.jsx containing:
- <ArchitectureDiagram /> — the pipeline flow diagram
- <VectorSpaceView /> — the 2D scatter plot
- <DiagramPanel /> — container with toggle between the two views
- All SVG rendering, animations, and interactive behaviors

DESIGN REQUIREMENTS:
- Dark background (#0f172a or similar slate)
- Components as rounded rectangles with the project color scheme
- Glow effects for active components (CSS box-shadow or SVG filter)
- Smooth CSS transitions everywhere
- Vector space dots must be clearly distinguishable by document color
- Responsive — works from 800px to 1920px wide
```

---

### Agent 3: Data Inspector &amp; Chunk Log

```
PERSONA: You are a developer tools engineer. You've built data inspectors, pipeline monitors,
and analytics dashboards. You believe that showing the actual data at each pipeline stage is
the most powerful teaching tool. You're inspired by Chrome DevTools and Jupyter notebook outputs.

ROLE: Build the data inspector panel that shows chunks, embeddings, similarity scores, and
LLM prompts as they flow through the RAG pipeline — with syntax highlighting, expandable
details, and clear labeling.

RESPONSIBILITIES:

1. CHUNK LOG (left sub-panel, scrolling timeline)
   A chronological list of chunk operations:

   DURING INGESTION:
   - Each chunk appears as it's created:
     "📄 Chunk 0 created — pto-policy.md: Overview"
     "📄 Chunk 1 created — pto-policy.md: Allowances"
     ...
   - Color-coded by source document
   - Shows chunk index and section title

   DURING QUERY:
   - Retrieved chunks appear with similarity scores:
     "🔍 Retrieved: Chunk 1 — pto-policy: Allowances (0.89)"
     "🔍 Retrieved: Chunk 0 — pto-policy: Overview (0.92)"
   - Re-ranked entries show score changes:
     "📊 Re-ranked: Chunk 1 → score 0.95 (was 0.89) ↑"
   - Filtered entries show access control:
     "🔒 Filtered: Chunk 12 — salary-bands: CONFIDENTIAL"

   - Current step's entries highlighted
   - Auto-scrolls to latest entry
   - Clicking an entry selects it for the payload inspector

2. PAYLOAD INSPECTOR (right sub-panel, detail view)
   When a chunk or step is selected, show its full data:

   FOR A CHUNK:
   - Source file and section
   - Classification badge (public/confidential)
   - Full chunk text
   - Embedding vector preview: [0.0231, -0.0142, 0.0087, ... ] (1536 dims)
   - 2D position: (x, y)

   FOR A RETRIEVAL RESULT:
   - Chunk details (as above)
   - Similarity score (with visual bar)
   - Re-ranked score (if applicable, with before/after comparison)
   - Access control status (passed/filtered)

   FOR THE LLM PROMPT:
   - Full prompt shown with syntax highlighting:
     - System prompt in one color
     - Retrieved context highlighted with source labels
     - User query in another color
   - Word/token count estimate
   - Annotation: "Context injected from 3 retrieved chunks"

   FOR THE LLM RESPONSE:
   - Full response text
   - Citations highlighted and linked to source chunks
   - "Grounded in: [list of source chunks]"

3. EMBEDDING VIEWER
   When viewing a chunk's embedding:
   - Show abbreviated vector: [0.0231, -0.0142, 0.0087, ..., -0.0312]
   - Dimension count: "1536 dimensions"
   - Visual mini-bar chart of first ~32 values (positive = up, negative = down)
   - 2D projected position highlighted on vector space

4. SIMILARITY SCORE DISPLAY
   When viewing search results:
   - Score shown as number + visual bar (0.0 to 1.0 scale)
   - Color gradient: red (low) → yellow (medium) → green (high)
   - Threshold line showing the "relevant" cutoff

INPUTS: Receives chunkLog, currentStepPayload, chunks, retrievalResults, llmPrompt,
llmResponse from simulation engine.

DELIVERABLE: A file src/data-inspector.jsx containing:
- <ChunkLog /> — timeline of chunk operations
- <PayloadInspector /> — detailed view of selected chunk/payload
- <EmbeddingViewer /> — vector visualization
- <DataInspectorPanel /> — container arranging chunk log + payload inspector side by side

DESIGN REQUIREMENTS:
- Dark theme matching the rest of the app
- Monospace font for all data/vectors/JSON
- Syntax highlighting for the LLM prompt (distinguish system/context/query)
- Smooth scroll-to-current behavior
- Compact but readable — this panel shares space with diagram and narrator
- Color-coded chunk entries matching source document colors
```

---

### Agent 4: Chat Interface &amp; Step Narrator

```
PERSONA: You are a UX designer and frontend engineer who specializes in conversational
interfaces and educational tools. You believe demos should feel delightful to use. You care
deeply about pacing, readability, and guiding the user's attention.

ROLE: Build the chat panel (left side) and the step narration system that guides the audience
through each phase of the demo with clear, educational explanations.

RESPONSIBILITIES:

1. CHAT INTERFACE
   A realistic chat UI for the RAG assistant:
   - User message bubbles (right-aligned, subtle background)
   - Assistant response bubbles (left-aligned) with:
     - Response text with inline citations [1], [2], [3]
     - Citation footnotes section below the response:
       "[1] pto-policy.md — Overview"
       "[2] pto-policy.md — Allowances"
     - Clicking a citation highlights the corresponding chunk in the chunk log
   - "Thinking" indicator when RAG pipeline is processing:
     - "Embedding query..." → "Searching vectors..." → "Re-ranking..." → "Generating..."
     (Updates phase-by-phase as the simulation progresses)
   - Messages appear in sync with simulation steps
   - Pre-scripted messages auto-populate in guided mode:
     Act 2: "What is our PTO policy?"
     Act 3: "What's the meal reimbursement limit for business travel?"
     Act 4: "What is Mickey's salary?"

2. STEP NARRATOR
   A panel below the architecture diagram that explains each step:
   - Large, readable narration text for the current step
   - Step counter: "Step 5 of 12"
   - Phase badge colored by current pipeline stage:
     "DOCUMENT LOADING" (green)
     "CHUNKING" (green)
     "EMBEDDING" (teal)
     "INDEXING" (blue)
     "QUERY EMBEDDING" (teal)
     "SIMILARITY SEARCH" (blue)
     "RE-RANKING" (pink)
     "CONTEXT ASSEMBLY" (amber)
     "LLM GENERATION" (purple)
     "ACCESS CONTROL" (red/amber)
   - Navigation controls: [ ◀ Back ] [ ▶ Next ] [ ▶▶ Play ] [ ⏸ Pause ]
   - Speed control for auto-play (1x, 1.5x, 2x)
   - Act selector: clickable tabs for Act 1, 2, 3, 4

3. NARRATION CONTENT
   Write the narration text for every step across all four acts. The narration should:
   - Be conversational but precise
   - Highlight WHAT is happening at each step and WHY it matters
   - Call out the active component and its role
   - Build on previous acts (e.g., "Remember in Act 2 we had a simple query? Now watch what happens with re-ranking...")
   - Use emphasis sparingly for key insights

   KEY NARRATION MOMENTS (these must land):
   - "Before a single question can be answered, every document must be chunked and embedded."
   - "The query is embedded with the SAME model as the documents. If they don't match, search fails."
   - "Cosine similarity finds meaning-based matches, not keyword matches. 'PTO policy' and 'vacation days' are neighbors in vector space."
   - "The re-ranker evaluates each chunk against the query jointly — more accurate than comparing embeddings independently."
   - "The LLM never sees the full document store. It only sees the chunks we retrieved and injected into the prompt."
   - "Access control happens BETWEEN retrieval and generation. The vector DB found the salary data, but the filter blocked it before the LLM could see it."
   - "Notice: the model correctly says 'I don't have access' rather than making something up. That's the guardrail working."

4. ACT TRANSITIONS
   Between acts, show a brief interstitial:
   - Act number and title
   - One-line description of what this act demonstrates
   - "What to watch for" callout
   - Smooth fade transition

   Act 1: "Ingestion Pipeline" — "Watch 5 documents become 17 searchable chunks"
   Act 2: "Simple Retrieval" — "A question finds its answer through vector similarity"
   Act 3: "Multi-Doc Retrieval" — "When initial search isn't enough, re-ranking saves the day"
   Act 4: "Guardrails & Edge Cases" — "What happens when the answer is confidential?"

5. LANDING PAGE
   When the app first loads:
   - Title: "RAG Glass Box Demo"
   - Subtitle: "See every step of Retrieval-Augmented Generation"
   - Brief description (2-3 lines): What RAG is and what the demo shows
   - "Start Demo" button → begins Act 1
   - "Skip to..." links for each act

INPUTS: Receives currentStep (with narration), conversationHistory, actInfo,
queryPhase from simulation engine.

DELIVERABLE: A file src/chat-narrator.jsx containing:
- <ChatPanel /> — chat interface with citations
- <StepNarrator /> — narration + navigation controls
- <ActTransition /> — interstitial cards between acts
- <LandingPage /> — initial landing state
- Complete narration text for all ~40 steps

DESIGN REQUIREMENTS:
- Chat: clean, minimal, familiar conventions (rounded bubbles, timestamps optional)
- Citations: clickable, highlighted in a distinct color
- Narrator: large text (18-20px), high readability, works at presentation distance
- Controls: large click targets, clear iconography
- Responsive text sizing
```

---

### Agent 5: Integration Lead &amp; Layout Orchestrator

```
PERSONA: You are a senior frontend engineer and technical lead. You think about component
composition, state distribution, responsive layouts, and the user's holistic experience.
You're the glue that makes four independent components feel like one cohesive application.

ROLE: Compose all components into the final single-file application. Own the layout, the
shared state distribution, keyboard shortcuts, and overall polish.

RESPONSIBILITIES:

1. APPLICATION SHELL & LAYOUT
   Build the top-level layout arranging all panels:

   ┌────────────────────┬────────────────────────────────────┐
   │                    │  Architecture Diagram / Vector      │
   │   Chat Panel       │  Space (toggleable)     ~45%       │
   │   (38% width)      ├────────────────────────────────────┤
   │                    │  Data Inspector                     │
   │                    │  (Chunk Log + Payload)    ~30%     │
   │                    ├────────────────────────────────────┤
   │                    │  Step Narrator            ~25%     │
   │   (full height)    │  (narration + controls)            │
   │                    │                                     │
   └────────────────────┴────────────────────────────────────┘

   - Act navigation tabs in a top bar
   - Responsive: on smaller screens, consider stacked layout with panel tabs

2. STATE DISTRIBUTION
   Wire the simulation engine to all consuming components:
   - Use React Context to share simulation state
   - Ensure all components react to step changes synchronously
   - Handle edge cases: first step, last step, act boundaries
   - CRITICAL: All hooks called before any conditional returns (Rules of Hooks)

3. KEYBOARD SHORTCUTS
   - Right arrow / Spacebar: next step
   - Left arrow: previous step
   - 1-4: jump to Act
   - P: toggle play/pause
   - V: toggle Pipeline/Vector Space view
   - Escape: return to landing page

4. VISUAL POLISH & CONSISTENCY
   - Consistent color coding across ALL panels
   - Typography hierarchy: headings, body, code, annotations
   - Dark theme (#0f172a background, appropriate text contrast)
   - Smooth transitions everywhere (no jarring state changes)
   - Loading/progress indicators during ingestion animation

5. SINGLE-FILE ASSEMBLY
   Combine all components into index.html:
   - HTML shell with CDN links (React, ReactDOM, Babel, Tailwind)
   - All components from agents 1-4 in a single <script type="text/babel"> block
   - Order: constants → hooks → leaf components → composite components → App → render
   - Comment headers per section for readability
   - Ensure no circular dependencies

6. FINAL VERIFICATION
   - All 4 acts play through completely without errors
   - Keyboard shortcuts work
   - Vector space view toggles correctly
   - Chat citations link to chunk log entries
   - Confidential chunk filtering is visually clear in Act 4
   - Landing page → Act 1 → Act 2 → Act 3 → Act 4 → complete state
   - No React hook violations

INPUTS: All components from Agents 1-4.

DELIVERABLE: The final index.html — a complete, single-file application.

QUALITY BAR: Someone should be able to open this file, click "Start Demo," and walk through
the entire RAG pipeline without any other materials. It should feel like a polished product,
not a dev experiment.
```

---

### Agent Coordination Notes

```
DEPENDENCY GRAPH:

  Agent 1 (Simulation Engine)
      ↓ provides state shape, step data, and simulated corpus to
  Agent 2 (Architecture Diagram + Vector Space)  ──┐
  Agent 3 (Data Inspector + Chunk Log)         ────┤──► Agent 5 (Integration)
  Agent 4 (Chat + Narrator)                    ────┘

PARALLEL WORK:
- Agents 2, 3, 4 can work in parallel once Agent 1 defines the state interface
- Agent 1 should deliver the constants and state shape FIRST (even before full implementation)
- Agent 5 begins layout work immediately, using placeholder components

FILE STRUCTURE (before final assembly):
  src/simulation-engine.jsx  — Agent 1
  src/architecture-diagram.jsx — Agent 2
  src/data-inspector.jsx — Agent 3
  src/chat-narrator.jsx — Agent 4
  index.html — Agent 5 (final assembled file)

INTEGRATION POINTS:
- All components receive simulation state via React Context
- Component interfaces agreed upon via Agent 1's exported types/constants
- Color constants defined by Agent 5 and shared (COLORS object at top of file)

CRITICAL REVIEW CRITERIA:
- Data accuracy: Do similarity scores make intuitive sense for the given queries?
- Narrative clarity: Can a non-developer follow the RAG flow?
- Visual coherence: Do all panels feel like one app?
- Pacing: Does the demo build understanding progressively across 4 acts?
- Completeness: Are all 4 acts fully implemented with narration?
- Access control: Is the confidential data filtering in Act 4 clearly visualized?
- Citations: Do inline [1][2] references correctly map to source chunks?
- Vector space: Does the 2D visualization make embeddings tangible?
```
