# ADR-0001: BearGearAdvisor ⊣ Bearings, Gears and Driving Elements Plant Operations Governor architecture

## Status

Accepted. `cloud-itonami-isic-2814` promoted from `:spec` to
`:implemented` in the `kotoba-lang/industry` registry, following the
verified fresh-scaffold protocol established by prior actors in this
fleet.

## Context

`cloud-itonami-isic-2814` publishes an OSS blueprint for bearings,
gears, gearing and driving elements **plant operations coordination**
(production-batch product-type/dimensional-tolerance-test/quantity/
defect-rate data logging, precision-machining/grinding-line-equipment
maintenance scheduling, safety-concern flagging, and outbound product
shipment coordination). Like every actor in this fleet, the blueprint
alone is not an implementation: this ADR records the governed-actor
architecture that promotes it to real, tested code, following the same
langgraph StateGraph + independent Governor + Phase 0->3 rollout
pattern established across the cloud-itonami fleet.

The closest domain analog is `cloud-itonami-isic-2818` (Manufacture of
power-driven hand tools): both are back-office coordination actors for
a fixed manufacturing plant with precision-tested, discrete-unit
finished-goods output and a real physical/consumer safety dimension,
and both share the same four-op shape
(`:log-production-batch`/`:schedule-maintenance`/`:flag-safety-
concern`/`:coordinate-shipment`), the same two-entity verified/
registered gate structure (equipment for maintenance scheduling, batch
for shipment coordination), and the same permanent equipment-actuation
and certification-authority blocks. This build mirrors
`cloud-itonami-isic-2818`'s architecture closely but adapts the hazard
profile, equipment vocabulary, and product taxonomy to the bearings/
gears plant: its finished goods are precision-machined, heat-treated
and ground transmission/driving-element components (ball bearings,
roller bearings, spur gears, helical gears, worm gears) rather than
hand-held power tools, so its equipment kinds are
`:precision-machining-line` and `:grinding-line` rather than 2818's
motor-assembly line and housing-molding press, and its routine
dimensional QC field is `:tolerance-test-um` (dimensional-tolerance
test, plausibility-checked 0-300 um -- informed by ISO 492 bearing
tolerance classes and AGMA/ISO 1328 gear quality grades) rather than
2818's `:hipot-test-kv` (double-insulation withstand test,
plausibility-checked 0-15 kV against IEC 60745). Like 2818, shipment
quantity is tracked in finished-unit UNITS (`:units`/`:quantity-
units`/`:shipped-units`), since bearings and gears are likewise
discrete counted units rather than a bulk weight.

This vertical shares 2818's structural DOMAIN-SPECIFIC permanent
block, adapted to the bearing/gear certification regime: precision
bearings and gears are subject to dimensional tolerance-class
certification regimes (e.g. ISO 492 bearing tolerance classes, AGMA
2000/ISO 1328 gear quality grades). This actor is never the
certification authority -- any proposal (regardless of op) that
declares `:issue-certification? true` is a HARD, PERMANENT,
unconditional block
(`beargearmfg.governor/certification-authority-blocked-violations`),
the same "no phase, no human override" posture as the equipment-
actuation block.

This vertical has NO pre-existing `kotoba-lang/beargearmfg`-style
capability library to wrap (verified: no such repo exists). This build
therefore uses self-contained domain logic -- pure functions in
`beargearmfg.registry` (equipment/batch verification, shipment-
quantity recompute, product-type validation, dimensional-tolerance
plausibility validation, defect-rate plausibility validation) are
re-verified independently by the governor, the same "ground truth, not
self-report" discipline established across prior actors (most
directly `cloud-itonami-isic-2818`'s `powertoolmfg.registry`).

This blueprint's own `:itonami.blueprint/governor` keyword,
`:bearings-gears-plant-operations-governor`, is grep-verified UNIQUE
fleet-wide (`gh search code "bearings-gears-plant-operations-governor"
--owner cloud-itonami`, zero hits before this repo was created).

## Decision

### Decision 1: Self-contained domain logic (no external bearing/gear-manufacturing capability library to wrap)

Unlike actors that delegate to pre-existing domain libraries, this
bearings/gears vertical has NO pre-existing capability library to
wrap. The equipment/batch-verification / shipment-quantity /
product-type / dimensional-tolerance / defect-rate validation
functions live as pure functions in `beargearmfg.registry` and are
re-verified independently by `beargearmfg.governor` -- the same
"ground truth, not self-report" discipline established across prior
actors (most directly `cloud-itonami-isic-2818`'s `powertoolmfg.registry`).

### Decision 2: Coordination, not control — scope boundary at the back-office

This actor is **strictly back-office coordination** of bearings/gears
plant operations. It does NOT:
- Control precision-machining or grinding-line equipment directly
- Make plant-safety or certification decisions (exclusive to the human plant supervisor / accredited certification body)
- Actuate machining/grinding-line equipment
- Self-issue a bearing/gear tolerance-class certification mark (e.g. ISO 492 / AGMA / ISO 1328)

All proposals are `:effect :propose` only. The advisor proposes; the
governor validates; escalation paths funnel to human plant-supervisor
approval. This is not a replacement for the supervisor's authority or
the certification body's authority — it is a proposal-screening and
documentation layer.

**CRITICAL SAFETY BOUNDARY**: bearings/gears manufacturing is a
safety-critical domain (precision-machining/heat-treatment/grinding
line hazards, materials-handling hazards, dimensional-tolerance
certification, downstream mechanical/consumer-safety consequence via
the driving elements the batch produces). Safety-concern flagging
NEVER auto-commits. All safety concerns escalate immediately to human
review.

### Decision 3: Safety-concern escalation — always human sign-off

`:flag-safety-concern` (materials-safety concern, equipment-safety
concern) ALWAYS escalates, never auto-commits. This is not a
"low-stakes proposal" -- it is a circuit-breaker that must reach human
authority.

### Decision 4: Two independent verified/registered gates (equipment AND batch), not one

Like `cloud-itonami-isic-2818`, this vertical has TWO entity kinds
each gating a different op: `:schedule-maintenance` independently
verifies the referenced **equipment** unit's own `:verified?`/
`:registered?` fields; `:coordinate-shipment` independently verifies
the referenced **batch**'s own `:verified?`/`:registered?` fields.
Both are the same "plant/batch record must be independently
verified/registered before any action" HARD invariant applied to the
two distinct record kinds this domain actually has.
`:coordinate-shipment` additionally independently recomputes whether a
batch's own recorded shipped-to-date unit quantity plus the
proposal's own claimed unit quantity would exceed the batch's own
recorded production quantity -- never taken on the advisor's
self-report.

### Decision 5: HARD invariants (no override)

Four HARD governor invariants (elaborated into twelve concrete checks
in `beargearmfg.governor`, mirroring `cloud-itonami-isic-2818`'s own
elaboration of its HARD invariants into concrete checks) block
proposals and cannot be overridden by human approval:
1. Plant/batch record (equipment for maintenance, batch for shipment) must be independently verified/registered before any action is taken against it, and a shipment's quantity must independently recompute within the batch's own logged production quantity
2. Proposals must be `:effect :propose` only (never direct equipment control)
3. Direct precision-machining/grinding-line-equipment control, equipment actuation, or self-issued tolerance-class certification is permanently blocked
4. The op allowlist is closed — `:log-production-batch`/`:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` only

## Consequences

(+) Bearings/gears plant operations back-office now has a documented,
governed, auditable coordination layer that funnels all decisions
through independent validation before human approval.

(+) The "coordination, not control" boundary is explicit in code: all
`:effect :propose`, all real-world actuation requires human plant-
supervisor sign-off, and no tolerance-class certification mark can
ever be self-issued.

(+) Scope is bounded and verifiable: four HARD invariants (elaborated
into twelve concrete governor checks) protect against scope creep into
unauthorized equipment operation, equipment actuation, or
certification self-issuance. Safety concerns are a circuit-breaker,
not a threshold.

(+) Safety-critical discipline is explicit: safety-concern flagging
cannot be rate-limited, suppressed, or auto-decided by phase gate.
Human review is mandatory.

(-) Still a simulation/proposal layer, not a real plant-operations
control system. Equipment actuation, line operation, and certification
issuance remain human-/institution-controlled via external channels.

(-) No integration with real plant-management databases (equipment
telemetry, batch tracking, freight dispatch, certification-body APIs)
— this is a standalone coordinator blueprint.

## Verification

- `cloud-itonami-isic-2814`: `clojure -M:test` green (all tests pass;
  see the superproject ADR and `kotoba-lang/industry` registry entry
  for the exact `Ran N tests containing M assertions, 0 failures, 0
  errors` output, verified from an independent fresh clone), `clojure
  -M:lint` clean, `clojure -M:dev:run` demo narrative exercises
  proposal submission, escalation, and every HARD-hold scenario
  directly (not-propose-effect, unknown-op, equipment-not-verified,
  batch-not-verified, shipment-quantity-exceeded, equipment-actuate-
  blocked, certification-authority-blocked, already-scheduled,
  invalid-product-type, invalid-tolerance-test-um, invalid-defect-rate).
- All source is `.cljc` (portable ClojureScript / JVM / nbb) — no
  JVM-only interop; the actor graph is invoked exclusively via
  `langgraph.graph/run*` (not `.invoke`, which is not cljs-portable).
- Audit ledger is append-only, all decisions are traced; every settled
  request (commit or hold) leaves exactly one ledger fact.
- `deps.edn` pins `io.github.kotoba-lang/langgraph` and
  `io.github.kotoba-lang/langchain` via `:local/root` directly in the
  top-level `:deps` (not only under a `:dev` alias), so a bare
  `clojure -M:test` resolves offline inside the monorepo checkout.
