# Legal Knowledge System (Catala + Local RAG)

An internal legal knowledge system over the document corpus in `./corpus`.

The core idea: **a legal document is two things mixed together.** Some clauses are
rules — they have conditions, exceptions, and compute an answer. The rest is
definitions, recitals, policy, and prose. Today both halves get handled the same
way: shoved into a vector store and paraphrased by a language model, which is
exactly the wrong treatment for the half that has a right answer.

So we split them:

- **Rule-like clauses are compiled to [Catala](https://catala-lang.org)** — a
  language designed for law, with first-class support for base cases and
  exception hierarchies. Catala is *executed, never read.* No model ever
  interprets the rule at question time; the compiled scope runs and the answer
  is whatever the law computes.
- **Everything else goes to MongoDB vector search**, running locally, quoted
  with citations, never paraphrased into a number.

Every answer is labeled with which engine produced it. The two are never blended
silently.

Two things fall out of this for free. Compiling a new document forces its rules
through the type checker and the existing fact-pattern suite, so a clause that
**contradicts** the corpus fails at ingest instead of being quietly averaged into
an embedding. And because the literate Catala source keeps the original clause
text directly above the code, every computed answer can show the clause that
decided it.

---

## Runtime constraints (hard)

These are non-negotiable and shape every component:

| Constraint | Detail |
|---|---|
| Agent runtime | The product's runtime is an **OpenClaw agent**. |
| Inference | **All** runtime inference goes to the local model at `$LOCAL_LLM_URL` (**Nemotron via vLLM**). |
| No remote calls | **No remote LLM/API calls anywhere in the runtime path.** Not for embeddings, not for reranking, not for a "quick fallback". |
| Execution | Compiled Catala executes inside an **OpenShell sandbox**. |
| Vector search | **MongoDB**, running **locally** — Atlas *Local* (or equivalent) so `$vectorSearch` works without a cloud cluster. No Atlas cloud endpoint in the runtime path. |
| Embeddings | Generated locally, same no-remote-calls rule as inference. |
| Disclosure | The full stack is declared in `WRITEUP.md`. |

Build-time agents may use whatever tooling is available; the *runtime* path is
local-only. That boundary must be explicit in `WRITEUP.md`.

---

## Step 0 — Catala cheatsheet (one agent, 30 min cap)

Before anything else, one agent reads the Catala docs and language reference and
writes **`CATALA_CHEATSHEET.md`**:

- `scope` / `struct` / `enum` / `exception` / `label` syntax
- three golden examples lifted from the official repo
- common compiler errors and their fixes

**Every later agent and every runtime prompt loads this file.** It is the shared
ground truth for the language, and it is what keeps a 30-minute doc-read from
being repeated (badly) by every downstream agent.

**Catala is only ever written through a `generate -> catala typecheck -> repair`
loop.** Never hand-written and hoped for; never emitted without the type checker
signing off.

---

## Components

One sub-agent per component, fanned out in this priority order. Each has a
wall-clock cap. **When a component hits its cap, ship what works and record the
gaps in `WRITEUP.md`** — a partial component with an honest gap list beats a
component that ran long and blocked the demo.

### 1. Ingestion — 3h

One pipeline, used two ways: batch over all of `./corpus`, and incrementally on
each new file.

- Classify each clause **rule-like or not**.
- Rule-like clauses become **Catala modules** with typed scopes, structs, and
  **explicit exception hierarchies** (not flattened into nested conditionals —
  the exception structure is the point).
- The literate source keeps the **original clause text directly above the code**.
  This is the provenance mechanism the chat layer reads back.
- Non-rule clauses are embedded and written to a **MongoDB collection with a
  vector search index**, each with a pointer to the module it qualifies.
- The store must still be **reproducible from the repo**: commit the clause
  records + embeddings as a seed (dump or JSON) and rebuild the index on setup,
  so a clone plus a local mongod reproduces the corpus without a re-embed run.

**Conflict detection is an ingest gate.** On ingest of a new document, run the
full fact-pattern test suite. Any Catala conflict error or failed test **blocks
the merge and surfaces both clauses** — the new one and the one it contradicts.

### 2. Chat — 2h

- **Answering from Catala** means *executing the scope*: extract typed inputs
  from the question and run it.
- If inputs are missing, **list exactly what's needed** — read straight off the
  scope's input signature. No guessing at a fact in order to produce a number.
- Every computed answer renders as: **result + the clause text that decided it +
  the exception branch taken.**
- **Answering from MongoDB** means a `$vectorSearch` query, then quoting the
  retrieved clauses with citations.
- **Never blend silently.** Each part of an answer is labeled with the engine
  that produced it.

### 3. Drafting — stretch, 1.5h

Roundtrip check for new documents:

1. Write Catala.
2. Render it to English.
3. **Re-encode with a fresh agent that has not seen the original.**
4. Diff the ASTs. Loop up to **3 times**.

Non-convergence is not a failure to hide — it is reported as
**"ambiguous draft"**, with the diff attached. A clause that two independent
encoders read differently is a clause that is genuinely ambiguous.

---

## Verification (runs alongside every component)

`/loop` against an **adversarial reviewer sub-agent**:

- It sees **the source document and the artifact**.
- It never sees **the implementer's reasoning**.
- **It never approves anything.**

Its only output is attacks:

- **fact patterns** where the module and the clause disagree, or
- **questions** where the chat answer and the document disagree.

**Every counterexample becomes a permanent `clerk test` case.** Budget **25
attempts per module**, then move on. Keep a running counter of
**attempts / counterexamples / fixed** — it goes in the demo.

The asymmetry is deliberate: an agent that can only attack and can only see the
artifact cannot be talked into agreeing with the reasoning that produced the bug.

---

## Done

Done is not "the components exist." Done is:

- [ ] The demo script in **`DEMO.md` runs end to end on the box**, showing:
  - [ ] a **computed answer with its trace** (result + clause + exception branch)
  - [ ] **missing-fact elicitation** (question with insufficient inputs → exact list of what's needed)
  - [ ] a **conflicting document blocked at ingest**, with both clauses surfaced
  - [ ] an **adversarial counter** and the attempts/counterexamples/fixed counter
- [ ] **`catala typecheck` is clean.**
- [ ] **Every scope has tests covering its exception branches.**

---

## Deliverables

| File | Contents |
|---|---|
| `CATALA_CHEATSHEET.md` | Step 0 output. Loaded by every later agent and every runtime prompt. |
| `WRITEUP.md` | Full stack declaration, plus the gap list from every component that hit its cap. |
| `DEMO.md` | The end-to-end demo script. |
| Catala modules | Literate sources; clause text above code. |
| Vector store | MongoDB collection + vector search index, pointers to qualified modules. |
| Store seed | Committed clause records and embeddings, plus the script that rebuilds the index. |
| `clerk` tests | Fact-pattern suite, including every adversarial counterexample, permanently. |

---

## Notes for whoever picks this up

- This repo is currently empty apart from this README (git is initialized, no
  commits of substance yet). `./corpus` is not present — the corpus is the first
  input the ingestion pipeline needs, and everything else keys off it.
- The ordering is a priority order, not just a schedule: **ingestion before
  chat before drafting.** If time runs out, it runs out at the drafting end.
- The caps are wall-clock and they are real. Shipping partial with a documented
  gap is the specified behavior, not a fallback.

---

## Current state of this repo

| Path | What it is |
|---|---|
| `catala/` | **A patched Catala compiler**, vendored from upstream `b7623302` (`nightly-137`) with seven fixes — see `catala/PATCHES.md`. Upstream's suite passes: `tests/` 707/707, `tests-extra/proof` 66/66. |
| `catala-fixes.patch` | The delta from upstream, 437 lines across 15 files. |
| `CATALA_CHEATSHEET.md` | Step 0 deliverable. Syntax, the `clerk` workflow, machine-readable introspection, twelve legal-encoding traps, the silent-drop hazards, and the gate matrix — every claim reproduced against the compiler. |
| `catala-example/` | A working three-article allowance rule (`clerk test` green) plus a conflicting variant that demonstrates the ingest gate surfacing both clauses. |

Two findings that shape the design more than the rest:

- **`catala proof` (z3) detects exception overlaps and statutory gaps statically
  and hands back a counterexample** — no test needed. It found a contradiction
  that only existed in a narrow band, and located one overlap in a 100-statute
  corpus in 0.43 s. Feed its counterexample back through the interpreter and the
  runtime error names exactly the two conflicting clauses.
- **Conflict detection is scope-local.** Two statutes ingested as separate
  scopes contradict each other with no diagnostic from tests *or* proof. So
  "one module per document" defeats contradiction detection: contradicting
  clauses must land as competing definitions in the *same* scope, which means
  the pipeline must decide two clauses are about the same thing before Catala
  can help. Catala does not solve that step.

Still absent: `./corpus`, `$LOCAL_LLM_URL`, MongoDB, and the ingestion, chat and
drafting components.
