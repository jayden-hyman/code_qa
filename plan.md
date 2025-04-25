# 📀 MVP Blueprint — Code-Aware Retrieval + Generation Pipeline (Tree-sitter Optional)

## A. Index Build (CI / nightly cron)

| # | Stage | What it does | Tool / Notes |
|---|-------|--------------|--------------|
| **A-1** | **Chunker** | Slice every source file into overlapping chunks. | **Start** with `CodeSplitter` (`max_chars=1000`, `overlap=100`). Swap to Tree-sitter later. |
| **A-2** | **Dense embeddings** | Embed each chunk → `chunk_index` | FAISS or LanceDB + `bge-code` / `text-embedding-3-large` |
| **A-3** | **Sparse index** | Build trigram/BM25 or regex index → `sparse_index` | Zoekt, ripgrep – fast literal search |
| **A-4** | **Hierarchical summaries** | LLM creates 1-sentence summary for every file **and** folder | Store vectors in `summary_index` (same DB) |
| **A-5** | **Code graph** | *(Skip for MVP)* — add when switching to Tree-sitter | Will hold callers/callees/import edges |
| **A-6** | **Metadata tables** | `files {path, summary, lang}` (+ empty `funcs` table for now) | SQLite/LanceDB; cheap lookup by path |

---

## B. Runtime Ticket Flow (backend job)

1. **Query enrichment**  
   *LLM + HyDE* → expanded query set **Q**.

2. **Hybrid first-pass retrieval**  
```python
   dense_hits  = chunk_index.search(Q, k=200)
   sparse_hits = sparse_index.search(Q, k=200)
   candidates  = dense_hits | sparse_hits
```

3. **Cross-encoder re-rank** to top 30 snippets.

4. **Load full context bundle**  
   * if file < 800 LOC → include whole file  
   * else include surrounding chunk + header comment

5. **Pass to agent** with these tools:

| Tool | Purpose |
|------|---------|
| `list_files(dir)` | returns `[ {path, summary} ]` |
| `get_file(path)` | returns full source |
| `summarize(text)` | compresses large snippet |
| `drop_context(tag)` | evicts snippets to save tokens |

*(`list_funcs`, `get_func`, `graph_neighbors` will be added after Tree-sitter upgrade.)*

---

## C. Agent Loop (max 3 cycles)

1. Evaluate current context → **enough?**  
2. If **no**, call toolbox to fetch deeper context (e.g., `get_file`, `summarize`, etc.).  
3. Optionally `drop_context` or `summarize` chunks to stay under token limit.  
4. Repeat (≤ 3 iterations or 40 K tokens).  
5. **Generate** answer/patch & cite paths + lines.

---

### 🚣️ Development Plan

1. Get all file types working with tree-sitter

2. Implement LLM generating a query from a user query
3. Implement full file context retrieval for top results
4. Implement LLM creating comments for each file and function
    - Figure out how these work with embeddings (same space, different db? idk)
5. Now go back to beginning and add HyDE, maybe regex to get the best of both worlds
6. Then add agentic loop of diving deeper into files and hopping around the codebase
