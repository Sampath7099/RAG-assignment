# RAG: Baseline Retrieval vs. Cross-Encoder Re-Ranking

Comparing a bi-encoder-only RAG pipeline against a two-stage retrieve-then-rerank pipeline
over the Gloomhaven rulebook.

## Run it

```bash
pip install pypdf sentence-transformers google-genai
export GEMINI_API_KEY="your-key"   # free from https://aistudio.google.com/apikey
jupyter notebook rag_rerank.ipynb
```

The key must be exported **before** launching Jupyter, or the kernel won't see it.
Everything except the two answer-generation cells runs without a key.

## Files

| File | What it is |
|---|---|
| `rag_rerank.ipynb` | The deliverable. Run top to bottom. |
| `rag_rerank_executed.ipynb` | Same notebook with saved outputs, including all 7 Answer 1 / Answer 2 pairs. |
| `report.tex` / `report.pdf` | The written report. |
| `gloomhaven_rulebook.pdf` | Corpus — 74k chars, 39 sections, 167 chunks at size 550. Not committed; fetch it with the curl command below. |

## Corpus

The rulebook PDF is not committed (it is not ours to redistribute). Fetch it before running:

```bash
curl -sL https://www.orderofgamers.com/downloads/Gloomhaven_v1.1.pdf -o gloomhaven_rulebook.pdf
```

Then export your key and launch Jupyter:

```bash
export GEMINI_API_KEY="your-key-here"
jupyter notebook rag_rerank.ipynb
```

## Setup

- **Embeddings:** `all-MiniLM-L6-v2` (local, no API)
- **Re-ranker:** `cross-encoder/ms-marco-MiniLM-L-6-v2`
- **LLM:** `gemini-3.8-flash`
- **Vector store:** a NumPy matmul. 167 chunks does not need FAISS.

Both pipelines share stage-1 retrieval and call the *same* `answer()` function with the
*same* prompt string, so the only variable is which 3 chunks get selected.

## Results

Queries are phrased in synonyms rather than the rulebook's own vocabulary ("strike back"
for *retaliate*, "vitality" for *HP*) — that lexical gap is what a bi-encoder struggles with.
Each query carries a `gold` substring that must appear in a chunk for that chunk to actually
answer the question, which makes "did retrieval work" measurable instead of a judgement call.

| verdict | bi | ce | query |
|---|---|---|---|
| **FIXED** | 7 | 1 | am I allowed to skip part of what my card tells me to do? |
| **FIXED** | 6 | 2 | may a stunned figure drink a healing potion on its turn? |
| **FIXED** | 7 | 3 | what occurs if a monster has nothing it is able to aim at? |
| **FIXED** | 9 | 2 | can an enemy creature grab the treasure chest in the room? |
| ~~FIXED~~ artifact | 4 | 3 | may I stockpile two of the same piece of equipment? |
| both correct | 1 | 1 | does mending someone's injuries also cure the venom? |
| both correct | 2 | 1 | can a beast walk through a shut entryway like a hero can? |
| **BROKEN** | 2 | 6 | do I get a bigger hand of cards as I grow more powerful? |
| **BROKEN** | 2 | 4 | do sealed entrances open when someone steps onto them? |
| stage-1 miss | — | — | if I strike back at someone, does that count as attacking? |
| stage-1 miss | — | — | is there a ceiling on restoring a hero's vitality? |

Re-ranking produced **4 genuine rescues**, broke 2, and one apparent fix was a measurement
artifact (see caveat 3).

In every rescue the LLM confirms the retrieval result independently — Answer 1 said some
variant of *"the context does not contain the answer"*, while Answer 2 quoted the governing
rule. Example:

> **Q:** may a stunned figure drink a healing potion on its turn?
> **Answer 1 (baseline):** "the answer is not contained within the text... does not mention
> the 'stunned' condition"
> **Answer 2 (re-ranked):** "No... *STUN The figure cannot perform any abilities or use items
> except, if a character, to perform a long rest.*"

### Latency

| | mean |
|---|---|
| Baseline retrieval | 7.0 ms |
| + cross-encoder re-rank | 160.6 ms |
| Overhead | **22.8×** |

### Why not cross-encode the whole corpus

Measured at 13.2 ms per (query, chunk) pair:

| Scope | Time per query |
|---|---|
| Top-10 re-rank (what we do) | 0.13 s |
| This corpus (167 chunks) | 2.20 s |
| 100k chunks | 22 min |
| 1M chunks | 3.7 hours |

Bi-encoder chunk vectors are computed once and reused forever. A cross-encoder score depends
on query and chunk *jointly*, so nothing can be precomputed — cost grows linearly with corpus
size, at query time.

### Chunk size matters as much as the models

| size | chunks | fixed | broken | stage-1 misses |
|---|---|---|---|---|
| 450 | 219 | 2 | 0 | 3 |
| 550 | 167 | **5** | 2 | 2 |
| 700 | 126 | 2 | 1 | 2 |

## Two caveats worth keeping

1. **Re-ranking is not a free win.** It demoted a correct chunk out of the top 3 twice —
   though in both cases another chunk in the context carried the same rule, so neither
   answer actually got worse.
2. **Recall@10 is a hard ceiling.** A re-ranker can only reorder what stage 1 handed it. Two
   queries failed in both pipelines because the correct chunk was never in the top 10 at all.
3. **Substring-gold undercounts.** The rulebook states the duplicate-items rule twice in
   different words. The gold string matched only one phrasing, so the baseline was scored as
   wrong when its top-3 held the other, equally correct sentence. Any retrieval metric built
   on exact-substring matching inherits this flaw.
