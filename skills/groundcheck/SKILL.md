---
name: groundcheck
description: Use when the user wants to fact-check, verify, or ground existing content — checking generated text, an answer, a report, RAG output, documentation, or code for fabricated or unsupported claims. Triggers on requests like "verify these claims", "check for hallucinations", "are these citations / numbers / APIs real", "fact-check this before I publish", or "is this actually supported by the source". Extracts atomic claims, gathers evidence (web, source files, docs, RAG context), assigns a per-claim verdict with citations, and flags which claims to retract, qualify, or send back. This is SINGLE-agent claim verification grounded in evidence — NOT multi-agent debate over which decision is best (that is agent-arena). Do not use for pure opinions, creative content, or code logic already covered by deterministic tests.
license: MIT
metadata:
  version: "0.1.2"
  author: zhjai
  tags: "groundcheck, fact-check, hallucination, claim-verification, groundedness, citation-verification, rag-evaluation, evidence-checking, verification-gate"
  related_skills: "agent-arena"
---

# GroundCheck

## Overview

GroundCheck is a **single-agent, evidence-grounded claim-verification** skill. It takes content that already exists — a generated answer, a report, RAG output, documentation, or code — extracts the verifiable claims, checks each one against real evidence, and returns a per-claim verdict with citations plus an action (keep / revise / retract / send back).

**It treats hallucination, not overconfidence.** The mechanism is deliberately single-agent + grounding, because multi-agent debate can *reinforce* a shared hallucination rather than catch it. For multi-perspective debate over which decision is best, use the companion skill `agent-arena` instead.

**Core idea:** every factual claim is decomposed to atomic form, grounded in deterministic external evidence, and given a traceable, contestable, time-bounded verdict. The verifier is not an authority — its own output must remain auditable.

## When to Use

- "Verify these claims / fact-check this / check for hallucinations"
- "Are these citations / numbers / APIs / quotes real?"
- "Is this answer actually supported by the source / retrieved context?"
- Before publishing or depending on generated content (report, PR, docs)
- Verifying RAG output groundedness (is the answer supported by what was retrieved?)
- As a **fact-gate** inside any multi-agent flow (see Interop)

## When NOT to Use

- Pure opinions, preferences, or creative content (no verifiable facts)
- Code logic already covered by deterministic tests (tests beat fact-check; but "does this API exist / behave this way" IS in scope)
- "Which option is better?" decisions needing multi-perspective weighing → that is `agent-arena`
- Low-stakes content where the user asked for speed

## Core Protocol

### 1. Extract atomic claims

Decompose the content into **atomic** claims — one checkable assertion each. **Compound claims must be split**, with a parent grouping recorded (`atomicity_parent`).

> Why: "partial support" on a bundled claim can launder a false conclusion — one true sub-claim makes the whole bundle look supported. Atomic decomposition prevents this.

### 2. Classify each claim

`factual` · `numerical` · `temporal` · `causal` · `comparative` · `legal-med-fin` · `code-api` · `citation` · `interpretation` (the last is usually out of scope — flag, do not verify). Enum values are defined normatively in [`interop/claim-ledger.md`](../../interop/claim-ledger.md).

### 3. Gather evidence (grounding)

"Grounding" here means one thing — **checking a claim against a verifiable external source**. That single notion covers source/test verification, doc/spec lookup, current-fact web search, citation-attribution checks, and RAG groundedness (is the claim supported by what was actually retrieved?). Prefer deterministic, external sources, in roughly this order of strength:

- run tests / execute code / compute numbers
- read source files, configs, dependency manifests
- fetch official docs, specs, papers
- web search for current facts
- RAG context (for groundedness checks: is the claim supported by what was actually retrieved?)

Record source, exact quote, source kind, reliability, and **the date the evidence was observed** (evidence goes stale).

### 4. Assign a verdict (with a state machine)

Verdict states: `supported` · `partially_supported` · `refuted` · `unverifiable` · `outdated` · `needs_qualification`.

Claim lifecycle (state machine):

```
extracted → checked → (challenged → revised → rechecked)* → resolved | unresolved
```

### 5. Produce the ledger + actions

Emit a **Claim Ledger** (see `interop/claim-ledger.md`). Each claim carries an `action`:

- `keep` — supported (or `unverifiable`, with `flag: low_confidence`); leave as is
- `revise` — needs_qualification / partially_supported / outdated → rewrite with the caveat, or re-ground
- `retract` — refuted with no salvageable version → remove the claim
- `send_back` — refuted (or material `partially_supported`) owned by another agent → return to that agent to fix (see Interop)

`action` is exactly one of these four; low-confidence passes use the separate `flag` field, not an action. Enums are normative in [`interop/claim-ledger.md`](../../interop/claim-ledger.md).

## Anti-False-Authority (the verifier is not the truth)

GroundCheck's own output can be wrong: extraction can miss scope, evidence can be stale, quotes can be cherry-picked. Downstream agents must NOT treat the ledger as ground truth. Therefore every verdict must be:

- **Traceable** — carry its evidence, source, and the checker (`checker_agent`/tool).
- **Contestable** — a refuted claim can be defended with stronger counter-evidence (which reopens it: `challenged → rechecked`).
- **Time-bounded** — carry `evidence_date` / `checked_at`; a verdict expires when its evidence could have changed.
- **Humble** — prefer `unverifiable` over a guessed verdict; never manufacture a citation to support a verdict.

## Interop with multi-agent systems

GroundCheck is a **generic, pluggable fact-gate** — any multi-agent system (agent-arena, CrewAI, AutoGen, LangGraph, …) can use it via the shared claim-ledger contract. It is not subordinate to agent-arena. Full contract: `interop/fact-gate.md`.

### Two gates (not one)

- **Pre-debate gate** — after each agent's independent answer, before debate. Catches baseline factual errors so debate cannot reinforce a shared hallucination.
- **Post-debate gate** — re-check claims that were *newly introduced or materially changed* during debate.

### Send-back (the "reject" path)

When a `refuted` claim is owned by a specific agent (`source_agent`), return it to that agent with the refuting evidence and demand a revision. Critical independence rules:

- The original answer is **immutable** — revisions are appended (`revision_of`), the original error stays on record.
- A revision must **cite what changed**, not silently overwrite.
- Send the **raw evidence only** — never the verdict label or `dispute_reason` (those stay internal to the ledger); sending a conclusion anchors the source agent into appeasing the checker instead of reasoning from evidence.

### Anti-loop (termination)

- A claim is sent back at most `max_send_backs` times (**default 2**), tracked by `send_back_count` in the ledger.
- Still `refuted` after the cap → mark `unresolved`, stop, and disclose it in the final synthesis ("this claim was checked false and not fixed").
- The base case is **deterministic evidence**, never "have another agent check again". Verification depth is finite: single-agent check → (escalate once to evidence_arena) → stop.

### Escalation to agent-arena (reverse direction)

When evidence is **disputed, high-stakes, adversarial, or retrieval is weak**, escalate that claim to `agent-arena`'s `evidence_arena` mode for multi-perspective scrutiny. This is single, one-way, and does not loop back into GroundCheck.

## Common Mistakes

1. **Not decomposing compound claims** — one true sub-claim launders a false bundle.
2. **Treating the ledger as truth** — the verifier hallucinates too; keep verdicts traceable and contestable.
3. **Sending conclusions, not evidence, on send-back** — contaminates the source agent's independence.
4. **Mutating the original answer** — keep it immutable; append revisions.
5. **Infinite re-checking** — terminate on deterministic evidence, cap send-backs, mark `unresolved`.
6. **Verifying opinions** — only factual/checkable claims are in scope.
7. **Ignoring evidence staleness** — a verdict without a date is not trustworthy for current facts.

## Example Prompts

- "GroundCheck this report — verify the numbers and citations before I publish."
- "This is a generated answer; check for fabricated facts and tell me what to retract."
- "Verify the API methods cited here actually exist in the library."
- "Is this RAG answer actually supported by the retrieved context?"
- "Run GroundCheck as a fact-gate on each agent's answer before they debate."

## Verification Checklist

- [ ] Compound claims were decomposed to atomic form with parent grouping.
- [ ] Each claim has a verdict backed by traceable, dated evidence.
- [ ] Refuted claims got a clear action (retract / revise / send_back).
- [ ] Send-backs carried evidence (not conclusions); originals kept immutable.
- [ ] Send-back loop was capped; unresolved claims disclosed.
- [ ] The ledger is presented as contestable, not as ground truth.
