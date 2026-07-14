# 📊 Infosys Earnings Contradiction Detector

A Retrieval-Augmented Generation (RAG) system that detects contradictions in Infosys earnings call statements across quarterly reports (Q1, Q2, Q3 FY26).

Built as a portfolio project to demonstrate end-to-end RAG pipeline design — from document ingestion to LLM-based reasoning.

---

## What It Does

Given a topic (e.g. "hiring plans" or "revenue guidance"), the system:
1. Retrieves relevant statements from two selected quarters using semantic search
2. Passes those statements to an LLM
3. Returns a structured verdict: **CONTRADICTION FOUND / NO CONTRADICTION / INSUFFICIENT DATA**
4. Shows the exact source chunks and similarity scores used to reach the verdict

### Example Finding
Querying **"utilization and headcount capacity"** (Q1 vs Q3) surfaces a genuine strategic shift:
- **Q1:** *"We are operating at peak headcount — future growth will need additional headcount either through subcontractors or our own employees"*
- **Q3:** *"Investment in talent continues. Net headcount increased by 5,000 to 337,000 employees. Utilization down 1% as we create capacity for future growth"*

**Verdict: CONTRADICTION FOUND** — Q1 frames peak utilization as a constraint. Q3 shows deliberate headcount expansion and intentional utilization reduction to build future capacity. That is a directional reversal.

---

## Architecture

```
PDF Transcripts (Q1, Q2, Q3 FY26)
        ↓
  PyMuPDF Parser
        ↓
  Text Cleaner (regex strips boilerplate headers)
        ↓
  Chunker (200 words, 30-word overlap)
        ↓
  sentence-transformers Embedder (all-MiniLM-L6-v2)
        ↓
  ChromaDB Vector Store (cosine similarity, EphemeralClient)
        ↓
  User Query → Embed → Quarter-filtered Similarity Search
        ↓
  Top-k Chunks from Each Quarter → LLM Prompt
        ↓
  Groq API (llama-3.1-8b-instant) → Structured Verdict
        ↓
  ipywidgets UI (verdict + source chunks + similarity scores)
```

## Design Decisions

### Chunking strategy
**Chunk size: 200 words, overlap: 30 words**

Larger chunks (500 words) produced diluted embeddings — a single vector representing multiple topics retrieves poorly for any one of them. 200 words keeps each chunk semantically focused on one idea.

Overlap prevents meaning loss at chunk boundaries. A sentence split across two chunks loses its context without it.

### Why cosine similarity over euclidean distance
Cosine similarity measures the angle between vectors, not their magnitude. Two chunks expressing the same idea in different lengths produce vectors pointing in similar directions — cosine captures this; euclidean distance doesn't.

### Why not LangChain or LlamaIndex
The ingestion, chunking, embedding, and retrieval pipeline is built manually. This makes the system debuggable and explainable. Using a framework would hide the decisions that interviewers ask about — chunk size, overlap rationale, similarity metric choice.

### Contradiction definition
A contradiction is defined as a **clear reversal in direction or stance**, not just updated numbers. Quarter-on-quarter guidance upgrades are expected and not contradictions. The LLM prompt enforces this distinction explicitly.

---

## Stack

| Component | Tool |
|---|---|
| PDF parsing | PyMuPDF (`fitz`) |
| Embeddings | `sentence-transformers` — all-MiniLM-L6-v2 |
| Vector store | ChromaDB (EphemeralClient) |
| LLM | Groq API — llama-3.1-8b-instant |
| UI | `ipywidgets` |
| Runtime | Google Colab |

---

## Evaluation

Tested on 6 topics across 3 quarter pairs:

| Topic | Quarter Pair | Verdict | Correct? |
|---|---|---|---|
| Utilization and headcount capacity | Q1 vs Q3 | CONTRADICTION | ✅ |
| Discretionary spending from clients | Q1 vs Q3 | CONTRADICTION | ✅ |
| Large deal pipeline and wins | Q2 vs Q3 | NO CONTRADICTION | ✅ |
| AI agents and enterprise adoption | Q1 vs Q3 | NO CONTRADICTION | ✅ |
| Operating margin guidance | Q1 vs Q3 | CONTRADICTION | ❌ False positive |
| Acquisition strategy and M&A | Q1 vs Q2 | NO CONTRADICTION | ✅ |

**Accuracy: 5/6 (83%)**

The false positive on operating margin occurred because retrieval pulled margin discussion chunks rather than the actual guidance statement. The LLM then reasoned on wrong evidence.

---

## Known Limitations

**Retrieval quality is the primary bottleneck.** Cosine similarity distances above 0.5 indicate weak semantic matches. When retrieval fails, the LLM receives irrelevant context and produces false positives regardless of reasoning quality. This is the most common failure mode.

**Domain vocabulary mismatch.** `all-MiniLM-L6-v2` is a general-purpose embedding model. Finance-specific terms like "RPP", "subcon", "onsite utilization" may not embed as precisely as they would in a domain-trained model.

**No persistent storage.** ChromaDB runs in memory. Every session requires re-embedding all transcripts (~30 seconds on CPU).

**LLM hallucination risk.** The structured prompt reduces but does not eliminate hallucination. Verdicts should always be verified against the displayed source chunks.

---

## How to Run

1. Open the notebook in Google Colab
2. Upload transcripts to Google Drive:
   - `infosys_q1_fy26.pdf`
   - `infosys_q2_fy26.pdf`
   - `infosys_q3_fy26.pdf`
3. Add your Groq API key to Colab Secrets as `GROQ_API_KEY`
4. Run all cells top to bottom
5. Use the UI to enter a topic and select a quarter pair

Transcripts are sourced from:
- [Infosys Investor Relations](https://www.infosys.com/investors/reports-filings/quarterly-results.html)
- [Motley Fool Earnings Transcripts](https://www.fool.com/earnings-call-transcripts/)

---

## What I Would Improve With More Time

- **Hybrid search** — combine BM25 keyword search with semantic search to improve retrieval on finance-specific terms
- **Domain embeddings** — use a finance-trained embedding model (e.g. `FinBERT` embeddings) for better semantic matching on financial vocabulary
- **Persistent vector store** — store ChromaDB on Drive so transcripts don't need re-embedding each session
- **Confidence scoring** — surface retrieval distance alongside the verdict so users can judge reliability
- **Multi-company support** — extend to TCS and Wipro transcripts for cross-company contradiction detection

---

## Author

Vancha Garg  
B.Tech CSE (Bioinformatics), VIT Vellore  
[LinkedIn](#) | [GitHub](#)
