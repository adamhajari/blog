---
layout: post
title: "Connecting LLMs to the fast.ai Forum"
---

I'm working through the [fast.ai 2022 course](https://course.fast.ai/). The lectures and notebooks are excellent, but they're frozen in 2022, and a lot has changed in the Python ML ecosystem since then. When I hit errors in the notebooks, I often ask Claude to help debug. That's usually productive, but the best fixes usually aren't always in the course materials. They're in the [fast.ai forum](https://forums.fast.ai/), where thousands of students have already worked through the same breakage: renamed APIs, deprecated arguments, version pins that actually work.

The forum is the living knowledge base for the course. The problem is finding anything in it.

## The search problem

Discourse's built-in search is fine for remembering a thread you half recall. It's less helpful when you're staring at a stack trace and trying to discover whether anyone has already solved `TypeError: ... unexpected keyword argument` for this week's combination of `fastai`, `torch`, and `transformers`.

I kept alt-tabbing to the forum, trying different keyword guesses, skimming long threads, and still missing the post that had the exact workaround. Meanwhile Claude was reasoning from 2022 course material and generic library knowledge which was sometimes useful, but not less so than "here's what worked for someone last month on lesson 4."

What I wanted was simple: when I ask Claude a course question, it should search the forum first.

## MCP as the interface

[MCP](https://modelcontextprotocol.io/) (Model Context Protocol) is the obvious glue. Claude Code (and other agents) can call tools exposed by an MCP server. I built [**fastai-forum-mcp**](https://github.com/adamhajari/fastai-forum-mcp): a small Python server that exposes one tool, `search_forum`, backed by a local copy of the entire forum corpus.

The agent workflow looks like this:

1. You hit an error or have a question about a notebook.
2. Claude calls `search_forum` with a natural-language query (often several queries with different phrasings).
3. The tool returns ranked excerpts with URLs, dates, authors, and tags.
4. Claude synthesizes an answer grounded in real community posts — preferably recent ones with lots of likes.

## Architecture: crawl, index, serve

The system has three layers:

```
forums.fast.ai  ──►  forum_crawler.py  ──►  data/posts/*.json
                                              │
                    build_index.py ──────────►│──► BM25 keyword index
                    build_embeddings.py ─────►│──► FAISS semantic index
                                              │
                    mcp_server.py ◄───────────┘
                         │
                    search_forum tool
```

### 1. Crawl the forum

[forums.fast.ai](https://forums.fast.ai/) runs on Discourse, which has a public JSON API so no browser automation required. The [crawler](https://github.com/adamhajari/fastai-forum-mcp/blob/main/forum_crawler.py) stores one JSON file per topic under `data/posts/`, plus a `metadata.json` index with timestamps, tags, and post counts. Since a full crawl from scratch can take up to 24 hours, new users download a [pre-built snapshot](https://huggingface.co/datasets/adamhajari/fastai-forum) (~52 MB compressed) and then run incremental sync for anything newer.

The corpus is roughly **200k posts** across **~21k topics** — small enough to keep in memory, large enough that manual search doesn't scale.

### 2. Build search indexes

Two complementary indexes sit on top of the JSON files:

| Index | Builder | Good for |
|-------|---------|----------|
| **BM25** (keyword) | `build_index.py` | Exact error messages, function names, version strings |
| **FAISS** (semantic) | `build_embeddings.py` | Conceptual questions where wording differs from the posts |

**Hybrid search** (the default) combines both with [Reciprocal Rank Fusion](https://plg.uwaterloo.ca/~gvcormac/cormac/pubs/sigir2009-rrf.pdf): a post that ranks well on keywords *and* on meaning rises to the top. Results are also lightly re-ranked for recency and like count which is important when you're debugging 2022 notebooks with 2024 library stacks.

BM25 has one annoyance: IDF statistics are global, so the keyword index must be rebuilt after each crawl. I documented a [SQLite FTS5 alternative](https://github.com/adamhajari/fastai-forum-mcp/blob/main/search-index-design.md) for incremental updates if that becomes painful; for now, rebuilding takes a few minutes.

Semantic indexing takes longer (~20 minutes on Apple Silicon with `all-MiniLM-L6-v2`) but search itself is fast — querying 200k vectors on CPU is milliseconds. I ran evals comparing embedding models; `nomic-embed-text-v1` did better on retrieval quality if you're willing to pay the build-time cost.

### 3. Serve via MCP

[`mcp_server.py`](https://github.com/adamhajari/fastai-forum-mcp/blob/main/mcp_server.py) loads both indexes at startup and exposes:

```python
search_forum(query: str, n_results: int = 20, mode: str = "hybrid")
```

Modes: `hybrid` (default), `semantic`, or `bm25`. If embeddings haven't been built yet, semantic and hybrid gracefully fall back to keyword search.

Claude Code picks it up automatically via [`.mcp.json`](https://github.com/adamhajari/fastai-forum-mcp/blob/main/.mcp.json) when you work inside the repo:

```json
{
  "mcpServers": {
    "fastai-forum": {
      "command": "uv",
      "args": ["run", "python", "mcp_server.py"]
    }
  }
}
```

## What it's like in practice

A typical session: I'm on lesson 7, `learn.fine_tune()` throws something about `before_batch` callbacks. I paste the traceback. Claude searches the forum for the error string, then for broader phrases like "fine_tune callback fastai v2", and surfaces posts from 2023–2025 with version-specific workarounds. The answer cites real URLs I can open and verify.

It's not perfect. Retrieval can miss, and old highly-upvoted posts can outrank newer fixes, but it's dramatically better than either Claude or I searching alone. The model gets structured excerpts instead of guessing; I get links to the full threads.

## Try it yourself

```bash
git clone https://github.com/adamhajari/fastai-forum-mcp
cd fastai-forum-mcp
uv sync

uv run python forum_crawler.py          # download snapshot + incremental sync
uv run python build_index.py            # BM25 (~few minutes)
uv run python build_embeddings.py       # FAISS (~20 minutes)

# Register with Claude Code (from outside the repo):
claude mcp add fastai-forum -- uv --directory /path/to/fastai-forum-mcp run python mcp_server.py
```

Full setup details are in the [README](https://github.com/adamhajari/fastai-forum-mcp).

## What I learned

**The forum is the course's second textbook.** For a moving-target stack like deep learning, community knowledge ages more gracefully than frozen notebooks. Instrumenting an agent to search that corpus is high leverage.

**Hybrid retrieval beats either alone.** Keyword search nails exact tracebacks; semantic search finds posts that describe the same problem in different words. RRF is a simple way to get both without hand-tuning weights.

**MCP is a good shape for this.** One well-designed tool (`search_forum`) plus clear agent instructions beats bolting a custom RAG pipeline into every chat UI. The crawler and indexes are ordinary Python; only the thin MCP layer needs to know about agents.

**Publish the data.** Hosting the crawled snapshot on [Hugging Face](https://huggingface.co/datasets/adamhajari/fastai-forum) means nobody else needs to wait 24 hours for a first run. If you maintain a similar corpus for another community, consider doing the same.

---

If you're working through fast.ai (or any course with an active forum), I hope this saves you some tab-switching. Issues and PRs welcome on [GitHub](https://github.com/adamhajari/fastai-forum-mcp).
