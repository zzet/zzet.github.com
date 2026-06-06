---
title: "Cursor not understanding my codebase? Bolt on a graph"
description: "Cursor not understanding my codebase? Cline missing references? Keep your tool and add a code-graph MCP server so the agent queries edges, not guesses."
date: 2026-05-22
lastmod: 2026-06-06
draft: false
slug: "when-codebase-context-isnt-enough-cursor-cline"
keywords: ["cursor not understanding my codebase", "cursor only reads first 250 lines", "add codebase indexing to Cline", "continue @codebase deprecated replacement", "mcp for cursor codebase", "improve cursor codebase context"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

You ask Cursor to rename a function and update its callers. It edits two of them, misses a third in a module it never opened, and confidently tells you it's done. Or you point Cline at a 200k-line repo and it reads file after file, following imports like a careful intern, then loses the thread on the cross-file relationship that mattered. If **Cursor is not understanding your codebase** the way you expected, you are not imagining it — and you do not have to switch editors to fix it.

This is the most common complaint on Cursor's own forum, in different costumes: `@codebase` going missing after an update, files mysteriously absent from the index, the agent referencing a symbol you renamed last week, or a monorepo whose indexing spins forever. The model is fine. The *retrieval* underneath it is leaking — and each tool leaks for a different, explainable reason.

The fix in this article is deliberately non-disruptive: keep the agent you already use, and bolt a code-graph MCP server onto it so the agent calls deterministic graph tools instead of guessing from fragments. If you never install anything, you will still leave understanding exactly why your tool loses the thread and what a better retrieval layer looks like.

> **Short answer:** Cursor chunks your code, embeds it, stores the vectors remotely, and retrieves only the top-relevant chunks per query — approximate retrieval, not whole-repo loading — and its agent reads files in roughly 250-line slices. Cline does not index at all; it crawls file-by-file and can miss code in other files. Continue deprecated `@codebase` entirely. The non-disruptive fix is to add a code-graph MCP server (like Gortex) so the agent calls `find_usages`, `get_callers`, and `smart_context` — exact, verifiable edges — instead of guessing.

## Why Cursor is not understanding your codebase

Cursor's index is genuinely well-engineered, and it helps to know what it actually does before blaming it. Per [Cursor's own blog on securely indexing large codebases](https://cursor.com/blog/secure-codebase-indexing), it chunks your code into syntactic pieces locally, turns those chunks into embeddings, and stores the embeddings plus obfuscated paths and line numbers in Turbopuffer, a remote vector database. A Merkle tree of SHA-256 hashes detects changes for fast incremental re-index, and because codebases in an org average roughly 92% similarity, teammates can reuse the index instead of rebuilding it. None of your raw code is kept — it is "gone after the life of the request."

The structural catch is in the retrieval model, not the security model. On each query, Cursor retrieves the **top-relevant chunks**, not the whole repo. Vector similarity is approximate: it surfaces code that *looks* related, which is not the same as code that is *called by*, *implements*, or *references* the thing you asked about. When `@codebase` "misses a reference," that is approximate top-k retrieval doing exactly what it is designed to do — and missing an exact edge it was never built to track.

Then there is the read behavior. As users report repeatedly on Cursor's forum, the agent's `read_file` tool returns only the [first ~250 lines of a file per call](https://forum.cursor.com/t/read-files-in-chunks-of-up-to-250-lines/52597) unless the file has been edited or manually attached, with one thread quoting the agent: "Reading the entire file is not allowed in most cases." Related threads run from [February 2025 through 2026](https://forum.cursor.com/t/read-250-lines-in-agent-mode/83618), so the behavior has been durable. Treat the exact number as forum-observed rather than an official constant — but the symptom is real: the agent crawls long files in slices and can act before it has seen the whole picture.

Stack those two facts and the forum complaints write themselves: [can't access @codebase](https://forum.cursor.com/t/cant-access-codebase/54931), [files missing from @codebase](https://forum.cursor.com/t/files-missing-from-codebase/48522), and [monorepo indexing going into an infinite loop](https://forum.cursor.com/t/when-using-cursor-in-a-monorepo-codebase-indexing-goes-into-an-infinite-loop/71739). Approximate retrieval plus chunk-bounded reads is a leaky picture of a large or recently-renamed repo.

## Why Cline and Continue leak too — for opposite reasons

Cursor is not the only tool with a retrieval gap. The other two popular agents fail differently, and it is worth being fair to both.

**Cline deliberately does not index.** Its position is principled: [no RAG, no embeddings, no vector databases](https://cline.bot/blog/why-cline-doesnt-index-your-codebase-and-why-thats-a-good-thing). It reads files one at a time, following imports like a human, betting on growing context windows. The critique of vector RAG behind that stance is partly correct — chunking can tear apart code logic, and an index is a snapshot that drifts stale. The strengths are real too: always-fresh, fully transparent (you see every file it reads), and no second copy of your code leaves the machine. But Cline concedes its own weakness plainly: the file-by-file approach "can miss conceptually related code in different files" and is "inefficient for large codebases with similar patterns." That is the cross-file edge problem, stated by the tool's own authors.

**Continue removed `@codebase` outright.** Continue's documentation now carries a page literally titled ["Codebase (Deprecated)"](https://docs.continue.dev/reference/deprecated-codebase): the `@Codebase` context provider "has been deprecated in favor of a more integrated approach to codebase awareness." What does Continue recommend instead? Its own guide on [making Agent mode aware of your codebase](https://docs.continue.dev/guides/codebase-documentation-awareness) points users to built-in file/search tools, `.continue/rules` files, and — notably — **MCP servers**: DeepWiki MCP for public GitHub repos, and a custom MCP server for internal code. Continue is telling you to reach for an MCP server. That is a gift to the approach below.

| Tool | How it gets context | Where it leaks |
|---|---|---|
| **Cursor** | chunk → embed → remote vector DB; top-k retrieval; ~250-line reads | approximate retrieval misses exact edges; chunk-bounded reads |
| **Cline** | no index; file-by-file crawl following imports | "can miss conceptually related code in different files" (their words) |
| **Continue** | `@codebase` deprecated; rules + built-in tools + MCP | no built-in repo-wide retrieval; you supply it via MCP |

## The thesis: keep your tool, add the graph

None of these three problems requires a new editor. They require a better retrieval layer the agent can call — one that returns deterministic, verifiable relationships instead of approximations or single-file reads.

That is precisely what MCP exists for. The Model Context Protocol is the socket your agent already speaks; a code-graph MCP server is what you plug into the other end. Cursor and Cline support MCP, and Continue's docs actively recommend it. So the move is: add one server, and the agent gains tools like "every reference to this symbol," "who calls this," and "the minimal working set for this task" — alongside, not instead of, whatever built-in index it already has.

This is the case made at length in [what a code knowledge graph is and why your AI agent needs one](/gortex/code-knowledge-graph-for-llms/). The short version: a graph stores **edges** — calls, implements, references — that embeddings only approximate and a file-by-file crawl can never see all of. For the deeper contrast between Cursor's embedding index and a structural graph, see [how Cursor indexes your codebase: embeddings versus a structural graph](/gortex/how-cursor-indexes-codebase-embeddings-vs-graph/).

## Mapping graph tools to your exact complaints

[Gortex](https://github.com/zzet/gortex) is one such server: a graph-native code-intelligence engine that indexes a repo into an in-memory knowledge graph and exposes it over MCP, CLI, and HTTP. Here is how its tools answer the specific failures above.

| Your complaint | The tool that answers it |
|---|---|
| "`@codebase` missed a reference" / "Cline missed the related file" | `find_usages` — every reference, with typed context, zero false negatives |
| "What calls this?" before I change it | `get_callers` (reverse) / `get_call_chain` (forward) |
| "Stop reading 250 lines at a time" | `smart_context` — one call returns the task-aware minimal working set |
| "Read the whole symbol, not a slice" | `get_symbol_source` — the entire symbol, never truncated at 250 lines |
| "Find it by name even with odd casing" | `search_symbols` — BM25, camelCase-aware |

Two of these are worth dwelling on. `get_symbol_source` returns one symbol's full source regardless of file length — it reports `tokens_saved` per call and runs about **80% fewer tokens** than reading the surrounding file, which directly sidesteps the chunked-read problem. And `smart_context` replaces what would otherwise be **5–10 separate exploration calls** with a single task-aware working set plus a `blast_radius` block, so the agent stops crawling files in slices to assemble context by hand. Across these read patterns, the documented saving is **3–50× fewer tokens per response** (peak around 50× on identifier lookups), not a flat multiple.

Crucially, this is not the frozen index Cline warns about. Retrieval is **hybrid** — BM25 lexical, default-on semantic embeddings, and graph-centrality reranking, fused via Reciprocal Rank Fusion — so it is graph *plus* vector, not graph *versus* vector. And it is not a snapshot: an fsnotify watch issues surgical per-file graph patches with roughly **200ms incremental re-index** (versus a multi-second full rebuild), and `.git/HEAD` watching reconciles branch switches. That is a direct answer to the "snapshot frozen in time" objection — the graph stays current as you type, no manual reindex.

## Setup: one install, then per-agent config

A single install detects and configures every agent on your machine — Cursor, Cline, and Continue among the **15 supported integrations**:

```bash
curl -fsSL https://get.gortex.dev | sh
```

If you prefer to wire it in by hand (or the auto-configure missed an agent), here are the per-agent config shapes. Verify the file paths against current docs, since these tools revise their schemas periodically.

**Cursor** — project `.cursor/mcp.json` or global `~/.cursor/mcp.json` ([Cursor MCP docs](https://cursor.com/docs/mcp)):

```jsonc
{
  "mcpServers": {
    "gortex": {
      "command": "gortex",
      "args": ["mcp"]
    }
  }
}
```

**Cline** — `cline_mcp_settings.json`, reachable via the MCP Servers icon → Configure MCP Servers ([Cline MCP docs](https://docs.cline.bot/mcp/configuring-mcp-servers)):

```jsonc
{
  "mcpServers": {
    "gortex": {
      "command": "gortex",
      "args": ["mcp"],
      "disabled": false
    }
  }
}
```

**Continue** — `.continue/mcpServers/gortex.yaml` in the workspace, or an `mcpServers` block in `config.yaml` ([Continue MCP docs](https://docs.continue.dev/customize/deep-dives/mcp)):

```yaml
mcpServers:
  - name: gortex
    command: gortex
    args: ["mcp"]
```

Once wired in, the agent simply has new tools available; it will call them when a task needs references, callers, or a working set. You can keep using Cursor's built-in index at the same time — the graph complements it, it does not replace it.

## Honest concessions

This approach is not free, and a few caveats matter.

It adds a **local binary and a daemon** to your setup. Cursor's built-in index is genuinely zero-config — it just works the moment you open a repo. Gortex asks you to install something and run a background process. That is a real cost; the payoff is deterministic edges and untruncated symbol reads, but if your repo is small and Cursor's index already serves you, the built-in path is lighter.

There is also a **Cursor-specific limit**. Cursor enforces a [40-tool cap across all enabled MCP servers](https://forum.cursor.com/t/mcp-server-40-tool-limit-in-cursor-is-this-frustrating-your-workflow/81627) — beyond 40, tools become unavailable to the agent. Gortex exposes **100+ tools**, so under Cursor you must enable a curated subset (Settings → Tools & MCP lets you toggle individual tools per server). Cline and Continue have no such cap, so they can use the full set. Pick the handful that map to your complaints — `find_usages`, `get_callers`, `smart_context`, `get_symbol_source`, `search_symbols` — and you are well under the limit.

Finally, the defaults are tuned for zero setup, not maximum recall. The baked **GloVe-50d** embeddings need no API key and start instantly, but for the strongest semantic recall you opt into MiniLM, Ollama, or OpenAI. Exact-symbol search is where the graph dominates; concept and multi-hop semantic recall is more modest. And a couple of refactors (`move_symbol`, `inline_symbol`) are Go-only for now. None of this changes the core trade — verifiable edges over approximations — but you should know it going in. Gortex is also Apache 2.0 and runs entirely on your machine, which is the broader argument in [local-first, open-source code intelligence with no cloud and no API key](/gortex/local-first-code-intelligence-no-cloud/).

## FAQ

### Why is Cursor not understanding my codebase?

Cursor indexes by chunking your code, embedding the chunks, and storing the vectors in a remote database; on each query it retrieves only the top-relevant chunks, not the whole repo. Its agent also reads files in roughly 250-line slices. Approximate retrieval plus chunk-bounded reads is why `@codebase` can miss references on large or recently-renamed code. You can fix it without switching tools by adding a code-graph MCP server so the agent queries exact call and reference edges instead of guessing from fragments.

### Does Cursor really only read 250 lines of a file?

According to Cursor's community forum, the agent's `read_file` tool returns only the first ~250 lines of a file per call unless the file has been edited or manually attached, with threads reporting this from early 2025 into 2026. It is observed behavior, not an officially published spec, so treat the exact number as a forum report. A graph MCP tool like `get_symbol_source` returns an entire symbol regardless of file length, which sidesteps the chunked-read problem.

### Is Continue's @codebase deprecated, and what replaces it?

Yes. Continue's documentation now carries a page titled "Codebase (Deprecated)." The recommended replacement is built-in file-exploration and search tools, `.continue/rules` files, and MCP servers — Continue explicitly suggests DeepWiki MCP for public repos and custom MCP servers for internal code. A code-graph MCP server such as Gortex is a drop-in for that role and exposes deterministic graph tools instead of approximate RAG.

### How do I add codebase indexing or better context to Cline?

Cline deliberately does not index your codebase — no RAG, no embeddings, no vector DB — and acknowledges its file-by-file approach can miss conceptually related code in different files. To give it cross-file understanding, add an MCP server in `cline_mcp_settings.json` that exposes graph tools like `find_usages`, `get_callers`, and `smart_context`. Cline has no cap on the number of MCP tools, so you can enable the full set.

### Can one MCP server give Cursor, Cline, and Continue better context at once?

Yes. A standards-compliant code-graph MCP server like Gortex installs into all three — a single install auto-configures every detected agent (15 in total). The agent then calls deterministic graph tools instead of relying on chunked retrieval. One caveat: Cursor caps MCP at 40 tools, so under Cursor enable a curated subset; Cline and Continue have no such limit. For the broader picture of what a code-intelligence MCP server should do, see [the code-intelligence MCP server your setup is missing](/gortex/mcp-server-for-codebase-context/).

## Bottom line

Your editor is not broken; its retrieval layer is leaking, and each one leaks for a reason you can name. Cursor does approximate top-k retrieval and reads files in slices. Cline crawls file-by-file and misses cross-file edges by its own admission. Continue removed `@codebase` and now tells you to wire in an MCP server. The common fix is the same in all three cases: keep the tool, add a graph, and let the agent call `find_usages`, `get_callers`, `smart_context`, and `get_symbol_source` — exact, verifiable, untruncated — instead of guessing from fragments. The honest costs are a local binary and daemon versus the zero-config built-in index, and Cursor's 40-tool cap meaning you curate which tools you enable there. If your agent acts on what it finds, deterministic edges are the trade worth making.

[github.com/zzet/gortex](https://github.com/zzet/gortex)
