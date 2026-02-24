# RAG Architecture Deep Dive

A comprehensive guide to how Retrieval-Augmented Generation works — from the core insight down to the exact data flow between components at each stage of the pipeline.

---

## 1. What is RAG?

Large Language Models have a fundamental limitation: **their knowledge is frozen at training time.** They can't access your company's internal documents, today's data, or any information created after their training cutoff. They also can't cite their sources — they generate from parametric memory, which means they can sound confident while being wrong.

**Retrieval-Augmented Generation (RAG)** solves this by giving the model access to external data at query time. Instead of relying solely on what the model "memorized" during training, RAG retrieves relevant documents from a knowledge base and injects them into the prompt as context. The model then generates a response grounded in that retrieved evidence.

```
┌─────────────────────────────────────────────────────────────┐
│                     WITHOUT RAG                              │
│                                                              │
│  User: "What is our PTO policy?"                            │
│           │                                                  │
│           ▼                                                  │
│       ┌────────┐                                            │
│       │  LLM   │  ← Only knows what it learned in training  │
│       └────┬───┘                                            │
│            │                                                 │
│            ▼                                                 │
│  "I don't have access to your specific company policies..."  │
│  (or worse: confidently makes something up)                  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      WITH RAG                                │
│                                                              │
│  User: "What is our PTO policy?"                            │
│           │                                                  │
│           ▼                                                  │
│       ┌──────────┐    ┌──────────────┐                      │
│       │ Retrieve │───►│  Vector DB   │                      │
│       │  Engine  │◄───│ (your docs)  │                      │
│       └────┬─────┘    └──────────────┘                      │
│            │ relevant chunks                                 │
│            ▼                                                  │
│       ┌────────┐                                            │
│       │  LLM   │  ← Now sees your actual PTO policy text    │
│       └────┬───┘                                            │
│            │                                                 │
│            ▼                                                  │
│  "According to the PTO policy, employees with 0-2 years     │
│   receive 15 days annually..." [1]                          │
└─────────────────────────────────────────────────────────────┘
```

**The core insight:** RAG doesn't change the model. It changes what the model can *see* at the moment it generates a response.

---

## 2. The Two Phases

RAG operates in two distinct phases that happen at different times:

```
PHASE 1: INGESTION (Offline — happens once, or periodically)
═══════════════════════════════════════════════════════════

  Raw Documents → Chunking → Embedding → Vector DB

  "Prepare the knowledge base so it's searchable"


PHASE 2: RETRIEVAL + GENERATION (Online — happens per query)
═══════════════════════════════════════════════════════════

  User Query → Embed Query → Search → Re-rank → Assemble Prompt → LLM → Response

  "Find relevant context and generate a grounded answer"
```

| Aspect | Ingestion Phase | Query Phase |
|--------|----------------|-------------|
| **When** | Before any queries (batch/offline) | Per user question (real-time) |
| **Latency** | Minutes to hours (doesn't matter) | Milliseconds to seconds (critical) |
| **Frequency** | Once per document update | Every user query |
| **Cost** | Embedding API calls (batch) | Embedding + LLM API calls (per query) |
| **Output** | Indexed vector database | Grounded text response |

**Key insight:** You pay the ingestion cost once. Every subsequent query just does a fast vector search and one LLM call.

---

## 3. The Ingestion Pipeline

This phase transforms raw documents into a searchable vector index. Here's every step:

```
┌──────────┐    ┌──────────┐    ┌───────────────┐    ┌───────────┐
│  Raw     │    │          │    │   Embedding   │    │           │
│  Docs    │───►│ Chunker  │───►│    Model      │───►│ Vector DB │
│          │    │          │    │               │    │           │
│ .md .pdf │    │ Split    │    │ text → vector │    │ Store +   │
│ .txt etc │    │ into     │    │ (e.g. 1536d)  │    │ index     │
│          │    │ segments │    │               │    │ vectors   │
└──────────┘    └──────────┘    └───────────────┘    └───────────┘
```

### Step 1: Document Loading

Raw files are loaded and converted to plain text. Different file types need different parsers:

- **Markdown (.md)** → Strip formatting, preserve structure
- **PDF (.pdf)** → Extract text via PDF parser (OCR if scanned)
- **HTML (.html)** → Strip tags, preserve content
- **CSV/structured data** → Convert rows to natural language statements

Each loaded document carries **metadata** — at minimum, the source filename and any classification labels (e.g., "confidential", "public").

### Step 2: Chunking

Documents are split into smaller segments called **chunks**. This is one of the most important decisions in the entire pipeline, because chunk size directly affects retrieval quality.

**Why chunk at all?** Two reasons:
1. Documents are often too long to fit in a single embedding (and even if they fit, the embedding quality degrades for very long texts)
2. Retrieval should return *relevant passages*, not entire documents — precision matters

**Common chunking strategies:**

| Strategy | How It Works | Best For |
|----------|-------------|----------|
| **Fixed-size** | Split every N characters/tokens with overlap | Simple, predictable |
| **Recursive** | Split by paragraphs → sentences → words, respecting boundaries | Structured documents |
| **Semantic** | Use embeddings to find natural topic boundaries | Long, flowing text |
| **Document-aware** | Split on headers, sections, or logical boundaries | Markdown, HTML, code |

```
EXAMPLE: Chunking a policy document (recursive, ~300 tokens per chunk)

Original: "# PTO Policy\n\n## Overview\nDisney Corp believes in work-life
balance...\n\n## PTO Allowances\n\n### By Tenure\n| Years | Days |..."

Chunk 1: "PTO Policy — Overview: Disney Corp believes in work-life balance
          and provides generous paid time off to all employees."
          [metadata: source=pto-policy.md, section=Overview, chunk_index=0]

Chunk 2: "PTO Allowances by Tenure: 0-2 years: 15 days, 3-5 years: 20 days,
          6-10 years: 25 days, 10+ years: 30 days. PTO accrues bi-weekly.
          Maximum carryover: 5 days per year."
          [metadata: source=pto-policy.md, section=Allowances, chunk_index=1]

Chunk 3: "Requesting Time Off: Submit requests through the HR portal at least
          2 weeks in advance. Manager approval required for requests over 3
          consecutive days. Blackout periods may apply."
          [metadata: source=pto-policy.md, section=Requesting, chunk_index=2]

Chunk 4: "Holidays and Sick Leave: Disney Corp observes 11 paid holidays...
          Separate sick leave: 10 days per year, no carryover."
          [metadata: source=pto-policy.md, section=Holidays, chunk_index=3]
```

**Critical parameter: overlap.** Chunks typically overlap by 10-20% to avoid cutting off context at boundaries. If a relevant sentence spans two chunks, the overlap ensures at least one chunk contains it completely.

### Step 3: Embedding

Each chunk is transformed into a **dense vector** — a list of floating-point numbers that encodes the chunk's semantic meaning. This is done by an **embedding model** (like `text-embedding-3-small` or similar).

```
Input text:  "PTO accrues bi-weekly. Maximum carryover: 5 days per year."
                                    │
                                    ▼
                          ┌─────────────────┐
                          │ Embedding Model  │
                          │                  │
                          │ text-embedding-  │
                          │ 3-small (1536d)  │
                          └────────┬─────────┘
                                   │
                                   ▼
Output vector: [0.0231, -0.0142, 0.0087, ..., -0.0312]  (1536 numbers)
```

**Why this works:** The embedding model has been trained so that texts with similar *meanings* produce vectors that are close together in the vector space, regardless of the exact words used. "PTO policy" and "vacation days allowance" would produce nearby vectors even though they share few words.

### Step 4: Indexing

Vectors are stored in a **vector database** (like Pinecone, Weaviate, ChromaDB, or pgvector) along with:
- The original chunk text
- The metadata (source file, section, classification, etc.)
- The vector itself

```
Vector DB Record:
{
  "id": "pto-policy-chunk-1",
  "vector": [0.0231, -0.0142, 0.0087, ...],    // 1536 dimensions
  "text": "PTO Allowances by Tenure: 0-2 years: 15 days...",
  "metadata": {
    "source": "pto-policy.md",
    "section": "Allowances",
    "classification": "public",
    "chunk_index": 1,
    "ingested_at": "2024-01-15T10:30:00Z"
  }
}
```

The vector database builds an **index** over the vectors to enable fast similarity search. Instead of comparing a query against every single vector (brute force), indexing structures like **HNSW** (Hierarchical Navigable Small World) or **IVF** (Inverted File Index) allow approximate nearest-neighbor search in milliseconds, even over millions of vectors.

---

## 4. The Query-Time Pipeline

This is where RAG earns its name. When a user asks a question, here's exactly what happens:

```
USER: "What is our PTO policy?"
  │
  ▼
┌─────────────────────────────────────────────────────────────────┐
│                      RAG QUERY PIPELINE                          │
│                                                                   │
│  1. EMBED THE QUERY                                               │
│     "What is our PTO policy?" → [0.0198, -0.0156, 0.0091, ...]  │
│     (Same embedding model used during ingestion!)                 │
│                    │                                              │
│                    ▼                                              │
│  2. SIMILARITY SEARCH                                             │
│     Compare query vector against all indexed chunk vectors        │
│     Return top-K most similar chunks                              │
│                    │                                              │
│     Results:                                                      │
│     ┌─────────────────────────────────────────┬─────────┐        │
│     │ Chunk                                    │ Score   │        │
│     ├─────────────────────────────────────────┼─────────┤        │
│     │ pto-policy: Overview                     │ 0.92    │        │
│     │ pto-policy: Allowances by Tenure         │ 0.89    │        │
│     │ pto-policy: Holidays and Sick Leave      │ 0.84    │        │
│     │ expense-policy: Submission Process       │ 0.61    │        │
│     │ employee-directory: Engineering Dept     │ 0.43    │        │
│     └─────────────────────────────────────────┴─────────┘        │
│                    │                                              │
│                    ▼                                              │
│  3. RE-RANK (optional but recommended)                            │
│     A cross-encoder re-ranker scores (query, chunk) pairs         │
│     more precisely than embedding similarity alone                │
│                    │                                              │
│                    ▼                                              │
│  4. CONTEXT ASSEMBLY                                              │
│     Take top-K chunks (after re-ranking) and format them          │
│     into a context block for the LLM prompt                       │
│                    │                                              │
│                    ▼                                              │
│  5. LLM GENERATION                                                │
│     Send: system prompt + retrieved context + user query → LLM    │
│     Receive: grounded response with citations                     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
  │
  ▼
RESPONSE: "According to the PTO policy, new employees (0-2 years)
receive 15 days of PTO annually, increasing to 30 days for those
with 10+ years of tenure. PTO accrues bi-weekly with a maximum
carryover of 5 days per year. [1][2]"
```

### Step 1: Query Embedding

The user's question is embedded using **the exact same model** that was used during ingestion. This is critical — if you use a different model, the vectors won't be in the same space, and similarity search won't work.

```
Ingestion model:  text-embedding-3-small  ← must match
Query model:      text-embedding-3-small  ← must match
```

### Step 2: Similarity Search

The query vector is compared against all chunk vectors in the database. The most common similarity metric is **cosine similarity** (explained in Section 5). The database returns the top-K nearest neighbors — typically K=3 to K=10.

### Step 3: Re-ranking

Initial retrieval uses **bi-encoder** similarity — the query and chunks are embedded independently, then compared. This is fast but can miss nuance.

A **cross-encoder re-ranker** takes each (query, chunk) pair and scores them jointly, producing more accurate relevance scores. This is slower but more precise — it can catch cases where a chunk is semantically related but not actually answering the question.

```
Before re-ranking:                    After re-ranking:
1. pto-policy: Overview     (0.92)    1. pto-policy: Allowances   (0.95)
2. pto-policy: Allowances   (0.89)    2. pto-policy: Overview     (0.88)
3. pto-policy: Holidays     (0.84)    3. pto-policy: Holidays     (0.82)
```

In this example, the re-ranker promoted the "Allowances" chunk because it more directly answers the question about what the PTO policy actually provides.

### Step 4: Context Assembly

The top-K chunks (after re-ranking) are formatted into a context block that will be injected into the LLM prompt. Each chunk is labeled with its source for citation purposes.

### Step 5: LLM Generation

The assembled prompt is sent to the LLM. The model generates a response grounded in the retrieved context, with inline citations referencing the source chunks.

---

## 5. Vector Similarity Explained

This is the mathematical core of RAG. If you understand this section, you understand why RAG works.

### Embeddings: Text as Points in Space

An embedding model maps text to a point in high-dimensional space (typically 768 to 3072 dimensions). Texts with similar meanings land near each other:

```
Imagine a simplified 2D embedding space:

                    ▲ Dimension 2
                    │
         "vacation  │  "PTO policy" •
          days"  •  │         • "time off allowance"
                    │
                    │    • "sick leave"
                    │
                    │                    • "expense report"
                    │           • "reimbursement process"
  ──────────────────┼──────────────────────────► Dimension 1
                    │
          • "salary │
           bands"   │
                    │    • "compensation data"
                    │
```

In reality, these spaces have 1536+ dimensions, but the principle is the same: semantic similarity = spatial proximity.

### Cosine Similarity

The standard similarity metric for embeddings. It measures the angle between two vectors, ignoring magnitude:

```
                    ▲
                   /│
                  / │
            B → /  │  cos(θ) = (A · B) / (|A| × |B|)
               /   │
              / θ  │
             /─────┼──────► A

  cos(0°)  = 1.0   (identical direction — maximum similarity)
  cos(90°) = 0.0   (perpendicular — no similarity)
  cos(180°) = -1.0 (opposite — maximum dissimilarity)
```

In practice, embedding similarity scores typically range from 0.3 (unrelated) to 0.95+ (near-identical meaning). A score above 0.8 usually indicates strong semantic relevance.

### Why Embeddings Work for Search

Traditional keyword search fails when words don't match:
- Query: "How many vacation days do I get?"
- Document: "PTO Allowances by Tenure: 0-2 years: 15 days"
- **Keyword search: no match** ("vacation" ≠ "PTO", "get" ≠ "Allowances")
- **Vector search: strong match** (both are about time-off entitlements)

The embedding model learned during its training that "vacation days," "PTO," "time off," and "paid leave" are semantically equivalent. The vectors encode meaning, not just words.

---

## 6. The Prompt Construction

This is the step that connects retrieval to generation. The host application constructs a prompt that contains three parts:

```json
{
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful HR assistant. Answer questions using ONLY the provided context. If the context doesn't contain the answer, say so. Always cite your sources using [1], [2], etc."
    },
    {
      "role": "user",
      "content": "Context from knowledge base:\n\n[1] Source: pto-policy.md (Section: Overview)\nDisney Corp believes in work-life balance and provides generous paid time off to all employees.\n\n[2] Source: pto-policy.md (Section: Allowances)\nPTO Allowances by Tenure: 0-2 years: 15 days, 3-5 years: 20 days, 6-10 years: 25 days, 10+ years: 30 days. PTO accrues bi-weekly. Maximum carryover: 5 days per year.\n\n[3] Source: pto-policy.md (Section: Holidays)\nDisney Corp observes 11 paid holidays including New Year's Day, Memorial Day, Independence Day, Thanksgiving, Christmas, and more. Separate sick leave: 10 sick days per year.\n\n---\nQuestion: What is our PTO policy?"
    }
  ]
}
```

**Critical design decisions in prompt construction:**

1. **System prompt sets the rules:** "Answer ONLY from context" prevents hallucination. "Cite your sources" enables verification.

2. **Context comes before the question:** The model reads sequentially. Placing context first ensures it's processed before the question.

3. **Source labels enable citations:** Each chunk is tagged with `[1]`, `[2]`, etc., so the model can reference them in its response.

4. **The user's original question is preserved verbatim** at the end — it's the actual instruction the model follows.

**What the model sees vs. what the user sees:**

```
What the USER typed:
  "What is our PTO policy?"

What the MODEL sees:
  System: "You are a helpful HR assistant. Answer using ONLY the provided context..."
  User: "[1] Source: pto-policy.md ... [2] Source: pto-policy.md ...
         Question: What is our PTO policy?"

What the USER gets back:
  "According to company policy, employees receive PTO based on tenure:
   new employees (0-2 years) get 15 days, increasing to 30 days for
   those with 10+ years. PTO accrues bi-weekly with a 5-day annual
   carryover limit. [1][2]"
```

The user never sees the retrieved context directly — they just see a well-cited answer. **The context injection is invisible**, just like how tool_result injection is invisible in MCP.

---

## 7. Guardrails & Access Control

RAG introduces new security considerations because the model now has access to your actual data. Retrieval must be governed.

### Metadata Filtering

Every chunk carries metadata, including classification labels. Before returning search results, the system filters based on the user's access level:

```
Query: "What is Mickey's salary?"
User Role: regular_employee

Search results (before filtering):
  ✓ employee-directory: Mickey Mouse (public)         → PASS
  ✗ salary-bands: Engineering Compensation (confidential) → BLOCKED
  ✗ system-credentials: AWS Credentials (restricted)  → BLOCKED

Filtered results sent to LLM:
  Only the employee directory chunk (public info)

LLM response:
  "Mickey Mouse is a Senior Software Engineer in the Orlando office.
   I don't have access to salary information for this query."
```

### Confidential Document Handling

Documents with sensitive classifications should:
1. Be embedded and indexed normally (so the system knows they exist)
2. Be filtered out at query time based on the requester's access level
3. Never leak content into the LLM prompt for unauthorized users

This is a **retrieval-time filter**, not an ingestion-time exclusion. If you exclude confidential docs from the index entirely, you can't even detect when a user is asking about something sensitive.

### Hallucination Detection

The system prompt instructs the model to answer only from provided context. But models can still hallucinate. Defense strategies:

1. **"I don't know" instruction:** Explicitly tell the model to say "I don't have enough information" when context is insufficient
2. **Citation requirement:** If every claim must cite a source chunk, unsupported claims are easy to spot
3. **Confidence scoring:** Some systems ask the model to self-assess confidence
4. **Retrieval score threshold:** If the best chunk has a similarity score below a threshold (e.g., 0.7), the system can preemptively respond "I don't have relevant information" without even calling the LLM

---

## 8. Key Architectural Components

```
┌─────────────────────────────────────────────────────────────────┐
│                      RAG SYSTEM ARCHITECTURE                     │
│                                                                   │
│  ┌──────────┐   ┌───────────┐   ┌───────────────┐              │
│  │ Document │   │           │   │   Embedding   │              │
│  │  Store   │──►│  Chunking │──►│    Model      │──┐           │
│  │          │   │  Engine   │   │               │  │           │
│  └──────────┘   └───────────┘   └───────────────┘  │           │
│  Raw files      Split docs      Convert text        │           │
│  with metadata  into segments   to vectors          │           │
│                                                      │           │
│                     ┌────────────────────────────────┘           │
│                     │                                            │
│                     ▼                                            │
│              ┌─────────────┐                                    │
│              │  Vector DB  │  Store vectors + metadata + text    │
│              │             │  Index for fast similarity search   │
│              └──────┬──────┘                                    │
│                     │                                            │
│                     │ query-time path                            │
│                     ▼                                            │
│              ┌─────────────┐   ┌───────────┐   ┌─────────┐     │
│              │   Query     │──►│ Re-ranker │──►│  LLM    │     │
│              │   Engine    │   │           │   │         │     │
│              └─────────────┘   └───────────┘   └─────────┘     │
│              Embed query,      Refine          Generate         │
│              search vectors,   relevance       grounded         │
│              filter by access  scores          response         │
│                                                                   │
│              ┌─────────────────────────────────────────┐        │
│              │          Application Host                │        │
│              │  Orchestrates the full pipeline,         │        │
│              │  manages access control, builds prompts, │        │
│              │  renders responses to the user           │        │
│              └─────────────────────────────────────────┘        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

| Component | Role | Example Technologies |
|-----------|------|---------------------|
| **Document Store** | Holds raw source documents with metadata | S3, local filesystem, SharePoint |
| **Chunking Engine** | Splits documents into retrieval-sized segments | LangChain splitters, LlamaIndex, custom |
| **Embedding Model** | Converts text to dense vectors | OpenAI text-embedding-3, Cohere embed, local models |
| **Vector Database** | Stores and indexes vectors for fast search | Pinecone, Weaviate, ChromaDB, pgvector |
| **Query Engine** | Orchestrates embed → search → filter pipeline | Custom code, LangChain, LlamaIndex |
| **Re-ranker** | Refines search results with cross-encoder scoring | Cohere rerank, cross-encoder models |
| **LLM** | Generates the final grounded response | Claude, GPT-4, Llama, Mistral |
| **Application Host** | Orchestrates everything, manages access, renders UI | Your application code |

---

## 9. Common Failure Modes

### Bad Chunking
**Symptom:** The model gives partial or incoherent answers.
**Cause:** Chunks split in the middle of relevant information. A fact about meal limits is split across two chunks, and only one is retrieved.
**Fix:** Use overlap, respect document structure, tune chunk size.

### Wrong K Value
**Symptom:** Answers are either too vague (K too low, missing context) or diluted with irrelevant content (K too high, noisy context).
**Cause:** K=1 might miss critical context. K=20 floods the prompt with marginally relevant chunks.
**Fix:** Start with K=3-5. Use re-ranking to improve precision. Monitor answer quality.

### No-Answer Scenarios
**Symptom:** The model fabricates an answer when the knowledge base genuinely doesn't contain the information.
**Cause:** The system prompt doesn't strongly enough instruct the model to say "I don't know."
**Fix:** Explicit "if the context doesn't contain the answer, say so" instruction. Retrieval score thresholds.

### Context Window Overflow
**Symptom:** Retrieved chunks are truncated or the model ignores later context.
**Cause:** Too many chunks retrieved, or chunks are too large, exceeding the model's context window.
**Fix:** Smaller chunks, lower K, summarize before injecting, use models with larger context windows.

### Embedding Model Mismatch
**Symptom:** Search returns completely irrelevant results.
**Cause:** The query was embedded with a different model than the chunks.
**Fix:** Always use the exact same model and version for ingestion and query embedding.

### Stale Index
**Symptom:** The model gives outdated answers despite documents being updated.
**Cause:** The ingestion pipeline hasn't re-processed updated documents.
**Fix:** Incremental re-indexing when documents change. Track document versions.

### Confidential Data Leakage
**Symptom:** The model reveals information the user shouldn't have access to.
**Cause:** Access control filters are missing or misconfigured.
**Fix:** Always filter retrieved chunks by user permissions before injecting into the prompt.

---

## 10. Key Takeaways

1. **RAG doesn't change the model — it changes what the model can see.** The model is still the same LLM; you're just giving it relevant context at query time.

2. **Ingestion and retrieval use the same embedding model.** If they don't match, the vector space is inconsistent and search fails.

3. **Chunking is the most underrated decision.** Bad chunks lead to bad retrieval, which leads to bad answers. Chunk size, overlap, and boundary-awareness all matter.

4. **The prompt construction is invisible to the user.** Just like MCP's tool_result injection, the user asks a question and gets an answer — they never see the retrieved context.

5. **Cosine similarity measures meaning, not keywords.** "Vacation days" and "PTO allowance" are far apart in keyword space but close in embedding space. This is why vector search beats keyword search.

6. **Re-ranking is cheap insurance.** A cross-encoder re-ranker catches relevance errors that bi-encoder similarity misses. The latency cost is small; the quality improvement is significant.

7. **Access control happens at retrieval time, not ingestion time.** Index everything, but filter based on who is asking. This lets you detect sensitive queries without leaking data.

8. **The application host orchestrates everything.** Like MCP's host, the RAG application is the glue — it doesn't do AI, it coordinates: embed, search, filter, assemble prompt, call LLM, render response.

9. **"I don't know" is a feature, not a failure.** A well-designed RAG system knows when it doesn't have enough information and says so, rather than hallucinating.

10. **Every step is observable.** Chunks created, embeddings generated, similarity scores computed, re-ranking applied, prompt assembled, response generated — each step produces inspectable data. That's what makes RAG a "glass box."
