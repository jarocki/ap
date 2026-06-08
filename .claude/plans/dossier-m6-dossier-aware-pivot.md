# M-6 — Dossier-Aware Auto-Pivot Policy (per-slice plan)

**Status:** planner-staged 2026-06-08 by W-68-M6-DOSSIER-PIVOT planner stage. Implementer slice `wi-68-m6-impl-01` to follow.
**Workflow:** `w-68-m6-dossier-pivot`
**Goal:** `g-68-m6-pivot`
**Work item to dispatch:** `wi-68-m6-impl-01`
**Drives:** Phase 17I of `MASTER_PLAN.md`. Phase 17I carries the binding decisions and slice index; this document carries full rationale, layering diagram, ranking-formula derivation, and decomposition detail. When the two diverge, Phase 17I wins for binding decisions; this document wins for narrative.

**Inherits from:** Phase 16 §M-6, `.claude/plans/dossier-reframe-v2-roadmap.md` §M-6. Phase 11 (F60 3-gate pivot policy), Phase 17B (M-1 panel), Phase 17D (M-2 extractors + `infer_dossier_state_full` + SLOT_EVIDENCE_TYPES mapping), Phase 17F (M-3 scoring), Phase 17G (M-4 persistence + `load_dossier_state`), Phase 17H (M-5 denial + active falsification + `load_predictions_log`) are prerequisites; all landed by 2026-06-07. Worktree base: AP main at merge `e29e8b1` (M-5 landed, impl `c5dd6bf`).

---

## 1. Goal (single paragraph)

Extend F60's 3-gate auto-pivot policy with a fourth, dossier-aware **candidate-ordering** layer that runs **before** the gates. When the AP agent (or any cascade caller) feeds N candidate SCOs into `EventBus.process_results(...)`, the new layer re-orders the candidate list so that candidates whose evidence type maps to an EMPTY or PARTIAL high-weight dossier slot are evaluated first. Because the F60 budget gate (per-cascade default 5, per-session default 50) consumes budget for the first allowed candidates, this ordering deterministically directs the budget at the pivots that would best fill the dossier puzzle. The 3-gate engine (`PivotPolicy.evaluate`) is BYTEWISE UNCHANGED — F60 invariants are preserved by construction. No new LLM tool, no new event subtype, no schema migration, no `core/workspace.py` edit. The fourth layer is opt-out via a single config flag (`auto_pivot_policy.dossier_aware_ranking`, default `True`) so analysts can disable the dossier preference if they want pure F60 order.

After M-6, a hunt that produces 10 candidate domains plus 3 candidate IP addresses against a workspace where Identity (weight 5.0) is EMPTY and Infrastructure (weight 2.0) is FILLED will spend the 5-callback per-cascade budget on the pivots that contribute Identity / TTPs / Capability evidence first, rather than burning all 5 on the Infrastructure pivots that arrived first lexically. The same hunt against a workspace where all 9 slots are FILLED ranks identically to F60 (no slot pressure → stable sort preserves original order), so M-6 is a no-op when the dossier has nothing left to fill.

**Out-of-scope (explicit, deferred):**
- **No change to `PivotPolicy.evaluate` or any of its three gates.** F60 stays byte-identical. M-6 sits in the layer ABOVE F60, ordering candidates before they reach `publish()`.
- **No change to `EventBus.publish` ordering of subscribers.** When a single SCO has multiple subscribed downstream modules, those callbacks still fire in registration order (a separate, downstream concern). M-6's ranking operates on the source SCO list, not the per-SCO callback list. Per-callback ranking is a possible future slice if profile data shows it matters.
- **No new LLM tool.** The existing `auto_pivot` surface (autopivot on/off chat meta + `_execute_run_module` cascade wiring) is unchanged. M-6 changes the ranking inside the cascade, not the user-facing surface.
- **No new ScoreEvent subtype.** M-6 doesn't emit "this pivot was preferred because slot X is empty" events — pivot decisions are diagnostic, not score-relevant. The existing `DecisionLogEntry` shape (F60, DEC-60-PIVOT-POLICY-005) gains one optional diagnostic field (`dossier_weight: float | None`) so the decision-log inspector can see why a pivot was ranked where it was. The seven required keys (source_sco_id, source_sco_value, candidate_module, gate, verdict, reason, depth) stay; the new field is additive-optional, non-breaking.
- **No new event-bus subscriber.** F60 invariant: `EventBus.publish` and `PivotPolicy.evaluate` are the sole gate authorities. M-6's ranker is a pure function called inline from `process_results`, mirroring the M-4 / M-5 pattern.
- **No revalidation of cached rankings.** DossierState is loaded once at the START of each `process_results` call (one extra `load_dossier_state` per hunt — same cost as M-4's `pre_dossier` load, which already runs once per hunt at the agent/tools.py call site). Within a single `process_results` pass, the state is treated as immutable. If a callback fired by the same pass would have changed slot status, the change is observed on the NEXT hunt, not within the pass — this matches M-4's "pre_dossier vs post_dossier are snapshotted at hunt boundaries" discipline.
- **No `core/pivot_policy.py` modification.** It stays byte-identical (Sacred Practice 12 — single source of truth for the 3-gate engine).
- **No `core/event_bus.py` modification beyond a single optional hook point.** `process_results` gains one optional `ranker: Callable | None = None` keyword parameter; when None, behavior is byte-identical to F60 (the input list is iterated in order). When a ranker is supplied, the SCO list is passed through `ranker(results, source_module) -> list[dict]` before iteration. The default value None means existing callers (any direct `process_results` user) get F60 behavior with zero change. The new layer is supplied by `agent/tools.py` at the single in-tree call site.
- **No vocabulary additions to `SLOT_EVIDENCE_TYPES`.** The M-6 ranker reuses the existing M-1/M-2/M-5 SCO-type→slot mapping authority (`dossier/slots.py::SLOT_EVIDENCE_TYPES`). Any future SCO-type mapping additions land in that authority, and the ranker picks them up for free.

---

## 2. Architecture

### 2.1 Layering authority — one new pure-function module, one optional ranker hook

```
+---------------------------------------------------------------------+
|  Caller: agent/tools.py::run_module (hunt path)                     |
|                                                                     |
|  Existing M-4/M-5 wiring (UNCHANGED in M-6):                        |
|    1. pre_dossier = load_dossier_state(workspace_mgr) or default    |
|    2. predictions_log = load_predictions_log(workspace_mgr)         |
|    3. ... (M-3 scoring, M-4 prediction validation, M-5 falsify) ... |
|                                                                     |
|  M-6 NEW wiring (additive, opt-out via config):                     |
|   3a. if cfg.auto_pivot_policy.dossier_aware_ranking:               |
|         ranker = make_dossier_pivot_ranker(pre_dossier)             |
|       else:                                                         |
|         ranker = None                                               |
|   3b. event_bus.process_results(                                    |
|         results,                                                    |
|         source_module=module_path,                                  |
|         depth=0,                                                    |
|         dry_run=dry_run,                                            |
|         ranker=ranker,        # M-6 NEW optional kwarg              |
|       )                                                             |
|                                                                     |
|  Inside event_bus.process_results (UNCHANGED except for one         |
|  optional kwarg + a 1-line apply-ranker call):                      |
|    self._policy.reset_cascade_budget()                              |
|    ranked = ranker(results, source_module) if ranker else results   |
|    for result in ranked:                                            |
|        ... build PivotEvent ... self.publish(event, dry_run=...)    |
|                                                                     |
|  Inside event_bus.publish (BYTEWISE UNCHANGED):                     |
|    for callback in self._subscribers.get(event.stix_type, []):      |
|        decision = self._policy.evaluate(...)                        |
|        if decision.verdict == "skip": continue                      |
|        ... callback(event) ...                                      |
+---------------------------------------------------------------------+
```

**One new module ships:**

- `src/adversary_pursuit/core/dossier_pivot.py` — pure functions only. Public API:
  - `make_dossier_pivot_ranker(dossier_state: DossierState) -> Callable[[list[dict], str], list[dict]]`
    Returns a closure capturing the dossier state. The closure takes `(results, source_module)` and returns a new list ordered by descending slot-fill score. Stable: ties fall back to F60's existing order (the source list's input order).
  - `compute_slot_fill_score(sco_type: str, dossier_state: DossierState) -> float`
    Pure function. Returns `Σ over slots-that-sco_type-could-fill ( SLOT_WEIGHTS[slot] × status_multiplier(slot_status) )`. Returns `0.0` for SCO types not in `SLOT_EVIDENCE_TYPES`. Returns `0.0` for SCO types whose every candidate slot is already FILLED or DEFERRED.
  - `STATUS_MULTIPLIERS: dict[SlotStatus, float]` — module-level constant (DEC-M6-PIVOT-003).

**One existing module gains one optional kwarg + a 1-line apply:**

- `src/adversary_pursuit/core/event_bus.py` — `EventBus.process_results(self, results, source_module, depth=0, *, dry_run=False)` becomes `EventBus.process_results(self, results, source_module, depth=0, *, dry_run=False, ranker: Callable[[list[dict], str], list[dict]] | None = None)`. Body change: `ranked = ranker(results, source_module) if ranker is not None else results` immediately after `self._policy.reset_cascade_budget()`. The `for result in results` loop becomes `for result in ranked`. **No other change.** `publish`, `subscribe`, `register_module_subscriptions`, `clear_history`, decision-log handling — all byte-identical. The new kwarg defaults to None so existing callers (tests, future callers) get F60 behavior without explicit opt-in.

**One existing module gains one new wiring at the hunt site (additive only):**

- `src/adversary_pursuit/agent/tools.py` — `_execute_run_module` (the single in-tree caller of `event_bus.process_results`) constructs the optional `ranker` based on `self.config.general.auto_pivot_policy.dossier_aware_ranking` and the already-loaded `pre_dossier`. **No new dossier loads** — M-4's `pre_dossier = load_dossier_state(self.workspace_mgr) or default_deferred_state()` at line 449 already runs once per hunt and is reused. M-6 just passes that snapshot into `make_dossier_pivot_ranker(pre_dossier)` and threads the closure into the existing `event_bus.process_results(...)` call at line 634.

**One existing module gains one new field (additive only):**

- `src/adversary_pursuit/core/config.py` — `AutoPivotPolicyConfig` gains `dossier_aware_ranking: bool = True`. Default `True` means M-6's behavior is ON for fresh installs; default-friendly TOML round-trip preserved per existing AutoPivotPolicyConfig serializer pattern (the `_strip_none` logic at config.py:298 already handles new boolean fields without modification — verified by reading the existing implementation).

**Modules that are BYTEWISE UNCHANGED in M-6:**

- `src/adversary_pursuit/core/pivot_policy.py` — F60 invariant. The 3-gate engine, IOC value rules, confidence threshold, budget counters, decision-log shape — all byte-identical.
- `src/adversary_pursuit/core/console.py` — has no pivot wiring (planner verified: `grep -n "pivot\|event_bus" core/console.py` returns nothing). The dispatch context's mention of "core/console.py" as a pivot wiring site is incorrect for this codebase; M-6 leaves console.py byte-identical, as did F60 and M-5.
- `src/adversary_pursuit/dossier/slot_inference.py` — M-5 byte-identical.
- `src/adversary_pursuit/dossier/slots.py` — M-1/M-2 byte-identical. `SLOT_WEIGHTS` and `SLOT_EVIDENCE_TYPES` are M-6's read-only inputs.
- `src/adversary_pursuit/dossier/state.py` — M-4 byte-identical.
- `src/adversary_pursuit/dossier/predictions.py` — M-5 byte-identical.
- `src/adversary_pursuit/dossier/scoring.py` — M-5 byte-identical (no new event subtype).
- `src/adversary_pursuit/core/workspace.py` — F59 / M-5 BYTEWISE UNCHANGED (no schema change, no new public method, no new reserved action).
- `src/adversary_pursuit/models/database.py` — DEC-DB-002 BYTEWISE UNCHANGED.
- `src/adversary_pursuit/dossier/panel.py` — M-1 byte-identical.
- `src/adversary_pursuit/gamification/scoring.py` — M-3 byte-identical.
- `src/adversary_pursuit/gamification/celebrations.py` — F63 byte-identical.
- `src/adversary_pursuit/core/streak.py` — F62 byte-identical.
- `src/adversary_pursuit/agent/chat.py` — M-5 byte-identical (no new meta-command; the `autopivot on/off` toggle remains the user surface).

### 2.2 F60 wrapper vs replace — wrap (DEC-M6-PIVOT-001)

The dispatch context posed this explicit decision: wrap F60 or re-implement. Wrap is the answer for three independent reasons:

| option | description | verdict | rationale |
|--------|-------------|---------|-----------|
| (a) **wrap: new ranker layer above F60** (recommended) | New `core/dossier_pivot.py` module owns the ranker. `EventBus.process_results` gains one optional `ranker` kwarg. `PivotPolicy.evaluate` is byte-identical. F60 owns the gate decision; M-6 owns the candidate ordering. | **accepted** | Single source of truth for the 3-gate engine preserved (Sacred Practice 12). F60 tests stay byte-identical and continue to assert the gate semantics. M-6 tests own ranking semantics. The two concerns are orthogonal: F60 says "should this pivot be allowed?"; M-6 says "in what order should candidates be presented?". Wrap also preserves opt-out cleanly via the config flag. |
| (b) re-implement: dossier-aware logic injected into `PivotPolicy.evaluate` | `PivotPolicy.evaluate` gains a 4th gate ("dossier slot pressure") that runs after budget and modulates the verdict. | **rejected** | Violates the F60 invariant that the 3 gates are the gate authority (DEC-60-PIVOT-POLICY-002). Mixing gate logic (allow/skip) with ranking logic (ordering) in the same call boundary produces a method that does two unrelated jobs. The "4th gate" framing also doesn't fit the semantics — dossier preference isn't a YES/NO decision, it's a priority weight; forcing it into the gate shape produces an ugly hybrid. |
| (c) re-implement: dossier-aware logic replaces `PivotPolicy.evaluate` entirely | A new `DossierAwarePivotPolicy` class subsumes F60 and re-implements its gates. | **rejected** | Largest blast radius. Forces every F60 test to rewrite against the new class. Forces every DEC-60-* invariant to be re-litigated. M-6 is a "policy upgrade" by name only — the underlying gates don't need to change at all; only the ordering of candidates that hit them does. Replacement is the wrong tool. |

The wrap pattern also makes M-6 trivially toggleable: the `dossier_aware_ranking: bool = True` config flag selects between "supply the ranker" and "pass None". With None, behavior is byte-identical to F60 — a clean kill switch for any unforeseen interaction.

### 2.3 New module location — `core/dossier_pivot.py` (DEC-M6-PIVOT-002)

The dispatch context posed this decision: extend `core/pivot_policy.py` (currently on M-5's forbidden list — would be lifted for M-6) or land a new `core/dossier_pivot.py` module that wraps F60. New module preferred.

| option | description | verdict | rationale |
|--------|-------------|---------|-----------|
| (a) **new module: `core/dossier_pivot.py`** (recommended) | New module owns `make_dossier_pivot_ranker`, `compute_slot_fill_score`, `STATUS_MULTIPLIERS`. `core/pivot_policy.py` stays byte-identical. | **accepted** | Separation of concerns: F60 (`pivot_policy.py`) owns gate semantics; M-6 (`dossier_pivot.py`) owns ranking semantics. New module is the dossier-aware layer in name as well as in code. Future readers see the layer name and know what it's for. Lifts the M-5 "forbidden: pivot_policy.py" line cleanly — pivot_policy.py just stays forbidden because nothing in M-6 needs to touch it. |
| (b) extend `core/pivot_policy.py` with ranker functions | New `make_dossier_pivot_ranker` / `compute_slot_fill_score` ship at the bottom of `pivot_policy.py`. | **rejected** | The 3-gate engine and the ranker are independent concerns. Co-locating them blurs ownership (any future reader has to learn which functions are gate-side and which are ranker-side). Also: `pivot_policy.py` currently has zero dossier dependencies — adding the `from adversary_pursuit.dossier.slots import ...` import to it introduces a new cross-module dependency direction (core → dossier) that didn't exist before. The new module isolates that dependency. |
| (c) extend `dossier/__init__.py` (or a new `dossier/pivot_ranking.py`) | The ranker lives in the dossier package as a dossier-side helper. | **rejected** | The `dossier/` package is the slot inference + persistence authority. Adding a function that THREADS dossier data into the core event-bus wiring crosses a layer line in the wrong direction (dossier doesn't know about core's event bus). The ranker is a core-side consumer of dossier data, not a dossier-side producer. Belongs in `core/`. |

### 2.4 Slot-fill score formula (DEC-M6-PIVOT-003 / -004)

The ranker scores each candidate SCO by summing, over the slots that the SCO's type could fill, the slot's importance weight multiplied by a status-dependent multiplier. Higher score = higher rank = evaluated first against F60's gates.

```
score(sco) = Σ_{slot ∈ SLOT_EVIDENCE_TYPES[sco["type"]]} (
    SLOT_WEIGHTS[slot] × STATUS_MULTIPLIERS[dossier_state.slots[slot].status]
)
```

with:

```python
# DEC-M6-PIVOT-003 — single authority for status weighting in dossier_pivot.py
STATUS_MULTIPLIERS: dict[SlotStatus, float] = {
    SlotStatus.EMPTY: 1.0,      # max pressure — slot has no evidence yet
    SlotStatus.PARTIAL: 0.5,    # half pressure — slot has some evidence
    SlotStatus.FILLED: 0.0,     # no pressure — slot is already done
    SlotStatus.DEFERRED: 0.0,   # no pressure — inference path not active yet
}
```

**Worked example.** A workspace has Identity=EMPTY (5.0×1.0=5.0), Infrastructure=FILLED (2.0×0.0=0.0), TTPs=PARTIAL (3.0×0.5=1.5). The source module produces these candidates:
- `email-addr "actor@example.com"` → fills Identity → score 5.0
- `url "https://malware.example/payload.exe"` → fills TTPs → score 1.5
- `ipv4-addr "1.2.3.4"` → fills Infrastructure → score 0.0
- `domain-name "actor-c2.test"` → fills Infrastructure → score 0.0
- `x509-certificate "..."` → fills Identity → score 5.0
- `file "evil.exe"` → fills TTPs → score 1.5

Ranked output (descending score; ties preserve input order — stable sort): email-addr (5.0), x509-certificate (5.0), url (1.5), file (1.5), ipv4-addr (0.0), domain-name (0.0). The F60 per-cascade budget of 5 now allows the four high-value pivots plus one of the two Infrastructure pivots — exactly what M-6 promises.

**SCO types not in SLOT_EVIDENCE_TYPES** (e.g., `mutex`, `windows-registry-key` — types that no current AP module emits but that may appear in the future) get score `0.0` and sort to the end. They are still iterated; F60 still evaluates them; if budget remains they still fire. M-6 never *prevents* a pivot — it only re-orders.

### 2.5 Combination with F60 confidence-gate score (DEC-M6-PIVOT-005)

The dispatch context posed this decision: additive or multiplicative combination of M-6's slot-fill score with F60's confidence-gate score.

**Decision: neither.** F60's confidence gate is a boolean threshold (`x_abuse_confidence_score < confidence_threshold` → skip), not a score that ranks candidates. There is no F60 "score" to combine with. M-6's ranking is a pure pre-filter ordering; the F60 gates remain hard YES/NO checks that run on the M-6-ordered list. If a future slice introduces a confidence-as-a-rank-score in F60, that slice will re-open this decision; in M-6, the rank is the slot-fill score alone.

**A secondary tie-break order** (DEC-M6-PIVOT-006) applies when two candidates have equal slot-fill score:

1. Higher `x_abuse_confidence_score` first (when both have the field; missing field treated as `-1`). Mirrors F60's confidence-gate preference for high-confidence indicators.
2. Stable: original input order.

The tie-break is intentionally conservative — it does not invent ordering when F60 wouldn't have either. A test (`test_dossier_pivot.py`) asserts the tie-break is applied deterministically.

### 2.6 Decision-log diagnostic field (DEC-M6-PIVOT-007)

The existing `DecisionLogEntry` TypedDict (F60, DEC-60-PIVOT-POLICY-005) has seven required keys. M-6 adds **one optional field**: `dossier_weight: float | None`. The seven required keys stay required; downstream consumers see the new field as additive. When the M-6 ranker has scored a candidate, the value is the slot-fill score for that SCO. When the M-6 ranker was disabled (config flag off, or no ranker supplied to `process_results`), the value is `None`. The `EventBus.publish` body sets the field after `self._policy.build_log_entry(...)` returns the base entry, by reading from a small per-call lookup the M-6 ranker populates as a side-channel (a dict keyed by `(sco_id, source_sco_value)` → `dossier_weight`).

This is the only modification to F60 surfaces in M-6 — and it is strictly additive. The new field is non-breaking because (a) TypedDict additions don't change required-key contracts and (b) the field defaults to `None` for any consumer that doesn't inspect it.

**Alternative considered:** dropping the diagnostic field entirely (leave the ranker invisible in the decision log). Rejected — diagnostics matter for debugging "why did the ranker pick this order?" questions that will inevitably arise. Without the field, the only way to introspect M-6 decisions is to re-run `compute_slot_fill_score` by hand. Adding the field once costs ~5 lines and saves future debuggers hours.

### 2.7 Config field — `auto_pivot_policy.dossier_aware_ranking` (DEC-M6-PIVOT-008)

`AutoPivotPolicyConfig` gains:

```python
dossier_aware_ranking: bool = True
"""When True (default), agent/tools.py supplies a dossier-aware ranker to
EventBus.process_results so cascade candidates are evaluated in descending
slot-fill-score order. When False, candidates are evaluated in source-list order
(byte-identical F60 behavior). Opt-out kill switch for analysts who prefer pure
F60 ordering. M-6 (DEC-M6-PIVOT-008)."""
```

TOML round-trip: the existing `_strip_none` logic at `core/config.py:298` only strips `None` values; the new boolean field always has a value (default `True` or user-set `False`), so it round-trips through `[general.auto_pivot_policy]` as `dossier_aware_ranking = true/false`. No serializer change.

The flag is read once per hunt in `agent/tools.py::_execute_run_module` (the only in-tree consumer). Tests cover both flag values: when False, ranker is None and `process_results` behavior matches F60; when True, the ranker is supplied.

### 2.8 Cache invalidation (DEC-M6-PIVOT-009)

The DossierState used by the M-6 ranker is the same `pre_dossier` snapshot that M-4's hunt-site wiring loads once per hunt (`agent/tools.py:449`: `pre_dossier = load_dossier_state(self.workspace_mgr) or default_deferred_state()`). No additional `load_dossier_state` call. No memoization across hunts; no cache to invalidate.

Within a single `process_results` pass, the snapshot is treated as immutable — if a callback fired during the pass would have changed slot status (e.g., the first allowed pivot returns a new x509-certificate that would have transitioned Identity from EMPTY → PARTIAL), the change is observed on the NEXT hunt's pre_dossier, not within the pass. This matches M-4's "pre_dossier and post_dossier are snapshotted at hunt boundaries" discipline (DEC-M4-PERSIST-001). Mid-pass invalidation would force a re-rank after every callback — large complexity for unclear benefit. M-4's hunt-boundary semantics are inherited as canon.

### 2.9 Read-paths and integration surfaces

- **DossierState read authority:** `dossier/state.py::load_dossier_state(workspace_mgr)` (M-4) and `dossier/state.py::default_deferred_state()` (M-4 fallback for fresh workspaces). M-6 does NOT call these directly — it consumes the already-loaded `pre_dossier` from `_execute_run_module`'s local scope.
- **SCO-type → slot mapping authority:** `dossier/slots.py::SLOT_EVIDENCE_TYPES` (M-1, extended by M-2). The ranker reads this dict directly; any future SCO-type addition to the mapping is picked up by the ranker for free.
- **Slot weight authority:** `dossier/slots.py::SLOT_WEIGHTS` (M-1, DEC-M1-SLOTS-WEIGHT-AUTHORITY-001). The ranker imports this dict directly. No copy, no shadow, no override.
- **Slot status enum:** `dossier/slots.py::SlotStatus` (M-1). The `STATUS_MULTIPLIERS` map keys off this enum's members.
- **Event-bus invocation site:** `agent/tools.py::_execute_run_module` (single in-tree caller of `EventBus.process_results`). M-6 wires the ranker here.
- **Pivot-policy gate authority:** `core/pivot_policy.py::PivotPolicy` (F60). M-6 does NOT modify this class.
- **Config read site:** `core/config.py::AutoPivotPolicyConfig` (F60). M-6 adds one boolean field.

---

## 3. Removal targets (no parallel-authority residue)

M-6 is purely additive. There is no parallel authority to delete because no previous slice has shipped any dossier-aware ranking logic. The M-1 / M-2 / M-3 / M-4 / M-5 deferred-to-M-6 markers in the source already point to M-6 as the owner:

- `MASTER_PLAN.md:1707` lists M-6 as the owner of the dossier-aware auto-pivot policy.
- `MASTER_PLAN.md:1951` (Phase 17B / M-1 plan section) marks "No pivot-policy slot input — deferred to M-6" — M-6 closes this deferral.
- `MASTER_PLAN.md:2258` (Phase 17F / M-3 plan section) marks "No dossier-aware auto-pivot policy budget — M-6 owns" — M-6 closes this deferral.

There are no shadow ranker implementations to retire. There is no legacy "dossier preference" config flag to migrate. The `_missing_confidence_policy` registry (F60 DEC-60-PIVOT-POLICY-004) is unrelated and stays as-is.

---

## 4. The load-bearing acceptance test

The compound integration test that proves M-6 ships lives in `tests/test_dossier_pivot_integration.py` (NEW file). Three stages:

**Stage A — ranker reorders by slot pressure:**
1. Fresh workspace. Persist a fabricated DossierState via `save_dossier_state(workspace_mgr, state)` where Identity=EMPTY (weight 5.0), TTPs=PARTIAL (3.0), Infrastructure=FILLED (2.0), and the other slots are mixed.
2. Build a synthetic `results` list of 6 candidates in lexical/Infrastructure-first order: `[domain1, ipv4, email, url, x509, file]`.
3. Construct an `EventBus` with the M-6 ranker via `make_dossier_pivot_ranker(loaded_state)`. Subscribe 6 distinct dummy callbacks (one per SCO type), each tagged with a unique `_module_path` and instrumented to record fire-order.
4. Call `await event_bus.process_results(results, source_module="test/source", ranker=ranker)`.
5. Assert fire order: `[email, x509, url, file, domain1, ipv4]` — the two Identity SCOs first (score 5.0), then the two TTPs SCOs (score 1.5), then the two Infrastructure SCOs (score 0.0). Stable: within each tier the input order is preserved.

**Stage B — M-6 disabled → byte-identical F60 behavior:**
1. Same fabricated workspace + same `results` list.
2. Set `config.general.auto_pivot_policy.dossier_aware_ranking = False`.
3. Wire `process_results` with `ranker=None` (the path `_execute_run_module` takes when the flag is False).
4. Assert fire order: `[domain1, ipv4, email, url, x509, file]` — original input order.
5. Assert `event_bus.get_decision_log()` entries have `dossier_weight is None` for each entry (the diagnostic field is None when the ranker wasn't supplied).

**Stage C — budget is consumed by high-value pivots:**
1. Same fabricated workspace + a `results` list of 10 candidates: `[infra1, infra2, infra3, infra4, infra5, id1, id2, ttps1, ttps2, ttps3]` (lexical order favors Infrastructure).
2. Set `AutoPivotPolicyConfig(max_per_cascade=5)` (default).
3. Wire the M-6 ranker.
4. Call `await event_bus.process_results(results, source_module="test/source", ranker=ranker)`.
5. Assert exactly 5 callbacks fired (per-cascade budget), and they are the 2 Identity SCOs + 3 TTPs SCOs — NOT the 5 Infrastructure SCOs that arrived first.
6. Assert the decision log shows 5 ALLOW entries (Identity + TTPs) and 5 SKIP entries (Infrastructure, reason="budget"). All ALLOW entries carry `dossier_weight > 0`; the SKIP entries carry the slot-fill scores too (computed by the ranker), so the decision log records "we skipped this 0.0-score pivot at the budget gate" with full diagnostic context.

This is the "M-6 ships" acceptance test; it is mandatory in the Evaluation Contract (§7).

---

## 5. Invariant preservation matrix

| invariant | scope | M-6 check |
|-----------|-------|-----------|
| F59 (workspace single authority for SCO persistence) | `core/workspace.py` | BYTEWISE UNCHANGED. M-6 reads no workspace state directly; the pre_dossier snapshot is loaded by the M-4 hunt-site code that M-6 leaves alone. Test gate: `git diff main -- src/adversary_pursuit/core/workspace.py` is empty. |
| F60 (3-gate pivot policy + event-bus invariants) | `core/pivot_policy.py`, `core/event_bus.py` | `pivot_policy.py` BYTEWISE UNCHANGED — the 3-gate authority is preserved. `event_bus.py` gains one optional kwarg + a 1-line apply on `process_results`; `publish`, `subscribe`, `register_module_subscriptions`, decision-log handling — all byte-identical. The new kwarg defaults to None so existing callers and tests are unaffected. Test gate: `git diff main -- src/adversary_pursuit/core/pivot_policy.py` is empty. The F60 test suite (decision-log shape, budget short-circuit, IOC-value precedence, confidence threshold semantics, gate ordering DEC-60-PIVOT-POLICY-002) stays byte-identical and continues to pass without modification. |
| F62 (StreakManager single authority) | `core/streak.py` | BYTEWISE UNCHANGED. M-6 emits no score events; F62 sees no new event types. |
| F63 (milestone catch-up + sentinel-row pattern) | `gamification/celebrations.py` | UNCHANGED. M-6 changes only pivot ordering; cascade results still flow into `get_total_score()` via the existing F60 path. |
| F64 (de-duplicate LLM narration vs Rich panel) | `agent/tools.py::_DOSSIER_ACTIONS` filter | UNCHANGED. M-6 emits no new event subtype; the F64 filter set stays at the M-5 3-tuple. |
| Sacred Practice 12 (one authority per operational fact) | new + existing | The 3-gate engine has one authority (`core/pivot_policy.py`). The slot-weight + SCO-type mapping has one authority (`dossier/slots.py`). The ranker has one authority (`core/dossier_pivot.py`). The dossier-state read has one authority (`dossier/state.py`). No fact has two owners. |
| DEC-M1-SLOTS-WEIGHT-AUTHORITY-001 | `dossier/slots.py` | BYTEWISE UNCHANGED. M-6 imports `SLOT_WEIGHTS` read-only; never copies or shadows it. |
| DEC-M1-DOSSIER-001 (SLOT_EVIDENCE_TYPES is single authority) | `dossier/slots.py` | BYTEWISE UNCHANGED. M-6 imports `SLOT_EVIDENCE_TYPES` read-only. Any future SCO-type mapping addition lands in that authority and the ranker picks it up automatically. |
| DEC-60-PIVOT-POLICY-001 (PivotPolicy.evaluate is sole gate authority) | `core/pivot_policy.py` | PRESERVED. M-6 does not modify `evaluate` and does not introduce any parallel gate authority. The new ranker runs BEFORE the gates and only affects iteration order, not gate semantics. |
| DEC-60-PIVOT-POLICY-002 (3-gate ordering: ioc_value → confidence → budget) | `core/pivot_policy.py` | PRESERVED. M-6 does not add a 4th gate. The ranker is a pre-filter ordering layer, not a gate. The 3-gate order, short-circuit semantics, and decision shape stay byte-identical. |
| DEC-60-PIVOT-POLICY-005 (DecisionLogEntry shape) | `core/pivot_policy.py`, `core/event_bus.py` | EXTENDED ADDITIVELY. The seven required keys stay required. One optional field (`dossier_weight: float | None`) is added; existing consumers see no breaking change. Test gate: F60 decision-log tests continue to assert the seven required keys; new M-6 tests assert the optional field is populated when the ranker is supplied and `None` otherwise. |
| DEC-60-PIVOT-POLICY-006 (per-cascade + per-session budgets are sole flow control) | `core/pivot_policy.py` | PRESERVED. M-6 does not change budget semantics. The ranker only affects which candidates consume budget first. |
| DEC-M4-PERSIST-001 (DossierState persistence authority) | `dossier/state.py` | PRESERVED. M-6 consumes the M-4 snapshot via the M-4 read path; no new persistence layer, no schema change. |
| DEC-M5-* (M-5 surfaces) | M-5 modules | PRESERVED. M-6 does not touch M-5 surfaces. |

---

## 6. Evaluation Contract (9-key, ~25–35 tests)

**required_tests:**

The implementer ships ~25–35 tests across these files. Counts are minimums.

- `tests/test_dossier_pivot.py` **(NEW, ~12 tests)**:
  - `compute_slot_fill_score`: empty dossier (all slots EMPTY) → SCO whose type maps to a single slot returns `SLOT_WEIGHTS[slot]` (1)
  - `compute_slot_fill_score`: all slots FILLED → any SCO returns 0.0 (1)
  - `compute_slot_fill_score`: PARTIAL slot → returns weight × 0.5 (1)
  - `compute_slot_fill_score`: DEFERRED slot → returns 0.0 (1)
  - `compute_slot_fill_score`: SCO type not in SLOT_EVIDENCE_TYPES → returns 0.0 (1)
  - `compute_slot_fill_score`: SCO type that maps to two slots (none currently exists in M-5 mapping, but ranker must be robust to it) → returns sum of contributions (1)
  - `make_dossier_pivot_ranker`: returns callable that takes (results, source_module) and returns a new list (does not mutate input) (1)
  - `make_dossier_pivot_ranker`: ranks Identity-filling SCO above Infrastructure-filling SCO when Identity is EMPTY and Infrastructure is FILLED (1)
  - `make_dossier_pivot_ranker`: stable sort — ties preserve original input order (1)
  - `make_dossier_pivot_ranker`: tie-break on `x_abuse_confidence_score` when slot-fill scores are equal (higher confidence first; missing-field treated as -1) (DEC-M6-PIVOT-006) (1)
  - `make_dossier_pivot_ranker`: empty results list → empty output (1)
  - `STATUS_MULTIPLIERS` contains exactly 4 keys (EMPTY=1.0, PARTIAL=0.5, FILLED=0.0, DEFERRED=0.0); regression guard against accidental tweaks (1)

- `tests/test_dossier_pivot_eventbus.py` **(NEW, ~6 tests)**:
  - `EventBus.process_results(results, source_module, ranker=None)` → byte-identical to F60 (iteration follows input order) (1)
  - `EventBus.process_results(results, source_module, ranker=ranker)` → iteration follows ranked order (1)
  - `EventBus.process_results` does not call `ranker` when results is empty (1)
  - `EventBus.process_results` propagates `dry_run` to publish unchanged (regression — no kwarg-shadowing) (1)
  - `EventBus.process_results` calls `self._policy.reset_cascade_budget()` exactly once per call (regression) (1)
  - `EventBus.process_results` with ranker still respects the per-cascade budget (the ranked-first candidates allowed; the rest skip on budget) (1)

- `tests/test_dossier_pivot_integration.py` **(NEW, ~6 tests)**:
  - Stage A (§4): ranker reorders 6 candidates by slot pressure → Identity-first → TTPs → Infrastructure (1)
  - Stage B (§4): config flag False → byte-identical F60 ordering + decision log `dossier_weight` is None (1)
  - Stage C (§4): budget consumed by high-value pivots — Infrastructure (score 0.0) starves at budget gate (1)
  - All-EMPTY dossier: ranker is byte-identical to F60 order when every slot weight × multiplier is the same (Identity=5.0×1.0, Predictions=4.0×1.0 differ but no SCO type fills Predictions; the SCO-type-to-slot bias still sorts by SLOT_WEIGHTS) — assert ordering matches deterministic rule (1)
  - All-FILLED dossier: ranker is byte-identical to F60 order (all slot-fill scores 0.0; stable sort preserves input order) (1)
  - Decision log: each ALLOW entry carries `dossier_weight > 0` when ranker is supplied; each SKIP-on-budget entry also carries the score so diagnostics are complete (1)

- `tests/test_agent_tools.py` **(EXTEND, ~3 tests)**:
  - `_execute_run_module` supplies a ranker to `event_bus.process_results` when `cfg.general.auto_pivot_policy.dossier_aware_ranking` is True (default) (1)
  - `_execute_run_module` supplies `ranker=None` when the flag is False (1)
  - `_execute_run_module` reuses the already-loaded `pre_dossier` snapshot — no second `load_dossier_state` call (regression guard against double-load) (1)

- `tests/test_config.py` **(EXTEND, ~3 tests)**:
  - `AutoPivotPolicyConfig.dossier_aware_ranking` defaults to True (1)
  - TOML round-trip: writing the config with `dossier_aware_ranking = false` and reading it back preserves the value (1)
  - Backward-compat: a config TOML that omits `dossier_aware_ranking` deserializes with the default True (1)

- `tests/test_event_bus.py` or `tests/test_pivot_policy.py` **(EXTEND, ~2 tests for regression)**:
  - `EventBus.process_results` without the `ranker` kwarg (positional-arg-only call) is byte-identical to F60 — verifies the new kwarg is optional and didn't accidentally shift the existing positional signature (1)
  - `PivotPolicy.evaluate` returns the same `PolicyDecision` shape post-M-6 as pre-M-6 (regression guard — F60 byte invariant) (1)

**Total: ~32 new + extended tests.** Full suite green: ≥ M-5 baseline + ~32 new M-6 tests. Implementer must report the actual pre/post test counts in the readiness summary.

**required_evidence:**
- Full pytest output green for the worktree.
- `git diff main -- src/adversary_pursuit/core/pivot_policy.py` is empty (F60 BYTEWISE UNCHANGED).
- `git diff main -- src/adversary_pursuit/core/workspace.py` is empty (F59 BYTEWISE UNCHANGED, inherited from M-5).
- `git diff main -- src/adversary_pursuit/core/console.py` is empty (no pivot wiring there; M-6 stays out).
- `git diff main -- src/adversary_pursuit/models/database.py` is empty (DEC-DB-002 preserved).
- `git diff main -- src/adversary_pursuit/dossier/slots.py` is empty (DEC-M1-SLOTS-WEIGHT-AUTHORITY-001 preserved).
- `git diff main -- src/adversary_pursuit/dossier/state.py` is empty (M-4 byte-identical).
- `git diff main -- src/adversary_pursuit/dossier/predictions.py` is empty (M-5 byte-identical).
- `git diff main -- src/adversary_pursuit/dossier/scoring.py` is empty (M-3/M-5 byte-identical).
- `git diff main -- src/adversary_pursuit/dossier/slot_inference.py` is empty (M-5 byte-identical).
- `git diff main -- src/adversary_pursuit/dossier/panel.py` is empty (M-1 byte-identical).
- `git diff main -- src/adversary_pursuit/agent/chat.py` is empty (M-5 byte-identical).
- `git diff main -- src/adversary_pursuit/gamification/scoring.py` is empty.
- `git diff main -- src/adversary_pursuit/gamification/celebrations.py` is empty.
- `git diff main -- src/adversary_pursuit/core/streak.py` is empty.
- Demo trace (or test transcript) showing the §4 three-stage acceptance scenario: ranker reorders for slot pressure, opt-out disables ranking, budget gets consumed by high-value pivots.
- Tool count audit: `grep -c '"name":' src/adversary_pursuit/agent/tools.py` returns the same count as on M-5's `main` (30 tools) — M-6 adds no new LLM tool.

**required_authority_invariants:**
- F59: `core/workspace.py` BYTEWISE UNCHANGED.
- F60: `core/pivot_policy.py` BYTEWISE UNCHANGED. `core/event_bus.py` changes are limited to one optional kwarg + 1-line apply on `process_results`. `publish`, `subscribe`, `register_module_subscriptions`, decision-log shape (seven required keys) — all byte-identical. The F60 test suite passes without modification.
- F62: `core/streak.py` BYTEWISE UNCHANGED. No new score events; streak math sees no new event types.
- F63: `gamification/celebrations.py` UNCHANGED. M-6 emits no score events; milestone catch-up math unaffected.
- F64: `_DOSSIER_ACTIONS` filter UNCHANGED (no new event subtype to filter).
- Sacred Practice 12: ranker authority = `core/dossier_pivot.py`; 3-gate authority = `core/pivot_policy.py`; slot weight authority = `dossier/slots.py`; dossier state read authority = `dossier/state.py`. No fact has two owners.
- DEC-M1-SLOTS-WEIGHT-AUTHORITY-001: `SLOT_WEIGHTS` UNCHANGED; read-only consumer.
- DEC-M1-DOSSIER-001 (SLOT_EVIDENCE_TYPES single authority): UNCHANGED; read-only consumer.
- DEC-60-PIVOT-POLICY-001..007: ALL PRESERVED. M-6 explicitly does not relitigate any F60 decision.

**required_integration_points:**
- `core/dossier_pivot.py` (NEW: `make_dossier_pivot_ranker(dossier_state) -> Callable`, `compute_slot_fill_score(sco_type, dossier_state) -> float`, `STATUS_MULTIPLIERS: dict[SlotStatus, float]`).
- `core/event_bus.py` (EXTEND: `process_results` gains `ranker: Callable | None = None` keyword parameter; body adds `ranked = ranker(results, source_module) if ranker is not None else results` immediately after `self._policy.reset_cascade_budget()`; the `for result in results` loop iterates `ranked`; the per-call dossier-weight side-channel dict is built by the ranker and consulted by `publish` when building the decision-log entry's `dossier_weight` field).
- `core/pivot_policy.py` (EXTEND: ONLY the `DecisionLogEntry` TypedDict gains the optional `dossier_weight: float | None` field. `PivotPolicy` class is BYTEWISE UNCHANGED. `build_log_entry` is BYTEWISE UNCHANGED — the new field is populated by `EventBus.publish` after the helper returns, not inside the helper).
- `core/config.py` (EXTEND: `AutoPivotPolicyConfig` gains `dossier_aware_ranking: bool = True`).
- `agent/tools.py` (EXTEND: `_execute_run_module` wires the ranker. Insertion point: between the existing M-4 load of `pre_dossier` at line 449 and the `event_bus.process_results(...)` call at line 633. Read `self.config.general.auto_pivot_policy.dossier_aware_ranking`; when True, build `ranker = make_dossier_pivot_ranker(pre_dossier)`; pass `ranker=ranker` to `process_results`. No new `load_dossier_state` call — reuse the M-4 `pre_dossier`).
- NO change to `dossier/__init__.py` (the new symbols live in `core/dossier_pivot.py`, not in `dossier/`). NO change to `dossier/state.py`, `dossier/slots.py`, `dossier/slot_inference.py`, `dossier/predictions.py`, `dossier/scoring.py`, `dossier/panel.py`. NO change to `agent/chat.py`. NO change to `core/console.py`. NO change to `core/workspace.py`. NO change to `models/database.py`.

**Note on the `core/pivot_policy.py` "one-field" edit:** the `DecisionLogEntry` TypedDict edit is the ONLY change to that file. The forbidden-list note "BYTEWISE UNCHANGED" above refers to the `PivotPolicy` class and its methods; the TypedDict addition at the top of the module is a strictly additive type-shape extension that does not affect the gate engine. If the implementer prefers an alternate location for the `dossier_weight` field (e.g., a new `M6DecisionLogEntry` extension TypedDict in `core/dossier_pivot.py` that wraps the F60 entry), that is acceptable and would preserve `pivot_policy.py` BYTEWISE UNCHANGED. The plan picks the in-place additive field for ergonomic reasons (consumers see one entry shape, not a discriminated union), but the implementer may elevate the choice to the reviewer if the type-checker raises objections. Either resolution preserves all F60 invariants. (DEC-M6-PIVOT-007 records this implementer-latitude clause.)

**forbidden_shortcuts:**
- NO modification of `core/pivot_policy.py::PivotPolicy` class body or any of its methods (`evaluate`, `build_log_entry`, `_evaluate_ioc_value`, `_evaluate_confidence`, `_evaluate_budget`, `_load_static_rules`, `_load_user_lists`, etc.). The class is BYTEWISE UNCHANGED in M-6.
- NO 4th gate added to `PivotPolicy.evaluate`. DEC-60-PIVOT-POLICY-002 (3-gate ordering) is preserved.
- NO modification of `EventBus.publish` ordering of subscribers (the per-SCO callback iteration order is unchanged).
- NO modification of `core/workspace.py` (F59 / M-5 BYTEWISE UNCHANGED).
- NO modification of `models/database.py` (DEC-DB-002 preserved).
- NO modification of `dossier/slots.py`, `dossier/state.py`, `dossier/predictions.py`, `dossier/scoring.py`, `dossier/slot_inference.py`, `dossier/panel.py`.
- NO modification of `agent/chat.py` (no new meta-command for M-6 — `autopivot on/off` remains the user surface).
- NO new LLM tool. Tool count stays at 30.
- NO new ScoreEvent action. No `dossier_pivot_preferred` event, no `slot_pressure_ranked` event.
- NO new event-bus subscriber.
- NO second `load_dossier_state` call inside `_execute_run_module` — reuse the M-4 `pre_dossier`.
- NO memoization of the ranker across hunts. The ranker is rebuilt each hunt from the current `pre_dossier`.
- NO mid-pass re-ranking (within a single `process_results` call) when a callback would have changed slot status. Hunt-boundary snapshot semantics are inherited from M-4.
- NO opt-in flag inversion. The flag is `dossier_aware_ranking: bool = True` (default True). Inverting the default to False with a "set this to True to opt in" knob would either (a) leave M-6 invisible to existing users until they discover the flag, or (b) require shipping a migration message. Default True with opt-out is the correct shape.
- NO Rich markup in any new module or test output.
- NO refactor of `agent/tools.py`, `core/event_bus.py`, or `core/pivot_policy.py` beyond the documented additions.

**rollback_boundary:** single feature branch revertible as one merge commit. Revert restores M-5 byte state; removes `core/dossier_pivot.py`, the M-6 `EventBus.process_results` `ranker` kwarg, the M-6 `AutoPivotPolicyConfig.dossier_aware_ranking` field, the M-6 `DecisionLogEntry.dossier_weight` optional field, and the M-6 `agent/tools.py::_execute_run_module` ranker-wiring lines. Workspaces persisted under M-6 are unaffected — M-6 ships no new persistence, no new schema, no new event types. Config TOML files that wrote `[general.auto_pivot_policy]` with `dossier_aware_ranking = true/false` will deserialize fine under M-5's `AutoPivotPolicyConfig` because Pydantic's extra-field default (`ignore`) drops unknown fields silently — the post-revert workspace continues to function without manual intervention. No schema migrations, no settings changes, `streak.json` untouched. Documented no-op rollback hazard: none.

**acceptance_notes:** the implementer should treat M-6 as a "smallest-possible additive layer that influences ordering without changing the gate" slice. The biggest risk is over-engineering — building a 4th gate, a re-implementation of `PivotPolicy`, or a parallel ranker authority. The smallest correct change is the pure-function ranker module + one kwarg + one config field + one wiring line in `_execute_run_module`. If the implementer finds themselves modifying `PivotPolicy.evaluate` or duplicating gate logic, that is a sign the layer line has been crossed — halt and report. The §4 three-stage acceptance test is the binding contract: Stage A proves the ranking; Stage B proves the opt-out; Stage C proves the budget interaction. All three must pass.

The implementer should also verify the F60 test suite passes BYTEWISE — if any F60 test fails post-M-6, M-6 has accidentally changed gate semantics and must be re-scoped.

**ready_for_guardian_definition:**
- All required_tests green; full suite green ≥ M-5 baseline + new M-6 tests.
- All forbidden-file `git diff main` outputs empty (paste each, verifying byte-identical).
- `core/pivot_policy.py::PivotPolicy` class diff is empty (the file diff is limited to the `DecisionLogEntry` TypedDict addition if the in-place option is taken; the implementer may take the alternate-location option per DEC-M6-PIVOT-007 to keep `pivot_policy.py` BYTEWISE UNCHANGED, in which case the diff is fully empty).
- `core/event_bus.py` diff is limited to the documented `process_results` kwarg + 1-line apply + (optional) per-call dossier-weight side-channel.
- `core/config.py` diff is limited to the new `AutoPivotPolicyConfig.dossier_aware_ranking` field.
- `agent/tools.py` diff is limited to the documented ranker-wiring lines in `_execute_run_module`.
- Phase 17I appended to `MASTER_PLAN.md` AND committed in the same commit as source (AP #74 orphan-prevention; M-3 / M-4 / M-5 demonstrated the pattern works).
- Phase 17H status flipped: `in-progress` → `completed (landed 2026-06-07, merge e29e8b1, impl c5dd6bf)`. M-5 closeout drift fixed in this commit.
- "Active Phase Pointer" tail-line updated from `W-68-M5-DENIAL-STRATEGIES` to `W-68-M6-DOSSIER-PIVOT`.
- Plan Status table gains a Phase 17I row.
- Tool count audit: `grep -c '"name":' src/adversary_pursuit/agent/tools.py` returns the same count as on main (30 tools).
- Implementer commit message follows `feat(dossier-pivot):` or `feat(pivot):` prefix, references `#68` + `DEC-M6-PIVOT-001..009`.

---

## 7. Scope Manifest

**Allowed / Required (the implementer MUST touch these):**
- `src/adversary_pursuit/core/dossier_pivot.py` **(NEW)** — pure-function ranker module: `make_dossier_pivot_ranker`, `compute_slot_fill_score`, `STATUS_MULTIPLIERS`.
- `src/adversary_pursuit/core/event_bus.py` (EXTEND: `process_results` gains optional `ranker` kwarg + 1-line apply; optional per-call dossier-weight side-channel for decision-log enrichment).
- `src/adversary_pursuit/core/pivot_policy.py` (EXTEND, narrow: `DecisionLogEntry` TypedDict gains optional `dossier_weight: float | None` field — see DEC-M6-PIVOT-007 for the alternate-location latitude. `PivotPolicy` class and all its methods are BYTEWISE UNCHANGED).
- `src/adversary_pursuit/core/config.py` (EXTEND: `AutoPivotPolicyConfig` gains `dossier_aware_ranking: bool = True`).
- `src/adversary_pursuit/agent/tools.py` (EXTEND, narrow: `_execute_run_module` wires the ranker between the existing M-4 `pre_dossier` load and the existing `event_bus.process_results` call).
- `tests/test_dossier_pivot.py` **(NEW)** — ranker unit tests.
- `tests/test_dossier_pivot_eventbus.py` **(NEW)** — `EventBus.process_results` kwarg-integration tests.
- `tests/test_dossier_pivot_integration.py` **(NEW)** — §4 three-stage acceptance test.
- `tests/test_agent_tools.py` (EXTEND — `_execute_run_module` ranker wiring tests).
- `tests/test_config.py` (EXTEND — flag default + TOML round-trip).
- `tests/test_event_bus.py` or `tests/test_pivot_policy.py` (EXTEND — F60 regression guards).
- `MASTER_PLAN.md` — Phase 17I section + Phase 17H status flip + Plan Status table row + "Active Phase Pointer" tail-line update. **Implementer MUST `git add MASTER_PLAN.md` in the same commit as source (AP #74 orphan-prevention).**

**Forbidden (preserved authorities):**
- `src/adversary_pursuit/core/workspace.py` (F59 / M-5 BYTEWISE UNCHANGED)
- `src/adversary_pursuit/models/database.py` (DEC-DB-002 preserved)
- `src/adversary_pursuit/dossier/slots.py` (DEC-M1-SLOTS-WEIGHT-AUTHORITY-001 + SLOT_EVIDENCE_TYPES authority preserved)
- `src/adversary_pursuit/dossier/state.py` (M-4 byte-identical)
- `src/adversary_pursuit/dossier/predictions.py` (M-5 byte-identical)
- `src/adversary_pursuit/dossier/scoring.py` (M-3 / M-5 byte-identical)
- `src/adversary_pursuit/dossier/slot_inference.py` (M-5 byte-identical)
- `src/adversary_pursuit/dossier/panel.py` (M-1 byte-identical)
- `src/adversary_pursuit/dossier/__init__.py` (no new exports — the ranker lives in `core/dossier_pivot.py`, not in `dossier/`)
- `src/adversary_pursuit/gamification/scoring.py` (M-3 byte-identical)
- `src/adversary_pursuit/gamification/celebrations.py` (F63 invariant)
- `src/adversary_pursuit/core/streak.py` (F62 invariant)
- `src/adversary_pursuit/core/console.py` (no pivot wiring; M-6 stays out)
- `src/adversary_pursuit/agent/chat.py` (M-5 byte-identical; no new meta-command)
- `src/adversary_pursuit/agent/runner.py` (no agent-runner changes)
- `src/adversary_pursuit/gamification/modes.py` (C-1/C-2 territory)
- `src/adversary_pursuit/modules/**` (no module changes)
- `pyproject.toml`, hooks, settings, `CLAUDE.md`, `agents/`, `runtime/`

**Expected state authorities touched:**
- in-memory `DossierState` (read at hunt start via existing M-4 `pre_dossier` load; passed to the M-6 ranker; not mutated)
- in-memory `list[dict]` of candidate SCOs (re-ordered by the ranker; the ranker returns a NEW list — input is not mutated)
- in-memory `EventBus._policy._cascade_count` and `_session_count` budget counters (incremented by F60 via the unchanged path; M-6 only affects which candidates consume budget first)
- in-memory `EventBus._decision_log` list (each entry gains the optional `dossier_weight` field when the ranker is supplied)
- no SQLite writes; no schema-events sentinel rows; no workspace state mutations whatsoever

---

## 8. Decision Log (Phase 17I / M-6 binding)

| DEC ID | Decision | Rationale |
|--------|----------|-----------|
| **DEC-M6-PIVOT-001** | M-6 ships as a NEW layer ABOVE F60, not as a modification to F60. The 3-gate engine in `core/pivot_policy.py::PivotPolicy.evaluate` is BYTEWISE UNCHANGED. M-6's dossier-aware logic lives in a new module `core/dossier_pivot.py` as a pure-function candidate-ordering layer that runs in `EventBus.process_results` BEFORE the F60 gates see candidates. | Sacred Practice 12 (single source of truth for the 3-gate engine). DEC-60-PIVOT-POLICY-001 (PivotPolicy.evaluate is sole gate authority) preserved. DEC-60-PIVOT-POLICY-002 (3-gate ordering) preserved. The dispatch context's "wrap is preferred" guidance is honored: wrapping is the architecturally clean answer because the gate decision (boolean allow/skip) and the rank decision (continuous priority weight) are independent concerns that benefit from separate authorities. |
| **DEC-M6-PIVOT-002** | The ranker lives in NEW module `src/adversary_pursuit/core/dossier_pivot.py`. Not in `pivot_policy.py` (would blur ownership). Not in `dossier/__init__.py` or `dossier/pivot_ranking.py` (would cross the dossier→core layer line in the wrong direction). | Module name communicates intent. `pivot_policy.py` owns gate semantics; `dossier_pivot.py` owns dossier-aware ranking semantics. The new module imports from `dossier/slots.py` (one-way `core → dossier` import, consistent with `pivot_policy.py`'s import of `core.config`); no reverse dependency is introduced into the dossier package. |
| **DEC-M6-PIVOT-003** | `STATUS_MULTIPLIERS: dict[SlotStatus, float] = {EMPTY: 1.0, PARTIAL: 0.5, FILLED: 0.0, DEFERRED: 0.0}` is the single authority for status-to-multiplier conversion. Lives in `core/dossier_pivot.py` as a module-level constant. | Single source of truth. EMPTY at max pressure (1.0) is the canonical "this slot has no evidence — prefer pivots that fill it" signal. PARTIAL at half pressure (0.5) reflects "some evidence — still useful to fill more, but less critical." FILLED at 0.0 means "no further pressure — slot is done." DEFERRED at 0.0 means "no inference path active yet — don't preference pivots based on a slot we can't score" (in M-6, only Targeting is DEFERRED; M-5 promoted Denial out of DEFERRED). The 1.0 / 0.5 / 0.0 / 0.0 shape is intentionally crude — refinement to non-linear curves can land in a future slice if profile data shows the current shape mis-ranks. |
| **DEC-M6-PIVOT-004** | Slot-fill score formula: `score(sco) = Σ_{slot ∈ SLOT_EVIDENCE_TYPES[sco["type"]]} (SLOT_WEIGHTS[slot] × STATUS_MULTIPLIERS[dossier_state.slots[slot].status])`. SCO types absent from SLOT_EVIDENCE_TYPES score 0.0. The formula is sum-over-slots (additive), not product, so a multi-slot SCO type accumulates contributions naturally. | Additive aggregation aligns with the M-3 score-event model (per-IOC weights ADD across multiple matching slot fills; no per-slot product semantics anywhere in the codebase). The formula reuses `SLOT_WEIGHTS` and `SLOT_EVIDENCE_TYPES` read-only (Sacred Practice 12). The fact that most M-5 SCO types map to a single slot (only Predictions and Denial have non-SCO routes) makes the additive form usually a single-term sum; the form generalizes cleanly if a future SCO type maps to multiple slots. |
| **DEC-M6-PIVOT-005** | Slot-fill score is the SOLE ranking input. There is no "combine with F60 confidence-gate score" step because F60's confidence gate is a boolean threshold, not a ranking score. The M-6 ranker pre-orders; F60 then evaluates each candidate against its three gates exactly as before. | The dispatch context posed additive-vs-multiplicative combination of two ranking scores; planner finding: F60 has no ranking score to combine with — only gates. Inventing a "confidence score" inside M-6 just to give the multiplication a left operand would shadow F60's confidence-threshold semantics and create a parallel authority for confidence interpretation (Sacred Practice 12 violation). The clean answer is "the M-6 score is the rank; F60 gates run on the ranked order; both remain in their proper layers." |
| **DEC-M6-PIVOT-006** | Tie-break when two candidates have equal slot-fill score: (1) higher `x_abuse_confidence_score` first (missing field treated as -1); (2) stable — original input order preserved. | Tier 1 of the tie-break mirrors F60's confidence-gate preference for high-confidence indicators (analyst signal). Tier 2 preserves determinism so the rank is reproducible across runs with identical inputs. Stable sort fall-back means the ranker is byte-deterministic — important for test assertions and for debugging "why did the ranker pick this order?" questions. |
| **DEC-M6-PIVOT-007** | `DecisionLogEntry` (F60 TypedDict at `core/pivot_policy.py`) gains ONE optional field `dossier_weight: float | None`. The seven required keys stay required (DEC-60-PIVOT-POLICY-005 preserved). The new field is populated by `EventBus.publish` after `build_log_entry` returns, by consulting a small per-call dossier-weight side-channel dict that the ranker populates (keyed by source_sco_value or sco_id). When the ranker is None (M-6 disabled), the field is None for every entry. **Implementer latitude:** if the type-checker objects to the optional-field addition in F60's TypedDict, the implementer may instead define a new TypedDict in `core/dossier_pivot.py` that extends F60's shape with the extra field; either resolution preserves the seven required keys and the F60 invariant. | Diagnostics matter for debugging M-6 decisions. Without the field, introspecting "why was this pivot ranked above that one?" requires re-running `compute_slot_fill_score` by hand. The field is strictly additive (existing consumers don't break). Latitude clause documents the only place a reviewer might need to make a judgment call. |
| **DEC-M6-PIVOT-008** | `AutoPivotPolicyConfig.dossier_aware_ranking: bool = True` (default ON). Flag is read once per hunt by `agent/tools.py::_execute_run_module`; when True, builds the M-6 ranker and passes it to `event_bus.process_results`; when False, passes `ranker=None` and behavior is byte-identical to F60. The default is True because M-6 ships as an enhancement that users want by default; the flag is the opt-out kill switch for unforeseen interactions. | Opt-out default beats opt-in: opt-in defaults leave M-6 invisible to existing users until they discover the flag. The flag exists for safety (rollback without revert) and for analyst preference (some workflows may want pure F60 order for reproducibility against historical decision logs). TOML round-trip is preserved by the existing `_strip_none` logic at `core/config.py:298` — no serializer change. |
| **DEC-M6-PIVOT-009** | Cache invalidation: the M-6 ranker consumes the `pre_dossier` snapshot loaded once per hunt by the existing M-4 wiring at `agent/tools.py:449`. The ranker reuses that snapshot; no second `load_dossier_state` call is made. Within a single `process_results` pass, the snapshot is treated as immutable — if a callback fired during the pass would have changed slot status, the change is observed on the NEXT hunt's `pre_dossier`, not within the pass. Hunt-boundary snapshot semantics inherited from M-4 (DEC-M4-PERSIST-001). | Mid-pass invalidation would force a re-rank after every callback — large complexity for unclear benefit. M-4's hunt-boundary discipline is established and works; reusing it is the smaller, safer answer. The "no second load" rule is enforced by a regression test in `tests/test_agent_tools.py` to prevent future drift. |

---

## 9. Open question for the user (none)

No user-decision boundary is required to start the implementer. All planner decisions are recorded in §8. The dispatch context's "additive or multiplicative" combination question is resolved in DEC-M6-PIVOT-005 (neither — F60 has no ranking score to combine with). The dispatch context's "wrap vs replace F60" question is resolved in DEC-M6-PIVOT-001 (wrap). The dispatch context's "where does the dossier-aware logic live" question is resolved in DEC-M6-PIVOT-002 (new module `core/dossier_pivot.py`). The dispatch context's "tie-break" question is resolved in DEC-M6-PIVOT-006 (confidence then stable). The dispatch context's "cache invalidation" question is resolved in DEC-M6-PIVOT-009 (no cache; hunt-boundary snapshot semantics inherited from M-4).

If the implementer surfaces an unforeseen blast radius (e.g., the F60 test suite breaks because the `DecisionLogEntry` TypedDict edit is rejected by strict type-checking), the implementer halts and reports — that is a planner re-stage trigger, not an in-flight design call. The DEC-M6-PIVOT-007 implementer-latitude clause covers the most likely such surprise (alternate-location for the diagnostic field).

---

## 10. Subsequent Workflow Cue

After M-6 lands, the recommended next workflow is **M-7 — Reports / Celebrations / Badges Dossier-Aware Upgrade (absorbs issue #32)** per `.claude/plans/dossier-reframe-v2-roadmap.md` §M-7. M-7 honors F64 panel-authority invariants and depends on M-3 + M-4 + M-5 + M-6. M-7 prefers post-C-1 (LLM persona voice for narrated celebrations); C-1 has landed since 2026-05-28, so M-7 can schedule any time after M-6 lands.

M-8 (Cleanup, Closeout, and Novel-Method Achievement) closes the v0.3.x dossier roadmap and depends on M-7.

C-3 (Philosophy + Bureaucratese modes — `sun_tzu`, `bruce_lee`, `bureaucrat`) remains independent of the dossier roadmap (DEC-30-CHARACTER-V2-007) and may land in parallel with M-7 or in any wave.

The Targeting slot 5 inference path remains DEFERRED after M-6. A future slice (not currently scheduled — likely M-7 or M-8) will introduce either a user-supplied victim-industry profile or a victim-industry extractor from SCO data that AP modules surface in a future update. The planner that opens that slice records the trigger criteria as `DEC-MX-TARGETING-001`. M-6's ranker treats Targeting as DEFERRED (multiplier 0.0) so it never preferences pivots based on a slot we cannot currently score; when Targeting becomes a real slot, the ranker picks up the change automatically via `SLOT_EVIDENCE_TYPES` + `SLOT_WEIGHTS` (no M-6 change required).
