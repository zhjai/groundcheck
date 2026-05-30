# Changelog

## v0.1.1

Cross-file consistency fixes found via an agent-arena review (Codex caught schema drift across the three spec files):

- **Unify enums**: claim `type` and `action` now match across SKILL.md and claim-ledger.md; `claim-ledger.md` is declared the normative enum source.
- **Deterministic anti-loop**: replace the informal "1–2 times" with `send_back_count` / `max_send_backs` (default 2) in the ledger; clarify that the original entry keeps its verdict immutably while only revision entries advance state.
- **Stronger send-back independence**: only raw evidence is returned to the source agent — never the verdict label or `dispute_reason` (internal-only) — preventing anchoring. (Previously fact-gate.md said to send `dispute_reason`, contradicting the independence rule.)
- **Aligned routing** across SKILL/fact-gate/ledger: `send_back` triggers on `refuted` and material `partially_supported`; added `outdated` and `partially_supported` to the gate diagram; replaced the invalid `keep + flag` action with `action: keep` + a separate `flag` field.
- **Clarify "grounding"** as one notion: checking a claim against a verifiable external source (covers source/test, docs, web, citation, and RAG groundedness).

## v0.1.0

- Initial preview of the `groundcheck` skill: single-agent, evidence-grounded claim verification for catching hallucinations in existing content (generated answers, reports, RAG output, docs, code).
- Core protocol: atomic claim decomposition with parent grouping, claim taxonomy, evidence grounding, six-state verdicts (`supported` / `partially_supported` / `refuted` / `unverifiable` / `outdated` / `needs_qualification`) with a claim state machine, and `keep` / `revise` / `retract` / `send_back` actions.
- Shared Claim Ledger interop schema (`interop/claim-ledger.md`) so any multi-agent system can consume verdicts.
- Fact-gate contract (`interop/fact-gate.md`): pre- and post-debate gates, send-back that keeps the original answer immutable and feeds back evidence (not conclusions) to protect agent independence, anti-loop termination, and one-way escalation to agent-arena's `evidence_arena`.
- Anti-false-authority requirements: every verdict must be traceable, contestable, and time-bounded — the verifier is not an authority.
- 20-prompt trigger eval set (`evals/eval-set.md`).
- Design synthesized via agent-arena (Claude Code × Codex) over multiple rounds; naming and Claim Ledger schema cross-critiqued by Codex.
