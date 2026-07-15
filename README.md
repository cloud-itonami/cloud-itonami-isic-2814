# cloud-itonami-isic-2814: Manufacture of bearings, gears, gearing and driving elements

Open Business Blueprint for **ISIC 2814**: manufacture of bearings, gears, gearing and driving elements — an autonomous "actor" (LLM advisor behind an independent Governor, langgraph-clj StateGraph, append-only audit ledger) that coordinates back-office **bearing/gear-plant operations**: production-batch data logging (product-type/dimensional-tolerance-test/quantity/defect-rate), precision-machining/grinding-line-equipment maintenance scheduling, safety-concern flagging, and outbound product shipment coordination.

This repository designs a forkable OSS business for bearing/gear-plant
operations: run by a qualified operator so a plant keeps its own
operating records instead of renting a closed SaaS.

## Scope: plant operations coordination, not machining/grinding-line control

ISIC 2814 covers the **manufacturing plant** that precision-machines, heat-treats and grinds finished bearings, gears, gearing and driving elements (ball bearings, roller bearings, spur gears, helical gears, worm gears) — including dimensional-tolerance testing — before shipment. This actor coordinates the back-office record keeping around that plant — it never touches the precision-machining/grinding-line equipment directly, and it is never a bearing/gear tolerance-class certification authority (e.g. ISO 492 bearing tolerance class or AGMA/ISO 1328 gear quality mark).

## What this actor does

Proposes **plant operations coordination**, not equipment operation:
- `:log-production-batch` — machining/heat-treatment/grinding batch, output-quality/tolerance-test data logging (administrative, not an operational decision)
- `:schedule-maintenance` — machining/grinding-equipment maintenance scheduling proposal
- `:flag-safety-concern` — surface a materials-safety/equipment-safety concern (always escalates)
- `:coordinate-shipment` — outbound product shipment coordination proposal

## What this actor does NOT do

**CRITICAL SCOPE BOUNDARY — this is a safety-critical domain**
(precision-machining/heat-treatment/grinding line equipment,
materials-handling hazards, dimensional-tolerance certification,
downstream mechanical/consumer-safety consequence via the driving
elements the batch produces):

- Does NOT control precision-machining or grinding-line equipment directly
- Does NOT make plant-safety or certification decisions (that's the plant supervisor's / certification body's exclusive human/institutional authority)
- Does NOT actuate machining/grinding-line equipment (human plant supervisor decides)
- Does NOT self-issue a bearing/gear tolerance-class certification mark (e.g. ISO 492 / AGMA / ISO 1328 — the accredited certification body's exclusive authority — a PERMANENT, unconditional block)
- ONLY proposes/coordinates operations back-office; all actuation and certification requires explicit human/institutional authority
- Safety-concern flagging ALWAYS escalates — never auto-decided, no confidence threshold or phase below escalation

## Architecture

Classic governed-actor pattern (`beargearmfg.operation/build`, a langgraph-clj StateGraph):
1. **`beargearmfg.advisor`** (sealed intelligence node, `BearGearAdvisor`): proposes decisions only, never commits
2. **`beargearmfg.governor`** (independent, `Bearings, Gears and Driving Elements Plant Operations Governor`): validates against domain rules, re-derived from `beargearmfg.registry`'s pure functions and `beargearmfg.store`'s SSoT -- never trusts the advisor's own self-report
   - HARD invariants (always `:hold`, no override):
     - Plant/batch record must be independently verified/registered (`:verified?` AND `:registered?`) before any action is taken against it (equipment before maintenance scheduling, batch before shipment coordination)
     - The request's own `:effect` must be `:propose` (never a direct-write bypass)
     - `:op` must be in the closed four-op allowlist
     - The proposal's own `:effect` must be one of the four propose-shaped effects (no direct precision-machining/grinding-line-equipment control)
     - Directly actuating precision-machining/grinding-line equipment (`:actuate-equipment? true`) is a PERMANENT, unconditional block
     - Self-issuing a bearing/gear tolerance-class certification mark (`:issue-certification? true`, any op) is a PERMANENT, unconditional block
     - A shipment may not push a batch's own recorded shipped quantity past its own logged production quantity (independently recomputed)
     - No double-scheduling the same maintenance record
     - No fabricated `:product-type` value on a production-batch patch
     - No physically implausible `:tolerance-test-um` value on a production-batch patch
     - No physically implausible `:defect-rate-percent` value on a production-batch patch
   - ESCALATE (always human sign-off, overridable by a human):
     - `:flag-safety-concern` always escalates, regardless of confidence
     - Low-confidence proposals
3. **`beargearmfg.phase`** (Phase 0->3 rollout): `:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` are NEVER in any phase's `:auto` set (permanent, matching the governor's own posture); only `:log-production-batch` may auto-commit at phase 3 when clean
4. **`beargearmfg.store`** (append-only audit ledger + SSoT): a single `MemStore` backend behind a `Store` protocol (see ns docstring for why a second Datomic-backed backend is out of scope for this build)

## Development

```bash
# Run tests (top-level deps.edn already pins langgraph+langchain local/root)
clojure -M:test

# Run tests via the workspace :dev override alias (equivalent, kept for sibling-repo parity)
clojure -M:dev:test

# Run the demo
clojure -M:dev:run

# Lint
clojure -M:lint
```

## Status

`:implemented` — `governor.cljc`/`store.cljc`/`advisor.cljc`/`registry.cljc` + `deps.edn` complete the module set; tests green, demo runnable, langgraph-clj integration verified.

## License

AGPL-3.0-or-later
