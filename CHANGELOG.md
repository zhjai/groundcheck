# Changelog

## v0.1.0

- Initial preview of the `groundcheck` skill: single-agent, evidence-grounded claim verification for catching hallucinations in existing content (generated answers, reports, RAG output, docs, code).
- Core protocol: atomic claim decomposition with parent grouping, claim taxonomy, evidence grounding, six-state verdicts (`supported` / `partially_supported` / `refuted` / `unverifiable` / `outdated` / `needs_qualification`) with a claim state machine, and `keep` / `revise` / `retract` / `send_back` actions.
- Shared Claim Ledger interop schema (`interop/claim-ledger.md`) so any multi-agent system can consume verdicts.
- Fact-gate contract (`interop/fact-gate.md`): pre- and post-debate gates, send-back that keeps the original answer immutable and feeds back evidence (not conclusions) to protect agent independence, anti-loop termination, and one-way escalation to agent-arena's `evidence_arena`.
- Anti-false-authority requirements: every verdict must be traceable, contestable, and time-bounded — the verifier is not an authority.
- 20-prompt trigger eval set (`evals/eval-set.md`).
- Design synthesized via agent-arena (Claude Code × Codex) over multiple rounds; naming and Claim Ledger schema cross-critiqued by Codex.
