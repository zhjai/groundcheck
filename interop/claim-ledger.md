# Claim Ledger — shared interop schema

The Claim Ledger is the **contract** between GroundCheck and any multi-agent system
(agent-arena, CrewAI, AutoGen, LangGraph, …). GroundCheck emits it; orchestrators
consume it to route `send_back` / `revise` / `retract` / `keep`.

## Schema

```yaml
claim_ledger:
  - id: c1                          # stable id within a run
    claim: "DynamoDB strongly-consistent reads work on global tables"
    normalized_claim: "DynamoDB global tables support strongly consistent reads"
    atomicity_parent: null          # set to a parent id if this is a decomposed sub-claim
    source_agent: claude            # who asserted it (enables send_back); null if from plain content
    source_location: "answer §2, line 4"   # where in the content the claim appears
    type: code-api                  # factual|numerical|temporal|causal|comparative|legal-med-fin|code-api|citation|interpretation
    scope: "AWS DynamoDB, global tables"    # what the claim is bounded to
    verdict: refuted                # supported|partially_supported|refuted|unverifiable|outdated|needs_qualification
    state: resolved                 # extracted|checked|challenged|revised|rechecked|resolved|unresolved
    severity: high                  # impact if wrong: low|medium|high
    confidence: high                # checker's confidence in THIS verdict: low|medium|high
    evidence:
      - source: "https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html"
        quote: "Global tables ... use eventual consistency"
        kind: doc                   # test|source|doc|web|rag|calc
        reliability: high           # source reliability: low|medium|high
        evidence_date: "2026-05-30" # when the evidence was observed (staleness!)
    checker_agent: codex            # who/what produced this verdict (auditability)
    checked_at: "2026-05-30"
    action: send_back               # keep|revise|retract|send_back
    dispute_reason: "global tables are eventually consistent; claim overgeneralizes single-region behavior"
    revision_of: null               # id of the claim this revises (append-only history)
    status_history:                 # immutable trail
      - {state: extracted, at: "2026-05-30"}
      - {state: checked,   at: "2026-05-30", verdict: refuted}
```

## Field notes (why each exists)

- **atomicity_parent / normalized_claim** — compound claims are split to atomic form so a
  true sub-claim can't launder a false bundle; the parent groups them back for reporting.
- **source_agent / source_location** — make `send_back` precise: return the right claim to
  the right agent at the right spot.
- **scope / type** — bound what is being checked; prevents over/under-claiming.
- **evidence_date / checked_at** — verdicts are **time-bounded**; current-fact verdicts expire.
- **reliability / source / quote / kind** — traceability; a verdict is only as good as its cited evidence.
- **checker_agent** — the verifier is auditable; it is not an anonymous authority.
- **state / status_history / revision_of** — the claim state machine and an **append-only**
  trail; originals are never mutated.
- **severity** — lets the orchestrator prioritize (a refuted high-severity claim blocks; a
  low-severity unverifiable one can pass with a flag).

## Verdict → action mapping (default)

| verdict | typical action | gate behavior |
|---|---|---|
| supported | keep | pass |
| partially_supported | revise | pass with caveat or send_back if material |
| needs_qualification | revise | demand caveat |
| outdated | revise | re-ground against current source |
| unverifiable | keep + flag | pass with low-confidence flag |
| refuted | retract or send_back | block / return to source_agent |

## Anti-false-authority requirement

Consumers MUST treat the ledger as **contestable**, not as ground truth:
a claim with strong counter-evidence can reopen (`challenged → rechecked`), and any verdict
older than its evidence's freshness window should be re-checked, not trusted blindly.
