# Fact-Gate Contract — plugging GroundCheck into a multi-agent flow

GroundCheck is a **generic, pluggable verification gate**. Any multi-agent system can use it
through the [Claim Ledger](./claim-ledger.md) contract. This file specifies *where* the gate
sits, *how* send-back works, and *how* it terminates.

## Where the gate sits (two gates)

```
  each agent generates an INDEPENDENT answer
        │
        ▼
  ┌──────────────────────────────────────────┐
  │ PRE-DEBATE GATE                            │
  │ groundcheck(answer) per agent → ledger     │
  └──────────────────────────────────────────┘
        │  route by verdict
        ├─ refuted              → SEND BACK to source_agent (+ evidence) → revise
        ├─ partially_supported  → revise; SEND BACK if the unsupported part is material
        ├─ needs_qualification  → demand caveat (revise)
        ├─ outdated             → re-ground against current source (revise)
        ├─ unverifiable         → pass, flag: low_confidence
        └─ supported            → pass
        │
        ▼
  cross-critique / debate
        │
        ▼
  ┌──────────────────────────────────────────┐
  │ POST-DEBATE GATE                           │
  │ groundcheck(only NEW or CHANGED claims)    │
  └──────────────────────────────────────────┘
        │
        ▼
  synthesis (unresolved claims disclosed)
```

**Why pre-debate:** catch factual errors before debate, so multi-agent debate cannot reinforce
a shared hallucination. **Why also post-debate:** some claims only appear or change during debate.

## Send-back protocol (the "reject" path)

When a `refuted` claim has a `source_agent`:

1. Return to that agent: the claim and the **raw refuting evidence** (source + quote + date).
   Do **not** send the verdict label or `dispute_reason` — those stay internal to the ledger;
   sending a conclusion anchors the agent.
2. The agent produces a revision recorded as a new ledger entry with `revision_of: <claim id>`.
   The original entry is **immutable** and stays on record.
3. The revision must **state what changed** and why; it is then re-checked (`rechecked`).

**Independence guard:** sending evidence (not conclusions) prevents the source agent from
overfitting to appease the checker instead of reasoning from the evidence itself.

## Termination (anti-loop)

- A claim is sent back at most `max_send_backs` times (**default 2**), tracked by `send_back_count`.
- Still `refuted` after the cap → `state: unresolved`, stop, and **disclose in synthesis**
  ("claim X by agent Y was checked false and not resolved").
- The base case is **deterministic evidence**, never "have another agent check again".
- One-way escalation only: `groundcheck → (disputed/high-stakes) → agent-arena evidence_arena → stop`.
  The reverse (evidence_arena calling groundcheck which calls evidence_arena…) is forbidden — these
  are two depths of ONE verification stack, not mutually-recursive tools.

## Minimal integration (any framework)

A host only needs to:

1. Call groundcheck on a piece of content → receive a Claim Ledger.
2. Read each entry's `verdict` + `action` + `source_agent`.
3. Apply routing (block / send_back / caveat / pass).
4. Respect termination (cap send-backs, surface `unresolved`).

No shared runtime is required — only the ledger JSON/YAML contract.

## agent-arena hook (one line on their side)

agent-arena's SKILL.md need only state:

> After independent generation, run `groundcheck` as a pre-debate fact-gate; `refuted` claims
> are sent back to their `source_agent` before debate. Re-check new/changed claims post-debate.
