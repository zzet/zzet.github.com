---
title: "Stop AI agent hallucinating code: a context problem"
description: "How to stop AI agent hallucinating code: why agents invent functions that don't exist, and how grounding generation in your repo's real symbols reduces it."
date: 2026-05-23
lastmod: 2026-06-06
draft: false
slug: "stop-ai-agent-hallucinating-code"
keywords: ["stop AI agent hallucinating code", "stop Claude Code hallucinating functions", "AI invents functions that do not exist", "knowledge graph to reduce LLM hallucinations", "AI generated code breaks on refactor", "reduce code hallucination"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

The agent writes ten clean lines. They read well. They reference `UserService.findByEmail(email)` — a method that does not exist in your repo. Or it passes a raw `userId` string where the real function wants a `*User`. Or it imports from `internal/auth/session` when the package lives at `internal/session/auth`. InfoWorld's practitioner survey collects the same complaint, including a hallucinated parameter that "crashed our staging environment."

To **stop an AI agent hallucinating code**, the instinct is to reach for a smarter model. That is usually the wrong lever. A more capable model still does not know your private symbol table — and when it doesn't know, it guesses, confidently, in your house style.

{{< lead >}}Hallucination is a context problem, not a model problem. An ungrounded model has to fill the gap with the statistically likely guess. Give it the real symbols of your repo and the gap closes.{{< /lead >}}

This article argues that thesis with cited research, separates two different kinds of hallucination that need two different fixes, and then shows the concrete tools — graph queries and pre-merge verification — that ground generation in code that actually exists. It also concedes, up front, what grounding cannot do.

---

## Why your AI agent invents functions that don't exist

A language model does not look anything up by default. It predicts the next token from a distribution learned over training data. Microsoft's Rama Ramaswamy, quoted in InfoWorld's [how to keep AI hallucinations out of your code](https://www.infoworld.com/article/3822251/how-to-keep-ai-hallucinations-out-of-your-code.html), puts it plainly: outputs are "based on statistical likelihoods rather than deterministic logic." When the prompt asks for a method to fetch a user by email and your codebase isn't in the context, `findByEmail` is an extremely likely token sequence. So the model emits it — whether or not it exists.

This is not a rare edge case. The [De-Hallucinator study](https://arxiv.org/html/2401.01701v3) found that **44% of all function-level code-completion tasks were affected by API hallucination**, and among the tasks where the baseline model failed, 59% involved an API usage it could not correctly predict. The same paper shows the cure is grounding: feeding the model retrieved, project-specific API references lifted **API recall by 23.9–61.0%** and cut edit distance by 23.3–50.6%. The model didn't get smarter. It got the facts.

The supply-chain flavor of this makes the cost concrete. The USENIX Security 2025 paper ["We Have a Package for You!"](https://arxiv.org/abs/2406.10279) generated 576,000 code samples across 16 models and found **19.7% of recommended packages were hallucinated** — 205,474 unique fake package names. Attackers now register those names ahead of time. CSO Online's [slopsquatting explainer](https://www.csoonline.com/article/3961304/ai-hallucinations-lead-to-new-cyber-threat-slopsquatting.html) describes the result: a confident `pip install` of a ghost pulls adversary-controlled code.

The mechanism is the same in every case. No grounding → the model fills the gap with a plausible guess → the guess is sometimes wrong → the wrong guess is shaped exactly like real code, so it survives review.

---

## Two kinds of hallucination need two kinds of ground truth

Most advice flattens "code hallucination" into one bucket. It is two, and they have different fixes.

| Class | What the model invents | Why | Ground truth lives in |
|---|---|---|---|
| **Library / API hallucination** | A wrong signature, method, or version for a third-party dependency | Training data is stale or generic about that library | The library's **current docs** |
| **Your-own-codebase hallucination** | A function, parameter, import path, or call pattern in *this* repo — or a signature that has since changed | The model never saw your private symbols, or saw an old snapshot | Your repo's **actual symbols** |

Conflating these leads to buying the wrong fix. If your agent invents a Stripe API call, a fresher symbol graph of *your* code won't help — the truth is in Stripe's docs. If your agent invents `UserService.findByEmail`, fresher library docs won't help — the truth is in your repo. You need both ground truths, pointed at the right class.

### Library hallucination: docs are the fix (credit where it's due)

For external libraries, [Context7](https://github.com/upstash/context7) is the strong answer. It fetches version-specific documentation into the prompt so the model stops citing the API as it existed two training cycles ago. As of 2026-06-06 it is MIT-licensed, open source, free without an API key (a free key raises rate limits), and one of the most-used MCP servers (50k+ GitHub stars). For class #1, recommend it. It does that one job well.

What Context7 cannot do is tell the agent that `UserService.findByEmail` doesn't exist in *your* repo. It has no view of your private symbols, signatures, or import paths. That is class #2 — and a different mechanism owns it.

---

## The solution menu for your-own-codebase hallucination

The grounding question is: where does the agent get facts about *your* code? The honest menu:

| Approach | How it grounds | Where it leaks |
|---|---|---|
| **Bigger / better model** | It doesn't — no repo facts enter the prompt | Still guesses your symbols; the expensive guesses survive review |
| **grep + agent exploration** | Agent reads files, greps names, infers patterns | Ungrounded exploration *invites* invention; matches in comments/strings/dead code mislead it |
| **Vector RAG (embeddings)** | Retrieves similar-looking chunks by cosine similarity | Approximates relevance; a near-miss chunk can prime a near-miss symbol; no notion of "this caller is real" |
| **Fresh library docs (Context7)** | Authoritative for third-party APIs | No view of your private symbols — wrong class for class #2 |
| **Code knowledge graph** | Returns only real symbols, signatures, callers, import paths; verifies before merge | Reduces, does not eliminate; the model can still misuse a *real* symbol |

The industry is drifting toward "let the agent explore with tools." Continue.dev, for instance, [deprecated its `@Codebase` context provider](https://docs.continue.dev/reference/deprecated-codebase) "in favor of a more integrated approach to codebase awareness" — agent-mode tool exploration. Continue's local-first, fully open-source embeddings are a genuine strength. But ungrounded exploration is exactly the condition under which a model invents symbols: it greps, sees something close, and confidently extrapolates. The fix isn't to stop the agent exploring. It's to make exploration return ground truth instead of approximations.

### Why a graph grounds better than vector similarity

Vector RAG is approximate by construction. [Cursor's indexing](https://cursor.com/blog/secure-codebase-indexing) computes a Merkle tree of file hashes, splits changed files into syntactic chunks, embeds them, stores vectors, and retrieves by similarity. By default its agent reads roughly the first 250 lines of a file and search returns about 100 lines (product behavior as of 2026-06-06, not a hard spec). Similarity is a useful prior — but cosine distance between embeddings is a *guess* about relevance, and a guess is precisely what hallucination is made of.

A graph stores deterministic edges instead. As TigerGraph frames it in [why LLMs need knowledge graphs to reduce hallucinations](https://www.tigergraph.com/blog/reducing-ai-hallucinations-why-llms-need-knowledge-graphs-for-accuracy/), grounding in a verified graph constrains prediction to a "factual search space" — the model is anchored to vetted facts rather than left to free-associate. A `calls` edge is not "probably a caller." It *is* a caller. A signature returned from the graph is the signature on disk right now, not a memory from training. That distinction — verifiable edges versus approximate similarity — is the whole argument, and it's why the right framing is graph *plus* vector, not graph versus vector. For the full mechanics, see [graph RAG vs vector RAG for code](/gortex/graph-rag-vs-vector-rag-for-code/) and the pillar, [what a code knowledge graph is and why your AI agent needs one](/gortex/code-knowledge-graph-for-llms/).

---

## How a code graph helps stop an AI agent hallucinating code (and Claude Code hallucinating functions)

[Gortex](https://github.com/zzet/gortex) indexes a repository into an in-memory knowledge graph and exposes it to the agent over an MCP server — 100+ tools, 257 languages, Apache 2.0. The graph *is* the ground truth. Here's the tool-to-pain map for class #2 hallucination.

**The model wants a method that may not exist → it queries the real symbol table.** Instead of emitting `findByEmail` from priors, the agent calls `search_symbols` (BM25, camelCase-aware) and `get_symbol` to retrieve the actual method and its signature. On exact symbol-name queries the lexical ranker hits R@5 of 96.8% — if the method exists, the agent finds the real one; if it doesn't, the agent learns that before writing the call.

```jsonc
// get_symbol → returns the real signature, from disk, now
{ "name": "UserRepo.FindByEmail",
  "meta": { "signature": "func (r *UserRepo) FindByEmail(ctx context.Context, email string) (*User, error)" } }
```

That single fact prevents the two classic mistakes at once: the wrong name (it's `UserRepo.FindByEmail`, not `UserService.findByEmail`) and the wrong argument shape (it takes `ctx, email`, not a bare `User`).

**The model wants a call pattern → it reads real ones.** `find_usages` returns every reference with a `context` tag (call, parameter_type, return_type, field, value), with zero of grep's false positives from comments and strings. The agent copies how the function is actually invoked instead of inventing a plausible-looking invocation.

**The model wants an import → it resolves the real one.** Rather than guessing the package path, `find_declaration` walks from a use site to the declaration, so the agent emits the import path that resolves on this machine — not a path that merely looks idiomatic.

**Before it ships → the change is verified against reality.** `verify_change` checks a proposed signature change against every real caller and interface implementor; `explain_change_impact` returns the risk-tiered blast radius. A fabricated or mismatched caller is caught before merge, not after CI fails. Impact analysis is precomputed (depth-3 reach index, measured p95 ≈ 0.01 ms), so "what breaks?" is cheap enough to ask on every edit — covered in depth in [what breaks if I change this function?](/gortex/impact-analysis-what-breaks-if-i-change-this/).

| Before — ungrounded | After — graph-grounded |
|---|---|
| Emits `UserService.findByEmail(id)` from priors | `get_symbol` returns `UserRepo.FindByEmail(ctx, email)` |
| Copies a call pattern from a stale memory | `find_usages` shows real call sites with context tags |
| Guesses the import path | `find_declaration` resolves the actual declaration |
| Wrong caller surfaces when CI fails | `verify_change` flags it pre-merge |

Token cost is not the point of this article, but it matters for adoption: `get_symbol_source` returns a function with ~80% fewer tokens than reading the whole file, and Gortex reports 3–50× fewer tokens per response than naive file reads (peak ~50× on identifier lookups). Grounding the agent isn't a tax — it's usually cheaper than letting it read files and guess.

### "AI generated code breaks on refactor" is a stale-context problem

A specific, expensive failure: the agent generates calls to a symbol you *renamed last week*. The model isn't hallucinating from thin air — it's primed with dead facts. Often the source is your own instructions file. A `CLAUDE.md` or `AGENTS.md` that still says "use `OldAuth.Verify()`" hands the agent a ghost to call.

Gortex's `audit_agent_config` scans `CLAUDE.md`, `AGENTS.md`, `.cursor/rules`, Copilot instructions, and Windsurf configs for symbol references that no longer match the live graph — so you stop priming the agent with deleted facts. And when you do rename, `rename_symbol` updates cross-file references through the graph so nothing dangles. Keep the graph and the instructions in sync and "breaks on refactor" largely stops happening at the root. For why this failure mode compounds at scale, see [why coding agents fail past 400k lines](/gortex/why-ai-coding-agents-fail-large-codebases/).

---

## The honest limits: ground, then verify

Simon Willison argues, in [hallucinations in code are the least dangerous form of LLM mistakes](https://simonwillison.net/2025/Mar/2/hallucinations-in-code/), that hallucinated methods are nearly harmless because "the moment you run LLM generated code, any hallucinated methods will be instantly obvious: you'll get an error." He's right about the cheap case. A method that doesn't exist throws on the first run, and the compiler or interpreter catches it for free.

That's the case grounding helps *least* with — and it's the case that matters *least*. The expensive cases are the ones a graph addresses:

- **The retry tax.** Even a caught hallucination costs a generate → error → re-prompt → regenerate loop. Multiply across a session and across a team. Grounding skips the loop by getting it right the first time.
- **Semantically valid but wrong.** The agent calls a *real* method that compiles but is the wrong one — `Delete` instead of `SoftDelete`. No error. `find_usages` and `verify_change` surface the mismatch where a compiler can't.
- **Runtime-only failures.** A wrong-but-valid argument passes type checks and fails in production — the staging crash InfoWorld describes.
- **Refactor-time breakage.** Calls to renamed or removed symbols that a stale instructions file kept alive.

And the concession that keeps the rest credible: **a graph reduces hallucination; it does not eliminate it.** The model can still reason wrongly over correct facts, or misuse a real symbol in a real way. Grounding shrinks the gap the model would otherwise fill with a guess — it does not abolish the model's capacity to be wrong. The discipline is *ground + verify*, never *ground and trust*. Tests, impact checks, and running the code remain required. A tool that promised to eliminate hallucination would be hallucinating about hallucination.

A few more honest edges: Gortex's default semantic embeddings are lightweight GloVe-50d (zero setup, no API key); for stronger semantic recall you opt into MiniLM, Ollama, or OpenAI. `move_symbol` and `inline_symbol` are Go-only for now. Benchmarks are single-machine numbers that vary 2–5× by hardware. None of that changes the core claim — for class #2 hallucination, deterministic graph facts beat a model guessing.

---

## FAQ

### Why does my AI agent invent functions that don't exist in my codebase?

Because it predicts the statistically likely next token rather than querying your repo. With no grounding in your actual symbols, the model fills the gap with a plausible-looking guess — InfoWorld notes outputs are "based on statistical likelihoods rather than deterministic logic," and research found 44% of function-level completions are affected by this kind of API hallucination. The fix is to give the agent your real symbol table — actual signatures, callers, and import paths — via a [code knowledge graph](/gortex/code-knowledge-graph-for-llms/) it can query.

### How do I stop Claude Code from hallucinating functions?

Ground it in your repo's real symbols and verify before it ships. Instead of letting the agent grep-and-guess, give it graph tools that return only real symbols (`search_symbols`), real signatures (`get_symbol`), real call patterns (`find_usages`), and the correct import (`find_declaration`), plus a pre-merge check that a proposed change matches every real caller (`verify_change`). Also run `audit_agent_config` so your `CLAUDE.md`/`AGENTS.md` aren't priming the agent with renamed or deleted symbols.

### Can a knowledge graph eliminate LLM hallucinations?

No — it reduces them. A graph removes the gap the model would otherwise fill with a guess by anchoring generation to verified, deterministic facts (calls, references, signatures), which TigerGraph describes as constraining prediction to a "factual search space." But the model can still misuse a real symbol or reason wrongly over correct facts, so verification — impact checks, tests, running the code — is still required.

### Is library hallucination the same as codebase hallucination?

No, and they need different ground truths. Library/API hallucination is the model inventing a wrong signature for a third-party dependency because its training data is stale; the fix is fresh docs (for example, Context7). Your-own-codebase hallucination is the model inventing a function or import path in your repo; the fix is a code graph of your actual symbols. Use both: docs for external libraries, graph for internal code.

### Why does AI-generated code break when I refactor?

Usually because the agent is working from a stale mental model — your instructions file (`CLAUDE.md` / `AGENTS.md`) or its retrieved context still references symbols you've since renamed or removed, so it generates calls to ghosts. Keep the graph and the instructions in sync: `audit_agent_config` flags stale symbol references against the live graph, and graph-aware `rename_symbol` updates cross-file references so nothing dangles.

---

## Bottom line

Reaching for a bigger model treats hallucination as a model problem. It is a context problem. An ungrounded model fills the gap with the statistically likely guess; a grounded one queries the real symbols. Split the problem in two: fresh docs for library hallucination (Context7 is the right tool), and a code graph for your-own-codebase hallucination — where deterministic edges beat approximate similarity, and a pre-merge check catches a fabricated caller before it ships.

The graph reduces hallucination; it does not eliminate it. Ground, then verify — that's the honest version, and it's the one that holds up.

[github.com/zzet/gortex](https://github.com/zzet/gortex)
