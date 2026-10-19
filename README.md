# SAIB_Agent
# FAA SAIB Industrial Agent

A grounded question-answering agent over FAA **Special Airworthiness Information Bulletins (SAIBs)**. Ask it a maintenance or airworthiness question in plain English; it retrieves the relevant bulletins, answers using *only* those documents, cites the exact SAIB number(s) it used, and refuses when the corpus doesn't contain the answer.

Built end-to-end on Databricks — Unity Catalog, Vector Search, and the Foundation Model API.

---

## What it does

- Parses 56 FAA SAIB PDFs into structured Delta tables.
- Indexes them in Databricks Vector Search (one chunk per bulletin, so each document's *condition* and *recommended action* stay together).
- On a query: retrieves the top-k bulletins by semantic similarity, feeds them to Llama 3.3 70B with a strict grounding prompt, and returns an answer plus the source SAIB numbers.
- If the answer isn't in the retrieved documents, it replies `NOT_FOUND` rather than guessing.

---

## Architecture

```
PDFs (Volume)
   │  ai_parse_document  (Databricks SQL AI function)
   ▼
saibs_raw  (Delta)              ── structured: saib_number, subject, background, recommendations, full_text
   │  one-chunk-per-doc
   ▼
saib_chunks  (Delta, CDF on)   ── index source
   │  Delta Sync (TRIGGERED)
   ▼
saib_index  (Vector Search)    ── embeddings: databricks-gte-large-en (1024-dim)
   │  similarity_search(top_k)
   ▼
rag_answer()                   ── Llama 3.3 70B (Foundation Model API), temperature 0.0
   ▼
{ answer, sources: [SAIB numbers] }
```

| Layer | Choice |
|---|---|
| Storage / governance | Unity Catalog (`demo_projects.default`) |
| PDF parsing | `ai_parse_document` |
| Embeddings | `databricks-gte-large-en` |
| Vector store | Databricks Vector Search — Delta Sync index, TRIGGERED pipeline |
| LLM | `databricks-meta-llama-3-3-70b-instruct`, temperature 0.0 |
| Grounding | System prompt restricts answers to retrieved context; `NOT_FOUND` as an explicit refusal token |

---

## Data

- **Source:** FAA Special Airworthiness Information Bulletins (public, via the FAA Dynamic Regulatory System).
- **Corpus:** 56 bulletins.
- **Chunking:** one chunk per bulletin. SAIBs are short (1–2 pages); splitting them would separate the described condition from its recommended action, which defeats the point. Each chunk is labelled with its SAIB number in-context so the model can cite correctly.

---

## Evaluation

A 13-query adversarial eval set across four categories. Scored as-is — no retuning to flatter the numbers.

| Category | What it tests | Result |
|---|---|---|
| Clean | Direct, answerable from one bulletin | 3/3 |
| Vague | Underspecified phrasing, answer still present | 4/4 |
| Cross-doc | Two bulletins plausibly match; correct one must win | 3/3 |
| No-answer | Question the corpus can't answer — must refuse | 3/3 (correct `NOT_FOUND`) |
| **Total** | | **13/13** |

**Read this honestly:** 13 queries over 56 documents is a **correctness check, not a precision benchmark.** Its value isn't the perfect score — it's that it catches real defects. This exact eval surfaced a stale-ID bug where the index was returning uncleaned SAIB numbers (`DRS-2026-15`, `AIR-21-15%20R1`) while the answer logic expected cleaned ones, producing false misses. The fix (rewriting the chunk table with cleaned IDs and re-syncing the index) is what took the score to 13/13. A larger corpus and a bigger eval set would be needed to make any claim about retrieval precision.

---

## How to reproduce

1. Upload the SAIB PDFs to a Unity Catalog Volume (default path in the notebook: `/Volumes/demo_projects/default/dynamic_regulatory_system`).
2. Open `saib_rag_agent.py` in Databricks (it imports as a notebook — Databricks source format).
3. Run top-to-bottom. The notebook:
   - parses the PDFs and writes `saibs_raw`,
   - builds and cleans `saib_chunks`,
   - creates the Vector Search endpoint and index,
   - defines `rag_answer()`,
   - runs the eval set and prints the results table.
4. **Wait for the endpoint to report `ONLINE` before the index-creation cell runs.** Creating the index against a still-provisioning endpoint can wedge it (learned the hard way — an index sat stuck for 20+ hours).

> **Cost note:** the Vector Search endpoint bills while it exists, even idle. Delete it when you're done (`vsc.delete_endpoint(...)`).

---

## Limitations

- **Small corpus** — 56 documents. Fine for a grounded-QA demo; not a production knowledge base.
- **Small eval** — 13 queries. Correctness check, not a precision/recall benchmark.
- **No multi-turn** — each query is independent; no conversation memory.
- **No tool-calling (v1)** — this is retrieval + grounded generation. The "agent can look up past failures / check sensor thresholds / suggest actions" tool layer is scoped for v2.
- **No metadata filtering** — retrieval is pure semantic similarity; no filtering by aircraft make/model (make/model extraction was unreliable on this corpus and was descoped).
- **Manual sync** — the index uses a TRIGGERED pipeline, so new documents require an explicit `index.sync()`.

---

## Next steps

- Add the tool-calling layer (similar-failure lookup, threshold checks) — the "agent," not just RAG.
- Metadata-filtered retrieval once make/model extraction is reliable.
- Larger corpus + expanded eval set to make real retrieval-quality claims.

---

## Repository

```
saib_rag_agent.py     # end-to-end Databricks notebook (source format)
README.md
docs/                 # demo screenshot
```
