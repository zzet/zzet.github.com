---
title: "Taint analysis sources and sinks: SAST from the code graph"
description: "Taint analysis sources and sinks, explained — plus how a code property graph runs source-to-sink and hardcoded-secret checks as an agent-edit-time guardrail."
date: 2026-05-26
lastmod: 2026-06-06
draft: false
slug: "sast-taint-analysis-from-code-graph"
keywords: ["taint analysis sources and sinks", "SAST security scan local codebase", "code property graph explained", "data flow analysis source to sink", "find hardcoded secrets ast", "security code graph mcp"]
tags: ["go", "mcp", "ai-agents", "gortex", "tools"]
categories: ["gortex"]
---

Your agent writes a handler that reads an `id` from the query string, builds a SQL statement by string concatenation, and runs it. **Taint analysis sources and sinks** is exactly the model that catches this — the query string is the source, `db.Query` is the sink — but the security check that knows it ran *after* the agent already wrote and committed the vulnerable query. The injection only surfaces later: in a CI scan if you're lucky, a bug bounty report if you're not.

That gap is the whole problem. SAST is almost always a separate tool and a separate run — a CI job, a post-PR gate — and AI agents are happy to ship injectable, secret-leaking code in the window before it fires. Veracode's [2025 GenAI Code Security Report](https://www.veracode.com/wp-content/uploads/2025_GenAI_Code_Security_Report_Final.pdf) found that **45% of AI-generated code samples failed security tests** across 80 tasks and 100+ models; SQL injection passed only ~80% of the time, so roughly one in five attempts was still vulnerable. Their [Spring 2026 update](https://www.veracode.com/blog/spring-2026-genai-code-security/) reports the models are still failing despite louder capability claims.

The thesis: if you already run a code graph to let an AI agent navigate your repo, basic taint and source-to-sink checks come from the *same graph*, cheap enough to run at agent-edit time as a guardrail rather than waiting for CI.

> **Short answer:** In taint analysis, a *source* is where untrusted data enters (an HTTP parameter, a request body, a file read), and a *sink* is a sensitive operation that should never receive it unsanitized (a SQL exec, a shell command, a template render). Taint tracking follows the data from source to sink through *propagators* (assignments, function calls); a *sanitizer* on the path — validation, escaping, parameterization — clears the taint. If tainted data reaches a sink with no sanitizer between them, you likely have an injection. Dedicated engines — Semgrep, CodeQL, Joern — do this deeply. A code-graph engine like Gortex does a lighter version (`taint_paths`, `flow_between`, `search_ast` detectors) over the local graph the agent already uses, as a fast edit-time gate — not a replacement for a real SAST program.

---

## What a code property graph is (explained)

A **code property graph (CPG)** is a single queryable graph that merges three views of a program at shared statement and predicate nodes:

- the **abstract syntax tree (AST)** — the program's syntax,
- the **control-flow graph (CFG)** — the order statements execute in,
- the **program dependence graph (PDG)** — which statements depend on which others' data.

It was introduced by Yamaguchi, Golde, Arp, and Rieck in [*Modeling and Discovering Vulnerabilities with Code Property Graphs*](https://en.wikipedia.org/wiki/Code_property_graph) at the IEEE Symposium on Security and Privacy in 2014. Merging the three layers lets one query reason about syntax, control, *and* data flow at once — for example, "find every place an HTTP parameter reaches a SQL string without passing through a sanitizer," a question that touches all three views and can only be answered in one traversal by a graph that fuses them. [Joern](https://docs.joern.io/) is the best-known open-source CPG engine; CodeQL offers a comparable semantic data-flow model.

This is the security cousin of the structure described in [what a code knowledge graph is and why your AI agent needs one](/gortex/code-knowledge-graph-for-llms/): the same nodes-and-edges representation that answers "who calls this?" also, with data-flow edges added, answers "does attacker-controlled data reach this dangerous call?"

## Taint analysis sources and sinks, explained

Taint analysis is a dataflow analysis that tracks untrusted ("tainted") data as it moves through a program. The vocabulary is small and worth getting exactly right, because the SERP for it is owned by precise vendor docs:

| Term | What it is | Example |
|---|---|---|
| **Source** | Where untrusted data enters | `r.URL.Query().Get("id")`, request body, file read |
| **Propagator** | A step that carries taint forward | assignment, concatenation, a function call returning derived data |
| **Sanitizer** | The off-ramp that clears taint | input validation, escaping, query parameterization |
| **Sink** | A sensitive operation that must not receive tainted data | `db.Exec(...)`, `exec.Command(...)`, template render, file path open |

[Semgrep's taint mode](https://docs.semgrep.dev/writing-rules/data-flow/taint-mode/overview) frames it as data flowing from `pattern-sources` to `pattern-sinks` through propagators, with `pattern-sanitizers` as the off-ramp. A finding is a path: source → (propagators) → sink, with no sanitizer in between.

One nuance separates real security analysis from naive value tracking. Plain **data flow** tracks a value only while the value is preserved. **Taint tracking** keeps following the data even when the value changes, as long as the unsafe object still derives from it. [CodeQL's example](https://codeql.github.com/docs/writing-codeql-queries/about-data-flow-analysis/) is `y = x + 1`: data flow tracks `x`, but taint tracking also marks `y` as tainted, because `y` derives from untrusted `x`. Security analysis almost always wants taint tracking, because attacker influence survives concatenation, formatting, and arithmetic — exactly the transformations an agent writes without thinking.

### Why engines restrict to declared sources and sinks

Why not just compute *all* the data flow and flag anything dangerous? [CodeQL's docs](https://codeql.github.com/docs/writing-codeql-queries/about-data-flow-analysis/) give the honest answer: it's infeasible. Flow runs through stdlib functions whose source you don't have; aliasing is hard to resolve precisely; the full flow graph is too large and slow to compute. So engines restrict analysis to *declared* sources and sinks and lean on **modeling** — pre-built descriptions of which framework functions are sources, sinks, or sanitizers. CodeQL ships hundreds per vulnerability class. That modeling is most of the value, and most of the work, in a serious SAST engine.

## Finding hardcoded secrets: regex vs AST context

Secrets are the other thing agents leak. The default approach is pattern matching: [TruffleHog and Gitleaks](https://www.jit.io/resources/appsec-tools/trufflehog-vs-gitleaks-a-detailed-comparison-of-secret-scanning-tools) use regex and entropy to flag credential-shaped strings, with high recall but **low precision and high false-positive rates** — a regex can't tell an AWS key from a test fixture or a benign random token.

An AST- or graph-aware detector cuts the noise by looking at *structure*: is this literal actually assigned into a credential-shaped field, or passed to an auth/connection function? That context turns a noisy match into a higher-confidence finding. It's the same reason graph navigation beats `grep` for code search, covered in [the grep replacement for AI agents — lexical, structural, and graph](/gortex/grep-replacement-for-ai-agents/): a literal match knows the bytes; a graph knows what the bytes *are*.

## The honest menu: dedicated SAST and CPG engines

Before any code-graph guardrail, here is the real field — deeper, more precise tools than anything described later, and for a serious security program you want one of them in CI.

| Tool | What it is | License (as of 2026) | Honest caveat |
|---|---|---|---|
| [**Semgrep**](https://docs.semgrep.dev/writing-rules/data-flow/taint-mode/overview) | Pattern-as-code rules with explicit taint mode; huge community rule library; excellent DX | Community Edition [LGPL-2.1](https://semgrep.dev/docs/semgrep-pro-vs-oss) | CE is **single-file**; propagators work intraprocedurally; cross-file/interprocedural taint is **Pro-only** |
| [**CodeQL**](https://codeql.github.com/docs/writing-codeql-queries/about-data-flow-analysis/) | Semantic query engine with deep, precise global data flow and hundreds of framework models; powers GitHub code scanning | Queries MIT; [CLI/engine license](https://github.com/github/codeql-cli-binaries/blob/main/LICENSE.md) restricts use | Free use limited to OSS / academic / CI on GitHub-hosted OSS; private repos need a commercial GitHub Advanced Security license |
| [**Joern**](https://docs.joern.io/) | The open-source CPG reference — real AST+CFG+PDG graph with a query language and taint engine; often called open-source CodeQL | Apache-2.0 | JVM/Scala-heavy with a steeper query learning curve |
| [**Apiiro**](https://apiiro.com/product/deep-code-analysis-dca-apiiro/) | Enterprise ASPM; Deep Code Analysis builds a software graph (control flow, data flow, APIs, deps, secrets) before AI SAST | Commercial | Enterprise-only, no free local CLI; AI SAST was in public preview as of Dec 2025 |

**Semgrep** is the friendliest place to *write* a taint rule, but its free CE is single-file. **CodeQL** is arguably the deepest open data-flow engine with the richest modeled sources and sinks, but its license is only free for open-source work. **Joern** is the genuine open-source CPG, at the cost of a JVM/Scala query surface. **Apiiro** is the closest positional match to the "graph first, then security" thesis, but it's a commercial enterprise platform.

This is a mature, well-served field; none of what follows competes on depth. What none of them is, by default, is *inside the loop the agent is already running* — a local graph the agent queries while it edits.

## SAST from the same graph: a code-graph guardrail over MCP

Here is the wedge. The graph an agent uses to navigate code is structurally the same thing SAST needs to reason about data flow — nodes for symbols, edges for relationships. If the graph already carries value-flow and capability edges, basic taint and source-to-sink checks fall out of it, locally, with no extra run.

Gortex is a code-graph engine that indexes a repo into an in-memory graph and exposes it over CLI, an MCP server, and a web UI, under the Apache 2.0 license. It is **CPG-lite**, not a full CPG: AST-level structure plus call/reference edges plus value-flow edges. The concession is below — but for an edit-time first gate, CPG-lite is enough, and it answers what "security code graph mcp" searchers are really asking: can the agent run this check itself, over MCP, without a SaaS account?

The relevant tools, all over the same local graph:

- **`taint_paths`** — a pattern-driven source-to-sink sweep. Declare source and sink patterns; it sweeps the graph for paths between them.
- **`flow_between`** — ranked dataflow paths between two points, traversing `value_flow`, `arg_of`, and `returns_to` edges. The "is there a path?" primitive.
- **`search_ast`** detectors — bundled structural detectors including `sql-string-concat`, `weak-crypto`, and `hardcoded-secret`. These are AST-shaped, so the secret detector reasons about *where* a literal lands, not just its entropy.
- **Capability edges** — `reads_env`, `executes_process`, and `accesses_field` are traversable, so you can ask which symbols touch the environment or spawn a process.
- **`analyze` kinds** — `env_var_users`, `config_readers`, `sql_call_sites`, `unsafe_patterns`, for inventory sweeps ("show me every SQL call site").

### A worked source-to-sink example

Take the handler from the top. An agent writes:

```go
func getUser(w http.ResponseWriter, r *http.Request) {
    id := r.URL.Query().Get("id")            // SOURCE: untrusted input
    q := "SELECT * FROM users WHERE id = " + id  // PROPAGATOR: concatenation
    rows, _ := db.Query(q)                    // SINK: SQL exec, no sanitizer
    // ...
}
```

`id` is a source, the concatenation is a propagator, `db.Query` is a sink, and nothing in between parameterizes or validates. At edit time, two checks fire — both MCP tools the agent calls against the local graph:

```text
# Structural detector: a string concatenation reaching a SQL call
search_ast detector="sql-string-concat" path="internal/handlers/"

# Source-to-sink: does request input reach a SQL exec with no sanitizer between?
taint_paths source="r.URL.Query().Get" sink="db.Query"
```

The graph already holds the `value_flow` edge from `id` into `q` and the `arg_of` edge from `q` into `db.Query`, so `flow_between` returns the path before the diff is written. Wired into an MCP-aware agent as a PreToolUse check, the same query blocks the edit and tells the agent to parameterize instead — no CI wait, no separate run. This is the security analogue of [impact analysis — what breaks if I change this function?](/gortex/impact-analysis-what-breaks-if-i-change-this/): "does tainted data reach this sink?" is the same traversal as "what reaches this symbol?", asked over data-flow edges. And it's *deterministic* for the same reason graph edges beat embedding similarity, argued in [graph RAG vs vector RAG for code](/gortex/graph-rag-vs-vector-rag-for-code/): a `value_flow` edge is a resolved relationship, not a similarity score — the path is either in the graph or it isn't.

### The strong, explicit concession

Gortex is not a SAST tool and does not replace one. Be precise about the gap:

- **Depth.** Semgrep, CodeQL, and Joern have mature taint solvers and large libraries of *modeled* framework sources, sinks, and sanitizers. Gortex's graph is CPG-lite — AST-level structure plus call/reference and value-flow edges — **not** a full AST+CFG+PDG at Joern/CodeQL precision. Don't expect equivalent precision or modeled-framework coverage.
- **Scope.** The bundled detectors are the three named above; the `analyze` security kinds are the handful listed. This is a first gate, not a vulnerability catalog.
- **Role.** Use Gortex as a fast, local, edit-time guardrail and audit aid, and run a real SAST program in CI for depth — complementary, not substitutes. (Two more honest limits: `move_symbol` / `inline_symbol` are Go-only for now, and Gortex's headline token figure is a 3–50× per-response range, not a flat claim.)

For completeness: Gortex also runs prompt-injection screening on every tool call, but that protects *the agent itself* from poisoned inputs — it is not application-code SAST coverage.

---

## FAQ

### What are sources and sinks in taint analysis?

A source is any point where untrusted data enters — an HTTP parameter, a request body, a file read. A sink is a sensitive operation that should never receive tainted data uncleaned — a SQL exec, a shell command, a template render. Taint analysis tracks whether tainted data flows from source to sink through propagators like assignments and function calls. A sanitizer on the path — validation, escaping, parameterization — clears the taint; if tainted data reaches a sink with none in between, you likely have an injection.

### What is a code property graph (CPG)?

A code property graph merges three views of a program — the AST (syntax), the CFG (execution order), and the PDG (data dependencies) — into one queryable graph at shared statement and predicate nodes. It was introduced by Yamaguchi, Golde, Arp, and Rieck at IEEE S&P 2014. Because syntax, control, and data flow live together, a single query can find, for example, every HTTP parameter that reaches a SQL string without sanitization. Joern is the best-known open-source CPG engine.

### What is the difference between data flow analysis and taint tracking?

Plain data flow tracks a value while the value is preserved. Taint tracking keeps following the data even when the value changes, as long as the unsafe object still derives from it. CodeQL's example is `y = x + 1`: data flow tracks `x`, but taint tracking also marks `y` as tainted because it derives from untrusted input. Security analysis almost always uses taint tracking, because attacker influence survives concatenation, formatting, and arithmetic.

### Can I run a SAST security scan on a local codebase without a SaaS account?

Yes. Joern (Apache-2.0) runs locally and builds a real CPG with a taint engine. Semgrep Community Edition (LGPL-2.1) runs locally but does single-file analysis only — cross-file taint needs the paid Pro engine. CodeQL's CLI runs locally too, but its license limits free use to open-source codebases, academic research, or CI on GitHub-hosted OSS. If you already run a code-graph engine like Gortex for your agent, basic source-to-sink sweeps and AST detectors come from the same local graph — a fast edit-time gate, not a replacement for a dedicated SAST program.

### Why is finding hardcoded secrets with regex unreliable, and does AST help?

Regex and entropy scanners (TruffleHog, Gitleaks) flag credential-shaped strings with high recall but lack context, so they fire on test fixtures, sample values, and benign random strings — high false-positive rates, low precision. An AST/graph-aware detector checks structure instead: is this literal actually assigned into a credential-shaped field or passed to an auth function? That context turns a noisy match into a higher-confidence finding, which is why graph engines pair a `hardcoded-secret` detector with the value-flow graph.

---

## Bottom line

Security is usually a separate tool and a separate run — and by the time CI fires, the agent already wrote the injectable query and committed the secret. The fix isn't to deepen CI; it's to add a cheap first gate *inside the edit loop*. The code graph an agent already uses to navigate is structurally the same thing taint analysis needs, so basic source-to-sink checks and AST detectors fall out of it locally, deterministically, over MCP: `taint_paths` and `flow_between` over `value_flow`/`arg_of`/`returns_to` edges, the `sql-string-concat`, `weak-crypto`, and `hardcoded-secret` detectors in `search_ast`, traversable `reads_env`/`executes_process` capability edges, and `analyze` kinds like `sql_call_sites` and `unsafe_patterns`. Hold the concession firmly: Gortex is CPG-lite — a guardrail and audit aid, not a SAST program. Run Semgrep, CodeQL, or Joern in CI for depth, and use the graph as the fast, free, local first line that catches the obvious injection before the agent ever writes it.

[github.com/zzet/gortex](https://github.com/zzet/gortex)
